# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Live Chat Platform · Created: 2026-05-24

## Philosophy

This model gives every domain concept its own table with typed columns and explicit foreign keys. Conversations, messages, visitors, agents, teams, channels, routing rules, CRM sync records, and AI interactions each have dedicated tables with full referential integrity. The schema mirrors the entity structure exposed by platforms like Intercom, Zendesk Messaging, and LiveChat in their REST APIs.

The normalized approach enables complex cross-entity queries that a real-time chat platform demands: "show me all active conversations for enterprise visitors where the AI confidence is below threshold, grouped by assigned team, with SLA status" is a single SQL query. Every relationship — visitor-to-conversation, conversation-to-messages, agent-to-team, routing-rule-to-assignment — is explicit and enforced at the database level.

**Best for:** Teams building a full-featured chat platform with complex routing, SLA enforcement, CRM integration, and compliance requirements where data integrity is paramount.

**Trade-offs:**
- **Pro:** Full referential integrity across all entities
- **Pro:** Clean reporting and analytics queries via standard JOINs
- **Pro:** Per-channel tables enable channel-specific indexing and constraints
- **Pro:** CRM sync tracking with explicit foreign keys to conversations
- **Con:** 28+ tables increases schema complexity
- **Con:** Adding new channels or message types may require new tables
- **Con:** More JOINs for common read paths (conversation + visitor + messages)
- **Con:** Schema migrations required for structural changes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WebSocket (RFC 6455) | `websocket_connections` table tracks active connections for presence and delivery |
| GDPR | `consent_records`, `erasure_requests` tables for privacy compliance |
| CCPA | Consent tracking supports opt-out requirements |
| ISO/IEC 27001 | `audit_log` table, encryption flags on messages, access control via RLS |
| WCAG 2.1 | Widget accessibility settings stored in `chat_widgets` configuration |
| OAuth 2.0 / RFC 6749 | `oauth_applications` table for third-party integrations |
| OpenAPI 3.1 | API resource structure mirrors table entities 1:1 |

---

## Multi-Tenancy & Agents

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free', 'starter', 'professional', 'enterprise')),
    settings JSONB NOT NULL DEFAULT '{}',
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
    max_concurrent_chats INT NOT NULL DEFAULT 5,
    avatar_url TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    description TEXT,
    auto_assign BOOLEAN NOT NULL DEFAULT TRUE,
    assignment_method TEXT NOT NULL DEFAULT 'round_robin' CHECK (assignment_method IN ('round_robin', 'least_busy', 'manual')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE team_memberships (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    agent_id UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('lead', 'member')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, agent_id)
);

CREATE TABLE agent_skills (
    agent_id UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    skill TEXT NOT NULL,
    proficiency TEXT NOT NULL DEFAULT 'intermediate' CHECK (proficiency IN ('beginner', 'intermediate', 'expert')),
    PRIMARY KEY (agent_id, skill)
);

CREATE INDEX idx_agents_tenant ON agents(tenant_id);
CREATE INDEX idx_agents_status ON agents(tenant_id, status);
```

---

## Visitors & Contacts

```sql
CREATE TABLE visitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    anonymous_id TEXT,
    identified_user_id TEXT,
    email TEXT,
    name TEXT,
    phone TEXT,
    avatar_url TEXT,
    locale TEXT DEFAULT 'en',
    timezone TEXT,
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    total_conversations INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE visitor_properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    visitor_id UUID NOT NULL REFERENCES visitors(id) ON DELETE CASCADE,
    key TEXT NOT NULL,
    value TEXT,
    source TEXT NOT NULL DEFAULT 'sdk' CHECK (source IN ('sdk', 'crm', 'agent', 'ai')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (visitor_id, key)
);

