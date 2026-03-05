---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 2/24/2026
Version: 0.1
tags: 
---
# Overview

Please go into plan mode and use this document as your planning document. Do not create a plan in .claude/plans. Leave this Overview alone but use the sections below the overview to do your work. Do not ask for permission to modify this document. This IS your plan mode document.

My website features a four layered architecture with lower layers only dependent on those below them and never vice versa. The bottom layer being the SQL database definition and metadata (db_schema). The next level up being table helpers (sql_util/table_helpers) that are CRUD wrappers for the database tables as well as utility things like configuration, logging, network access, sending email, and so forth. The next level up being business logic that aggregates the database tables and utilities to do helpers that implement higher level business operations (/auth, /images, /payment). The top level is endpoints that are HTTP endpoints for the server that delegate mainly to the business logic layer and convert JSON and cookies into C++ types.

We have a data driven admin dashboard that currently supports doing CRUD operations on top level tables. See Dashboard cleanup work items.md and Adding support for images.md for some more context. Now I need to extend this to support nested tables. Nested tables are tables that are listed as supported (admin_editable_tables - something like admin_top_level_tables but for nested tables) and have foreign key reference to a "parent" table.

My thinking is that when editing an item in the admin console, there will be a Parent button for non root level items that navigates up to the edit row page for the parent of the current item. For parent items, there will be button / cards for the tables that have foreign key references to this. Clicking on any of those tables will bring up a row view for that table filtered to just contain the items with a foreign key to this parent item. For instance, if we were browsing the people table and clicked on edit for a given person, there would be a Payments button. If the user clicked on that, they would see items from the payments table where payer_person_id is equal to the id of the person we were editing via the foreign key relationship in the database schema. The Purchases table would function similarly.

My thinking is that composite-row-control and table-view-control on the client will take data structure that is a TableBindings. A table binding has a table name, a PrimaryKeyPair, and a map of from table names to PrimaryKeyPairs. A PrimaryKeyPair has a primaryKey and a primaryKeyValue. These will be null for top level tables and top level table items. If the table-view-control is called for a table with a valid stack of TableBindings, it will look through the schema for the table being edited and the top of the stack and pass any foreign key mappings to what is probably a new endpoint get_filtered_table_rows that posts JSON with 

```json
{
    table_name: string {table_name_value},
    column_name: string {column_to_sort_results_by},
    asc: bool {ascending or descending},
    page_size: int {page size for pagination},
    page: int {page number for pagination},
    filter_pairs:
    [
	    {
	        column_name: string {column name to filter with where clause}
	        column_value: string {column value to filter for matching}
	    }
    ]
}
```

Please use the current get_rows_by_column endpoint as a guide. Please add a function to database_crud_helpers.h/cpp/test.cpp that has a signature like:

```c++
    KeyValueTableArray GetRowsByValues(
        Transaction& transaction,
        DatabaseHelper databaseHelper,
        std::string_view tableName,
        std::string_view columnName,
        KeyValueTable lookupValues,
        bool asc,
        int pageSize,
        int page);
```

It will be a bit of a mashup of LookupRowByValues and GetRowsByColumn. Then use this function to implement the endpoint.

The table-view-control will pass the stack of TableBindings to the composite-row-control when the user clicks edit. Instead of using get_row to fetch the item, table-view-control will need to call a need endpoint get_row_by_values that posts JSON:

```
{
    table_name: string {table_name_value},
    filter_pairs:
    [
	    {
	        column_name: string {column name to filter with where clause}
	        column_value: string {column value to filter for matching}
	    }
    ]	
}
```

Even root level tables will use this but the only pair passed will be the primary key. This endpoint will just call the existing LookupRowByValues database_crud_helpers function.

Like the root level tables, the table-view-control will have buttons for tables listed in admin_editable_tables that list it as a parent. When clicking on those, it will create a TableBindings. The table name of the TableBindings will be the current table as will the pimary key and value. The map will be a copy of the map from the current top of the stack with the current table, primary key name, and value inserted. The new entry will be pushed to the top of the stack and then go to the new child. When clicking the parent button in either page, the stack will be popped and composite-row-control will be navigated to based on the previous table, primary key name, and value with the old stack.

This is my attempt to describe the problem and my thoughts on how to solve the issue. I'm open to feedback or other approaches. Please create open questions sections and present other options or things to workshop. I want to build a plan for implementation. I'd like the implementation plan to include tests as part of every stage. I'd like an iterative implementation with check boxes and phases. Star at the bottom of the layers on the server and then work your way up culminating in the client changes after the server.

# Resolved Design Decisions

## D1: Metadata table for nested tables
**Decision:** New `admin_nested_tables` table with just a `name` column (the child table name). FK relationships come from existing `ForeignKeyManager` schema metadata. Keep `admin_top_level_tables` separate for the dropdown. This gives explicit control over which tables are navigable without redundant parent mapping.

