---

# Dalgo Access Control — Complete UX Design Specifications

I've reviewed the planning document and existing frontend patterns. Here are the complete UX designs for all seven features, designed specifically for NGO users with no technical background.

---

## 1. Role Assignment During User Invite

### Layout Description

**Location**: Settings > User Management > Invite User flow

**Modal Structure**:
- Width: `sm:max-w-lg` (wider than standard to accommodate role cards)
- Two-step wizard UI: Step 1 (Email + Name) → Step 2 (Role Selection)
- Visual progress indicator at top: "1. User Details" → "2. Choose Role"

**Step 2 - Role Selection Screen**:
```
┌─────────────────────────────────────────────┐
│  Invite New User                         ×  │
│                                             │
│  ● User Details    ● Choose Role            │ ← Progress dots
│  ────────────────────────                   │
│                                             │
│  What can [name] do in Dalgo?               │ ← Contextual heading
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │ ○ Org Admin                           │ │ ← Radio card
│  │   Manage users and access all data    │ │
│  │   Best for: Team leads, M&E managers  │ │
│  └───────────────────────────────────────┘ │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │ ○ Pipeline Manager                    │ │
│  │   Build data pipelines and reports    │ │
│  │   Best for: Data managers, IT staff   │ │
│  └───────────────────────────────────────┘ │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │ ● Analyst                             │ │ ← Selected state
│  │   Create dashboards, charts, reports  │ │
│  │   Best for: M&E officers, analysts    │ │
│  └───────────────────────────────────────┘ │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │ ○ Viewer                              │ │
│  │   View dashboards and reports only    │ │
│  │   Best for: Program staff, partners   │ │
│  └───────────────────────────────────────┘ │
│                                             │
│  Not sure which role to pick?  [See Guide] │ ← Help link
│                                             │
│          [Back]  [Send Invitation]          │
└─────────────────────────────────────────────┘
```

### Component Structure
- **Radio Group** (`components/ui/radio-group.tsx`) wrapping role cards
- **Card** components for each role with hover state
- **Dialog** (`components/ui/dialog.tsx`) for the modal
- **Button** with `variant="ghost"` + `style={{ backgroundColor: 'var(--primary)' }}` for primary CTA

### Copy/Microcopy

**Step 1 heading**: "Invite New User"
**Step 2 heading**: "What can [FirstName] do in Dalgo?"

**Role Descriptions** (exact copy):

| Role | Title | Description | Best For |
|------|-------|-------------|----------|
| Org Admin | Org Admin | Manage users and access all data | Team leads, M&E managers |
| Pipeline Manager | Pipeline Manager | Build data pipelines and reports | Data managers, IT staff |
| Analyst | Analyst | Create dashboards, charts, reports | M&E officers, analysts |
| Viewer | Viewer | View dashboards and reports only | Program staff, partners |

**Help Link Text**: "Not sure which role to pick? [See Guide]"

**Button Labels**:
- Step 1: "Next" (primary), "Cancel" (secondary)
- Step 2: "Back" (secondary), "Send Invitation" (primary, disabled until role selected)

**Empty Selection State**: No role selected by default. User must choose.

**Validation Messages**:
- No role selected on submit: "Please select a role for [name]"
- Email already exists: "[email] is already invited or a member"

### Interaction Design

**Step 1 → Step 2 Transition**:
1. User enters email + name, clicks "Next"
2. Fade transition to Step 2 (role selection)
3. Progress indicator updates: dot 1 filled, dot 2 active

**Role Card Interactions**:
- **Default**: White background, gray border (`border`), radio unfilled
- **Hover**: Light blue background (`bg-blue-50`), border becomes `border-blue-200`
- **Selected**: Blue border (`border-primary`), radio filled, subtle shadow (`shadow-sm`)
- **Click anywhere on card**: Selects that role (entire card is clickable)

**"See Guide" Link**:
- Opens accordion below the cards (smooth slide-down)
- Shows comparison table (see Section 6 for details)
- Changes to "Hide Guide" when open

**Send Invitation Flow**:
1. Click "Send Invitation"
2. Button shows loading spinner ("Sending...")
3. On success: Toast "Invitation sent to [email]", modal closes, user list refreshes
4. On error: Inline error message below buttons, modal stays open

### Edge Cases

**User clicks "Back" on Step 2**:
- Returns to Step 1, preserves entered email/name
- Selected role is preserved (if they go forward again)

**User closes modal mid-flow**:
- Confirmation dialog: "Discard invitation draft?"
- Options: "Keep Editing" (stay), "Discard" (close)

**Email domain not recognized**:
- No blocking, but show info tooltip: "Make sure [email] can receive emails"

**User selects Org Admin**:
- Show warning card below role cards:
  ```
  ⚠️ Org Admins can invite/remove users and access all data.
  Only assign this role to trusted team members.
  ```
- Background: `bg-orange-50`, border: `border-orange-200`

**Role picker loads with error**:
- Show simplified fallback: Dropdown with role names only
- Log error to console for debugging

### Accessibility

**Keyboard Navigation**:
- Tab through role cards (each card focusable)
- Arrow keys move between roles (standard radio group behavior)
- Enter/Space selects focused role
- Escape closes modal (with confirmation if data entered)

**Screen Reader Announcements**:
- Radio group label: "Select role for [FirstName LastName]"
- Each role: "[Role Name]. [Description]. Best for: [personas]"
- Selected state: "Selected: [Role Name]"
- Step transitions: "Step 2 of 2: Choose Role"

**Focus Management**:
- On Step 1 → Step 2: Focus moves to first role card
- On modal open: Focus on email input
- On modal close: Focus returns to "Invite User" button

---

## 2. Role Management Page (Settings)

### Layout Description

**Location**: Settings > User Management (existing page)

**Page Structure** (follows standard Dalgo list page pattern):
```
┌─────────────────────────────────────────────────────────┐
│ [Header: Fixed, border-bottom]                          │
│                                                          │
│  User Management                     [+ Invite User]    │ ← Page title + CTA
│  Manage who has access to your data                     │ ← Subtitle
│                                                          │
│  [Tabs: Members | Pending Invites | Role Guide]         │ ← Tab navigation
│  ─────────────────────────────────────────              │
├─────────────────────────────────────────────────────────┤
│ [Scrollable Content Area]                               │
│                                                          │
│  Members (12)                                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Name ↑    Email           Role        Actions    │  │ ← Table header
│  ├──────────────────────────────────────────────────┤  │
│  │ Sarah Lee  sarah@ngo.org  Org Admin    [⋮]      │  │
│  │ John Doe   john@ngo.org   Analyst      [⋮]      │  │
│  │ Jane Smith jane@ngo.org   Viewer       [⋮]      │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Component Structure

**Table Implementation**:
- Standard `components/ui/table.tsx`
- Columns: Avatar + Name, Email, Role (with badge), Actions (dropdown)
- Sortable by Name (default), Email, Role
- Sticky header on scroll

**Role Badge Design**:
- **Org Admin**: `bg-purple-100 text-purple-700 border-purple-300`
- **Pipeline Manager**: `bg-blue-100 text-blue-700 border-blue-300`
- **Analyst**: `bg-green-100 text-green-700 border-green-300`
- **Viewer**: `bg-gray-100 text-gray-700 border-gray-300`

**Actions Dropdown** (three-dot menu):
- Icon: `MoreVertical` from lucide-react
- Options: "Change Role", "Resend Invite" (if pending), "Remove User"

### Copy/Microcopy

**Page Title**: "User Management"
**Subtitle**: "Manage who has access to your data"

**Tab Labels**:
- "Members ([count])" — Active users
- "Pending Invites ([count])" — Not yet accepted
- "Role Guide" — Comparison table

**Table Column Headers**:
- "Name" (with sort arrow)
- "Email"
- "Role"
- "Actions"

**Empty States**:
- No members: "No users yet. Click 'Invite User' to get started."
- No pending invites: "All invitations have been accepted."
- Search returns nothing: "No users found matching '[search query]'"

**Action Menu Options**:
- "Change Role"
- "Resend Invite" (pending only)
- "Remove User" (destructive, red text)

### Interaction Design

**Change Role Flow**:
1. Click "Change Role" from dropdown
2. **Inline Dropdown** appears in the table row (replaces role badge)
   - Select component with all available roles
   - Options: Same as invite flow (Org Admin, Pipeline Manager, Analyst, Viewer)
   - Auto-expands, focused
3. User selects new role
4. **Confirmation Popover** appears below the dropdown:
   ```
   ┌────────────────────────────────────────┐
   │ Change [Name]'s role to [New Role]?   │
   │                                        │
   │ They will:                             │
   │ • [Gain/Lose] access to [features]     │
   │ • [Keep/Lose] access to [data]         │
   │                                        │
   │      [Cancel]  [Change Role]           │
   └────────────────────────────────────────┘
   ```
5. On "Change Role": API call, toast notification, table updates
6. On "Cancel" or click outside: Dropdown collapses, reverts to previous role

**Remove User Flow**:
1. Click "Remove User" from dropdown
2. **Alert Dialog** (`components/ui/alert-dialog.tsx`):
   ```
   Remove [Name] from your organization?
   
   [Name] will immediately lose access to all data and features.
   This action cannot be undone.
   
        [Cancel]  [Remove User]
   ```
   - "Remove User" button is `variant="destructive"`
3. On confirm: API call, toast "User removed", row fades out
4. If removing last Org Admin: Block with error message
   ```
   Cannot remove the last Org Admin
   
   At least one Org Admin is required.
   Assign another user as Org Admin first.
   ```

**Pending Invites Tab**:
- Same table structure
- Role badge shows role they were invited as
- "Resend Invite" option sends new email, updates "Sent" timestamp
- "Invited on [date]" shown in place of last login

### Edge Cases

**User tries to remove themselves**:
- Show warning: "You cannot remove yourself. Ask another Org Admin to remove you."

**User tries to change their own role to Viewer**:
- Allow, but show confirmation: "You will lose access to this settings page. Continue?"

**Last Org Admin tries to change to non-admin role**:
- Block with error: "Cannot change role. At least one Org Admin is required."

**User has pending changes (role change in progress)**:
- Disable all other actions until current change completes or is cancelled

**API fails during role change**:
- Dropdown reverts to previous role
- Toast error: "Failed to update role. Please try again."
- Console logs full error for debugging

**Email address exceeds column width**:
- Truncate with ellipsis, show full email on hover tooltip

### Accessibility

**Keyboard Navigation**:
- Tab through table rows
- Enter on row: Opens action dropdown
- Arrow keys: Navigate dropdown options
- Escape: Closes dropdown, cancels role change

**Screen Reader**:
- Table announced as "User management table, 12 users"
- Each row: "[Name], [Email], Role: [Role], Actions"
- Dropdown: "Actions for [Name]"
- Role change: "Change role for [Name], current role [Old Role], new role [New Role]"

**Focus States**:
- All interactive elements have visible focus rings (`focus-visible:ring-2 ring-primary`)
- Role dropdown auto-focuses when opened
- After role change: Focus returns to action menu button

**Color Contrast**:
- All role badges pass WCAG AA (verified in design tokens)
- Red "Remove" text: `#dc2626` on white = 4.5:1 ratio

