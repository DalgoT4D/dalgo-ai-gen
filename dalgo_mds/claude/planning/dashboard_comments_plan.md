# Dashboard & Chart Comments Feature - Implementation Plan

## Context

Dalgo dashboards currently have no way for org users to collaborate, discuss, or annotate charts and dashboards. Users need the ability to leave comments on individual chart components within dashboards and on the overall dashboard itself. Comments should support threaded replies, @mentions with email notifications, and be restricted to authenticated org users only (not visible on public dashboards).

---

## High-Level Design

```
┌──────────────────────────────────────────────────────────┐
│                    Dashboard View                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │  Chart A  │  │  Chart B  │  │  Dashboard Actions   │   │
│  │  💬 (2)  │  │  💬 (0)  │  │  💬 Dashboard (5)    │   │
│  └──────────┘  └──────────┘  └──────────────────────┘   │
│                                                          │
│  Click 💬 → Popover with threaded comments               │
│  @mention → Notification (in-app + email via Celery)     │
└──────────────────────────────────────────────────────────┘
```

**Services affected**: DDP_backend (Django), webapp_v2 (Next.js)
**Not affected**: prefect-proxy, Airbyte, app/

---

## 1. Backend Implementation

### 1.1 Data Models

**New file**: `DDP_backend/ddpui/models/comment.py`

```python
class Comment(models.Model):
    id = models.BigAutoField(primary_key=True)

    # Polymorphic target
    target_type = models.CharField(max_length=20, choices=[("dashboard","Dashboard"),("chart","Chart")])
    dashboard = models.ForeignKey(Dashboard, on_delete=models.CASCADE, null=True, blank=True, related_name="comments")
    chart = models.ForeignKey(Chart, on_delete=models.CASCADE, null=True, blank=True, related_name="comments")

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

    class Meta:
        db_table = "comment"
        ordering = ["created_at"]
        indexes = [
            models.Index(fields=["dashboard", "created_at"]),
            models.Index(fields=["chart", "created_at"]),
            models.Index(fields=["parent_comment", "created_at"]),
            models.Index(fields=["org", "created_at"]),
        ]


class CommentMention(models.Model):
    id = models.BigAutoField(primary_key=True)
    comment = models.ForeignKey(Comment, on_delete=models.CASCADE, related_name="mentions")
    mentioned_user = models.ForeignKey(OrgUser, on_delete=models.CASCADE, related_name="comment_mentions")
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = "comment_mention"
        unique_together = [["comment", "mentioned_user"]]
```

**For notifications**: Reuse the existing `Notification` + `NotificationRecipient` models at `DDP_backend/ddpui/models/notifications.py`. Create notifications through the existing `ddpui/core/notifications/notifications_functions.py` system rather than a separate CommentNotification model. This avoids duplicating notification infrastructure.

### 1.2 Core Module

**New directory**: `DDP_backend/ddpui/core/comments/`

```
ddpui/core/comments/
├── __init__.py
├── comment_service.py      # CRUD, listing, threading logic
├── mention_service.py      # @mention parsing, notification dispatch
└── exceptions.py           # CommentError, CommentNotFoundError, etc.
```

**`exceptions.py`** - Follow pattern from `ddpui/core/charts/` (no exceptions.py file exists there yet, but follow the CLAUDE.md template):

```python
class CommentError(Exception):
    def __init__(self, message, error_code="COMMENT_ERROR"):
        self.message = message
        self.error_code = error_code
        super().__init__(self.message)

class CommentNotFoundError(CommentError): ...
class CommentValidationError(CommentError): ...
class CommentPermissionError(CommentError): ...
```

**`comment_service.py`** key methods:

| Method | Description |
|--------|-------------|
| `list_comments(target_type, target_id, org)` | List top-level comments with nested replies (max 5 depth) |
| `get_comment(comment_id, org)` | Get single comment with replies |
| `create_comment(data, orguser)` | Create comment, parse mentions, create notifications |
| `update_comment(comment_id, org, orguser, content)` | Update (author-only), re-parse mentions |
| `delete_comment(comment_id, org, orguser)` | Soft-delete (author-only) |
| `get_comment_counts(target_type, target_ids, org)` | Batch get comment counts for multiple targets |

