# Dalgo — User Groups & Resource-Level Sharing Plan

## Document Status
- **Created**: 2026-04-16
- **Status**: PLAN COMPLETE — Ready for review
- **Depends on**: Unified Access Control Plan (Phase 1 role fixes should ship first)
- **Linear**: DALGO-1204

---

## Problem Statement

An NGO has 3 dashboards: **x**, **y**, **z**. They want their "Field Team" (5 users) to see only **x** and **y**, not **z**. Today this is impossible — users either see all org dashboards (Analyst+) or nothing (Viewer has no `can_view_dashboards`).

---

## Solution Overview

Three layers working together:

```
Layer 1: Roles (who can do what platform-wide)
  └── Account Manager, Pipeline Manager, Analyst, Viewer

Layer 2: User Groups (organize people)
  └── "Field Team", "Finance Team", "Partners"

Layer 3: Resource Shares (who can see which dashboard/chart)
  └── Dashboard x shared with "Field Team" (view)
  └── Dashboard y shared with "Field Team" (edit)
```

---

## Part 1: Data Models

### 1.1 UserGroup

**File**: `ddpui/models/user_groups.py` (new)

```python
class UserGroup(models.Model):
    id = models.BigAutoField(primary_key=True)
    org = models.ForeignKey(Org, on_delete=models.CASCADE, related_name="user_groups")
    name = models.CharField(max_length=100)           # "Field Team"
    description = models.TextField(blank=True, null=True)
    created_by = models.ForeignKey(OrgUser, on_delete=models.CASCADE, related_name="groups_created")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "user_group"
        unique_together = ("org", "name")
        ordering = ["name"]
```

### 1.2 UserGroupMember

```python
class UserGroupMember(models.Model):
    id = models.BigAutoField(primary_key=True)
    user_group = models.ForeignKey(UserGroup, on_delete=models.CASCADE, related_name="members")
    orguser = models.ForeignKey(OrgUser, on_delete=models.CASCADE, related_name="group_memberships")
    added_by = models.ForeignKey(OrgUser, on_delete=models.CASCADE, related_name="group_additions")
    added_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = "user_group_member"
        unique_together = ("user_group", "orguser")
```

### 1.3 ResourceShare

**File**: `ddpui/models/resource_sharing.py` (new)

```python
class SharePermission(models.TextChoices):
    VIEW = "view"
    EDIT = "edit"

class ShareStatus(models.TextChoices):
    ACTIVE = "active"
    PENDING = "pending"    # invited user hasn't accepted yet

class ResourceType(models.TextChoices):
    DASHBOARD = "dashboard"
    CHART = "chart"

class ResourceShare(models.Model):
    id = models.BigAutoField(primary_key=True)
    org = models.ForeignKey(Org, on_delete=models.CASCADE)

    # What is being shared
    resource_type = models.CharField(max_length=20, choices=ResourceType.choices)
    resource_id = models.IntegerField()

    # Who it's shared with (exactly one of these three must be set)
    shared_with_user = models.ForeignKey(
        OrgUser, null=True, blank=True, on_delete=models.CASCADE,
        related_name="resource_shares_received"
    )
    shared_with_group = models.ForeignKey(
        UserGroup, null=True, blank=True, on_delete=models.CASCADE,
        related_name="resource_shares"
    )
    shared_with_email = models.EmailField(null=True, blank=True)  # pending invite

    # Permission level
    permission = models.CharField(
        max_length=10, choices=SharePermission.choices, default=SharePermission.VIEW
    )
    status = models.CharField(
        max_length=10, choices=ShareStatus.choices, default=ShareStatus.ACTIVE
    )

    # Inheritance tracking (chart inherited from dashboard share)
    is_inherited = models.BooleanField(default=False)
    parent_share = models.ForeignKey(
        "self", null=True, blank=True, on_delete=models.CASCADE,
        related_name="inherited_shares"
    )

    # Metadata
    shared_by = models.ForeignKey(OrgUser, on_delete=models.CASCADE, related_name="resource_shares_given")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "resource_share"
        indexes = [
            models.Index(fields=["resource_type", "resource_id"]),
            models.Index(fields=["shared_with_user"]),
            models.Index(fields=["shared_with_group"]),
            models.Index(fields=["shared_with_email"]),
        ]
        constraints = [
            # At least one target must be set
            models.CheckConstraint(
                check=(
                    models.Q(shared_with_user__isnull=False) |
                    models.Q(shared_with_group__isnull=False) |
                    models.Q(shared_with_email__isnull=False)
                ),
                name="resource_share_has_target"
            )
        ]
```