---

## 3. Permission-Gated Navigation

### Layout Description

The sidebar dynamically shows/hides nav items based on the user's role permissions. This is transparent to the user — they simply see a sidebar that matches their access level.

**Sidebar Structure** (already exists in `components/main-layout.tsx`):
```
Collapsed Mode              Expanded Mode
┌─────┐                    ┌─────────────────┐
│ [I] │ Impact             │ [I] Impact      │
│ [M] │ Metrics            │ [M] Metrics     │
│ [C] │ Charts             │ [C] Charts      │
│ [D] │ Dashboards         │ [D] Dashboards ▾│
│     │                    │   └─Usage       │
│ [R] │ Reports            │ [R] Reports     │
│ [D] │ Data               │ [D] Data       ▾│
│     │                    │   ├─Overview    │
│     │                    │   ├─Ingest      │
│     │                    │   ├─Transform   │
│     │                    │   ├─Orchestrate │
│     │                    │   └─Explore     │
│ [S] │ Settings           │ [S] Settings   ▾│
│     │                    │   ├─Billing     │
│     │                    │   ├─Users       │
│     │                    │   └─About       │
└─────┘                    └─────────────────┘
```

### Sidebar by Role (What Each Role Sees)

**Org Admin** (sees everything):
- Impact, Metrics, Charts, Dashboards (+Usage), Reports
- Data (Overview, Ingest, Transform, Orchestrate, Explore, Quality)
- Alerts, Settings (Billing, User Management, About)

**Pipeline Manager** (build focus):
- Impact, Charts, Dashboards (+Usage), Reports
- Data (Overview, Ingest, Transform, Orchestrate, Explore, Quality)
- Alerts, Settings (About only)

**Analyst** (consume focus):
- Impact, Metrics, Charts, Dashboards (+Usage), Reports
- Settings (About only)

**Viewer** (read-only):
- Impact, Charts, Dashboards, Reports
- Settings (About only)

### Permission Mapping

Add to `components/main-layout.tsx` `NavItemType`:
```typescript
interface NavItemType {
  title: string;
  href: string;
  icon: React.ComponentType<{ className?: string }>;
  isActive: boolean;
  children?: NavItemType[];
  hide?: boolean;
  requiredPermission?: string; // NEW
}
```

**Permission Requirements**:
| Nav Item | Required Permission |
|----------|---------------------|
| Impact | (always visible) |
| Metrics | (always visible) |
| Charts | `can_view_charts` |
| Dashboards | `can_view_dashboards` |
| Reports | `can_view_reports` |
| Data > Overview | `can_view_pipeline_overview` |
| Data > Ingest | `can_view_sources` |
| Data > Transform | `can_view_dbt_workspace` |
| Data > Orchestrate | `can_view_pipelines` |
| Data > Explore | `can_view_warehouse_data` |
| Data > Quality | Feature flag + `can_view_dbt_workspace` |
| Alerts | (always visible for now) |
| Settings > Billing | `can_view_orgusers` |
| Settings > User Management | `can_view_orgusers` |
| Settings > About | (always visible) |

### Implementation Pattern

**In `getNavItems()` function**:
```typescript
const getNavItems = (
  currentPath: string,
  hasSupersetSetup: boolean,
  isFeatureFlagEnabled: (flag: FeatureFlagKeys) => boolean,
  transformType?: string,
  hasPermission?: (permission: string) => boolean // NEW
): NavItemType[] => {
  const allNavItems: NavItemType[] = [
    {
      title: 'Charts',
      href: '/charts',
      icon: ChartBarBig,
      isActive: currentPath.startsWith('/charts'),
      requiredPermission: 'can_view_charts',
    },
    // ... other items
  ];

  // Filter items based on permissions
  return allNavItems.filter(item => {
    if (item.requiredPermission && !hasPermission?.(item.requiredPermission)) {
      return false;
    }
    if (item.children) {
      item.children = item.children.filter(child => {
        return !child.requiredPermission || hasPermission?.(child.requiredPermission);
      });
      // Hide parent if all children are hidden
      if (item.children.length === 0) return false;
    }
    return !item.hide;
  });
};
```

**Hook into auth store**:
```typescript
const { hasPermission } = useUserPermissions();
const navItems = getNavItems(pathname, hasSupersetSetup, isFeatureFlagEnabled, transformType, hasPermission);
```

### Interaction Design

**Behavior**:
- Items the user lacks permission for are **completely hidden** (not grayed out or locked)
- No indication that hidden items exist (clean, focused UI)
- Sidebar automatically collapses "Data" section if all children are hidden
- Settings section always shows "About", hides other items based on permissions

**Why Hide Instead of Show-as-Locked**:
- Simpler mental model for non-technical users
- Avoids questions like "Why can't I access Ingest?"
- Reduces visual clutter
- Follows principle: "If they can't use it, they shouldn't see it"

### Edge Cases

**User's role changes while logged in**:
- After role change, backend invalidates JWT
- On next API call: Token refresh, new permissions loaded
- Sidebar re-renders with new nav items
- If currently on a page they no longer have access to: Redirect to first accessible page (see Section 4)