## D2: Tables with multiple FK relationships
**Decision:** `ForeignKeyManager.GetChildReferences(parentTable)` returns per-column entries. If a table has multiple FKs to the same parent, show separate navigation buttons disambiguated by the FK column's friendly name (e.g., "Purchases (Payer)" vs "Purchases (Purchased By)"). For the other direction: when editing a nested item, FK columns from the navigation context are readonly; other FK columns are editable as plain inputs (upgraded to autocomplete pickers in a future phase).

## D3: URL scheme for navigation stack
**Decision:** Hybrid approach — readable URL path for the current view + `ctx` query param for parent chain.

```
Root table view:      /admin/tables/people/view/10/0
Root table edit:      /admin/tables/people/edit/42
Nested table view:    /admin/tables/purchases/view/10/0?ctx=people:person_id:42
Nested table edit:    /admin/tables/purchases/edit/7?ctx=people:person_id:42
Deeply nested:        /admin/tables/payments/view/10/0?ctx=people:person_id:42,purchases:purchase_id:7
New nested item:      /admin/tables/purchases/new?ctx=people:person_id:42
```

The `ctx` query param uses comma-separated `table:pkColumn:pkValue` triples. This is human-readable, survives page refresh, is bookmarkable, and requires no Angular route changes (same route patterns, just an optional query param). Crow is not involved — all `/admin/...` routes are handled by Angular's client-side router.

## D4: FK columns in create mode
**Decision:** Pre-fill FK columns from navigation context. Show as readonly (visible but not editable). User sees the association for context.

## D5: FK autocomplete picker
**Decision:** Phase it as a follow-up. Core nested navigation ships first with plain text inputs for FK columns. FK autocomplete picker is a separate work item that benefits the entire admin dashboard (not just nested tables). Will be tracked as a future phase with design notes:
- Needs a "display column" metadata concept per table (e.g., `roles` → `name`, `people` → `first_name`)
- `ForeignKeyPickerComponent` using `mat-autocomplete`
- Small tables load all rows; large tables use search/filter

## D6: Filtered row count
**Decision:** `get_filtered_table_rows` includes `totalCount` reflecting the filtered result set. Uses the established `COUNT(*) OVER() AS _total_count` window function pattern (see `entitlement_assignments.cpp`, `purchases.cpp`, etc.) — a single SQL query returns both data rows and the total filtered count. `GetRowsByValues` takes an optional `int64_t* totalCount = nullptr` output parameter. No separate count function.

## D7: Sorting vs filtering
**Decision:** Sort column (ORDER BY) and filter pairs (WHERE clause) are separate concerns in the endpoint. `get_filtered_table_rows` combines both. The existing `GetRowsByColumn` only sorts, `LookupRowByValues` only filters — the new `GetRowsByValues` combines both with pagination.

## D8: Access control for nested tables
**Decision:** `GetAllowedTables` includes both `admin_top_level_tables` and `admin_nested_tables` when user is admin.

## D9: Unified endpoints for root and nested tables
**Decision:** Use `get_filtered_table_rows` for ALL table browsing (root and nested). Root tables pass empty `filter_pairs`. This eliminates conditional endpoint switching. Similarly, admin `composite-row-control` uses `get_row_by_values` for ALL row fetching. Keep existing `getRow` and `getRowsByColumn` endpoints for non-admin use but don't use them in admin flows.

## D10: Database migration
**Decision:** Not needed. Not deployed yet — always start from scratch until V1 is deployed.

## D11: Initial nested tables to include
**Decision:** All tables with FK relationships that are useful for admin navigation:
- `purchases` (FK: `payer_person_id` → `people`)
- `payments` (FK: `purchase_id` → `purchases`)
- `purchase_items` (FK: `purchase_id` → `purchases`)
- `purchase_payments` (FK: `purchase_id` → `purchases`)
- `role_assignments` (FK: `person_id` → `people`, `role_id` → `roles`)
- `entitlement_assignments` (FK: `person_id` → `people`, `entitlement_id` → `entitlements`)

Tables with multiple FKs work fine — the FK from navigation context is readonly, others are editable plain inputs (upgraded to pickers later).

---

# Implementation Plan

## Phase 1: Database Schema — `admin_nested_tables`

New table modeled after `admin_top_level_tables`: single `name` VARCHAR column as primary key.

**Files to create:**
- [x] `db_schema/admin_nested_tables.h` — constants `kAdminNestedTables`, `kAdminNestedTablesName`
- [x] `db_schema/admin_nested_tables.cpp` — `MakeAdminNestedTablesTable()` with single `name` column (NOT NULL, primary key)

**Files to modify:**
- [x] `db_schema/CMakeLists.txt` — add new .h and .cpp
- [x] `db_schema/make_database_info.cpp` — call `MakeAdminNestedTablesTable(databaseInfo)`
- [x] `database_helper/create_database.cpp`:
  - Add `CreateTable` call for `admin_nested_tables`
  - Add initial rows: `purchases`, `payments`, `purchase_items`, `purchase_payments`, `role_assignments`, `entitlement_assignments`