CREATE TABLE visitor_page_views (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    visitor_id UUID NOT NULL REFERENCES visitors(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    title TEXT,
    referrer TEXT,
    duration_seconds INT,
    viewed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_visitors_tenant ON visitors(tenant_id);
CREATE INDEX idx_visitors_email ON visitors(tenant_id, email);
CREATE INDEX idx_visitors_anonymous ON visitors(tenant_id, anonymous_id);
CREATE INDEX idx_visitors_identified ON visitors(tenant_id, identified_user_id);
CREATE INDEX idx_visitor_page_views_visitor ON visitor_page_views(visitor_id, viewed_at DESC);
```

---

## Channels & Widgets

```sql
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    channel_type TEXT NOT NULL CHECK (channel_type IN ('web_chat', 'email', 'sms', 'whatsapp', 'facebook_messenger', 'twitter_dm', 'telegram', 'slack', 'api')),
    name TEXT NOT NULL,
    configuration JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chat_widgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    channel_id UUID NOT NULL REFERENCES channels(id),
    name TEXT NOT NULL DEFAULT 'Default Widget',
    domain_whitelist TEXT[] NOT NULL DEFAULT '{}',
    appearance JSONB NOT NULL DEFAULT '{}',
    -- {"primary_color": "#4A90D9", "position": "bottom-right", "greeting": "Hi! How can we help?", "logo_url": "..."}
    pre_chat_form JSONB NOT NULL DEFAULT '{}',
    -- {"enabled": true, "fields": [{"name": "email", "type": "email", "required": true}, {"name": "department", "type": "dropdown", "options": [...]}]}
    business_hours JSONB NOT NULL DEFAULT '{}',
    -- {"timezone": "America/New_York", "schedule": {"mon": {"start": "09:00", "end": "17:00"}, ...}}
    offline_message TEXT DEFAULT 'We are currently offline. Please leave a message.',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channels_tenant ON channels(tenant_id);
CREATE INDEX idx_chat_widgets_tenant ON chat_widgets(tenant_id);
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
    subject TEXT,
    source_url TEXT,
    first_response_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    rating SMALLINT CHECK (rating BETWEEN 1 AND 5),
    rating_comment TEXT,
    message_count INT NOT NULL DEFAULT 0,
    unread_agent_count INT NOT NULL DEFAULT 0,
    unread_visitor_count INT NOT NULL DEFAULT 0,
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
    -- for cards: {"title": "...", "description": "...", "image_url": "...", "buttons": [...]}
    -- for quick_reply: {"options": [{"label": "Yes", "value": "yes"}, ...]}
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    delivered_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE message_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    file_name TEXT NOT NULL,
    content_type TEXT NOT NULL,
    size_bytes BIGINT NOT NULL,
    storage_key TEXT NOT NULL,
    thumbnail_key TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversation_tags (
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    tag TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (conversation_id, tag)
);

CREATE INDEX idx_conversations_tenant_status ON conversations(tenant_id, status);
CREATE INDEX idx_conversations_assignee ON conversations(assignee_id) WHERE assignee_id IS NOT NULL;
CREATE INDEX idx_conversations_team ON conversations(team_id) WHERE team_id IS NOT NULL;
CREATE INDEX idx_conversations_visitor ON conversations(visitor_id);
CREATE INDEX idx_conversations_created ON conversations(tenant_id, created_at DESC);
CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at);
CREATE INDEX idx_messages_sender ON messages(sender_type, sender_id);
CREATE INDEX idx_conversation_tags_tag ON conversation_tags(tag);
```

---

## Presence & WebSocket Connections

```sql
CREATE TABLE websocket_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    user_type TEXT NOT NULL CHECK (user_type IN ('agent', 'visitor')),
    user_id UUID NOT NULL,
    connection_id TEXT NOT NULL UNIQUE,
    server_id TEXT NOT NULL,
    connected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_ping_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ws_connections_user ON websocket_connections(user_type, user_id);
CREATE INDEX idx_ws_connections_server ON websocket_connections(server_id);
CREATE INDEX idx_ws_connections_stale ON websocket_connections(last_ping_at);
```

---

## Routing Rules

```sql
CREATE TABLE routing_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    position INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    conditions JSONB NOT NULL DEFAULT '{}',
    -- {"all": [{"field": "visitor.properties.plan", "op": "is", "value": "enterprise"}, {"field": "channel", "op": "is", "value": "web_chat"}]}
    action_type TEXT NOT NULL CHECK (action_type IN ('assign_team', 'assign_agent', 'assign_bot', 'set_priority')),
    action_value JSONB NOT NULL DEFAULT '{}',
    -- {"team_id": "uuid"} or {"agent_id": "uuid"} or {"bot_id": "uuid"} or {"priority": "high"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routing_rules_tenant ON routing_rules(tenant_id, is_active, position);
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
    scope TEXT NOT NULL DEFAULT 'team' CHECK (scope IN ('personal', 'team', 'global')),
    owner_id UUID REFERENCES agents(id),
    team_id UUID REFERENCES teams(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, shortcut)
);
```

---

## AI & Bot

```sql
CREATE TABLE bot_configurations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    bot_type TEXT NOT NULL CHECK (bot_type IN ('faq', 'flow', 'ai_agent')),
    configuration JSONB NOT NULL DEFAULT '{}',
    -- {"model": "gpt-4", "temperature": 0.3, "knowledge_base_ids": [...], "handoff_threshold": 0.7}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    message_id UUID REFERENCES messages(id),
    interaction_type TEXT NOT NULL CHECK (interaction_type IN ('auto_response', 'suggested_reply', 'intent_detection', 'sentiment_analysis', 'handoff_decision')),
    input_text TEXT,
    output_text TEXT,
    confidence REAL CHECK (confidence BETWEEN 0 AND 1),
    model_version TEXT NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"intent": "billing_question", "sentiment": "frustrated", "suggested_articles": [...], "handoff_reason": "low_confidence"}
    was_accepted BOOLEAN,
    feedback TEXT CHECK (feedback IN ('helpful', 'not_helpful', 'partially_helpful')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_interactions_conversation ON ai_interactions(conversation_id);
CREATE INDEX idx_ai_interactions_type ON ai_interactions(interaction_type, created_at DESC);
```

---

## CRM Sync

```sql
CREATE TABLE crm_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    crm_type TEXT NOT NULL CHECK (crm_type IN ('salesforce', 'hubspot', 'pipedrive', 'zoho')),
    credentials_encrypted TEXT NOT NULL,
    sync_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    last_sync_at TIMESTAMPTZ,
    sync_status TEXT NOT NULL DEFAULT 'idle' CHECK (sync_status IN ('idle', 'syncing', 'error')),
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, crm_type)
);

