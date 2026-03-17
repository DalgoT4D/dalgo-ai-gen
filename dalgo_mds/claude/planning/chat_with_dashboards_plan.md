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
  - authenticated WebSocket for starting chat sessions, sending chat messages, resuming chat sessions, and receiving queued/progress/answer/error/done events
- LangGraph is the orchestration layer.
- We do not add `langchain` as a production dependency.
- Chroma runs as a sidecar container and stores one collection per org: `org_<org_id>`.
- dbt artifacts are refreshed every 3 hours, org by org, using `dbt deps` and `dbt docs generate` through Dalgo’s existing dbt setup.
- Chat persistence uses dedicated `DashboardChatSession` and `DashboardChatMessage` tables. It does not reuse `LlmSession`.

### Must-keep product behaviors

- Intent routing between data questions, explanation questions, small talk, irrelevant questions, and clarification requests.
- Forced tool usage on turn 1 for data questions.
- Dashboard-scoped allowlist built from chart datasources plus upstream dbt lineage.
- Distinct-values lookup before applying text filters in generated SQL.
- Read-only SQL execution with strict validation.
- Retrieval from org context, dashboard context, dashboard export, dbt manifest, and dbt catalog.
- Cross-dashboard guidance when the current dashboard is not the right context for the user’s question.
- Source citations in the final answer payload.
- Future support for showing sanitized “agent steps” without storing or exposing raw chain-of-thought.

---

## 2. Approach

### Why dedicated chat session and message tables?

Dashboard chat is a multi-turn, dashboard-scoped, websocket-delivered workflow. It needs:

- session lifecycle state
- ordered message history
- per-message chart focus
- assistant payloads with citations and SQL summaries
- sanitized graph progress trace

This is a different persistence problem from Dalgo’s existing summarization flows. The clean design is dedicated `DashboardChatSession` and `DashboardChatMessage` tables rather than extending `LlmSession` or storing the entire conversation in one JSON column.

### Why WebSocket-first for chat?

Dalgo already has authenticated websocket consumers built on `BaseConsumer`, JWT auth, and `orgslug` scoping. Chat is stateful and conversational, so putting the chat lifecycle on the websocket is easier to follow than splitting writes between HTTP and the socket.

- HTTP remains for settings, consent, context editing, and readiness/status reads.
- WebSocket handles:
  - `start_session`
  - `send_message`
  - `resume_session`
- The same socket then delivers:
  - `session_started`
  - `queued`
  - `progress`
  - `assistant_message`
  - `error`
  - `done`

This matches Dalgo’s existing consumer model better and removes chat-specific write APIs that are not strictly necessary.

### End-to-end chat flow

```text
User opens dashboard
  -> frontend calls org status/settings APIs over HTTP
  -> frontend opens authenticated dashboard-chat websocket
  -> frontend sends start_session
  -> backend creates DashboardChatSession and returns session_started
  -> user sends send_message over the same websocket
  -> backend stores a DashboardChatMessage(role=user), marks session queued, and emits queued
  -> worker loads org/dashboard context, queries Chroma, builds allowlist, and runs LangGraph
  -> worker validates and executes safe SQL if needed
  -> worker stores a DashboardChatMessage(role=assistant) and publishes events to the session channel group
  -> backend emits done
  -> later turns reuse the same session_id over the same websocket
  -> on reconnect, frontend sends resume_session to get the persisted message history for that session
```

### How ingestion works

```text
Org eligible for AI dashboard chat
  -> scheduler enqueues one ingest job for that org
  -> worker runs dbt deps + dbt docs generate
  -> worker reads:
     - OrgAIContext.markdown
     - DashboardAIContext.markdown for dashboards in the org
     - GET /api/dashboards/{id}/export/ payloads
     - OrgDbt.manifest_json
     - OrgDbt.catalog_json
  -> worker chunks documents deterministically
  -> worker diffs against the org’s Chroma collection
  -> worker upserts changed docs and deletes stale/disabled-source docs
  -> worker stores manifest/catalog JSON and freshness timestamps on OrgDbt
```

### How targeted re-ingest works

Targeted re-ingest is triggered by:

- org context changes
- dashboard context changes
- dashboard export changes that affect chart context

Targeted re-ingest does **not** rerun `dbt docs generate`. It reuses the latest successful `manifest_json` and `catalog_json` already stored on `OrgDbt` and rebuilds the vector collection with the latest markdown and dashboard export content.

### How the Chroma sidecar fits

We do not create a new repo for Chroma. The sidecar is just an additional runtime service.

- Local: add a Chroma service/container to Dalgo local runtime
- Backend env vars point to that service
- Backend code owns collection naming, document IDs, ingestion, deletion, and querying

