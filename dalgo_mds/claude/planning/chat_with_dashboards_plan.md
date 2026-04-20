# Chat with Dashboards - Implementation Plan

## 1. Overall

### What are we building?

We are adding a native **Chat with Dashboards** experience to Dalgo. A logged-in user viewing a dashboard can ask questions about:

- the dashboard itself
- the charts on that dashboard
- the warehouse data used by those charts
- the organization-level and dashboard-level business context maintained by Dalgo

The feature is implemented across:

- `DDP_backend` for schema, APIs, ingestion, chat orchestration, and websocket delivery
- `webapp_v2` for settings, consent flow, context editing, dashboard chat UI, and websocket client

`prefect-proxy` is not changed in v1.

### Core rules

- The feature is hidden unless the org-level feature flag `AI_DASHBOARD_CHAT` is enabled.
- Even when the feature flag is enabled, chat is unavailable until the org has explicitly enabled the consent setting `ai_data_sharing_enabled`.
- Chat is available only on authenticated native dashboard pages in v1. Public/shared dashboards do not get chat.
- Browser polling is not allowed.
- The reviewed transport is:
  - HTTP for org settings, status, and context management
  - authenticated WebSocket for sending chat messages and receiving progress/assistant_message/error events
- LangGraph is the orchestration layer.
- We do not add `langchain` as a production dependency.
- Chroma runs as a sidecar container and stores one collection per org: `org_<org_id>`.
- dbt artifacts are refreshed every 3 hours, org by org, using `dbt deps` and `dbt docs generate` through Dalgo’s existing dbt setup.
- Chat persistence uses dedicated `DashboardChatSession` and `DashboardChatMessage` tables. It does not reuse `LlmSession`.
- If we show progress in v1, it is a single user-safe label: `thinking`. We do not expose raw reasoning or tool-call internals.

### Must-keep product behaviors

- Intent routing between data questions, explanation questions, small talk, irrelevant questions, and clarification requests.
- Forced tool usage on turn 1 for data questions.
- Dashboard-scoped allowlist built from chart datasources plus upstream dbt lineage.
- Distinct-values lookup before applying text filters in generated SQL.
- Read-only SQL execution with strict validation.
- Retrieval from org context, dashboard context, dashboard export, dbt manifest, and dbt catalog.
- Cross-dashboard guidance when the current dashboard is not the right context for the user’s question.
- Source citations in the final answer payload.
- If progress is shown, it is coarse and user-safe rather than raw reasoning.

---

## 2. Approach

### What we are reusing from existing work

- Keep the dashboard export endpoint from backend PR `#1255` as the context contract for chart and dashboard metadata.
- From backend PR `#1199`, reuse only any useful settings, permission, and feature-flag patterns that still match current Dalgo conventions. Do not reuse its chat runtime, transport, or persistence design.
- From frontend PR `#162`, reuse only the useful shell and layout ideas for the dashboard chat trigger, panel, and settings UI. Replace its transport and API contract completely.

### Why dedicated chat session and message tables?

Dashboard chat is a multi-turn, dashboard-scoped, websocket-delivered workflow. It needs:

- ordered message history
- assistant payloads with citations and SQL summaries
- optional user-safe progress events

This is a different persistence problem from Dalgo’s existing summarization flows. The clean design is dedicated `DashboardChatSession` and `DashboardChatMessage` tables rather than extending `LlmSession` or storing the entire conversation in one JSON column.

### Why WebSocket-first for chat?

Dalgo already has authenticated websocket consumers built on `BaseConsumer`, JWT auth, and `orgslug` scoping. Chat is stateful and conversational, so putting the chat lifecycle on the websocket is easier to follow than splitting writes between HTTP and the socket.

- HTTP remains for settings, consent, context editing, and readiness/status reads.
- WebSocket handles:
  - `send_message`
- The same socket then delivers:
  - `progress`
  - `assistant_message`
  - `error`

This matches Dalgo’s existing consumer model better and removes chat-specific write APIs that are not strictly necessary.

### End-to-end chat flow

