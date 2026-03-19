# Report Comments — Implementation Reference

This document captures the full implementation of collaborative comments on report snapshots in Dalgo, including the Figma-based UI redesign.

---

## Architecture Overview

```
Frontend (webapp_v2)                          Backend (DDP_backend)
┌─────────────────────┐                      ┌──────────────────────────────┐
│ types/comments.ts   │                      │ models/comment.py            │
│ hooks/api/          │  ──HTTP/JSON──▶      │ schemas/comment_schema.py    │
│   useComments.ts    │                      │ api/comments_api.py          │
│ components/reports/ │                      │ core/comments/               │
│   comment-popover   │  ◀──JSON────         │   comment_service.py         │
│   comment-icon      │                      │   mention_service.py         │
│   utils.ts          │                      │   exceptions.py              │
└─────────────────────┘                      └──────────────────────────────┘
```

**Key principle**: API → Core Service → Models (one-way dependency). The API layer is thin — it validates via Pydantic schemas, checks permissions, calls the service, and wraps responses.

**Targeting model**: Comments are scoped to a **report snapshot** (`ReportSnapshot`). Each comment targets either the `summary` (executive summary level) or a specific `chart` (by `snapshot_chart_id`). This is a flat comment model — no threading or nesting.

**Target types** are managed via an application-level enum (`CommentTargetType`) — NOT via Django model `choices`. This avoids unnecessary migrations when adding new target types.

---

## Database Models (`ddpui/models/comment.py`)

### `CommentTargetType` (Application-level enum)

```python
class CommentTargetType:
    CHART = "chart"
    SUMMARY = "summary"
    ALL = [CHART, SUMMARY]
```

**Note**: This is NOT a Django enum. It's a plain class used for validation in the service layer. The `target_type` CharField has NO `choices=` parameter, so adding new target types doesn't require a migration.

Two models:

### `Comment`

The core comment record.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `BigAutoField` | Primary key |
| `target_type` | `CharField(max_length=20)` | `"chart"` or `"summary"` (no DB-level choices) |
| `snapshot` | `ForeignKey(ReportSnapshot)` | Nullable, for report snapshot comments |
| `snapshot_chart_id` | `IntegerField` | Nullable. Chart ID within `frozen_chart_configs` (NOT a FK) |
| `content` | `TextField` | Comment text body (max 5000 chars enforced at schema level) |
| `mentioned_emails` | `JSONField` | Default `[]`. List of email addresses mentioned in this comment |
| `author` | `ForeignKey(OrgUser)` | Comment author |
| `org` | `ForeignKey(Org)` | Multi-tenant isolation |
| `created_at` | `DateTimeField` | Auto-set on creation |
| `updated_at` | `DateTimeField` | Auto-set on every save |

**Indexes**: `(org, created_at)`

**Behavioral notes**: Mentions are stored as a JSON list of email strings instead of a separate table. Deletes are hard deletes (row removed).

### `CommentReadStatus`

Tracks per-user read state for each comment target on a report. `last_read_at` is updated when the user closes the comment popover. Comments with `created_at > last_read_at` are considered "new".

| Field | Type | Description |
|-------|------|-------------|
| `id` | `BigAutoField` | Primary key |
| `user` | `ForeignKey(OrgUser)` | The user |
| `snapshot` | `ForeignKey(ReportSnapshot)` | The report snapshot |
| `target_type` | `CharField(max_length=20)` | `"summary"` or `"chart"` (no DB-level choices) |
| `chart_id` | `IntegerField` | Nullable. Chart ID within frozen_chart_configs (for chart-level read status) |
| `last_read_at` | `DateTimeField` | When the user last read this target's comments |

**Constraints**: `unique_together = [("user", "snapshot", "target_type", "chart_id")]`

**Indexes**: `(user, snapshot)`

---

## Backend Schemas (`ddpui/schemas/comment_schema.py`)

### Request Schemas

#### `CommentCreate` — body for `POST /api/comments/`

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `snapshot_id` | `int` | Yes | — | ReportSnapshot ID |
| `target_type` | `str` | Yes | `"summary"` or `"chart"` | What the comment is attached to |
| `chart_id` | `int \| null` | No | Required when `target_type="chart"` | Chart ID within frozen_chart_configs |
| `content` | `str` | Yes | min 1, max 5000 chars | Comment text body (may contain @email mentions) |