This keeps operational ownership in `DDP_backend`, which is the correct home for auth, tenant isolation, orchestration, and warehouse access.

### Stack decisions

The backend app environment must support:

- `elementary-data==0.22.0`
- `langgraph==0.0.69`
- `chromadb==0.4.24`
- `openai==1.55.3`
- `onnxruntime==1.20.1`

This is the required dependency line for the reviewed implementation. The important point is that LangGraph compatibility is solved inside the existing backend environment; we are not creating a separate runtime service for the chat agent.

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

- We do **not** add an `ai_dashboard_chat_enabled` column here. Feature enablement remains in Dalgo’s existing feature flag system.
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
    manifest_json = models.JSONField(null=True, blank=True)
    catalog_json = models.JSONField(null=True, blank=True)
    docs_generated_at = models.DateTimeField(null=True, blank=True)
    vector_last_ingested_at = models.DateTimeField(null=True, blank=True)
```

Key choices:

- We use cleaned-up names because these fields are properties of the dbt project, not AI-specific state.
- `manifest_json` and `catalog_json` store the latest successful dbt docs artifacts so targeted re-ingest can run without rerunning `dbt docs generate`.
- Content digests for change detection are computed from these JSON values during ingestion; they do not need separate persisted hash columns in v1.
- `docs_generated_at` and `vector_last_ingested_at` remain the operational freshness markers exposed through the status/settings APIs.

### 3.5 New table: `DashboardChatSession`

```python
class DashboardChatSessionStatus(str, Enum):
    INITIALIZED = "initialized"
    QUEUED = "queued"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"


class DashboardChatSession(models.Model):
    session_id = models.UUIDField(editable=False, unique=True, default=uuid.uuid4)
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    orguser = models.ForeignKey(OrgUser, null=True, on_delete=models.SET_NULL)
    dashboard = models.ForeignKey("ddpui.Dashboard", on_delete=models.SET_NULL, null=True)
    selected_chart = models.ForeignKey(
        "ddpui.Chart", on_delete=models.SET_NULL, null=True, blank=True
    )
    status = models.CharField(
        max_length=50,
        choices=DashboardChatSessionStatus.choices(),
        default=DashboardChatSessionStatus.INITIALIZED,
    )
    feedback = models.TextField(null=True, blank=True)
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

- Session-level status stays on the session row.
- `selected_chart` stores the current/default chart context for the session and is updated whenever the user changes chart focus.
- Message history does **not** live in a JSON column on the session.
- Chart focus is stored per message, not once at session level, because the user can change chart focus over time.

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
    selected_chart = models.ForeignKey(
        "ddpui.Chart", on_delete=models.SET_NULL, null=True, blank=True
    )
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

`payload` is used for structured assistant response data such as citations, related dashboards, warnings, SQL summary, usage, and graph trace.

Example assistant-message payload:

```json
{
  "citations": [],
  "related_dashboards": [],
  "warnings": [],
  "sql": "SELECT ...",
  "sql_results": [],
  "usage": {
    "model": "gpt-4o-mini",
    "input_tokens": 0,
    "output_tokens": 0
  },
  "graph_trace": [
    {
      "node": "retrieve_docs",
      "label": "Finding the right data sources",
      "status": "completed"
    }
  ]
}
```

Key choices:

- We use a normalized message table because storing an entire conversation in a session JSON column does not scale well.
- The latest assistant response is simply the latest assistant message row. We do **not** duplicate it as a separate `latest_response` session field.
- We do **not** keep a vague `request_meta` field on the session. Message-specific client IDs and assistant payloads are stored on the message row they belong to.
- `graph_trace` is sanitized and safe for future UI display.

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

All success responses use Dalgo’s standard envelope:

```json
{
  "success": true,
  "res": {}
}
```

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
  "success": true,
  "res": {
    "dashboard_id": 17,
    "dashboard_title": "Donor Report",
    "dashboard_context_markdown": "## Dashboard context ...",
    "dashboard_context_updated_by": "manager@org.org",
    "dashboard_context_updated_at": "2026-03-17T10:16:00Z",
    "vector_last_ingested_at": "2026-03-17T09:05:00Z"
  }
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
- marks the org as needing targeted re-ingest

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

- `start_session`
- `send_message`
- `resume_session`

#### `start_session`

```json
{
  "action": "start_session",
  "selected_chart_id": 128
}
```

Server response:

```json
{
  "event_type": "session_started",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62",
  "dashboard_id": 42,
  "occurred_at": "2026-03-17T10:20:00Z",
  "data": {
    "status": "initialized",
    "selected_chart_id": 128
  }
}
```

