# DALGO DASHBOARD TABS
## Technical Specification
**Version 1.0 • March 2026**

---

## What We're Building

A tabs system for dashboards that allows users to organize charts into multiple tabs — replacing cluttered single-view dashboards with organized, tabbed views.

---

## Delivery Timeline

| Milestone | Dates | Duration | Focus |
|-----------|-------|----------|-------|
| Milestone 1 | Mar 19-20 (Wed-Thu) | 2 days | Tab Bar UI & Tab Management |
| Milestone 2 | Mar 21, 24 (Fri, Mon) | 2 days | Add Charts Inside Tabs |
| Milestone 3 | Mar 25-27 (Tue-Thu) | 3 days | Backend Integration & Backward Compatibility |
| Milestone 4 | Mar 28, 31, Apr 1 (Fri, Mon, Tue) | 3 days | Preview Mode Logic & Testing |

**Total: 10 working days • Start: March 19 • End: April 1**
**Note:** Weekends (Sat-Sun) excluded

---

## Milestone 1 — Tab Bar UI & Tab Management

**Mar 19-20 (Wed-Thu) | Goal:** Users can see tabs, switch between them, add new tabs, rename, and remove tabs.

### What We're Building

| Feature | Description |
|---------|-------------|
| Tab Bar UI | Horizontal bar showing all tabs with active tab highlighted |
| Default Tab | New dashboards start with "Untitled Tab 1", new tabs are "Untitled Tab 2", "Untitled Tab 3", etc. |
| Add Tab | Click "+" to add a new tab (no limit) |
| Remove Tab | Click "x" to remove a tab (cannot remove last tab) |
| Rename Tab | Double-click tab name to rename |
| Switch Tab | Click on tab to switch view |

### How It Works

When a user creates a new dashboard, it starts with a single "Untitled Tab 1". The tab bar appears at the top of the dashboard canvas. Users can add unlimited tabs (named "Untitled Tab 2", "Untitled Tab 3", etc.). Each tab maintains its own layout and charts independently. Switching tabs instantly shows that tab's content.

When an existing dashboard (without tabs) is opened in edit mode, all existing charts are automatically displayed inside a default "Untitled Tab 1" — providing a unified editing experience.

### UI Specifications

| Element | Behavior |
|---------|----------|
| Tab Bar | Horizontal, below dashboard title, above chart area |
| Active Tab | Highlighted with bottom border or background color |
| Add Button (+) | Appears after last tab, visible only in edit mode |
| Remove Button (x) | Appears on each tab (except when only 1 tab), visible only in edit mode |
| Max Tabs | No limit |
| Tab Name | Max 50 characters |

### Deliverables

- Tab Bar component with static tabs
- Tab switching functionality
- Add tab, remove tab, rename tab
- Testing & bug fixes

---

## Milestone 2 — Add Charts Inside Tabs

**Mar 21, 24 (Fri, Mon) | Goal:** Charts are added inside the active tab, not directly on the dashboard.

### What We're Building

| Feature | Description |
|---------|-------------|
| Chart Addition Flow | User selects tab → clicks "Add Chart" → chart appears in active tab |
| Per-Tab Layout | Each tab has its own grid layout and components |
| Duplicate Charts | Same chart can exist in multiple tabs |

### How It Works

When a user clicks "Add Chart", the system checks which tab is currently active and adds the chart to that tab's layout. Each tab maintains its own `layout_config` and `components` structure. Users can add the same chart to multiple tabs if needed — each instance is independent.

### Data Flow

| Step | Action |
|------|--------|
| 1 | User selects a tab (clicks on it) |
| 2 | User clicks "Add Chart" button |
| 3 | Chart selector modal opens |
| 4 | User selects a chart |
| 5 | Chart is added to active tab's `layout_config` and `components` |
| 6 | Tab content re-renders with new chart |

### Deliverables

- Update "Add Chart" flow to target active tab
- Update "Add Text" flow to target active tab
- Duplicate chart support
- Per-tab layout management

---

## Milestone 3 — Backend Integration & Backward Compatibility

**Mar 25-27 (Tue-Thu) | Goal:** Persist tabs to database. Existing dashboards continue to work without changes.

### What We're Building

| Feature | Description |
|---------|-------------|
| Tabs Field | New `tabs` JSONField on Dashboard model |
| Save/Load Tabs | API updates to handle tabs structure |
| Backward Compatibility | Existing dashboards (tabs = null) render normally |
| Auto-Migration on Edit | Existing dashboards get wrapped in "Untitled Tab" when edited |

### How It Works

A new `tabs` JSONField is added to the Dashboard model. When saving a dashboard with tabs, the entire tabs structure is stored in this field. When loading, if `tabs` is null, the dashboard renders using the existing `layout_config` and `components` (backward compatible). If `tabs` has data, it renders using the tabs structure.

### Data Model

**Dashboard Model (updated)**