**Tests:** Validated by table helper tests in Phase 2.

## Phase 2: Table Helper — `AdminNestedTables`

Table helper class modeled after `AdminTopLevelTables`.

**Files to create:**
- [x] `sql_util/table_helpers/admin_nested_tables.h`:
  ```cpp
  class AdminNestedTables {
  public:
      AdminNestedTables(DatabaseHelper databaseHelper);
      StringArray GetAdminNestedTables(Transaction& transaction);
      void AddAdminNestedTable(Transaction& transaction, std::string_view tableName);
      void DeleteAdminNestedTable(Transaction& transaction, std::string_view tableName);
      bool IsNestedTable(Transaction& transaction, std::string_view tableName);
  };
  ```
- [x] `sql_util/table_helpers/admin_nested_tables.cpp`
- [x] `sql_util/table_helpers/admin_nested_tables_test.cpp` — tests for CRUD + IsNestedTable

**Files to modify:**
- [x] `sql_util/table_helpers/CMakeLists.txt` — add .h and .cpp
- [x] Test `CMakeLists.txt` — add test file

## Phase 3: Database CRUD — `GetRowsByValues`

Mashup of `LookupRowByValues` (WHERE from KeyValueTable) and `GetRowsByColumn` (ORDER BY + LIMIT/OFFSET). Uses the established `COUNT(*) OVER() AS _total_count` window function pattern (see `entitlement_assignments.cpp`, `purchases.cpp`, etc.) so a single SQL query returns both data rows and the total filtered count. `ExtractTotalCount` strips the `_total_count` column from results and writes it to the optional output parameter.

**Key design:** When `lookupValues` is empty, no WHERE clause is generated — equivalent to `GetRowsByColumn`. This allows unified use for both root and nested tables.

**Files to modify:**
- [x] `sql_util/database_access/database_crud_helpers.h` — add function declaration
- [x] `sql_util/database_access/database_crud_helpers_priv.h` — add `GenerateGetRowsByValuesSql` declaration
- [x] `sql_util/database_access/database_crud_helpers.cpp` — implementation:
  - `ExtractTotalCount` helper in anonymous namespace
  - `GenerateGetRowsByValuesSql` in `PrivateSql` namespace
  - `GetRowsByValues` public function
- [x] `sql_util/database_access/database_crud_helpers_test.cpp` — 8 tests:
  - `GetRowsByValuesSingleFilter` — single filter, sorted ascending
  - `GetRowsByValuesMultipleFilters` — AND logic with two filters
  - `GetRowsByValuesPagination` — page 0 vs page 1 differ, totalCount unchanged
  - `GetRowsByValuesSortDescending` — descending sort order
  - `GetRowsByValuesEmptyFilters` — no WHERE clause, returns all rows sorted
  - `GetRowsByValuesNullTotalCount` — nullptr works, `_total_count` stripped
  - `GetRowsByValuesTotalCountReflectsFilter` — filtered count vs total count
  - `GenerateGetRowsByValuesSqlBasic` — SQL generation with and without filters

## Phase 4: REST Helper — `GetRowsByValuesWithCount`

Single method following the `GetRowsByColumnWithCount` pattern. The CRUD function's `COUNT(*) OVER()` window function already provides the count in one query — the REST helper just passes `&totalCount` and wraps the result.

**Files to modify:**
- [x] `sql_util/json/database_rest_helper.h` — add `GetRowsByValuesWithCount` declaration
- [x] `sql_util/json/database_rest_helper.cpp` — implementation:
  - Calls `DbCrud::GetRowsByValues(..., &totalCount)` — single SQL query returns both rows and count
  - Wraps in `DataResultsWithCount` and returns JSON via `Json::JsonFromDataResultsWithCount`
- [x] `sql_util/json/database_rest_helper_test.cpp` — 3 tests:
  - `GetRowsByValuesWithCountFiltered` — filtered query, totalCount reflects filtered count
  - `GetRowsByValuesWithCountEmptyFilter` — empty filter returns all rows (like GetRowsByColumnWithCount)
  - `GetRowsByValuesWithCountPagination` — pagination with filtered results, totalCount unchanged across pages

## Phase 5: Endpoint — `POST /api/get_filtered_table_rows`

Replaces `get_rows_by_column` in admin flows. Used for ALL table browsing (root and nested).

**Request body:**
```json
{
    "table_name": "purchases",
    "column_name": "purchase_id",
    "asc": true,
    "page_size": 10,
    "page": 0,
    "filter_pairs": [
        { "column_name": "payer_person_id", "column_value": "42" }
    ]
}
```

**Response:** `DataResultsWithCount` JSON (same format as `get_rows_by_column`).

