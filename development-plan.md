# Live Chat Platform — Phased Development Plan

> Project: 304-live-chat-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents into a concrete, buildable specification for an AI-native, open-source live chat platform. The core product is a multi-tenant, WebSocket-based customer conversation system with intent-aware routing, real-time agent assist, autonomous tier-1 AI resolution, and CRM synchronisation.

---

## Core Requirements Summary

- **What it does** — Real-time customer conversations (support/sales/CS) over a WebSocket transport, with a brandable embeddable widget, an agent inbox, intent- and CRM-aware routing, AI assist + autonomous resolution, and bidirectional CRM sync. Self-hostable (data residency) and deployable as managed cloud.
- **Primary personas** — Customer success managers at PLG SaaS companies; inbound sales reps qualifying leads; support managers deflecting tickets; e-commerce operators needing 24/7 coverage. Plus the technical buyer/admin configuring routing and integrations.
- **Key differentiators** — Visitor intent scoring (proactive engagement), confidence-gated autonomous tier-1 resolution with context-preserving handoff, CRM-aware routing (deal stage / health / tier), real-time agent assist, and sentiment/churn-risk detection. Open-source self-hostable, AI-native without Intercom-tier pricing.
- **MVP scope** (features.md "Must-have") — Real-time chat (WebSocket), multi-channel (web/email/SMS), pre-chat forms, canned responses/quick replies, chat history + transcripts, CRM integration (HubSpot/Salesforce), agent app, basic analytics.
- **v1.1 scope** ("Should-have") — AI suggested replies, omnichannel inbox, visitor tracking/identification, proactive chat triggers, brandable widget, routing rules (skill/team), SLA + escalation, REST API.
- **Backlog** ("Nice-to-have") — Autonomous AI resolution (50%+), lead qualification, real-time sentiment escalation, agent skill matching, knowledge-aware routing, meeting booking, multi-language.
- **Deployment** — Hybrid: Docker-composed self-hosted stack + managed cloud. Horizontally scalable WebSocket tier with Redis pub/sub fan-out.
- **Integration surface** — REST API (OpenAPI 3.1), outbound webhooks (HMAC-SHA256), OAuth 2.0 for CRM/marketplace, WebSocket subscriptions, LLM provider APIs.
- **Standards** — WebSocket RFC 6455; OpenAPI 3.1; OAuth 2.0 (RFC 6749); JSON Schema 2020-12; GDPR/CCPA (consent, retention, erasure); ISO/IEC 27001 (encryption in transit/at rest, RBAC, audit log); WCAG 2.1 AA + WAI-ARIA (widget); CloudEvents 1.0 envelope for webhook/event payloads.
- **Data model chosen** — **Suggestion 3 (Hybrid Relational + JSONB)** as the primary operational schema: typed columns for hot query fields (status, assignee, timestamps) with JSONB for variable data (visitor properties, channel config, widget appearance, routing conditions, AI metadata). This minimises migrations during early iteration while preserving indexed queries. Analytics adopt **Suggestion 4's continuous-aggregate pattern** (hourly/daily rollups) in Phase 9. Suggestion 1's explicit CRM-sync / consent / audit tables are folded in where referential integrity matters (compliance).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | The product is API/realtime/frontend-heavy, not ML-training-heavy. One language across the WebSocket gateway, REST API, widget, and agent SPA reduces context-switching and lets message/event types be shared. LLM work is API calls, not local model training. |
| Runtime/monorepo | pnpm workspaces + Turborepo | Multiple deployables (api, ws-gateway, widget, agent-app, worker) share `packages/shared` (types, schemas, event contracts). pnpm gives fast, disk-efficient installs; Turborepo caches builds/tests. |
| HTTP/API framework | Fastify 5 | Highest-throughput mature Node HTTP framework; first-class JSON Schema validation (aligns with JSON Schema 2020-12) and `@fastify/swagger` generates OpenAPI 3.1 directly from route schemas. |
| WebSocket transport | `ws` + custom protocol over RFC 6455, fronted by Redis pub/sub | Raw `ws` keeps full control of the wire protocol and back-pressure. Redis pub/sub fans messages across horizontally-scaled gateway nodes so any node can deliver to any connection. |
| Database | PostgreSQL 16 | The hybrid model needs relational integrity + GIN-indexed JSONB; Postgres does both natively. Row-Level Security enforces tenant isolation. TimescaleDB extension added in Phase 9 for analytics continuous aggregates (Suggestion 4). |
| ORM / query layer | Drizzle ORM | Type-safe SQL-first schema in TypeScript, generates migrations, supports JSONB and partial indexes, and keeps generated types in `packages/shared`. Lighter and more transparent than Prisma for a JSONB-heavy schema. |
| Cache / pub-sub / presence | Redis 7 | Doubles as WebSocket fan-out bus, presence store (`websocket_connections` hot data), typing-indicator relay (ephemeral), rate-limit counters, and BullMQ backing store. |
| Task queue | BullMQ (Redis-backed) | Async workloads: webhook delivery + retry, CRM sync, AI calls that must not block the WS loop, analytics rollups, GDPR erasure jobs. |
| Object storage | S3-compatible (MinIO self-hosted / S3 cloud) | Message attachments. Presigned upload/download keeps large payloads off the API. |
| LLM access | Vercel AI SDK (provider-agnostic) | Unified interface over OpenAI/Anthropic/local models with streaming + tool calling; provider failover for self-hosted users who bring their own keys/models. Used for intent, suggested replies, autonomous resolution, sentiment. |
| Frontend — widget | Preact + Vite, shipped as a single embeddable `<script>` | Preact keeps the embedded bundle tiny (<40KB gz) to avoid hurting host-page performance. Built as a web component / shadow-DOM island so host CSS can't leak in. WCAG 2.1 AA + WAI-ARIA from the start. |
| Frontend — agent app | React 18 + Vite + TanStack Query + Tailwind + shadcn/ui | Rich dashboard SPA (inbox, conversation view, presence, analytics). TanStack Query for REST cache; a thin WS client subscribes to live events. |
| Mobile agent app | React Native (Expo) — Phase 10 | Shares the same REST/WS client and shared types; Expo accelerates iOS/Android delivery. |
| Auth | Lucia-style session auth for agents + JWT for widget/visitor + OAuth 2.0 for CRM/marketplace | Agents use httpOnly session cookies; visitors get short-lived signed JWTs scoped to a conversation; third-party integrations use OAuth 2.0 (RFC 6749). API access uses hashed API keys with scopes. |
| Validation / schemas | Zod (+ `zod-to-json-schema`) | Single source of truth for request/event validation; emits JSON Schema for OpenAPI and for the public webhook contract. |
| Testing | Vitest (unit/integration) + Playwright (widget & agent E2E) + Testcontainers (real Postgres/Redis) | Vitest is fast and Vite-native; Playwright drives the embedded widget and agent SPA incl. accessibility checks; Testcontainers spins up real Postgres/Redis for integration tests. |
| Code quality | ESLint (typescript-eslint) + Prettier + `tsc --noEmit` | Standard, CI-enforced. |
| Containerisation | Docker + docker-compose (self-host) ; Helm chart (Phase 11) | One-command self-hosted stack; Helm for cluster deployments. |
| CI | GitHub Actions | Lint, typecheck, unit + integration (Testcontainers), build, Docker image publish. |
| Observability | OpenTelemetry + Pino structured logs | Traces across api → ws-gateway → worker; Pino JSON logs. Metrics feed dashboards and SLA monitors. |

