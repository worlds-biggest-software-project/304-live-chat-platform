# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Live Chat Platform · Created: 2026-05-24

## Philosophy

This model keeps the core conversation and message fields as typed relational columns while using JSONB for variable data: visitor properties, channel-specific configuration, widget appearance, routing conditions, AI interaction metadata, and CRM sync state. The result is roughly half the table count of the normalized model while retaining fast indexed queries on the fields that matter most (status, assignee, timestamp).

The hybrid approach matches how modern chat platforms like Crisp, Chatwoot, and Freshchat expose their APIs: a handful of well-defined top-level fields with flexible metadata bags. It enables rapid iteration — adding a new channel type, visitor property, or AI feature requires no schema migration, just new keys in existing JSONB columns.

**Best for:** Teams building an MVP or early-stage product that needs to ship fast, iterate on visitor data models, and support diverse channel types without schema rigidity.

**Trade-offs:**
- **Pro:** ~12 tables vs. 27 in normalized — dramatically simpler schema
- **Pro:** New visitor properties, channel types, and AI features need zero migrations
- **Pro:** GIN indexes on JSONB enable efficient containment queries
- **Pro:** Visitor traits, CRM data, and AI classification all accessible without JOINs
- **Con:** No foreign key constraints within JSONB
- **Con:** Complex JSONB aggregations slower than typed column aggregations
- **Con:** Application-level validation required for JSONB subfields
- **Con:** JSONB columns can grow unbounded without discipline

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WebSocket (RFC 6455) | Presence tracked via visitor/agent `is_online` fields, not a separate table |
| GDPR | `visitors.consent` JSONB, `visitors.erasure_status` for right-to-erasure |
| ISO/IEC 27001 | `audit_log` table with JSONB changes for security audit |
| WCAG 2.1 | Widget accessibility settings in `channels.widget_config` JSONB |
| OAuth 2.0 | `integrations` table covers CRM and third-party auth |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free', 'starter', 'professional', 'enterprise')),
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"business_hours": {"timezone": "America/New_York", "schedule": {...}}, "routing": {"default_team": "uuid", "method": "round_robin"}, "branding": {...}}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'agent' CHECK (role IN ('owner', 'admin', 'agent')),
    status TEXT NOT NULL DEFAULT 'offline' CHECK (status IN ('online', 'away', 'busy', 'offline')),
    team_ids UUID[] NOT NULL DEFAULT '{}',
    skills TEXT[] NOT NULL DEFAULT '{}',
    max_concurrent_chats INT NOT NULL DEFAULT 5,
    active_conversation_count INT NOT NULL DEFAULT 0,
    preferences JSONB NOT NULL DEFAULT '{}',
    -- {"timezone": "America/New_York", "notifications": {"sound": true, "desktop": true}, "signature": "..."}
    last_active_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"auto_assign": true, "assignment_method": "round_robin", "max_per_agent": 5, "skills_required": ["billing"]}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_agents_tenant ON agents(tenant_id);
CREATE INDEX idx_agents_status ON agents(tenant_id, status);
CREATE INDEX idx_agents_skills ON agents USING GIN(skills);
CREATE INDEX idx_agents_teams ON agents USING GIN(team_ids);
```

---

## Visitors

```sql
CREATE TABLE visitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    anonymous_id TEXT,
    identified_user_id TEXT,
    email TEXT,
    name TEXT,
    is_online BOOLEAN NOT NULL DEFAULT FALSE,
    traits JSONB NOT NULL DEFAULT '{}',
    -- {"plan": "enterprise", "company": "Acme Corp", "company_size": "500+", "industry": "saas", "ltv_cents": 120000, "health_score": 85}
    browser_info JSONB NOT NULL DEFAULT '{}',
    -- {"user_agent": "...", "language": "en-US", "timezone": "America/New_York", "screen_resolution": "1920x1080"}
    current_page JSONB NOT NULL DEFAULT '{}',
    -- {"url": "https://app.example.com/settings", "title": "Settings", "referrer": "..."}
    consent JSONB NOT NULL DEFAULT '{}',
    -- {"tracking": {"granted": true, "at": "2026-01-15T..."}, "marketing": {"granted": false}}
    crm_data JSONB NOT NULL DEFAULT '{}',
    -- {"salesforce": {"contact_id": "003...", "account_id": "001...", "deal_stage": "Negotiation"}, "hubspot": {"contact_id": 12345}}
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    total_conversations INT NOT NULL DEFAULT 0,
    total_page_views INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_visitors_tenant ON visitors(tenant_id);