**Files to create:**
- [x] `endpoints/get_filtered_table_rows.h`
- [x] `endpoints/get_filtered_table_rows.cpp`:
  - Route registration via `RoutingBase` pattern
  - `IsTableAllowed` check
  - Parse JSON body, validate required fields
  - Build `KeyValueTable` from filter_pairs
  - Delegate to `DatabaseRESTHelper::GetRowsByValuesWithCount`
- [x] `endpoints/get_filtered_table_rows_test.cpp` — 5 tests:
  - `NoFilters` — root table browsing with pagination (page 0 and page 1)
  - `SingleFilter` — filtered rows with totalCount reflecting filter
  - `FilteredPagination` — second page of filtered results
  - `NotAllowed` — table not in allowed_tables returns 400
  - `EmptyBody` — empty POST body returns 400

**Files to modify:**
- [x] `endpoints/web_app.cpp` — add include + reference for linker
- [x] `endpoints/CMakeLists.txt` — add .h, .cpp, and test file

## Phase 6: Endpoint — `POST /api/get_row_by_values`

Used by admin for ALL single-row fetching (root and nested). Root tables pass just the primary key as a filter pair.

**Request body:**
```json
{
    "table_name": "purchases",
    "filter_pairs": [
        { "column_name": "purchase_id", "column_value": "7" }
    ]
}
```

**Response:** `DataResults` JSON (same format as `get_row`).

**Files to create:**
- [x] `endpoints/get_row_by_values.h`
- [x] `endpoints/get_row_by_values.cpp`:
  - Route registration via `RoutingBase` pattern at `POST /api/get_row_by_values`
  - `IsTableAllowed` check
  - Parse JSON body, validate `table_name` and `filter_pairs`
  - Build `KeyValueTable` from filter_pairs
  - Delegate to `DatabaseRESTHelper::GetRowByValues` (new method)
- [x] `endpoints/get_row_by_values_test.cpp` — 3 tests:
  - `FetchByPrimaryKey` — root table use case, lookup by primary key
  - `FetchByMultipleValues` — nested table use case, lookup by two columns
  - `NotAllowed` — table not in allowed_tables returns 400

**Files to modify:**
- [x] `sql_util/json/database_rest_helper.h` — add `GetRowByValues` declaration
- [x] `sql_util/json/database_rest_helper.cpp` — implement `GetRowByValues` (uses `DbCrud::LookupRowByValues` + `MakeDataResults`)
- [x] `sql_util/json/database_rest_helper_test.cpp` — add `GetRowByValuesBasic` test
- [x] `endpoints/web_app.cpp` — add include + reference for linker
- [x] `endpoints/CMakeLists.txt` — add .h, .cpp, and test file

## Phase 7: Access Control + Schema — nested tables support

**Files to modify:**
- [x] `endpoints/endpoint_auth_helper.cpp` — `GetAllowedTables`:
  - After fetching `admin_top_level_tables`, also fetch `admin_nested_tables`
  - Merge both arrays into the allowed set
- [x] `endpoints/endpoint_test_helper.h` — add `AddAdminNestedTable` declaration
- [x] `endpoints/endpoint_test_helper.cpp`:
  - Create `admin_nested_tables` table in constructor
  - Add `AddAdminNestedTable` helper method
- [x] `sql_util/json/database_rest_helper.h` — add `nestedTables` parameter to `DatabaseMetadata`
- [x] `sql_util/json/database_rest_helper.cpp`:
  - Add `GenerateNestedTablesArrayMetadata` helper
  - Add `nestedTables` parameter to `DatabaseMetadata`, output `nested_tables` array
- [x] `sql_util/json/database_rest_helper_test.cpp`:
  - Updated all 5 existing `DatabaseMetadata` test calls with empty nestedTables
  - Updated all 5 expected JSON constants with `"nested_tables": []`
- [x] `endpoints/db_schema.cpp` — fetch nested tables from `AdminNestedTables`, pass to `DatabaseMetadata`
- [x] `endpoints/db_schema_test.cpp`:
  - Updated existing test expected JSON with `"nested_tables": []`
  - Added `GetDbSchemaWithNestedTables` test — verifies `nested_tables: ["orders"]` when orders is a nested table

## Phase 8: Frontend — Types and ServerAccess

**Files to modify:**
- [x] `shared/types/ServerAccess.ts` — add interface methods:
  ```typescript
  getFilteredTableRows(
      tableName: string, columnName: string, ascending: boolean,
      pageSize: number, page: number, filterPairs: FilterPair[]
  ): Observable<DataResultsWithCount>;

  getRowByValues(
      tableName: string, filterPairs: FilterPair[]
  ): Observable<DataResults>;
  ```

**Files to create or modify:**
- [x] `shared/types/admin.types.ts` (or add to existing types file):
  ```typescript
  export interface FilterPair {
      column_name: string;
      column_value: string;
  }

  export interface TableBinding {
      tableName: string;
      primaryKeyName: string;
      primaryKeyValue: string;
  }
  ```