**All "Data" children hidden**:
- Hide the entire "Data" section (don't show empty parent)

**User has custom role with unusual permissions**:
- Sidebar still works (driven by permission checks, not hardcoded roles)
- May see unexpected combinations (e.g., can see Dashboards but not Charts)

**JavaScript error in permission check**:
- Fail open: Show nav items (better than locking users out)
- Log error to console for debugging
- Show toast: "There was a problem loading your menu. Please refresh."

### Accessibility

**Screen Reader**:
- No change needed — hidden items are not in DOM
- Nav list announced as usual: "Navigation menu, [count] items"

**Keyboard Navigation**:
- Tab order unaffected (hidden items not focusable)

---

## 4. "No Access" Experience

### When This Appears

**Triggers**:
1. User navigates directly to a URL they don't have permission for (e.g., Viewer types `/pipeline`)
2. User's role changes while they're on a restricted page
3. User clicks a deep link/bookmark to a page they no longer have access to
4. Permission check fails on initial page load

### Layout Description

**Full-Page Error State** (not a modal):
```
┌─────────────────────────────────────────────┐
│ [Normal app header/navbar]                  │
├─────────────────────────────────────────────┤
│                                             │
│                                             │
│              [Lock Icon]                    │ ← 80x80px, gray-400
│                                             │
│         You don't have access               │ ← text-2xl font-bold
│                                             │
│   You need additional permissions to        │
│   view this page.                           │ ← text-muted-foreground
│                                             │
│   Contact your Organization Admin to        │
│   request access.                           │
│                                             │
│   Organization Admin: Sarah Lee             │ ← If available
│   sarah@ngo.org                             │
│                                             │
│       [Go to Dashboards]                    │ ← Primary CTA
│                                             │
└─────────────────────────────────────────────┘
```

### Component Structure

**New Component**: `components/ui/permission-gate.tsx`

```typescript
interface PermissionGateProps {
  requiredPermission: string | string[]; // One or all required
  mode?: 'all' | 'any'; // Default 'all'
  children: React.ReactNode;
  fallback?: React.ReactNode; // Optional custom fallback
}

export function PermissionGate({ 
  requiredPermission, 
  mode = 'all',
  children,
  fallback 
}: PermissionGateProps) {
  const { hasPermission, hasAllPermissions, hasAnyPermission } = useUserPermissions();
  
  const permissions = Array.isArray(requiredPermission) ? requiredPermission : [requiredPermission];
  const hasAccess = mode === 'all' 
    ? hasAllPermissions(permissions)
    : hasAnyPermission(permissions);
  
  if (!hasAccess) {
    return fallback || <NoAccessPage />;
  }
  
  return <>{children}</>;
}
```

### Copy/Microcopy

**Heading**: "You don't have access"

**Body Text**:
```
You need additional permissions to view this page.

Contact your Organization Admin to request access.
```

**With Admin Info**:
```
You need additional permissions to view this page.

Contact your Organization Admin to request access:

Organization Admin: [Name]
[Email]
```

**CTA Button**: "Go to Dashboards" (or first accessible page)

**Page Title** (browser tab): "Access Denied - Dalgo"

### Interaction Design

**Initial Load**:
1. Permission check runs before component renders
2. If denied: Show no access page immediately (no flash of content)
3. If loading: Show skeleton/spinner until permissions resolve

**Auto-Redirect Logic**:
- After 3 seconds: Auto-redirect to first accessible page
- Countdown shown: "Redirecting to Dashboards in 3 seconds..."
- User can click button to redirect immediately
- User can click "Stay here" to cancel countdown (if debugging/support scenario)

**Finding Org Admin Info**:
- Query API: `GET /api/org/admins` (returns list of Org Admin users)
- Show first Org Admin found (preferably current user's direct manager if that data exists)
- If no Org Admins found: Skip the contact info section

**"Go to Dashboards" Button**:
- Navigates to first accessible page in this order:
  1. `/dashboards` (if has `can_view_dashboards`)
  2. `/charts` (if has `can_view_charts`)
  3. `/reports` (if has `can_view_reports`)
  4. `/impact` (fallback, always accessible)

### Edge Cases

**User has no permissions at all**:
- Show same page but button goes to `/impact` (always accessible)
- Contact info: "Contact support@dalgo.org"

**Page requires multiple permissions, user has some but not all**:
- Still show no access page
- Optional future enhancement: "You're missing permission to [action]"

**User got here via public share link that expired**:
- Different page: "This link is no longer active"
- CTA: "Request access" (sends email to dashboard owner)

**API fails to load Org Admin list**:
- Skip contact info section
- Still show generic "Contact your Organization Admin" text

**User navigates to a 404 page**:
- Show standard 404 (different component)
- Heading: "Page not found"
- Body: "The page you're looking for doesn't exist."
- CTA: Same redirect logic as no access page

### Accessibility

**Screen Reader**:
- Page announced as: "Access denied. You don't have permission to view this page."
- Contact info: "Organization admin: [Name], email [Email]"
- Auto-redirect: "Redirecting in 3 seconds. Press escape to cancel."

**Keyboard Navigation**:
- Focus lands on "Go to Dashboards" button on load
- Escape key: Cancels auto-redirect (if enabled)
- Tab: Cycles through button, admin email (if mailto link)

**Focus Management**:
- On page render: Focus moves to primary button
- After redirect: Standard page focus behavior

---

## 5. Dashboard/Report Sharing UI

### Layout Description

The share modal is already implemented (`components/ui/share-modal.tsx`). We need to enhance it for Phase 3 (user-to-user sharing) and ensure it works well for the new roles.

**Current Modal** (already exists):
- Public sharing toggle
- Email sharing for reports
- Copy link button

**Enhanced Modal** (Phase 3):
```
┌─────────────────────────────────────────────┐
│  Share Dashboard                         ×  │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 🛡️ Organization Access      [Default]│   │ ← Card 1
│  │ Users in your org with proper        │   │
│  │ permissions can access this          │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 🔗 Public Access            [Toggle] │   │ ← Card 2
│  │ Anyone with the link can view        │   │
│  │                                      │   │
│  │ ⚠️ Your data is now exposed...       │   │ ← If enabled
│  │                                      │   │
│  │ Share this dashboard:                │   │
│  │ [Copy Public Link]                   │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 👥 Share with Team Members     [New] │   │ ← Card 3 (Phase 3)
│  │                                      │   │
│  │ [Search users...]                    │   │ ← Combobox
│  │                                      │   │
│  │ Shared with (3):                     │   │
│  │                                      │   │
│  │ Sarah Lee        [Can edit ▾]  [×]   │   │ ← User row
│  │ John Doe         [Can view ▾]  [×]   │   │
│  │ Jane Smith       [Can view ▾]  [×]   │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ ✉️ Share via Email                   │   │ ← Card 4 (Reports only)
│  │ [email input + chips]                │   │
│  │ [message textarea]                   │   │
│  │ [Send to X recipients]               │   │
│  └─────────────────────────────────────┘   │
│                                             │
│                          [Close]            │
└─────────────────────────────────────────────┘
```

### Component Breakdown

**Card 1 — Organization Access**:
- Read-only, always present
- Shows `Badge variant="secondary"` with "Default"
- Purpose: Clarify that org-level access is always on

**Card 2 — Public Access**:
- Toggle switch (existing functionality)
- Warning appears when enabled (existing)
- Copy link button when enabled (existing)
- Analytics shown if `public_access_count > 0` (existing)

**Card 3 — Share with Team Members** (Phase 3 only):
- User search combobox (Radix Combobox)
  - Search by name or email
  - Shows avatar + name + email + current role badge
  - Filtered results (max 10 shown)
- Permission dropdown per user:
  - "Can view" (default)
  - "Can edit" (Analysts and above)
  - "Can manage" (can reshare, only Org Admins can grant this)
- Remove button (X icon) per user
- "Shared with" count updates dynamically

**Card 4 — Share via Email** (existing, reports only):
- Only shown if `onShareViaEmail` prop provided
- Email input with chips (existing)
- Personal message textarea (existing)
- Send button (existing)

### Copy/Microcopy

**Modal Titles**:
- Dashboards: "Share Dashboard"
- Reports: "Share Report"
- Charts: "Share Chart" (future)

**Card 1 Heading**: "Organization Access"
**Card 1 Description**: "Users in your organization with proper permissions can access this [dashboard/report/chart]"

**Card 2 Heading**: "Public Access"
**Card 2 Description**: "Anyone with the link can view this [dashboard/report/chart]"

**Card 3 Heading**: "Share with Team Members"
**Card 3 Description**: "Give specific users access to this [dashboard/report/chart]"

**Card 4 Heading**: "Share via Email"
**Card 4 Description**: "Send a PDF and link to recipients. Public access will be enabled automatically."

**Permission Dropdown Options**:
- "Can view" — "View dashboards and charts"
- "Can edit" — "Edit dashboards and charts"
- "Can manage" — "Edit and manage sharing settings"

**Search Placeholder**: "Search by name or email..."

**Empty Search Results**: "No users found"

**Shared With Section**:
- Zero users: (section hidden)
- One user: "Shared with 1 person:"
- Multiple: "Shared with [N] people:"

### Interaction Design

**Adding a User**:
1. User types in search box
2. Dropdown shows filtered users (debounced 300ms)
3. User clicks on a result
4. API call: `POST /sharing/dashboard/{id}/` with `{ user_id, permission: 'view' }`
5. User added to "Shared with" list below
6. Search box clears
7. Toast: "[Name] can now view this dashboard"

**Changing Permission Level**:
1. User clicks permission dropdown for a shared user
2. Dropdown expands (Can view, Can edit, Can manage)
3. User selects new permission
4. API call: `PATCH /sharing/dashboard/{id}/{share_id}/` with `{ permission: 'edit' }`
5. Dropdown updates
6. Toast: "[Name] can now edit this dashboard"

**Removing a User**:
1. User clicks X button
2. Confirmation popover: "Remove [Name]'s access?"
3. On confirm: API call `DELETE /sharing/dashboard/{id}/{share_id}/`
4. Row fades out
5. Toast: "[Name] can no longer access this dashboard"

**Search Behavior**:
- Searches across name, email, role
- Filters out:
  - Current user (you)
  - Users already in "Shared with" list
  - Users who have org-wide access (Org Admins, Pipeline Managers if they have `can_view_dashboards`)
- Shows max 10 results
- If more than 10 matches: "Showing 10 of [N] results. Keep typing..."

**Permission Restrictions**:
- Only Org Admins can grant "Can manage"
- Analysts can share with "Can view" or "Can edit" only
- If current user is Analyst and tries to set "Can manage": Disabled with tooltip "Only Org Admins can grant management access"

### Edge Cases

**User shares with someone who doesn't have `can_view_dashboards`**:
- Backend creates share record
- Frontend shows info icon next to their name: "[Name] will be able to view this once their role permits dashboard access"
- Alternative: Filter search results to only show users with base permission (cleaner UX)

**User removes the last person with "Can manage" access**:
- Show warning: "You are removing the last user who can manage sharing. You may lose access if you don't have management permissions."
- Require confirmation

**User tries to add 50+ people**:
- Show limit warning after 20: "You've shared with 20 people. Consider using Public Access for larger audiences."
- No hard limit (for now), but pagination in shared list after 10

**Share modal opened for public dashboard with 1000+ access count**:
- Show analytics in Card 2: "Viewed 1000+ times"
- Suggestion: "This dashboard is popular. Consider creating a report for better performance."

**API fails during share operation**:
- Optimistic UI: Show user in list immediately
- On error: Remove from list, show toast with error
- Retry button in toast: "Failed to share. [Retry]"

**User has slow connection**:
- Show loading spinner in search results: "Searching..."
- Debounce search input (300ms)
- Skeleton rows in "Shared with" list while loading

### Accessibility

**Keyboard Navigation**:
- Tab through: Search input → User results → Permission dropdowns → Remove buttons
- Arrow keys: Navigate search results
- Enter: Select focused result
- Escape: Close dropdowns, close modal

**Screen Reader**:
- Modal announced: "Share [entity type] dialog"
- Search: "Search users to share with. Type to filter."
- Results: "[N] users found. [Name], [Email], [Role]"
- Permission dropdown: "Change permission for [Name]. Current: [Permission]"
- Remove button: "Remove [Name]'s access"

**Focus Management**:
- On modal open: Focus on search input (if Phase 3) or first toggle
- After adding user: Focus returns to search input
- After removing user: Focus moves to next user's permission dropdown (or search if last)

---

## 6. Role Information/Education

### Layout Description

**Location 1: Role Guide Tab** (Settings > User Management > Role Guide)

**Full-Page Comparison Table**:
```
┌─────────────────────────────────────────────────────────────────┐
│ User Management                                                 │
│ Manage who has access to your data                             │
│                                                                 │
│ [Tabs: Members | Pending Invites | Role Guide]                 │
│                                ──────────────                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Understanding Roles in Dalgo                                   │ ← text-2xl
│                                                                 │
│  Choose the right role based on what each person does.         │ ← Subtitle
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           Org Admin  Pipeline Mgr  Analyst  Viewer       │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ WHO IS THIS FOR?                                         │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Team leads,  Data managers, M&E officers Program staff,  │  │
│  │ M&E managers IT staff       Analysts     Partners        │  │
│  │                                                          │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ MANAGE USERS & SETTINGS                                   │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Invite users      ✓         —           —         —      │  │
│  │ Change roles      ✓         —           —         —      │  │
│  │ Manage billing    ✓         —           —         —      │  │
│  │                                                          │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ BUILD DATA INFRASTRUCTURE                                 │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Connect sources   ✓         ✓           —         —      │  │
│  │ Transform data    ✓         ✓           —         —      │  │
│  │ Run pipelines     ✓         ✓           —         —      │  │
│  │                                                          │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ WORK WITH DATA                                            │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ View dashboards   ✓         ✓           ✓         ✓      │  │
│  │ Create dashboards ✓         —           ✓         —      │  │
│  │ Edit dashboards   ✓         —           ✓         —      │  │
│  │ View charts       ✓         ✓           ✓         ✓      │  │
│  │ Create charts     ✓         —           ✓         —      │  │
│  │ View reports      ✓         ✓           ✓         ✓      │  │
│  │ Create reports    ✓         —           ✓         —      │  │
│  │ Share dashboards  ✓         —           ✓         —      │  │
│  │ Download PDFs     ✓         ✓           ✓         ✓      │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Common Questions                                               │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  ▸ What if someone needs to both build pipelines and create    │
│    dashboards?                                                 │
│    Assign them as Org Admin if they also manage users,         │
│    otherwise assign as Pipeline Manager or Analyst depending   │
│    on their primary responsibility.                            │
│                                                                 │
│  ▸ Can a Viewer comment on dashboards?                         │
│    Yes, Viewers can comment and export PDFs. They cannot edit. │
│                                                                 │
│  ▸ Can I change someone's role later?                          │
│    Yes, Org Admins can change anyone's role at any time.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Location 2: Inline Help** (Invite Modal)

When user clicks "See Guide" during invite:
- Accordion expands below role cards
- Shows condensed version of comparison table
- Columns: Role name, "Best for", "Can do" (3-4 bullet points)
- Collapse with "Hide Guide"

**Location 3: User Profile Page** (Your Account)

Small info card:
```
┌─────────────────────────────────────┐
│ Your Role: Analyst                  │
│                                     │
│ You can:                            │
│ • Create dashboards, charts, reports│
│ • View and analyze data             │
│ • Share with your team              │
│                                     │
│ You cannot:                         │
│ • Invite or manage users            │
│ • Build data pipelines              │
│ • Connect new data sources          │
│                                     │
│ Need additional access?             │
│ Contact Sarah Lee (sarah@ngo.org)   │
└─────────────────────────────────────┘
```

### Copy/Microcopy

**Page Heading**: "Understanding Roles in Dalgo"
**Subtitle**: "Choose the right role based on what each person does."

**Table Section Headers**:
- "WHO IS THIS FOR?" (personas)
- "MANAGE USERS & SETTINGS"
- "BUILD DATA INFRASTRUCTURE"
- "WORK WITH DATA"

**Checkmarks**: Use ✓ (green `text-green-600`) for "Yes", — (gray `text-gray-300`) for "No"

**FAQ Heading**: "Common Questions"

**FAQ Questions**:
1. "What if someone needs to both build pipelines and create dashboards?"
   - Answer: "Assign them as Org Admin if they also manage users, otherwise assign as Pipeline Manager or Analyst depending on their primary responsibility."

2. "Can a Viewer comment on dashboards?"
   - Answer: "Yes, Viewers can comment and export PDFs. They cannot edit."

3. "Can I change someone's role later?"
   - Answer: "Yes, Org Admins can change anyone's role at any time from the Members tab."

4. "What happens when I change someone's role?"
   - Answer: "They immediately gain or lose access to features based on their new role. If they're currently using Dalgo, they'll see changes after their next page refresh."

**User Profile Card**:
- Heading: "Your Role: [Role Name]"
- "You can:" (3-5 bullet points)
- "You cannot:" (2-3 bullet points)
- "Need additional access? Contact [Admin Name] ([Email])"

### Interaction Design

**Role Guide Tab**:
- Loads immediately (no API call, static content)
- Table is sticky-header on scroll
- Mobile: Table scrolls horizontally
- Print-friendly (clean print stylesheet)

**Inline Help in Invite Modal**:
- Click "See Guide": Accordion slide-down (300ms)
- Link text changes to "Hide Guide"
- Click again: Accordion collapses
- If user selects a role: Accordion stays open (doesn't auto-close)

**FAQ Accordion** (optional enhancement):
- Questions collapsed by default
- Click to expand (chevron rotates)
- Multiple can be open at once

### Edge Cases

**Mobile View**:
- Table scrolls horizontally with sticky first column (role names)
- "Swipe to see more →" hint shown on first view
- FAQ questions shown full-width

**Print View**:
- Table fits on one page (landscape)
- FAQ expanded by default
- Header/footer with Dalgo branding

**User has custom role not in the table**:
- Profile card shows generic:
  - "Your Role: [Custom Role Name]"
  - "You have custom permissions. Contact your admin for details."

**Content is stale (roles updated but table not)**:
- Add timestamp footer: "Role definitions last updated: [date]"
- Link to "View changelog" (future enhancement)

### Accessibility

**Screen Reader**:
- Table announced: "Role comparison table, 4 columns, [N] rows"
- Each cell: "[Feature name], [Role name], [Yes/No]"
- FAQ: "Frequently asked questions, [N] items"

**Keyboard Navigation**:
- Tab through table cells (left to right, top to bottom)
- Arrow keys navigate table (standard table nav)
- FAQ: Tab to question, Enter to expand, Tab to next

**High Contrast Mode**:
- Checkmarks use ✓ symbol (not just color)
- Section headers have underline (not just bold)

---

## 7. Viewer Experience

### Layout Description

When a Viewer logs in, they see a simplified, focused interface optimized for consumption (not creation).

**Viewer's Home Screen** (`/impact` or `/dashboards`):
```
┌─────────────────────────────────────────────────────────┐
│ [Header with Dalgo logo, Org selector, User menu]      │
├────┬────────────────────────────────────────────────────┤
│ [I]│ Welcome back, [Name]                               │
│ [C]│                                                     │
│ [D]│ Your Dashboards (4)                                │
│ [R]│ ┌────────────────┬────────────────┬──────────────┐│
│ [S]│ │ [Dashboard 1]  │ [Dashboard 2]  │ [Dashboard 3]││
│    │ │ 12 charts      │ 8 charts       │ 5 charts     ││
│    │ └────────────────┴────────────────┴──────────────┘│
│    │                                                     │
│    │ Recent Reports (2)                                 │
│    │ ┌─────────────────────────────────────────────┐   │
│    │ │ Monthly Impact Report - March 2026          │   │
│    │ │ Created by Sarah Lee • 2 days ago           │   │
│    │ └─────────────────────────────────────────────┘   │
│    │                                                     │
└────┴─────────────────────────────────────────────────────┘
```

**Viewer's Sidebar** (collapsed by default on first login):
- Impact (home icon)
- Charts
- Dashboards
- Reports
- Settings > About

**No visible**:
- Metrics, Data section, Alerts, Settings > User Management, Settings > Billing
- Any "Create" or "Edit" buttons

### Dashboard View for Viewer

When a Viewer opens a dashboard:
```
┌─────────────────────────────────────────────────────────┐
│ Sales Dashboard                                         │
│ Last updated: 2 hours ago                               │
│                                                         │
│ [🔖 Bookmark] [💬 Comments (3)] [⬇️ Download PDF]       │ ← Actions
│                                                         │
├─────────────────────────────────────────────────────────┤
│ [Charts render here in grid layout]                     │
│                                                         │
│ [Chart 1]      [Chart 2]      [Chart 3]                │
│                                                         │
│ [Chart 4]      [Chart 5]      [Chart 6]                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**What's Visible**:
- Dashboard title and description
- All charts (read-only)
- Chart filters (can interact)
- Dashboard filters (can interact)
- Bookmark button (save to favorites)
- Comments section (can add/view comments)
- Download PDF button
- Refresh button (reload data)

**What's Hidden**:
- "Edit Dashboard" button
- "Add Chart" button
- Chart edit controls (settings icon, delete)
- Dashboard settings icon
- Share button (only Analysts+ can share)

### Charts List for Viewer

```
┌─────────────────────────────────────────────────────────┐
│ Charts                                                   │
│ Browse and view charts from your team                   │
│                                                         │
│ [Search...] [Filter by Type ▾] [Sort: Recent ▾]        │ ← Filters only
│                                                         │
├─────────────────────────────────────────────────────────┤
│ ┌───────────────┬───────────────┬───────────────┐      │
│ │ [Chart Card]  │ [Chart Card]  │ [Chart Card]  │      │
│ │ Bar Chart     │ Line Chart    │ Pie Chart     │      │
│ │ By Sarah Lee  │ By John Doe   │ By Sarah Lee  │      │
│ │ [View]        │ [View]        │ [View]        │      │
│ └───────────────┴───────────────┴───────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**What's Visible**:
- Search, filter, sort controls
- Chart previews (thumbnail or icon)
- Creator name
- "View" button

**What's Hidden**:
- "Create Chart" button (top-right CTA)
- "Edit" or "Delete" options in card menus

### Reports List for Viewer

Same pattern as Charts:
- Can view all reports shared with them
- Can download PDFs
- Cannot create or edit reports

### Commenting Feature

**Comment Thread** (shown on dashboards/charts):
```
┌─────────────────────────────────────────────┐
│ Comments (3)                                │
│                                             │
│ Sarah Lee • 2 hours ago                     │
│ "The March numbers look great! 🎉"          │
│ [Reply]                                     │
│                                             │
│   └─ John Doe • 1 hour ago                  │
│      "Agreed! Let's discuss in tomorrow's   │
│       meeting."                             │
│      [Reply]                                │
│                                             │
│ Jane Smith • 5 hours ago                    │
│ "Can you add a filter for region?"          │
│ [Reply]                                     │
│                                             │
│ ┌─────────────────────────────────────────┐ │
│ │ Add a comment...                        │ │ ← Textarea
│ └─────────────────────────────────────────┘ │
│                    [Post Comment]           │
└─────────────────────────────────────────────┘
```

**Viewer Can**:
- View all comments
- Add new comments
- Reply to comments
- Edit/delete their own comments

**Viewer Cannot**:
- Resolve comments (only creators/editors)
- Delete others' comments

### Copy/Microcopy

**Home Screen Heading**: "Welcome back, [FirstName]"

**Dashboard List Empty State**:
- "No dashboards yet"
- "Ask your team to share dashboards with you, or check Reports."

**Charts List Empty State**:
- "No charts available"
- "Charts will appear here once your team creates them."

**Reports List Empty State**:
- "No reports yet"
- "Reports will appear here once created."

**PDF Download Button**: "Download PDF" (not "Export")

**Comment Input Placeholder**: "Add a comment..."

**Comment Button**: "Post Comment"

### Interaction Design

**First-Time Viewer Experience**:
1. User logs in for first time as Viewer
2. Redirect to `/dashboards` (or `/impact` if no dashboards)
3. Show onboarding tooltip on sidebar: "You're a Viewer. You can browse dashboards, charts, and reports, and leave comments. Contact your admin to request edit access."
4. Tooltip auto-dismisses after 10 seconds or user clicks "Got it"

**Dashboard Interaction**:
- Chart filters work normally (Viewer can change date ranges, categories, etc.)
- Filter changes update charts immediately
- Filters reset on page refresh (not saved to dashboard)
- If Viewer tries to click a "hidden" edit button (e.g., via browser dev tools): Show toast "You don't have permission to edit"

**Bookmarking**:
- Bookmark button toggles favorite status
- Favorited dashboards appear in "Your Dashboards" section on home screen
- Stored per-user in database

**PDF Download**:
- Click "Download PDF" → Shows loading spinner
- After 2-3 seconds: Browser downloads `[Dashboard Name] - [Date].pdf`
- Uses Playwright backend (per download_report_feature.md plan)

**Comments**:
- Viewer types comment → Click "Post Comment"
- Comment appears immediately (optimistic UI)
- If API fails: Comment disappears, toast error, retry button
- Real-time updates via polling (every 30 seconds when comments panel open)

### Edge Cases

**Viewer lands on a dashboard they can't access** (shared link expired):
- Redirect to No Access page (Section 4)
- Show specific message: "This dashboard is no longer shared with you. Contact [Owner] to request access."

**Viewer uses browser back button after logout**:
- Standard redirect to login (handled by auth middleware)
- After login: Return to previous page if still accessible

**Viewer's role upgraded to Analyst mid-session**:
- Next API call refreshes permissions
- Sidebar updates to show new items
- Toast: "Your role has been updated to Analyst. You now have access to more features!"
- Page doesn't auto-reload (avoids disrupting work)

**Chart fails to load for Viewer**:
- Show chart error state: "Failed to load chart. [Retry]"
- No edit button shown (even in error state)
- Retry button reloads that chart only

**Viewer tries to share a dashboard via browser share menu**:
- Browser share shows generic URL
- Recipient sees no access page (unless they have permission)
- Future enhancement: "Share with team" button that opens email/Slack share

**Comment contains @mention**:
- Currently: Plain text, no special handling
- Future enhancement: Highlight mentions, send email notifications

### Accessibility

**Screen Reader**:
- Home screen: "Dashboard list, [N] dashboards"
- Each card: "[Dashboard name], [N] charts, Created by [Creator]"
- Dashboard view: "Dashboard: [Name]. [N] charts. Read-only view."
- Comment section: "Comments, [N] comments. Add a comment."

**Keyboard Navigation**:
- Tab through dashboard cards (each card focusable)
- Enter on card: Opens dashboard
- Tab through charts in dashboard view
- Tab to comment input → Type → Tab to button → Enter to post

**Focus Indicators**:
- All interactive elements have visible focus rings
- Focus order: Search → Filters → Cards → Comments

**Color Contrast**:
- All text passes WCAG AA (minimum 4.5:1 ratio)
- Chart colors verified for colorblind accessibility

---

## Summary of Design Patterns Used

### UI Components (All Shadcn/Radix-based)
- **Dialog** — Modals (invite, share)
- **Alert Dialog** — Destructive confirmations (remove user)
- **Card** — Role cards, share cards, info cards
- **Radio Group** — Role selection
- **Select** — Role dropdown in table
- **Switch** — Public sharing toggle
- **Combobox** — User search
- **Badge** — Role badges, status indicators
- **Button** — CTAs, actions
- **Table** — User management list

### Color Tokens
- **Primary**: `var(--primary)` #00897B (teal)
- **Role Colors**:
  - Org Admin: Purple (`bg-purple-100 text-purple-700`)
  - Pipeline Manager: Blue (`bg-blue-100 text-blue-700`)
  - Analyst: Green (`bg-green-100 text-green-700`)
  - Viewer: Gray (`bg-gray-100 text-gray-700`)
- **Destructive**: Red (`text-destructive`, `variant="destructive"`)
- **Muted**: `text-muted-foreground` for subtitles, descriptions

### Typography
- **Page headings**: `text-3xl font-bold`
- **Section headings**: `text-xl font-semibold`
- **Card titles**: `text-sm font-medium`
- **Body text**: `text-base`
- **Descriptions**: `text-sm text-muted-foreground`
- **Labels**: `text-xs font-medium`

### Spacing
- **Card padding**: `p-4`
- **Section gaps**: `space-y-6`
- **Button groups**: `gap-3`

### Accessibility Features
- All interactive elements have `data-testid`
- All modals/dialogs trap focus
- All forms have associated labels
- All colors meet WCAG AA contrast
- All icon-only buttons have `aria-label`
- All state changes announced to screen readers

---

## Files to Create/Modify

### Phase 1 (Immediate)
- Backend seed data (not frontend)

### Phase 2 (Frontend Nav)
**Modify**:
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/main-layout.tsx` — Add `requiredPermission` filtering

**Create**:
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/ui/permission-gate.tsx` — Route guard component
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/ui/no-access-page.tsx` — No access experience
- `/Users/siddhant/Documents/Dalgo/webapp_v2/app/access-denied/page.tsx` — Route for no access (optional)

### Phase 3 (Resource Sharing)
**Create**:
- `/Users/siddhant/Documents/Dalgo/webapp_v2/hooks/api/useSharing.ts` — SWR hooks for sharing API
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/ui/user-share-section.tsx` — Team sharing card component

**Modify**:
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/ui/share-modal.tsx` — Add Phase 3 user sharing card

### User Management (TBD - depends on existing implementation)
**Find and modify**:
- User invite flow (likely in `/app/settings/user-management/` or `/components/settings/`)
- User table component (likely in `/components/settings/`)

**Create**:
- Role comparison table component
- Role guide tab content
- User profile role info card

---

This completes the comprehensive UX design for all seven features. Each section includes exact layouts, copy, interaction specs, edge cases, and accessibility requirements ready for implementation.
</parameter>
</invoke>

Now I have a complete understanding. Let me create the comprehensive UX design document for you:

---

# Dalgo Access Control — Complete UX Design Specifications

I've reviewed the planning document and your existing frontend patterns. Here's the complete UX design for all seven features, designed specifically for NGO program staff who use Google Workspace and WhatsApp (not enterprise SaaS power users).

## Design Philosophy for This Project

- **Plain language only** — No jargon like "permissions matrix" or "RBAC"
- **Show, don't explain** — Users see what they can do, not what they can't
- **Familiar patterns** — Radio cards (like Google Forms), toggles (like Gmail settings)
- **One decision at a time** — Step-by-step flows, never overwhelming choices
- **Immediate clarity** — Every role description answers "Who is this for?" first

---

## 1. Role Assignment During User Invite

### The Problem
When an Org Admin invites someone, they need to pick a role. But role names like "Pipeline Manager" and "Analyst" don't mean anything to a program director. We need to help them make the right choice without reading documentation.

### The Solution

**Two-step wizard modal** with visual progress:

**Step 1: Basic Info**
```
┌─────────────────────────────────────┐
│  Invite New User                 ×  │
│                                     │
│  ● User Details    ○ Choose Role    │ ← Progress dots
│  ────────────────                   │
│                                     │
│  Email *                            │
│  ┌───────────────────────────────┐ │
│  │ sarah@ngo.org                 │ │
│  └───────────────────────────────┘ │
│                                     │
│  First Name *                       │
│  ┌───────────────────────────────┐ │
│  │ Sarah                         │ │
│  └───────────────────────────────┘ │
│                                     │
│  Last Name *                        │
│  ┌───────────────────────────────┐ │
│  │ Lee                           │ │
│  └───────────────────────────────┘ │
│                                     │
│          [Cancel]  [Next]           │
└─────────────────────────────────────┘
```

**Step 2: Role Selection** (where the magic happens)
```
┌──────────────────────────────────────────┐
│  Invite New User                      ×  │
│                                          │
│  ● User Details    ● Choose Role         │
│  ────────────────────────                │
│                                          │
│  What can Sarah do in Dalgo?            │ ← Personalized heading
│                                          │
│  ┌────────────────────────────────────┐ │
│  │ ○ Org Admin                        │ │ ← Radio card (clickable)
│  │   Manage users and access all data │ │
│  │   👥 Best for: Team leads, M&E mgrs│ │ ← Icon + persona
│  └────────────────────────────────────┘ │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │ ○ Pipeline Manager                 │ │
│  │   Build data pipelines and reports │ │
│  │   🔧 Best for: Data mgrs, IT staff │ │
│  └────────────────────────────────────┘ │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │ ● Analyst                          │ │ ← Selected (blue border)
│  │   Create dashboards, charts, reports│ │
│  │   📊 Best for: M&E officers        │ │
│  └────────────────────────────────────┘ │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │ ○ Viewer                           │ │
│  │   View dashboards and reports only │ │
│  │   👁️ Best for: Program staff       │ │
│  └────────────────────────────────────┘ │
│                                          │
│  Not sure? [Compare all roles]          │ ← Expands help accordion
│                                          │
│       [Back]  [Send Invitation]          │
└──────────────────────────────────────────┘
```

**If user selects "Org Admin"** — show warning:
```
  ┌────────────────────────────────────┐
  │ ⚠️ Org Admins can invite and       │
  │ remove users and access all data.  │
  │ Only assign this to trusted people.│
  └────────────────────────────────────┘
```

### Exact Copy (for implementation)

| Element | Copy |
|---------|------|
| Modal title | "Invite New User" |
| Step 1 heading | User Details |
| Step 2 heading | "What can [FirstName] do in Dalgo?" |
| Role 1 | **Org Admin**<br>Manage users and access all data<br>👥 Best for: Team leads, M&E managers |
| Role 2 | **Pipeline Manager**<br>Build data pipelines and reports<br>🔧 Best for: Data managers, IT staff |
| Role 3 | **Analyst**<br>Create dashboards, charts, reports<br>📊 Best for: M&E officers, analysts |
| Role 4 | **Viewer**<br>View dashboards and reports only<br>👁️ Best for: Program staff, partners |
| Help link | "Not sure? [Compare all roles]" |
| Org Admin warning | ⚠️ Org Admins can invite and remove users and access all data. Only assign this to trusted people. |
| Button labels | Step 1: "Cancel" / "Next"<br>Step 2: "Back" / "Send Invitation" |

### Interactions

**Hover states:**
- Role cards: Light background `bg-blue-50`, border becomes `border-blue-200`
- Selected card: Blue border `border-primary`, filled radio, subtle shadow

**Click behavior:**
- Clicking anywhere on the card selects that role (entire card is a big target)
- "Compare all roles" link expands accordion showing simplified comparison table
- "Send Invitation" is disabled (gray) until a role is selected

**Success flow:**
1. User clicks "Send Invitation"
2. Button shows spinner: "Sending..."
3. Toast notification: "Invitation sent to sarah@ngo.org"
4. Modal closes
5. User appears in "Pending Invites" tab with selected role badge

**Error handling:**
- Email already invited: Inline error below email input: "sarah@ngo.org is already invited or a member"
- No role selected: Inline error below cards: "Please select a role for Sarah"
- API fails: Toast error: "Failed to send invitation. Please try again."

### Accessibility

- **Keyboard**: Tab through cards, Space/Enter selects, Escape closes
- **Screen reader**: "Select role for Sarah Lee. Four options. Org Admin. Manage users and access all data. Best for team leads and M&E managers."
- **Focus**: On modal open → email input. On Step 2 → first role card.

---

## 2. Role Management Page (Settings)

### The Problem
Org Admins need to see who has what role and change roles easily. The current table might just show "Analyst" with no context on what that means or how to change it.

### The Solution

**Tabbed interface** with three sections:

```
┌─────────────────────────────────────────────────────┐
│ [Fixed Header]                                      │
│                                                      │
│  User Management              [+ Invite User]       │ ← CTA button (teal)
│  Manage who has access to your data                 │
│                                                      │
│  [Members (12)] [Pending (2)] [Role Guide]          │ ← Tabs
│  ────────────                                        │
├──────────────────────────────────────────────────────┤
│ [Scrollable Content]                                 │
│                                                      │
│  [Search users...]                     [Filter: All]│
│                                                      │
│  ┌──────────────────────────────────────────────┐  │
│  │ Name ↑       Email          Role    Actions  │  │
│  ├──────────────────────────────────────────────┤  │
│  │ Sarah Lee    sarah@ngo.org  Org Admin  [⋮]  │  │ ← Purple badge
│  │ John Doe     john@ngo.org   Analyst    [⋮]  │  │ ← Green badge
│  │ Jane Smith   jane@ngo.org   Viewer     [⋮]  │  │ ← Gray badge
│  └──────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Role Badges (Visual Specification)

| Role | Colors | Example |
|------|--------|---------|
| Org Admin | `bg-purple-100 text-purple-700 border border-purple-300` | <span style="background:#f3e8ff; color:#7e22ce; border:1px solid #d8b4fe; padding:2px 8px; border-radius:4px; font-size:12px;">Org Admin</span> |
| Pipeline Manager | `bg-blue-100 text-blue-700 border border-blue-300` | <span style="background:#dbeafe; color:#1d4ed8; border:1px solid #93c5fd; padding:2px 8px; border-radius:4px; font-size:12px;">Pipeline Manager</span> |
| Analyst | `bg-green-100 text-green-700 border border-green-300` | <span style="background:#d1fae5; color:#047857; border:1px solid #86efac; padding:2px 8px; border-radius:4px; font-size:12px;">Analyst</span> |
| Viewer | `bg-gray-100 text-gray-700 border border-gray-300` | <span style="background:#f3f4f6; color:#374151; border:1px solid:#d1d5db; padding:2px 8px; border-radius:4px; font-size:12px;">Viewer</span> |

### Change Role Flow

**User clicks three-dot menu → "Change Role":**

```
Before:
│ John Doe  john@ngo.org  [Analyst ▾]  [⋮] │

After click:
│ John Doe  john@ngo.org  [Dropdown▾]   │  │ ← Inline dropdown opens
│                         Org Admin        │
│                         Pipeline Manager │
│                         → Analyst ✓      │  ← Current (checkmark)
│                         Viewer           │
```

**User selects "Org Admin" → Confirmation popover appears:**

```
┌────────────────────────────────────────┐
│ Change John's role to Org Admin?      │
│                                        │
│ John will gain the ability to:        │
│ • Invite and remove users              │
│ • Change other users' roles            │
│ • Access all organization data         │
│                                        │
│      [Cancel]  [Change Role]           │
└────────────────────────────────────────┘
```

### Remove User Flow

**User clicks "Remove User" → Alert dialog:**

```
┌────────────────────────────────────────┐
│  Remove John Doe from your             │
│  organization?                         │
│                                        │
│  John will immediately lose access to  │
│  all data and features. This cannot be │
│  undone.                               │
│                                        │
│       [Cancel]  [Remove User]          │ ← Red destructive button
└────────────────────────────────────────┘
```

### "Role Guide" Tab

When user clicks the "Role Guide" tab, they see a full comparison table:

```
┌─────────────────────────────────────────────────────────┐
│  Understanding Roles in Dalgo                           │
│                                                         │
│  Choose the right role based on what each person does. │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │           Org Admin Pipeline  Analyst  Viewer     │ │
│  │                      Manager                      │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ WHO IS THIS FOR?                                  │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ Team leads, Data mgrs, M&E officers, Program     │ │
│  │ M&E managers IT staff   Analysts     staff       │ │
│  │                                                   │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ MANAGE USERS & SETTINGS                           │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ Invite users     ✓         —          —      —   │ │
│  │ Change roles     ✓         —          —      —   │ │
│  │ Manage billing   ✓         —          —      —   │ │
│  │                                                   │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ BUILD DATA INFRASTRUCTURE                         │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ Connect sources  ✓         ✓          —      —   │ │
│  │ Transform data   ✓         ✓          —      —   │ │
│  │ Run pipelines    ✓         ✓          —      —   │ │
│  │                                                   │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ WORK WITH DATA                                    │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ View dashboards  ✓         ✓          ✓      ✓  │ │
│  │ Create dashboards✓         —          ✓      —   │ │
│  │ View charts      ✓         ✓          ✓      ✓  │ │
│  │ Create charts    ✓         —          ✓      —   │ │
│  │ Share dashboards ✓         —          ✓      —   │ │
│  │ Download PDFs    ✓         ✓          ✓      ✓  │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  Common Questions                                       │
│  ──────────────────────────────────────────────────────│
│                                                         │
│  ▸ What if someone needs to both build pipelines and   │
│    create dashboards?                                  │
│                                                         │
│    Assign them as Org Admin if they also manage users, │
│    otherwise choose based on their main responsibility.│
│                                                         │
│  ▸ Can a Viewer comment on dashboards?                 │
│                                                         │
│    Yes! Viewers can comment and export PDFs. They just │
│    can't edit or create new content.                   │
│                                                         │
│  ▸ Can I change someone's role later?                  │
│                                                         │
│    Yes, you can change any user's role at any time     │
│    from the Members tab.                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Edge Cases

| Situation | Behavior |
|-----------|----------|
| Last Org Admin tries to change their role | Block with error: "Cannot change role. At least one Org Admin is required." |
| User tries to remove themselves | Show warning: "You cannot remove yourself. Ask another Org Admin to remove you." |
| Email too long for table cell | Truncate with ellipsis, show full email on hover tooltip |
| Remove user API fails | Toast error: "Failed to remove user. Please try again." Row stays in table. |

---

## 3. Permission-Gated Navigation

### The Problem
All users currently see the same sidebar. A Viewer shouldn't see "Ingest" or "Orchestrate" — it's confusing and clutters their interface.

### The Solution

**Dynamic sidebar that adapts to the user's role.** Items they can't access are completely hidden (not grayed out or locked — just gone).

### What Each Role Sees

**Org Admin** (everything):
```
│ Impact
│ Metrics
│ Charts
│ Dashboards ▾
│ Reports
│ Data ▾
│   └─ Overview
│   └─ Ingest
│   └─ Transform
│   └─ Orchestrate
│   └─ Explore
│ Alerts
│ Settings ▾
│   └─ Billing
│   └─ User Management
│   └─ About
```

**Pipeline Manager** (build focus):
```
│ Impact
│ Charts
│ Dashboards ▾
│ Reports
│ Data ▾
│   └─ Overview
│   └─ Ingest
│   └─ Transform
│   └─ Orchestrate
│   └─ Explore
│ Alerts
│ Settings ▾
│   └─ About
```

**Analyst** (consume focus):
```
│ Impact
│ Metrics
│ Charts
│ Dashboards ▾
│ Reports
│ Settings ▾
│   └─ About
```

**Viewer** (read-only):
```
│ Impact
│ Charts
│ Dashboards
│ Reports
│ Settings ▾
│   └─ About
```

### Permission Mapping

| Nav Item | Required Permission |
|----------|---------------------|
| Impact | *(always visible)* |
| Metrics | *(always visible)* |
| Charts | `can_view_charts` |
| Dashboards | `can_view_dashboards` |
| Reports | `can_view_reports` |
| Data > Overview | `can_view_pipeline_overview` |
| Data > Ingest | `can_view_sources` |
| Data > Transform | `can_view_dbt_workspace` |
| Data > Orchestrate | `can_view_pipelines` |
| Data > Explore | `can_view_warehouse_data` |
| Alerts | *(always visible for now)* |
| Settings > Billing | `can_view_orgusers` |
| Settings > User Management | `can_view_orgusers` |
| Settings > About | *(always visible)* |

### Implementation Notes

**Modify `components/main-layout.tsx`:**

Add `requiredPermission` field to `NavItemType`:
```typescript
interface NavItemType {
  title: string;
  href: string;
  icon: React.ComponentType<{ className?: string }>;
  isActive: boolean;
  children?: NavItemType[];
  hide?: boolean;
  requiredPermission?: string; // NEW
}
```

Filter nav items in `getNavItems()`:
```typescript
const { hasPermission } = useUserPermissions();

return allNavItems.filter(item => {
  if (item.requiredPermission && !hasPermission(item.requiredPermission)) {
    return false; // Hide if user lacks permission
  }
  if (item.children) {
    // Filter children too
    item.children = item.children.filter(child => 
      !child.requiredPermission || hasPermission(child.requiredPermission)
    );
    // Hide parent if all children hidden
    if (item.children.length === 0) return false;
  }
  return !item.hide;
});
```

### Why Hide Instead of Show-as-Locked?

**Simpler mental model.** A program officer using Dalgo shouldn't wonder "Why can't I access Transform?" — they should just see the tools they can use. Hiding unavailable features:
- Reduces cognitive load
- Avoids questions/confusion
- Makes each role's interface feel purposefully designed for them
- Follows the principle: "If they can't use it, they shouldn't see it"

### Edge Cases

| Situation | Behavior |
|-----------|----------|
| User's role changes while logged in | Next API call refreshes permissions → sidebar re-renders with new items |
| All "Data" children hidden | Hide entire "Data" parent section |
| User on restricted page when role changes | Redirect to first accessible page (see Section 4) |

---

## 4. "No Access" Experience

### The Problem
A Viewer types `/pipeline` into the URL bar. What happens? Currently: probably a crash or confusing error. We need a clear, helpful "you don't have access" page.

### The Solution

**Full-page error state** (not a 404, not a modal):

```
┌─────────────────────────────────────────────┐
│ [Normal Dalgo header with logo/nav]        │
├─────────────────────────────────────────────┤
│                                             │
│                                             │
│              🔒                             │ ← 80x80px lock icon
│                                             │
│         You don't have access               │ ← text-2xl font-bold
│                                             │
│   You need additional permissions to        │ ← text-muted-foreground
│   view this page.                           │
│                                             │
│   Contact your Organization Admin to        │
│   request access:                           │
│                                             │
│   Sarah Lee                                 │ ← If admin info available
│   sarah@ngo.org                             │
│                                             │
│       [Go to Dashboards]                    │ ← Primary button (teal)
│                                             │
│   Redirecting in 3 seconds...               │ ← Auto-redirect countdown
│                                             │
└─────────────────────────────────────────────┘
```

### Copy Variations

**With admin contact info:**
```
You don't have access

You need additional permissions to view this page.

Contact your Organization Admin to request access:

Sarah Lee
sarah@ngo.org
```

**Without admin info:**
```
You don't have access

You need additional permissions to view this page.

Contact your Organization Admin to request access.
```

**Auto-redirect text:**
- "Redirecting in 3 seconds..."
- "Redirecting in 2 seconds..."
- "Redirecting in 1 second..."
- *(then redirects)*

### Smart Redirect Logic

Button goes to the **first accessible page** in this order:
1. `/dashboards` (if has `can_view_dashboards`)
2. `/charts` (if has `can_view_charts`)
3. `/reports` (if has `can_view_reports`)
4. `/impact` (fallback, always accessible)

### When This Appears

**Triggers:**
1. User types a restricted URL directly (e.g., Viewer goes to `/pipeline`)
2. User clicks an old bookmark to a page they no longer have access to
3. User's role downgraded while they're on a restricted page
4. Permission check fails on page load

### How It's Different from a 404

| 404 (Page Not Found) | No Access (Permission Denied) |
|----------------------|-------------------------------|
| URL doesn't exist | URL exists but user can't access it |
| "This page doesn't exist" | "You don't have permission" |
| No contact info | Shows admin contact info |
| Suggests going home | Suggests contacting admin |
| Page title: "Page Not Found" | Page title: "Access Denied" |

### Implementation

**New component**: `components/ui/no-access-page.tsx`

**New route guard**: `components/ui/permission-gate.tsx`

```typescript
interface PermissionGateProps {
  requiredPermission: string | string[];
  mode?: 'all' | 'any'; // Default 'all'
  children: React.ReactNode;
}

export function PermissionGate({ requiredPermission, mode = 'all', children }: Props) {
  const { hasPermission, hasAllPermissions, hasAnyPermission } = useUserPermissions();
  
  const permissions = Array.isArray(requiredPermission) ? requiredPermission : [requiredPermission];
  const hasAccess = mode === 'all' 
    ? hasAllPermissions(permissions)
    : hasAnyPermission(permissions);
  
  if (!hasAccess) {
    return <NoAccessPage />;
  }
  
  return <>{children}</>;
}
```

**Usage in page files:**
```typescript
// app/pipeline/page.tsx
export default function PipelinePage() {
  return (
    <PermissionGate requiredPermission="can_view_pipeline_overview">
      <PipelineList />
    </PermissionGate>
  );
}
```

### Edge Cases

| Situation | Behavior |
|-----------|----------|
| User has no permissions at all (broken state) | Show no access page, button goes to `/impact`, contact: "support@dalgo.org" |
| API fails to load admin list | Skip admin contact info, show generic "Contact your Organization Admin" |
| User navigates to actual 404 page | Show 404 page (different component), not no access page |
| User presses Escape during countdown | Cancel auto-redirect, remove countdown text |

### Accessibility

- **Keyboard**: Focus lands on "Go to Dashboards" button on load. Escape cancels redirect.
- **Screen reader**: "Access denied. You don't have permission to view this page. Contact Organization Admin Sarah Lee at sarah@ngo.org. Redirecting in 3 seconds."
- **Page title**: "Access Denied - Dalgo" (browser tab)

---

## 5. Dashboard/Report Sharing UI

### The Problem
Currently, sharing is all-or-nothing: either everyone in the org sees it, or it's public. We need **user-to-user sharing** (Phase 3) so an Analyst can share a specific dashboard with specific teammates.

### The Solution

**Enhanced share modal** with four cards (sections):

```
┌─────────────────────────────────────────────┐
│  Share Dashboard                         ×  │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 🛡️ Organization Access    [Default]  │   │ ← Card 1 (read-only)
│  │ Users in your org with proper        │   │
│  │ permissions can access this.         │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 🔗 Public Access          [Toggle]   │   │ ← Card 2 (existing)
│  │ Anyone with the link can view        │   │
│  │                                      │   │
│  │ ⚠️ Your data is exposed to internet  │   │ ← Warning (if enabled)
│  │                                      │   │
│  │ [Copy Public Link]                   │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 👥 Share with Team Members           │   │ ← Card 3 (Phase 3 new)
│  │                                      │   │
│  │ [Search users...]                    │   │ ← Combobox
│  │                                      │   │
│  │ Shared with (3):                     │   │
│  │                                      │   │
│  │ Sarah Lee    [Can edit ▾]      [×]   │   │ ← User row
│  │ John Doe     [Can view ▾]      [×]   │   │
│  │ Jane Smith   [Can view ▾]      [×]   │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ ✉️ Share via Email                   │   │ ← Card 4 (reports only)
│  │ [email chips + message + send button]│   │
│  └─────────────────────────────────────┘   │
│                                             │
│                          [Close]            │
└─────────────────────────────────────────────┘
```

### Card Breakdown

**Card 1: Organization Access** (always shown, read-only)
- Clarifies that org-level access is the default
- Badge says "Default"
- Not interactive (informational only)

**Card 2: Public Access** (already implemented)
- Toggle switch to enable/disable
- Warning appears when enabled
- Copy link button when enabled
- Analytics (view count) if available

**Card 3: Share with Team Members** (Phase 3 — new feature)
- User search combobox (type to filter)
- Dropdown per user: "Can view", "Can edit", "Can manage"
- Remove button per user (X icon)
- Dynamic "Shared with (N)" count

**Card 4: Share via Email** (already implemented, reports only)
- Email input with chips
- Personal message textarea
- Send button

### Card 3 Deep Dive (User-to-User Sharing)

**Search Combobox:**
```
┌─────────────────────────────────────┐
│ Search by name or email...          │
└─────────────────────────────────────┘
   ↓ User types "sar"
┌─────────────────────────────────────┐
│ Sarah Lee                           │ ← Result row
│ sarah@ngo.org • Org Admin           │    (avatar + name + email + role badge)
├─────────────────────────────────────┤
│ Sarah Johnson                       │
│ sjohnson@ngo.org • Analyst          │
└─────────────────────────────────────┘
```

**User Row (after adding):**
```
┌───────────────────────────────────────────┐
│ Sarah Lee     [Can edit ▾]          [×]   │
└───────────────────────────────────────────┘
  ↑ Name         ↑ Permission dropdown  ↑ Remove
```

**Permission Dropdown:**
```
[Can edit ▾]
   ↓ Expands to:
┌─────────────────────────────────┐
│ Can view                        │ ← View dashboards and charts
│ Can edit         ✓              │ ← Edit dashboards and charts (selected)
│ Can manage                      │ ← Edit and manage sharing settings
└─────────────────────────────────┘
```

### Copy (Exact Text)

**Card Headings:**
- Card 1: "Organization Access"
- Card 2: "Public Access"
- Card 3: "Share with Team Members"
- Card 4: "Share via Email"

**Card Descriptions:**
- Card 1: "Users in your organization with proper permissions can access this [dashboard/report/chart]."
- Card 2: "Anyone with the link can view this [dashboard/report/chart]."
- Card 3: "Give specific users access to this [dashboard/report/chart]."
- Card 4: "Send a PDF and link to recipients. Public access will be enabled automatically."

**Permission Levels:**
| Value | Label | Description (tooltip) |
|-------|-------|----------------------|
| view | Can view | View dashboards and charts |
| edit | Can edit | Edit dashboards and charts |
| manage | Can manage | Edit and manage sharing settings |

**Search States:**
- Placeholder: "Search by name or email..."
- No results: "No users found"
- Too many results: "Showing 10 of [N] results. Keep typing..."

**Toast Messages:**
- Added user: "[Name] can now view this dashboard"
- Changed permission: "[Name] can now edit this dashboard"
- Removed user: "[Name] can no longer access this dashboard"

### Interactions

**Adding a User:**
1. User types in search box
2. Results appear (debounced 300ms)
3. User clicks a result
4. API call: `POST /sharing/dashboard/{id}/`
5. User appears in "Shared with" list
6. Search box clears
7. Toast notification

**Changing Permission:**
1. Click permission dropdown
2. Select new permission
3. API call: `PATCH /sharing/dashboard/{id}/{share_id}/`
4. Dropdown updates
5. Toast notification

**Removing a User:**
1. Click X button
2. Confirmation: "Remove [Name]'s access?" / "Cancel" / "Remove"
3. API call: `DELETE /sharing/dashboard/{id}/{share_id}/`
4. Row fades out
5. Toast notification

### Search Filtering Logic

**Show:**
- All users in the organization
- Sorted by: Name (A-Z)
- Max 10 results at a time

**Filter out:**
- Current user (yourself)
- Users already in "Shared with" list
- (Optional) Users who already have org-wide access (e.g., Org Admins)

### Edge Cases

| Situation | Behavior |
|-----------|----------|
| User shares with 20+ people | Show info after 20: "You've shared with 20 people. Consider using Public Access for larger audiences." |
| Only Org Admins can grant "Can manage" | If Analyst tries to select it: Disabled with tooltip "Only Org Admins can grant management access" |
| User removes last person with "Can manage" | Warning: "You may lose access. Continue?" |
| API fails during share | Optimistic UI: Show user, then remove on error. Toast: "Failed to share. [Retry]" |

### Accessibility

- **Keyboard**: Tab through search → results → permission dropdowns → remove buttons
- **Screen reader**: "Search users to share with. Type to filter. 2 results found. Sarah Lee, sarah@ngo.org, Org Admin."
- **Focus**: On modal open → search input. After adding user → back to search.

---

## 6. Role Information/Education

### The Problem
Users need to understand what each role can do **without reading documentation**. The invite flow has brief descriptions, but sometimes people want to compare side-by-side.

### The Solution

**Three places to learn about roles:**

### 1. Role Guide Tab (Settings)

Full comparison table in Settings > User Management > Role Guide:

```
┌─────────────────────────────────────────────────────────┐
│ Understanding Roles in Dalgo                            │
│                                                         │
│ Choose the right role based on what each person does.  │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐│
│ │          Org Admin Pipeline  Analyst  Viewer        ││
│ │                     Manager                         ││
│ ├─────────────────────────────────────────────────────┤│
│ │ WHO IS THIS FOR?                                    ││
│ ├─────────────────────────────────────────────────────┤│
│ │ Team leads, Data mgrs, M&E officers, Program staff ││
│ │ M&E managers IT staff   Analysts     Partners      ││
│ │                                                     ││
│ ├─────────────────────────────────────────────────────┤│
│ │ MANAGE USERS & SETTINGS                             ││
│ ├─────────────────────────────────────────────────────┤│
│ │ Invite users     ✓         —          —      —     ││
│ │ Change roles     ✓         —          —      —     ││
│ │ Manage billing   ✓         —          —      —     ││
│ │                                                     ││
│ ├─────────────────────────────────────────────────────┤│
│ │ BUILD DATA INFRASTRUCTURE                           ││
│ ├─────────────────────────────────────────────────────┤│
│ │ Connect sources  ✓         ✓          —      —     ││
│ │ Transform data   ✓         ✓          —      —     ││
│ │ Run pipelines    ✓         ✓          —      —     ││
│ │                                                     ││
│ ├─────────────────────────────────────────────────────┤│
│ │ WORK WITH DATA                                      ││
│ ├─────────────────────────────────────────────────────┤│
│ │ View dashboards  ✓         ✓          ✓      ✓    ││
│ │ Create dashboards✓         —          ✓      —     ││
│ │ View charts      ✓         ✓          ✓      ✓    ││
│ │ Create charts    ✓         —          ✓      —     ││
│ │ Share dashboards ✓         —          ✓      —     ││
│ │ Download PDFs    ✓         ✓          ✓      ✓    ││
│ └─────────────────────────────────────────────────────┘│
│                                                         │
│ Common Questions                                        │
│ ─────────────────────────────────────────────────────── │
│                                                         │
│ ▸ What if someone needs to both build pipelines and    │
│   create dashboards?                                   │
│                                                         │
│   Assign them as Org Admin if they also manage users,  │
│   otherwise choose based on their main responsibility. │
│                                                         │
│ ▸ Can a Viewer comment on dashboards?                  │
│                                                         │
│   Yes! Viewers can comment and export PDFs. They just  │
│   can't edit or create new content.                    │
│                                                         │
│ ▸ Can I change someone's role later?                   │
│                                                         │
│   Yes, you can change any user's role at any time from │
│   the Members tab.                                     │
│                                                         │
│ ▸ What happens when I change someone's role?           │
│                                                         │
│   They immediately gain or lose access. If they're     │
│   using Dalgo, they'll see changes after refreshing.   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2. Inline Help (Invite Modal)

When user clicks "Compare all roles" during invite, an accordion expands with a condensed table:

```
┌──────────────────────────────────────────┐
│ Not sure? [Compare all roles]            │ ← Link (collapsed)
└──────────────────────────────────────────┘
  ↓ Expands to:
┌──────────────────────────────────────────┐
│ [Hide guide]                             │ ← Link (now says "Hide")
│                                          │
│ Quick Comparison:                        │
│ ┌──────────────────────────────────────┐│
│ │ Org Admin      Can do everything     ││
│ │                Best for: Team leads  ││
│ ├──────────────────────────────────────┤│
│ │ Pipeline Mgr   Build data pipelines  ││
│ │                Best for: IT staff    ││
│ ├──────────────────────────────────────┤│
│ │ Analyst        Create dashboards     ││
│ │                Best for: M&E officers││
│ ├──────────────────────────────────────┤│
│ │ Viewer         View only             ││
│ │                Best for: Program staff││
│ └──────────────────────────────────────┘│
└──────────────────────────────────────────┘
```

### 3. User Profile (Your Role Info)

Small card in account settings showing "Your Role":

```
┌─────────────────────────────────────┐
│ Your Role: Analyst                  │
│                                     │
│ You can:                            │
│ • Create dashboards, charts, reports│
│ • View and analyze data             │
│ • Share with your team              │
│ • Download PDFs and exports         │
│                                     │
│ You cannot:                         │
│ • Invite or manage users            │
│ • Build data pipelines              │
│ • Connect new data sources          │
│                                     │
│ Need additional access?             │
│ Contact Sarah Lee (sarah@ngo.org)   │
└─────────────────────────────────────┘
```

### Copy (Section Headers)

**Table Sections:**
- "WHO IS THIS FOR?"
- "MANAGE USERS & SETTINGS"
- "BUILD DATA INFRASTRUCTURE"
- "WORK WITH DATA"

**Checkmarks:**
- ✓ = Yes (green `text-green-600`)
- — = No (gray `text-gray-300`)

**FAQ Questions:**
1. "What if someone needs to both build pipelines and create dashboards?"
2. "Can a Viewer comment on dashboards?"
3. "Can I change someone's role later?"
4. "What happens when I change someone's role?"

### Edge Cases

| Situation | Behavior |
|-----------|----------|
| Mobile view | Table scrolls horizontally, sticky first column (role names) |
| Print view | Table fits one page (landscape), FAQ expanded |
| User has custom role not in table | Profile card shows: "You have custom permissions. Contact your admin for details." |

---

## 7. Viewer Experience

### The Problem
Viewers are read-only users. They shouldn't see "Create" buttons or edit controls. Their interface should feel purposefully simple, not broken.

### The Solution

**Simplified, consumption-focused interface** with no creation/editing UI.

### Viewer's Sidebar

```
│ Impact          ← Home screen
│ Charts          ← Browse charts
│ Dashboards      ← Browse dashboards
│ Reports         ← Browse reports
│ Settings ▾
│   └─ About
```

**Not visible:**
- Metrics
- Data (entire section)
- Alerts
- Settings > Billing
- Settings > User Management

### Viewer's Home Screen

Landing page shows curated content:

```
┌─────────────────────────────────────────────────────────┐
│ Welcome back, Jane                                      │
│                                                         │
│ Your Dashboards (4)                                     │
│ ┌────────────────┬────────────────┬────────────────┐   │
│ │ [Dashboard 1]  │ [Dashboard 2]  │ [Dashboard 3]  │   │
│ │ 12 charts      │ 8 charts       │ 5 charts       │   │
│ │ [Open]         │ [Open]         │ [Open]         │   │
│ └────────────────┴────────────────┴────────────────┘   │
│                                                         │
│ Recent Reports (2)                                      │
│ ┌─────────────────────────────────────────────────┐    │
│ │ Monthly Impact Report - March 2026              │    │
│ │ Created by Sarah Lee • 2 days ago               │    │
│ │ [View Report] [Download PDF]                    │    │
│ └─────────────────────────────────────────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Dashboard View (Viewer)

When a Viewer opens a dashboard:

```
┌─────────────────────────────────────────────────────────┐
│ Sales Dashboard                                         │
│ Last updated: 2 hours ago                               │
│                                                         │
│ [🔖 Bookmark] [💬 Comments (3)] [⬇️ Download PDF]       │ ← Actions
│                                                         │
├─────────────────────────────────────────────────────────┤
│ [Charts render in grid layout]                          │
│                                                         │
│ [Chart 1]      [Chart 2]      [Chart 3]                │
│                                                         │
│ [Chart 4]      [Chart 5]      [Chart 6]                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**What Viewer Can Do:**
- View all charts
- Use chart filters (date, category, etc.)
- Use dashboard filters
- Bookmark dashboard (save to favorites)
- Add/view comments
- Download PDF
- Refresh data

**What's Hidden:**
- "Edit Dashboard" button
- "Add Chart" button
- Chart edit/delete controls
- Dashboard settings icon
- Share button

### Charts List (Viewer)

```
┌─────────────────────────────────────────────────────────┐
│ Charts                                                   │
│ Browse and view charts from your team                   │
│                                                         │
│ [Search...] [Filter: All Types ▾] [Sort: Recent ▾]     │
│                                                         │
├─────────────────────────────────────────────────────────┤
│ ┌───────────────┬───────────────┬───────────────┐      │
│ │ [Chart Card]  │ [Chart Card]  │ [Chart Card]  │      │
│ │ Bar Chart     │ Line Chart    │ Pie Chart     │      │
│ │ By Sarah Lee  │ By John Doe   │ By Sarah Lee  │      │
│ │ [View]        │ [View]        │ [View]        │      │
│ └───────────────┴───────────────┴───────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**No "Create Chart" button** in top-right corner.

### Commenting Feature

Viewers can comment. Comment thread UI:

```
┌─────────────────────────────────────────────┐
│ Comments (3)                                │
│                                             │
│ Sarah Lee • 2 hours ago                     │
│ "The March numbers look great! 🎉"          │
│ [Reply]                                     │
│                                             │
│   └─ John Doe • 1 hour ago                  │
│      "Agreed! Let's discuss tomorrow."      │
│      [Reply]                                │
│                                             │
│ Jane Smith • 5 hours ago                    │
│ "Can you add a filter for region?"          │
│ [Reply]                                     │
│                                             │
│ ┌─────────────────────────────────────────┐ │
│ │ Add a comment...                        │ │
│ └─────────────────────────────────────────┘ │
│                    [Post Comment]           │
└─────────────────────────────────────────────┘
```

**Viewer Can:**
- View all comments
- Add new comments
- Reply to comments
- Edit/delete their own comments

**Viewer Cannot:**
- Resolve comments (creators/editors only)
- Delete others' comments

### Copy (Viewer-Specific)

**Home Heading**: "Welcome back, [FirstName]"

**Empty States:**
- Dashboards: "No dashboards yet. Ask your team to share dashboards with you."
- Charts: "No charts available. Charts will appear here once your team creates them."
- Reports: "No reports yet. Reports will appear here once created."

**Button Labels:**
- "Download PDF" (not "Export")
- "Post Comment" (not "Submit")
- "View Report" (not "Open")

### First-Time Viewer Onboarding

On first login, show tooltip on sidebar:

```
┌─────────────────────────────────────────┐
│ You're a Viewer                         │
│                                         │
│ You can browse dashboards, charts, and  │
│ reports, and leave comments.            │
│                                         │
│ Contact your admin to request edit      │
│ access.                                 │
│                                         │
│                   [Got it]              │
└─────────────────────────────────────────┘
```

Auto-dismisses after 10 seconds.

### Edge Cases

| Situation | Behavior |
|-----------|----------|
| Viewer lands on dashboard they can't access | Redirect to No Access page: "This dashboard is no longer shared with you. Contact [Owner] to request access." |
| Chart fails to load | Show error state: "Failed to load chart. [Retry]" (no edit button shown) |
| Viewer's role upgraded mid-session | Next API call refreshes permissions → sidebar updates → Toast: "Your role has been updated to Analyst!" |
| Comment contains @mention | Currently plain text. Future: Highlight mentions, send email notifications. |

---

## Summary: Design System Tokens

### Component Library
All UI built with **Shadcn components** (Radix UI + Tailwind):
- Dialog, Alert Dialog, Card, Radio Group, Select, Switch
- Combobox, Badge, Button, Table, Tooltip

### Color Palette

| Element | Color | Tailwind Class |
|---------|-------|----------------|
| Primary (teal) | #00897B | `bg-primary` or `style={{ backgroundColor: 'var(--primary)' }}` |
| Org Admin badge | Purple | `bg-purple-100 text-purple-700 border-purple-300` |
| Pipeline Manager badge | Blue | `bg-blue-100 text-blue-700 border-blue-300` |
| Analyst badge | Green | `bg-green-100 text-green-700 border-green-300` |
| Viewer badge | Gray | `bg-gray-100 text-gray-700 border-gray-300` |
| Destructive (red) | - | `text-destructive` or `variant="destructive"` |
| Muted text | - | `text-muted-foreground` |

### Typography

| Usage | Class |
|-------|-------|
| Page headings | `text-3xl font-bold` |
| Section headings | `text-xl font-semibold` |
| Card titles | `text-sm font-medium` |
| Body text | `text-base` |
| Descriptions | `text-sm text-muted-foreground` |
| Labels | `text-xs font-medium` |

### Spacing

| Usage | Value |
|-------|-------|
| Card padding | `p-4` |
| Section gaps | `space-y-6` |
| Button groups | `gap-3` |

---

## Files to Create/Modify

### Phase 2 (Frontend Navigation & Route Protection)

**Modify:**
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/main-layout.tsx`
  - Add `requiredPermission` field to nav items
  - Filter items based on user permissions

**Create:**
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/ui/permission-gate.tsx`
  - Route guard wrapper component
  - Checks permission, shows no access page if denied

- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/ui/no-access-page.tsx`
  - Full-page "You don't have access" component
  - Shows admin contact info, auto-redirect

### Phase 3 (Resource Sharing)

**Create:**
- `/Users/siddhant/Documents/Dalgo/webapp_v2/hooks/api/useSharing.ts`
  - SWR hooks: `useSharedUsers`, `useShareDashboard`, `useUnshare`

- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/ui/user-share-section.tsx`
  - "Share with Team Members" card component
  - User search combobox + shared user list

**Modify:**
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/ui/share-modal.tsx`
  - Insert Card 3 (user sharing) between public toggle and email sections

### User Management (TBD based on existing code)

**Find and modify:**
- User invite modal component (likely in `/app/settings/user-management/` or `/components/settings/`)
- User management table component

**Create:**
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/settings/role-guide.tsx`
  - Full comparison table component
- `/Users/siddhant/Documents/Dalgo/webapp_v2/components/settings/role-info-card.tsx`
  - User profile "Your Role" card

---

## Next Steps

1. **Phase 1 (Backend)** — Update role permissions in seed data (not frontend work)
2. **Phase 2 (This design)** — Implement permission-gated navigation + no access page
3. **Phase 3 (This design)** — Add user-to-user sharing to share modal
4. **User Management (This design)** — Build invite flow + role management table + role guide

All designs are ready for implementation. Each section includes exact copy, component specs, interaction flows, edge cases, and accessibility requirements.