**Why not Django ContentType GenericFK?**
- Simpler queries (no extra join to `content_type` table)
- Only 2 resource types (dashboard, chart) — enum is clearer
- Easier to reason about in access checks

---

## Part 2: Access Logic Per Role

### 2.1 Who sees what

| Role | Dashboard/Chart Listing | How |
|------|------------------------|-----|
| **Account Manager** | ALL org dashboards/charts | `Dashboard.objects.filter(org=org)` — no change from today |
| **Pipeline Manager** | ALL org dashboards/charts | Same as AM (V2 spec: "same access except user mgmt & warehouse") |
| **Analyst** | ALL org dashboards/charts | Same as today — Analysts have `can_view/create/edit/delete_dashboards/charts` |
| **Viewer** | ONLY shared dashboards/charts | Filtered by `ResourceShare` (direct + group + inherited) |

### 2.2 Who can share

| Action | AM | PM | Analyst | Viewer |
|--------|:--:|:--:|:-------:|:------:|
| Share own dashboard/chart with user | Yes | Yes | Yes | No |
| Share own dashboard/chart with group | Yes | Yes | Yes | No |
| Share ANY dashboard/chart | Yes | No | No | No |
| Create user groups | Yes | Yes | Yes | No |
| Manage group members | Yes | Own groups | Own groups | No |
| Invite unmatched email as Viewer | Yes | Yes | Yes (backend-enforced) | No |

### 2.3 Access check function

**File**: `ddpui/core/access_service.py` (new)

```python
def can_access_resource(orguser: OrgUser, resource_type: str, resource_id: int, required: str = "view") -> bool:
    """
    Check if user can access a specific resource.

    Order of checks:
    1. Is user the creator? → Yes
    2. Is user's role AM/PM/Analyst? → Yes (org-wide access for these roles)
    3. Is there a direct ResourceShare for this user? → Check permission level
    4. Is there a group ResourceShare where user is a member? → Check permission level
    5. Is the resource public? → View only
    6. → Denied
    """
    ...

def get_accessible_dashboards(orguser: OrgUser, **filters) -> QuerySet:
    """
    Return dashboards the user can access.

    - AM/PM/Analyst: All org dashboards (existing behavior)
    - Viewer: Only dashboards shared directly, via group, or public
    """
    base_qs = Dashboard.objects.filter(org=orguser.org)

    if orguser.new_role.slug in ("super-admin", "account-manager", "pipeline-manager", "analyst"):
        # Org-wide access — existing behavior, no change
        return apply_filters(base_qs, **filters)

    # Viewer: shared-only access
    user_groups = UserGroup.objects.filter(members__orguser=orguser)

    shared_dashboard_ids = ResourceShare.objects.filter(
        org=orguser.org,
        resource_type="dashboard",
        status="active",
    ).filter(
        Q(shared_with_user=orguser) |
        Q(shared_with_group__in=user_groups)
    ).values_list("resource_id", flat=True)

    public_dashboard_ids = Dashboard.objects.filter(
        org=orguser.org, is_public=True
    ).values_list("id", flat=True)

    return apply_filters(
        base_qs.filter(
            Q(id__in=shared_dashboard_ids) |
            Q(id__in=public_dashboard_ids) |
            Q(created_by=orguser)  # own dashboards (shouldn't exist for Viewer, but safe)
        ),
        **filters
    )

def get_accessible_charts(orguser: OrgUser, **filters) -> QuerySet:
    """
    Return charts the user can access.

    - AM/PM/Analyst: All org charts (existing behavior)
    - Viewer: Only charts shared directly, via group, or inherited from dashboard
    """
    base_qs = Chart.objects.filter(org=orguser.org)

    if orguser.new_role.slug in ("super-admin", "account-manager", "pipeline-manager", "analyst"):
        return apply_filters(base_qs, **filters)

    # Viewer: shared + inherited charts
    user_groups = UserGroup.objects.filter(members__orguser=orguser)

    shared_chart_ids = ResourceShare.objects.filter(
        org=orguser.org,
        resource_type="chart",
        status="active",
    ).filter(
        Q(shared_with_user=orguser) |
        Q(shared_with_group__in=user_groups)
    ).values_list("resource_id", flat=True)

    return apply_filters(
        base_qs.filter(id__in=shared_chart_ids),
        **filters
    )
```