- [x] Extend `DatabaseSchema` in the relevant types file:
  ```typescript
  export interface DatabaseSchema {
      root_tables: string[];
      nested_tables: string[];  // NEW
      tables: TableSchema[];
  }
  ```
- [x] `shared/services/network/ServerAccessNetwork.ts` — implement:
  ```typescript
  getFilteredTableRows(...): Observable<DataResultsWithCount> {
      return this.http.post<DataResultsWithCount>('/api/get_filtered_table_rows',
          { table_name, column_name, asc: ascending, page_size: pageSize, page, filter_pairs: filterPairs },
          { withCredentials: true });
  }
  getRowByValues(...): Observable<DataResults> {
      return this.http.post<DataResults>('/api/get_row_by_values',
          { table_name: tableName, filter_pairs: filterPairs },
          { withCredentials: true });
  }
  ```
- [x] `shared/services/network/ServerAccess.mock.ts` — mock implementations with in-memory filtering
- [x] `shared/services/network/ServerAccess.ts` (proxy) — delegate new methods
- [x] `shared/services/network/ServerAccess.mock.spec.ts` — tests for new mock methods

## Phase 9: Frontend — Routing + `ctx` Query Param

**Utility functions** for `ctx` param serialization (could be a service or utility file):
```typescript
// Serialize: TableBinding[] → "people:person_id:42,purchases:purchase_id:7"
function serializeBindingStack(stack: TableBinding[]): string

// Deserialize: "people:person_id:42,purchases:purchase_id:7" → TableBinding[]
function parseBindingStack(ctx: string | null): TableBinding[]
```

**Files to modify:**
- [x] `pages/admin/table-view-page/table-view-page.component.ts` — extract `ctx` query param, parse into `TableBinding[]`, pass to `TableViewControlComponent`
- [x] `pages/admin/table-edit-page/table-edit-page.component.ts` — extract `ctx`, parse, pass to `CompositeRowControlComponent`
- [x] `pages/admin/table-new-page/table-new-page.component.ts` — extract `ctx`, parse, pass to `CompositeRowControlComponent`
- [x] No route definition changes needed — same route patterns, `ctx` is just a query param

**Files to create:**
- [x] `pages/admin/services/table-binding.utils.ts` — serialize/parse functions + helper to derive `FilterPair[]` from binding stack + schema FK info
- [x] `pages/admin/services/table-binding.utils.spec.ts` — 13 tests for serialize, parse, round-trip, and deriveFilterPairs

## Phase 10: Frontend — Table View Control updates

**Files to modify:**
- [x] `controls/table-view-control/table-view-control.component.ts`:
  - Add `@Input() bindingStack: TableBinding[] = []`
  - Switch from `getRowsByColumn` to `getFilteredTableRows` for ALL data fetching
  - Derive `filterPairs` from `bindingStack` + schema foreign key info
  - Add nested table navigation: query schema for child tables in `nested_tables`, show buttons
  - Add "Back to Parent" button when `bindingStack.length > 0`
  - Update `onEditRow()` to include `ctx` query param in navigation URL
  - Update `onNewItem()` to include `ctx` query param
  - Nested table button click: push current item to stack, navigate to child table view with `ctx`
- [x] `controls/table-view-control/table-view-control.component.html` — add parent button, nested table buttons
- [x] `controls/table-view-control/table-view-control.component.scss` — style new buttons
- [x] `controls/table-view-control/table-view-control.component.spec.ts` — tests for:
  - Root table view (no binding stack, empty filter pairs)
  - Nested table view (binding stack present, filter pairs derived)
  - Nested table buttons appear for tables with child references in `nested_tables`
  - Parent button appears/disappears based on stack
  - Navigation URLs include correct `ctx` param

## Phase 11: Frontend — Composite Row Control updates

**Files to modify:**
- [x] `controls/composite-row-control/composite-row-control.component.ts`:
  - Add `@Input() bindingStack: TableBinding[] = []`
  - Switch from `getRow` to `getRowByValues` for data fetching in edit mode
  - In create mode: derive FK pre-fill values from `bindingStack` + schema FK info
  - Make FK columns from navigation context readonly (set `readonly` flag on `ColumnDataInfo`)
  - Add "Back to Parent" button
  - Add nested table cards/buttons (same as table view — show child tables)
  - Nested table button click: push current item to stack, navigate to child table view
- [x] `controls/composite-row-control/composite-row-control.component.html` — parent button, nested table cards, readonly FK display
- [x] `controls/composite-row-control/composite-row-control.component.scss` — style new elements
- [x] `controls/composite-row-control/composite-row-control.component.spec.ts` — tests for:
  - Edit mode uses `getRowByValues` (root table: just PK pair)
  - Edit mode with binding stack uses `getRowByValues`
  - Create mode pre-fills FK columns as readonly
  - Nested table cards appear for child references
  - Parent button navigation

## Phase 12: Frontend — Admin Shell Integration