### Project Structure

```
live-chat-platform/
├── package.json                # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml          # postgres, redis, minio, api, ws-gateway, worker, agent-app
├── Dockerfile.api
├── Dockerfile.ws-gateway
├── Dockerfile.worker
├── .github/workflows/ci.yml
├── packages/
│   ├── shared/                 # types, zod schemas, event contracts, WS protocol
│   │   ├── src/
│   │   │   ├── events.ts        # CloudEvents-style domain events + type registry
│   │   │   ├── ws-protocol.ts   # client<->server WS frame schemas
│   │   │   ├── schemas/         # zod schemas (message, conversation, visitor, routing…)
│   │   │   └── index.ts
│   ├── db/                      # drizzle schema, migrations, RLS policies, seed
│   │   ├── src/schema/          # tenants, agents, conversations, messages, …
│   │   ├── drizzle/             # generated SQL migrations
│   │   └── src/client.ts
│   └── ai/                      # LLM client, prompt templates, intent/sentiment/assist
│       ├── src/prompts/
│       └── src/index.ts
├── apps/
│   ├── api/                    # Fastify REST API + OpenAPI + webhooks emit + auth
│   │   ├── src/routes/
│   │   ├── src/services/        # routing engine, SLA, CRM sync orchestration
│   │   ├── src/plugins/         # auth, rbac, rate-limit, tenant-context (RLS)
│   │   └── src/server.ts
│   ├── ws-gateway/             # raw `ws` server, presence, Redis fan-out, typing relay
│   │   └── src/
│   ├── worker/                 # BullMQ processors: webhooks, crm-sync, ai, analytics, gdpr
│   │   └── src/processors/
│   ├── widget/                 # Preact embeddable widget (shadow DOM, a11y)
│   │   └── src/
│   ├── agent-app/              # React agent inbox SPA
│   │   └── src/
│   └── mobile/                 # React Native (Expo) — Phase 10
├── tests/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/               # sample conversations, webhooks, CRM payloads
└── docs/
    └── openapi.json            # generated
```

---

## Phase 1: Foundation, Schema & Tenant Context

### Purpose
Establish the monorepo, the hybrid database schema with tenant isolation, the shared type/schema package, and a running (but empty) API and WS gateway. Everything downstream depends on the data model, tenant-scoping, and shared contracts existing first. After this phase the stack boots via `docker-compose up`, migrations apply, and a health endpoint returns 200.

### Tasks

#### 1.1 — Monorepo & toolchain
**What**: pnpm + Turborepo workspace with the package/app layout above, shared ESLint/Prettier/tsconfig, and CI skeleton.

**Design**:
- Root `pnpm-workspace.yaml` includes `packages/*` and `apps/*`.
- `turbo.json` pipelines: `build`, `lint`, `typecheck`, `test` (with `dependsOn: ["^build"]`).
- Base `tsconfig.base.json` with `"strict": true`, `"moduleResolution": "bundler"`, path aliases (`@lcp/shared`, `@lcp/db`, `@lcp/ai`).
- `.github/workflows/ci.yml`: matrix job running `pnpm lint`, `pnpm typecheck`, `pnpm test`, `pnpm build`. Postgres + Redis service containers for integration tests.

**Testing**:
- `Unit: pnpm install && turbo build` succeeds with empty packages.
- `CI: workflow runs green on a no-op commit.`

#### 1.2 — Hybrid database schema (Drizzle)
**What**: Implement the Suggestion-3 hybrid schema as Drizzle table definitions plus generated SQL migrations.