**`mention_service.py`** key methods:

| Method | Description |
|--------|-------------|
| `parse_mentions(content, org)` | Extract @email patterns, resolve to OrgUser records |
| `create_mention_records(comment, mentioned_users)` | Create CommentMention entries |
| `send_mention_notifications(comment, mentioned_users, author)` | Create Notification + send email via Celery |
| `send_reply_notification(comment, author)` | Notify parent comment author |

**Mention parsing regex**: `r'@([\w.+-]+@[\w.-]+\.\w+)'` (matches full email addresses like `@john@example.com`)

**Notification dispatch**: Use existing Celery infrastructure. The existing notification system (`ddpui/core/notifications/notifications_functions.py`) already handles creating notifications with recipients and has email sending capability. We'll create notifications through that system with a comment-specific message format.

### 1.3 Schemas

**New file**: `DDP_backend/ddpui/schemas/comment_schema.py`

Follow the pattern from `ddpui/schemas/dashboard_schema.py` and `ddpui/schemas/chart_schema.py`.

**Request schemas**:
- `CommentCreate(Schema)` - target_type, target_id, content (max 5000), parent_comment_id (optional)
- `CommentUpdate(Schema)` - content (max 5000)

**Response schemas**:
- `CommentAuthorResponse(Schema)` - email, name (from User.first_name + last_name)
- `CommentResponse(Schema)` - id, target_type, target_id, content, author, parent_comment_id, is_edited, is_deleted, created_at, updated_at, mentions[], replies[], reply_count. Include `from_model()` classmethod.
- `CommentListResponse(Schema)` - data[], total
- `CommentCountResponse(Schema)` - counts: dict[int, int] (target_id → count)

### 1.4 API Endpoints

**New file**: `DDP_backend/ddpui/api/comments_api.py`

**Router**: `comments_router = Router(tags=["Comments"])`
**Mount at**: `/api/comments/` in `ddpui/routes.py`

| Endpoint | Method | Permission | Description |
|----------|--------|------------|-------------|
| `/` | GET | `can_view_dashboards` | List comments (query: target_type, target_id) |
| `/{comment_id}/` | GET | `can_view_dashboards` | Get single comment |
| `/` | POST | `can_view_dashboards` | Create comment |
| `/{comment_id}/` | PUT | `can_view_dashboards` | Update comment (author-only enforced in service) |
| `/{comment_id}/` | DELETE | `can_view_dashboards` | Soft-delete comment (author-only enforced in service) |
| `/counts/` | GET | `can_view_dashboards` | Get comment counts for multiple targets (query: target_type, target_ids) |
| `/mentionable-users/` | GET | `can_view_dashboards` | List org users for @mention autocomplete |

**Permission note**: All authenticated org users who can view dashboards can comment. Edit/delete is restricted to the comment author at the service layer (not via permissions).

Use `api_response()` wrapper from `ddpui/utils/response_wrapper.py` for all responses.

### 1.5 Celery Task for Email

**Modify**: `DDP_backend/ddpui/celeryworkers/tasks.py`

Add a `send_comment_notification_email` task that:
1. Receives notification_id and subject
2. Loads the notification + comment data
3. Builds email with comment content, author, link to dashboard/chart
4. Sends via Django's `send_mail`

### 1.6 Register in Routes

**Modify**: `DDP_backend/ddpui/routes.py`

```python
from ddpui.api.comments_api import comments_router
comments_router.tags = ["Comments"]
src_api.add_router("/api/comments/", comments_router)
```

### 1.7 Migration

```bash
cd DDP_backend && python manage.py makemigrations && python manage.py migrate
```

---

## 2. Frontend Implementation

### 2.1 Types

**New file**: `webapp_v2/types/comments.ts`