---

## Part 3: Chart Inheritance from Dashboards

### 3.1 The problem

Dashboard `components` is a JSON field. Charts are referenced by `chartId` inside:
```json
{
  "comp_1": {"type": "chart", "config": {"chartId": 42}},
  "comp_2": {"type": "chart", "config": {"chartId": 57}},
  "comp_3": {"type": "text", "config": {"content": "..."}}
}
```

There is no FK relationship. We need to extract chart IDs from this JSON when sharing a dashboard.

### 3.2 How inheritance works

When dashboard X is shared (with user or group):

```
1. Create ResourceShare for dashboard X
2. Extract chart IDs from dashboard.components JSON
3. For each chart ID:
   a. Create ResourceShare for that chart with:
      - is_inherited = True
      - parent_share = the dashboard ResourceShare
      - Same permission level as dashboard share
      - Same target (user or group)
4. Inherited chart shares appear in "My Charts" for the recipient
```

### 3.3 Cascade rules

| Dashboard action | Chart inheritance effect |
|-----------------|------------------------|
| Share dashboard with user/group | Create inherited chart shares |
| Change dashboard share permission (view→edit) | Update inherited chart shares |
| Revoke dashboard share | Delete inherited chart shares (via `parent_share` cascade) |
| Add new chart to dashboard | Create inherited share for new chart (for all existing dashboard shares) |
| Remove chart from dashboard | Delete inherited shares for that chart |
| Dashboard components JSON updated | Diff old vs new chart IDs, create/delete inherited shares |

### 3.4 Implementation

**File**: `ddpui/core/sharing_service.py` (new)

```python
def get_dashboard_chart_ids(dashboard: Dashboard) -> List[int]:
    """Extract chart IDs from dashboard.components JSON."""
    chart_ids = []
    if dashboard.components:
        for component_id, component in dashboard.components.items():
            if component.get("type") == "chart":
                chart_id = component.get("config", {}).get("chartId")
                if chart_id:
                    chart_ids.append(chart_id)
    return chart_ids

def share_dashboard(
    sharer: OrgUser,
    dashboard: Dashboard,
    target_user: Optional[OrgUser] = None,
    target_group: Optional[UserGroup] = None,
    target_email: Optional[str] = None,
    permission: str = "view",
) -> ResourceShare:
    """Share a dashboard and create inherited chart shares."""

    # 1. Create dashboard share
    dashboard_share = ResourceShare.objects.create(
        org=dashboard.org,
        resource_type="dashboard",
        resource_id=dashboard.id,
        shared_with_user=target_user,
        shared_with_group=target_group,
        shared_with_email=target_email,
        permission=permission,
        status="pending" if target_email and not target_user else "active",
        shared_by=sharer,
    )

    # 2. Create inherited chart shares
    chart_ids = get_dashboard_chart_ids(dashboard)
    for chart_id in chart_ids:
        ResourceShare.objects.create(
            org=dashboard.org,
            resource_type="chart",
            resource_id=chart_id,
            shared_with_user=target_user,
            shared_with_group=target_group,
            shared_with_email=target_email,
            permission=permission,
            status=dashboard_share.status,
            is_inherited=True,
            parent_share=dashboard_share,
            shared_by=sharer,
        )

    return dashboard_share

def sync_dashboard_chart_shares(dashboard: Dashboard):
    """
    Called when dashboard.components changes.
    Ensures inherited chart shares match current dashboard charts.
    """
    current_chart_ids = set(get_dashboard_chart_ids(dashboard))

    # Get all dashboard-level shares
    dashboard_shares = ResourceShare.objects.filter(
        resource_type="dashboard",
        resource_id=dashboard.id,
    )

    for dashboard_share in dashboard_shares:
        # Get existing inherited chart shares for this dashboard share
        existing_inherited = ResourceShare.objects.filter(
            parent_share=dashboard_share,
            is_inherited=True,
        )
        existing_chart_ids = set(
            existing_inherited.values_list("resource_id", flat=True)
        )

        # Add missing charts
        for chart_id in current_chart_ids - existing_chart_ids:
            ResourceShare.objects.create(
                org=dashboard.org,
                resource_type="chart",
                resource_id=chart_id,
                shared_with_user=dashboard_share.shared_with_user,
                shared_with_group=dashboard_share.shared_with_group,
                shared_with_email=dashboard_share.shared_with_email,
                permission=dashboard_share.permission,
                status=dashboard_share.status,
                is_inherited=True,
                parent_share=dashboard_share,
                shared_by=dashboard_share.shared_by,
            )

        # Remove charts no longer in dashboard
        existing_inherited.filter(
            resource_id__in=existing_chart_ids - current_chart_ids
        ).delete()
```