```json
{
  "snapshot_id": 42,
  "target_type": "chart",
  "chart_id": 7,
  "content": "This number looks off @noopur@projecttech4dev.org"
}
```

#### `CommentUpdate` — body for `PUT /api/comments/{comment_id}/`

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `content` | `str` | Yes | min 1, max 5000 chars | Updated comment text |

```json
{
  "content": "Updated: this number looks correct now"
}
```

#### `MarkReadRequest` — body for `POST /api/comments/mark-read/`

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `snapshot_id` | `int` | Yes | — | ReportSnapshot ID |
| `target_type` | `str` | Yes | `"summary"` or `"chart"` | Which target to mark read |
| `chart_id` | `int \| null` | No | Required when `target_type="chart"` | Chart ID within snapshot |

```json
{
  "snapshot_id": 42,
  "target_type": "summary"
}
```

### Response Schemas

#### `CommentAuthorResponse` — nested inside CommentResponse

| Field | Type | Description |
|-------|------|-------------|
| `email` | `str` | User's email address |
| `name` | `str \| null` | Display name (`first_name` + `last_name` from Django User), null if not set |

```json
{ "email": "noopur@projecttech4dev.org", "name": "Noopur Raval" }
```

**Helper**: `_get_user_name(orguser)` joins `user.first_name` and `user.last_name`, returns `None` if both empty.

#### `CommentMentionResponse` — nested inside CommentResponse

| Field | Type | Description |
|-------|------|-------------|
| `email` | `str` | Mentioned user's email |
| `name` | `str \| null` | Mentioned user's display name |

```json
{ "email": "siddhant@dalgo.in", "name": "Siddhant" }
```

#### `CommentResponse` — single comment, returned by list/create/update endpoints

| Field | Type | Description |
|-------|------|-------------|
| `id` | `int` | Comment primary key |
| `target_type` | `str` | `"summary"` or `"chart"` |
| `snapshot_id` | `int` | ReportSnapshot ID |
| `chart_id` | `int \| null` | Chart ID within frozen_chart_configs (null for report-level) |
| `content` | `str` | Comment text body |
| `author` | `CommentAuthorResponse` | Author email and name |
| `is_new` | `bool` | Whether this comment is unread by the requesting user (computed from `CommentReadStatus.last_read_at`) |
| `created_at` | `datetime` | ISO-8601 creation timestamp |
| `updated_at` | `datetime` | ISO-8601 last update timestamp |
| `mentions` | `CommentMentionResponse[]` | List of @mentioned users |

**`from_model(cls, comment)`** mapping:
- `author.email` ← `comment.author.user.email`
- `author.name` ← `_get_user_name(comment.author)`
- `chart_id` ← `comment.snapshot_chart_id`
- `is_new` ← `getattr(comment, "is_new", False)` (set by service layer)
- `mentions` ← resolved from `comment.mentioned_emails` JSONField (batch-resolved in service layer)

```json
{
  "id": 15,
  "target_type": "chart",
  "snapshot_id": 42,
  "chart_id": 7,
  "content": "This number looks off @siddhant@dalgo.in",
  "author": {
    "email": "noopur@projecttech4dev.org",
    "name": "Noopur Raval"
  },
  "is_new": true,
  "created_at": "2026-03-13T10:30:00Z",
  "updated_at": "2026-03-13T10:30:00Z",
  "mentions": [
    { "email": "siddhant@dalgo.in", "name": "Siddhant" }
  ]
}
```

#### `CommentStateEntry` — nested inside CommentStatesResponse

| Field | Type | Description |
|-------|------|-------------|
| `state` | `str` | One of: `"none"`, `"unread"`, `"read"`, `"mentioned"` |
| `count` | `int` | Total number of comments for this target |
| `unread_count` | `int` | Number of unread comments for this target |

#### `CommentStatesResponse` — returned by `GET /api/comments/states/`

| Field | Type | Description |
|-------|------|-------------|
| `states` | `Dict[str, CommentStateEntry]` | Map of target key → state+counts. Key is `"summary"` for summary-level, or the chart_id as a string (e.g., `"7"`) for chart-level |

```json
{
  "states": {
    "summary": { "state": "read", "count": 3, "unread_count": 0 },
    "7": { "state": "mentioned", "count": 5, "unread_count": 3 },
    "12": { "state": "unread", "count": 2, "unread_count": 1 }
  }
}
```

#### `MentionableUserResponse` — returned by `GET /api/comments/mentionable-users/`