**Design** (core tables; typed columns + JSONB bags):
```typescript
// packages/db/src/schema/tenants.ts
export const tenants = pgTable('tenants', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  plan: text('plan', { enum: ['free','starter','professional','enterprise'] }).notNull().default('free'),
  settings: jsonb('settings').$type<TenantSettings>().notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// agents: id, tenantId, email, name, role(owner|admin|agent),
//   status(online|away|busy|offline), maxConcurrentChats, skills text[],
//   passwordHash, avatarUrl, timezone  — UNIQUE(tenantId,email)
// teams: id, tenantId, name, assignmentMethod(round_robin|least_busy|manual), config jsonb
// team_memberships(teamId, agentId, role) PK(teamId,agentId)
// visitors: id, tenantId, anonymousId, identifiedUserId, email, name, phone,
//   properties jsonb (CRM + SDK traits), locale, firstSeenAt, lastSeenAt, totalConversations
// channels: id, tenantId, channelType(web_chat|email|sms|whatsapp|...), name, configuration jsonb, isActive
// chat_widgets: id, tenantId, channelId, domainWhitelist text[],
//   appearance jsonb, preChatForm jsonb, businessHours jsonb, offlineMessage, isActive
// conversations: id, tenantId, conversationNumber bigint, visitorId, channelId,
//   assigneeId, teamId, status(pending|active|snoozed|resolved|closed),
//   priority(urgent|high|normal|low), subject, sourceUrl,
//   firstResponseAt, resolvedAt, closedAt, rating smallint, ratingComment,
//   messageCount, unreadAgentCount, unreadVisitorCount,
//   aiMeta jsonb (intent, sentiment, churnRisk, lastConfidence) — UNIQUE(tenantId,conversationNumber)
// messages: id, conversationId, senderType(visitor|agent|bot|system), senderId,
//   messageType(text|image|file|card|carousel|quick_reply|form|system),
//   contentText, contentHtml, contentData jsonb, isPrivate, deliveredAt, readAt, createdAt
// message_attachments: id, messageId, fileName, contentType, sizeBytes, storageKey, thumbnailKey
// conversation_tags(conversationId, tag) PK
// routing_rules: id, tenantId, name, position, isActive, conditions jsonb, actionType, actionValue jsonb
// canned_responses: id, tenantId, shortcut, title, content, scope, ownerId, teamId — UNIQUE(tenantId,shortcut)
```
Indexes mirror Suggestion 1/3: `idx_conversations_tenant_status`, partial `idx_conversations_assignee WHERE assignee IS NOT NULL`, `idx_messages_conversation(conversationId, createdAt)`, GIN index on `visitors.properties` and `conversations.aiMeta`.
- `conversation_number` allocated via per-tenant Postgres sequence or `SELECT max+1 FOR UPDATE` inside the create transaction.

**Testing**:
- `Integration (Testcontainers): apply all migrations on fresh Postgres → no errors; pg_dump shows expected tables/indexes.`
- `Unit: inserting a conversation without tenantId → NOT NULL violation.`
- `Integration: GIN containment query visitors.properties @> '{"plan":"enterprise"}' uses the GIN index (EXPLAIN).`

#### 1.3 — Row-Level Security & tenant context
**What**: Enforce tenant isolation at the DB layer plus a request-scoped tenant context.

**Design**:
- Enable RLS on `conversations`, `messages`, `visitors`, `agents`, `channels`, `routing_rules`, `canned_responses`.
- Policy: `USING (tenant_id = current_setting('app.current_tenant_id')::uuid)`.
- Fastify plugin `tenantContext`: resolves tenant from auth (agent session / API key / widget JWT) and runs each request inside a transaction that executes `SET LOCAL app.current_tenant_id = $1`.
- A privileged migration/admin connection bypasses RLS via a dedicated role.

**Testing**:
- `Integration: query conversations with tenant A context → only A's rows; switching to B yields B's rows.`
- `Integration: request with no tenant context → RLS returns zero rows (fail-closed).`

#### 1.4 — Shared contracts package
**What**: `packages/shared` exporting Zod schemas + TypeScript types for messages, conversations, visitors, the WS wire protocol, and the domain-event registry (CloudEvents envelope).

**Design**:
```typescript
// events.ts — CloudEvents 1.0 envelope reused for webhooks & internal bus
export interface DomainEvent<T = unknown> {
  id: string; specversion: '1.0'; source: string;        // e.g. '/live-chat'
  type: string;                                           // 'conversation.started'
  subject: string;                                        // conversationId
  time: string;                                           // ISO-8601
  tenantId: string; data: T;
}
export const EVENT_TYPES = [
  'conversation.started','conversation.assigned','conversation.transferred',
  'conversation.resolved','conversation.closed','conversation.rated',
  'message.sent','message.delivered','message.read',
  'visitor.identified','agent.status_changed',
  'ai.intent_detected','ai.response_suggested','ai.auto_responded','ai.handoff_triggered','ai.sentiment_detected',
] as const;
```
WS protocol frames (Zod-validated, discriminated union on `op`): `auth`, `subscribe`, `message`, `typing`, `presence`, `ack`, `error`, plus server pushes `message.new`, `conversation.updated`, `typing`, `presence`, `agent.assist`.

**Testing**:
- `Unit: valid message frame parses; unknown op → ZodError.`
- `Unit: zod-to-json-schema emits a JSON Schema 2020-12 doc for the public webhook event.`