```typescript
interface Comment {
  id: number;
  target_type: 'dashboard' | 'chart';
  target_id: number;
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
```

### 2.2 API Hooks

**New file**: `webapp_v2/hooks/api/useComments.ts`

Uses SWR pattern from existing `hooks/api/useDashboards.ts`:

```typescript
// SWR hooks
useComments(targetType, targetId)         // List comments for a target
useCommentCounts(targetType, targetIds)   // Batch comment counts
useMentionableUsers()                     // Org users for @mention autocomplete

// Mutation functions (not SWR, just apiPost/apiPut/apiDelete)
createComment(payload)
updateComment(commentId, content)
deleteComment(commentId)
```

### 2.3 Comment Popover Component

**New file**: `webapp_v2/components/dashboard/comment-popover.tsx`

A Radix `Popover` (reusing existing `components/ui/popover.tsx`) that:

1. **Trigger**: A ghost Button with `MessageCircle` icon (from lucide-react) + Badge showing comment count
2. **Content**: Fixed-height (500px) popover with:
   - Header: "Comments" title + count
   - ScrollArea with threaded comment list
   - Input area: Textarea + Send button, Cmd+Enter shortcut
   - Reply mode: "Replying to..." indicator with cancel
3. **Each comment**: Avatar (initials) + author name + timestamp + content + Reply/Edit/Delete buttons
4. **Replies**: Indented (ml-8) under parent comment, collapsible if > 3

**Props**:
```typescript
interface CommentPopoverProps {
  targetType: 'dashboard' | 'chart';
  targetId: number;
  triggerClassName?: string;
}
```

**@mention autocomplete**: When user types `@`, show a dropdown of org users fetched from `useMentionableUsers()`. Select to insert `@email@example.com` into textarea. Use a simple filtered list (no complex mention library needed for v1).

### 2.4 Integration Points

#### Chart View (`chart-element-view.tsx`)

**Modify**: `webapp_v2/components/dashboard/chart-element-view.tsx`

Add `<CommentPopover>` to the chart's action toolbar (the area with refresh/export/fullscreen buttons). Only show when NOT in public mode.

```tsx
// In the toolbar area (around the existing export/fullscreen buttons)
{!isPublicMode && (
  <CommentPopover targetType="chart" targetId={chartId} triggerClassName="h-7 w-7 p-0" />
)}
```

#### Dashboard View (`dashboard-native-view.tsx`)

**Modify**: `webapp_v2/components/dashboard/dashboard-native-view.tsx`

Add `<CommentPopover>` to the `ResponsiveDashboardActions` toolbar area. Only show when NOT in public mode.

```tsx
// In the dashboard actions area
{!isPublicMode && (
  <CommentPopover targetType="dashboard" targetId={dashboardId} triggerClassName="h-8" />
)}
```

#### Header Notification Integration

The existing notification bell in `webapp_v2/components/header.tsx` (line 186) already fetches unread count from `/api/notifications/unread_count` and navigates to `/notifications`. Since we're creating notifications through the existing system, comment notifications will automatically appear in the existing notification count and list. **No changes needed to the header**.

---

## 3. Files to Create

| # | File | Service |
|---|------|---------|
| 1 | `DDP_backend/ddpui/models/comment.py` | Backend |
| 2 | `DDP_backend/ddpui/core/comments/__init__.py` | Backend |
| 3 | `DDP_backend/ddpui/core/comments/comment_service.py` | Backend |
| 4 | `DDP_backend/ddpui/core/comments/mention_service.py` | Backend |
| 5 | `DDP_backend/ddpui/core/comments/exceptions.py` | Backend |
| 6 | `DDP_backend/ddpui/schemas/comment_schema.py` | Backend |
| 7 | `DDP_backend/ddpui/api/comments_api.py` | Backend |
| 8 | `webapp_v2/types/comments.ts` | Frontend |
| 9 | `webapp_v2/hooks/api/useComments.ts` | Frontend |
| 10 | `webapp_v2/components/dashboard/comment-popover.tsx` | Frontend |