```text
User opens dashboard
  -> frontend calls org status/settings APIs over HTTP
  -> frontend opens authenticated dashboard-chat websocket
  -> user sends send_message over the websocket
  -> if no session_id is provided, backend creates DashboardChatSession
  -> backend stores a DashboardChatMessage(role=user) and starts processing
  -> worker loads org/dashboard context, queries Chroma, builds allowlist, and runs LangGraph
  -> worker validates and executes safe SQL if needed
  -> worker publishes a progress event with label `thinking`
  -> worker stores a DashboardChatMessage(role=assistant) and publishes events to the session channel group
  -> later turns reuse the same session_id over the same websocket
  -> a full page refresh starts a new chat session in v1
```

### How AI context build works

```text
Org eligible for AI dashboard chat
  -> scheduler enqueues one context-build job for that org
  -> worker runs dbt deps + dbt docs generate
  -> worker reads:
     - OrgAIContext.markdown
     - DashboardAIContext.markdown for dashboards in the org
     - GET /api/dashboards/{id}/export/ payloads
     - generated manifest.json from the dbt workspace
     - generated catalog.json from the dbt workspace
  -> worker chunks documents deterministically
  -> worker diffs against the org’s Chroma collection
  -> worker upserts changed docs and deletes stale/disabled-source docs
  -> worker stores freshness timestamps on OrgDbt
```

In v1, saved org and dashboard context changes are picked up by the next scheduled context-build run. We do not add a separate targeted re-ingest path.

### How the Chroma sidecar fits

We do not create a new repo for Chroma. The sidecar is just an additional runtime service.

- Local: add a Chroma service/container to Dalgo local runtime
- Backend env vars point to that service
- Backend code owns collection naming, document IDs, ingestion, deletion, and querying

This keeps operational ownership in `DDP_backend`, which is the correct home for auth, tenant isolation, orchestration, and warehouse access.

### Stack decisions

After the latest `main` updates in `DDP_backend`, dbt and Elementary are no longer part of the backend app dependency set. They continue to run from the separate dbt execution environment.

For this feature, the plan is:

- keep dbt and Elementary in the separate dbt environment already used by Dalgo
- add only chat-runtime dependencies to the backend app environment, specifically `langgraph`, `chromadb`, `openai`, and `onnxruntime`
- invoke dbt artifact generation through the existing dbt environment rather than importing dbt packages into the main backend app runtime

This keeps the backend app environment smaller and matches the current architecture on `main`.

---

## 3. Database Schema

### 3.1 `OrgPreferences`

`OrgPreferences` remains the home for opt-in / settings-style state only.

```python
class OrgPreferences(models.Model):
    ai_data_sharing_enabled = models.BooleanField(default=False)
    ai_data_sharing_consented_by = models.ForeignKey(
        OrgUser,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="ai_data_sharing_consents",
    )
    ai_data_sharing_consented_at = models.DateTimeField(null=True, blank=True)
```

Key choices:

- We do **not** add any new feature-enable column here. Feature availability continues to use Dalgo’s existing feature flag system, while account managers only control data-sharing consent.
- `ai_data_sharing_consented_by` and `ai_data_sharing_consented_at` record the last affirmative consent action.
- Org-level AI context does **not** live here. It moves to a dedicated table.

### 3.2 New table: `OrgAIContext`

```python
class OrgAIContext(models.Model):
    org = models.OneToOneField(
        Org,
        on_delete=models.CASCADE,
        related_name="ai_context",
    )
    markdown = models.TextField(blank=True, default="")
    updated_by = models.ForeignKey(
        OrgUser,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="org_ai_context_updates",
    )
    updated_at = models.DateTimeField(null=True, blank=True)
```

Key choices:

- Org AI context is its own concept and should not live in `OrgPreferences`.
- One row per org is enough in v1.

### 3.3 New table: `DashboardAIContext`

```python
class DashboardAIContext(models.Model):
    dashboard = models.OneToOneField(
        "ddpui.Dashboard",
        on_delete=models.CASCADE,
        related_name="ai_context",
    )
    markdown = models.TextField(blank=True, default="")
    updated_by = models.ForeignKey(
        OrgUser,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="dashboard_ai_context_updates",
    )
    updated_at = models.DateTimeField(null=True, blank=True)
```

Key choices:

- Dashboard AI context is also isolated in its own table so we do not keep adding AI-specific content fields to the core dashboard model.
- One row per dashboard is enough in v1.

### 3.4 `OrgDbt`

We extend `OrgDbt` with dbt artifact freshness and vector freshness metadata.

```python
class OrgDbt(models.Model):
    docs_generated_at = models.DateTimeField(null=True, blank=True)
    vector_last_ingested_at = models.DateTimeField(null=True, blank=True)
```