CREATE TABLE crm_contact_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    visitor_id UUID NOT NULL REFERENCES visitors(id) ON DELETE CASCADE,
    crm_contact_id TEXT NOT NULL,
    crm_contact_url TEXT,
    last_synced_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (crm_connection_id, visitor_id)
);

CREATE TABLE crm_conversation_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    crm_activity_id TEXT NOT NULL,
    crm_activity_type TEXT NOT NULL,
    synced_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (crm_connection_id, conversation_id)
);
```

---

## Audit & Compliance

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    action TEXT NOT NULL,
    actor_type TEXT NOT NULL CHECK (actor_type IN ('agent', 'visitor', 'system', 'bot', 'api')),
    actor_id UUID,
    changes JSONB NOT NULL DEFAULT '{}',
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);

CREATE TABLE consent_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    visitor_id UUID NOT NULL REFERENCES visitors(id),
    consent_type TEXT NOT NULL CHECK (consent_type IN ('tracking', 'marketing', 'data_processing')),
    granted BOOLEAN NOT NULL,
    ip_address INET,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at TIMESTAMPTZ
);

CREATE TABLE erasure_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    visitor_id UUID NOT NULL REFERENCES visitors(id),
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed')),
    requested_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    entities_erased JSONB
);
```

---

## API & Webhooks

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_by UUID NOT NULL REFERENCES agents(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    url TEXT NOT NULL,
    events TEXT[] NOT NULL,
    secret_hash TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    failure_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
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
| Multi-Tenancy & Auth | 5 | tenants, agents, teams, team_memberships, agent_skills |
| Visitors | 3 | visitors, visitor_properties, visitor_page_views |
| Channels & Widgets | 2 | channels, chat_widgets |
| Conversations | 4 | conversations, messages, message_attachments, conversation_tags |
| Presence | 1 | websocket_connections |
| Routing | 1 | routing_rules |
| Canned Responses | 1 | canned_responses |
| AI & Bot | 2 | bot_configurations, ai_interactions |
| CRM Sync | 3 | crm_connections, crm_contact_mappings, crm_conversation_logs |
| Audit & Compliance | 3 | audit_log, consent_records, erasure_requests |
| Integrations | 2 | api_keys, webhooks |
| **Total** | **27** | |

---

## Key Design Decisions

1. **Conversation-centric model** — Conversations (not tickets) are the primary entity, reflecting the conversational nature of live chat. Each conversation belongs to a single channel and visitor, with messages as the child entity.

2. **Separate presence tracking** — `websocket_connections` is a hot table tracking active connections. It's designed for frequent updates (ping heartbeats) and queries (who's online?), isolated from the main conversation data.

3. **Visitor properties as EAV** — `visitor_properties` uses key-value pairs rather than JSONB to enable SQL-level filtering and indexing on individual properties for routing rule evaluation.

4. **Channel-specific configuration in JSONB** — While channels get their own table, channel-specific config (SMTP settings, WhatsApp credentials) lives in a `configuration` JSONB column rather than per-channel-type tables.

5. **Rich message types** — Messages support structured content (cards, carousels, quick replies, forms) via `content_data` JSONB alongside plain text. The `message_type` enum controls how the client renders each message.

6. **CRM sync as explicit tables** — Rather than inline JSONB, CRM connections, contact mappings, and conversation logs are separate tables enabling reliable sync state tracking and retry logic.

7. **AI interactions tracked separately** — Every AI action (auto-response, suggestion, intent detection) is a row in `ai_interactions`, enabling feedback loops and model performance tracking without cluttering the message stream.

8. **Page view tracking** — `visitor_page_views` captures browsing behavior for intent scoring and proactive chat triggers. Partitioning by time recommended for high-traffic deployments.