| Field | Type | Description |
|-------|------|-------------|
| id | BigInt PK | Primary key |
| title | VARCHAR(255) | Dashboard title |
| layout_config | JSONField | Grid layout (used by old dashboards) |
| components | JSONField | Component configs (used by old dashboards) |
| tabs | JSONField, nullable | **NEW** — Tabs structure. NULL = old dashboard |

**Tabs JSON Structure**

```json
{
  "tabs": [
    {
      "id": "tab-1",
      "title": "Untitled Tab 1",
      "order": 0,
      "layout_config": [...],
      "components": {...}
    },
    {
      "id": "tab-2",
      "title": "Untitled Tab 2",
      "order": 1,
      "layout_config": [...],
      "components": {...}
    }
  ],
  "activeTabId": "tab-1"
}
```

**Field Descriptions**

| Field | Type | Description |
|-------|------|-------------|
| tabs | Array | List of tab objects |
| tabs[].id | String | Unique tab identifier |
| tabs[].title | String | Tab display name (max 50 chars) |
| tabs[].order | Integer | Tab position (0-indexed) |
| tabs[].layout_config | Array | Grid layout items for this tab |
| tabs[].components | Object | Component configs for this tab |
| activeTabId | String | Currently active tab ID |

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/dashboards/:id/ | Returns dashboard with `tabs` field (null or data) |
| PUT | /api/dashboards/:id/ | Accepts `tabs` in request body, validates structure |
| POST | /api/dashboards/ | Creates dashboard with default "Untitled Tab 1" in `tabs` |

### Validation Rules

| Rule | Validation |
|------|------------|
| Max tabs | No limit |
| Tab title | Required, max 50 characters |
| Tab ID | Required, unique within dashboard |
| At least one tab | Required when `tabs` is not null |

### Backward Compatibility

| Dashboard State | `tabs` Field | Rendering |
|-----------------|--------------|-----------|
| Old (existing) | `null` | Use `layout_config` & `components` directly |
| New (with tabs) | Has data | Use `tabs` structure |
| Old opened in edit | `null` | Frontend wraps in "Untitled Tab 1" |
| Old saved after edit | Has data | Converted to tabs structure |

### Deliverables

- Add `tabs` field to Dashboard model, migration
- Update GET/PUT API to handle tabs
- Frontend: save/load tabs on dashboard save/load
- Backward compatibility for existing dashboards

---

## Milestone 4 — Preview Mode Logic & Testing

**Mar 28, 31, Apr 1 (Fri, Mon, Tue) | Goal:** Preview mode hides tab bar when only 1 tab. Full end-to-end testing.

### What We're Building

| Feature | Description |
|---------|-------------|
| Single Tab Preview | When 1 tab exists, hide tab bar, show charts directly |
| Multi-Tab Preview | When 2 or more tabs exist, show tab bar with tab switching |
| No Edit Controls | No add/remove/rename buttons visible in preview |

### How It Works

In preview (view) mode, the system checks how many tabs exist. If only 1 tab, the tab bar is hidden completely and charts render as a normal dashboard. If 2 or more tabs exist, the tab bar appears (without edit controls) and users can click to switch between tabs.

### Preview Mode Behavior

| Tabs Count | Tab Bar | Edit Controls | User Experience |
|------------|---------|---------------|-----------------|
| 1 tab | Hidden | Hidden | Normal dashboard view |
| 2 or more tabs | Visible | Hidden | Tab switching only |

### Deliverables

- Preview mode: conditional tab bar rendering
- End-to-end testing: new & existing dashboards
- Bug fixes, UX polish

---

## Deliverables by Date

| Date | Milestone | Deliverable |
|------|-----------|-------------|
| Mar 20 (Thu) | M1 – Complete | Tab Bar UI, add/remove/rename tabs, tab switching |
| Mar 24 (Mon) | M2 – Complete | Charts added inside tabs, duplicate chart support |
| Mar 27 (Thu) | M3 – Complete | Backend `tabs` field, API updates, backward compatibility |
| Apr 1 (Tue) | M4 – Complete | Preview mode logic, full testing, ready for deployment |

---

## Key Rules Summary

| Rule | Details |
|------|---------|
| Default tab | New dashboards start with "Untitled Tab 1" |
| Charts location | Always inside tabs (not directly on dashboard) |
| Max tabs | No limit |
| Duplicate charts | Allowed in different tabs |
| Existing dashboards | No migration needed, continue to work |
| Edit mode (existing) | Existing charts shown in "Untitled Tab 1" |
| Preview (1 tab) | Tab bar hidden, normal dashboard look |
| Preview (2 or more tabs) | Tab bar visible, click to switch |

---

## Key Risks & Mitigations

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Existing dashboard breakage | Low | Backward compat with null check; extensive testing |
| Performance with many charts | Low | Same rendering as current dashboards |
| User confusion with new UI | Medium | Single tab = hidden tab bar; familiar experience |
| Data loss on tab delete | Medium | Confirm dialog before deleting tab with charts |

---

**Dalgo Engineering Team • March 2026**