Key choices:

- We use cleaned-up names because these fields are properties of the dbt project, not AI-specific state.
- We do **not** store full dbt docs artifacts in SQL in v1.
- `docs_generated_at` and `vector_last_ingested_at` remain the operational freshness markers exposed through the status/settings APIs.

### 3.5 New table: `DashboardChatSession`

```python
class DashboardChatSession(models.Model):
    session_id = models.UUIDField(editable=False, unique=True, default=uuid.uuid4)
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    orguser = models.ForeignKey(OrgUser, null=True, on_delete=models.SET_NULL)
    dashboard = models.ForeignKey("ddpui.Dashboard", on_delete=models.SET_NULL, null=True)
    created_at = models.DateTimeField(auto_created=True, default=timezone.now)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "dashboard_chat_session"
        ordering = ["-updated_at"]
        indexes = [
            models.Index(
                fields=["org", "dashboard", "created_at"],
                name="dchat_sess_org_dash_idx",
            ),
        ]
```

Key choices:

- The session row stays minimal in v1. It exists to group messages by dashboard/user context, not to model a large session state machine.
- Message history does **not** live in a JSON column on the session.
- Chat is scoped to the dashboard as a whole in v1. We do not add chart-specific chat state.

### 3.6 New table: `DashboardChatMessage`

```python
class DashboardChatMessageRole(str, Enum):
    USER = "user"
    ASSISTANT = "assistant"


class DashboardChatMessage(models.Model):
    session = models.ForeignKey(
        DashboardChatSession,
        on_delete=models.CASCADE,
        related_name="messages",
    )
    sequence_number = models.PositiveIntegerField()
    role = models.CharField(
        max_length=20,
        choices=DashboardChatMessageRole.choices(),
    )
    content = models.TextField(blank=True, default="")
    client_message_id = models.CharField(max_length=100, null=True, blank=True)
    payload = models.JSONField(null=True, blank=True)
    created_at = models.DateTimeField(auto_created=True, default=timezone.now)

    class Meta:
        ordering = ["sequence_number"]
        constraints = [
            models.UniqueConstraint(
                fields=["session", "sequence_number"],
                name="dchat_message_session_seq_unique",
            ),
        ]
```

`payload` is used for structured assistant response data such as citations, related dashboards, warnings, and SQL summary.

Example assistant-message payload:

```json
{
  "citations": [],
  "related_dashboards": [],
  "warnings": [],
  "sql": "SELECT ..."
}
```

Key choices:

- We use a normalized message table because storing an entire conversation in a session JSON column does not scale well.
- The latest assistant response is simply the latest assistant message row. We do **not** duplicate it as a separate `latest_response` session field.
- We do **not** keep a vague `request_meta` field on the session. Message-specific client IDs and assistant payloads are stored on the message row they belong to.
- If we later want richer progress UI, it should be derived from coarse progress events rather than raw reasoning data.

---

## 4. Vector Document Design

### Collection strategy

- One Chroma collection per org: `org_<org_id>`
- No shared multi-tenant collection in v1

This makes:

- org isolation simpler
- rebuilds simpler
- source-type deletion simpler

### Source types

The ingester produces these source types:

- `org_context`
- `dashboard_context`
- `dashboard_export`
- `dbt_manifest`
- `dbt_catalog`

### Metadata shape

Each vector document stores:

```json
{
  "org_id": 42,
  "source_type": "dashboard_export",
  "source_identifier": "dashboard:17:chart:128",
  "dashboard_id": 17,
  "chart_id": 128,
  "title": "Donor funding by quarter",
  "document_hash": "..."
}
```

### Document ID strategy

Document IDs are deterministic:

```text
<org_id>:<source_type>:<source_identifier>:<chunk_index>:<content_hash>
```

This supports:

- idempotent upserts
- precise stale-document deletion
- clean diffing between the target document set and the current collection

### Chunking rules

- Org/dashboard markdown:
  - split by headings first
  - then paragraph-based chunks around 1,000 characters
  - small overlap for continuity
- Dashboard export:
  - one dashboard summary document
  - one chart document per chart
- Manifest:
  - one document per dbt model
  - include description, schema, parents, and important columns
- Catalog:
  - one document per dbt model
  - include schema, data types, nullability, and useful catalog descriptions

### Source controls