### 3.5 Hook: Sync on dashboard save

In `DashboardService.update_dashboard()` (or wherever `dashboard.components` is saved), call:
```python
# After dashboard.save()
sync_dashboard_chart_shares(dashboard)
```

---

## Part 4: Paste-to-Share Flow

### 4.1 How it works

User opens share modal → pastes multiple emails → backend processes each:

```
Input: ["alice@ngo.org", "bob@ngo.org", "unknown@new.org"]

Step 1: Lookup each email against OrgUser in the same org
  ├── alice@ngo.org  → OrgUser found (Analyst)   → MATCHED
  ├── bob@ngo.org    → OrgUser found (Viewer)     → MATCHED
  └── unknown@new.org → No OrgUser                → UNMATCHED

Step 2: For matched users → create ResourceShare immediately
  ├── alice: ResourceShare(shared_with_user=alice, permission=view, status=active)
  └── bob:   ResourceShare(shared_with_user=bob, permission=view, status=active)

Step 3: For unmatched emails → show in UI with options
  └── unknown@new.org: "Not found on Dalgo. [Invite as Viewer] [Remove]"

Step 4: If user clicks "Invite as Viewer":
  ├── Create Invitation(invited_email="unknown@new.org", invited_new_role=Viewer)
  ├── Create ResourceShare(shared_with_email="unknown@new.org", status=pending)
  └── Send invitation email

Step 5: When invited user accepts:
  ├── Create OrgUser with Viewer role
  ├── Find all pending ResourceShares with shared_with_email
  ├── Update: shared_with_user = new OrgUser, shared_with_email = null, status = active
  └── User can now see shared dashboards
```

### 4.2 API endpoint

**POST** `/sharing/{resource_type}/{resource_id}/`

```python
class ShareRequest(Schema):
    emails: List[str]                    # ["alice@ngo.org", "bob@ngo.org"]
    permission: str = "view"             # "view" or "edit"
    group_id: Optional[int] = None       # share with group instead of emails

class ShareResponse(Schema):
    shared: List[ShareResultItem]        # successfully shared
    unmatched: List[UnmatchedEmail]      # emails not found
    already_shared: List[str]            # emails already have access

class UnmatchedEmail(Schema):
    email: str
    can_invite: bool                     # True if sharer has can_create_invitation
```

### 4.3 Analyst invitation constraint

When the sharer is an Analyst:
- Backend checks `orguser.new_role.slug == "analyst"`
- Invitation is created with `invited_new_role = Role.objects.get(slug="viewer")`
- No role selection UI is shown — always Viewer
- If sharer is AM, they can choose any role (existing invitation flow)

---

## Part 5: API Endpoints

### 5.1 User Groups

**File**: `ddpui/api/user_group_api.py` (new)

| Endpoint | Method | Permission | Description |
|----------|--------|-----------|-------------|
| `/groups/` | GET | `can_view_orgusers` | List all groups in org |
| `/groups/` | POST | `can_share_dashboards` | Create a group |
| `/groups/{id}/` | GET | `can_view_orgusers` | Get group details + members |
| `/groups/{id}/` | PUT | `can_share_dashboards` | Update group name/description |
| `/groups/{id}/` | DELETE | `can_share_dashboards` | Delete group (cascades shares) |
| `/groups/{id}/members/` | POST | `can_share_dashboards` | Add members to group |
| `/groups/{id}/members/{orguser_id}/` | DELETE | `can_share_dashboards` | Remove member from group |