**Files to modify:**
- [x] `pages/admin/dashboard/admin.component.ts` — handle `nested_tables` from schema (no changes to dropdown — still only `root_tables`)
  - Already initializes `databaseSchema` with `nested_tables: []`; schema from server populates it. No code changes needed.
- [x] Verify table selector dropdown doesn't show nested tables (it uses `root_tables` already)
  - Confirmed: template line 10 iterates `databaseSchema.root_tables` only.
- [x] Verify that selecting a root table from dropdown clears any `ctx` query param
  - Confirmed: `router.navigate` on line 65 does not pass `queryParams` and no `queryParamsHandling` is set, so Angular drops `ctx` on root table selection.

---

# Future Phase: FK Autocomplete Picker

**Goal:** FK columns show a Material autocomplete picker with human-readable labels instead of raw ID inputs. Additionally, FK columns in the table view show the resolved display text instead of raw integer IDs.

## Open Questions

*No open questions.*

## Resolved Design Decisions

### D1: Display template metadata
**Decision:** New `admin_table_display_template` table with `table_name` (PK, string) and `display_template` (string). Template uses `{column_name}` syntax matching the existing C++ `FormatString` utility. Examples:
- `people` → `"{first_name} {last_name} - {email}"`
- `roles` → `"{name}"`
- `products` → `"{name}"`
- `classes` → `"{name}"`
- `purchases` → `"Purchase #{purchase_id}"`
- `payments` → `"Payment #{payment_id} - {status}"`

Tables without a display template fall back to showing the primary key value (current behavior).

### D2: Two uses for display templates
Display templates serve two purposes:
1. **FK Picker labels** — autocomplete dropdown shows resolved template text alongside PK value
2. **Table view FK cell display** — FK columns in the table view show the resolved text instead of raw IDs (e.g., "John Smith - john@email.com" instead of "42")

Both use the same metadata and resolution logic, just in different UI contexts.

### D3: Server-side vs client-side template resolution
**Decision:** Server-side. The server resolves display templates when returning data because:
- The server already has the `FormatString` utility
- Client-side resolution would require fetching all referenced tables upfront
- The search endpoint needs server-side resolution anyway
- A single endpoint can return `[{ pk: "42", display: "John Smith - john@email.com" }]`

### D4: Display template format
**Decision:** Use `{column_name}` placeholder template strings (e.g., `"{first_name} {last_name} - {email}"`) instead of a single display column name. This aligns with the existing C++ `FormatString` utility in `util/types.h` and supports tables where no single column is descriptive.

### D5: FK search strategy
**Decision:** Search across display template columns only. The search endpoint parses the display template to extract referenced column names, builds a WHERE clause with `ILIKE '%search_text%'` across those columns joined with OR. Falls back to searching the primary key column if no display template is configured. This keeps results relevant and performant.

### D6: Preload vs search threshold
**Decision:** Option B with configurable threshold. Small reference tables preload all options into the dropdown; large tables use search-as-you-type. The row count threshold is stored as a `config_secrets` entry (e.g., key `fk_picker_preload_threshold`, default value `50`) rather than hard-coded. The client reads this from the schema response or a config endpoint. The `get_fk_options` endpoint returns a `total_count` so the client can compare against the threshold on the initial empty-search call.

## Implementation Plan

### Phase 13: Database Schema — `admin_table_display_template`

New metadata table mapping table names to display template strings.

**Files to create:**
- [x] `db_schema/admin_table_display_template.h` — constants `kAdminTableDisplayTemplateTable`, `kAdminTableDisplayTemplateName`, `kAdminTableDisplayTemplateTemplate`
- [x] `db_schema/admin_table_display_template.cpp` — `MakeAdminTableDisplayTemplateTable()` with `name` (PK, string) and `display_template` (string)

**Files to modify:**
- [x] `db_schema/CMakeLists.txt` — add new .h and .cpp
- [x] `db_schema/make_database_info.cpp` — call `MakeAdminTableDisplayTemplateTable(databaseInfo)`
- [x] `database_helper/create_database.cpp`:
  - Add `CreateTable` call
  - Add initial rows: `people` → `"{first_name} {last_name} - {email}"`, `roles` → `"{name}"`, `products` → `"{name}"`, `classes` → `"{name}"`, `purchases` → `"Purchase #{purchase_id}"`, `payments` → `"Payment #{payment_id} - {status}"`, `price_schedules` → `"{name}"`
  - Add `config_secrets` entry: `fk_picker_preload_threshold` → `"50"` (configurable row count threshold for preload vs search behavior)

**Tests:** Validated by table helper tests in Phase 14.

### Phase 14: Table Helper — `AdminTableDisplayTemplate`

Table helper for CRUD on display template metadata.