## 4. Files to Modify

| # | File | Change |
|---|------|--------|
| 1 | `DDP_backend/ddpui/routes.py` | Add `comments_router` import + mount at `/api/comments/` |
| 2 | `DDP_backend/ddpui/celeryworkers/tasks.py` | Add `send_comment_notification_email` task |
| 3 | `webapp_v2/components/dashboard/chart-element-view.tsx` | Add CommentPopover to chart toolbar |
| 4 | `webapp_v2/components/dashboard/dashboard-native-view.tsx` | Add CommentPopover to dashboard actions |

---

## 5. Implementation Order

### Phase 1: Backend Models + Service (do first)
1. Create `models/comment.py` with Comment + CommentMention
2. Run `makemigrations` and `migrate`
3. Create `core/comments/exceptions.py`
4. Create `core/comments/comment_service.py` (CRUD + threading)
5. Create `core/comments/mention_service.py` (parsing + notifications)
6. Create `schemas/comment_schema.py`
7. Create `api/comments_api.py`
8. Register router in `routes.py`
9. Add Celery email task

### Phase 2: Backend Tests
10. Create `DDP_backend/ddpui/tests/core/test_comment_service.py`
11. Create `DDP_backend/ddpui/tests/core/test_mention_service.py`
12. Create `DDP_backend/ddpui/tests/api/test_comments_api.py`

### Phase 3: Frontend
13. Create `types/comments.ts`
14. Create `hooks/api/useComments.ts`
15. Create `components/dashboard/comment-popover.tsx`
16. Integrate into `chart-element-view.tsx`
17. Integrate into `dashboard-native-view.tsx`

### Phase 4: Frontend Tests
18. Create `webapp_v2/__tests__/hooks/api/useComments.test.ts`
19. Create `webapp_v2/__tests__/components/dashboard/comment-popover.test.tsx`

---

## 6. Testing Strategy

### Backend Unit Tests

**File**: `DDP_backend/ddpui/tests/core/test_comment_service.py`

| Test Case | Description |
|-----------|-------------|
| `test_create_comment_on_dashboard` | Create comment on dashboard, verify fields |
| `test_create_comment_on_chart` | Create comment on chart, verify fields |
| `test_create_reply` | Create reply, verify parent_comment link |
| `test_nested_replies_max_depth` | Verify replies render up to 5 levels |
| `test_mention_parsing_full_email` | `@user@example.com` resolves to OrgUser |
| `test_mention_parsing_nonexistent_user` | Unknown @mention is silently ignored |
| `test_mention_creates_notification` | Mention creates Notification + NotificationRecipient |
| `test_reply_notifies_parent_author` | Reply creates notification for parent comment author |
| `test_self_mention_no_notification` | Mentioning yourself doesn't create notification |
| `test_update_comment_author_only` | Non-author gets CommentPermissionError |
| `test_update_comment_sets_is_edited` | Updated comment has is_edited=True |
| `test_delete_comment_soft` | Soft delete sets is_deleted=True |
| `test_delete_comment_author_only` | Non-author gets CommentPermissionError |
| `test_list_comments_org_isolation` | Comments from other orgs are not visible |
| `test_list_comments_excludes_deleted` | Soft-deleted comments not in list |
| `test_comment_counts_batch` | Batch counts for multiple charts/dashboards |
| `test_invalid_target_type` | Raises CommentValidationError |
| `test_parent_comment_target_mismatch` | Reply on different target raises error |

**File**: `DDP_backend/ddpui/tests/api/test_comments_api.py`

| Test Case | Description |
|-----------|-------------|
| `test_create_comment_api_201` | POST /api/comments/ returns success |
| `test_list_comments_api` | GET /api/comments/?target_type=dashboard&target_id=1 |
| `test_update_comment_api` | PUT /api/comments/1/ |
| `test_delete_comment_api` | DELETE /api/comments/1/ |
| `test_comment_counts_api` | GET /api/comments/counts/ |
| `test_mentionable_users_api` | GET /api/comments/mentionable-users/ |
| `test_unauthenticated_403` | Requests without auth return 403 |