**Group creation constraint**: Only group creator or AM can manage group membership.

### 5.2 Resource Sharing

**File**: `ddpui/api/sharing_api.py` (new)

| Endpoint | Method | Permission | Description |
|----------|--------|-----------|-------------|
| `/sharing/{type}/{id}/` | GET | `can_view_dashboards` or `can_view_charts` | List all shares for a resource |
| `/sharing/{type}/{id}/` | POST | `can_share_dashboards` or `can_share_charts` | Share with users/group (paste-to-share) |
| `/sharing/{type}/{id}/{share_id}/` | PATCH | `can_share_dashboards` or `can_share_charts` | Change permission level |
| `/sharing/{type}/{id}/{share_id}/` | DELETE | `can_share_dashboards` or `can_share_charts` | Revoke share |
| `/sharing/{type}/{id}/invite/` | POST | `can_create_invitation` | Invite unmatched email + create pending share |
| `/sharing/users/search/?q=` | GET | `can_view_orgusers` | Search org users for share picker |

**Sharing constraint**: Only the resource creator or AM can share. AM can share any resource.

### 5.3 Modified existing endpoints

| Endpoint | Change |
|----------|--------|
| `GET /dashboards/` | Use `get_accessible_dashboards()` instead of `Dashboard.objects.filter(org=org)` |
| `GET /charts/` | Use `get_accessible_charts()` instead of `Chart.objects.filter(org=org)` |
| `PUT /dashboards/{id}/` | After save, call `sync_dashboard_chart_shares()` if components changed |
| `POST /organizations/users/invite/accept/` | After accept, activate pending ResourceShares for that email |

---

## Part 6: New Permissions

| Permission | PK | Assigned to |
|-----------|-----|-------------|
| `can_share_charts` | 77 | SA, AM, PM, Analyst |
| `can_manage_groups` | 78 | SA, AM, PM, Analyst |

Viewer gets none of these — Viewers consume, they don't share or manage.

---

## Part 7: Frontend Changes

### 7.1 Share Modal Enhancement

**File**: `webapp_v2/components/ui/share-modal.tsx`

Current cards:
1. Organization Access (read-only info)
2. Public Access (toggle)
3. Share via Email (reports only)

**Add new cards**:

```
┌─────────────────────────────────────────────┐
│ Share Dashboard                           ×  │
│                                              │
│ ┌──────────────────────────────────────────┐ │
│ │ Organization Access              Default │ │  ← Existing
│ │ Users with proper role can access this   │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ ┌──────────────────────────────────────────┐ │
│ │ Public Access                   [Toggle] │ │  ← Existing
│ │ Anyone with the link can view            │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ ┌──────────────────────────────────────────┐ │
│ │ Share with People or Groups        [NEW] │ │  ← NEW
│ │                                          │ │
│ │ [Paste emails or search users/groups...] │ │  ← Input
│ │                                          │ │
│ │ ⚠ unknown@new.org — not on Dalgo         │ │  ← Unmatched
│ │   [Invite as Viewer] [Remove]            │ │
│ │                                          │ │
│ │ Shared with (3):                         │ │
│ │ ┌────────────────────────────────────┐   │ │
│ │ │ 👤 Alice Lee      [View ▾]    [×] │   │ │  ← User share
│ │ │ 👤 Bob Chen       [Edit ▾]    [×] │   │ │
│ │ │ 👥 Field Team     [View ▾]    [×] │   │ │  ← Group share
│ │ └────────────────────────────────────┘   │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ ┌──────────────────────────────────────────┐ │
│ │ Share via Email               (Reports)  │ │  ← Existing
│ └──────────────────────────────────────────┘ │
│                                              │
│                                    [Close]   │
└──────────────────────────────────────────────┘
```

**Search input behavior**:
- Types `@` → shows user results
- Types group name → shows group results (prefixed with 👥)
- Pastes multiple emails (comma/newline separated) → batch lookup
- Debounce: 300ms

