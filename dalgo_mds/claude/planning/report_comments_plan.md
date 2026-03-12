# Report Comments Feature - Implementation Plan

## Context

Dalgo reports (frozen dashboard snapshots) currently have no collaboration mechanism. Users need the ability to discuss and annotate individual charts and the overall report. This plan adds **chart-level comments** and **report-level comments** on report snapshots, with threaded replies, @mentions, and email notifications. Scoped to reports only (not live dashboards).

---

## Design Summary

```
Report Snapshot Viewer
┌─────────────────────────────────────────────────────────┐
│  Header:  Title  |  Download  Share  💬(3)  Save        │  ← Report-level comments
│  Period: Jan-Mar  |  Created by: user                   │
├─────────────────────────────────────────────────────────┤
│  Executive Summary                                       │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐             │
│  │   Chart A         │  │   Chart B         │             │
│  │          💬(2) ⬇ ⬜│  │          💬(0) ⬇ ⬜│             │  ← Chart-level comments
│  └──────────────────┘  └──────────────────┘             │
└─────────────────────────────────────────────────────────┘

Click 💬 → Popover with threaded comments + @mention autocomplete
```

**Target type mapping:**
- `target_type="report"` + `snapshot_id` → Report-level comment
- `target_type="chart"` + `snapshot_id` + `chart_id` → Chart-level comment within that report

---

## 1. Backend

### 1.1 Model — `DDP_backend/ddpui/models/comment.py`

```python
class Comment(models.Model):
    id = models.BigAutoField(primary_key=True)

    # Target: report-level or chart-level within a report
    target_type = models.CharField(max_length=20, choices=[("report","Report"),("chart","Chart")])
    snapshot = models.ForeignKey(ReportSnapshot, on_delete=models.CASCADE, related_name="comments")
    chart_id = models.IntegerField(null=True, blank=True)  # Integer, NOT FK (charts are frozen in snapshots)

    content = models.TextField()  # max 5000 chars enforced at schema level

    # Threading
    parent_comment = models.ForeignKey('self', on_delete=models.CASCADE, null=True, blank=True, related_name="replies")

    # Author + multi-tenancy
    author = models.ForeignKey(OrgUser, on_delete=models.CASCADE, related_name="comments_authored")
    org = models.ForeignKey(Org, on_delete=models.CASCADE)

    # State
    is_edited = models.BooleanField(default=False)
    is_deleted = models.BooleanField(default=False)  # soft delete
    deleted_at = models.DateTimeField(null=True, blank=True)

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

class CommentMention(models.Model):
    id = models.BigAutoField(primary_key=True)
    comment = models.ForeignKey(Comment, on_delete=models.CASCADE, related_name="mentions")
    mentioned_user = models.ForeignKey(OrgUser, on_delete=models.CASCADE, related_name="comment_mentions")
    created_at = models.DateTimeField(auto_now_add=True)
```

**Key decision**: `chart_id` is an integer (not FK to Chart) because report snapshots store frozen chart configs and don't reference live Chart records. The chart_id maps to keys in `frozen_chart_configs`.

### 1.2 Exceptions — `DDP_backend/ddpui/core/comments/exceptions.py`

Standard pattern: `CommentError` (base), `CommentNotFoundError`, `CommentValidationError`, `CommentPermissionError`

### 1.3 Service — `DDP_backend/ddpui/core/comments/comment_service.py`

`CommentService` static methods:

| Method | Description |
|--------|-------------|
| `list_comments(snapshot_id, org, target_type, chart_id=None)` | Top-level comments with nested replies |
| `create_comment(data, orguser)` | Create + parse mentions + notify |
| `update_comment(comment_id, org, orguser, content)` | Author-only, sets `is_edited=True` |
| `delete_comment(comment_id, org, orguser)` | Author-only soft delete |
| `get_comment_counts(snapshot_id, org)` | Dict of `{chart_id: count, "report": count}` for badge numbers |

### 1.4 Mention Service — `DDP_backend/ddpui/core/comments/mention_service.py`

| Method | Description |
|--------|-------------|
| `parse_mentions(content, org)` | Extract `@email` patterns, resolve to OrgUser |
| `create_mention_records(comment, users)` | Create CommentMention rows |
| `send_mention_notifications(comment, mentioned_users, author)` | Create Notification via existing system |
| `send_reply_notification(comment, author)` | Notify parent comment author |

Uses existing `Notification` + `NotificationRecipient` models from `ddpui/models/notifications.py` and `ddpui/core/notifications/notifications_functions.py`.

### 1.5 Schemas — `DDP_backend/ddpui/schemas/comment_schema.py`

**Request:**
- `CommentCreate(Schema)`: `target_type`, `snapshot_id`, `chart_id` (optional), `content` (max 5000), `parent_comment_id` (optional)
- `CommentUpdate(Schema)`: `content` (max 5000)

**Response:**
- `CommentAuthorResponse`: `email`, `name`
- `CommentResponse`: `id`, `target_type`, `snapshot_id`, `chart_id`, `content`, `author`, `parent_comment_id`, `is_edited`, `is_deleted`, `created_at`, `updated_at`, `mentions[]`, `replies[]`, `reply_count` + `from_model()` classmethod
- `CommentCountResponse`: `counts: dict` mapping chart_id (or "report") to count

