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

---

## What Each Role Experiences

### Account Manager — Sarah (M&E Lead)

**Persona**: Manages the org's data team, onboards new staff, oversees all programs.

**Sidebar**: Everything visible — Ingest, Transform, Orchestrate, Dashboards, Charts, Reports, Settings (Users, Billing, About)

**Dashboards**: Sees ALL 15 org dashboards. Can create, edit, delete, share any of them.

**Sharing**: Opens any dashboard → Share → pastes emails or selects groups → assigns view/edit. Can share dashboards she didn't create. Sees an unmatched email → invites them as any role (Analyst, Pipeline Manager, Viewer).

**Groups**: Creates "Leadership Team", "Field Staff", "External Partners". Adds/removes members from any group. Sees all groups in the org.

**User Management**: Full access — invites users, changes roles, removes users.

**Example day**: Sarah creates a new "Quarterly Impact" dashboard → shares it with "Leadership Team" (view) and "M&E Staff" group (edit) → invites an external consultant as Analyst → checks pipeline status in Orchestrate.

---

### Pipeline Manager — Raj (Data Engineer / Implementation Partner)

**Persona**: Sets up data sources, builds transformations, manages pipelines. Also reviews dashboards.

**Sidebar**: Everything visible — Ingest, Transform, Orchestrate, Dashboards, Charts, Reports, Settings (About only, no Users/Billing)

**Dashboards**: Sees ALL 15 org dashboards. Can create, edit, delete, share his own dashboards.

**Sharing**: Opens his own dashboard → Share → pastes emails or selects groups → assigns view/edit. Cannot share dashboards created by others (only AM can).

**Groups**: Creates groups for his team. Manages members in his own groups.

**What he can't do**: Manage users, change roles, access billing, manage warehouse config.

**Example day**: Raj connects a new data source in Ingest → builds dbt models in Transform → sets up a pipeline schedule in Orchestrate → creates a dashboard showing pipeline health → shares it with "Data Team" group (edit).

---

### Analyst — Priya (M&E Officer)

**Persona**: Creates dashboards and charts to track program outcomes. Works with data explorer. Doesn't touch pipelines.

**Sidebar**: Dashboards, Charts, Reports, Data Explorer, Settings (About only). Does NOT see Ingest, Transform, Orchestrate, Users, Billing.

**Dashboards**: Sees ALL org dashboards. Can create, edit, delete, share her own.

**Sharing**: Opens her dashboard → Share → pastes emails or selects groups → assigns view/edit. Sees an unmatched email → can invite them as **Viewer only** (cannot choose other roles). Cannot share dashboards created by others.

**Groups**: Creates groups like "Program Leads". Manages members in her own groups.

**What she can't do**: Access pipelines, dbt, orchestration. Manage users or billing. Invite non-Viewer roles.

**Example day**: Priya opens Data Explorer to query beneficiary data → creates a "Field Performance" chart → adds it to her "Program Outcomes" dashboard → shares the dashboard with "Field Staff" group (view) → pastes an external partner's email → invites them as Viewer.

---

### Viewer — James (Program Staff / Funder)

**Persona**: Needs to check specific dashboards regularly. Doesn't create anything. May be external.

**Sidebar**: Dashboards, Charts, Reports, Settings (About only). Nothing else.

**Dashboards**: Sees ONLY dashboards explicitly shared with him — directly or via a group he belongs to. If nothing is shared, he sees an empty list.

**Charts**: Sees charts shared directly with him, plus charts inherited from shared dashboards. Inherited charts show "via Program Outcomes dashboard".

**What he can do**:
- View shared dashboards and interact with filters (date range, region, etc.)
- Download PDFs of dashboards/reports
- See charts within shared dashboards

**What he can't do**: Create, edit, or delete anything. Share with others. Access pipelines, transforms, data explorer. See dashboards not shared with him.

**What he sees when he logs in**:
```
Shared with you

Dashboards (2)
┌──────────────────┬──────────────────┐
│ Program Outcomes │ Field Activity   │
│ 12 charts        │ 8 charts         │
│ View access      │ View access      │
└──────────────────┴──────────────────┘

Charts (20)
3 directly shared + 17 via dashboards
```

**Example day**: James logs in → sees 2 dashboards shared with him → opens "Program Outcomes" → changes date filter to Q1 → downloads PDF for his board meeting.

---

### Side-by-Side: What the sidebar looks like per role

```
Account Manager       Pipeline Manager      Analyst              Viewer
─────────────────    ─────────────────    ─────────────────    ─────────────────
Ingest               Ingest               Dashboards           Dashboards
Transform            Transform            Charts               Charts
Orchestrate          Orchestrate          Reports              Reports
Dashboards           Dashboards           Data Explorer        Settings
Charts               Charts               Settings               └─ About
Reports              Reports                └─ About
Data Explorer        Data Explorer
Settings             Settings
  ├─ Users             └─ About
  ├─ Billing
  └─ About
```

---

### Side-by-Side: Same dashboard, different roles

Dashboard "Program Outcomes" — owned by Priya (Analyst):

| What they see | Account Manager (Sarah) | Pipeline Manager (Raj) | Analyst (Priya) | Viewer (James) |
|--------------|:-:|:-:|:-:|:-:|
| Dashboard visible? | Yes (all org) | Yes (all org) | Yes (all org) | Only if shared |
| Edit button | Yes | Yes | Yes (owner) | No |
| Delete button | Yes | Yes | Yes (owner) | No |
| Share button | Yes | Yes | Yes (owner) | No |
| Add chart button | Yes | Yes | Yes (owner) | No |
| Chart filters | Yes | Yes | Yes | Yes |
| Download PDF | Yes | Yes | Yes | Yes |
| View-only banner | No | No | No | Yes — "You have view access" |

---

### Sharing scenario across roles

Priya (Analyst) shares "Program Outcomes" dashboard with "Field Staff" group (view) and James (Viewer) individually (view):

| Step | What happens |
|------|-------------|
| Priya opens Share modal | Sees paste field + group search |
| Types "Field Staff" | Group appears in search, selects it, sets "View" |
| Pastes james@ngo.org | Matched — James is an existing Viewer user |
| Pastes unknown@partner.org | Unmatched — shows warning: "Not on Dalgo. [Invite as Viewer] [Remove]" |
| Clicks "Invite as Viewer" | Invitation sent. Pending share created. |
| Clicks Share | Dashboard shared with Field Staff group + James |
| James logs in | Sees "Program Outcomes" in his dashboard list. All 12 charts inside are also visible. |
| unknown@partner.org accepts invite | Creates Viewer account. Pending share activates. Sees the dashboard. |
| Field Staff member Alice logs in | Sees "Program Outcomes" because she's in the group. Didn't need to be shared individually. |
| Priya adds a new chart to the dashboard | Alice, James, and the new partner automatically see it (chart inheritance). |
| Priya revokes Field Staff group access | Alice no longer sees the dashboard or its charts. James still sees it (individual share). |

---

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