V1 keeps source-type enablement as backend runtime configuration rather than a persisted per-org field.

- enabled source types are included during ingestion
- disabled source types are omitted during ingestion and ignored during retrieval
- if Dalgo later needs per-org source overrides, they should live in a dedicated AI settings model rather than `OrgPreferences`

---

## 5. API Design

Response shapes follow the conventions of the existing router they live under.

- `orgpreferences` endpoints use Dalgo’s standard success envelope:

```json
{
  "success": true,
  "res": {}
}
```

- native dashboard endpoints follow the existing `dashboard_native_api` convention and return the typed payload directly instead of wrapping it in a success envelope

### 5.1 Org settings APIs

```http
GET /api/orgpreferences/ai-dashboard-chat
PUT /api/orgpreferences/ai-dashboard-chat
GET /api/orgpreferences/ai-dashboard-chat/status
```

#### `GET /api/orgpreferences/ai-dashboard-chat`

Purpose:

- load org-level consent and context settings
- show dbt/vector freshness

Response:

```json
{
  "success": true,
  "res": {
    "feature_flag_enabled": true,
    "ai_data_sharing_enabled": true,
    "ai_data_sharing_consented_by": "manager@org.org",
    "ai_data_sharing_consented_at": "2026-03-17T10:00:00Z",
    "org_context_markdown": "## Org context ...",
    "org_context_updated_by": "manager@org.org",
    "org_context_updated_at": "2026-03-17T10:15:00Z",
    "dbt_configured": true,
    "docs_generated_at": "2026-03-17T09:00:00Z",
    "vector_last_ingested_at": "2026-03-17T09:05:00Z"
  }
}
```

#### `PUT /api/orgpreferences/ai-dashboard-chat`

Request:

```json
{
  "ai_data_sharing_enabled": true,
  "org_context_markdown": "## Org context ..."
}
```

Behavior:

- requires `can_manage_org_settings`
- if consent changes from `false` to `true`, backend stamps `ai_data_sharing_consented_by` and `ai_data_sharing_consented_at`
- updates `OrgAIContext.markdown` and stamps its editor metadata
- updating markdown marks the org as needing re-ingest

#### `GET /api/orgpreferences/ai-dashboard-chat/status`

Response:

```json
{
  "success": true,
  "res": {
    "feature_flag_enabled": true,
    "ai_data_sharing_enabled": true,
    "chat_available": true,
    "dbt_configured": true,
    "docs_generated_at": "2026-03-17T09:00:00Z",
    "vector_last_ingested_at": "2026-03-17T09:05:00Z"
  }
}
```

`chat_available` is computed as:

```text
feature_flag_enabled
AND ai_data_sharing_enabled
AND dbt_configured
AND vector_last_ingested_at is not null
```

### 5.2 Dashboard context APIs

```http
GET /api/dashboards/{dashboard_id}/export/
GET /api/dashboards/{dashboard_id}/ai-context/
PUT /api/dashboards/{dashboard_id}/ai-context/
```

#### `GET /api/dashboards/{dashboard_id}/export/`

Purpose:

- return dashboard configuration for downstream context ingestion
- include the dashboard payload and all referenced chart configs

#### `GET /api/dashboards/{dashboard_id}/ai-context/`

Response:

```json
{
  "dashboard_id": 17,
  "dashboard_title": "Donor Report",
  "dashboard_context_markdown": "## Dashboard context ...",
  "dashboard_context_updated_by": "manager@org.org",
  "dashboard_context_updated_at": "2026-03-17T10:16:00Z",
  "vector_last_ingested_at": "2026-03-17T09:05:00Z"
}
```

#### `PUT /api/dashboards/{dashboard_id}/ai-context/`

Request:

```json
{
  "dashboard_context_markdown": "## Dashboard context ..."
}
```

Behavior:

- requires `can_manage_org_settings`
- updates `DashboardAIContext.markdown` and stamps editor metadata
- change is picked up in the next scheduled context-build run

### 5.3 Chat websocket contract

Endpoint:

```http
GET /wss/dashboards/{dashboard_id}/chat/?token=<jwt>&orgslug=<org_slug>
```

The chat websocket follows Dalgo’s existing authenticated consumer pattern:

- `BaseConsumer`
- JWT in the `token` query param
- `orgslug` in the query param
- native dashboard authorization enforced after connect
- each active chat session maps to a channel-layer group keyed by `session_id`