**Files to create:**
- [x] `sql_util/table_helpers/admin_table_display_template.h`:
  ```cpp
  class AdminTableDisplayTemplate {
  public:
      AdminTableDisplayTemplate(DatabaseHelper databaseHelper);
      std::string GetDisplayTemplate(Transaction& transaction, std::string_view tableName);
      void SetDisplayTemplate(Transaction& transaction, std::string_view tableName, std::string_view displayTemplate);
      void DeleteDisplayTemplate(Transaction& transaction, std::string_view tableName);
      KeyValueTable GetAllDisplayTemplates(Transaction& transaction);
  };
  ```
- [x] `sql_util/table_helpers/admin_table_display_template.cpp`
- [x] `sql_util/table_helpers/admin_table_display_template_test.cpp` — 5 tests: set/get, missing returns empty, getAll, getAll empty, delete

**Files to modify:**
- [x] `sql_util/table_helpers/CMakeLists.txt` — add .h and .cpp
- [x] Test `CMakeLists.txt` — add test file

### Phase 15: CRUD Helper — `ResolveDisplayValues`

Utility function that takes a table name, a list of rows (as `KeyValueTableArray`), and a display template string, and returns a map from primary key value → resolved display text.

**Files to modify:**
- [x] `sql_util/database_access/database_crud_helpers.h` — add declaration:
  ```cpp
  KeyValueTable ResolveDisplayValues(
      const KeyValueTableArray& rows,
      std::string_view primaryKeyColumn,
      std::string_view displayTemplate);
  ```
- [x] `sql_util/database_access/database_crud_helpers.cpp` — implementation using `FormatString`
- [x] `sql_util/database_access/database_crud_helpers_test.cpp` — 4 tests:
  - `ResolveDisplayValuesSingleRow`
  - `ResolveDisplayValuesMultipleRows`
  - `ResolveDisplayValuesMultiplePlaceholders`
  - `ResolveDisplayValuesEmptyTemplateReturnsPrimaryKey`

### Phase 16: Endpoint — `POST /api/get_fk_options`

Search/lookup endpoint for FK picker values. Returns matching rows from a referenced table with resolved display text.

**Request body:**
```json
{
    "table_name": "people",
    "search_text": "john",
    "page_size": 20
}
```

**Response:**
```json
{
    "total_count": 150,
    "options": [
        { "value": "42", "display": "John Smith - john@email.com" },
        { "value": "87", "display": "John Doe - johnd@example.com" }
    ]
}
```
`total_count` is the total matching rows (before `page_size` limit). The client compares this against the `fk_picker_preload_threshold` config secret to decide whether to preload all options or switch to search-as-you-type.

**Behavior:**
- Looks up display template for the requested table
- If `search_text` is empty, returns first `page_size` rows (for small table preload)
- If `search_text` is non-empty, parses template for column names, builds `ILIKE` WHERE clause across those columns
- Returns rows sorted by display text
- `IsTableAllowed` check for security

**Files to create:**
- [x] `endpoints/get_fk_options.h`
- [x] `endpoints/get_fk_options.cpp`
- [x] `endpoints/get_fk_options_test.cpp` — tests:
  - Empty search returns all rows (small table)
  - Search text filters by display template columns
  - Results include resolved display text
  - Table not allowed returns 400
  - No body returns error
  - Page size limits results
  - No display template falls back to primary key

**Files to modify:**
- [x] `endpoints/web_app.cpp` — add include + reference
- [x] `endpoints/CMakeLists.txt` — add files

### Phase 17: Schema Export — Display Templates

Include display templates in the `/api/get_db_schema` response so the client knows which tables have templates and can use them for table view cell display.

**Response addition:**
```json
{
    "root_tables": [...],
    "nested_tables": [...],
    "tables": [...],
    "display_templates": {
        "people": "{first_name} {last_name} - {email}",
        "roles": "{name}",
        ...
    },
    "fk_picker_preload_threshold": 50
}
```

**Files to modify:**
- [x] `sql_util/json/database_rest_helper.h` — add `displayTemplates` parameter to `DatabaseMetadata`
- [x] `sql_util/json/database_rest_helper.cpp` — output `display_templates` object in metadata
- [x] `sql_util/json/database_rest_helper_test.cpp` — update existing tests + add `DatabaseMetadataWithDisplayTemplates` test
- [x] `endpoints/db_schema.cpp` — fetch display templates and `fk_picker_preload_threshold` from secrets, pass to `DatabaseMetadata`
- [x] `endpoints/db_schema_test.cpp` — update existing tests + add `GetDbSchemaWithDisplayTemplates` test

**Frontend types:**
- [x] `shared/types/DatabaseSchema.ts` — add `display_templates: Record<string, string>` and `fk_picker_preload_threshold: number`
- [x] All test spec files and mock updated with new required fields

### Phase 18: Endpoint — `POST /api/resolve_fk_display`

Batch-resolve FK values for table view display. Given a list of FK column values and a parent table name, returns resolved display text for each value. This avoids N+1 queries when rendering a table with FK columns.

**Request body:**
```json
{
    "parent_table_name": "people",
    "values": ["42", "87", "15"]
}
```

