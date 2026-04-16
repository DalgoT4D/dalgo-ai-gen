# Dalgo Unified Access Control & RLS — Detailed Analysis + Roadmap

---

## Part 1: What Exists Today

### 1.1 Role System

**5 roles** defined in `seed/001_roles.json`:

| Role | Slug | Level | Intended Use |
|------|------|-------|-------------|
| Super Admin | `super-admin` | 5 | Full platform control |
| Account Manager | `account-manager` | 4 | Org administration |
| Pipeline Manager | `pipeline-manager` | 3 | Data pipeline management |
| Analyst | `analyst` | 2 | Data analysis |
| Guest | `guest` | 1 | Read-only |

**Models**: `Role(uuid, slug, name, level)`, `Permission(uuid, slug, name)`, `RolePermission(role FK, permission FK)` in `ddpui/models/role_based_access.py`.

### 1.2 Permission System (73 permissions)

**Permission enforcement** (`ddpui/auth.py:30-51`):
- `@has_permission(["slug1", "slug2"])` decorator on API endpoints
- Checks `request.permissions` (set superset check — user must have ALL listed slugs)
- Raises 403 if missing

**JWT middleware** (`ddpui/auth.py:94-196`):
- On every request: validates JWT → extracts user_id → resolves OrgUser → loads permissions
- **Redis caching**: `dalgo_permissions_key` stores `{role_id: [permission_slugs]}`, `orguser_role:{user_id}` stores `{orguser_id: role_id}`
- Falls back to DB if Redis miss, then re-populates cache

### 1.3 Current Role → Permission Matrix

**Analyst (role 4) currently has 46 permissions** including:
- ALL pipeline permissions (view/create/edit/delete pipelines, sources, connections)
- ALL dbt permissions (view/edit dbt models, operations, run orgtasks)
- ALL dashboard/chart permissions (view/create/edit/delete/share)
- Warehouse data viewing

**Guest (role 5) currently has 9 permissions**:
- `can_view_pipeline_overview`, `can_view_sources`, `can_view_warehouses`, `can_view_connections`
- `can_view_dbt_workspace`, `can_view_orgtasks`, `can_view_warehouse_data`
- `can_view_invitations`, `public`
- **Missing**: `can_view_dashboards` (pk=64), `can_view_charts` (pk=65) — Guest can't even view dashboards/charts!

### 1.4 Dashboard Access Control

**File**: `ddpui/api/dashboard_native_api.py`

| What | How it works | Gap |
|------|-------------|-----|
| Listing | `can_view_dashboards` + org filter | Shows ALL org dashboards — no per-resource ACL |
| Create | `can_create_dashboards` | Auto-assigns to org |
| Edit | `can_edit_dashboards` + org filter | Any editor can edit any dashboard |
| Delete | `can_delete_dashboards` + org filter | Any deleter can delete any dashboard |
| Public sharing | `can_share_dashboards` + **creator-only check** (line 450) | Only creator can toggle public. No user-to-user sharing |
| Public access | Token-based, no auth | `secrets.token_urlsafe(48)`, tracks access count |

**Public sharing flow**:
1. Creator calls `PUT /{dashboard_id}/share/` with `is_public=true`
2. System generates 48-char token, sets `public_share_token`, `public_shared_at`
3. Public endpoint: `GET /api/public/dashboards/{token}/` — no auth, validates token + `is_public=True`
4. Tracks: `public_access_count`, `last_public_accessed`

### 1.5 Chart Access Control

**File**: `ddpui/api/charts_api.py`

| What | How it works | Gap |
|------|-------------|-----|
| Listing | `can_view_charts` + org filter | Shows ALL org charts |
| Schema access | `has_schema_access()` at line 109 | **TODO: always returns True** |
| Sharing | None | No public sharing, no user-to-user sharing |

### 1.6 Report/Snapshot Access Control

**File**: `ddpui/api/report_api.py`

| What | How it works | Gap |
|------|-------------|-----|
| Listing | `can_view_dashboards` (reuses dashboard permission) | No report-specific permissions |
| Create | `can_create_dashboards` | Reuses dashboard permission |
| Public sharing | `can_share_dashboards` + creator-only | Same pattern as dashboards |
| Email sharing | New in PR (feature branch) | Sends PDF via SES |

### 1.7 Frontend Permission Usage

**File**: `webapp_v2/hooks/api/usePermissions.ts`

Permissions are checked in the frontend but **sparsely**:

| Permission | Where Used | What It Controls |
|-----------|-----------|-----------------|
| `can_create_org` | `header.tsx:63` | "Create Organization" button |
| `can_create_invitation` | `UserManagement.tsx:16` | "Invite User" button |
| `can_view_invitations` | `UserManagement.tsx:17` | Invitations tab |
| `can_edit_orguser` | `UsersTable.tsx:79` | Edit role action |
| `can_delete_orguser` | `UsersTable.tsx:80` | Delete user action |
| `can_view_dashboards` | `/app/dashboards/[id]/page.tsx:24` | Dashboard detail page access |
| `can_initiate_org_plan_upgrade` | `billing.tsx` | Upgrade button |

**Sidebar navigation** (`main-layout.tsx`): Controlled by **feature flags only**, NOT by role/permissions. All authenticated users see the same nav items.

**Route protection**: Minimal. Only page-level permission checks on some pages. No middleware-level auth guards (middleware.ts only handles CORS for `/share/*` routes).

### 1.8 Query Execution Layer (RLS injection points)

**AggQueryBuilder** (`ddpui/core/datainsights/query_builder.py:15-161`):
- `where_clause(condition)` method (line 101-104) — accumulates WHERE conditions
- All WHERE clauses applied with AND logic in `build()` method (line 124-126)

**Chart query flow**:
```
API endpoint → build_chart_query() → apply_dashboard_filters() → apply_chart_filters() → execute_chart_query() → warehouse.execute(sql)
```

**7 authenticated endpoints that execute warehouse queries** (all in `charts_api.py`):
1. `POST /chart-data/` (line 528) — main chart data
2. `GET /{chart_id}/data/` (line 1083) — chart data by ID
3. `POST /chart-data-preview/` (line 573) — preview data
4. `POST /chart-data-preview/total-rows/` (line 700) — row count
5. `POST /map-data-overlay/` (line 387) — map overlays
6. `POST /download-csv/` (line 1027) — CSV export
7. `POST /map-data/` (line 894) — map chart data