| Field | Type | Description |
|-------|------|-------------|
| `email` | `str` | User's email |
| `name` | `str \| null` | User's display name |

**`from_orguser(cls, orguser)`** mapping: `email` ← `orguser.user.email`, `name` ← `_get_user_name(orguser)`

```json
{ "email": "noopur@projecttech4dev.org", "name": "Noopur Raval" }
```

### API Response Wrapper

All endpoints wrap data in `ApiResponse`:

```json
{
  "success": true,
  "message": "Comment created",
  "data": { ... }
}
```

Error responses (Django Ninja HttpError):
```json
{
  "detail": "Chart 99 not found in snapshot 42"
}
```

---

## Full API Contract (`ddpui/api/comments_api.py`)

**Router**: `comments_router = Router()` mounted at `/api/comments/` in `routes.py`.

All endpoints require `@has_permission(["can_view_dashboards"])`.

**CRITICAL: Route ordering** — Named routes (`/states/`, `/mentionable-users/`, `/mark-read/`) MUST be registered BEFORE `/{comment_id}/` to avoid 405 errors.

### `GET /api/comments/states/?snapshot_id={N}`

Returns icon states for all targets in a snapshot (for badge rendering).

**Query params**: `snapshot_id` (int, required)

**Response**: `ApiResponse<CommentStatesResponse>`
```json
{
  "success": true,
  "data": {
    "states": {
      "summary": { "state": "read", "count": 0 },
      "7": { "state": "mentioned", "count": 3 }
    }
  }
}
```

**Errors**: 400 if snapshot not found

---

### `GET /api/comments/mentionable-users/`

Returns all org users available for @mention autocomplete.

**Query params**: none

**Response**: `ApiResponse<MentionableUserResponse[]>`
```json
{
  "success": true,
  "data": [
    { "email": "noopur@projecttech4dev.org", "name": "Noopur Raval" },
    { "email": "siddhant@dalgo.in", "name": "Siddhant" },
    { "email": "admin@org.com", "name": null }
  ]
}
```

---

### `POST /api/comments/mark-read/`

Marks a target's comments as read for the current user (upserts `CommentReadStatus`).

**Body**: `MarkReadRequest`
```json
{ "snapshot_id": 42, "target_type": "chart", "chart_id": 7 }
```

**Response**: `ApiResponse` (no data)
```json
{ "success": true, "message": "Marked as read" }
```

**Errors**: 400 if snapshot not found

---

### `GET /api/comments/?snapshot_id={N}&target_type={T}&chart_id={N}`

Lists all non-deleted comments for a target, ordered chronologically by `created_at`.

**Query params**:
- `snapshot_id` (int, required)
- `target_type` (str, required): `"summary"` or `"chart"`
- `chart_id` (int, optional): required when `target_type="chart"`

**Response**: `ApiResponse<CommentResponse[]>`
```json
{
  "success": true,
  "data": [
    {
      "id": 14,
      "target_type": "chart",
      "snapshot_id": 42,
      "chart_id": 7,
      "content": "Great progress here",
      "author": { "email": "admin@org.com", "name": null },
      "is_new": false,
      "created_at": "2026-03-11T08:00:00Z",
      "updated_at": "2026-03-11T08:00:00Z",
      "mentions": []
    },
    {
      "id": 15,
      "target_type": "chart",
      "snapshot_id": 42,
      "chart_id": 7,
      "content": "This number looks off @siddhant@dalgo.in",
      "author": { "email": "noopur@projecttech4dev.org", "name": "Noopur Raval" },
      "is_new": true,
      "created_at": "2026-03-13T10:30:00Z",
      "updated_at": "2026-03-13T10:30:00Z",
      "mentions": [{ "email": "siddhant@dalgo.in", "name": "Siddhant" }]
    }
  ]
}
```

**Errors**: 400 if `chart_id` missing for chart target, or snapshot not found

---

### `POST /api/comments/`

Creates a new comment.

**Body**: `CommentCreate`
```json
{
  "snapshot_id": 42,
  "target_type": "chart",
  "chart_id": 7,
  "content": "This number looks off @siddhant@dalgo.in"
}
```

