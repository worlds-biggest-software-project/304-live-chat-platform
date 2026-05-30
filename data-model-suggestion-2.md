# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Live Chat Platform · Created: 2026-05-24

## Philosophy

Live chat is inherently event-driven: a visitor connects, types, sends a message, an agent responds, the conversation is routed, an AI suggests a reply, the visitor disconnects. This model captures every interaction as an immutable event in a single append-only store. The event stream is the source of truth; conversation state, agent dashboards, analytics, and CRM sync are all derived by projecting events into materialised read models.

This approach is particularly natural for real-time messaging because the WebSocket message flow already produces a stream of discrete events. Instead of translating that stream into UPDATE statements on a mutable `conversations` table, each event is appended as-is. Temporal queries ("what was the agent doing when the visitor dropped off?"), session replay, and AI training data all come directly from the event stream without separate audit tables.

GDPR compliance is simpler: instead of cascading DELETE across multiple tables, erasure anonymizes event payloads while preserving the event structure for audit.

**Best for:** Teams prioritizing complete interaction history, real-time analytics, AI training on conversation patterns, and regulatory compliance where every action must be auditable.

**Trade-offs:**
- **Pro:** Complete, immutable record of every interaction — no audit log drift
- **Pro:** Event stream is native AI/ML training data
- **Pro:** Temporal queries ("what happened at time T?") are trivial
- **Pro:** Natural fit for WebSocket message streams
- **Con:** Read model eventual consistency — slight lag for dashboards
- **Con:** Event versioning needed as schema evolves
- **Con:** More complex application code (event handlers + projections)
- **Con:** High-throughput typing indicators create event volume pressure

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WebSocket (RFC 6455) | WebSocket frames map directly to events in the event store |
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (type, source, subject, time) |
| GDPR | Events anonymized (not deleted) on erasure — audit structure preserved |
| ISO/IEC 27001 | Event store IS the audit log — tamper-evident by design |
| CCPA | Erasure requests processed by anonymizing visitor PII in events |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    stream_type TEXT NOT NULL CHECK (stream_type IN ('conversation', 'visitor', 'agent', 'channel')),
    stream_id UUID NOT NULL,
    sequence_number BIGINT NOT NULL,
    event_type TEXT NOT NULL,
    ce_source TEXT NOT NULL DEFAULT '/live-chat',
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"actor_type": "agent", "actor_id": "uuid", "ip_address": "...", "ws_connection_id": "..."}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_events_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, occurred_at DESC);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at DESC);
```

### Event Type Registry

```sql
CREATE TABLE event_type_registry (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO event_type_registry (event_type, stream_type, description) VALUES
-- Conversation lifecycle
('conversation.started',       'conversation', 'New conversation initiated by visitor'),
('conversation.assigned',      'conversation', 'Conversation assigned to agent or team'),
('conversation.transferred',   'conversation', 'Conversation transferred to another agent/team'),
('conversation.snoozed',       'conversation', 'Conversation snoozed until a future time'),
('conversation.resolved',      'conversation', 'Conversation marked resolved'),
('conversation.closed',        'conversation', 'Conversation closed'),
('conversation.reopened',      'conversation', 'Conversation reopened by visitor or agent'),
('conversation.rated',         'conversation', 'Visitor rated the conversation'),
('conversation.tagged',        'conversation', 'Tag added to conversation'),
-- Messages
('message.sent',               'conversation', 'Message sent by visitor, agent, or bot'),
('message.delivered',          'conversation', 'Message delivered to recipient'),
('message.read',               'conversation', 'Message read by recipient'),
('message.attachment_added',   'conversation', 'File attached to message'),
-- Typing & presence
('visitor.typing_started',     'conversation', 'Visitor started typing'),
('visitor.typing_stopped',     'conversation', 'Visitor stopped typing'),
('agent.typing_started',       'conversation', 'Agent started typing'),
-- Visitor
('visitor.identified',         'visitor',      'Anonymous visitor identified with user ID'),
('visitor.page_viewed',        'visitor',      'Visitor viewed a page'),
('visitor.properties_updated', 'visitor',      'Visitor properties changed'),
('visitor.connected',          'visitor',      'Visitor connected via WebSocket'),
('visitor.disconnected',       'visitor',      'Visitor disconnected'),
-- Agent
('agent.status_changed',       'agent',        'Agent changed online status'),
('agent.connected',            'agent',        'Agent connected via WebSocket'),
('agent.disconnected',         'agent',        'Agent disconnected'),
-- AI
('ai.intent_detected',         'conversation', 'AI classified visitor intent'),
('ai.response_suggested',      'conversation', 'AI suggested a response to agent'),
('ai.auto_responded',          'conversation', 'AI sent autonomous response'),
('ai.handoff_triggered',       'conversation', 'AI handed off to human agent'),
('ai.sentiment_detected',      'conversation', 'AI detected sentiment change'),
-- GDPR
('visitor.erased',             'visitor',      'Visitor PII erased (GDPR)');
```

---

## Snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    sequence_number BIGINT NOT NULL,
    snapshot_data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_sequence BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status TEXT NOT NULL DEFAULT 'running' CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message TEXT
);
```

---

## Materialised Read Models

### Conversation Read Model

```sql
CREATE TABLE rm_conversations (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    conversation_number BIGINT NOT NULL,
    visitor_id UUID NOT NULL,
    channel TEXT NOT NULL,
    assignee_id UUID,
    team_id UUID,
    status TEXT NOT NULL,
    priority TEXT NOT NULL DEFAULT 'normal',
    subject TEXT,
    last_message_text TEXT,
    last_message_at TIMESTAMPTZ,
    last_message_sender_type TEXT,
    message_count INT NOT NULL DEFAULT 0,
    unread_count INT NOT NULL DEFAULT 0,
    tags TEXT[] NOT NULL DEFAULT '{}',
    ai_intent TEXT,
    ai_sentiment TEXT,
    first_response_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    rating SMALLINT,
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_conv_tenant_status ON rm_conversations(tenant_id, status);
CREATE INDEX idx_rm_conv_assignee ON rm_conversations(assignee_id);
CREATE INDEX idx_rm_conv_visitor ON rm_conversations(visitor_id);
CREATE INDEX idx_rm_conv_updated ON rm_conversations(tenant_id, updated_at DESC);
```

### Message Read Model

```sql
CREATE TABLE rm_messages (
    id UUID PRIMARY KEY,
    conversation_id UUID NOT NULL,
    sender_type TEXT NOT NULL,
    sender_id UUID,
    message_type TEXT NOT NULL,
    content_text TEXT,
    content_html TEXT,
    content_data JSONB,
    is_private BOOLEAN NOT NULL DEFAULT FALSE,
    delivered_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    attachments JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_messages_conv ON rm_messages(conversation_id, created_at);
```

### Agent Status Read Model

```sql
CREATE TABLE rm_agent_status (
    agent_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'offline',
    active_conversations INT NOT NULL DEFAULT 0,
    max_concurrent INT NOT NULL DEFAULT 5,
    last_activity_at TIMESTAMPTZ,
    connected_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_agent_status_available ON rm_agent_status(tenant_id, status)
    WHERE status = 'online' AND active_conversations < max_concurrent;
```

### Visitor Read Model

```sql
CREATE TABLE rm_visitors (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    anonymous_id TEXT,
    identified_user_id TEXT,
    email TEXT,
    name TEXT,
    properties JSONB NOT NULL DEFAULT '{}',
    current_page_url TEXT,
    current_page_title TEXT,
    is_online BOOLEAN NOT NULL DEFAULT FALSE,
    total_conversations INT NOT NULL DEFAULT 0,
    total_page_views INT NOT NULL DEFAULT 0,
    first_seen_at TIMESTAMPTZ,
    last_seen_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_visitors_tenant ON rm_visitors(tenant_id);
CREATE INDEX idx_rm_visitors_online ON rm_visitors(tenant_id, is_online) WHERE is_online;
```

### Analytics Read Model

```sql
CREATE TABLE rm_conversation_metrics (
    tenant_id UUID NOT NULL,
    period_date DATE NOT NULL,
    period_hour SMALLINT NOT NULL CHECK (period_hour BETWEEN 0 AND 23),
    channel TEXT NOT NULL,
    conversations_started INT NOT NULL DEFAULT 0,
    conversations_resolved INT NOT NULL DEFAULT 0,
    messages_sent INT NOT NULL DEFAULT 0,
    avg_first_response_ms BIGINT,
    avg_resolution_ms BIGINT,
    ai_auto_resolved INT NOT NULL DEFAULT 0,
    ai_handoffs INT NOT NULL DEFAULT 0,
    satisfaction_sum INT NOT NULL DEFAULT 0,
    satisfaction_count INT NOT NULL DEFAULT 0,
    PRIMARY KEY (tenant_id, period_date, period_hour, channel)
);
```

---

## Reference Data

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'agent',
    skills TEXT[] NOT NULL DEFAULT '{}',
    max_concurrent_chats INT NOT NULL DEFAULT 5,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    assignment_method TEXT NOT NULL DEFAULT 'round_robin',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    channel_type TEXT NOT NULL,
    name TEXT NOT NULL,
    configuration JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE routing_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    position INT NOT NULL DEFAULT 0,
    conditions JSONB NOT NULL DEFAULT '{}',
    action_type TEXT NOT NULL,
    action_value JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE canned_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    shortcut TEXT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, shortcut)
);
```

---

## Typing Indicator Optimization

```sql
-- Typing events are high-frequency and ephemeral.
-- Store in a separate, unlogged table (not WAL-replicated) for performance.
-- These are NOT part of the durable event store.
CREATE UNLOGGED TABLE typing_indicators (
    conversation_id UUID NOT NULL,
    user_type TEXT NOT NULL CHECK (user_type IN ('visitor', 'agent')),
    user_id UUID NOT NULL,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (conversation_id, user_type, user_id)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store, event_type_registry, stream_snapshots |
| Projections | 1 | projection_checkpoints |
| Read Models | 5 | rm_conversations, rm_messages, rm_agent_status, rm_visitors, rm_conversation_metrics |
| Reference Data | 6 | tenants, agents, teams, channels, routing_rules, canned_responses |
| Ephemeral | 1 | typing_indicators (UNLOGGED) |
| **Total** | **16** | |

---

## Key Design Decisions

1. **WebSocket frames → events** — Each WebSocket message (visitor message, agent reply, typing indicator, presence change) maps to an event type. The application layer writes events; projections derive state. No mutable conversation table is the source of truth.

2. **Typing indicators excluded from event store** — Typing events are high-frequency (~2-5/second per active conversation) and ephemeral. They use an UNLOGGED table for real-time relay without polluting the durable event stream.

3. **Hourly metrics projection** — `rm_conversation_metrics` aggregates per tenant × date × hour × channel, enabling fast dashboard queries without scanning the event store.

4. **Agent availability as projection** — `rm_agent_status` is derived from agent.connected, agent.disconnected, and conversation.assigned events. The `active_conversations < max_concurrent` index enables instant routing queries.

5. **Visitor properties in read model** — `rm_visitors.properties` JSONB is projected from `visitor.properties_updated` events, enabling routing rule evaluation without event replay.

6. **CloudEvents envelope** — Events include `ce_source` and `ce_specversion` fields for future integration with external event consumers (analytics platforms, CRM sync workers).

7. **GDPR via anonymization** — `visitor.erased` events trigger anonymization of all prior events in the visitor's stream. The event structure (timestamps, types) is preserved for audit; PII in `event_data` is stripped.

8. **Message read model separate from events** — `rm_messages` provides the fast read path for conversation history. The event store retains the full history including delivery/read receipts and edits.