#### 1.5 — Bootable API + WS gateway skeleton
**What**: Fastify API with `/healthz` and `/openapi.json`; `ws-gateway` accepting connections and echoing; `docker-compose` wiring postgres/redis/minio.

**Design**: `@fastify/swagger` + `@fastify/swagger-ui` produce OpenAPI 3.1 at build/boot. WS gateway upgrades on `/ws`, validates an `auth` frame within 5s or closes (code 4001).

**Testing**:
- `Integration: GET /healthz → 200 {db:"ok",redis:"ok"}.`
- `E2E: docker-compose up → all services healthy; ws connect + auth-timeout closes idle socket.`

---

## Phase 2: Conversations, Messages & REST Core

### Purpose
Build the conversational heart over REST first (deterministic, easy to test) before adding the realtime layer. After this phase, conversations and messages can be created, listed, read, and transcribed entirely through the API, with full tenant scoping and RBAC.

### Tasks

#### 2.1 — Agent auth & RBAC
**What**: Agent signup/login (session cookies), API keys (hashed, scoped), and an RBAC guard.

**Design**:
- `POST /v1/auth/login` → httpOnly secure session cookie; passwords hashed with argon2id.
- API keys: store `key_hash` (SHA-256) + `key_prefix`; scopes array (`conversations:read`, `conversations:write`, `visitors:read`, `webhooks:manage`, …).
- `requireRole(min: 'agent'|'admin'|'owner')` and `requireScope(scope)` Fastify preHandlers.

**Testing**:
- `Unit: argon2 verify round-trip.`
- `Integration: agent role calling admin route → 403; API key missing scope → 403.`
- `Integration: expired/invalid session → 401.`

#### 2.2 — Conversation lifecycle API
**What**: CRUD + state transitions for conversations.

**Design** — endpoints (all tenant-scoped, OpenAPI-documented):
```
POST   /v1/conversations              # create (visitorId, channelId, subject?)
GET    /v1/conversations              # filter: status, assigneeId, teamId, tag, cursor pagination
GET    /v1/conversations/:id
PATCH  /v1/conversations/:id          # assignee, team, priority, status, tags
POST   /v1/conversations/:id/resolve  # sets status=resolved, resolvedAt
POST   /v1/conversations/:id/close
POST   /v1/conversations/:id/reopen
POST   /v1/conversations/:id/rate     # visitor: rating 1-5, comment
```
State machine: `pending → active → (snoozed ↔ active) → resolved → closed`; `closed/resolved → reopened (active)`. Illegal transitions → 409 with `allowedTransitions`. First agent message stamps `firstResponseAt`.

**Testing**:
- `Unit: transition resolved→active allowed (reopen); closed→pending rejected (409).`
- `Integration: create→list filters by status; cursor pagination returns stable order.`
- `Integration: firstResponseAt set only on first agent message.`

#### 2.3 — Messages & attachments
**What**: Send/list messages (incl. private agent notes), structured types, and presigned attachment upload.

**Design**:
```
POST /v1/conversations/:id/messages   # {senderType, messageType, contentText?, contentData?, isPrivate?}
GET  /v1/conversations/:id/messages   # ascending createdAt, cursor pagination
POST /v1/attachments/presign          # {fileName, contentType, sizeBytes} → {uploadUrl, storageKey}
```
On send: increment `messageCount`, set `unreadAgentCount`/`unreadVisitorCount` based on senderType, update `last_message_*` denorm fields. `isPrivate=true` messages are excluded from visitor-scoped reads. Validate `contentData` against the message-type schema (card/quick_reply/form).

**Testing**:
- `Unit: card message missing required "title" → ValidationError.`
- `Integration: private note not returned to visitor JWT; returned to agent.`
- `Integration: presign returns S3 URL; size over tenant limit → 413.`

#### 2.4 — Transcript & history
**What**: Searchable transcript export.

**Design**: `GET /v1/conversations/:id/transcript?format=json|html|txt`. Full-text search `GET /v1/messages/search?q=` via Postgres `tsvector` GIN index on `messages.content_text`.

**Testing**:
- `Integration: transcript html excludes private notes by default; ?includePrivate=true (admin) includes them.`
- `Integration: search "refund" returns matching messages ranked by ts_rank.`

---

## Phase 3: Realtime Layer (WebSocket Gateway)

### Purpose
This is the product's beating heart — sub-second bidirectional delivery (RFC 6455) across horizontally scaled gateway nodes. After this phase, visitors and agents exchange messages live, see typing indicators and presence, and the REST store and WS layer stay consistent.

### Tasks

#### 3.1 — Connection, auth & subscription protocol
**What**: Authenticated WS sessions that subscribe to conversation/agent channels.

**Design**:
- Upgrade `/ws`; first frame must be `{op:"auth", token}` (visitor JWT or agent session token) within 5s.
- On auth, register connection in Redis: `HSET ws:conn:{connId}` + `SADD ws:user:{type}:{userId} {connId}` with TTL refreshed by heartbeat ping/pong (close stale after 60s).
- `{op:"subscribe", channel:"conversation:{id}"}` — authorize against tenant + participation, then `SADD ws:sub:{channel} {connId}`.
- Map to Suggestion-1 `websocket_connections` semantics but store hot data in Redis, not Postgres.

**Testing**:
- `Integration (real Redis): two gateway nodes; message published on node A reaches a subscriber on node B.`
- `Integration: visitor subscribing to a conversation they don't own → error frame 4003.`
- `Integration: missed pong → connection reaped, presence updated.`

