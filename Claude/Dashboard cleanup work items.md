---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 2/18/2026
Version: 0.1
tags: 
---
# Overview

Use this as your planning document to accomplish a number of user and admin dashboard related cleanup items. In particular, I would like to work on:

- Allow certain database generated fields to viewable but not editable (or present for new items)
- Format dates better (and generate metadata)
- Allow dates to be edited
- Have enums map to a drop down when editing
- Have bools map to a better value in the UI and a checkbox when editing

Please use this as the document for plan mode. Do not modify this Overview section but please replace the rest of the document with sections with your initial thoughts on what is needed to complete each task. Then I will work with you on fleshing out and implementing each section.

# 1. Read-Only and Auto-Generated Fields

## Design

Add two new boolean columns to `admin_column_data_info`:
- **`hidden`**: Column is not shown in the table view, not shown in create or edit forms. Useful for foreign key references and internal IDs.
- **`readonly`**: Column is shown in the table view and in edit mode (but not editable). Hidden entirely in create mode (server will auto-generate). Useful for timestamps like `created_us`, `updated_us`.

Primary keys already get implicit read-only treatment via the existing `isPrimaryKeyColumn()` logic and are not affected.

### Behavior Matrix

| Flag | Table View | Edit Form | Create Form |
|------|-----------|-----------|-------------|
| neither | Shown | Editable | Editable |
| `hidden=true` | Hidden | Hidden | Hidden |
| `readonly=true` | Shown | Shown, disabled | Hidden |
| `hidden=true` + `readonly=true` | Hidden | Hidden | Hidden |

### Initial metadata values to populate

| Table | Column | `hidden` | `readonly` | Reasoning |
|-------|--------|----------|-----------|-----------|
| purchases | created_us | — | true | Auto-generated timestamp |
| purchases | updated_us | — | true | Auto-generated timestamp |
| purchases | payer_person_id | true | — | FK, internal reference |
| payments | created_us | — | true | Auto-generated timestamp |
| payments | updated_us | — | true | Auto-generated timestamp |
| payments | payer_person_id | true | — | FK, internal reference |
| products | created_us | — | true | Auto-generated timestamp |
| products | updated_us | — | true | Auto-generated timestamp |
| people | created_at | — | true | Auto-generated timestamp |
| people | updated_at | — | true | Auto-generated timestamp |
| price_schedules | created_us | — | true | Auto-generated timestamp |
| price_schedules | updated_us | — | true | Auto-generated timestamp |
| purchase_items | created_us | — | true | Auto-generated timestamp |
| purchase_items | purchase_id | true | — | FK, internal reference |
| purchase_items | product_id | true | — | FK, internal reference |
| purchase_items | price_schedule_id | true | — | FK, internal reference |
| purchase_payments | created_us | — | true | Auto-generated timestamp |
| purchase_payments | purchase_id | true | — | FK, internal reference |
| purchase_payments | payment_id | true | — | FK, internal reference |
| entitlements | created_us | — | true | Auto-generated timestamp |
| entitlements | updated_us | — | true | Auto-generated timestamp |
| entitlements | purchase_id | true | — | FK, internal reference |
| entitlements | product_id | true | — | FK, internal reference |
| entitlement_assignments | created_us | — | true | Auto-generated timestamp |
| entitlement_assignments | entitlement_id | true | — | FK, internal reference |
| entitlement_assignments | person_id | true | — | FK, internal reference |

---

## Server Implementation

### 1.1 Add columns to `admin_column_data_info` schema

**File**: `server/knottyyoga_server/src/db_schema/admin_column_data_info.h`

- [ ] Add constant: `inline constexpr std::string_view kAdminColumnDataInfoHidden = "hidden";`
- [ ] Add constant: `inline constexpr std::string_view kAdminColumnDataInfoReadonly = "readonly";`

**File**: `server/knottyyoga_server/src/db_schema/admin_column_data_info.cpp`

- [ ] Add two new nullable bool columns at the end of `MakeAdminColumnDataInfoTable()`:
```cpp
databaseInfo.AddColumnNullable(
    kAdminColumnDataInfoTable,
    kAdminColumnDataInfoHidden,
    DB_TYPE_BOOL);
databaseInfo.AddColumnNullable(
    kAdminColumnDataInfoTable,
    kAdminColumnDataInfoReadonly,
    DB_TYPE_BOOL);
```

### 1.2 Update `AdminColumnDataInfo` table helper

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_column_data_info.h`

- [ ] Add `hidden` and `readonly` parameters to `AddAdminColumnDataInfo()`:
```cpp
void AddAdminColumnDataInfo(
    Transaction& transaction,
    std::string_view tableName,
    std::string_view columnName,
    std::string_view label,
    std::string_view hint = "",
    std::string_view placeHolder = "",
    std::string_view regex = "",
    std::string_view htmlInputType = "",
    std::string_view required = "",
    std::string_view maxLength = "",
    std::string_view defaultValue = "",
    std::string_view rows = "",
    std::string_view hidden = "",
    std::string_view readonly = "");
```

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_column_data_info.cpp`

- [ ] Add the new parameters to the `AddAdminColumnDataInfo` implementation, following the same pattern as existing optional params:
```cpp
if (!hidden.empty())
    AddToKeyValueTable(
        keyValueTable, DbSchema::kAdminColumnDataInfoHidden, hidden);
if (!readonly.empty())
    AddToKeyValueTable(
        keyValueTable, DbSchema::kAdminColumnDataInfoReadonly, readonly);
```

### 1.3 Update `GenerateColumnMetadata` in database_rest_helper.cpp

**File**: `server/knottyyoga_server/src/sql_util/json/database_rest_helper.cpp`

In the `GenerateColumnMetadata` function, add the new fields to both branches:

- [ ] In the `if(adminColumnDataInfo.GetAdminColumnDataInfo(...))` branch (has metadata), add:
```cpp
column["hidden"] = keyValueTable.at(
    static_cast<std::string>(DbSchema::kAdminColumnDataInfoHidden));
column["readonly"] = keyValueTable.at(
    static_cast<std::string>(DbSchema::kAdminColumnDataInfoReadonly));
```

- [ ] In the `else` branch (no metadata), add defaults:
```cpp
column["hidden"] = "f";
column["readonly"] = "f";
```

### 1.4 Populate metadata for auto-generated and FK columns

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`

- [ ] Update the `AddRow` lambda in `PopulateAdminColumnDataInfo` to accept `hidden` and `readonly` parameters and pass them through to `AddAdminColumnDataInfo`
- [ ] Add metadata rows for all timestamp and FK columns listed in the table above. These are columns that currently have NO `admin_column_data_info` row. Example for purchases:
```cpp
AddRow(DbSchema::kPurchasesTable, DbSchema::kPurchasesCreatedUs,
    "Created", "When this purchase was created", "text", "false",
    /*hidden=*/"", /*readonly=*/"true");
AddRow(DbSchema::kPurchasesTable, DbSchema::kPurchasesUpdatedUs,
    "Updated", "When this purchase was last updated", "text", "false",
    /*hidden=*/"", /*readonly=*/"true");
AddRow(DbSchema::kPurchasesTable, DbSchema::kPurchasesPayerPersonId,
    "Payer", "Person who made the purchase", "text", "false",
    /*hidden=*/"true", /*readonly=*/"");
```
- [ ] For existing metadata rows that already have `AddRow` calls, update them to pass the new parameters (most will pass empty strings for both, meaning neither hidden nor readonly)

### 1.5 Reset the database

- [ ] Run `knottyyoga_database_helper` to reset and initialize the database from scratch with the new schema columns and populated metadata

---

## Server Tests

### 1.6 Update `admin_column_data_info_test.cpp`

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_column_data_info_test.cpp`