Client actions sent over the websocket:

- `send_message`

#### `send_message`

```json
{
  "action": "send_message",
  "message": "Why did donor funding drop this quarter?",
  "client_message_id": "ui-1742206800-1"
}
```

For later turns on the same page, the frontend includes the returned session ID:

```json
{
  "action": "send_message",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62",
  "message": "Can you break that down by donor type?",
  "client_message_id": "ui-1742206800-2"
}
```

Validation:

- requires dashboard access
- org must have `AI_DASHBOARD_CHAT` enabled
- if `session_id` is omitted, backend creates a new `DashboardChatSession`
- if `session_id` is provided, it must belong to the current org and dashboard
- if org is not ready for chat, respond with an `error` event and do not start processing

Server events emitted after `send_message`:

- `progress`
- `assistant_message`
- `error`

Example `progress` event:

```json
{
  "event_type": "progress",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62",
  "dashboard_id": 42,
  "message_id": "a939f3cb-1b07-47ae-b4d5-a0c31eaf3f8d",
  "occurred_at": "2026-03-17T10:21:04Z",
  "data": {
    "label": "thinking"
  }
}
```

Example `assistant_message` event:

```json
{
  "event_type": "assistant_message",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62",
  "dashboard_id": 42,
  "message_id": "ffd4d3eb-d80a-4c0d-b9db-c531b999b55c",
  "occurred_at": "2026-03-17T10:21:12Z",
  "data": {
    "role": "assistant",
    "content": "...",
    "payload": {
      "citations": [],
      "related_dashboards": [],
      "warnings": [],
      "sql": "SELECT ..."
    }
  }
}
```

`progress` is intentionally minimal in v1 and always uses the user-safe label `thinking`.

### 5.4 No dedicated chat webhook in v1

The reviewed v1 design does **not** add a chat-specific webhook endpoint.

- browser-facing auth is already handled by the existing websocket pattern
- the async worker can publish directly to the channel layer
- if Dalgo later introduces an external chat-execution service, add a dedicated chat webhook with its own auth scheme instead of reusing notification webhook keys

---

## 6. Runtime Design

### LangGraph nodes

The runtime package lives in `ddpui/core/dashboard_chat/`.

The graph contains these nodes:

- `load_context`
- `route_intent`
- `retrieve_docs`
- `build_allowlist`
- `load_schema_snippets`
- `plan_query`
- `lookup_distinct_values`
- `generate_sql`
- `validate_sql`
- `execute_sql`
- `compose_answer`
- `finalize`

### Intent behavior

- Data questions always start with tool usage on turn 1.
- Explanation/context questions can answer from retrieval without SQL.
- Small talk and irrelevant questions return direct responses without warehouse access.
- Ambiguous questions return clarification prompts.

### Allowlist behavior

The allowlist is built from:

- chart datasources in the current dashboard export
- upstream dbt lineage from the current manifest

It is enforced for:

- retrieval of dbt-related docs
- schema lookup
- SQL validation
- table discovery utilities

### Distinct-values gate

Before generated SQL uses a text filter, the runtime must confirm allowed values using `get_distinct_values`.

This prevents:

- hallucinated filter values
- silent no-result answers caused by invented string values

### SQL guard behavior

V1 rules are fixed:

- single statement only
- `SELECT` or `WITH ... SELECT` only
- no DDL/DML
- no forbidden schemas
- enforce allowlist
- enforce row limit
- enforce timeout

### PII behavior

If a query attempts to return row-level values from columns that match sensitive-name heuristics, the runtime rejects the result and asks the user to aggregate or rephrase.

### Cross-dashboard behavior

If the current dashboard is not the right place to answer the question, the runtime returns:

- a helpful explanation
- a `related_dashboards` array with suggested dashboards

It does not try to answer the question from the wrong dashboard context.

### Citation behavior

The final response always includes citations when retrieval sources were used. Citation types are:

- `org_context`
- `dashboard_context`
- `dashboard_export`
- `dbt_manifest`
- `dbt_catalog`

---

## 7. Frontend Design

### Visibility and gating

- Add `FeatureFlagKeys.AI_DASHBOARD_CHAT` to the existing feature flag hook.
- If the flag is off, nothing about the feature appears in the webapp.
- If the flag is on, org settings are visible only to users with `can_manage_org_settings`.
- Dashboard chat is visible only when the status endpoint says `chat_available = true`.