**Plus** warehouse exploration endpoints (`warehouse_api.py`): schemas, tables, columns, column values.

**Plus** public endpoints (`public_api.py`): chart data for public dashboards.

---

## Part 2: What's Missing / What to Improve

### 2.1 Critical Gaps

| Gap | Impact | Priority |
|-----|--------|----------|
| **No per-resource sharing** | Can't share a specific dashboard with a specific user. It's all-or-nothing within org | High |
| **Guest can't view dashboards/charts** | Role meant for "view only" users can't actually view the core features | High — immediate fix |
| **Analyst has pipeline permissions** | Analysts can modify pipelines, dbt models — too much access for data consumers | High |
| **`has_schema_access()` is a TODO** | All users can query all schemas/tables in the warehouse | High — data exposure risk |
| **No sidebar permission gating** | Analysts/Guests see Ingest, Transform, Orchestrate nav items they can't use | Medium |
| **No route protection** | Direct URL access to unauthorized pages isn't blocked on frontend | Medium |
| **No report-specific permissions** | Reports reuse dashboard permissions — can't control separately | Low |
| **No chart sharing** | Charts can only be shared within dashboards, not individually | Low |

### 2.2 Role Definition Problems

**Guest (should be "Viewer")**:
- Name doesn't convey purpose — "Guest" sounds temporary
- Has 9 permissions but critically missing `can_view_dashboards` and `can_view_charts`
- Has `can_view_pipeline_overview`, `can_view_sources` etc. which it shouldn't need
- **Fix**: Rename to Viewer, give only dashboard/chart view permissions

**Analyst**:
- Has 46 permissions including full pipeline/dbt management
- V2 spec wants Analysts focused on data analysis only (dashboards, charts, exploration)
- **Fix**: Remove pipeline/dbt/source/connection management permissions
- **Risk**: Breaking change for existing Analyst users who manage pipelines

### 2.3 Access Control Architecture Gaps

**Current**: Two-layer model (Role permission → Org isolation)
```
Can user do X in their org? = has_permission(slug) + org filter
```

**Missing**: Three-layer model (Role permission → Org isolation → Resource-level)
```
Can user do X on this specific resource? = role check + org check + resource share check
```

### 2.4 Data-Level Security Gaps

**Current**: Any authenticated org user with `can_view_charts` can query ANY schema/table in the warehouse.

**Missing**:
- Schema-level access control (which users can see which schemas)
- Table-level access control (which users can see which tables)
- Row-level security (which rows within a table a user can see)

---

## Part 3: Improvement Plan

### Phase 1: Fix Immediate Role & Permission Issues

**Scope**: Seed data changes + minimal code changes. No new models.

#### 1a. Fix Guest permissions
- **Add**: `can_view_dashboards` (pk=64), `can_view_charts` (pk=65) to Guest role
- **Remove**: `can_view_pipeline_overview`, `can_view_sources`, `can_view_warehouses`, `can_view_connections`, `can_view_dbt_workspace`, `can_view_orgtasks` from Guest
- **Keep**: `can_view_warehouse_data` (needed for chart data), `public`
- **Files**: `seed/003_role_permissions.json` + data migration

#### 1b. Rename Guest → Viewer
- **Update**: `seed/001_roles.json` pk=5: slug `guest` → `viewer`, name `Guest` → `Viewer`
- **Update**: `ddpui/auth.py` line ~16: `GUEST_ROLE = "guest"` → `VIEWER_ROLE = "viewer"`
- **Search & replace**: All references to "guest" in role contexts
- **Data migration**: Update existing Role record in DB
- **Frontend**: Update role display formatting in `UsersTable.tsx`

#### 1c. Restrict Analyst permissions
- **Remove from Analyst** (role pk=4):
  - Pipeline: `can_view_pipeline_overview`(1), `can_view_sources`(2), `can_create_source`(3), `can_edit_source`(4), `can_view_source`(5), `can_delete_source`(17), `can_sync_sources`(37)
  - Warehouses: `can_view_warehouses`(6), `can_view_warehouse`(7), `can_create_warehouse`(8), `can_edit_warehouse`(9), `can_delete_warehouses`(57)
  - Connections: `can_view_connection`(10), `can_create_connection`(12), `can_view_connections`(13), `can_reset_connection`(14), `can_edit_connection`(15), `can_delete_connection`(16)
  - Dbt: `can_create_dbt_workspace`(20), `can_edit_dbt_workspace`(21), `can_delete_dbt_workspace`(22), `can_view_dbt_workspace`(23), `can_create_dbt_docs`(25), `can_create_dbt_model`(38), `can_edit_dbt_operation`(39), `can_view_dbt_operation`(40), `can_edit_dbt_model`(41), `can_view_dbt_models`(42), `can_delete_dbt_model`(43), `can_delete_dbt_operation`(44)
  - Tasks: `can_view_master_tasks`(18), `can_view_master_task`(19), `can_run_orgtask`(24), `can_create_orgtask`(26), `can_view_orgtasks`(27), `can_delete_orgtask`(28)
  - Pipelines: `can_create_pipeline`(29), `can_view_pipelines`(30), `can_view_pipeline`(31), `can_delete_pipeline`(32), `can_edit_pipeline`(33), `can_run_pipeline`(34)
  - Other: `can_view_task_progress`(35), `can_manage_org_default_dashboard`(73)
- **Keep for Analyst**:
  - `can_view_dashboards`(64), `can_create_dashboards`(69), `can_edit_dashboards`(70), `can_delete_dashboards`(71)
  - `can_view_charts`(65), `can_create_charts`(66), `can_edit_charts`(67), `can_delete_charts`(68)
  - `can_share_dashboards`(72)
  - `can_view_warehouse_data`(45)
  - `can_view_usage_dashboard`(36)
  - `can_view_orgusers`(46), `can_view_invitations`(53)
  - `can_accept_tnc`(58), `can_view_flags`(59)
  - `can_request_llm_analysis_feature`(63)
- **Risk mitigation**: Data migration that checks if any existing Analysts actively use pipeline features. Log a warning. Consider a management command to bulk-upgrade Analysts → Pipeline Manager.