### Frontend Tests

**File**: `webapp_v2/__tests__/components/dashboard/comment-popover.test.tsx`

| Test Case | Description |
|-----------|-------------|
| `renders comment button` | Button with MessageCircle icon exists |
| `shows comment count badge` | Badge shows count when comments > 0 |
| `opens popover on click` | Popover content visible after click |
| `displays empty state` | Shows "No comments yet" when empty |
| `displays comments list` | Renders comments with author, content, time |
| `submits new comment` | Calls createComment on form submit |
| `handles reply mode` | Clicking Reply sets parent_comment_id |
| `handles edit` | Clicking Edit shows textarea with current content |
| `handles delete` | Clicking Delete calls deleteComment |

### Integration / Smoke Testing

1. Start backend: `uvicorn ddpui.asgi:application --reload --port 8002`
2. Start frontend: `cd webapp_v2 && npm run dev`
3. Navigate to a dashboard → verify comment icon on charts and dashboard actions bar
4. Create a comment → verify it appears
5. Reply to a comment → verify threading
6. @mention a user → verify notification appears in the bell icon
7. Edit/delete comments → verify behavior
8. Open public dashboard URL → verify comments are NOT visible

---

## 7. Error Handling Strategy

| Scenario | Backend | Frontend |
|----------|---------|----------|
| Comment not found | 404 HttpError | Toast error message |
| Permission denied (edit/delete) | 403 HttpError | Toast + hide Edit/Delete buttons for non-author |
| Invalid target | 400 HttpError | Toast error |
| Network failure | N/A | Toast "Failed to post comment", retry button |
| Empty comment | Schema validation (min_length=1) | Disable Send button when empty |
| Comment too long | Schema validation (max_length=5000) | Character counter in textarea |

---

## 8. Key Patterns to Reuse

| Pattern | Source File |
|---------|------------|
| API response wrapper | `DDP_backend/ddpui/utils/response_wrapper.py` → `api_response()` |
| Permission decorator | `DDP_backend/ddpui/auth.py` → `@has_permission()` |
| Core module structure | `DDP_backend/ddpui/core/charts/` (service + operations pattern) |
| Schema design | `DDP_backend/ddpui/schemas/dashboard_schema.py` (from_model pattern) |
| Router registration | `DDP_backend/ddpui/routes.py` (import + tags + add_router) |
| Existing notification system | `DDP_backend/ddpui/core/notifications/notifications_functions.py` |
| SWR hooks | `webapp_v2/hooks/api/useDashboards.ts` (data fetching pattern) |
| API client | `webapp_v2/lib/api.ts` → `apiGet, apiPost, apiPut, apiDelete` |
| Popover UI | `webapp_v2/components/ui/popover.tsx` (Radix Popover) |
| Chart toolbar | `webapp_v2/components/dashboard/chart-element-view.tsx` (action buttons area) |
| Dashboard actions | `webapp_v2/components/dashboard/dashboard-native-view.tsx` |
| Toast notifications | `sonner` library (already used in the project) |
| Badge component | `webapp_v2/components/ui/badge.tsx` |
| ScrollArea | `webapp_v2/components/ui/scroll-area.tsx` |
| Avatar | `webapp_v2/components/ui/avatar.tsx` |

---

## 9. Quality Checklist

- [x] All necessary context included (models, APIs, frontend components, existing patterns)
- [x] Validation gates are executable (test cases defined with specific assertions)
- [x] References existing patterns (core module, schema, API, SWR hooks)
- [x] Clear implementation path for backend + frontend
- [x] Error handling documented (table of scenarios)
- [x] Multi-tenant isolation enforced (org FK on Comment, all queries filtered by org)
- [x] Public dashboard exclusion (CommentPopover hidden when isPublicMode=true)
- [x] Reuses existing notification system (no duplicate infrastructure)