**Permission dropdown**: "View" (default) or "Edit"

### 7.2 User Group Management Page

**Route**: `/app/settings/user-groups`

```
┌──────────────────────────────────────────────────┐
│ User Groups                    [+ Create Group]  │
│ Organize users for easier sharing                │
│                                                  │
│ ┌────────────────────────────────────────────┐   │
│ │ Name         Members   Shared Resources    │   │
│ ├────────────────────────────────────────────┤   │
│ │ Field Team   5 users   3 dashboards        │   │
│ │ Finance      3 users   2 dashboards        │   │
│ │ Partners     8 users   1 dashboard         │   │
│ └────────────────────────────────────────────┘   │
│                                                  │
│ Empty state:                                     │
│ "No groups yet. Create a group to share          │
│  dashboards with multiple people at once."       │
└──────────────────────────────────────────────────┘
```

**Group detail page** (`/app/settings/user-groups/{id}`):

```
┌──────────────────────────────────────────────────┐
│ ← Back    Field Team                    [Edit]   │
│ Field staff across all regions                   │
│                                                  │
│ [Members (5)]  [Shared Dashboards (3)]           │
│                                                  │
│ Members:                          [+ Add Member] │
│ ┌────────────────────────────────────────────┐   │
│ │ Name            Email            Role  [×] │   │
│ │ Alice Lee       alice@ngo.org    Viewer [×]│   │
│ │ Bob Chen        bob@ngo.org      Viewer [×]│   │
│ │ Jane Smith      jane@ngo.org     Analyst[×]│   │
│ └────────────────────────────────────────────┘   │
│                                                  │
│ Shared Dashboards:                               │
│ ┌────────────────────────────────────────────┐   │
│ │ Dashboard           Permission             │   │
│ │ Program Outcomes    View                   │   │
│ │ Field Activity      Edit                   │   │
│ │ Regional Summary    View                   │   │
│ └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

### 7.3 Dashboard/Chart List Indicators

In the dashboard list (`/app/dashboards`), add sharing badges:

```
┌─────────────────────────────────────┐
│ Program Outcomes                     │
│ Updated 2 hours ago                  │
│ 👥 Shared with: Field Team, Finance  │  ← NEW badge
│ 🔗 Public                            │  ← Existing
└─────────────────────────────────────┘
```

In the chart list, show inherited access:

```
┌─────────────────────────────────────┐
│ Monthly Revenue                      │
│ Bar Chart · By Alice Lee             │
│ 📎 via "Program Outcomes" dashboard  │  ← Inherited indicator
└─────────────────────────────────────┘
```

### 7.4 Viewer Experience

When a Viewer logs in:

```
┌──────────────┬──────────────────────────────────┐
│ [Sidebar]    │ Shared with you                   │
│              │                                    │
│ Dashboards   │ Dashboards (2)                    │
│ Charts       │ ┌──────────────┬────────────────┐│
│ Reports      │ │ Program      │ Field Activity ││
│ Settings     │ │ Outcomes     │                ││
│   └ About    │ │ 12 charts    │ 8 charts       ││
│              │ │ View         │ Edit           ││
│              │ └──────────────┴────────────────┘│
│              │                                    │
│              │ Charts (15)                        │
│              │ 3 directly shared + 12 via dashboards│
│              │ ┌──────────┬──────────┬──────────┐│
│              │ │ Chart 1  │ Chart 2  │ Chart 3  ││
│              │ │ 📎 via PO│ 📎 via PO│ Direct   ││
│              │ └──────────┴──────────┴──────────┘│
└──────────────┴──────────────────────────────────┘
```

**What Viewer sees**:
- Dashboards shared directly or via group
- Charts shared directly, via group, or inherited from shared dashboards
- Permission badge: "View" or "Edit" per resource
- Inherited charts marked with source dashboard name

**What Viewer doesn't see**:
- Create/Edit/Delete buttons (unless they have Edit share permission)
- Share button
- Settings > User Management
- Data, Pipeline, Transform sections in sidebar

---

## Part 8: Detailed Scenario Walkthrough

### Scenario: Share dashboards x and y with Field Team

**Step 1**: Account Manager creates group
```
POST /groups/
{ "name": "Field Team", "description": "Field staff across all regions" }
→ 201: { "id": 1, "name": "Field Team" }
```

**Step 2**: Add members
```
POST /groups/1/members/
{ "orguser_ids": [10, 11, 12, 13, 14] }
→ 200: { "added": 5 }
```

**Step 3**: Share dashboard x with Field Team (view)
```
POST /sharing/dashboard/100/
{ "group_id": 1, "permission": "view" }