#### 1d. Add report-specific permissions
- **New permissions**: `can_view_reports`(74), `can_create_reports`(75), `can_share_reports`(76)
- **Assign**: Roles 1-4 get all three; Viewer gets `can_view_reports` only
- **Update**: `report_api.py` to use `can_view_reports` instead of `can_view_dashboards`

**Files to modify**:
- `seed/001_roles.json`, `seed/002_permissions.json`, `seed/003_role_permissions.json`
- `ddpui/auth.py`
- `ddpui/api/report_api.py`
- Data migration script
- Frontend: `UsersTable.tsx` role display

---

### Phase 2: Permission-Based Frontend Navigation & Route Protection

**Scope**: Frontend-only changes. Use existing permission data already available in `authStore`.

#### 2a. Sidebar permission gating

**File**: `webapp_v2/components/main-layout.tsx`

Add a `requiredPermission` field to each nav item:

```
Ingest (Sources/Connections): requires can_view_sources
Transform (dbt): requires can_view_dbt_workspace
Orchestrate (Pipelines): requires can_view_pipelines
Dashboards: requires can_view_dashboards
Charts: requires can_view_charts
Reports: requires can_view_reports (new permission)
Settings: requires can_view_orgusers
Usage: requires can_view_usage_dashboard
```

Filter `getNavItems()` output using `useUserPermissions().hasPermission()`. Items the user lacks permission for are **hidden entirely**.

#### 2b. Route guards

Create a `PermissionGate` wrapper component that:
- Checks required permission(s) for the current route
- Redirects to `/dashboards` (or first accessible page) if denied
- Shows brief "Access Denied" message before redirect

Apply to all page-level layouts:
- `/app/pipeline/*` → requires `can_view_pipelines`
- `/app/transforms/*` → requires `can_view_dbt_workspace`
- `/app/dashboards/*` → requires `can_view_dashboards`
- `/app/charts/*` → requires `can_view_charts`
- `/app/reports/*` → requires `can_view_reports`
- `/app/settings/*` → requires `can_view_orgusers`

**Files to modify**:
- `webapp_v2/components/main-layout.tsx` — nav item filtering
- New: `webapp_v2/components/ui/permission-gate.tsx` — reusable guard
- Page layouts for each route group

---

### Phase 3: Resource-Level Sharing

**Scope**: New model + API + frontend. Lets users share specific dashboards/charts with specific org users.

> **V2 spec alignment**: This phase implements the core of the V2 Access Control spec — paste-to-share, matched/unmatched email handling, chart inheritance from dashboards, Analyst invite-as-Viewer, and view/edit permission levels (no "manage" level in V1).

#### 3a. ResourceShare model (`ddpui/models/resource_sharing.py`)

```python
class SharePermission(models.TextChoices):
    VIEW = "view"
    EDIT = "edit"

class ShareStatus(models.TextChoices):
    ACTIVE = "active"
    PENDING = "pending"  # user invited but hasn't accepted yet

class ResourceShare(models.Model):
    uuid = models.UUIDField(default=uuid4, unique=True)
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    resource = GenericForeignKey("content_type", "object_id")
    shared_by = models.ForeignKey(OrgUser, on_delete=models.CASCADE, related_name="shares_given")
    shared_with = models.ForeignKey(OrgUser, null=True, blank=True, on_delete=models.CASCADE, related_name="shares_received")
    shared_with_email = models.EmailField(null=True, blank=True)  # for pending invites (unmatched emails)
    permission = models.CharField(max_length=10, choices=SharePermission.choices, default=SharePermission.VIEW)
    status = models.CharField(max_length=10, choices=ShareStatus.choices, default=ShareStatus.ACTIVE)
    is_inherited = models.BooleanField(default=False)  # True if inherited from parent dashboard
    parent_share = models.ForeignKey("self", null=True, blank=True, on_delete=models.CASCADE, related_name="inherited_shares")
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = ("content_type", "object_id", "shared_with")
        indexes = [
            models.Index(fields=["content_type", "object_id"]),
            models.Index(fields=["shared_with"]),
            models.Index(fields=["shared_with_email"]),
        ]
```

Uses Django ContentType GenericFK — one table for all resource types (Dashboard, Chart, ReportSnapshot).

**Key additions vs original plan**:
- `shared_with_email` — stores email for pending invites (unmatched emails from paste-to-share)
- `shared_with_group` — FK to `UserGroup` for group-based sharing (see `groups_and_sharing_plan.md`)
- `status` — tracks pending (invited but not accepted) vs active shares
- `is_inherited` + `parent_share` — tracks chart shares inherited from dashboard sharing
- No "manage" permission level — only owners can manage sharing (per V2 spec)

> **User Groups**: `UserGroup` and `UserGroupMember` models are defined in the detailed groups plan at `groups_and_sharing_plan.md`. ResourceShare supports sharing with individual users, groups, or pending emails — exactly one target per share record.

#### 3b. Chart inheritance from dashboards

> **V2 spec**: "If a dashboard is shared, charts inside it inherit the access (view/edit). Charts inherited via dashboards will appear in 'My Charts' and behave like directly shared view-only charts."

When a dashboard is shared:
1. Query all charts belonging to that dashboard
2. For each chart, create a `ResourceShare` with `is_inherited=True`, `parent_share=dashboard_share`
3. Inherited shares have same permission level as the dashboard share
4. If dashboard share is revoked, cascade-delete all inherited chart shares via `parent_share` FK
5. If dashboard share permission changes (view→edit), update inherited chart shares too
6. If a chart is removed from the dashboard, delete its inherited shares
7. Directly shared charts take precedence over inherited shares (higher permission wins)

**Edit access inheritance**: If a user is granted edit access to a dashboard, they also receive edit access to all charts within it.

#### 3c. Access check service (`ddpui/core/access_service.py`)

Central service:
- `can_access(orguser, resource, required="view")` — checks: creator? → role is Analyst+ (org-wide CRUD)? → ResourceShare exists? → is_public? → denied
- `get_accessible_resources(orguser, model_class)` — returns queryset:
  - **For Viewer role**: Only resources shared with them via ResourceShare (direct + inherited) + public
  - **For Analyst+**: All org resources (current behavior) + shared resources from other orgs (future)