**Response**: `ApiResponse<CommentResponse>`
```json
{
  "success": true,
  "message": "Comment created",
  "data": {
    "id": 15,
    "target_type": "chart",
    "snapshot_id": 42,
    "chart_id": 7,
    "content": "This number looks off @siddhant@dalgo.in",
    "author": { "email": "noopur@projecttech4dev.org", "name": "Noopur Raval" },
    "is_edited": false,
    "is_deleted": false,
    "is_new": false,
    "created_at": "2026-03-13T10:30:00Z",
    "updated_at": "2026-03-13T10:30:00Z",
    "mentions": [{ "email": "siddhant@dalgo.in", "name": "Siddhant" }]
  }
}
```

**Side effects**:
- `MentionService.process_mentions()` parses @email mentions from content
- Stores mentioned emails in `Comment.mentioned_emails` JSONField
- Creates `Notification` + `NotificationRecipient` records

**Errors**: 400 (invalid target_type, missing chart_id, chart not in snapshot's `frozen_chart_configs`), 500 (unexpected)

---

### `PUT /api/comments/{comment_id}/`

Updates a comment's content. Author-only.

**Path params**: `comment_id` (int)

**Body**: `CommentUpdate`
```json
{ "content": "Updated: this number is correct now" }
```

**Response**: `ApiResponse<CommentResponse>`
```json
{
  "success": true,
  "message": "Comment updated",
  "data": {
    "id": 15,
    "content": "Updated: this number is correct now",
    "..."
  }
}
```

**Side effects**: Clears `mentioned_emails`, then re-runs `MentionService.process_mentions()` on the new content.

**Errors**: 404 (not found or deleted), 403 (not the author)

---

### `DELETE /api/comments/{comment_id}/`

Hard-deletes a comment. Author-only. The row is permanently removed.

**Path params**: `comment_id` (int)

**Body**: none

**Response**: `ApiResponse` (no data)
```json
{ "success": true, "message": "Comment deleted" }
```

**Errors**: 404 (not found), 403 (not the author)

---

## Core Service (`ddpui/core/comments/comment_service.py`)

Static methods on `CommentService` class:

| Method | Purpose |
|--------|---------|
| `list_comments(snapshot_id, org, target_type, chart_id, orguser)` | Flat chronological list with `is_new` annotation based on `CommentReadStatus.last_read_at` |
| `create_comment(snapshot_id, org, orguser, target_type, content, chart_id)` | Validates target, creates comment, delegates to `MentionService.process_mentions()` |
| `update_comment(comment_id, org, orguser, content)` | Author-only, clears and re-processes mentions |
| `delete_comment(comment_id, org, orguser)` | Author-only hard delete |
| `get_comment_states(snapshot_id, org, orguser)` | Returns icon state + unread count per target |
| `mark_as_read(snapshot_id, org, orguser, target_type, chart_id)` | Upserts `CommentReadStatus` with `last_read_at = now()` |
| `get_mentionable_users(org)` | Returns all OrgUsers in the org, ordered by email |

### Icon state computation (`get_comment_states`)

1. Fetches all comments for the snapshot (values: `target_type`, `snapshot_chart_id`, `created_at`, `id`)
2. Fetches all `CommentReadStatus` entries for the user on this snapshot
3. Queries comments where `mentioned_emails__contains=[user_email]` to find mentioned comment IDs
4. Groups comments by target key (`CommentTargetType.SUMMARY` or `str(chart_id)`). Skips entries where `target_type="chart"` but `chart_id is None` (invalid data).
5. For each target:
   - Counts unread comments (`created_at > last_read_at`, or all if no read status)
   - Checks if any unread comment ID is in the mentioned set
6. Returns state priority: `mentioned > unread > read`

### Mention parsing (`MentionService`)

- Regex: `r'@([\w.+-]+@[\w.-]+\.\w+)'`
- Resolves each email to an `OrgUser` in the same org
- Stores matched emails in `Comment.mentioned_emails` JSONField (ignores unknown emails silently)
- Creates `Notification` + `NotificationRecipient` records for each mentioned user

### Exceptions (`ddpui/core/comments/exceptions.py`)

```
CommentError (base)
├── CommentNotFoundError     → 404
├── CommentValidationError   → 400
└── CommentPermissionError   → 403
```

Each has `message` and `error_code` attributes.

---

## Frontend Types (`types/comments.ts`)

```typescript
// Author/mention info
export interface CommentAuthor {
  email: string;
  name?: string;
}

export interface CommentMention {
  email: string;
  name?: string;
}

// Core comment object (matches CommentResponse from backend)
export interface Comment {
  id: number;
  target_type: 'summary' | 'chart';
  snapshot_id: number;
  chart_id?: number;
  content: string;
  author: CommentAuthor;
  is_new: boolean;
  created_at: string;
  updated_at: string;
  mentions: CommentMention[];
}

// Icon state for badge rendering
export type CommentIconState = 'none' | 'unread' | 'read' | 'mentioned';

// Per-target state entry
export interface CommentStateEntry {
  state: CommentIconState;
  count: number;        // total number of comments
  unread_count: number; // number of unread comments
}

// Map of all target states for a snapshot
export interface CommentStates {
  [key: string]: CommentStateEntry; // "summary" or chart_id string → { state, count, unread_count }
}

// Mentionable user for @mention dropdown
export interface MentionableUser {
  email: string;
  name?: string;
}

// Mutation payloads
export interface CreateCommentPayload {
  snapshot_id: number;
  target_type: 'summary' | 'chart';
  chart_id?: number;
  content: string;
}

export interface MarkReadPayload {
  snapshot_id: number;
  target_type: 'summary' | 'chart';
  chart_id?: number;
}
```

---

## Frontend API Hooks (`hooks/api/useComments.ts`)

### SWR Read Hooks

```typescript
// List comments for a target (conditional — only fetches when popover is open)
useComments(snapshotId: number | null, targetType, chartId?)
// SWR key: /api/comments/?snapshot_id=N&target_type=T&chart_id=N (null key when closed)
// Returns: { comments: Comment[], isLoading, isError, mutate }

// Get icon states for all targets in a snapshot
useCommentStates(snapshotId: number | null)
// SWR key: /api/comments/states/?snapshot_id=N
// Returns: { states: CommentStates, isLoading, isError, mutate }

// Get all org users for @mention autocomplete
useMentionableUsers()
// SWR key: /api/comments/mentionable-users/
// Returns: { users: MentionableUser[], isLoading, isError }
```

All hooks use `revalidateOnFocus: false`.

### Mutation Functions

```typescript
// POST /api/comments/
createComment(payload: CreateCommentPayload): Promise<Comment>

// PUT /api/comments/{commentId}/
updateComment(commentId: number, content: string): Promise<Comment>

// DELETE /api/comments/{commentId}/
deleteComment(commentId: number): Promise<void>

// POST /api/comments/mark-read/
markAsRead(payload: MarkReadPayload): Promise<void>
```

### Frontend API Response Wrapper

The hooks expect all backend responses wrapped in:
```typescript
interface ApiResponse<T> {
  success: boolean;
  data: T;
  message?: string;
}
```

SWR hooks extract `data?.data` (the inner payload) when returning values.

---

## Frontend Utilities (`components/reports/utils.ts`)

| Function | Purpose |
|----------|---------|
| `formatCommentTime(dateStr)` | Short relative timestamps: "just now", "10 mins. ago", "3 hrs. ago", "2 days ago", "1 week ago" |
| `getAvatarColor(email)` | Generates consistent color from email hash (31-multiply hash). 10-color palette: `#4285F4` blue, `#34A853` green, `#EA4335` red, `#FBBC04` yellow, `#E91E63` pink, `#9C27B0` purple, `#FF5722` deep orange, `#00BCD4` cyan, `#795548` brown, `#607D8B` blue grey |
| `getInitials(author)` | First letters of name words (max 2), or first letter of email |
| `parseCommentMentions(content)` | Regex `/@([\w.+-]+@[\w.-]+\.\w+)/g` splits text into `{type: 'text' \| 'mention', value}` array for rendering |
| `formatCreatedOn(dateStr)` | Report list table timestamps (different format, not used by comments) |
| `formatDateShort(dateStr)` | MM/DD/YYYY format for report headers |

---

## Frontend Components

### CommentIcon (`components/reports/comment-icon.tsx`)

Renders the trigger icon based on `CommentIconState` and `unreadCount`:

| State | Icon | Indicator |
|-------|------|-----------|
| `none` | `MessageCircle` outline | None |
| `read` | `MessageCircle` outline | Small outline dot (top-right, `border-muted-foreground`) — only shown when no unread badge |
| `unread` | `MessageCircle` outline | `bg-rose-500` count badge showing `unreadCount` |
| `mentioned` | `AtSign` icon | `bg-rose-500` count badge showing `unreadCount` |

**Props**: `state: CommentIconState`, `unreadCount?: number` (default 0), `className?: string`

Badge: `min-w-[16px] h-4 rounded-full bg-rose-500 text-[10px] text-white`. Shows `99+` for count > 99. Only shown when `unreadCount > 0`.
Icons: `h-4 w-4`.

### CommentPopover (`components/reports/comment-popover.tsx`)

Radix `Popover` with Figma-based redesign.

**Props**:
```typescript
interface CommentPopoverProps {
  snapshotId: number;
  targetType: 'summary' | 'chart';
  chartId?: number;
  state: CommentIconState;
  unreadCount?: number;
  triggerClassName?: string;
  onStateChange?: () => void;
}
```

**Design**:
- **No header** — comments flow directly at the top of the popover
- **Colored avatars** — each user gets a unique background color (from `getAvatarColor`) with white initials
- **Email displayed** — comment author shown as email address
- **Short timestamps** — "just now", "5 mins. ago", "2 days ago" via `formatCommentTime`
- **@mentions as styled text** — `CommentContent` component renders mentions in `text-primary font-medium`
- **Ellipsis menu** — author's own comments show `...` (`MoreHorizontal`) on hover (`group-hover:opacity-100`), opening a `DropdownMenu` with Edit and Delete
- **Single-line input** — plain `<input type="text">` with "Add a comment" placeholder
- **Circular send button** — `ArrowUp` icon in round button; `bg-muted` when empty, `bg-primary text-white` when text entered
- **Enter to send** — single Enter key submits
- **@mention dropdown** — positioned above input (`absolute bottom-full`), shows colored avatars per user
- **Popover width** — `w-80` (320px), max height `min(450px, 80vh)`

**Sub-components** (all `memo`'d):
- `MentionDropdown` — filters users by typed query, positioned absolutely above input
- `CommentContent` — parses @mentions into blue styled spans via `parseCommentMentions`
- `CommentItem` — renders single comment with avatar, email, timestamp, content, hover ellipsis menu

**Behavior**:
- Comments fetched via `useComments` only when popover is open (conditional SWR key: `open ? snapshotId : null`)
- Auto-scrolls to first unread comment on open (100ms delay)
- `markAsRead` called on close, then `onStateChange()` to refresh parent's icon states
- Edit mode: "Editing comment" indicator with cancel (X) button, input pre-filled with existing content
- New comment highlight: `bg-yellow-50 dark:bg-yellow-950/30` for comments where `is_new === true`

---

## Integration Points

### Executive Summary section (`app/reports/[snapshotId]/page.tsx`)

The comment popover is placed next to the "Executive Summary" heading inside the `beforeContent` section (not in the report header).

```tsx
beforeContent={
  <div className="border rounded-lg p-5 mb-2 bg-background">
    <div className="flex items-center justify-between mb-2">
      <h2 className="text-lg font-semibold">Executive Summary</h2>
      <CommentPopover
        snapshotId={snapshotId}
        targetType="summary"
        state={commentStates?.['summary']?.state ?? 'none'}
        count={commentStates?.['summary']?.count ?? 0}
        unreadCount={commentStates?.['summary']?.unread_count ?? 0}
        triggerClassName="h-8 w-8"
        onStateChange={handleCommentStateChange}
      />
    </div>
    <Textarea ... />
  </div>
}
```

### Chart cards (`components/dashboard/chart-element-view.tsx`)

```tsx
{frozenChartConfig && snapshotId && (
  <CommentPopover
    snapshotId={snapshotId}
    targetType="chart"
    chartId={chartId}
    state={(commentStates?.[String(chartId)]?.state as CommentIconState) ?? 'none'}
    count={commentStates?.[String(chartId)]?.count ?? 0}
    unreadCount={commentStates?.[String(chartId)]?.unread_count ?? 0}
    triggerClassName="h-7 w-7 p-0"
    onStateChange={onCommentStateChange}
  />
)}
```

### State management flow

1. `useCommentStates(snapshotId)` fetches all icon states at report page level
2. States passed down via props: `page` → `DashboardNativeView` → `ChartElementView`
3. After any comment action (create/edit/delete/close popover), `onStateChange()` calls `mutateCommentStates()` to refresh all badges

---

## Figma Design Decisions

1. **Colored avatars** — not gray; each user gets a stable unique color from a 10-color palette based on email hash
2. **Hover-reveal "..." menu** — Edit/Delete hidden by default, appear on hover via `group`/`group-hover:opacity-100`, uses Radix DropdownMenu
3. **Single-line input** — simpler than a textarea, Enter to send (not Cmd+Enter)
4. **Circular send button** — round, gray when empty, teal/primary when text entered, uses ArrowUp icon
5. **No popover header** — no "Comments" title bar, comments flow directly
6. **Rose-500 badge** — count badge and unread dot are red/pink (`bg-rose-500`), not the primary brand color
7. **@mentions rendered inline** — @email in comment text shown as `text-primary font-medium`
8. **Email as author label** — shows email address, not name (consistent with Figma)
9. **Short relative timestamps** — "just now", "10 mins. ago", "3 hrs. ago", "2 days ago"

---

## Gotchas & Lessons Learned

### 1. Django Ninja Route Ordering
Static path endpoints (`/states/`, `/mentionable-users/`, `/mark-read/`) MUST be registered before dynamic `/{comment_id}/`. Otherwise Django Ninja tries to parse "states" as an integer → 405 error.

### 2. Auth Store for Current User
`useAuthStore((s) => s.getCurrentOrgUser()?.email ?? '')` is the correct way to get the current user's email. Do NOT use `localStorage.getItem('dalgo-auth')` — the auth store uses `persist: false` with cookie-based auth, so that key doesn't exist in localStorage.

### 3. Conditional SWR Fetching
Pass `null` as the snapshot ID to `useComments` when the popover is closed: `useComments(open ? snapshotId : null, ...)`. This prevents unnecessary API calls.

### 4. Popover Interaction with Dropdowns
The `onInteractOutside` handler on `PopoverContent` prevents closing when clicking inside the mention dropdown by checking `target.closest('[data-testid="mention-dropdown"]')`.

### 5. Hard Delete
Comments are permanently removed from the database on delete. No soft-delete mechanism.

### 6. Multi-Tenant Isolation
Every backend query includes `org=org`. The org comes from `request.orguser.org` in the API layer.

### 7. Chart ID Validation
When creating a chart comment, the service validates that `str(chart_id)` exists as a key in the snapshot's `frozen_chart_configs` dict.

### 8. Read Status Per Target
`CommentReadStatus` tracks `last_read_at` per `(user, snapshot, target_type, chart_id)`. Read state is independent between summary-level comments and each chart's comments.

### 9. Cookie Auth for Local Dev
For local development with cookie-based auth, these settings must be correct in `DDP_backend/ddpui/settings.py`:
- `COOKIE_SECURE = False` (browsers reject `Secure` cookies on HTTP)
- `COOKIE_SAMESITE = "Lax"` (`None` requires `Secure=True`)
- Frontend `NEXT_PUBLIC_BACKEND_URL` must use `http://localhost:8002` (NOT `127.0.0.1` — different origins break cookie sending)

### 10. Application-Level Target Type Enum
`CommentTargetType` is a plain Python class, NOT a Django enum or model `choices`. This means adding new target types (e.g., `"kpi"`) requires NO migration — just add to the class and update the service layer validation.

### 11. Null Chart ID Guard in get_comment_states
When grouping comments by target in `get_comment_states`, always check `chart_id is not None` before converting to string key. Old or invalid data may have `target_type="chart"` with `snapshot_chart_id=None`, which would cause `int("None")` crash.

---

## File Inventory

### Backend — Created
| File | Purpose |
|------|---------|
| `ddpui/models/comment.py` | Comment, CommentReadStatus models |
| `ddpui/core/comments/__init__.py` | Public exports |
| `ddpui/core/comments/comment_service.py` | CRUD, icon states, mark-as-read, mentionable users |
| `ddpui/core/comments/mention_service.py` | @mention parsing, notification dispatch |
| `ddpui/core/comments/exceptions.py` | CommentError hierarchy |
| `ddpui/schemas/comment_schema.py` | Request/response Pydantic schemas |
| `ddpui/api/comments_api.py` | HTTP endpoints with permission checks |

### Backend — Modified
| File | Change |
|------|--------|
| `ddpui/routes.py` | Added `comments_router` mount at `/api/comments/` |

### Frontend — Created
| File | Purpose |
|------|---------|
| `types/comments.ts` | TypeScript interfaces for Comment, states, mentions, payloads |
| `hooks/api/useComments.ts` | SWR hooks + mutation functions |
| `components/reports/comment-popover.tsx` | Comment popover with @mentions, colored avatars, ellipsis menu |
| `components/reports/comment-icon.tsx` | Icon with state-based rendering and rose-500 badge |

### Frontend — Modified
| File | Change |
|------|--------|
| `components/reports/utils.ts` | Added `formatCommentTime`, `getAvatarColor`, `getInitials`, `parseCommentMentions` |
| `components/dashboard/chart-element-view.tsx` | Added `<CommentPopover>` to chart toolbar (report mode only) |
| `app/reports/[snapshotId]/page.tsx` | Added summary-level `<CommentPopover>` in Executive Summary `beforeContent` section, `useCommentStates` hook |
| `components/dashboard/dashboard-native-view.tsx` | Passes `snapshotId`, `commentStates`, `onCommentStateChange` to chart views |

---

## Migration

A single consolidated migration `0154_commentreadstatus_comment_and_more.py` creates both `Comment` and `CommentReadStatus` tables. It has no `choices=` on `target_type` fields (matching the model). Dependencies: `("ddpui", "0153_merge_20260314_1816")`.

**History**: Originally there were 3 separate migrations (0154, 0155, 0156) that were consolidated into this single migration during development.

---

## @Mention Notification Flow

When a user is @mentioned in a comment, here's the full end-to-end notification flow:

### 1. Comment Created → Mentions Parsed (`MentionService.process_mentions`)
- Extracts email patterns using regex: `@([\w.+-]+@[\w.-]+\.\w+)`
- Resolves each email to an `OrgUser` in the same org
- Stores matched emails in `Comment.mentioned_emails` JSONField
- Ignores unknown emails silently

### 2. Notification Created & Delivered (`MentionService.send_mention_notifications`)
- Creates a `Notification` record with message like *"user@example.com mentioned you in a comment..."*
- Creates `NotificationRecipient` records for each mentioned user (excluding the author)
- **Email via AWS SES**: Sent immediately (synchronous) if `UserPreferences.enable_email_notifications` is enabled
- **Discord webhook**: Sent if `OrgPreferences.enable_discord_notifications` is enabled and webhook URL is configured

### 3. Frontend Notification Display
- Frontend polls `GET /api/notifications/unread_count` every 30 seconds via SWR (`useUnreadCount` hook)
- Users see notifications in the **Notifications page** (`/notifications`)
- The comment icon changes to `AtSign` with "mentioned" state when there are unread mentions

### Key Backend Files for Notifications
| File | Purpose |
|------|---------|
| `ddpui/core/comments/mention_service.py` | @mention parsing, notification dispatch |
| `ddpui/core/notifications/notifications_functions.py` | Notification creation & email/discord delivery |
| `ddpui/models/notifications.py` | `Notification` and `NotificationRecipient` models |
| `ddpui/api/notifications_api.py` | Notification list, unread count, mark-read endpoints |

### Key Frontend Files for Notifications
| File | Purpose |
|------|---------|
| `hooks/api/useNotifications.ts` | SWR hooks for notifications + unread count (30s polling) |
| `types/notifications.ts` | TypeScript interfaces |
| `components/notifications/NotificationsList.tsx` | Notification list with pagination |
| `components/notifications/NotificationRow.tsx` | Individual notification display |

### Limitations
- **No real-time push**: Uses 30-second polling, not WebSockets
- **Email delivery is synchronous**: Happens immediately when comment is created (not queued via Celery)
- **Email-based mentions only**: Must use `@user@example.com` format, not `@username`
- **No per-feature notification preferences**: Only global email/discord toggle

---

## Potential Improvements

These are identified but not yet implemented:

1. **Delete confirmation** — currently deletes instantly on click, should show AlertDialog to prevent accidental deletions
2. **Auto-scroll after posting** — scroll to bottom after creating a new comment so the user sees it appear
3. **Loading skeleton on first open** — show a spinner/skeleton while SWR fetches comments, instead of briefly showing "No comments yet"
4. **Prefer name over email** — show `author.name || author.email` instead of always email (current implementation shows email to match Figma, but this may change)
5. **Keyboard navigation for @mentions** — arrow keys to navigate mention dropdown, Enter to select
6. **Optimistic UI updates** — immediately append new comment to list before SWR revalidation completes
7. **Character limit indicator** — show remaining characters near limit, or enforce max-length on input
8. **Send button tooltip** — tooltip "Send (Enter)" on the circular send button for discoverability