- [ ] Update the `MakeDataInfo` helper to accept `hidden` and `readonly` parameters:
```cpp
KeyValueTable MakeDataInfo(
    std::string_view id,
    std::string_view tableName,
    std::string_view columnName,
    std::string_view label,
    std::string_view hint = "",
    std::string_view placeHolder = "",
    std::string_view regex = "",
    std::string_view htmlInputType = "",
    std::string_view required = "",
    std::string_view maxLength = "",
    std::string_view defaultValue = "",
    std::string_view rows = "",
    std::string_view hidden = "",
    std::string_view readonly = "")
```
- [ ] Existing tests (`AddAdminColumnDataInfoBasic`, `DeleteAdminColumnDataInfoBasic`, `GetAdminColumnDataInfoBasic`) should continue passing with no changes to their test data (new params default to empty)
- [ ] Add new test: `AddAdminColumnDataInfoWithHiddenAndReadonly` — creates rows with `hidden = "true"` and `readonly = "true"`, verifies they are stored and retrieved correctly via `GetAdminColumnDataInfos` and `GetAdminColumnDataInfo`

### 1.7 Update `database_rest_helper_test.cpp`

**File**: `server/knottyyoga_server/src/sql_util/json/database_rest_helper_test.cpp`

- [ ] Update `kTestDatabaseSchema` JSON constant to include `"hidden"` and `"readonly"` fields in each column object (defaulting to `"f"`)
- [ ] Add a new test `DatabaseMetadataWithHiddenAndReadonly` that creates columns with hidden/readonly metadata and verifies the JSON output includes the correct values

---

## Client Implementation

### 1.8 Update `ColumnDataInfo` interface

**File**: `ui/src/app/shared/types/ColumnDataInfo.ts`

- [ ] Add two new optional fields:
```typescript
export interface ColumnDataInfo {
    // ... existing fields ...
    hidden?: boolean;
    readonly?: boolean;
}
```

### 1.9 Update `Conversion.ts`

**File**: `ui/src/app/shared/utils/Conversion.ts`

- [ ] Add parsing for the new fields in `ColumnDataInfoFromJSON()`:
```typescript
return {
    // ... existing fields ...
    hidden: parseBool(input['hidden']) as boolean | undefined,
    readonly: parseBool(input['readonly']) as boolean | undefined,
};
```

### 1.10 Update `composite-row-control` — filter and readonly logic

**File**: `ui/src/app/controls/composite-row-control/composite-row-control.component.ts`

- [ ] Add a `visibleColumns()` getter that filters out columns that shouldn't be shown:
```typescript
get visibleColumns(): { col: ColumnDataInfo; index: number }[] {
    return this.columnInfos
        .map((col, index) => ({ col, index }))
        .filter(({ col }) => {
            if (col.hidden) return false;
            if (col.readonly && this.isCreateMode) return false;
            return true;
        });
}
```

- [ ] Add `isReadonlyColumn(col)` method:
```typescript
isReadonlyColumn(col: ColumnDataInfo): boolean {
    return this.isPrimaryKeyColumn(col) || col.readonly === true;
}
```