- `share_resource(sharer, resource, target_orguser_or_email, permission)` — creates ResourceShare, handles inheritance for dashboards
- `unshare_resource(sharer, resource, target_orguser)` — deletes ResourceShare + cascades inherited shares

#### 3d. Paste-to-share flow (V2 spec alignment)

The share modal accepts multiple emails pasted at once. Backend processes each:

**Matched email** (user exists in org):
- Create `ResourceShare` with `status=active`, `shared_with=orguser`
- Apply permission level (default: view)

**Unmatched email** (user not on Dalgo):
- Frontend shows warning: `⚠ xyz@fake.org - not found on Dalgo`
- Option: `[Invite xyz@fake.org as Viewer] [Remove]`
- If invited:
  - Create invitation with `role=viewer` (backend-enforced for Analyst sharers)
  - Create `ResourceShare` with `status=pending`, `shared_with_email=email`, `shared_with=null`
  - Send invitation email
  - On acceptance: Update `ResourceShare` to `status=active`, set `shared_with=new_orguser`

#### 3e. Sharing API endpoints (`ddpui/api/sharing_api.py`)

| Endpoint | Method | Description |
|---|---|---|
| `/sharing/{type}/{id}/` | GET | List who has access (includes inherited shares) |
| `/sharing/{type}/{id}/` | POST | Share with user(s) — accepts list of emails + permission |
| `/sharing/{type}/{id}/{share_id}/` | PATCH | Change permission level (cascades to inherited if dashboard) |
| `/sharing/{type}/{id}/{share_id}/` | DELETE | Revoke access (cascades inherited chart shares if dashboard) |
| `/sharing/users/search/?q=` | GET | Search org users for share picker |
| `/sharing/{type}/{id}/invite/` | POST | Invite unmatched email as Viewer + create pending share |

New permissions: `can_share_charts`(77), `can_share_reports` (reuse 76 from Phase 1).

#### 3f. Update listing endpoints

Modify `list_dashboards()`, chart listing, and `list_snapshots()` to use `get_accessible_resources()`:
- **Viewer**: Only sees resources explicitly shared with them (direct + inherited) + public
- **Analyst+**: Sees all org resources (existing behavior maintained)
- Charts inherited via dashboard sharing appear in the user's "My Charts" list

#### 3g. Frontend share dialog

Replace dead email code in `share-modal.tsx` with paste-to-share UI:
- **Paste field**: Accept multiple emails, split on comma/space/newline
- **Email validation**: Check each against org user list
- **Matched users**: Show with avatar + name + role badge, default to view access
- **Unmatched emails**: Show warning icon, "Invite as Viewer" or "Remove" options
- **Current shares list**: Show all shared users with permission dropdown (view/edit) + revoke button
- **Inherited shares indicator**: Badge showing "via [Dashboard Name]" for inherited chart shares
- **Public link toggle**: Existing functionality
- **Email sharing**: Existing from PR (reports only)
- **Confirmation summary**: Before submitting, show summary of all actions

**Files to create**: `ddpui/models/resource_sharing.py`, `ddpui/core/access_service.py`, `ddpui/api/sharing_api.py`, frontend share dialog
**Files to modify**: `ddpui/api/dashboard_native_api.py`, `ddpui/api/charts_api.py`, `ddpui/api/report_api.py` (listing endpoints), `webapp_v2/components/ui/share-modal.tsx`

---

### Phase 4: Row-Level Security (RLS)

**Scope**: New models + query injection. Controls which data rows users can see.

#### 4a. Schema/Table access control

**New model** (`ddpui/models/rls.py`):

```python
class SchemaAccess(models.Model):
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    orguser = models.ForeignKey(OrgUser, null=True, blank=True, on_delete=models.CASCADE)
    role = models.ForeignKey(Role, null=True, blank=True, on_delete=models.CASCADE)
    schema_name = models.CharField(max_length=255)
    table_name = models.CharField(max_length=255, default="*")  # * = all tables
    allowed = models.BooleanField(default=True)
```

**Implement `has_schema_access()`** (`charts_api.py:109`):
- Check SchemaAccess for user-specific rules first, then role-based
- Super Admin / Account Manager bypass
- Default behavior (no rules) = allow all (backward compatible)

#### 4b. Row-level filtering

**New model**:

```python
class RLSPolicy(models.Model):
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
    schema_name = models.CharField(max_length=255)
    table_name = models.CharField(max_length=255)
    filter_column = models.CharField(max_length=255)
    filter_type = models.CharField(choices=[("exact", "Exact"), ("in", "In list"), ("user_attr", "User attribute")])
    filter_value = models.JSONField(default=dict)
    orguser = models.ForeignKey(OrgUser, null=True, blank=True, on_delete=models.CASCADE)
    role = models.ForeignKey(Role, null=True, blank=True, on_delete=models.CASCADE)
    enabled = models.BooleanField(default=True)
```

**User attributes**: Add `rls_attributes = models.JSONField(default=dict)` to OrgUser. Example: `{"region": "East", "department": "Sales"}`.

#### 4c. Query injection

**New service** (`ddpui/core/rls_service.py`):
- `get_rls_conditions(orguser, schema_name, table_name)` — looks up applicable policies, resolves `user_attr` values, returns list of SQLAlchemy WHERE conditions

**Integration point** in `build_chart_query()` (`charts_service.py`):
- After `fetch_from()` (line ~331), before `apply_dashboard_filters()` (line ~455)
- Call `rls_service.get_rls_conditions()` and apply each with `query_builder.where_clause()`
- This reuses the exact same pattern as `apply_dashboard_filters()` and `apply_chart_filters()`

**All 7 authenticated chart data endpoints** + warehouse exploration endpoints + public endpoints need RLS.

#### 4d. Admin API & UI

Backend: CRUD endpoints for RLS policies and schema access rules.
Frontend: Settings page for Super Admin / Account Manager to manage policies.

New permission: `can_manage_rls`(78), assigned to roles 1-2 only.

**Files to create**: `ddpui/models/rls.py`, `ddpui/core/rls_service.py`, `ddpui/api/rls_api.py`, frontend RLS admin page
**Files to modify**: `ddpui/models/org_user.py` (rls_attributes), `ddpui/api/charts_api.py` (has_schema_access), `ddpui/core/charts/charts_service.py` (RLS injection in build_chart_query)

---

