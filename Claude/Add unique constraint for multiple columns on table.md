---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 1/26/2026
Version: 0.1
tags: 
---
# Overview

In sql_util/schema, there are a bunch of metadata classes to represent the database from tables to columns. In database_info.h, we currently support a AddColumnUnique that causes the TableInfo to add a ColumnInfo with unique_ set to true. We also have sql_util/database_access/database_metadata.h/cpp. This class can reverse engineer the metadata from the actual database by inspecting the Postgres system views to build this metadata versus db_schema where we declare the schema of the database.

Postgres supports a different UNIQUE constraint on a table that can list multiple columns and make sure that the collection of those columns on a given row is unique on the table. There can be multiple of these and they can be named of postgres will generate a name for you. 

Inside table_info.h, I would like to have TableUniqueValues struct with string name that can be an empty string and then a StringArray columns with the list of columns for a given UNIQUE table constraint. I would like to add AddUniqueConstraintHelper(const TableUniqueValues& tableUniqueValues). Then use a variadic template technique like src\sql_util\database_access\transaction.h to make a template <class... Args> TableUniqueValues BuildTableUniqueValues(Args&&... args) that makes a TableUniqueValues like BuildExecParams makes an ExecParamsHelper. Use this function to add template <class... Args> void AddNamedUniqueConstraint\(std::string_view name, Args&&... args\) and template <class... Args> void AddUniqueConstraint\(Args&&... args\) that internally uses BuildTableUniqueValues. Also, declare a using TableUniqueValuesArray = std::vector\<TableUniqueValues> and add to TableInfoImpl a TableUniqueValuesArray tableUniqueValuesArray_ to hold these and a const TableUniqueValuesArray& GetTableUniqueValues() const that returns this value. Add this accessor to both TableInfo and TableInfoImpl.

On DatabaseInfo add a BuildTableUniqueValues and a AddUniqueConstraintHelper\(std::string_view tableName, const TableUniqueValues& tableUniqueValues\) like TableInfo then add the AddNamedUniqueConstraint / AddUniqueConstraint like TableInfo but have them take a tableName like AddUniqueConstraintHelper and then call AddUniqueConstraintHelper. Internally in DatabaseInfoImpl, just add AddUniqueConstraintHelper that the wrapper class forwards the call along.

Inside src\sql_util\database_access\database_metadata.h/cpp, please add a TableUniqueValuesArray ListTableUniqueValuesArrayForTable\(Transaction& transaction, std::string_view tableName\) and construct the necessary SQL needed to generate this for me. Then modify the code for DatabaseInfoFromDatabase so that we reverse engineer this information for each table from the database.

For all of these changes add the appropriate testing to cover the positive test cases (function name with Basic appended) and the edge cases and error cases. Throw an exception for trying to add a TableUniqueValues with no columns and add test cases for this.

Please create a plan in this document under the Plan section for how to do all of this. I want to start with a plan and not making any changes (yet!).

# Plan

## Overview

This plan adds support for multi-column UNIQUE constraints to the schema metadata system. The implementation follows existing patterns in the codebase and consists of four main parts:

1. **TableInfo changes** - Add `TableUniqueValues` struct and storage/accessors
2. **DatabaseInfo changes** - Add forwarding methods to add constraints by table name
3. **DatabaseMetadata changes** - Reverse-engineer constraints from Postgres system views
4. **Tests** - Comprehensive test coverage for all new functionality

---

## Part 1: TableInfo Changes

### File: `sql_util/schema/table_info.h`

**Add struct and type aliases (before TableInfoImpl forward declaration):**

```cpp
struct TableUniqueValues {
    std::string name;        // Constraint name (can be empty for auto-generated)
    StringArray columns;     // List of column names in the constraint

    bool operator==(const TableUniqueValues& rhs) const {
        return name == rhs.name && columns == rhs.columns;
    }
};

using TableUniqueValuesArray = std::vector<TableUniqueValues>;
```

**Add to TableInfo class (public section):**

```cpp
// Unique constraint helpers
void AddUniqueConstraintHelper(const TableUniqueValues& tableUniqueValues);

template <class... Args>
void AddNamedUniqueConstraint(std::string_view name, Args&&... args) {
    AddUniqueConstraintHelper(BuildTableUniqueValues(name, std::forward<Args>(args)...));
}

template <class... Args>
void AddUniqueConstraint(Args&&... args) {
    AddUniqueConstraintHelper(BuildTableUniqueValues("", std::forward<Args>(args)...));
}

const TableUniqueValuesArray& GetTableUniqueValues() const;
```

