# Data Model Suggestion 4: Time-Series / Analytics-First

> Project: Live Chat Platform · Created: 2026-05-24

## Philosophy

Live chat generates high-frequency, time-stamped data: messages, typing events, page views, presence changes, AI interactions, and delivery receipts. This model treats that data as time-series, using TimescaleDB hypertables for the high-volume event streams and continuous aggregates for real-time analytics dashboards. The operational tables (conversations, agents, visitors) remain relational; the analytics and interaction history is time-series-optimized.

The key insight is that a live chat platform's most expensive queries are analytics: "what's our average first response time this week?", "how many conversations did the AI resolve per hour?", "what's our CSAT trend over the past 90 days?" In a purely relational model, these require scanning millions of rows. With TimescaleDB continuous aggregates, the database pre-computes hourly and daily rollups automatically, making dashboard queries near-instant regardless of total data volume.

Compression policies automatically compress data older than 7 days (10-20x storage reduction), and retention policies drop raw event data after configurable periods while preserving aggregates indefinitely — a natural fit for GDPR retention requirements.

**Best for:** Teams building analytics-heavy chat platforms where dashboard performance, conversation volume metrics, and AI effectiveness tracking are first-class requirements.

**Trade-offs:**
- **Pro:** Dashboard queries are near-instant via pre-computed continuous aggregates
- **Pro:** 10-20x storage compression for historical data
- **Pro:** Retention policies align naturally with GDPR data minimization
- **Pro:** Time-range queries (last hour, last week, last quarter) are optimized
- **Con:** Requires TimescaleDB extension (not vanilla PostgreSQL)
- **Con:** Hypertable constraints (no foreign keys to hypertables, no UNIQUE across chunks)
- **Con:** More complex setup than pure relational
- **Con:** Update-heavy patterns (conversation status changes) don't fit time-series

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WebSocket (RFC 6455) | Message delivery events are time-series records in `chat_events` hypertable |
| GDPR | Compression + retention policies enforce data minimization; aggregates preserved after raw data drops |
| ISO/IEC 27001 | `chat_events` hypertable is an immutable audit trail |
| OpenTelemetry | Event schema aligns with OTel semantic conventions for spans/events |

---

## Operational Tables (Standard Relational)

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
    status TEXT NOT NULL DEFAULT 'offline' CHECK (status IN ('online', 'away', 'busy', 'offline')),
    skills TEXT[] NOT NULL DEFAULT '{}',
    max_concurrent_chats INT NOT NULL DEFAULT 5,
    active_conversation_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE visitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    anonymous_id TEXT,
    identified_user_id TEXT,
    email TEXT,
    name TEXT,
    traits JSONB NOT NULL DEFAULT '{}',
    is_online BOOLEAN NOT NULL DEFAULT FALSE,
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    channel_type TEXT NOT NULL,
    name TEXT NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    widget_config JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    conversation_number BIGINT NOT NULL,
    visitor_id UUID NOT NULL REFERENCES visitors(id),
    channel_id UUID NOT NULL REFERENCES channels(id),
    assignee_id UUID REFERENCES agents(id),
    team_id UUID REFERENCES teams(id),
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'snoozed', 'resolved', 'closed')),
    priority TEXT NOT NULL DEFAULT 'normal',
    tags TEXT[] NOT NULL DEFAULT '{}',
    ai_context JSONB NOT NULL DEFAULT '{}',
    message_count INT NOT NULL DEFAULT 0,
    first_response_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    rating SMALLINT CHECK (rating BETWEEN 1 AND 5),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, conversation_number)
);