### Phase 5: Future Scope (Not Implemented)

- **Time-bound access**: `expires_at` on ResourceShare (for donors, temporary consultants)
- **Audit logs**: Who accessed/shared/edited what resource when (visible to Account Managers)
- **Admin sharing override**: Let Account Managers see all shared resources and adjust sharing
- **Invite role control**: Optional toggle to restrict invite ability to Account Managers only
- **Dashboard-level RLS override**: All charts in dashboard inherit RLS policy

> **Note**: User Groups have been moved into Phase 3. See `groups_and_sharing_plan.md` for the full plan including data models, API endpoints, and frontend UX.

---

## Implementation Order

```
Phase 1 (Role fixes)          → Independent, no new models, low risk
                                 Note: Viewer role rename + permission fix ships here,
                                 but Viewer listing is org-wide until Phase 3 adds shared-only filtering
Phase 2 (Frontend nav/routes) → Frontend only, depends on Phase 1 permissions
Phase 3 (Resource sharing)    → New model + API. CRITICAL for Viewer role to work as V2 spec intends
                                 (shared-only access). Also enables paste-to-share, chart inheritance,
                                 and Analyst invite-as-Viewer
Phase 4 (RLS)                 → Depends on Phase 1 (role definitions final)
```

**Suggested order**: Phase 1 → Phase 2 (parallel with Phase 3 backend) → Phase 3 frontend → Phase 4

**Important dependency**: The Viewer role is only fully useful after Phase 3 ships. In the interim (Phase 1 only), Viewers will see all org dashboards/charts — not ideal but acceptable as a transitional state. Phase 3 tightens this to shared-only.

---

## Complete Current Roles × Permissions Matrix

### Role Summary

| ID | Role | Level | Intended Use | Actual Permission Count |
|---|---|---|---|---|
| 1 | Super Admin | 5 | Full platform control | 72 of 73 |
| 2 | Account Manager | 4 | Org administration | 71 of 73 |
| 3 | Pipeline Manager | 3 | Data infrastructure | 57 of 73 |
| 4 | Analyst | 2 | Data analysis | 46 of 73 |
| 5 | Guest | 1 | Read-only | 24 of 73 |

### Ingestion: Sources, Warehouses, Connections

| Permission | SA | AM | PM | An | Gu | Problem? |
|---|:---:|:---:|:---:|:---:|:---:|---|
| `can_view_pipeline_overview` (1) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see pipeline overview |
| `can_view_sources` (2) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see sources list |
| `can_create_source` (3) | ✓ | ✓ | ✓ | - | - | OK |
| `can_edit_source` (4) | ✓ | ✓ | ✓ | - | - | OK |
| `can_view_source` (5) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see source details |
| `can_delete_source` (17) | ✓ | ✓ | ✓ | - | - | OK |
| `can_sync_sources` (37) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can trigger syncs** |
| `can_view_warehouses` (6) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see warehouse config |
| `can_view_warehouse` (7) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see warehouse details |
| `can_create_warehouse` (8) | ✓ | ✓ | - | - | - | OK |
| `can_edit_warehouse` (9) | ✓ | ✓ | - | - | - | OK |
| `can_delete_warehouses` (57) | ✓ | ✓ | - | - | - | OK |
| `can_view_connection` (10) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see connection details |
| `can_create_connection` (12) | ✓ | ✓ | ✓ | - | - | OK |
| `can_view_connections` (13) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see connections list |
| `can_reset_connection` (14) | ✓ | ✓ | ✓ | - | - | OK |
| `can_edit_connection` (15) | ✓ | ✓ | ✓ | - | - | OK |
| `can_delete_connection` (16) | ✓ | ✓ | ✓ | - | - | OK |

### Transform: dbt

| Permission | SA | AM | PM | An | Gu | Problem? |
|---|:---:|:---:|:---:|:---:|:---:|---|
| `can_create_dbt_workspace` (20) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can create dbt workspace** |
| `can_edit_dbt_workspace` (21) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can edit dbt workspace** |
| `can_delete_dbt_workspace` (22) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can delete dbt workspace** |
| `can_view_dbt_workspace` (23) | ✓ | ✓ | ✓ | ✓ | ✓ | Guest sees dbt workspace |
| `can_create_dbt_docs` (25) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can generate dbt docs** |
| `can_create_dbt_model` (38) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can create dbt models** |
| `can_edit_dbt_operation` (39) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can edit dbt operations** |
| `can_view_dbt_operation` (40) | ✓ | ✓ | ✓ | ✓ | ✓ | Guest sees dbt operations |
| `can_edit_dbt_model` (41) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can edit dbt models** |
| `can_view_dbt_models` (42) | ✓ | ✓ | ✓ | ✓ | ✓ | Guest sees dbt models |
| `can_delete_dbt_model` (43) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can delete dbt models** |
| `can_delete_dbt_operation` (44) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can delete dbt operations** |

### Orchestrate: Pipelines & Tasks

| Permission | SA | AM | PM | An | Gu | Problem? |
|---|:---:|:---:|:---:|:---:|:---:|---|
| `can_view_master_tasks` (18) | ✓ | ✓ | ✓ | ✓ | ✓ | Guest sees task list |
| `can_view_master_task` (19) | ✓ | ✓ | ✓ | ✓ | ✓ | Guest sees task config |
| `can_run_orgtask` (24) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can run tasks** |
| `can_create_orgtask` (26) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can create tasks** |
| `can_view_orgtasks` (27) | ✓ | ✓ | ✓ | ✓ | ✓ | Guest sees org tasks |
| `can_delete_orgtask` (28) | ✓ | ✓ | ✓ | **✓** | - | **Analyst can delete tasks** |
| `can_create_pipeline` (29) | ✓ | ✓ | ✓ | - | - | OK |
| `can_view_pipelines` (30) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see pipelines |
| `can_view_pipeline` (31) | ✓ | ✓ | ✓ | ✓ | ✓ | Analyst/Guest see pipeline details |
| `can_delete_pipeline` (32) | ✓ | ✓ | ✓ | - | - | OK |
| `can_edit_pipeline` (33) | ✓ | ✓ | ✓ | - | - | OK |
| `can_run_pipeline` (34) | ✓ | ✓ | ✓ | - | - | OK |
| `can_view_task_progress` (35) | ✓ | ✓ | ✓ | ✓ | ✓ | All roles see task progress |