Validation:

- requires dashboard access
- org must have `AI_DASHBOARD_CHAT` enabled
- if `selected_chart_id` is provided, it must belong to the dashboard
- the consumer stores `selected_chart_id` on `DashboardChatSession.selected_chart` as the default chart context for the session

#### `send_message`

```json
{
  "action": "send_message",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62",
  "message": "Why did donor funding drop this quarter?",
  "selected_chart_id": 128,
  "client_message_id": "ui-1742206800-1"
}
```

Immediate server acknowledgement:

```json
{
  "event_type": "queued",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62",
  "dashboard_id": 42,
  "message_id": "a939f3cb-1b07-47ae-b4d5-a0c31eaf3f8d",
  "occurred_at": "2026-03-17T10:21:00Z",
  "data": {
    "status": "queued"
  }
}
```

Validation:

- session must belong to the current org and dashboard
- selected chart, if provided, must belong to the same dashboard
- if org is not ready for chat, respond with an `error` event and do not enqueue work
- if `selected_chart_id` is provided, the backend stores it on both `DashboardChatMessage.selected_chart` and `DashboardChatSession.selected_chart`

#### `resume_session`

```json
{
  "action": "resume_session",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62"
}
```

Server response:

```json
{
  "event_type": "session_started",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62",
  "dashboard_id": 42,
  "occurred_at": "2026-03-17T10:22:00Z",
  "data": {
    "status": "completed",
    "selected_chart_id": 128,
    "messages": [
      {
        "id": "a939f3cb-1b07-47ae-b4d5-a0c31eaf3f8d",
        "role": "user",
        "content": "Why did donor funding drop this quarter?",
        "selected_chart_id": 128,
        "created_at": "2026-03-17T10:21:00Z",
        "payload": null
      },
      {
        "id": "ffd4d3eb-d80a-4c0d-b9db-c531b999b55c",
        "role": "assistant",
        "content": "...",
        "selected_chart_id": 128,
        "created_at": "2026-03-17T10:21:12Z",
        "payload": {
          "citations": [],
          "related_dashboards": [],
          "warnings": [],
          "sql": "SELECT ..."
        }
      }
    ]
  }
}
```

Server events emitted after `send_message`:

- `queued`
- `progress`
- `assistant_message`
- `error`
- `done`

Example `progress` event:

```json
{
  "event_type": "progress",
  "session_id": "0d1d77b7-6d52-4a9a-b8a7-5a9c6e3f1b62",
  "dashboard_id": 42,
  "message_id": "a939f3cb-1b07-47ae-b4d5-a0c31eaf3f8d",
  "occurred_at": "2026-03-17T10:21:04Z",
  "data": {
    "label": "Finding the right data sources",
    "node": "retrieve_docs",
    "status": "completed"
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
    "selected_chart_id": 128,
    "payload": {
      "citations": [],
      "related_dashboards": [],
      "warnings": [],
      "sql": "SELECT ..."
    }
  }
}
```

`progress` labels are sanitized and safe for eventual user display.

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

Reuse the useful shell/layout ideas from frontend PR `#162`, but replace the transport completely.

The UI flow is:

- open authenticated dashboard-chat websocket
- send `start_session`
- send `send_message` for each user turn
- send `resume_session` after reconnect or reload
- listen for websocket events and update the panel state

### Chart focus

`DashboardNativeView` needs explicit selected-chart state.

This state is used for:

- carrying `selected_chart_id` into session/message requests
- prioritizing relevant chart context in the runtime
- showing in the chat panel that the user is asking about a specific chart

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
- one org-specific ingest job per org
- per-org lock to prevent concurrent rebuilds
- targeted re-ingest for context changes
- failure handling that preserves last good vector state

### Phase 4: Chat transport and orchestration

- websocket consumer
- websocket action handling for `start_session`, `send_message`, and `resume_session`
- async worker execution
- LangGraph runtime integration
- message persistence using `DashboardChatMessage`
- safe graph trace persistence

### Phase 5: Frontend integration

- feature flag key
- settings page
- consent modal
- context editors
- chat panel
- websocket client
- chart focus integration
- session recovery

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
- chart focus state

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
- websocket action tests for `start_session`, `send_message`, and `resume_session`

### Frontend

- feature-flag-hidden state
- permission-hidden state
- consent modal flow
- org context save
- dashboard context save
- websocket happy path
- websocket error path
- selected-chart propagation
- session reload recovery

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

The main remaining risk is operational hardening around scheduled dbt refresh failures and websocket/session recovery, but the architecture, schema, and API decisions are consistent and implementation-ready.