→ Backend creates:
  ResourceShare(resource_type=dashboard, resource_id=100, shared_with_group=1, permission=view)
  ResourceShare(resource_type=chart, resource_id=42, shared_with_group=1, permission=view, is_inherited=True, parent_share=above)
  ResourceShare(resource_type=chart, resource_id=57, shared_with_group=1, permission=view, is_inherited=True, parent_share=above)
```

**Step 4**: Share dashboard y with Field Team (edit)
```
POST /sharing/dashboard/101/
{ "group_id": 1, "permission": "edit" }

→ Backend creates:
  ResourceShare(resource_type=dashboard, resource_id=101, shared_with_group=1, permission=edit)
  ResourceShare(resource_type=chart, resource_id=63, shared_with_group=1, permission=edit, is_inherited=True, parent_share=above)
```

**Result**: Viewer user #10 (member of Field Team) now sees:
- Dashboard x (view) + its 2 charts (view)
- Dashboard y (edit) + its 1 chart (edit)
- Does NOT see dashboard z

**Step 5**: New user joins Field Team
```
POST /groups/1/members/
{ "orguser_ids": [15] }

→ User 15 immediately sees dashboards x and y
  (no additional sharing needed — group membership is checked at query time)
```

---

## Part 9: Edge Cases

### 9.1 Chart shared via multiple dashboards

Chart #42 is in dashboard x (shared as view) AND dashboard y (shared as edit):
- Two inherited ResourceShares exist with different permissions
- Access check uses **highest permission** (edit wins over view)
- UI shows both sources: "📎 via Dashboard x, Dashboard y"

### 9.2 Direct share overrides inherited share

If chart #42 is inherited as "view" from dashboard x, but directly shared as "edit":
- Direct share takes precedence
- UI shows "Edit (directly shared)" not "📎 via dashboard"

### 9.3 User is in multiple groups with different permissions

User is in "Field Team" (view on dashboard x) and "Editors" (edit on dashboard x):
- Highest permission wins (edit)
- This is resolved at query time, not stored

### 9.4 Group is deleted

```
DELETE /groups/1/
→ CASCADE: All ResourceShare with shared_with_group=1 are deleted
→ Members lose access to dashboards shared via that group
→ Direct user shares are unaffected
```

### 9.5 Dashboard components change (chart added/removed)

Chart #99 added to dashboard x (which is shared with Field Team):
```
PUT /dashboards/100/  (components updated)
→ sync_dashboard_chart_shares() runs
→ Creates: ResourceShare(chart, 99, group=Field Team, view, inherited)

Field Team members now see chart #99 in their "My Charts"
```

### 9.6 Unmatched email user finally accepts invite

```
1. Analyst shares dashboard x with "unknown@new.org"
2. ResourceShare created: shared_with_email="unknown@new.org", status=pending
3. Invitation sent
4. User accepts invitation → OrgUser created with Viewer role
5. POST /organizations/users/invite/accept/ triggers:
   → Find ResourceShare(shared_with_email="unknown@new.org")
   → Update: shared_with_user = new OrgUser, shared_with_email = null, status = active
