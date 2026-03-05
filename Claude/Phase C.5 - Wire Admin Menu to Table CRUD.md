---
fileClass: Project
Category: Claude
Status: Complete
Authors: Mason Bendixen
Last Updated: 2/11/2026
Version: 1.0
tags:
---
# Overview

Phase C.5 wires the new Generic Table CRUD Controls (built in phases C.0-C.4) into the Admin menu and dashboard, replacing the legacy table editor routes.

## Current State

**Admin Menu** (`ui/src/app/shared/services/header/mockHeaderResponse.ts`):
- "Admin" → `/admin` (dashboard)
- "Manage Users" → `/admin/users` (legacy route → `EditDbTableComponent`)
- "Class Schedule" → `/admin/classes` (legacy route → `EditDbTableComponent`)

**Admin Dashboard** (`ui/src/app/pages/admin/dashboard/admin.component.ts`):
- Table dropdown navigates to `/:tableName` relative route (legacy)

**New Routes** (from Phase C.3):
- `/admin/tables/:tableName/view/:pageSize/:pageOffset` → `TableViewPageComponent`
- `/admin/tables/:tableName/edit/:id` → `TableEditPageComponent`
- `/admin/tables/:tableName/new` → `TableNewPageComponent`

---

# Work Items

## C.5.1 - Update Admin Menu URLs

**File**: `ui/src/app/shared/services/header/mockHeaderResponse.ts`

Update the `adminDropdown` menu items to use the new table CRUD routes:

| Menu Item | Current URL | New URL |
|-----------|-------------|---------|
| Admin | `/admin` | `/admin` (no change) |
| Manage Users | `/admin/users` | `/admin/tables/people/view/10/0` |
| Class Schedule | `/admin/classes` | `/admin/tables/classes/view/10/0` |

**Code Change**:
```typescript
const adminDropdown: HeaderButton = {
  title: 'Admin',
  kind: HeaderButtonKind.Dropdown,
  menu: [
    {
      title: 'Admin',
      kind: HeaderButtonKind.InternalLink,
      goTo: '/admin',
    },
    {
      title: 'Manage Users',
      kind: HeaderButtonKind.InternalLink,
      goTo: '/admin/tables/people/view/10/0',
    },
    {
      title: 'Class Schedule',
      kind: HeaderButtonKind.InternalLink,
      goTo: '/admin/tables/classes/view/10/0',
    },
  ],
};
```

---

## C.5.2 - Update Admin Dashboard Table Selector

**File**: `ui/src/app/pages/admin/dashboard/admin.component.ts`

Update the table selection navigation to use the new routes.

**Current Code** (line 64):
```typescript
this._router.navigate([selectedTableName], {
  relativeTo: this._activatedRoute,
});
```

**New Code**:
```typescript
this._router.navigate(['tables', selectedTableName, 'view', '10', '0'], {
  relativeTo: this._activatedRoute,
});
```

---

## C.5.3 - Add Default Route Redirect

**File**: `ui/src/app/pages/admin/dashboard/admin.component.ts`

When the admin dashboard loads with no child route active, automatically navigate to the first table alphabetically (first in `root_tables` dropdown).

**Implementation**: In `ngOnInit()`, after loading the schema, check if no child route is active and navigate to the first table:

```typescript
// After schema loads, if no child route active, navigate to first table
if (!this._activatedRoute.firstChild?.snapshot.paramMap.get('tableName')) {
  const firstTable = this.databaseSchema.root_tables[0];
  if (firstTable) {
    this._router.navigate(['tables', firstTable, 'view', '10', '0'], {
      relativeTo: this._activatedRoute,
    });
  }
}
```

This approach is dynamic - it uses whatever table appears first in `root_tables` from the schema.

---

## C.5.4 - Update Tests

**Files to update**:
- `ui/src/app/shared/services/header/header.service.spec.ts` (if exists)
- `ui/src/app/pages/admin/dashboard/admin.component.spec.ts`
- `ui/src/app/pages/admin/table-crud.integration.spec.ts`

Add/update tests to verify:
1. Admin menu URLs point to new routes
2. Table selector navigates to new routes
3. Navigation flow works end-to-end

---

# Testing Checklist

- [ ] Admin menu "Manage Users" navigates to `/admin/tables/people/view/10/0`
- [ ] Admin menu "Class Schedule" navigates to `/admin/tables/classes/view/10/0`
- [ ] Dashboard table selector navigates to `/admin/tables/{tableName}/view/10/0`
- [ ] Edit button from table view navigates to `/admin/tables/{tableName}/edit/{id}`
- [ ] Add button from table view navigates to `/admin/tables/{tableName}/new`
- [ ] After edit/create, user returns to table view via `returnUrl`
- [ ] All existing tests pass

---

# Dependencies

**Requires**: Phases C.0-C.4 complete (already done)

**Tables available** (from `ServerAccess.mock.ts`):
- `classes` (friendly: "Classes")
- `people` (friendly: "People")

---

# Notes

- The legacy route `/:tableName` → `EditDbTableComponent` can be kept temporarily for backwards compatibility, or removed if no longer needed
- Page size default of 10 matches what was used in Phase C.4 integration tests
- The `people` table is used for "Manage Users" since that's the table containing user data

---

# Issue: Navigation Redirect Loop (Resolved)

## Problem

When clicking the Edit button in the table view, navigation would start to the edit page but immediately redirect back to the view page. The browser URL never changed, and the edit component never loaded.

## Root Cause

The `AdminComponent` had a subscription chain that caused unintended navigation:

1. User clicks Edit → Navigation starts to `/admin/tables/classes/edit/1`
2. `NavigationEnd` fires in `AdminComponent`
3. `AdminComponent` extracts `tableName` from the route and sets it on `TableManagementService`
4. `TableManagementService.selectedTableName$` emits the new value
5. This triggers `selectedTableNameInput.setValue('classes')` on the dropdown FormControl
6. The `valueChanges` subscription on the dropdown fires
7. **This triggers navigation back to `/admin/tables/classes/view/10/0`** - hijacking the original edit navigation

The dropdown was designed to navigate when the user selects a table, but it was also firing when the value was set programmatically during route changes.

## Solution

Use `{ emitEvent: false }` when setting the dropdown value programmatically:

```typescript
// Before (broken)
this.selectedTableNameInput.setValue(tableName ?? '');

// After (fixed)
this.selectedTableNameInput.setValue(tableName ?? '', { emitEvent: false });
```

This prevents the `valueChanges` subscription from firing when the value is set programmatically, while still allowing it to fire when the user interacts with the dropdown.

## Prevention

When using Angular Reactive Forms with FormControls that have `valueChanges` subscriptions that trigger side effects (like navigation), always use `{ emitEvent: false }` when setting values programmatically to avoid unintended side effects.