#### 3.2 — Message fan-out & ordering
**What**: Deliver new messages to all subscribers with at-least-once delivery and ack.

**Design**:
- Message send path (from REST 2.3 and WS) publishes a `message.new` event to Redis channel `tenant:{tid}:conversation:{cid}`. Each gateway node subscribed relays to local connections.
- Per-connection outbound queue with back-pressure; client sends `{op:"ack", messageId}`; unacked messages resent on reconnect using a `lastSeenMessageId` cursor.
- Ordering guaranteed per conversation by monotonic `created_at` + tiebreak `id`; client sorts on receipt.

**Testing**:
- `Integration: send 100 messages rapidly → client receives all, in order, exactly once after dedup by id.`
- `Integration: client reconnects with lastSeenMessageId → only newer messages replayed.`

#### 3.3 — Typing indicators & presence
**What**: Ephemeral typing relay and online/away presence.

**Design**: Typing frames relayed via Redis pub/sub only (never persisted — Suggestion 2's UNLOGGED rationale, here just Redis with 6s TTL). Presence derived from connection registry; agent `status` changes broadcast `agent.status_changed`.

**Testing**:
- `Integration: typing frame relayed to other participants, not echoed to sender; auto-expires.`
- `Integration: agent disconnect → presence flips offline within heartbeat window.`

---

## Phase 4: Embeddable Widget

### Purpose
The public, customer-facing surface. A tiny, accessible, brandable widget that any website embeds with one script tag and that speaks the WS protocol. After this phase, a real visitor can open a website, fill a pre-chat form, and chat live with an agent.

### Tasks

#### 4.1 — Loader & shadow-DOM island
**What**: One-line embed that injects an isolated widget.

**Design**: `<script src=".../widget.js" data-widget-id="..."></script>` boots a Preact app inside an open shadow root (style isolation). Loader fetches widget config (`GET /v1/widget/:id/config`, public, domain-whitelist checked against `Origin`). Lazy-loads the chat bundle on first open to keep initial payload tiny.

**Testing**:
- `E2E (Playwright): embed on a test page → bubble renders; host-page CSS does not leak into widget.`
- `Integration: config request from non-whitelisted Origin → 403.`

#### 4.2 — Pre-chat form & conversation start
**What**: Render configurable pre-chat form; create visitor + conversation; mint visitor JWT.

**Design**: Form schema from `chat_widgets.preChatForm` JSONB (`fields:[{name,type,required,options}]`). On submit → `POST /v1/widget/:id/start` returns `{visitorToken, conversationId}`; widget opens WS and subscribes.

**Testing**:
- `Unit: required field empty → inline validation error, no submit.`
- `E2E: complete form → conversation created, WS connected, greeting message shown.`

#### 4.3 — Chat UI: messaging, structured content, accessibility
**What**: Message list, composer, file upload, quick-reply/card rendering, WCAG 2.1 AA compliance.

**Design**: ARIA live region (`aria-live="polite"`) announces incoming messages; full keyboard nav; focus trap when open; respects `prefers-reduced-motion` and high-contrast. Renders `quick_reply`, `card`, `carousel`, `form` message types. Offline state shows `offlineMessage` and captures email.

**Testing**:
- `E2E (Playwright + axe-core): zero serious/critical accessibility violations.`
- `E2E: keyboard-only user can open, type, send, and close.`
- `E2E: quick-reply click sends the selected value as a visitor message.`

---

## Phase 5: Agent Inbox (Web App)

### Purpose
The workspace agents live in: a realtime inbox listing conversations, a conversation view with live messaging, canned responses, visitor context, and assignment controls. After this phase a support team can operate the platform end-to-end.

### Tasks

#### 5.1 — Inbox list & realtime updates
**What**: Filterable conversation list that updates live.

**Design**: TanStack Query for initial REST fetch (`GET /v1/conversations`), thin WS client subscribes to `tenant:{tid}:inbox` for `conversation.updated`/`message.new` to patch the cache. Views: Unassigned, Mine, Team, All; filters by status/tag/priority.

**Testing**:
- `E2E: new conversation appears in Unassigned without refresh.`
- `E2E: assigning to self moves it from Unassigned to Mine live.`

#### 5.2 — Conversation view & composer
**What**: Live thread, composer with canned-response autocomplete, private notes, file upload.

**Design**: Typing `/` triggers canned-response palette (`GET /v1/canned-responses?q=`), inserts `content` with variable substitution (`{{visitor.name}}`). Tab toggles public reply vs. private note. Agent typing emits throttled typing frames.

**Testing**:
- `E2E: agent reply appears in the visitor widget within 1s.`
- `E2E: "/" + shortcut inserts canned text with visitor name substituted.`
- `Unit: variable substitution leaves unknown placeholders intact.`

#### 5.3 — Visitor context panel
**What**: Side panel with visitor identity, properties, page-view history, and prior conversations.

**Design**: `GET /v1/visitors/:id` (incl. `properties` JSONB + CRM-mapped fields once Phase 7 lands) and `GET /v1/visitors/:id/conversations`.

**Testing**:
- `Integration: panel shows merged SDK + CRM properties; missing CRM data degrades gracefully.`

---

## Phase 6: Routing, SLA & Canned Responses

### Purpose
Turn the inbox from manual triage into an automated workflow engine — the competitive battleground identified in research (routing by intent, tier, CRM data). After this phase, new conversations auto-route to the right team/agent and SLA timers drive escalation.

### Tasks

#### 6.1 — Routing rule engine
**What**: Evaluate ordered routing rules on conversation start and assign accordingly.

**Design**: Rules from `routing_rules` (JSONB `conditions` like `{"all":[{"field":"visitor.properties.plan","op":"is","value":"enterprise"},{"field":"channel","op":"is","value":"web_chat"}]}`). Evaluator supports `all`/`any`, ops (`is`,`is_not`,`contains`,`gt`,`lt`,`in`), and field resolution against a flattened conversation+visitor+channel context. First matching active rule (by `position`) wins; `actionType` ∈ `assign_team|assign_agent|assign_bot|set_priority`. Team assignment then picks an agent via the team's `assignmentMethod` (round_robin via Redis counter, least_busy via active-conversation count, manual = leave unassigned).

**Testing**:
- `Unit: enterprise+web_chat visitor matches rule → assign_team(sales).`
- `Unit: no rule matches → falls through to default team.`
- `Unit: least_busy picks the online agent with fewest active conversations under max.`
- `Integration: round_robin distributes 6 conversations evenly across 3 agents.`

#### 6.2 — SLA policies & escalation
**What**: First-response and resolution SLA timers with escalation actions.

**Design**: `sla_policies` (tenantId, name, conditions jsonb, firstResponseMins, resolutionMins, escalation jsonb). On conversation start a BullMQ delayed job is scheduled; if `firstResponseAt` still null at deadline → escalate (reassign/notify/raise priority) and emit `sla.breached`. Timers cancelled when the condition is met.

**Testing**:
- `Integration (fake timers): no agent reply before deadline → escalation fires, sla.breached emitted.`
- `Integration: reply before deadline → job cancelled, no escalation.`

#### 6.3 — Canned responses management
**What**: CRUD + scoping (personal/team/global) for canned responses (composer consumed in 5.2).

**Design**: `/v1/canned-responses` CRUD with scope-based visibility; `shortcut` unique per tenant.

**Testing**:
- `Integration: personal canned response invisible to other agents; global visible to all.`

---

## Phase 7: CRM Integration & Webhooks

### Purpose
Connect conversations to the systems of record (HubSpot, Salesforce) and let customers react to events programmatically. CRM-aware data also powers routing (Phase 6) and AI (Phase 8). After this phase, contacts and conversation activity sync bidirectionally and external systems receive signed events.

### Tasks

#### 7.1 — Outbound webhooks
**What**: Deliver domain events to subscriber URLs with HMAC-SHA256 signatures and retry.

**Design**: `webhooks` table (url, events text[], secretHash, isActive, failureCount). Emitting a domain event enqueues a BullMQ `webhook.deliver` job per matching subscription. Payload = CloudEvents envelope (Phase 1.4); header `X-LCP-Signature: sha256=<hmac>` over the raw body. Exponential backoff (1m,5m,30m,2h,6h); auto-disable after N consecutive failures; delivery log retained.

**Testing**:
- `Integration: conversation.resolved → POST to subscriber with valid HMAC over body.`
- `Integration: subscriber 500 → retried with backoff; succeeds on attempt 2.`
- `Unit: signature verification helper accepts correct sig, rejects tampered body.`

#### 7.2 — OAuth 2.0 CRM connection
**What**: Connect HubSpot/Salesforce via OAuth 2.0 (RFC 6749); store encrypted credentials.

**Design**: `crm_connections` (crmType, credentialsEncrypted, syncEnabled, lastSyncAt, syncStatus). Standard authorization-code flow; tokens encrypted at rest (AES-256-GCM, key from env/KMS) — ISO 27001 alignment. Auto-refresh on expiry.

**Testing**:
- `Integration (mocked OAuth server): callback exchanges code → encrypted tokens stored; status=connected.`
- `Unit: credentials encrypt/decrypt round-trip; ciphertext != plaintext.`

#### 7.3 — Bidirectional contact & conversation sync
**What**: Map visitors↔CRM contacts and log conversations as CRM activities.

**Design**: `crm_contact_mappings` (crmConnectionId, visitorId, crmContactId) and `crm_conversation_logs`. On `visitor.identified` → upsert CRM contact, store mapping, and pull back CRM fields (deal stage, health score, tier) into `visitors.properties` for routing/AI. On `conversation.resolved` → log a CRM activity with transcript link. Per-provider adapter implements `findOrCreateContact`, `fetchContactProps`, `logActivity`; respects rate limits via BullMQ concurrency + retry.

**Testing**:
- `Integration (mocked HubSpot API): identified visitor with new email → contact created, mapping stored, deal_stage pulled into properties.`
- `Integration (mocked): existing CRM contact → mapped, not duplicated.`
- `Integration: resolved conversation → activity logged once (idempotent via UNIQUE(conn,conversation)).`

---

## Phase 8: AI Assist & Intent (AI-Native v1)

### Purpose
Deliver the AI-native differentiators that justify the product's positioning: real-time agent assist, visitor intent scoring for routing, and sentiment/churn detection — without yet handing the customer to an autonomous bot. After this phase agents are materially faster and routing is intent-aware.

### Tasks

#### 8.1 — AI service & prompt layer
**What**: `packages/ai` wrapper over the Vercel AI SDK with versioned prompt templates and an `ai_interactions` audit trail.

**Design**: Functions `detectIntent(messages, taxonomy)`, `analyzeSentiment(messages)`, `suggestReply(conversation, kb)`, each returning `{output, confidence, modelVersion, metadata}` persisted to `ai_interactions` (interactionType, confidence 0-1, wasAccepted, feedback). Prompts stored in `packages/ai/src/prompts/*.ts` with version tags. Provider configurable per tenant (BYO key for self-host). All calls run in the worker (off the WS loop) and stream results back via Redis → WS.

**Testing**:
- `Unit (mocked LLM): detectIntent returns one of the taxonomy labels + confidence; out-of-range confidence rejected.`
- `Integration: every AI call writes an ai_interactions row with modelVersion.`

#### 8.2 — Real-time agent assist
**What**: As the agent types (or on new visitor message), surface suggested replies + relevant KB snippets.

**Design**: New visitor message triggers a debounced worker job; results pushed as `agent.assist` WS frames `{suggestions:[{text,confidence}], articles:[{title,url,snippet}]}`. Agent can insert a suggestion (sets `ai_interactions.wasAccepted=true`) — the acceptance signal feeds future quality tracking.

**Testing**:
- `E2E (mocked AI): visitor asks a billing question → agent panel shows ≥1 suggestion within 2s.`
- `Integration: inserting a suggestion marks wasAccepted=true.`

#### 8.3 — Visitor intent scoring → routing
**What**: Classify intent at conversation start and feed it into the routing engine.

**Design**: On `conversation.started`, run `detectIntent`, write intent + confidence to `conversations.aiMeta`, then re-run routing (rules may reference `ai.intent`). Enables the underserved "knowledge-aware routing" from features.md.

**Testing**:
- `Integration: routing rule field "ai.intent" is "billing" → routes to billing team.`
- `Integration: low-confidence intent → falls back to default routing, flagged for review.`

#### 8.4 — Sentiment & churn-risk escalation
**What**: Detect negative sentiment / churn risk mid-conversation and trigger escalation.

**Design**: Periodic (every N visitor messages) sentiment analysis writes `aiMeta.sentiment`/`churnRisk`; crossing a configurable threshold emits `ai.sentiment_detected` and triggers escalation (notify senior CSM / raise priority) — reuses Phase 6.2 escalation machinery.

**Testing**:
- `Integration (mocked AI): escalating frustration → priority raised, sentiment event emitted, manager notified.`

---

## Phase 9: Analytics & Reporting

### Purpose
Give managers the metrics buyers expect (response time, resolution rate, CSAT, AI effectiveness) with dashboard queries that stay fast at volume. After this phase the agent app has an analytics section backed by pre-computed rollups.

### Tasks

#### 9.1 — Metrics pipeline (TimescaleDB continuous aggregates)
**What**: Adopt Suggestion 4's continuous-aggregate pattern for hourly/daily rollups.

**Design**: Add TimescaleDB extension. A `conversation_metrics` hypertable receives one row per measurable event (started/resolved/message/ai_auto_resolved/ai_handoff/satisfaction) tagged tenant×channel×time. Continuous aggregates materialise hourly and daily rollups (avg first-response ms, avg resolution ms, AI auto-resolution rate, CSAT). Retention/compression policies (compress >7d, drop raw >configurable) align with GDPR data minimisation.

**Testing**:
- `Integration (Timescale container): insert events across hours → hourly continuous aggregate reflects correct counts after refresh.`
- `Integration: compression policy compresses chunks older than 7 days.`

#### 9.2 — Analytics API & dashboard
**What**: Read endpoints + agent-app dashboard.

**Design**: `GET /v1/analytics/overview?from&to&channel`, `/agents`, `/ai-effectiveness` returning rollups. Dashboard charts: conversation volume, first-response/resolution time, CSAT trend, AI auto-resolution vs. handoff, agent leaderboard.

**Testing**:
- `Integration: overview for a 7-day window matches a brute-force count over raw fixtures.`
- `E2E: dashboard renders charts; date-range change refetches.`

---

## Phase 10: Autonomous Resolution, Omnichannel & Mobile

### Purpose
Close the gap with Intercom Fin / Tidio Lyro (autonomous tier-1), broaden channels beyond web chat, and give agents a mobile app. These are the high-value "backlog" features that depend on the full core being stable.

### Tasks

#### 10.1 — Autonomous tier-1 AI agent with confidence-gated handoff
**What**: A bot that resolves common queries end-to-end and hands off to a human when confidence is low.

**Design**: `bot_configurations` (botType `ai_agent`, config `{model, temperature, knowledgeBaseIds, handoffThreshold}`). When routing assigns a conversation to a bot, the worker generates responses grounded in the KB (retrieval over indexed articles). If answer confidence < `handoffThreshold` or the visitor requests a human, emit `ai.handoff_triggered`, re-route to a team with a context summary as a private note (context-preserving handoff). Every bot turn logged to `ai_interactions`.

**Testing**:
- `Integration (mocked AI): known FAQ → bot resolves, conversation resolved, no human assigned.`
- `Integration: low-confidence answer → handoff to human team with summary note attached.`
- `Integration: visitor types "talk to a human" → immediate handoff.`

#### 10.2 — Omnichannel inbound (email, SMS)
**What**: Ingest email and SMS into the same conversation model (MVP multi-channel).

**Design**: Inbound email (webhook from provider) and SMS (Twilio webhook) create/append to conversations keyed by sender identity; outbound agent replies routed back through the channel adapter. Channel-specific config lives in `channels.configuration` JSONB. Unified inbox already channel-agnostic from Phase 5.

**Testing**:
- `Integration (mocked provider): inbound email → new conversation on email channel; agent reply sent via provider.`
- `Integration: second email from same sender threads into the existing open conversation.`

#### 10.3 — Mobile agent app
**What**: React Native (Expo) app: inbox, conversation view, push notifications, presence.

**Design**: Reuses `@lcp/shared` types and the REST/WS clients; push via Expo notifications on new assignment/message.

**Testing**:
- `E2E (Expo/Detox): receive new-assignment push → open → reply delivered to visitor.`

---

## Phase 11: Compliance, Security Hardening & Deployment

### Purpose
Make the platform enterprise- and regulator-ready (GDPR/CCPA/ISO 27001) and production-deployable. After this phase the platform satisfies the compliance posture buyers demand and ships via Docker/Helm.

### Tasks

#### 11.1 — Consent, retention & right-to-erasure
**What**: GDPR/CCPA tooling (Suggestion 1's compliance tables).

**Design**: `consent_records` (visitor consent for tracking/marketing/data_processing), `erasure_requests` (status pending→processing→completed). Widget shows a consent banner before tracking. `POST /v1/visitors/:id/erase` enqueues a worker job that anonymises PII across visitors/messages/attachments while preserving conversation structure (Suggestion 2 anonymisation rationale). Configurable retention job purges transcripts past the tenant's window.

**Testing**:
- `Integration: erasure request → visitor PII nulled/anonymised, conversation skeleton retained, status=completed.`
- `Integration: retention job deletes messages older than configured window; aggregates preserved.`
- `E2E: widget without consent does not emit page-view tracking.`

#### 11.2 — Audit log & RBAC hardening
**What**: Tamper-evident audit trail and security review.

**Design**: `audit_log` (entityType, entityId, action, actorType, actorId, changes jsonb, ip) written on every mutating action via a Fastify hook. Enforce TLS 1.2+, AES-256 at rest for credentials/attachments, API-key rotation, per-tenant rate limiting (Redis token bucket), and webhook signature verification (Phase 7.1).

**Testing**:
- `Integration: assigning a conversation writes an audit_log row with before/after.`
- `Integration: exceeding rate limit → 429 with Retry-After.`
- `Security: OWASP-style checks — RLS prevents cross-tenant reads via forged IDs (returns 404/empty).`

#### 11.3 — Deployment artefacts
**What**: Production Docker images, docker-compose for self-host, Helm chart for clusters.

**Design**: Multi-stage Dockerfiles per app; `docker-compose.yml` brings up the full self-hosted stack (postgres+timescale, redis, minio, api, ws-gateway×N behind a load balancer, worker, agent-app). Helm chart parameterises replicas, secrets, and external Postgres/Redis. Health/readiness probes; graceful WS drain on shutdown.

**Testing**:
- `E2E: docker-compose up on a clean host → migrations run, widget loads, full visitor↔agent conversation works.`
- `Integration: scale ws-gateway to 2 replicas → cross-node delivery still works (Phase 3.1 test under compose).`
- `Integration: SIGTERM drains WS connections gracefully (clients reconnect, no message loss).`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Schema & Tenant Context ─── required by everything
    │
Phase 2: Conversations, Messages & REST Core ─── requires 1
    │
Phase 3: Realtime Layer (WebSocket Gateway)  ─── requires 2
    │
    ├── Phase 4: Embeddable Widget            ─── requires 3 (can parallel Phase 5)
    ├── Phase 5: Agent Inbox (Web App)        ─── requires 3 (can parallel Phase 4)
    │
Phase 6: Routing, SLA & Canned Responses     ─── requires 2 (engine), surfaces via 5
    │
Phase 7: CRM Integration & Webhooks          ─── requires 2; feeds 6 & 8 (can parallel 6)
    │
Phase 8: AI Assist & Intent                  ─── requires 3,5,6; richer with 7
    │
Phase 9: Analytics & Reporting               ─── requires 2,3 (can parallel 8)
    │
Phase 10: Autonomous AI / Omnichannel / Mobile ─ requires 6,7,8
    │
Phase 11: Compliance, Security & Deployment  ─── requires all (final hardening)
```

**Parallelism opportunities**
- Phases 4 and 5 can be built concurrently once Phase 3 lands (both consume the same WS protocol).
- Phases 6 and 7 can proceed in parallel after Phase 2 (routing engine vs. integrations).
- Phase 9 (analytics) can proceed in parallel with Phase 8 (AI) since both depend only on the core data flow.
- Within Phase 10, mobile (10.3) is independent of autonomous AI (10.1) and omnichannel (10.2).

**Estimated scope: large** (11 phases · 38 tasks · multiple deployables, realtime + AI + compliance surface).

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented and merged.
2. All unit and integration tests for the phase pass (`pnpm test`); integration tests run against real Postgres/Redis via Testcontainers.
3. ESLint and Prettier pass with no errors (`pnpm lint`).
4. Type checking passes (`tsc --noEmit`) across all affected packages.
5. Affected Docker images build successfully.
6. The phase's headline capability works end-to-end (demonstrated by at least one E2E test or a documented manual run).
7. New configuration options (env vars, JSONB config keys) are documented in the README/`.env.example`.
8. New or changed REST endpoints appear in the generated `docs/openapi.json` (OpenAPI 3.1) with request/response schemas.
9. New domain events appear in the event-type registry and (where externally relevant) the public webhook contract.
10. Database changes ship as a Drizzle migration; RLS policies cover any new tenant-scoped table.
11. For user-facing UI (widget, agent app): zero serious/critical axe-core accessibility violations (WCAG 2.1 AA).
```