6. User can now see dashboard x
```

### 9.7 Performance considerations

**Query for Viewer dashboard listing**:
```sql
SELECT d.* FROM dashboard d
WHERE d.org_id = %s
AND (
    d.id IN (
        SELECT rs.resource_id FROM resource_share rs
        WHERE rs.resource_type = 'dashboard'
        AND rs.status = 'active'
        AND (
            rs.shared_with_user_id = %s
            OR rs.shared_with_group_id IN (
                SELECT ugm.user_group_id FROM user_group_member ugm
                WHERE ugm.orguser_id = %s
            )
        )
    )
    OR d.is_public = TRUE
)
ORDER BY d.updated_at DESC
```

Indexes already defined on `resource_share`:
- `(resource_type, resource_id)` — for listing shares on a resource
- `(shared_with_user)` — for listing what's shared with a user
- `(shared_with_group)` — for listing what's shared with a group

For orgs with 50+ dashboards and 100+ shares, this should be fast. Can add Redis caching later if needed.

---

## Part 10: Implementation Phases

### Phase 3A: User Groups (1 week)
- [ ] `UserGroup` and `UserGroupMember` models + migrations
- [ ] Group API endpoints (CRUD + member management)
- [ ] Frontend: Group management page in Settings
- [ ] New permissions: `can_manage_groups`, `can_share_charts`
- [ ] Seed data update

### Phase 3B: Resource Sharing — Backend (1-2 weeks)
- [ ] `ResourceShare` model + migrations
- [ ] `access_service.py` — `can_access_resource()`, `get_accessible_dashboards/charts()`
- [ ] `sharing_service.py` — `share_dashboard()`, `sync_dashboard_chart_shares()`
- [ ] Sharing API endpoints (CRUD)
- [ ] Paste-to-share flow (matched/unmatched email handling)
- [ ] Analyst invite-as-Viewer constraint
- [ ] Modify dashboard listing to use `get_accessible_dashboards()`
- [ ] Modify chart listing to use `get_accessible_charts()`
- [ ] Hook `sync_dashboard_chart_shares()` into dashboard save
- [ ] Activate pending shares on invitation acceptance

### Phase 3C: Resource Sharing — Frontend (1-2 weeks)
- [ ] Enhanced share modal (user/group search, permission dropdown, unmatched email handling)
- [ ] Sharing badges on dashboard/chart list views
- [ ] Inherited chart indicator ("📎 via Dashboard X")
- [ ] Viewer experience (shared-only listing, no create/edit buttons, view-only banners)
- [ ] Group selector in share modal

### Phase 3D: Testing & Polish (1 week)
- [ ] Unit tests for access_service, sharing_service
- [ ] Integration tests for paste-to-share flow
- [ ] Test chart inheritance (add/remove charts from dashboard)
- [ ] Test group membership changes (add/remove members)
- [ ] Test pending invite → acceptance flow
- [ ] Test edge cases (multiple groups, conflicting permissions)
- [ ] Performance testing with 100+ shares

---

## Part 11: Files to Create / Modify

### New files
| File | Purpose |
|------|---------|
| `ddpui/models/user_groups.py` | UserGroup, UserGroupMember models |
| `ddpui/models/resource_sharing.py` | ResourceShare model |
| `ddpui/core/access_service.py` | Access check logic, resource listing |
| `ddpui/core/sharing_service.py` | Share/unshare, chart inheritance, paste-to-share |
| `ddpui/api/user_group_api.py` | Group CRUD + member management endpoints |
| `ddpui/api/sharing_api.py` | Resource sharing endpoints |
| `ddpui/schemas/sharing_schema.py` | Request/response schemas |
| `ddpui/schemas/user_group_schema.py` | Request/response schemas |
| `webapp_v2/app/settings/user-groups/page.tsx` | Group management page |
| `webapp_v2/app/settings/user-groups/[id]/page.tsx` | Group detail page |
| `webapp_v2/hooks/api/useGroups.ts` | Group API hooks |
| `webapp_v2/hooks/api/useSharing.ts` | Sharing API hooks |

### Modified files
| File | Change |
|------|--------|
| `ddpui/api/dashboard_native_api.py` | `list_dashboards()` → use `get_accessible_dashboards()` |
| `ddpui/api/charts_api.py` | `list_charts()` → use `get_accessible_charts()` |
| `ddpui/core/dashboard/dashboard_service.py` | Call `sync_dashboard_chart_shares()` on component update |
| `ddpui/api/user_org_api.py` | On invitation accept → activate pending ResourceShares |
| `seed/002_permissions.json` | Add `can_share_charts`(77), `can_manage_groups`(78) |
| `seed/003_role_permissions.json` | Assign new permissions to roles |
| `webapp_v2/components/ui/share-modal.tsx` | Add user/group sharing section |
| `webapp_v2/components/main-layout.tsx` | Add User Groups nav item under Settings |
