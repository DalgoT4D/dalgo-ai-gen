# Dalgo Access Control — High Level Plan

---

## Where We Are Today

- **5 roles**: Super Admin, Account Manager, Pipeline Manager, Analyst, Guest
- **Broken permissions**:
  - Guest can't view dashboards/charts (missing permissions)
  - Analyst has full pipeline/dbt write access — nearly same as Pipeline Manager
  - `has_schema_access()` is a TODO — any user can query any table
- **No per-resource sharing** — users with the right role see ALL org dashboards, or nothing
- **No user groups** — can't organize users into teams
- **Sidebar not gated by role** — everyone sees all nav items
- **No route protection** — unauthorized pages accessible via direct URL
- **Reports reuse dashboard permissions**
- **Public sharing works** — token-based, creator-only. No user-to-user sharing.

---

## What We're Building

### 1. Fix Roles & Permissions

Tighten roles to match actual responsibilities:

| Role | What they do | What changes |
|------|-------------|-------------|
| **Account Manager** | Full access + user management + warehouse | No change |
| **Pipeline Manager** | Full access to all modules except user management & warehouse | Keeps dashboard/chart CRUD (aligned with V2 spec) |
| **Analyst** | Dashboards, charts, data explorer only | Loses all pipeline/dbt/orchestration access (46 → 22 perms) |
| **Viewer** | View only dashboards/charts shared with them | Renamed from Guest. Sees only shared resources, not all org resources |

Other fixes:
- Analyst can invite Viewers via share modal (backend-enforced, can't pick role)
- New report-specific permissions (currently reports reuse dashboard perms)

### 2. Frontend Navigation & Route Gating

- Sidebar nav items filtered by role permissions — **hidden**, not grayed out
- Route guards redirect unauthorized users to first accessible page
- "No Access" page with org admin contact info
- Simpler UI for NGO users — if they can't use it, they don't see it

### 3. User Groups & Resource-Level Sharing

Three layers working together:

```
Roles           →  What you can do on the platform
User Groups     →  How people are organized ("Field Team", "Finance", "Partners")
Resource Shares →  Which dashboards/charts a person or group can access
```

**How it works — example**:

Org has dashboards x, y, z. Want "Field Team" to see only x and y:

1. Create group "Field Team" → add 5 users
2. Share dashboard x with "Field Team" (view access)
3. Share dashboard y with "Field Team" (edit access)
4. Members see x and y, never see z
5. New member added to group → automatically gets access

**Core concepts**:

- **ResourceShare** — a record saying "this dashboard/chart is shared with this user/group at view/edit level"
- **Chart inheritance** — sharing a dashboard automatically shares all charts inside it. Revoking cascades too.
- **Paste-to-share** — paste multiple emails in share modal:
  - Matched email (existing user) → immediate access
  - Unmatched email (not on Dalgo) → option to invite as Viewer, access granted after they accept
- **Viewer scoping** — Viewers see ONLY resources explicitly shared with them. AM/PM/Analyst continue to see all org resources (no change to their experience).

**Who can do what**:

| Action | Account Manager | Pipeline Manager | Analyst | Viewer |
|--------|:-:|:-:|:-:|:-:|
| See all org dashboards | Yes | Yes | Yes | No — shared only |
| Create/edit dashboards | Yes | Yes | Yes | No |
| Share own dashboards | Yes | Yes | Yes | No |
| Share any dashboard | Yes | No | No | No |
| Create user groups | Yes | Yes | Yes | No |
| Invite new users | Yes (any role) | No | Yes (Viewer only) | No |

**Key design decisions**:

- **Groups are flat** — no nested groups, no groups-within-groups. Simple for NGO staff.
- **View/Edit only** — no "manage" permission level. Only the resource owner can manage sharing.
- **Chart inheritance** — share a dashboard, all its charts come along. No need to share each chart individually.
- **Hide, don't gray out** — unauthorized nav items are completely hidden, not shown as locked.

### 4. Row-Level Security (Future)

- Control which data rows users can see within charts
- Schema/table access rules per user or role
- RLS policies inject WHERE clauses into chart queries
- Managed by Account Managers in Settings

---

## How These Layers Stack

```
┌─────────────────────────────────────────────────┐
│  Layer 4: Row-Level Security (future)           │
│  "Which rows of data can this user see?"        │
├─────────────────────────────────────────────────┤
│  Layer 3: Resource Shares + User Groups         │
│  "Which dashboards/charts can this user see?"   │
├─────────────────────────────────────────────────┤
│  Layer 2: Frontend Nav & Route Gating           │
│  "Which pages/modules can this user access?"    │
├─────────────────────────────────────────────────┤
│  Layer 1: Roles & Permissions                   │
│  "What can this user do on the platform?"       │
└─────────────────────────────────────────────────┘
```

Each layer builds on the one below. Layer 1 must ship first.

---

## Implementation Order

```
Layer 1 (Roles)     →  Ship first, independent, low risk
Layer 2 (Nav/Routes)→  Frontend only, parallel with Layer 3 backend
Layer 3 (Sharing)   →  Critical for Viewer role. Until this ships, Viewers see all org dashboards.
Layer 4 (RLS)       →  Future, depends on Layer 1 being finalized
```

---

## Detailed Plans

| Document | What it covers |
|----------|---------------|
| `unified_access_control_rls_plan.md` | Current state analysis, all 73 permissions mapped, role fixes, proposed permission matrices, RLS design |
| `groups_and_sharing_plan.md` | Data models, access logic, chart inheritance rules, paste-to-share flow, API endpoints, frontend wireframes, edge cases |
| `access_control_ux_design.md` | UX specs for all 7 features: invite flow, role management, nav gating, no-access page, share modal, role guide, viewer experience |