### Visualization: Dashboards, Charts & Reports

| Permission | SA | AM | PM | An | Gu | Problem? |
|---|:---:|:---:|:---:|:---:|:---:|---|
| `can_view_dashboards` (64) | ✓ | ✓ | ✓ | ✓ | ✓ | OK |
| `can_create_dashboards` (69) | ✓ | ✓ | ✓ | ✓ | - | OK |
| `can_edit_dashboards` (70) | ✓ | ✓ | ✓ | ✓ | - | OK |
| `can_delete_dashboards` (71) | ✓ | ✓ | ✓ | ✓ | - | OK |
| `can_share_dashboards` (72) | ✓ | ✓ | ✓ | ✓ | - | OK |
| `can_manage_org_default_dashboard` (73) | ✓ | ✓ | - | - | - | OK |
| `can_view_charts` (65) | ✓ | ✓ | ✓ | ✓ | ✓ | OK |
| `can_create_charts` (66) | ✓ | ✓ | ✓ | ✓ | - | OK |
| `can_edit_charts` (67) | ✓ | ✓ | ✓ | ✓ | - | OK |
| `can_delete_charts` (68) | ✓ | ✓ | ✓ | ✓ | - | OK |
| `can_view_warehouse_data` (45) | ✓ | ✓ | ✓ | ✓ | ✓ | Needed for chart data queries |
| Reports (all actions) | Reuses `can_*_dashboards` | | | | | **No report-specific permissions** |

### User & Org Management

| Permission | SA | AM | PM | An | Gu | Problem? |
|---|:---:|:---:|:---:|:---:|:---:|---|
| `can_create_org` (11) | ✓ | - | - | - | - | OK — Super Admin only |
| `can_view_orgusers` (46) | ✓ | ✓ | ✓ | ✓ | ✓ | All can see user list |
| `can_create_orguser` (47) | ✓ | ✓ | - | - | - | OK |
| `can_delete_orguser` (48) | ✓ | ✓ | - | - | - | OK |
| `can_edit_orguser` (49) | ✓ | ✓ | - | - | - | OK |
| `can_edit_orguser_role` (50) | ✓ | ✓ | - | - | - | OK |
| `can_create_invitation` (51) | ✓ | ✓ | - | - | - | OK |
| `can_view_invitations` (53) | ✓ | ✓ | ✓ | ✓ | - | OK |
| `can_edit_invitation` (54) | ✓ | ✓ | - | - | - | OK |
| `can_delete_invitation` (55) | ✓ | ✓ | - | - | - | OK |
| `can_resend_email_verification` (56) | ✓ | ✓ | - | - | - | OK |

### Settings & Platform

| Permission | SA | AM | PM | An | Gu | Problem? |
|---|:---:|:---:|:---:|:---:|:---:|---|
| `public` (52) | ✓ | ✓ | ✓ | ✓ | ✓ | OK |
| `can_accept_tnc` (58) | ✓ | ✓ | ✓ | ✓ | - | Guest can't accept T&C |
| `can_view_flags` (59) | ✓ | ✓ | ✓ | ✓ | ✓ | OK |
| `can_edit_llm_settings` (60) | ✓ | ✓ | - | - | - | OK |
| `can_edit_org_notification_settings` (61) | ✓ | ✓ | - | - | - | OK |
| `can_initiate_org_plan_upgrade` (62) | ✓ | ✓ | - | - | - | OK |
| `can_request_llm_analysis_feature` (63) | - | - | ✓ | ✓ | ✓ | SA/AM don't have this (they configure it instead) |
| `can_view_usage_dashboard` (36) | ✓ | ✓ | ✓ | ✓ | ✓ | OK |

---

## Problems Analysis

### Problem 1: Analyst ≈ Pipeline Manager for dbt (Critical)

Analyst has **full dbt write access** — create/edit/delete workspace, models, operations, and docs. This is identical to Pipeline Manager. An "Analyst" role meant for data consumers should not be modifying dbt transformation logic.

**Analyst currently has these infrastructure permissions that don't fit the role:**
- `can_sync_sources` — can trigger data syncs
- `can_create/edit/delete_dbt_workspace` — full dbt workspace management
- `can_create/edit/delete_dbt_model` — full dbt model management
- `can_create/edit/delete_dbt_operation` — full dbt operation management
- `can_create_dbt_docs` — dbt documentation generation
- `can_create/run/delete_orgtask` — task orchestration

### Problem 2: Guest sees the entire infrastructure (Medium)

Guest has 24 permissions including read access to sources, warehouses, connections, dbt workspace, dbt models/operations, pipelines, tasks, and task progress. A "Viewer" role should only need dashboards, charts, reports, and basic data access.

### Problem 3: No clear boundary between "Build" and "Consume" (Architectural)

The current role hierarchy doesn't separate data infrastructure (Ingest → Transform → Orchestrate) from data consumption (Dashboards → Charts → Reports). Pipeline Manager and Analyst overlap heavily, with Analyst getting almost all Pipeline Manager dbt permissions.

### Problem 4: Role names are confusing

- **"Guest"** sounds temporary/external — but it's meant to be a permanent "view-only" role
- **"Analyst"** implies data analysis — but has full dbt engineering access
- **"Pipeline Manager"** is clear for pipelines — but also has full dashboard/chart CRUD identical to Analyst

### Problem 5: Reports have no independent permission model

Reports reuse `can_*_dashboards`. You can't control report access separately from dashboard access.

### Problem 6: Seed data has duplicate PKs

`003_role_permissions.json` has duplicate PKs (213-216 appear twice, and pks 266-274 duplicate super admin dashboard/chart permissions). While Django handles this by using last-wins, it indicates maintenance issues.

---

## Proposed Improvement: Cleaner Role Model

### Design Principles

1. **Separate "Build" from "Consume"** — People who build data infrastructure vs. people who consume data
2. **Additive-ish hierarchy** — Higher roles get more, but Pipeline Manager and Analyst are **peer roles** with different focus areas
3. **Least privilege** — Each role gets only what it needs
4. **NGO-friendly naming** — Clear names that non-technical org admins understand

### Proposed Role Definitions (V2 Spec Aligned)