### 1.6 API — `DDP_backend/ddpui/api/comments_api.py`

Router: `comments_router = Router()`, mounted at `/api/comments/` in `routes.py`

| Endpoint | Method | Permission | Description |
|----------|--------|------------|-------------|
| `/` | GET | `can_view_dashboards` | List comments (query: `snapshot_id`, `target_type`, `chart_id`) |
| `/` | POST | `can_view_dashboards` | Create comment |
| `/{comment_id}/` | PUT | `can_view_dashboards` | Update (author-only in service) |
| `/{comment_id}/` | DELETE | `can_view_dashboards` | Soft-delete (author-only in service) |
| `/counts/` | GET | `can_view_dashboards` | Comment counts per chart + report for a snapshot |
| `/mentionable-users/` | GET | `can_view_dashboards` | Org users for @mention autocomplete |

Uses `api_response()` wrapper, `@has_permission`, and exception-to-HttpError conversion per CLAUDE.md patterns.

### 1.7 Celery Task

Add `send_comment_notification_email` to `DDP_backend/ddpui/celeryworkers/tasks.py`. Receives notification_id, loads comment data, builds email with link to report, sends via Django `send_mail`.

### 1.8 Route Registration

Modify `DDP_backend/ddpui/routes.py`:
```python
from ddpui.api.comments_api import comments_router
src_api.add_router("/api/comments/", comments_router)
comments_router.tags = ["Comments"]
```

---

## 2. Frontend

### 2.1 Types — `webapp_v2/types/comments.ts`

```typescript
interface Comment {
  id: number;
  target_type: 'report' | 'chart';
  snapshot_id: number;
  chart_id?: number;
  content: string;
  author: { email: string; name?: string };
  parent_comment_id?: number;
  is_edited: boolean;
  is_deleted: boolean;
  created_at: string;
  updated_at: string;
  mentions: Array<{ email: string; name?: string }>;
  replies: Comment[];
  reply_count: number;
}

interface CommentCounts {
  [key: string]: number;  // chart_id or "report" → count
}

interface MentionableUser {
  email: string;
  name?: string;
}
```

### 2.2 API Hooks — `webapp_v2/hooks/api/useComments.ts`

SWR hooks (following `useReports.ts` pattern):
- `useComments(snapshotId, targetType, chartId?)` — list comments for a target
- `useCommentCounts(snapshotId)` — all comment counts for a snapshot (badges)
- `useMentionableUsers()` — org users for autocomplete

Mutation functions:
- `createComment(payload)` → `apiPost('/api/comments/', payload)`
- `updateComment(commentId, content)` → `apiPut('/api/comments/{id}/', {content})`
- `deleteComment(commentId)` → `apiDelete('/api/comments/{id}/')`

### 2.3 Comment Popover — `webapp_v2/components/reports/comment-popover.tsx`

Radix `Popover` component reusing existing `components/ui/popover.tsx`:

**Props:**
```typescript
interface CommentPopoverProps {
  snapshotId: number;
  targetType: 'report' | 'chart';
  chartId?: number;
  triggerClassName?: string;
}
```

**Structure:**
1. **Trigger**: Ghost button with `MessageCircle` icon + badge count (from `useCommentCounts`)
2. **Content** (max-h 500px):
   - Header: "Comments" + count
   - ScrollArea with threaded comment list
   - Each comment: Avatar initials + author + timestamp + content + Reply/Edit/Delete
   - Replies: indented `ml-8`, collapsible if > 3
   - Input: Textarea + Send button (Cmd+Enter shortcut)
   - Reply indicator: "Replying to..." with cancel
3. **@mention**: On `@` keypress, show filtered dropdown of `useMentionableUsers()`. Select inserts `@email`.

Uses existing UI components: `Popover`, `Button`, `Badge`, `ScrollArea`, `Avatar`, `Textarea` from `components/ui/`.

### 2.4 Integration Points

#### Chart toolbar — `webapp_v2/components/dashboard/chart-element-view.tsx`

Add `CommentPopover` to the floating hover toolbar (alongside Download + Fullscreen buttons). Only show when in report mode (detect from `frozenChartConfig` presence).

```tsx
// Inside the toolbar div (line ~1703)
{frozenChartConfig && snapshotId && (
  <CommentPopover
    snapshotId={snapshotId}
    targetType="chart"
    chartId={chartId}
    triggerClassName="h-7 w-7 p-0"
  />
)}
```

**Requires**: Threading `snapshotId` from report page → `DashboardNativeView` → `ChartElementView`. Add `snapshotId?: number` prop to both components.

#### Report header — `webapp_v2/app/reports/[snapshotId]/page.tsx`

Add report-level comment button in the header actions area (alongside Download, Share, Save):

```tsx
// In the actions div (line ~227)
<CommentPopover
  snapshotId={snapshotId}
  targetType="report"
  triggerClassName="h-9 w-9"
/>
```

---

## 3. Files to Create