- [ ] Update `getControlValues()` to skip hidden and create-mode-readonly columns (don't send them to the server):
```typescript
getControlValues(): Record<string, string | undefined> {
    const values: Record<string, string | undefined> = {};
    this.controls.forEach((control, idx) => {
        const col = this.columnInfos[this.visibleColumns[idx]?.index ?? idx];
        // Skip hidden or readonly columns
        if (col?.hidden || col?.readonly) return;
        values[this.columnNames[this.visibleColumns[idx]?.index ?? idx]] = control.value;
    });
    return values;
}
```
Note: The exact implementation of `getControlValues` will depend on how `visibleColumns` maps back to `columnNames`. This needs careful alignment between the `@ViewChildren` query and the filtered list.

**File**: `ui/src/app/controls/composite-row-control/composite-row-control.component.html`

- [ ] Change the `@for` loop to iterate over `visibleColumns` instead of `columnInfos`:
```html
@for (entry of visibleColumns; track entry.col.column_name) {
<app-composite-control [dataInfo]="entry.col"
                       [value]="columnValues[entry.index]"
                       [readOnly]="isReadonlyColumn(entry.col)"
                       (valueChanged)="onValueChanged(entry.index, $event)">
</app-composite-control>
}
```

### 1.11 Update `table-view-control` — hide columns in table view

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.ts`

- [ ] Filter out hidden columns from the `columns` and `displayedColumns` arrays in `loadData()`:
```typescript
// After: this.columns = this.tableSchema.columns;
this.columns = this.tableSchema.columns.filter(c => !c.hidden);
this.displayedColumns = ['actions'].concat(this.columns.map(c => c.column_name));
```

No template changes needed — the template already iterates `columns`, so filtering the array is sufficient.

---

## Client Tests

### 1.12 Update `composite-row-control.component.spec.ts`

**File**: `ui/src/app/controls/composite-row-control/composite-row-control.component.spec.ts`

- [ ] **Test: hidden columns are not rendered in edit mode** — Set up a component with 3 columns where one has `hidden: true`. Verify only 2 `app-composite-control` elements are rendered.
- [ ] **Test: hidden columns are not rendered in create mode** — Same as above but with `isCreateMode = true`.
- [ ] **Test: readonly columns are shown as disabled in edit mode** — Set up a column with `readonly: true`, `isCreateMode = false`. Verify the `app-composite-control` has `readOnly = true`.
- [ ] **Test: readonly columns are hidden in create mode** — Set up a column with `readonly: true`, `isCreateMode = true`. Verify the column is not rendered.
- [ ] **Test: hidden column values are not included in submission** — Set up hidden columns, call `getControlValues()`, verify hidden columns are not in the result.

### 1.13 Update `table-view-control.component.spec.ts`

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.spec.ts`

- [ ] **Test: hidden columns are excluded from table headers** — Set up schema with a hidden column, verify `displayedColumns` doesn't include it.
- [ ] **Test: non-hidden columns are still displayed** — Verify other columns are unaffected.

### 1.14 Update `composite-control.component.spec.ts`

**File**: `ui/src/app/controls/composite-control/composite-control.component.spec.ts`

- [ ] Verify existing tests still pass (readonly is passed through to child controls, the composite-control itself doesn't need to know about hidden/readonly — that's handled at the row level)

---

# 2. Better Date Formatting

## Design

All timestamp columns in the database are stored as BIGINT microseconds since epoch. In the admin table view they currently render as raw numbers like `1708358400000000`. This section adds:

1. **Server**: Change `html_input_type` from `"text"` / `"number"` to `"date"` for all timestamp columns that have admin metadata.
2. **Client**: Add a `formatCellValue()` method to `table-view-control` that detects `html_input_type === "date"` columns and formats them.

### Formatting Rules
- **Recent (< 24 hours)**: Relative time — "just now", "5 minutes ago", "3 hours ago"
- **Older**: Absolute — "Feb 18, 2026 2:30 PM"

### Columns Affected (admin-visible tables only)

| Table | Column | Current `html_input_type` | New | Notes |
|-------|--------|---------------------------|-----|-------|
| people | created_at | `text` | `date` | readonly |
| people | updated_at | `text` | `date` | readonly |
| products | created_us | `text` | `date` | readonly |
| products | updated_us | `text` | `date` | readonly |
| price_schedules | valid_from_us | `number` | `date` | editable, required |
| price_schedules | valid_to_us | `number` | `date` | editable |
| price_schedules | created_us | `text` | `date` | readonly |
| price_schedules | updated_us | `text` | `date` | readonly |

Non-admin tables (purchases, payments, entitlements, etc.) don't have admin metadata and aren't shown in the admin UI, so they're out of scope.

---

## Server Implementation

### 2.1 Update `html_input_type` for timestamp columns

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`

Change the `html_input_type` parameter from `"text"` or `"number"` to `"date"` for each timestamp column's `AddRow` call in `PopulateAdminColumnDataInfo`:

- [ ] `kPeopleCreatedAt`: change `"text"` → `"date"`
- [ ] `kPeopleUpdatedAt`: change `"text"` → `"date"`
- [ ] `kPriceSchedulesValidFromUs`: change `"number"` → `"date"`
- [ ] `kPriceSchedulesValidToUs`: change `"number"` → `"date"`
- [ ] `kPriceSchedulesCreatedUs`: change `"text"` → `"date"`
- [ ] `kPriceSchedulesUpdatedUs`: change `"text"` → `"date"`
- [ ] `kProductsCreatedUs`: change `"text"` → `"date"`
- [ ] `kProductsUpdatedUs`: change `"text"` → `"date"`

### 2.2 Reset the database

- [ ] Run `knottyyoga_database_helper` to reset and initialize the database with the updated metadata

---

## Server Tests

### 2.3 Verify existing server tests still pass

No server test changes are expected — the existing `admin_column_data_info_test.cpp` and `database_rest_helper_test.cpp` tests use their own test metadata values (like `"htmlInputType1a"`) and are not affected by production data changes in `create_database.cpp`.

- [ ] Verify C++ tests still pass after the `create_database.cpp` changes

---

## Client Implementation

### 2.4 Create date formatting utility

**New file**: `ui/src/app/shared/utils/DateFormatting.ts`

Create a utility module with these functions:

- [ ] `formatMicroseconds(value: string): string` — Converts a microsecond BIGINT string to a human-readable date:
  - Parse: `new Date(Number(value) / 1000)` (microseconds → milliseconds)
  - If invalid date, return the raw value unchanged
  - If less than 1 minute ago: `"just now"`
  - If less than 1 hour ago: `"X minutes ago"`
  - If less than 24 hours ago: `"X hours ago"`
  - Otherwise: format as `"Feb 18, 2026 2:30 PM"` using `Intl.DateTimeFormat` with options:
    ```typescript
    { month: 'short', day: 'numeric', year: 'numeric', hour: 'numeric', minute: '2-digit', hour12: true }
    ```
- [ ] `isMicrosecondTimestamp(value: string): boolean` — Returns true if the value looks like a microsecond timestamp (all digits, length > 10)

### 2.5 Add `formatCellValue` to `table-view-control`

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.ts`

- [ ] Import `formatMicroseconds` from `DateFormatting.ts`
- [ ] Add method:
```typescript
formatCellValue(column: ColumnDataInfo, value: string): string {
    if (!value) return value;
    if (this.isTrueValue(column.hidden)) return '';
    const inputType = column.html_input_type;
    if (inputType === 'date' && isMicrosecondTimestamp(value)) {
        return formatMicroseconds(value);
    }
    return value;
}
```

### 2.6 Update table view template

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.html`

- [ ] Change raw value display to use formatting:
```html
<!-- Before -->
<td mat-cell *matCellDef="let row">{{ row[column.column_name] }}</td>

<!-- After -->
<td mat-cell *matCellDef="let row">{{ formatCellValue(column, row[column.column_name]) }}</td>
```

---

## Client Tests

### 2.7 Create date formatting utility tests

**New file**: `ui/src/app/shared/utils/DateFormatting.spec.ts`

- [ ] **Test: formatMicroseconds returns raw value for non-numeric input** — `formatMicroseconds("not-a-number")` returns `"not-a-number"`
- [ ] **Test: formatMicroseconds returns raw value for empty string** — `formatMicroseconds("")` returns `""`
- [ ] **Test: formatMicroseconds formats old date as absolute** — Use a timestamp from 2024 (well over 24 hours ago), verify output matches `"Mon DD, YYYY H:MM AM/PM"` pattern
- [ ] **Test: formatMicroseconds formats very recent as "just now"** — Use `Date.now() * 1000` (current time in microseconds), verify output is `"just now"`
- [ ] **Test: formatMicroseconds formats minutes ago** — Use `(Date.now() - 5 * 60 * 1000) * 1000` (5 minutes ago), verify output is `"5 minutes ago"`
- [ ] **Test: formatMicroseconds formats hours ago** — Use `(Date.now() - 3 * 3600 * 1000) * 1000` (3 hours ago), verify output is `"3 hours ago"`
- [ ] **Test: formatMicroseconds formats 1 minute ago as singular** — Verify `"1 minute ago"` not `"1 minutes ago"`
- [ ] **Test: formatMicroseconds formats 1 hour ago as singular** — Verify `"1 hour ago"` not `"1 hours ago"`
- [ ] **Test: isMicrosecondTimestamp returns true for large digit strings** — `isMicrosecondTimestamp("1708358400000000")` returns `true`
- [ ] **Test: isMicrosecondTimestamp returns false for short numbers** — `isMicrosecondTimestamp("42")` returns `false`
- [ ] **Test: isMicrosecondTimestamp returns false for non-numeric** — `isMicrosecondTimestamp("hello")` returns `false`
- [ ] **Test: isMicrosecondTimestamp returns false for ISO dates** — `isMicrosecondTimestamp("2026-02-18T14:30:00Z")` returns `false`

### 2.8 Update table-view-control tests

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.spec.ts`

- [ ] **Test: date columns display formatted values in table** — Set up schema with a column that has `html_input_type: 'date'` and a microsecond value. Verify the rendered `<td>` content is a formatted date, not the raw number.
- [ ] **Test: non-date columns display raw values unchanged** — Verify that columns without `html_input_type: 'date'` still show raw values.
- [ ] **Test: formatCellValue returns raw value for non-date column** — Call `component.formatCellValue(nonDateColumn, '12345')` and expect `'12345'`.
- [ ] **Test: formatCellValue returns formatted date for date column** — Call with a date column and microsecond value, expect a formatted string.

---

# 3. Date Editing

## Problem

The `simple-date` component exists and works, but timestamp columns stored as microseconds don't connect to it properly. The component expects ISO 8601 strings but the database values are microsecond integers. Additionally, `composite-control` doesn't pass `readOnly` to `simple-date`, so readonly timestamp columns would be editable.

## Design Decisions

- **Option B chosen**: Make `simple-date` smart enough to detect microsecond input and handle conversion internally, keeping `composite-control` simple.
- **Create mode date fields**: Default to empty (not "now").
- **Time precision**: Use `datetime` mode (date + time pickers), not date-only.
- **No server changes needed**: Section 2 already set `html_input_type = 'date'` for all timestamp columns.

## How It Works

1. `simple-date` receives a value string (could be ISO like `"2025-03-27T17:15:00Z"` or microseconds like `"1708358400000000"`)
2. On init, it detects the format via `isMicrosecondTimestamp()` (all digits, length > 10) and sets a private `isMicrosecondInput` flag
3. It converts the value to a `Date` object for the Angular Material date/time pickers
4. When the user changes the value, it emits back in the **same format** it received — microseconds if the input was microseconds, ISO otherwise
5. If `readOnly` is true, the FormControl is disabled (which automatically disables the Material date/time picker toggles)

---

## Client Implementation

### 3.1 Create date conversion utilities

**New file**: `ui/src/app/shared/utils/DateFormatting.ts`

- [ ] `isMicrosecondTimestamp(value: string): boolean` — returns true if value is all digits and length > 10
- [ ] `microsToDate(value: string): Date` — `new Date(Number(value) / 1000)`
- [ ] `dateToMicros(date: Date): string` — `String(date.getTime() * 1000)`

These will also be reused by Section 2's `formatMicroseconds()` display formatting.

### 3.2 Update `simple-date.component.ts`

**File**: `ui/src/app/controls/simple-date/simple-date.component.ts`

- [ ] Add `@Input() readOnly = false`
- [ ] Add private `isMicrosecondInput = false` flag to track input format
- [ ] Add `parseInputValue(value: string): Date | null` method:
  - Returns `null` for empty/falsy values (create mode stays empty)
  - Uses `isMicrosecondTimestamp()` to detect microsecond values
  - Sets `isMicrosecondInput` flag accordingly
  - Converts via `microsToDate()` or parses ISO via `new Date(value)`
- [ ] Update `ngOnInit()`:
  - Use `parseInputValue()` instead of raw `new Date(initialValue)`
  - Call `this.dateControl.disable()` when `readOnly` is true
  - In `valueChanges` subscription: emit `dateToMicros(date)` when `isMicrosecondInput`, otherwise emit `date.toISOString()` as before
- [ ] Update `ngOnChanges()`:
  - Use `parseInputValue()` when `value` input changes
  - Handle `readOnly` input changes: enable/disable FormControl accordingly

### 3.3 Pass `readOnly` to simple-date in composite-control template

**File**: `ui/src/app/controls/composite-control/composite-control.component.html`

- [ ] Add `[readOnly]="readOnly"` to the `app-simple-date` element

No changes needed in `simple-date.component.html` — disabling the FormControl automatically disables the Material date/time picker toggle buttons.

---

## Client Tests

### 3.4 Create `DateFormatting.spec.ts`

**New file**: `ui/src/app/shared/utils/DateFormatting.spec.ts`

- [ ] `isMicrosecondTimestamp` returns true for `"1708358400000000"` (16 digits)
- [ ] `isMicrosecondTimestamp` returns false for `"42"` (short number)
- [ ] `isMicrosecondTimestamp` returns false for `"hello"` (non-numeric)
- [ ] `isMicrosecondTimestamp` returns false for `"2026-02-18T14:30:00Z"` (ISO date string)
- [ ] `microsToDate` converts `"1708358400000000"` to correct Date
- [ ] `dateToMicros` converts Date back to correct microsecond string
- [ ] Round-trip: `dateToMicros(microsToDate(value))` returns original value

### 3.5 Update `simple-date.component.spec.ts`

**File**: `ui/src/app/controls/simple-date/simple-date.component.spec.ts`

- [ ] **Microsecond input parsing**: Set `value = "1708358400000000"`, verify the date picker shows the correct date (Feb 19, 2024)
- [ ] **Microsecond output**: Set microsecond input, simulate date change, verify `valueChange` emits a microsecond string (not ISO)
- [ ] **ISO input still works**: Existing tests cover this — verify they still pass
- [ ] **ReadOnly disables control**: Set `readOnly = true`, verify `dateControl.disabled` is true
- [ ] **ReadOnly prevents value emission**: Set `readOnly = true`, verify no `valueChange` events fire
- [ ] **Empty value in create mode**: Set `value = undefined`, verify picker is empty (no default date)
- [ ] **ngOnChanges with microsecond value**: Change `value` input to a microsecond string, verify picker updates correctly

---

## Verification

1. Run `ng test` from `ui/` — all tests pass including new ones
2. Manual browser testing:
   - Admin > Manage Data > products > edit a product: `created_us`/`updated_us` show disabled date/time pickers with correct dates
   - Admin > Manage Data > price_schedules > edit: `valid_from_us`/`valid_to_us` show editable date/time pickers with correct dates; change a value, save, verify correct microsecond storage
   - Admin > Manage Data > price_schedules > create new: `valid_from_us`/`valid_to_us` show empty pickers; `created_us`/`updated_us` are hidden (readonly + create mode)

## Files Summary

| File | Action |
|------|--------|
| `ui/src/app/shared/utils/DateFormatting.ts` | **Create** — microsecond detection and conversion |
| `ui/src/app/shared/utils/DateFormatting.spec.ts` | **Create** — utility tests |
| `ui/src/app/controls/simple-date/simple-date.component.ts` | **Modify** — readOnly, microsecond parsing/emission |
| `ui/src/app/controls/composite-control/composite-control.component.html` | **Modify** — pass readOnly to simple-date |
| `ui/src/app/controls/simple-date/simple-date.component.spec.ts` | **Modify** — microsecond and readOnly tests |

---

# 4. Enum Dropdown Mapping

## Design

### Problem
Several string columns represent a fixed set of values (enums) but are currently edited as free-text inputs. They should use a dropdown (`mat-select`) instead. Many enums are shared across tables (e.g., `currency` appears in payments, purchases, purchase_items, product_prices), so enums should be first-class entities independent from columns.

### Three-Table Architecture

Following Mason's design, enums are stored in three normalized tables:

**`admin_enums`** — Enum definitions
| Column | Type | Notes |
|--------|------|-------|
| `id` | `bigserial` PK | |
| `name` | `varchar` NOT NULL UNIQUE | e.g., "currency", "payment_status" |

**`admin_enum_values`** — Individual values for each enum
| Column | Type | Notes |
|--------|------|-------|
| `id` | `bigserial` PK | |
| `admin_enum_id` | `bigint` FK → `admin_enums.id` | Which enum this value belongs to |
| `name` | `varchar` NOT NULL | The string stored in the database, e.g., "USD", "COMPLETED" |
| `value` | `int` NOT NULL | Ordering integer (deterministic sort order for dropdown display) |

**`admin_column_enums`** — Binds an enum to a column
| Column | Type | Notes |
|--------|------|-------|
| `id` | `bigserial` PK | |
| `admin_enum_id` | `bigint` FK → `admin_enums.id` | Which enum to use |
| `admin_column_data_info_id` | `bigint` FK → `admin_column_data_info.column_data_info_id` | Which column uses this enum |

This design means:
- A single enum (like "currency") is defined once and bound to multiple columns
- Adding/removing enum values happens in one place and automatically applies everywhere
- The `value` integer provides deterministic ordering independent of primary key

### Enums Identified

| Enum Name | Values (ordered by `value`) | Columns Using It |
|-----------|---------------------------|------------------|
| `currency` | USD (0) | payments.currency, purchases.currency, purchase_items.currency, product_prices.currency |
| `payment_provider` | square (0), voucher (1), cash (2), comp (3) | payments.provider |
| `payment_status` | COMPLETED (0), PENDING (1), FAILED (2) | payments.status |
| `purchase_status` | pending (0), funded (1), partially_funded (2), cancelled (3), refunded (4) | purchases.status |
| `entitlement_status` | active (0), expired (1), revoked (2) | entitlements.status |
| `product_kind` | one_time (0), subscription (1) | products.kind |
| `validity_kind` | instant (0), days_from_activation (1) | product_entitlement_rules.validity_kind |
| `idempotency_status` | pending (0), completed (1), failed (2) | idempotency_keys.status |

### NOT Enums (free text fields)
- `entitlements.revoked_reason` — free text explanation
- `entitlement_assignments.removed_reason` — free text explanation
- `payments.refund_reason` — free text explanation
- `idempotency_keys.scope` — constructed string like "purchase_pay_card", not a fixed set

### How `GenerateColumnMetadata` Changes

Currently `GenerateColumnMetadata` only looks at `admin_column_data_info`. After this change, it will also:
1. Check `admin_column_enums` for a binding from the column's `column_data_info_id`
2. If a binding exists, set `html_input_type` to `"enum"` (overriding whatever is stored)
3. Look up the enum's values from `admin_enum_values` (ordered by `value`)
4. Add an `"enum_values"` array of strings to the column JSON

The JSON output for an enum column will look like:
```json
{
  "column_name": "currency",
  "html_input_type": "enum",
  "enum_values": ["USD"],
  ...
}
```

### Read-Only Enum Columns
Some enum columns shouldn't be directly editable by admins (managed by business logic). These are already handled by the existing `readonly` flag from Section 1. The relevant columns that have admin_column_data_info entries (products.kind) will not be made readonly — the admin should be able to set product kind when creating/editing products. Columns on tables not in admin_top_level_tables (purchases, payments, entitlements, etc.) don't have admin metadata and won't appear in the admin UI, so they're not a concern here.

---

## Server Implementation

### 4.1 Create `admin_enums` db_schema

**New file**: `server/knottyyoga_server/src/db_schema/admin_enums.h`
**New file**: `server/knottyyoga_server/src/db_schema/admin_enums.cpp`

- [ ] Define constants:
  - `kAdminEnumsTable = "admin_enums"`
  - `kAdminEnumsId = "id"` (bigserial PK)
  - `kAdminEnumsName = "name"` (varchar, NOT NULL, UNIQUE)
- [ ] Implement `MakeAdminEnumsTable(DatabaseInfo)`

### 4.2 Create `admin_enum_values` db_schema

**New file**: `server/knottyyoga_server/src/db_schema/admin_enum_values.h`
**New file**: `server/knottyyoga_server/src/db_schema/admin_enum_values.cpp`

- [ ] Define constants:
  - `kAdminEnumValuesTable = "admin_enum_values"`
  - `kAdminEnumValuesId = "id"` (bigserial PK)
  - `kAdminEnumValuesAdminEnumId = "admin_enum_id"` (bigint, FK → admin_enums.id)
  - `kAdminEnumValuesName = "name"` (varchar, NOT NULL)
  - `kAdminEnumValuesValue = "value"` (int, NOT NULL)
- [ ] Implement `MakeAdminEnumValuesTable(DatabaseInfo)`

### 4.3 Create `admin_column_enums` db_schema

**New file**: `server/knottyyoga_server/src/db_schema/admin_column_enums.h`
**New file**: `server/knottyyoga_server/src/db_schema/admin_column_enums.cpp`

- [ ] Define constants:
  - `kAdminColumnEnumsTable = "admin_column_enums"`
  - `kAdminColumnEnumsId = "id"` (bigserial PK)
  - `kAdminColumnEnumsAdminEnumId = "admin_enum_id"` (bigint, FK → admin_enums.id)
  - `kAdminColumnEnumsAdminColumnDataInfoId = "admin_column_data_info_id"` (bigint, FK → admin_column_data_info.column_data_info_id)
- [ ] Implement `MakeAdminColumnEnumsTable(DatabaseInfo)`

### 4.4 Update `db_schema/CMakeLists.txt`

**File**: `server/knottyyoga_server/src/db_schema/CMakeLists.txt`

- [ ] Add 6 new entries to `knotty_yoga_core`:
  ```
  admin_enums.h
  admin_enums.cpp
  admin_enum_values.h
  admin_enum_values.cpp
  admin_column_enums.h
  admin_column_enums.cpp
  ```

### 4.5 Create `admin_enums` table helper

**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_enums.h`
**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_enums.cpp`

- [ ] Class `AdminEnums` with:
  - `AddAdminEnum(transaction, name) → int64_t` — inserts enum, returns generated ID
  - `GetAdminEnums(transaction) → KeyValueTableArray` — list all enums
  - `GetAdminEnum(transaction, name, keyValueTable) → bool` — look up by name

### 4.6 Create `admin_enum_values` table helper

**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_enum_values.h`
**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_enum_values.cpp`

- [ ] Class `AdminEnumValues` with:
  - `AddAdminEnumValue(transaction, adminEnumId, name, value)` — insert a value
  - `GetAdminEnumValuesByEnumId(transaction, adminEnumId) → KeyValueTableArray` — get all values for an enum, ordered by `value`

### 4.7 Create `admin_column_enums` table helper

**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_column_enums.h`
**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_column_enums.cpp`

- [ ] Class `AdminColumnEnums` with:
  - `AddAdminColumnEnum(transaction, adminEnumId, adminColumnDataInfoId)` — bind an enum to a column
  - `GetAdminColumnEnumByColumnDataInfoId(transaction, columnDataInfoId, keyValueTable) → bool` — look up binding for a column
  - `GetAdminColumnEnums(transaction) → KeyValueTableArray` — list all bindings

### 4.8 Update `sql_util/table_helpers/CMakeLists.txt`

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/CMakeLists.txt`

- [ ] Add to `knotty_yoga_core`:
  ```
  admin_enums.h
  admin_enums.cpp
  admin_enum_values.h
  admin_enum_values.cpp
  admin_column_enums.h
  admin_column_enums.cpp
  ```
- [ ] Add to `knotty_yoga_tests`:
  ```
  admin_enums_test.cpp
  admin_enum_values_test.cpp
  admin_column_enums_test.cpp
  ```

### 4.9 Update `make_database_info.cpp`

**File**: `server/knottyyoga_server/src/db_schema/make_database_info.cpp`

- [ ] Add includes for the 3 new schema headers
- [ ] Add `MakeAdminEnumsTable(databaseInfo)` call (before admin_column_enums since it has FK deps)
- [ ] Add `MakeAdminEnumValuesTable(databaseInfo)` call
- [ ] Add `MakeAdminColumnEnumsTable(databaseInfo)` call

### 4.10 Update `create_database.cpp` — CreateTables

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`

- [ ] Add includes for the 3 new schema headers and table helper headers
- [ ] Add `CreateTable(DbSchema::kAdminEnumsTable)` in `CreateTables()` (after admin_column_data_info, before the non-admin tables)
- [ ] Add `CreateTable(DbSchema::kAdminEnumValuesTable)`
- [ ] Add `CreateTable(DbSchema::kAdminColumnEnumsTable)`

### 4.11 Update `create_database.cpp` — PopulateEnums

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`

- [ ] Create `PopulateAdminEnums()` function that:
  1. Creates each enum definition in `admin_enums`:
     - currency, payment_provider, payment_status, purchase_status, entitlement_status, product_kind, validity_kind, idempotency_status
  2. Creates each enum's values in `admin_enum_values` with ordering integers
  3. Binds enums to columns via `admin_column_enums` — this requires looking up the `column_data_info_id` for each column. For columns that already have `admin_column_data_info` entries (like `products.kind`), the ID is known from insertion order. For columns that don't have entries yet, new `admin_column_data_info` entries must be added first.

- [ ] Add the following tables to `PopulateAdminTopLevelTables()` so they can have `admin_column_data_info` entries (required for FK constraint). These tables will become admin-visible when child table support is added next:
  - `payments`, `purchases`, `purchase_items`, `product_prices`, `entitlements`, `product_entitlement_rules`, `idempotency_keys`

- [ ] Add new `admin_column_data_info` entries in `PopulateAdminColumnDataInfo()` for ALL enum columns that don't have entries yet. Currently only `products.kind` has an entry. New entries needed:
  - `payments.provider` — "Provider", "Payment provider", "text", "true"
  - `payments.status` — "Status", "Payment status", "text", "true"
  - `payments.currency` — "Currency", "Payment currency", "text", "true"
  - `purchases.status` — "Status", "Purchase status", "text", "true"
  - `purchases.currency` — "Currency", "Purchase currency", "text", "true"
  - `purchase_items.currency` — "Currency", "Item currency", "text", "true"
  - `product_prices.currency` — "Currency", "Price currency", "text", "true"
  - `entitlements.status` — "Status", "Entitlement status", "text", "true"
  - `product_entitlement_rules.validity_kind` — "Validity Kind", "How the entitlement activates", "text", "true"
  - `idempotency_keys.status` — "Status", "Idempotency key status", "text", "true"
  (The `html_input_type` will be overridden to `"enum"` by `GenerateColumnMetadata` once bindings exist)

- [ ] In `PopulateAdminEnums()`, create `admin_column_enums` bindings for ALL enum columns:
  - `products.kind` → product_kind
  - `payments.provider` → payment_provider
  - `payments.status` → payment_status
  - `payments.currency` → currency
  - `purchases.status` → purchase_status
  - `purchases.currency` → currency
  - `purchase_items.currency` → currency
  - `product_prices.currency` → currency
  - `entitlements.status` → entitlement_status
  - `product_entitlement_rules.validity_kind` → validity_kind
  - `idempotency_keys.status` → idempotency_status

- [ ] Call `PopulateAdminEnums()` in `PopulateTables()` (after `PopulateAdminColumnDataInfo`)

### 4.12 Update `GenerateColumnMetadata` in `database_rest_helper.cpp`

**File**: `server/knottyyoga_server/src/sql_util/json/database_rest_helper.cpp`

- [ ] Add includes for admin_enum headers and table helpers
- [ ] Add `AdminEnumValues` and `AdminColumnEnums` parameters to `GenerateColumnMetadata`, `GenerateColumnArrayMetadata`, `GenerateTableMetadata`, `GenerateTableArrayMetadata`, and `DatabaseMetadata`
- [ ] In `GenerateColumnMetadata`, after building the column JSON from admin_column_data_info:
  1. Get the `column_data_info_id` from the KeyValueTable (it's the PK of admin_column_data_info)
  2. Look up `admin_column_enums` by that ID to find if this column has an enum binding
  3. If yes: set `column["html_input_type"] = "enum"` and look up `admin_enum_values` by the enum ID, build a JSON array of value names (ordered by `value` int), and set `column["enum_values"] = array`
  4. If no: do nothing (leave html_input_type as-is)

### 4.13 Update `DatabaseMetadata` signature

**File**: `server/knottyyoga_server/src/sql_util/json/database_rest_helper.h`

- [ ] Add `AdminEnumValues` and `AdminColumnEnums` parameters to `DatabaseMetadata()`

### 4.14 Update endpoint that calls `DatabaseMetadata`

Find the endpoint that calls `DatabaseMetadata` and pass the new table helper instances.

- [ ] Identify the calling endpoint (likely in `endpoints/`)
- [ ] Construct `AdminEnumValues` and `AdminColumnEnums` instances and pass them

### 4.15 Reset the database

- [ ] Run `knottyyoga_database_helper` to create the new tables and populate enum data

---

## Server Tests

### 4.16 Create `admin_enums_test.cpp`

**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_enums_test.cpp`

- [ ] **Test: AddAdminEnumBasic** — Add two enums, verify via GetAdminEnums
- [ ] **Test: GetAdminEnumByName** — Add an enum, look up by name, verify found
- [ ] **Test: GetAdminEnumByNameNotFound** — Look up non-existent name, verify returns false

### 4.17 Create `admin_enum_values_test.cpp`

**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_enum_values_test.cpp`

- [ ] **Test: AddAdminEnumValueBasic** — Add enum, add 3 values, verify via GetAdminEnumValuesByEnumId
- [ ] **Test: GetAdminEnumValuesOrderedByValue** — Add values with non-sequential ordering ints, verify they come back sorted by `value`
- [ ] **Test: GetAdminEnumValuesForDifferentEnums** — Add values for 2 different enums, verify filtering by enum ID works

### 4.18 Create `admin_column_enums_test.cpp`

**New file**: `server/knottyyoga_server/src/sql_util/table_helpers/admin_column_enums_test.cpp`

- [ ] **Test: AddAdminColumnEnumBasic** — Create an admin_column_data_info entry, create an admin_enum, bind them, verify via GetAdminColumnEnumByColumnDataInfoId
- [ ] **Test: GetAdminColumnEnumNotFound** — Look up binding for a column that has no enum, verify returns false
- [ ] **Test: GetAdminColumnEnums** — Create multiple bindings, verify GetAdminColumnEnums returns all

### 4.19 Update `database_rest_helper_test.cpp`

**File**: `server/knottyyoga_server/src/sql_util/json/database_rest_helper_test.cpp`

- [ ] Update existing tests to pass the new parameters (empty table helpers — no enum bindings means no change in behavior)
- [ ] **Test: DatabaseMetadataWithEnumColumn** — Set up admin_enums + admin_enum_values + admin_column_enums binding for a test column. Call DatabaseMetadata, verify the column JSON has `html_input_type: "enum"` and `enum_values: ["value1", "value2"]`
- [ ] **Test: DatabaseMetadataWithoutEnumColumn** — Column without enum binding still has normal html_input_type, no enum_values field

---

## Client Implementation

### 4.20 Update `ColumnDataInfo` interface

**File**: `ui/src/app/shared/types/ColumnDataInfo.ts`

- [ ] Add `enum_values?: string[]` field

### 4.21 Update `Conversion.ts`

**File**: `ui/src/app/shared/utils/Conversion.ts`

- [ ] Add parsing for `enum_values`:
```typescript
function parseStringArray(val: unknown): string[] | undefined {
    if (!Array.isArray(val)) return undefined;
    return val.filter((v): v is string => typeof v === 'string');
}
// In the return object:
enum_values: parseStringArray(input['enum_values']),
```

### 4.22 Create `simple-enum` component

**New file**: `ui/src/app/controls/simple-enum/simple-enum.component.ts`
**New file**: `ui/src/app/controls/simple-enum/simple-enum.component.html`
**New file**: `ui/src/app/controls/simple-enum/simple-enum.component.scss`

- [ ] Component with inputs: `dataInfo?: ColumnDataInfo`, `value?: string`, `readOnly: boolean = false`
- [ ] Output: `valueChanged = new EventEmitter<string>()`
- [ ] Uses `FormControl` with `mat-select` and `mat-option` for each value in `dataInfo.enum_values`
- [ ] On init: set FormControl to the current `value`, disable if `readOnly`
- [ ] On value change: emit selected string
- [ ] ReadOnly mode: show the current value as plain text (not a disabled dropdown)
- [ ] Handle `ngOnChanges` for `value`, `dataInfo`, and `readOnly` changes
- [ ] Uses `MatSelectModule`, `MatFormFieldModule`, `ReactiveFormsModule`

### 4.23 Update `composite-control` to route enum

**File**: `ui/src/app/controls/composite-control/composite-control.component.ts`

- [ ] Import `SimpleEnumComponent`
- [ ] Add to `imports` array
- [ ] Add `'enum'` to the `activeComponent` type union
- [ ] Add `case 'enum':` to the switch in `determineActiveComponent()`

**File**: `ui/src/app/controls/composite-control/composite-control.component.html`

- [ ] Add `@case ('enum')` block with `<app-simple-enum>` element, passing `dataInfo`, `value`, `readOnly`, and `(valueChanged)`

### 4.24 Update table-view-control for enum display (optional enhancement)

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.ts`

- [ ] No changes needed — enum values are already readable strings like "one_time" or "COMPLETED". They'll display as plain text in the table, which is fine for now. We can add styling (badges/chips) later if desired.

---

## Client Tests

### 4.25 Create `simple-enum.component.spec.ts`

**New file**: `ui/src/app/controls/simple-enum/simple-enum.component.spec.ts`

- [ ] **Test: renders mat-select with enum values** — Set `dataInfo` with `enum_values: ['one_time', 'subscription']`, verify mat-options are rendered
- [ ] **Test: selects initial value** — Set `value = 'subscription'`, verify the select shows that value
- [ ] **Test: emits value on change** — Simulate selecting a new option, verify `valueChanged` emits the string
- [ ] **Test: readOnly shows plain text** — Set `readOnly = true`, verify mat-select is not rendered, text value is shown
- [ ] **Test: readOnly disables control** — Set `readOnly = true`, verify FormControl is disabled
- [ ] **Test: handles empty enum_values** — Set `dataInfo` with no `enum_values`, verify graceful fallback
- [ ] **Test: updates on value input change** — Change `value` input, verify select updates via ngOnChanges

### 4.26 Update `composite-control.component.spec.ts`

**File**: `ui/src/app/controls/composite-control/composite-control.component.spec.ts`

- [ ] **Test: 'enum' html_input_type routes to enum component** — Set `html_input_type: 'enum'`, verify `activeComponent` is `'enum'`

### 4.27 Update `Conversion.spec.ts` (if exists)

- [ ] **Test: ColumnDataInfoFromJSON parses enum_values array** — Input with `enum_values: ["a", "b"]`, verify output has string array
- [ ] **Test: ColumnDataInfoFromJSON handles missing enum_values** — Input without field, verify `undefined`

---

## Verification

1. Run C++ tests after server changes — all pass
2. Run `knottyyoga_database_helper` to reset database
3. Run `ng test` from `ui/` — all tests pass
4. Manual browser testing:
   - Admin > Manage Data > products > edit a product: `kind` shows as a dropdown with "one_time" and "subscription"
   - Admin > Manage Data > products > create new: `kind` shows as a dropdown
   - Selecting a value from the dropdown and saving works correctly

## Files Summary

### Server — New Files (12)
| File | Purpose |
|------|---------|
| `db_schema/admin_enums.h` | Enum definitions table schema |
| `db_schema/admin_enums.cpp` | Enum definitions table DDL |
| `db_schema/admin_enum_values.h` | Enum values table schema |
| `db_schema/admin_enum_values.cpp` | Enum values table DDL |
| `db_schema/admin_column_enums.h` | Column-enum binding table schema |
| `db_schema/admin_column_enums.cpp` | Column-enum binding table DDL |
| `sql_util/table_helpers/admin_enums.h` | Enum definitions CRUD |
| `sql_util/table_helpers/admin_enums.cpp` | Enum definitions CRUD impl |
| `sql_util/table_helpers/admin_enum_values.h` | Enum values CRUD |
| `sql_util/table_helpers/admin_enum_values.cpp` | Enum values CRUD impl |
| `sql_util/table_helpers/admin_column_enums.h` | Column-enum binding CRUD |
| `sql_util/table_helpers/admin_column_enums.cpp` | Column-enum binding CRUD impl |

### Server — Modified Files (6)
| File | Change |
|------|--------|
| `db_schema/CMakeLists.txt` | Add 6 new schema files |
| `db_schema/make_database_info.cpp` | Register 3 new tables |
| `sql_util/table_helpers/CMakeLists.txt` | Add 6 new helper files + 3 test files |
| `sql_util/json/database_rest_helper.h` | Add enum params to DatabaseMetadata |
| `sql_util/json/database_rest_helper.cpp` | Enum lookup + JSON array injection |
| `database_helper/create_database.cpp` | Create 3 tables, populate enums |

### Server — New Test Files (3)
| File | Tests |
|------|-------|
| `sql_util/table_helpers/admin_enums_test.cpp` | 3 tests |
| `sql_util/table_helpers/admin_enum_values_test.cpp` | 3 tests |
| `sql_util/table_helpers/admin_column_enums_test.cpp` | 3 tests |

### Server — Modified Test Files (1)
| File | Tests |
|------|-------|
| `sql_util/json/database_rest_helper_test.cpp` | 2 new tests + update existing |

### Client — New Files (4)
| File | Purpose |
|------|---------|
| `controls/simple-enum/simple-enum.component.ts` | Enum dropdown component |
| `controls/simple-enum/simple-enum.component.html` | Enum dropdown template |
| `controls/simple-enum/simple-enum.component.scss` | Enum dropdown styles |
| `controls/simple-enum/simple-enum.component.spec.ts` | 7 tests |

### Client — Modified Files (4)
| File | Change |
|------|--------|
| `shared/types/ColumnDataInfo.ts` | Add `enum_values?: string[]` |
| `shared/utils/Conversion.ts` | Parse enum_values array |
| `controls/composite-control/composite-control.component.ts` | Route 'enum' type |
| `controls/composite-control/composite-control.component.html` | Add enum case |

### Client — Modified Test Files (1-2)
| File | Tests |
|------|-------|
| `controls/composite-control/composite-control.component.spec.ts` | 1 new test |
| `shared/utils/Conversion.spec.ts` (if exists) | 2 new tests |

---

# 5. Better Bool Display and Editing

## Problem

Boolean fields display as `t` or `f` in the table view, which is not user-friendly. There are also several bugs in boolean handling:

1. **Table view**: Shows raw `t` or `f` from PostgreSQL — should show checkmark/X icons
2. **Routing bug**: Backend uses `html_input_type: "checkbox"` but the frontend switch in `determineActiveComponent()` only handles `"bool"`, so `is_active` fields on products and price_schedules currently render as text inputs instead of checkboxes
3. **Value parsing bug**: `simple-bool` checks `value === 'true'` but PostgreSQL sends `'t'`/`'f'` — so the checkbox won't reflect the actual database value
4. **No readOnly support**: `simple-bool` has no `readOnly` input, and `composite-control` doesn't pass readOnly to it
5. **No auto-detection**: Boolean columns without explicit `html_input_type` metadata fall through to text inputs

## Design Decisions

- **Table view**: Green `check_circle` icon for true, red `cancel` icon for false
- **ReadOnly mode**: Show plain "Yes" or "No" text (not a disabled checkbox)
- **Auto-detect**: Route `type === 'bool'` columns to the bool component even without `html_input_type` metadata
- **Standardize**: Change backend `"checkbox"` → `"bool"` for consistency

---

## Server Implementation

### 5.1 Standardize `html_input_type` for boolean columns

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`

- [ ] Change `kPriceSchedulesIsActive` entry: `"checkbox"` → `"bool"`
- [ ] Change `kProductsIsActive` entry: `"checkbox"` → `"bool"`

### 5.2 Reset the database

- [ ] Run `knottyyoga_database_helper` to reset and initialize with updated metadata

### 5.3 Verify existing server tests still pass

- [ ] No server test changes expected — tests use their own metadata values

---

## Client Implementation

### 5.4 Fix value parsing in `simple-bool.component.ts`

**File**: `ui/src/app/controls/simple-bool/simple-bool.component.ts`

- [ ] Add `@Input() readOnly = false`
- [ ] Add helper method `private isTrueValue(value: string | undefined): boolean` that checks for `'true'` and `'t'` (PostgreSQL sends `'t'`/`'f'`, mock sends `'true'`/`'false'`)
- [ ] Update `ngOnInit()` to use `isTrueValue()` instead of `value === 'true'`
- [ ] Update `ngOnChanges()` to use `isTrueValue()` instead of `value === 'true'` and `default_value === 'true'`
- [ ] In `ngOnInit()`, call `this.checkboxControl.disable()` when `readOnly` is true
- [ ] In `ngOnChanges()`, handle `readOnly` input changes (enable/disable)

### 5.5 Update `simple-bool.component.html` for readOnly display

**File**: `ui/src/app/controls/simple-bool/simple-bool.component.html`

- [ ] When `readOnly` is true, show "Yes" or "No" text instead of the checkbox:
```html
@if (readOnly) {
  <span class="readonly-bool">{{ checkboxControl.value ? 'Yes' : 'No' }}</span>
} @else {
  <mat-checkbox [formControl]="checkboxControl">{{ dataInfo?.label }}</mat-checkbox>
}
```

### 5.6 Pass `readOnly` to simple-bool in composite-control

**File**: `ui/src/app/controls/composite-control/composite-control.component.html`

- [ ] Add `[readOnly]="readOnly"` to the `app-simple-bool` element

### 5.7 Auto-detect bool columns in `composite-control.component.ts`

**File**: `ui/src/app/controls/composite-control/composite-control.component.ts`

- [ ] Add `case 'checkbox':` to the switch (falls through to `'bool'` for backwards compatibility)
- [ ] Add auto-detection before the switch: if `html_input_type` is unset but `type === 'bool'`, return `'bool'`

### 5.8 Add bool icon rendering to table-view-control

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.ts`

- [ ] Add `isBoolColumn(column: ColumnDataInfo): boolean` — returns true if `column.type === 'bool'` or `html_input_type === 'bool'` or `html_input_type === 'checkbox'`
- [ ] Add `getBoolValue(value: string): boolean` — returns true for `'t'`, `'true'`, `'1'`

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.html`

- [ ] Update the cell template to conditionally render mat-icon for bool columns:
```html
<td mat-cell *matCellDef="let row">
  @if (isBoolColumn(column)) {
    <mat-icon [class]="getBoolValue(row[column.column_name]) ? 'bool-true' : 'bool-false'">
      {{ getBoolValue(row[column.column_name]) ? 'check_circle' : 'cancel' }}
    </mat-icon>
  } @else {
    {{ formatCellValue(column, row[column.column_name]) }}
  }
</td>
```

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.scss`

- [ ] Add styles for bool icons:
```scss
.bool-true { color: #16a34a; }  // green
.bool-false { color: #dc2626; } // red
```

---

## Client Tests

### 5.9 Update `simple-bool.component.spec.ts`

**File**: `ui/src/app/controls/simple-bool/simple-bool.component.spec.ts`

- [ ] **Test: parses 't' as true** — Set `value = 't'`, verify `checkboxControl.value` is true
- [ ] **Test: parses 'f' as false** — Set `value = 'f'`, verify `checkboxControl.value` is false
- [ ] **Test: readOnly disables control** — Set `readOnly = true`, verify `checkboxControl.disabled` is true
- [ ] **Test: readOnly shows "Yes" text** — Set `readOnly = true`, `value = 'true'`, verify DOM contains "Yes"
- [ ] **Test: readOnly shows "No" text** — Set `readOnly = true`, `value = 'false'`, verify DOM contains "No"
- [ ] **Test: readOnly hides checkbox** — Set `readOnly = true`, verify no `mat-checkbox` in DOM

### 5.10 Update `table-view-control.component.spec.ts`

**File**: `ui/src/app/controls/table-view-control/table-view-control.component.spec.ts`

- [ ] **Test: bool columns show check_circle icon for true** — Schema with `type: 'bool'`, value `'t'`, verify `mat-icon` with `check_circle` and class `bool-true`
- [ ] **Test: bool columns show cancel icon for false** — Value `'f'`, verify `mat-icon` with `cancel` and class `bool-false`
- [ ] **Test: isBoolColumn returns true for bool type** — `type: 'bool'` returns true
- [ ] **Test: isBoolColumn returns false for non-bool type** — `type: 'VARCHAR'` returns false
- [ ] **Test: getBoolValue handles 't' and 'true'** — Both return true
- [ ] **Test: getBoolValue handles 'f' and 'false'** — Both return false

### 5.11 Update `composite-control.component.spec.ts`

**File**: `ui/src/app/controls/composite-control/composite-control.component.spec.ts`

- [ ] **Test: auto-detects bool column without html_input_type** — Set `type: 'bool'` with no `html_input_type`, verify `activeComponent` is `'bool'`
- [ ] **Test: 'checkbox' html_input_type routes to bool** — Set `html_input_type: 'checkbox'`, verify `activeComponent` is `'bool'`

---

## Verification

1. Run `ng test` from `ui/` — all tests pass
2. Manual browser testing:
   - Admin > Manage Data > products: `is_active` column shows green checkmark/red X in table
   - Edit a product: `is_active` shows as a checkbox (editable)
   - If a product has `readonly` bool columns, they show "Yes"/"No" text
   - Admin > Manage Data > price_schedules: same checkmark/X behavior for `is_active`

## Files Summary

| File | Action |
|------|--------|
| `server/.../database_helper/create_database.cpp` | **Modify** — change "checkbox" → "bool" (2 entries) |
| `ui/.../controls/simple-bool/simple-bool.component.ts` | **Modify** — readOnly, fix 't'/'f' parsing |
| `ui/.../controls/simple-bool/simple-bool.component.html` | **Modify** — readOnly Yes/No display |
| `ui/.../controls/simple-bool/simple-bool.component.spec.ts` | **Modify** — readOnly and 't'/'f' tests |
| `ui/.../controls/composite-control/composite-control.component.ts` | **Modify** — auto-detect bool, handle 'checkbox' |
| `ui/.../controls/composite-control/composite-control.component.html` | **Modify** — pass readOnly to simple-bool |
| `ui/.../controls/composite-control/composite-control.component.spec.ts` | **Modify** — auto-detect and checkbox routing tests |
| `ui/.../controls/table-view-control/table-view-control.component.ts` | **Modify** — isBoolColumn, getBoolValue helpers |
| `ui/.../controls/table-view-control/table-view-control.component.html` | **Modify** — conditional mat-icon for bools |
| `ui/.../controls/table-view-control/table-view-control.component.scss` | **Modify** — bool icon colors |
| `ui/.../controls/table-view-control/table-view-control.component.spec.ts` | **Modify** — bool icon and helper tests |

---

# Implementation Order

I'd suggest tackling these in this order, as later items build on earlier ones:

1. **Section 5: Bools** — Smallest change, good warmup. Auto-detect bools, fix table display.
2. **Section 1: Read-only fields** — Needed before date editing so auto-generated dates aren't accidentally editable.
3. **Section 2: Date formatting** — Improves table view readability. Requires backend metadata additions.
4. **Section 3: Date editing** — Builds on the metadata from Section 2 to make date fields properly editable.
5. **Section 4: Enum dropdowns** — Most complex (new component, backend schema change, metadata population).

Each section is designed to be independently implementable, but Sections 2 and 3 share the date metadata work and should be done together.

---

# Shared Infrastructure

Several sections need a `formatCellValue()` method on `table-view-control`. This single method would handle:
- **Dates**: Detect microsecond timestamps → format as readable date
- **Booleans**: Detect `t`/`f` → format as "Yes"/"No"
- **Enums**: Potentially display with styled badges (future)

This should be built as part of the first section we tackle and extended as we go.