CREATE INDEX idx_agents_tenant ON agents(tenant_id);
CREATE INDEX idx_visitors_tenant ON visitors(tenant_id);
CREATE INDEX idx_visitors_email ON visitors(tenant_id, email);
CREATE INDEX idx_conv_tenant_status ON conversations(tenant_id, status);
CREATE INDEX idx_conv_assignee ON conversations(assignee_id);
CREATE INDEX idx_conv_visitor ON conversations(visitor_id);
```

---

## Time-Series: Chat Events (Hypertable)

```sql
CREATE TABLE chat_events (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    conversation_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    -- message_sent, message_delivered, message_read, conversation_started,
    -- conversation_assigned, conversation_resolved, conversation_closed,
    -- conversation_rated, ai_auto_responded, ai_handoff, ai_intent_detected,
    -- visitor_page_viewed, agent_joined, agent_left
    sender_type TEXT,
    sender_id UUID,
    channel TEXT,
    payload JSONB NOT NULL DEFAULT '{}',
    -- message_sent: {"content_text": "...", "message_type": "text", "is_private": false, "attachments": [...]}
    -- conversation_assigned: {"assignee_id": "uuid", "team_id": "uuid", "method": "round_robin"}
    -- ai_intent_detected: {"intent": "billing", "confidence": 0.92, "sentiment": "neutral"}
    -- conversation_rated: {"rating": 5, "comment": "Great help!"}
    -- visitor_page_viewed: {"url": "...", "title": "...", "duration_ms": 45000}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

SELECT create_hypertable('chat_events', 'occurred_at');

CREATE INDEX idx_chat_events_conv ON chat_events(conversation_id, occurred_at DESC);
CREATE INDEX idx_chat_events_tenant ON chat_events(tenant_id, occurred_at DESC);
CREATE INDEX idx_chat_events_type ON chat_events(event_type, occurred_at DESC);
CREATE INDEX idx_chat_events_sender ON chat_events(sender_type, sender_id, occurred_at DESC);
```

---

## Time-Series: Visitor Activity (Hypertable)

```sql
CREATE TABLE visitor_activity (
    tenant_id UUID NOT NULL,
    visitor_id UUID NOT NULL,
    activity_type TEXT NOT NULL CHECK (activity_type IN ('page_view', 'connected', 'disconnected', 'identified', 'properties_updated')),
    page_url TEXT,
    page_title TEXT,
    referrer TEXT,
    metadata JSONB NOT NULL DEFAULT '{}',
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

SELECT create_hypertable('visitor_activity', 'occurred_at');

CREATE INDEX idx_visitor_activity_visitor ON visitor_activity(visitor_id, occurred_at DESC);
CREATE INDEX idx_visitor_activity_tenant ON visitor_activity(tenant_id, occurred_at DESC);
```

---

## Continuous Aggregates

### Hourly Conversation Metrics

```sql
CREATE MATERIALIZED VIEW hourly_conversation_metrics
WITH (timescaledb.continuous) AS
SELECT
    tenant_id,
    time_bucket('1 hour', occurred_at) AS bucket,
    channel,
    COUNT(*) FILTER (WHERE event_type = 'conversation_started') AS conversations_started,
    COUNT(*) FILTER (WHERE event_type = 'conversation_resolved') AS conversations_resolved,
    COUNT(*) FILTER (WHERE event_type = 'conversation_closed') AS conversations_closed,
    COUNT(*) FILTER (WHERE event_type = 'message_sent') AS messages_sent,
    COUNT(*) FILTER (WHERE event_type = 'message_sent' AND sender_type = 'visitor') AS visitor_messages,
    COUNT(*) FILTER (WHERE event_type = 'message_sent' AND sender_type = 'agent') AS agent_messages,
    COUNT(*) FILTER (WHERE event_type = 'message_sent' AND sender_type = 'bot') AS bot_messages,
    COUNT(*) FILTER (WHERE event_type = 'ai_auto_responded') AS ai_auto_resolved,
    COUNT(*) FILTER (WHERE event_type = 'ai_handoff') AS ai_handoffs,
    AVG((payload->>'rating')::INT) FILTER (WHERE event_type = 'conversation_rated') AS avg_rating,
    COUNT(*) FILTER (WHERE event_type = 'conversation_rated') AS rating_count
FROM chat_events
GROUP BY tenant_id, bucket, channel;

SELECT add_continuous_aggregate_policy('hourly_conversation_metrics',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Daily Conversation Metrics

```sql
CREATE MATERIALIZED VIEW daily_conversation_metrics
WITH (timescaledb.continuous) AS
SELECT
    tenant_id,
    time_bucket('1 day', occurred_at) AS bucket,
    channel,
    COUNT(*) FILTER (WHERE event_type = 'conversation_started') AS conversations_started,
    COUNT(*) FILTER (WHERE event_type = 'conversation_resolved') AS conversations_resolved,
    COUNT(*) FILTER (WHERE event_type = 'message_sent') AS messages_sent,
    COUNT(*) FILTER (WHERE event_type = 'ai_auto_responded') AS ai_auto_resolved,
    AVG((payload->>'rating')::INT) FILTER (WHERE event_type = 'conversation_rated') AS avg_rating,
    COUNT(*) FILTER (WHERE event_type = 'conversation_rated') AS rating_count
FROM chat_events
GROUP BY tenant_id, bucket, channel;

SELECT add_continuous_aggregate_policy('daily_conversation_metrics',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

### Hourly Agent Performance

```sql
CREATE MATERIALIZED VIEW hourly_agent_metrics
WITH (timescaledb.continuous) AS
SELECT
    tenant_id,
    sender_id AS agent_id,
    time_bucket('1 hour', occurred_at) AS bucket,
    COUNT(*) FILTER (WHERE event_type = 'message_sent' AND sender_type = 'agent') AS messages_sent,
    COUNT(*) FILTER (WHERE event_type = 'conversation_resolved') AS conversations_resolved,
    COUNT(*) FILTER (WHERE event_type = 'conversation_assigned') AS conversations_assigned,
    AVG((payload->>'rating')::INT) FILTER (WHERE event_type = 'conversation_rated') AS avg_rating
FROM chat_events
WHERE sender_type = 'agent'
GROUP BY tenant_id, sender_id, bucket;

SELECT add_continuous_aggregate_policy('hourly_agent_metrics',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Daily Visitor Engagement

```sql
CREATE MATERIALIZED VIEW daily_visitor_engagement
WITH (timescaledb.continuous) AS
SELECT
    tenant_id,
    time_bucket('1 day', occurred_at) AS bucket,
    COUNT(DISTINCT visitor_id) AS unique_visitors,
    COUNT(*) FILTER (WHERE activity_type = 'page_view') AS page_views,
    COUNT(*) FILTER (WHERE activity_type = 'connected') AS connections,
    COUNT(DISTINCT visitor_id) FILTER (WHERE activity_type = 'identified') AS identified_visitors
FROM visitor_activity
GROUP BY tenant_id, bucket;

SELECT add_continuous_aggregate_policy('daily_visitor_engagement',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

---

## Compression & Retention Policies

```sql
-- Compress chat events older than 7 days (10-20x storage reduction)
ALTER TABLE chat_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, conversation_id',
    timescaledb.compress_orderby = 'occurred_at DESC'
);
SELECT add_compression_policy('chat_events', INTERVAL '7 days');

-- Compress visitor activity older than 7 days
ALTER TABLE visitor_activity SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, visitor_id',
    timescaledb.compress_orderby = 'occurred_at DESC'
);
SELECT add_compression_policy('visitor_activity', INTERVAL '7 days');

-- Drop raw chat events older than 90 days (aggregates preserved)
SELECT add_retention_policy('chat_events', INTERVAL '90 days');

-- Drop raw visitor activity older than 30 days
SELECT add_retention_policy('visitor_activity', INTERVAL '30 days');
```

---

## Supporting Tables

```sql
CREATE TABLE routing_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    position INT NOT NULL DEFAULT 0,
    conditions JSONB NOT NULL DEFAULT '{}',
    actions JSONB NOT NULL DEFAULT '[]',
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

CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    integration_type TEXT NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

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

## Example Dashboard Queries

### Last 7 days conversation volume by channel

```sql
SELECT bucket::DATE AS day, channel,
       conversations_started, conversations_resolved, ai_auto_resolved,
       avg_rating
FROM daily_conversation_metrics
WHERE tenant_id = 'tenant-uuid'
  AND bucket >= now() - INTERVAL '7 days'
ORDER BY bucket DESC, channel;
```

### Real-time: conversations in the last hour

```sql
SELECT bucket, conversations_started, messages_sent, ai_auto_resolved
FROM hourly_conversation_metrics
WHERE tenant_id = 'tenant-uuid'
  AND bucket >= now() - INTERVAL '1 hour'
  AND channel = 'web_chat';
```

### Agent leaderboard (last 30 days)

```sql
SELECT a.name,
       SUM(m.conversations_resolved) AS total_resolved,
       SUM(m.messages_sent) AS total_messages,
       AVG(m.avg_rating) AS avg_rating
FROM hourly_agent_metrics m
JOIN agents a ON a.id = m.agent_id
WHERE m.tenant_id = 'tenant-uuid'
  AND m.bucket >= now() - INTERVAL '30 days'
GROUP BY a.name
ORDER BY total_resolved DESC;
```

### Reconstruct conversation history from events

```sql
SELECT event_type, sender_type,
       payload->>'content_text' AS message,
       occurred_at
FROM chat_events
WHERE conversation_id = 'conv-uuid'
  AND event_type IN ('message_sent', 'conversation_assigned', 'ai_intent_detected', 'conversation_resolved')
ORDER BY occurred_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Operational | 6 | tenants, agents, teams, visitors, channels, conversations |
| Time-Series (Hypertables) | 2 | chat_events, visitor_activity |
| Continuous Aggregates | 4 | hourly_conversation_metrics, daily_conversation_metrics, hourly_agent_metrics, daily_visitor_engagement |
| Supporting | 4 | routing_rules, canned_responses, integrations, api_keys |
| **Total** | **12 tables + 4 continuous aggregates** | |

---

## Key Design Decisions

1. **Messages as events, not a separate table** — Messages are `message_sent` events in the `chat_events` hypertable, not a separate `messages` table. The conversation history is reconstructed by filtering events. This eliminates dual-write and makes the event stream the single source of truth for conversation content.

2. **Continuous aggregates for dashboards** — Dashboard queries hit pre-computed hourly/daily aggregates, not raw event scans. A "last 30 days" CSAT query reads 30 rows from `daily_conversation_metrics` instead of scanning millions of events.

3. **Compression by conversation** — `compress_segmentby = 'tenant_id, conversation_id'` ensures conversation-level queries remain fast even on compressed data, since all events for a conversation are co-located in the same compressed segment.

4. **Retention with aggregate preservation** — Raw events are dropped after 90 days, but continuous aggregates are preserved indefinitely. This satisfies GDPR data minimization while retaining long-term trend data.

5. **Separate visitor activity hypertable** — Page views and presence events are high-volume but don't need conversation context. Separating them prevents the chat_events hypertable from being overwhelmed by browsing data.

6. **Conversations remain relational** — The `conversations` table is a standard relational table for operational state (status, assignee, priority). It's updated by the application when processing events, but the events are the authoritative history.

7. **No foreign keys to hypertables** — TimescaleDB hypertables don't support incoming foreign keys. `chat_events.conversation_id` references conversations conceptually but not via FK constraint. Application-level integrity enforcement.

8. **Event payload as JSONB** — Each event type has a different payload structure stored in JSONB. The `event_type` column determines how to interpret the payload. This avoids creating separate tables for each event type while keeping the schema uniform.