| # | File | Layer |
|---|------|-------|
| 1 | `DDP_backend/ddpui/models/comment.py` | Backend model |
| 2 | `DDP_backend/ddpui/core/comments/__init__.py` | Backend core |
| 3 | `DDP_backend/ddpui/core/comments/comment_service.py` | Backend service |
| 4 | `DDP_backend/ddpui/core/comments/mention_service.py` | Backend mentions |
| 5 | `DDP_backend/ddpui/core/comments/exceptions.py` | Backend exceptions |
| 6 | `DDP_backend/ddpui/schemas/comment_schema.py` | Backend schemas |
| 7 | `DDP_backend/ddpui/api/comments_api.py` | Backend API |
| 8 | `webapp_v2/types/comments.ts` | Frontend types |
| 9 | `webapp_v2/hooks/api/useComments.ts` | Frontend hooks |
| 10 | `webapp_v2/components/reports/comment-popover.tsx` | Frontend component |

## 4. Files to Modify

| # | File | Change |
|---|------|--------|
| 1 | `DDP_backend/ddpui/routes.py` | Add `comments_router` |
| 2 | `DDP_backend/ddpui/celeryworkers/tasks.py` | Add email notification task |
| 3 | `webapp_v2/components/dashboard/chart-element-view.tsx` | Add CommentPopover to toolbar + accept `snapshotId` prop |
| 4 | `webapp_v2/components/dashboard/dashboard-native-view.tsx` | Pass `snapshotId` through to ChartElementView |
| 5 | `webapp_v2/app/reports/[snapshotId]/page.tsx` | Add report-level CommentPopover in header + pass snapshotId to DashboardNativeView |

---

## 5. Implementation Order

### Phase 1: Backend Core (do first)
1. Create `models/comment.py` (Comment + CommentMention)
2. Run `makemigrations` + `migrate`
3. Create `core/comments/exceptions.py`
4. Create `core/comments/comment_service.py`
5. Create `core/comments/mention_service.py`
6. Create `schemas/comment_schema.py`
7. Create `api/comments_api.py`
8. Register router in `routes.py`
9. Add Celery email task

### Phase 2: Backend Tests
10. Create `tests/core/test_comment_service.py`
11. Create `tests/api/test_comments_api.py`

### Phase 3: Frontend
12. Create `types/comments.ts`
13. Create `hooks/api/useComments.ts`
14. Create `components/reports/comment-popover.tsx`
15. Modify `chart-element-view.tsx` — add CommentPopover + snapshotId prop
16. Modify `dashboard-native-view.tsx` — thread snapshotId to charts
17. Modify `reports/[snapshotId]/page.tsx` — add report-level comments + pass snapshotId

### Phase 4: Frontend Tests
18. Create `components/reports/__tests__/comment-popover.test.tsx`

---

## 6. Key Patterns to Reuse

| Pattern | Source |
|---------|--------|
| API response wrapper | `DDP_backend/ddpui/utils/response_wrapper.py` → `api_response()` |
| Permission decorator | `DDP_backend/ddpui/auth.py` → `@has_permission()` |
| Core module structure | `DDP_backend/ddpui/core/charts/` |
| Schema with `from_model()` | `DDP_backend/ddpui/schemas/dashboard_schema.py` |
| Router registration | `DDP_backend/ddpui/routes.py` |
| Notification system | `DDP_backend/ddpui/core/notifications/notifications_functions.py` |
| SWR hooks | `webapp_v2/hooks/api/useReports.ts` |
| API client | `webapp_v2/lib/api.ts` → `apiGet, apiPost, apiPut, apiDelete` |
| Popover UI | `webapp_v2/components/ui/popover.tsx` |
| Chart toolbar | `webapp_v2/components/dashboard/chart-element-view.tsx:1700-1736` |
| Toast notifications | `webapp_v2/lib/toast.ts` → `toastSuccess, toastError` |
| Report header actions | `webapp_v2/app/reports/[snapshotId]/page.tsx:227-254` |

---

## 7. Verification

### Backend
```bash
cd DDP_backend
python manage.py makemigrations && python manage.py migrate
uv run pytest ddpui/tests/core/test_comment_service.py -v
uv run pytest ddpui/tests/api/test_comments_api.py -v
```

Then test via API docs at `http://localhost:8002/api/docs`:
- POST `/api/comments/` with `target_type=report`, `snapshot_id=<id>`
- POST `/api/comments/` with `target_type=chart`, `snapshot_id=<id>`, `chart_id=<id>`
- GET `/api/comments/?snapshot_id=<id>&target_type=chart&chart_id=<id>`
- GET `/api/comments/counts/?snapshot_id=<id>`

### Frontend
```bash
cd webapp_v2
npm test -- components/reports/__tests__/comment-popover.test.tsx
npm run dev
```

Then manually:
1. Navigate to a report → verify 💬 icon in header for report-level comments
2. Hover over a chart → verify 💬 icon in floating toolbar
3. Click 💬 → popover opens with empty state
4. Create a comment → appears in list
5. Reply → threaded under parent
6. @mention a user → notification in bell icon
7. Edit/delete own comment → works
8. Open public report URL → comments NOT visible (no CommentPopover rendered)