**Add free function (in namespace DbSchema, after the class):**

```cpp
template <class... Args>
TableUniqueValues BuildTableUniqueValues(std::string_view name, Args&&... args) {
    static_assert((std::is_convertible_v<Args, std::string_view> && ...),
        "All arguments must be convertible to std::string_view");

    TableUniqueValues result;
    result.name = std::string(name);
    (result.columns.push_back(std::string(std::forward<Args>(args))), ...);
    return result;
}
```

### File: `sql_util/schema/table_info.cpp`

**Add to TableInfoImpl class:**

```cpp
// Private member
TableUniqueValuesArray tableUniqueValuesArray_;

// Public methods
void AddUniqueConstraintHelper(const TableUniqueValues& tableUniqueValues);
const TableUniqueValuesArray& GetTableUniqueValues() const;
```

**Implement TableInfoImpl::AddUniqueConstraintHelper:**
- Validate that `tableUniqueValues.columns` is not empty
- Throw `std::invalid_argument` with message: `"TableInfoImpl::AddUniqueConstraintHelper - unique constraint must have at least one column."`
- Add to `tableUniqueValuesArray_`

**Implement TableInfoImpl::GetTableUniqueValues:**
- Return `tableUniqueValuesArray_`

**Add forwarding methods to TableInfo:**
- `AddUniqueConstraintHelper` → forwards to `impl_->AddUniqueConstraintHelper`
- `GetTableUniqueValues` → forwards to `impl_->GetTableUniqueValues`

**Update operator== for TableInfo:**
- Add comparison of `GetTableUniqueValues()` arrays

---

## Part 2: DatabaseInfo Changes

### File: `sql_util/schema/database_info.h`

**Add to DatabaseInfo class (public section):**

```cpp
// Unique constraint helpers
void AddUniqueConstraintHelper(
    std::string_view tableName,
    const TableUniqueValues& tableUniqueValues);

template <class... Args>
void AddNamedUniqueConstraint(std::string_view tableName, std::string_view name, Args&&... args) {
    AddUniqueConstraintHelper(tableName, BuildTableUniqueValues(name, std::forward<Args>(args)...));
}

template <class... Args>
void AddUniqueConstraint(std::string_view tableName, Args&&... args) {
    AddUniqueConstraintHelper(tableName, BuildTableUniqueValues("", std::forward<Args>(args)...));
}
```

### File: `sql_util/schema/database_info.cpp`

**Add to DatabaseInfoImpl class:**

```cpp
void AddUniqueConstraintHelper(
    std::string_view tableName,
    const TableUniqueValues& tableUniqueValues);
```

**Implement DatabaseInfoImpl::AddUniqueConstraintHelper:**
- Get mutable reference to TableInfo via `GetTableInfo(tableName)`
- Call `tableInfo.AddUniqueConstraintHelper(tableUniqueValues)`

**Add forwarding method to DatabaseInfo:**
- `AddUniqueConstraintHelper` → forwards to `impl_->AddUniqueConstraintHelper`

---

## Part 3: DatabaseMetadata Changes

### File: `sql_util/database_access/database_metadata.h`

**Add function declaration:**

```cpp
// List all multi-column unique constraints for a table
TableUniqueValuesArray ListTableUniqueValuesArrayForTable(
    Transaction& transaction,
    std::string_view tableName);
```

**Note:** Need to add `#include "sql_util/schema/table_info.h"` and use `DbSchema::TableUniqueValuesArray` return type.

### File: `sql_util/database_access/database_metadata.cpp`

**Add SQL query constant:**

```cpp
constexpr std::string_view kListTableUniqueConstraintsSql = R"SQL(
SELECT
    tc.constraint_name,
    kcu.column_name,
    kcu.ordinal_position
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
    AND tc.table_name = kcu.table_name
WHERE tc.constraint_type = 'UNIQUE'
    AND tc.table_name = $1
    AND tc.table_schema = 'public'
ORDER BY tc.constraint_name, kcu.ordinal_position
)SQL";
```

**Implement ListTableUniqueValuesArrayForTable:**
1. Execute query with table name parameter
2. Group results by constraint_name
3. For each constraint, collect columns in ordinal order
4. Filter out single-column constraints (these are already handled by `ColumnInfo.unique_`)
5. Return array of `TableUniqueValues`

**Modify AddTableToDatabaseInfo (in anonymous namespace):**
- After adding all columns, call `ListTableUniqueValuesArrayForTable`
- For each returned constraint, call `databaseInfo.AddUniqueConstraintHelper`