CREATE INDEX idx_visitors_email ON visitors(tenant_id, email);
CREATE INDEX idx_visitors_anonymous ON visitors(tenant_id, anonymous_id);
CREATE INDEX idx_visitors_identified ON visitors(tenant_id, identified_user_id);
CREATE INDEX idx_visitors_online ON visitors(tenant_id, is_online) WHERE is_online;
CREATE INDEX idx_visitors_traits ON visitors USING GIN(traits);
```

---

## Channels

```sql
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    channel_type TEXT NOT NULL CHECK (channel_type IN ('web_chat', 'email', 'sms', 'whatsapp', 'facebook_messenger', 'twitter_dm', 'telegram', 'slack', 'api')),
    name TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    channel_config JSONB NOT NULL DEFAULT '{}',
    -- web_chat: {"domain_whitelist": ["example.com"], "auto_resolve_minutes": 30}
    -- email: {"imap_host": "...", "smtp_host": "...", "from_address": "support@..."}
    -- whatsapp: {"phone_number": "+1...", "business_id": "..."}
    widget_config JSONB NOT NULL DEFAULT '{}',
    -- {"primary_color": "#4A90D9", "position": "bottom-right", "greeting": "Hi!", "logo_url": "...", "pre_chat_form": {"enabled": true, "fields": [...]}, "offline_message": "..."}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channels_tenant ON channels(tenant_id, is_active);
```

---

## Conversations & Messages

```sql
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    conversation_number BIGINT NOT NULL,
    visitor_id UUID NOT NULL REFERENCES visitors(id),
    channel_id UUID NOT NULL REFERENCES channels(id),
    assignee_id UUID REFERENCES agents(id),
    team_id UUID REFERENCES teams(id),
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'snoozed', 'resolved', 'closed')),
    priority TEXT NOT NULL DEFAULT 'normal' CHECK (priority IN ('urgent', 'high', 'normal', 'low')),
    tags TEXT[] NOT NULL DEFAULT '{}',
    last_message_preview TEXT,
    last_message_at TIMESTAMPTZ,
    last_message_sender_type TEXT,
    message_count INT NOT NULL DEFAULT 0,
    ai_context JSONB NOT NULL DEFAULT '{}',
    -- {"intent": "billing_question", "intent_confidence": 0.92, "sentiment": "neutral", "language": "en", "auto_resolved": false, "suggested_articles": ["uuid1"]}
    sla_status JSONB NOT NULL DEFAULT '{}',
    -- {"first_response": {"target_at": "...", "achieved_at": "...", "breached": false}}
    routing_metadata JSONB NOT NULL DEFAULT '{}',
    -- {"source_url": "https://app.example.com/billing", "pre_chat_data": {"email": "...", "department": "billing"}, "referrer": "..."}
    rating SMALLINT CHECK (rating BETWEEN 1 AND 5),
    rating_comment TEXT,
    first_response_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    snoozed_until TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, conversation_number)
);

CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    sender_type TEXT NOT NULL CHECK (sender_type IN ('visitor', 'agent', 'bot', 'system')),
    sender_id UUID,
    message_type TEXT NOT NULL DEFAULT 'text' CHECK (message_type IN ('text', 'image', 'file', 'card', 'carousel', 'quick_reply', 'form', 'system')),
    content_text TEXT,
    content_html TEXT,
    content_data JSONB,
    -- card: {"title": "...", "subtitle": "...", "image_url": "...", "buttons": [{"label": "...", "action": "url", "value": "..."}]}
    -- quick_reply: {"prompt": "Was this helpful?", "options": [{"label": "Yes", "value": "yes"}, {"label": "No", "value": "no"}]}
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    attachments JSONB NOT NULL DEFAULT '[]',
    -- [{"file_name": "invoice.pdf", "content_type": "application/pdf", "size_bytes": 45000, "storage_key": "s3://...", "thumbnail_key": "..."}]
    delivered_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conv_tenant_status ON conversations(tenant_id, status);
CREATE INDEX idx_conv_assignee ON conversations(assignee_id) WHERE assignee_id IS NOT NULL;
CREATE INDEX idx_conv_visitor ON conversations(visitor_id);
CREATE INDEX idx_conv_updated ON conversations(tenant_id, updated_at DESC);
CREATE INDEX idx_conv_tags ON conversations USING GIN(tags);
CREATE INDEX idx_conv_ai ON conversations USING GIN(ai_context);
CREATE INDEX idx_messages_conv ON messages(conversation_id, created_at);
```

---

## Routing & Automation

```sql
CREATE TABLE automation_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    rule_type TEXT NOT NULL CHECK (rule_type IN ('routing', 'auto_assign', 'auto_tag', 'auto_respond', 'sla', 'proactive_trigger')),
    position INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    conditions JSONB NOT NULL DEFAULT '{}',
    -- {"all": [{"field": "visitor.traits.plan", "op": "is", "value": "enterprise"}, {"field": "channel_type", "op": "is", "value": "web_chat"}]}
    actions JSONB NOT NULL DEFAULT '[]',
    -- [{"type": "assign_team", "team_id": "..."}, {"type": "set_priority", "value": "high"}, {"type": "add_tag", "value": "enterprise"}]
    run_count BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_automation_rules_tenant ON automation_rules(tenant_id, is_active, rule_type);