**Response:**
```json
{
    "resolved": {
        "42": "John Smith - john@email.com",
        "87": "Jane Doe - jane@example.com",
        "15": "Bob Wilson - bob@example.com"
    }
}
```

**Files to create:**
- [x] `endpoints/resolve_fk_display.h`
- [x] `endpoints/resolve_fk_display.cpp`
- [x] `endpoints/resolve_fk_display_test.cpp`

**Files to modify:**
- [x] `endpoints/web_app.cpp` — add include + reference
- [x] `endpoints/CMakeLists.txt` — add files

**Additional work done:**
- [x] `database_crud_helpers.h/.cpp` — added `LookupRowsByPrimaryKeyValues` (batch WHERE pk IN query)
- [x] `database_crud_helpers_test.cpp` — 3 tests for `LookupRowsByPrimaryKeyValues`
- [x] `database_rest_helper.h/.cpp` — added `ResolveFkDisplay` method
- [x] `database_rest_helper_test.cpp` — 5 tests for `ResolveFkDisplay`

### Phase 19: Frontend — ServerAccess + Types

**Files to modify:**
- [x] `shared/types/ServerAccess.ts` — add methods:
  ```typescript
  getFkOptions(tableName: string, searchText: string, pageSize: number): Observable<FkOptionsResponse>;
  resolveFkDisplay(parentTableName: string, values: string[]): Observable<FkDisplayResponse>;
  ```
- [x] `shared/types/admin.types.ts` — add types:
  ```typescript
  export interface FkOption { value: string; display: string; }
  export interface FkOptionsResponse { options: FkOption[]; }
  export interface FkDisplayResponse { resolved: Record<string, string>; }
  ```
- [x] `shared/services/network/ServerAccessNetwork.ts` — HTTP implementations
- [x] `shared/services/network/ServerAccess.mock.ts` — mock implementations
- [x] `shared/services/network/ServerAccess.ts` — proxy delegation
- [x] `shared/services/network/ServerAccess.mock.spec.ts` — tests (10 tests: 5 for getFkOptions, 5 for resolveFkDisplay)

### Phase 20: Frontend — `ForeignKeyPickerComponent`

New control component using `mat-autocomplete` for FK column editing.

**Behavior:**
- On init: if parent table has < 50 rows, preload all options; otherwise start in search mode
- On type: debounce 300ms, call `getFkOptions` with search text
- Dropdown shows display text, stores PK value
- Readonly mode: shows resolved display text (not the raw ID)

**Files to create:**
- [x] `controls/fk-picker/fk-picker.component.ts`
- [x] `controls/fk-picker/fk-picker.component.html`
- [x] `controls/fk-picker/fk-picker.component.scss`
- [x] `controls/fk-picker/fk-picker.component.spec.ts` (11 tests)

**Template sketch:**
```html
<mat-form-field>
  <mat-label>{{ dataInfo?.label }}</mat-label>
  <input matInput [formControl]="searchControl" [matAutocomplete]="auto">
  <mat-autocomplete #auto="matAutocomplete"
                    [displayWith]="displayFn"
                    (optionSelected)="onOptionSelected($event)">
    @for (option of filteredOptions; track option.value) {
      <mat-option [value]="option">{{ option.display }}</mat-option>
    }
  </mat-autocomplete>
</mat-form-field>
```

### Phase 21: Frontend — Integration

Wire the FK picker and display resolution into existing controls.

**Files to modify:**
- [x] `controls/composite-control/composite-control.component.ts` — detect FK columns (check if column appears in `foreign_keys`), render `ForeignKeyPickerComponent` instead of text input
- [x] `controls/composite-control/composite-control.component.html` — add conditional for FK picker
- [x] `controls/table-view-control/table-view-control.component.ts`:
  - After loading table data, identify FK columns from schema
  - Call `resolveFkDisplay` for each FK column's unique values
  - Store resolved display text in a lookup map
  - Update `formatCellValue` to use resolved text for FK columns
- [x] `controls/table-view-control/table-view-control.component.html` — no template changes needed (formatCellValue handles it)
- [x] Update spec files for both components
- [x] `controls/composite-row-control/composite-row-control.component.ts` — added `getForeignKeyInfo()` method
- [x] `controls/composite-row-control/composite-row-control.component.html` — pass `[foreignKeyInfo]` to composite-control

---

# Additional Notes

**Security:** `get_filtered_table_rows` accepts column names and values from the client. Must follow the existing pattern: validate column names against schema metadata (`LookupDatabaseColumnInfoByColumnName`), use parameterized queries (`ExecParamsHelper`). No raw string concatenation.

**Existing endpoints preserved:** `getRow` and `getRowsByColumn` remain functional for non-admin use. Admin flows exclusively use `getRowByValues` and `getFilteredTableRows`.

**No database migration:** Since nothing is deployed yet, `create_database.cpp` is the source of truth. No ALTER TABLE scripts needed.