### Settings page

The settings page must include:

- a single top-level toggle with exact text: `I consent to share data with AI`
- a required modal explaining data sharing and OpenAI usage when enabling the toggle
- org markdown editor with preview/save
- dashboard selector
- dashboard markdown editor with preview/save
- last-updated metadata
- dbt/vector freshness timestamps

### Dashboard chat UI

Reuse the useful shell/layout ideas from frontend PR `#162`, but replace the transport completely. Reuse from backend PR `#1199` is limited to any useful settings, permission, and feature-flag patterns; do not reuse its chat runtime or transport design.

The UI flow is:

- open authenticated dashboard-chat websocket
- send `send_message` for each user turn
- let the backend create the session implicitly on the first turn
- reuse the returned `session_id` for later turns on the same page
- listen for websocket events and update the panel state
- a full page refresh starts a new chat session in v1

### Public dashboards

Public/shared dashboards do not get chat in v1.

---

## 8. Implementation Phases

### Phase 1: Backend foundation

- Finalize schema changes for `OrgPreferences`, `Dashboard`, `OrgDbt`, and `DashboardChatSession`
- Finalize org settings and dashboard context APIs
- Keep the dashboard export endpoint as the ingestion contract
- Keep feature flag and permission wiring in the existing Dalgo patterns

### Phase 2: Vector and ingestion layer

- Chroma sidecar config
- vector document builders
- document diffing and stale deletion
- org-scoped collection management
- dbt docs generation helper

### Phase 3: Scheduled freshness

- 3-hour beat task
- one org-specific context-build job per org
- per-org lock to prevent concurrent rebuilds
- context changes are incorporated by the next scheduled run
- failure handling that preserves last good vector state

### Phase 4: Chat transport and orchestration

- websocket consumer
- websocket action handling for `send_message`
- async worker execution
- LangGraph runtime integration
- message persistence using `DashboardChatMessage`
- one minimal progress event label: `thinking`

### Phase 5: Frontend integration

- feature flag key
- settings page
- consent modal
- context editors
- chat panel
- websocket client
- fresh session on page load

### Phase 6: Hardening

- focused backend tests
- focused frontend tests
- local env and run docs
- rollout checklist for pilot orgs

---

## 9. Repo Impact

### `DDP_backend`

Owns:

- schema
- org settings/context APIs
- dashboard export consumption
- vector infrastructure
- ingest jobs
- scheduler
- websocket delivery
- websocket action handling
- LangGraph runtime

### `webapp_v2`

Owns:

- feature flag plumbing
- settings page
- consent UX
- context editors
- dashboard chat UI
- websocket client

### `prefect-proxy`

No code changes in v1.

---

## 10. Testing Strategy

### Backend

- model tests for `DashboardChatSession` lifecycle and JSON fields
- settings/context API tests
- dashboard export tests
- ingestion diffing tests
- disabled-source deletion tests
- dbt docs refresh tests
- scheduler and per-org isolation tests
- runtime tests for:
  - intent routing
  - allowlist enforcement
  - distinct-values gating
  - SQL guard failures
  - PII rejection
  - citation payload shape
  - cross-dashboard suggestions
- websocket auth and event-ordering tests
- websocket action tests for `send_message`

### Frontend

- feature-flag-hidden state
- permission-hidden state
- consent modal flow
- org context save
- dashboard context save
- websocket happy path
- websocket error path
- page refresh starts a new session

### End-to-end

- flag off hides everything
- flag on plus no consent shows only the settings surface for authorized users
- consent on plus completed ingest enables dashboard chat
- dbt changes appear after the next scheduled refresh
- disabled sources stop influencing answers
- org data never leaks across collections or SQL access

---

## 11. External References

- LangGraph docs: [https://langchain-ai.github.io/langgraph/](https://langchain-ai.github.io/langgraph/)
- Chroma docs: [https://docs.trychroma.com/](https://docs.trychroma.com/)
- Django Channels docs: [https://channels.readthedocs.io/](https://channels.readthedocs.io/)
- dbt docs reference: [https://docs.getdbt.com/](https://docs.getdbt.com/)

---

## 12. Confidence Score

**9.0 / 10**

The main remaining risk is operational hardening around scheduled dbt refresh failures and websocket behavior under failure/reconnect conditions, but the architecture, schema, and API decisions are consistent and implementation-ready.