```

---

## Integrations

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    integration_type TEXT NOT NULL CHECK (integration_type IN ('salesforce', 'hubspot', 'pipedrive', 'slack', 'zapier', 'custom_webhook')),
    name TEXT NOT NULL,
    credentials_encrypted TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    -- salesforce: {"instance_url": "...", "sync_contacts": true, "sync_conversations": true, "field_mapping": {...}}
    -- slack: {"workspace_id": "...", "channel_id": "...", "notify_on": ["new_conversation", "sla_breach"]}
    -- webhook: {"url": "...", "events": ["conversation.started", "message.sent"], "secret": "..."}
    sync_status JSONB NOT NULL DEFAULT '{}',
    -- {"last_sync_at": "...", "status": "idle", "contacts_synced": 1234, "error": null}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_tenant ON integrations(tenant_id, is_active);
```

---

## Canned Responses

```sql
CREATE TABLE canned_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    shortcut TEXT NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    scope TEXT NOT NULL DEFAULT 'global' CHECK (scope IN ('personal', 'team', 'global')),
    owner_id UUID REFERENCES agents(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, shortcut)
);
```

---

## Audit

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    action TEXT NOT NULL,
    actor_type TEXT NOT NULL,
    actor_id UUID,
    changes JSONB NOT NULL DEFAULT '{}',
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
```

---

## API Keys

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Route conversation based on visitor traits

```sql
SELECT ar.actions
FROM automation_rules ar
WHERE ar.tenant_id = 'tenant-uuid'
  AND ar.is_active = TRUE
  AND ar.rule_type = 'routing'
  AND (
    ar.conditions @> '{"all": [{"field": "visitor.traits.plan", "op": "is", "value": "enterprise"}]}'
  )
ORDER BY ar.position
LIMIT 1;
```

### Find available agents with matching skills

```sql
SELECT a.id, a.name, a.active_conversation_count
FROM agents a
WHERE a.tenant_id = 'tenant-uuid'
  AND a.status = 'online'
  AND a.active_conversation_count < a.max_concurrent_chats
  AND a.skills @> ARRAY['billing']
  AND a.team_ids @> ARRAY['team-uuid']::UUID[]
ORDER BY a.active_conversation_count ASC
LIMIT 1;
```

### CRM-aware conversation list for enterprise visitors

```sql
SELECT c.id, c.conversation_number, c.status,
       v.name AS visitor_name,
       v.traits->>'plan' AS plan,
       v.crm_data->'salesforce'->>'deal_stage' AS sf_deal_stage,
       c.ai_context->>'intent' AS intent,
       c.ai_context->>'sentiment' AS sentiment
FROM conversations c
JOIN visitors v ON c.visitor_id = v.id
WHERE c.tenant_id = 'tenant-uuid'
  AND c.status IN ('pending', 'active')
  AND v.traits @> '{"plan": "enterprise"}'
ORDER BY c.updated_at DESC;
```

---

## Row-Level Security

```sql
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE visitors ENABLE ROW LEVEL SECURITY;
ALTER TABLE agents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON conversations
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON visitors
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON agents
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core | 3 | tenants, agents, teams |
| Visitors | 1 | visitors (traits, browser_info, consent, crm_data all JSONB) |
| Channels | 1 | channels (channel_config + widget_config JSONB) |
| Conversations | 2 | conversations (ai_context, sla_status, routing_metadata JSONB), messages (attachments JSONB) |
| Automation | 1 | automation_rules (conditions + actions JSONB) |
| Integrations | 1 | integrations (credentials, config, sync_status JSONB) |
| Canned Responses | 1 | canned_responses |
| Audit | 1 | audit_log |
| API | 1 | api_keys |
| **Total** | **12** | |

---

## Key Design Decisions

1. **Visitor as a single rich entity** — All visitor data (traits, browser info, current page, consent, CRM links) lives in one row with JSONB columns. No separate visitor_properties, page_views, or CRM mapping tables. Trade-off: loses browsing history (only current page tracked) but dramatically simplifies reads.

2. **Channel + widget config merged** — One `channels` table with `channel_config` and `widget_config` JSONB columns replaces separate channels, chat_widgets, and per-channel-type tables. Adding WhatsApp or Telegram is a new row, not a new table.

3. **AI context inline on conversations** — Intent, sentiment, confidence, and suggested articles are a JSONB column on the conversation, not a separate table. GIN index enables filtering ("show all frustrated visitors").

4. **CRM data on visitor, not separate tables** — `visitors.crm_data` stores Salesforce/HubSpot IDs and synced fields inline. Simpler reads but no separate sync state tracking — sync status is on the `integrations` table instead.

5. **Unified integrations table** — All third-party connections (CRM, Slack, webhooks) use a single `integrations` table with type-specific config JSONB. No per-integration-type tables.

6. **Conversation counter caches** — `message_count`, `last_message_preview`, `last_message_at` are denormalized on the conversation for fast inbox rendering without counting/joining messages.

7. **12 tables total** — Less than half the normalized model. Every eliminated table is a JSONB column on an existing table, trading referential integrity for schema flexibility and query simplicity.