---

## Part 4: Tests

### File: `sql_util/schema/table_info_test.cpp`

**Add tests:**

1. `BuildTableUniqueValuesBasic` - Verify struct is built correctly with name and columns
2. `BuildTableUniqueValuesEmptyName` - Verify empty name works
3. `AddUniqueConstraintHelperBasic` - Add constraint and verify via GetTableUniqueValues
4. `AddUniqueConstraintBasic` - Use variadic template, verify columns are captured
5. `AddNamedUniqueConstraintBasic` - Use variadic template with name
6. `AddUniqueConstraintNoColumns` - Verify exception thrown for empty columns
7. `GetTableUniqueValuesEmpty` - Verify empty array returned when no constraints
8. `GetTableUniqueValuesMultiple` - Add multiple constraints, verify all returned
9. `TableInfoEqualityWithUniqueConstraints` - Verify operator== considers constraints

### File: `sql_util/schema/database_info_test.cpp`

**Add tests:**

1. `AddUniqueConstraintHelperBasic` - Add via DatabaseInfo, verify on TableInfo
2. `AddUniqueConstraintBasic` - Use variadic template on DatabaseInfo
3. `AddNamedUniqueConstraintBasic` - Use variadic template with name on DatabaseInfo
4. `AddUniqueConstraintTableNotFound` - Verify exception when table doesn't exist

### File: `sql_util/database_access/database_metadata_test.cpp`

**Add helper function:**

```cpp
void MakeTableWithMultiColumnUnique(
    Transaction& transaction,
    TestDatabaseUtil& testDatabaseUtil) {
    auto databaseInfo = testDatabaseUtil.GetDatabaseInfo();
    constexpr std::string_view kProducts = "products";
    databaseInfo.AddTable(kProducts);
    databaseInfo.AddColumnPrimaryKey(kProducts, "product_id", DB_TYPE_SERIAL);
    databaseInfo.AddColumnSimple(kProducts, "sku", DB_TYPE_STRING);
    databaseInfo.AddColumnSimple(kProducts, "region", DB_TYPE_STRING);
    databaseInfo.AddColumnSimple(kProducts, "warehouse", DB_TYPE_STRING);
    databaseInfo.AddNamedUniqueConstraint(kProducts, "uq_sku_region", "sku", "region");
    databaseInfo.AddUniqueConstraint(kProducts, "region", "warehouse");
    DbOps::CreateTable(transaction, databaseInfo, kProducts);
}
```

**Add tests:**

1. `ListTableUniqueValuesArrayForTableBasic` - Create table with multi-column unique, verify returned
2. `ListTableUniqueValuesArrayForTableEmpty` - Table with no multi-column unique constraints
3. `ListTableUniqueValuesArrayForTableSingleColumnExcluded` - Verify single-column uniques are NOT returned (handled separately)
4. `ListTableUniqueValuesArrayForTableMultiple` - Multiple constraints on same table
5. `DatabaseInfoFromDatabaseWithUniqueConstraints` - Full round-trip: create table with constraints, reverse engineer, compare

---

## Implementation Order

1. **table_info.h/cpp** - Core struct and TableInfo methods
2. **table_info_test.cpp** - Tests for TableInfo (run to verify)
3. **database_info.h/cpp** - DatabaseInfo forwarding methods
4. **database_info_test.cpp** - Tests for DatabaseInfo (run to verify)
5. **database_metadata.h/cpp** - SQL query and reverse engineering
6. **database_metadata_test.cpp** - Integration tests (run to verify)

---

## SQL DDL Generation (Future Consideration)

The `DbOps::GenerateCreateTableSql` function in `db_and_table_operations.cpp` will need to be updated to emit the UNIQUE constraints. This should:
1. After column definitions, add constraint clauses
2. For named constraints: `CONSTRAINT {name} UNIQUE ({col1}, {col2}, ...)`
3. For unnamed constraints: `UNIQUE ({col1}, {col2}, ...)`

This may be done as part of this work or as a follow-up task.

Mason- Let's do this as part of this work for sure. Good catch!

---

## Edge Cases and Validation

- **Empty columns array**: Throw exception in `AddUniqueConstraintHelper`
- **Duplicate constraint names**: PostgreSQL handles this; no need to validate in code
- **Column doesn't exist on table**: PostgreSQL handles this at CREATE TABLE time
- **Single-column constraint via multi-column API**: Allowed, but redundant with `AddColumnUnique`