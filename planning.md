# Dalgo Dashboard Tabs - Implementation Plan

## Overview

Adding a tabs feature to dashboards. Users can organize charts into multiple tabs for better organization.

---

## How It Works

### Creating & Editing Dashboard

- New dashboard starts with one **"Untitled Tab"**
- User selects a tab → clicks "Add Chart" → chart added to that tab
- User can add more tabs using **"+"** button (max 10 tabs)
- User can rename tabs (double-click) and remove tabs (click x)
- Cannot remove the last tab

### Viewing Dashboard (Preview Mode)

| Tabs Count | What User Sees |
|------------|----------------|
| 1 tab | Normal dashboard view (no tab bar) |
| 2+ tabs | Tab bar visible, click to switch |

---

## Key Rules

| Rule | Details |
|------|---------|
| Charts location | Always inside tabs (not directly on dashboard) |
| Duplicate charts | Allowed in different tabs |
| Max tabs | 10 per dashboard |
| Existing dashboards | No migration needed, continue to work as before |

---

## Implementation Steps

### Backend Changes

| # | Task | Details |
|---|------|---------|
| 1 | Add `tabs` field to Dashboard model | New JSONField to store tabs data |
| 2 | Update Dashboard API | Save/load tabs when creating/updating dashboard |
| 3 | Add validation | Max 10 tabs, required fields (id, title, order) |
| 4 | No migration needed | Existing dashboards have `tabs = null`, continue to work |

### Frontend Changes

| # | Task | Details |
|---|------|---------|
| 1 | Create Tab Bar UI | Add, remove, rename, switch tabs |
| 2 | Update "Add Chart" flow | Charts go inside active tab |
| 3 | New dashboard setup | Start with default "Untitled Tab" |
| 4 | Preview mode logic | Hide tab bar when only 1 tab |
| 5 | Backward compatibility | Check if `tabs` exists, render accordingly |
| 6 | Testing | Test all scenarios |

---

## Data Structure

### Backend Schema

**Dashboard Model (updated):**
```
Dashboard
├── id
├── title
├── layout_config    ← Used by old dashboards
├── components       ← Used by old dashboards
├── tabs (NEW)       ← JSONField for tabs data (null for old dashboards)
└── ... other fields
```

**Tabs JSON Structure:**
```json
{
  "tabs": [
    {
      "id": "tab-1",
      "title": "Untitled Tab",
      "order": 0,
      "layout_config": [...],
      "components": {...}
    },
    {
      "id": "tab-2",
      "title": "Sales",
      "order": 1,
      "layout_config": [...],
      "components": {...}
    }
  ],
  "activeTabId": "tab-1"
}
```

### How It Works

| Dashboard Type | `tabs` field | Rendering |
|----------------|--------------|-----------|
| Old (existing) | `null` | Use `layout_config` & `components` directly |
| New (with tabs) | Has data | Use tabs structure |

---

## Visual Reference

**Edit Mode:**
```
┌──────────────────────────────────────────────┐
│ [Untitled Tab] [Sales Tab] [+]               │
├──────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐                    │
│  │ Chart 1 │  │ Chart 2 │                    │
│  └─────────┘  └─────────┘                    │
└──────────────────────────────────────────────┘
```

**Preview Mode (1 tab) - no tab bar:**
```
┌──────────────────────────────────────────────┐
│  ┌─────────┐  ┌─────────┐                    │
│  │ Chart 1 │  │ Chart 2 │                    │
│  └─────────┘  └─────────┘                    │
└──────────────────────────────────────────────┘
```

**Preview Mode (2+ tabs) - tab bar visible:**
```
┌──────────────────────────────────────────────┐
│ [Untitled Tab] [Sales Tab]                   │
├──────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐                    │
│  │ Chart 1 │  │ Chart 2 │                    │
│  └─────────┘  └─────────┘                    │
└──────────────────────────────────────────────┘
```