| Role | Name | Level | Focus | Who is this? |
|---|---|---|---|---|
| 1 | **Super Admin** | 5 | Everything + platform | Dalgo team / IT admin |
| 2 | **Account Manager** | 4 | Full access to all modules + user management + warehouse | Org admin / M&E lead |
| 3 | **Pipeline Manager** | 3 | Full access to all modules including dashboards/charts, **except** user management and warehouse | Data engineer / IT staff / Implementation partner |
| 4 | **Analyst** | 2 | Dashboards, charts, data explorer only. Can invite Viewers via share modal | Program analyst / M&E officer |
| 5 | **Viewer** | 1 | View-only access to dashboards/charts **explicitly shared with them** | Program staff / partners / funders |

**Key change (aligned with V2 spec)**: Pipeline Manager gets full access to all modules including dashboards/charts (same as Account Manager except user management and warehouse config). Analyst focuses on data consumption (dashboards, charts, data explorer). Viewer can **only** see resources explicitly shared with them (not org-wide access).

### Proposed Full Permission Matrix

#### Ingestion

| Permission | SA | AM | PM | An | Viewer |
|---|:---:|:---:|:---:|:---:|:---:|
| `can_view_pipeline_overview` | ✓ | ✓ | ✓ | - | - |
| `can_view_sources` | ✓ | ✓ | ✓ | - | - |
| `can_create_source` | ✓ | ✓ | ✓ | - | - |
| `can_edit_source` | ✓ | ✓ | ✓ | - | - |
| `can_view_source` | ✓ | ✓ | ✓ | - | - |
| `can_delete_source` | ✓ | ✓ | ✓ | - | - |
| `can_sync_sources` | ✓ | ✓ | ✓ | - | - |
| `can_view_warehouses` | ✓ | ✓ | ✓ | - | - |
| `can_view_warehouse` | ✓ | ✓ | ✓ | - | - |
| `can_create_warehouse` | ✓ | ✓ | - | - | - |
| `can_edit_warehouse` | ✓ | ✓ | - | - | - |
| `can_delete_warehouses` | ✓ | ✓ | - | - | - |
| `can_view_connection` | ✓ | ✓ | ✓ | - | - |
| `can_create_connection` | ✓ | ✓ | ✓ | - | - |
| `can_view_connections` | ✓ | ✓ | ✓ | - | - |
| `can_reset_connection` | ✓ | ✓ | ✓ | - | - |
| `can_edit_connection` | ✓ | ✓ | ✓ | - | - |
| `can_delete_connection` | ✓ | ✓ | ✓ | - | - |

#### Transform (dbt)

| Permission | SA | AM | PM | An | Viewer |
|---|:---:|:---:|:---:|:---:|:---:|
| `can_create_dbt_workspace` | ✓ | ✓ | ✓ | - | - |
| `can_edit_dbt_workspace` | ✓ | ✓ | ✓ | - | - |
| `can_delete_dbt_workspace` | ✓ | ✓ | ✓ | - | - |
| `can_view_dbt_workspace` | ✓ | ✓ | ✓ | - | - |
| `can_create_dbt_docs` | ✓ | ✓ | ✓ | - | - |
| `can_create_dbt_model` | ✓ | ✓ | ✓ | - | - |
| `can_edit_dbt_operation` | ✓ | ✓ | ✓ | - | - |
| `can_view_dbt_operation` | ✓ | ✓ | ✓ | - | - |
| `can_edit_dbt_model` | ✓ | ✓ | ✓ | - | - |
| `can_view_dbt_models` | ✓ | ✓ | ✓ | - | - |
| `can_delete_dbt_model` | ✓ | ✓ | ✓ | - | - |
| `can_delete_dbt_operation` | ✓ | ✓ | ✓ | - | - |

#### Orchestrate (Pipelines & Tasks)

| Permission | SA | AM | PM | An | Viewer |
|---|:---:|:---:|:---:|:---:|:---:|
| `can_view_master_tasks` | ✓ | ✓ | ✓ | - | - |
| `can_view_master_task` | ✓ | ✓ | ✓ | - | - |
| `can_run_orgtask` | ✓ | ✓ | ✓ | - | - |
| `can_create_orgtask` | ✓ | ✓ | ✓ | - | - |
| `can_view_orgtasks` | ✓ | ✓ | ✓ | - | - |
| `can_delete_orgtask` | ✓ | ✓ | ✓ | - | - |
| `can_create_pipeline` | ✓ | ✓ | ✓ | - | - |
| `can_view_pipelines` | ✓ | ✓ | ✓ | - | - |
| `can_view_pipeline` | ✓ | ✓ | ✓ | - | - |
| `can_delete_pipeline` | ✓ | ✓ | ✓ | - | - |
| `can_edit_pipeline` | ✓ | ✓ | ✓ | - | - |
| `can_run_pipeline` | ✓ | ✓ | ✓ | - | - |
| `can_view_task_progress` | ✓ | ✓ | ✓ | - | - |

#### Visualization (Dashboards, Charts, Reports)

> **V2 spec alignment**: Pipeline Manager has full dashboard/chart access (same as Account Manager except user/warehouse management). Viewer access is **shared-only** — they see only resources explicitly shared with them, not all org resources. The `can_view_dashboards`/`can_view_charts` permission for Viewer is a role-level gate; the actual resource listing is filtered by ResourceShare records (Phase 3).

| Permission | SA | AM | PM | An | Viewer |
|---|:---:|:---:|:---:|:---:|:---:|
| `can_view_dashboards` | ✓ | ✓ | ✓ | ✓ | ✓ * |
| `can_create_dashboards` | ✓ | ✓ | ✓ | ✓ | - |
| `can_edit_dashboards` | ✓ | ✓ | ✓ | ✓ | - |
| `can_delete_dashboards` | ✓ | ✓ | ✓ | ✓ | - |
| `can_share_dashboards` | ✓ | ✓ | ✓ | ✓ | - |
| `can_manage_org_default_dashboard` | ✓ | ✓ | - | - | - |
| `can_view_charts` | ✓ | ✓ | ✓ | ✓ | ✓ * |
| `can_create_charts` | ✓ | ✓ | ✓ | ✓ | - |
| `can_edit_charts` | ✓ | ✓ | ✓ | ✓ | - |
| `can_delete_charts` | ✓ | ✓ | ✓ | ✓ | - |
| `can_view_warehouse_data` | ✓ | ✓ | ✓ | ✓ | ✓ |
| **`can_view_reports`** (NEW) | ✓ | ✓ | ✓ | ✓ | ✓ * |
| **`can_create_reports`** (NEW) | ✓ | ✓ | ✓ | ✓ | - |
| **`can_edit_reports`** (NEW) | ✓ | ✓ | ✓ | ✓ | - |
| **`can_delete_reports`** (NEW) | ✓ | ✓ | ✓ | ✓ | - |
| **`can_share_reports`** (NEW) | ✓ | ✓ | ✓ | ✓ | - |

> \* **Viewer**: Has the role-level permission but listing endpoints filter to only show resources explicitly shared with them via `ResourceShare` (Phase 3). Without any shares, Viewers see empty dashboards/charts/reports lists.

#### User & Org Management

> **V2 spec alignment**: Analysts can invite new users as Viewers via the sharing modal. The `can_create_invitation` permission is granted to Analysts, but the backend enforces that Analysts can **only** invite with the Viewer role. Account Managers can invite any role.

| Permission | SA | AM | PM | An | Viewer |
|---|:---:|:---:|:---:|:---:|:---:|
| `can_create_org` | ✓ | - | - | - | - |
| `can_view_orgusers` | ✓ | ✓ | ✓ | ✓ | - |
| `can_create_orguser` | ✓ | ✓ | - | - | - |
| `can_delete_orguser` | ✓ | ✓ | - | - | - |
| `can_edit_orguser` | ✓ | ✓ | - | - | - |
| `can_edit_orguser_role` | ✓ | ✓ | - | - | - |
| `can_create_invitation` | ✓ | ✓ | - | ✓ ** | - |
| `can_view_invitations` | ✓ | ✓ | ✓ | ✓ | - |
| `can_edit_invitation` | ✓ | ✓ | - | - | - |
| `can_delete_invitation` | ✓ | ✓ | - | - | - |
| `can_resend_email_verification` | ✓ | ✓ | - | - | - |

> \*\* **Analyst invitation constraint**: Backend enforces that when role=Analyst, the invitation API only allows `invited_role=viewer`. This is triggered from the share modal paste-to-share flow when an email is unmatched (user doesn't exist on Dalgo). The Analyst cannot choose the invitee's role — it is always Viewer.

#### Settings & Platform

| Permission | SA | AM | PM | An | Viewer |
|---|:---:|:---:|:---:|:---:|:---:|
| `public` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `can_accept_tnc` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `can_view_flags` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `can_edit_llm_settings` | ✓ | ✓ | - | - | - |
| `can_edit_org_notification_settings` | ✓ | ✓ | - | - | - |
| `can_initiate_org_plan_upgrade` | ✓ | ✓ | - | - | - |
| `can_request_llm_analysis_feature` | - | - | ✓ | ✓ | ✓ |
| `can_view_usage_dashboard` | ✓ | ✓ | ✓ | ✓ | - |

### Summary of Changes from Current State (V2 Spec Aligned)

| Change | Current | Proposed | Impact |
|---|---|---|---|
| **Rename Guest → Viewer** | `guest` (slug) | `viewer` (slug) | Cosmetic + clearer intent |
| **Strip Analyst of dbt write access** | Analyst has 10 dbt write perms | Analyst has 0 dbt perms | **Breaking** — existing Analysts lose dbt access |
| **Strip Analyst of task/pipeline write** | Analyst can create/run/delete tasks | Analyst has no orchestration perms | **Breaking** |
| **Strip Analyst of sync sources** | Analyst can trigger syncs | No source access | **Breaking** |
| **Strip Analyst of infrastructure view** | Analyst sees sources, warehouses, connections, pipelines | No infrastructure visibility | **Breaking** |
| **Pipeline Mgr keeps dashboard/chart CRUD** | PM can create/edit/delete | PM **keeps** full dashboard/chart CRUD | No change — aligned with V2 spec ("same as AM except user mgmt & warehouse") |
| **Strip Guest/Viewer of infrastructure view** | Guest sees 15+ infrastructure perms | Viewer sees only shared dashboards/charts/reports | Simplification |
| **Viewer sees only shared resources** | Guest sees all org dashboards (if had permission) | Viewer sees **only** resources explicitly shared with them | **Requires Phase 3** — ResourceShare model must be in place |
| **Analyst can invite Viewers** | Only SA/AM can invite | Analyst gets `can_create_invitation` (backend-limited to Viewer role) | New capability — from share modal paste-to-share flow |
| **Chart inheritance from dashboards** | No inheritance | Charts inside shared dashboard auto-shared with same permission | New — inherited ResourceShare records with cascade |
| **Add report-specific permissions** | Reports use `can_*_dashboards` | 5 new `can_*_reports` permissions | Non-breaking — new permissions |
| **Add `can_accept_tnc` to Viewer** | Guest can't accept T&C | Viewer can accept T&C | Fix — Viewers need to accept T&C |
| **Proposed new permission count** | SA:72, AM:71, PM:57, An:46, Gu:24 | SA:72, AM:71, PM:57, An:22, Vi:11 | Analyst tightened, PM unchanged |

### Open Question: Do some NGOs need a "Full Access" role?

In many NGOs, one person does **both** pipeline management and data analysis. Under the proposed model, they'd need Account Manager access — which also gives user management powers.

**Options:**
- **A) No new role**: Promote those users to Account Manager. Simple but grants unnecessary user management access.
- **B) Add a "Data Manager" role** (level 3.5): Gets Pipeline Manager + Analyst permissions without user/org management. Adds complexity but solves the overlap problem cleanly.
- **C) Support multiple roles per user**: Most flexible, but requires significant architecture change (current system is single-role).

---

## Risk Notes

1. **Analyst restriction (Phase 1)**: Breaking change. Need migration script + communication.
2. **RLS performance (Phase 4)**: Extra WHERE clauses on every query. Mitigate with Redis caching of resolved conditions.
3. **Backward compatibility**: Phase 1 changes Guest/Analyst roles. Orgs using current permissions will be affected.
4. **ContentType GenericFK (Phase 3)**: Slightly harder to query than direct FKs, but polymorphism benefit outweighs cost.
