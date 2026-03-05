---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 1/27/2026
Version: 0.1
tags: 
---
# Overview

Inside the document:

Payment Design Document.md

We have a section for: Add base tables: products, price_schedules, product_prices, product_entitlement_rules

I'd like to put together a plan here for the work that needs to be done. Please use the Payment Design Document and code base for reference and let's put together a plan to add these tables to the server tree under src/db_schema. Please match the formatting and conventions of the existing files in that directory. Please look at the file src/database_helper/create_database.cpp. We need to add the include for each new table, an entry for CreateTables, a Populate{TableName} function for each table, and eadd each such call to PopulateTables. Let's add dummy entries for now for a $9 intro workshop and a $160 massage. Let's have the price schedule start 1/1/2026 and have null for the end. Both of these are single seat entitlements and USD for currency.

For now, let's add the following tables: products, price_schedules, product_prices, product_entitlement_rules, purchases, purchase_items, payments, purchase_payments, entitlements, entitlement_assignments.

For now, I'm not going to deal with fixed sized strings so just use DB_TYPE_STRING for everything. Please note that many of the tables say Unique constraint with a list of columns. We support this with DatabaseInfo::AddUniqueConstraint.

Please do this in planning mode and update this document for building your plan in planning mode. Do not modify this overview section but please modify / replace the section after the overview. I repeat, generate a plan in this file first. We will iterate on this document until I'm happy and then I'll tell you explicitly to execute the plan when I feel ready.

# Steps
- [x] Create `db_schema/products.h` and `db_schema/products.cpp` ✅ 2026-01-28
- [x] Create `db_schema/price_schedules.h` and `db_schema/price_schedules.cpp` ✅ 2026-01-28
- [x] Create `db_schema/product_prices.h` and `db_schema/product_prices.cpp` ✅ 2026-01-28
- [x] Create `db_schema/product_entitlement_rules.h` and `db_schema/product_entitlement_rules.cpp` ✅ 2026-01-28
- [x] Create `db_schema/purchases.h` and `db_schema/purchases.cpp` ✅ 2026-01-28
- [x] Create `db_schema/purchase_items.h` and `db_schema/purchase_items.cpp` ✅ 2026-01-28
- [x] Create `db_schema/payments.h` and `db_schema/payments.cpp` ✅ 2026-01-28
- [x] Create `db_schema/purchase_payments.h` and `db_schema/purchase_payments.cpp` ✅ 2026-01-28
- [x] Create `db_schema/entitlements.h` and `db_schema/entitlements.cpp` ✅ 2026-01-28
- [x] Create `db_schema/entitlement_assignments.h` and `db_schema/entitlement_assignments.cpp` ✅ 2026-01-28
- [x] Update `db_schema/CMakeLists.txt` to include all 20 new files ✅ 2026-01-28
- [x] Update `db_schema/make_database_info.cpp` with includes and Make*Table() calls ✅ 2026-01-28
- [x] Update `database_helper/create_database.cpp` with includes, CreateTable calls, Populate functions, and PopulateTables calls ✅ 2026-01-28

---

# Tables to Create

## 1. products
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| code | STRING | UNIQUE |
| name | STRING | NOT NULL |
| description | STRING | NOT NULL |
| kind | STRING | NOT NULL ("one_time" or "subscription") |
| is_active | BOOL | NOT NULL, DEFAULT FALSE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

## 2. price_schedules
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| name | STRING | NOT NULL |
| valid_from_us | BIGINT | NOT NULL |
| valid_to_us | BIGINT | NULLABLE |
| is_active | BOOL | NOT NULL, DEFAULT FALSE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

## 3. product_prices
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| product_id | BIGINT | FK → products |
| price_schedule_id | BIGINT | FK → price_schedules |
| permission_id | BIGINT | FK → permissions, NULLABLE |
| currency | STRING | NOT NULL, DEFAULT 'USD' |
| amount_cents | BIGINT | NOT NULL |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

**Unique Constraint**: (product_id, price_schedule_id, permission_id)

## 4. product_entitlement_rules
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| product_id | BIGINT | FK → products, UNIQUE |
| grants_permission_id | BIGINT | FK → permissions, NULLABLE |
| seats_default | BIGINT | NOT NULL, DEFAULT 1 |
| validity_kind | STRING | NOT NULL |
| validity_days | BIGINT | NULLABLE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

## 5. purchases
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| payer_person_id | BIGINT | FK → people |
| status | STRING | NOT NULL |
| currency | STRING | NOT NULL |
| subtotal_cents | BIGINT | NOT NULL |
| tax_cents | BIGINT | NOT NULL, DEFAULT 0 |
| total_cents | BIGINT | NOT NULL |
| paid_cents | BIGINT | NOT NULL, DEFAULT 0 |
| portal_note | STRING | NULLABLE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

## 6. purchase_items
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| purchase_id | BIGINT | FK → purchases |
| product_id | BIGINT | FK → products |
| quantity | BIGINT | NOT NULL |
| currency | STRING | NOT NULL |
| unit_price_cents | BIGINT | NOT NULL |
| line_total_cents | BIGINT | NOT NULL |
| price_schedule_id | BIGINT | FK → price_schedules |
| pricing_permission_id | BIGINT | FK → permissions, NULLABLE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |

## 7. payments
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| provider | STRING | NOT NULL |
| provider_payment_id | STRING | NOT NULL |
| status | STRING | NOT NULL |
| currency | STRING | NOT NULL |
| amount_cents | BIGINT | NOT NULL |
| covered_amount_cents | BIGINT | NOT NULL, DEFAULT 0 |
| payer_person_id | BIGINT | FK → people |
| portal_note | STRING | NULLABLE |
| refund_for_payment_id | BIGINT | FK → payments, NULLABLE |
| refund_reason | STRING | NULLABLE |
| raw_provider_json | STRING | NULLABLE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

**Unique Constraint**: (provider, provider_payment_id)

## 8. purchase_payments
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| purchase_id | BIGINT | FK → purchases |
| payment_id | BIGINT | FK → payments |
| applied_cents | BIGINT | NOT NULL |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |

**Unique Constraint**: (purchase_id, payment_id)

## 9. entitlements
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| purchase_id | BIGINT | FK → purchases |
| purchase_item_id | BIGINT | FK → purchase_items, NULLABLE |
| product_id | BIGINT | FK → products |
| valid_from_us | BIGINT | NOT NULL |
| valid_to_us | BIGINT | NOT NULL |
| seats_total | BIGINT | NOT NULL |
| status | STRING | NOT NULL |
| revoked_us | BIGINT | NULLABLE |
| revoked_reason | STRING | NULLABLE |
| notes | STRING | NULLABLE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

## 10. entitlement_assignments
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| entitlement_id | BIGINT | FK → entitlements |
| person_id | BIGINT | FK → people |
| removed_us | BIGINT | NULLABLE |
| removed_reason | STRING | NULLABLE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |

**Unique Constraint**: (entitlement_id, person_id) - partial unique where removed_us IS NULL deferred to application logic

---

# Seed Data

## Products
| code | name | description | kind | is_active |
|------|------|-------------|------|-----------|
| intro-workshop | Intro Workshop | Introduction to partner acrobatics | one_time | true |
| massage-60 | 60-Minute Massage | Relaxing therapeutic massage | one_time | true |

## Price Schedule
| name | valid_from_us | valid_to_us | is_active |
|------|---------------|-------------|-----------|
| 2026 Standard Pricing | 1704067200000000 (1/1/2026 UTC) | NULL | true |

## Product Prices
| product | price_schedule | permission_id | currency | amount_cents |
|---------|----------------|---------------|----------|--------------|
| intro-workshop | 2026 Standard Pricing | NULL | USD | 900 |
| massage-60 | 2026 Standard Pricing | NULL | USD | 16000 |

## Product Entitlement Rules
| product | grants_permission_id | seats_default | validity_kind | validity_days |
|---------|---------------------|---------------|---------------|---------------|
| intro-workshop | NULL | 1 | instant | NULL |
| massage-60 | NULL | 1 | instant | NULL |

---

# Implementation Order

Due to foreign key dependencies, tables must be created in this order:

1. `products` (no dependencies)
2. `price_schedules` (no dependencies)
3. `product_prices` (depends on products, price_schedules, permissions)
4. `product_entitlement_rules` (depends on products, permissions)
5. `purchases` (depends on people)
6. `purchase_items` (depends on purchases, products, price_schedules, permissions)
7. `payments` (depends on people, self-reference for refunds)
8. `purchase_payments` (depends on purchases, payments)
9. `entitlements` (depends on purchases, purchase_items, products)
10. `entitlement_assignments` (depends on entitlements, people)

---

# Verification

1. Build the server: `cd server/knottyyoga_server/build && cmake .. && make`
2. Run tests: `cd test/build && make && bin/knottyyoga_tests`
3. Clear and recreate database: `cd database_server && clear-database.cmd`
4. Start server and verify tables exist:
   ```sql
   \c knottyyoga;
   \dt products
   SELECT * FROM products;
   SELECT * FROM price_schedules;
   SELECT * FROM product_prices;
   ```
5. Verify seed data: 2 products, 1 price schedule, 2 product prices, 2 entitlement rules

---

# Design: Nullable Foreign Key Support

## Problem

The current `DatabaseInfo` API only supports non-nullable foreign keys via `AddColumnForeignKeyRef`. Several payment tables require nullable foreign keys where:
- The column can be NULL (no reference)
- When NOT NULL, the value must reference a valid row in the parent table

## Affected Columns

| Table | Column | References | Why Nullable |
|-------|--------|------------|--------------|
| product_prices | permission_id | permissions.id | NULL = fallback price for all users |
| product_entitlement_rules | grants_permission_id | permissions.id | Product may not grant any permission |
| purchase_items | pricing_permission_id | permissions.id | NULL when fallback price was used |
| payments | refund_for_payment_id | payments.id | Only refunds reference another payment |
| entitlements | purchase_item_id | purchase_items.id | Manual entitlements not tied to a line item |

## Current Workaround

Using `AddColumnNullable` with `DB_TYPE_BIGINT` - creates nullable column but no FK constraint. The relationship is documented but not enforced at the database level.

## Proposed Solution

Add `AddColumnForeignKeyRefNullable` method to `DatabaseInfo`:

```cpp
// database_info.h
void AddColumnForeignKeyRefNullable(
    std::string_view tableNameParent,
    std::string_view columnNameParent,
    std::string_view tableNameChild,
    std::string_view columnNameChild);
```

## SQL Behavior

```sql
-- Non-nullable FK (existing)
column_name BIGINT NOT NULL REFERENCES parent_table(id)

-- Nullable FK (proposed)
column_name BIGINT REFERENCES parent_table(id)
```

When `column_name` is NULL, the FK constraint is not checked. When it has a value, that value must exist in the parent table.

---

## Implementation Changes

### 1. ForeignKeyManager Changes

**File: `sql_util/schema/foreign_key_manager.h`**

```cpp
// Modify ForeignKeyManager class:
void AddForeignKeyReference(
    const TableColumnPair& parent,
    const TableColumnPair& child,
    bool nullable = false);  // Add default parameter

bool IsNullable(
    const TableColumnPair& parent,
    const TableColumnPair& child) const;  // New method
```

**File: `sql_util/schema/foreign_key_manager.cpp`**

```cpp
// Modify ForeignKeyReference struct in ForeignKeyManagerImpl:
struct ForeignKeyReference {
    TableColumnPair parent;
    TableColumnPair child;
    bool nullable = false;  // Add new field
};

// Modify ForeignKeyManagerImpl class:
void AddForeignKeyReference(
    const TableColumnPair& parent,
    const TableColumnPair& child,
    bool nullable);  // No default here - wrapper provides it

bool IsNullable(
    const TableColumnPair& parent,
    const TableColumnPair& child) const;  // New method
```

### 2. DatabaseInfo Changes

**File: `sql_util/schema/database_info.h`**

```cpp
// Add new method:
void AddColumnForeignKeyRefNullable(
    std::string_view tableNameParent,
    std::string_view columnNameParent,
    std::string_view tableNameChild,
    std::string_view columnNameChild);
```

**File: `sql_util/schema/database_info.cpp`**

- Add `AddColumnForeignKeyRefNullable` to `DatabaseInfoImpl`
- Add public wrapper in `DatabaseInfo`
- Implementation calls `AddColumnNullable` (instead of non-nullable) and passes `nullable=true` to `ForeignKeyManager::AddForeignKeyReference`

### 3. DDL Generation Changes

**File: `sql_util/database_access/db_and_table_operations.cpp`**

Modify `GenerateCreateTableSql` to check nullability when generating FK constraints:

```cpp
// Current code (line 29-35):
if (columnInfo.IsUnique()) {
    ssField << " UNIQUE";
}
else if (!columnInfo.IsNullable()) {
    ssField << " NOT NULL";
}

// This already handles it correctly!
// If the column is nullable (from AddColumnNullable), NOT NULL is not added.
// The FK constraint is added separately as a table constraint.
```

The current implementation already handles this correctly because:
1. `AddColumnForeignKeyRefNullable` will call `AddColumnNullable` which sets `nullable_=true`
2. `GenerateCreateTableSql` checks `columnInfo.IsNullable()` and skips `NOT NULL` if true
3. FK constraints are added as separate table constraints (lines 40-54)

No changes needed to `db_and_table_operations.cpp` - the existing logic handles it.

### 4. Schema Introspection Changes

**File: `sql_util/database_access/database_metadata.h`**

```cpp
// Add new method:
bool IsForeignKeyNullable(
    Transaction& transaction,
    std::string_view parentTable,
    std::string_view parentColumn,
    std::string_view childTable,
    std::string_view childColumn);
```

**File: `sql_util/database_access/database_metadata.cpp`**

```cpp
// SQL to check if FK column is nullable:
constexpr std::string_view kIsForeignKeyNullableSql = R"SQL(
SELECT c.is_nullable
FROM information_schema.columns c
WHERE c.table_name = $1
  AND c.column_name = $2
  AND c.table_schema = 'public'
)SQL";

bool IsForeignKeyNullable(
    Transaction& transaction,
    std::string_view parentTable,
    std::string_view parentColumn,
    std::string_view childTable,
    std::string_view childColumn) {
    KeyValueTableArray result = transaction.RunSqlStatementReturningKeyValueTableArray(
        kIsForeignKeyNullableSql, childTable, childColumn);
    if (result.empty()) {
        return false;
    }
    return result[0].at("is_nullable") == "YES";
}
```

**Modify `AddTableToDatabaseInfo` in `database_metadata.cpp`:**

```cpp
// Change line 379-384 from:
if (iter != foreignKeyTable.end()) {
    databaseInfo.AddColumnForeignKeyRef(
        iter->second.parentTable,
        iter->second.parentColumn,
        tableInfo.tableName,
        iter->first);
}

// To:
if (iter != foreignKeyTable.end()) {
    bool isNullable = IsForeignKeyNullable(
        transaction,
        iter->second.parentTable,
        iter->second.parentColumn,
        tableInfo.tableName,
        iter->first);
    if (isNullable) {
        databaseInfo.AddColumnForeignKeyRefNullable(
            iter->second.parentTable,
            iter->second.parentColumn,
            tableInfo.tableName,
            iter->first);
    } else {
        databaseInfo.AddColumnForeignKeyRef(
            iter->second.parentTable,
            iter->second.parentColumn,
            tableInfo.tableName,
            iter->first);
    }
}
```

### 5. Payment Table Updates

Update 5 files to use `AddColumnForeignKeyRefNullable`:
- `product_prices.cpp` - `permission_id`
- `product_entitlement_rules.cpp` - `grants_permission_id`
- `purchase_items.cpp` - `pricing_permission_id`
- `payments.cpp` - `refund_for_payment_id`
- `entitlements.cpp` - `purchase_item_id`

---

## Testing

### Unit Tests for ForeignKeyManager

**File: `sql_util/schema/foreign_key_manager_test.cpp`** (new or existing)

```cpp
TEST(ForeignKeyManagerTest, AddForeignKeyReferenceNullable) {
    ForeignKeyManager manager;
    TableColumnPair parent{"parent_table", "id"};
    TableColumnPair child{"child_table", "parent_id"};

    manager.AddForeignKeyReference(parent, child, true);

    EXPECT_TRUE(manager.HasForeignReference(child));
    EXPECT_TRUE(manager.IsNullable(parent, child));
}

TEST(ForeignKeyManagerTest, AddForeignKeyReferenceNotNullable) {
    ForeignKeyManager manager;
    TableColumnPair parent{"parent_table", "id"};
    TableColumnPair child{"child_table", "parent_id"};

    manager.AddForeignKeyReference(parent, child, false);

    EXPECT_TRUE(manager.HasForeignReference(child));
    EXPECT_FALSE(manager.IsNullable(parent, child));
}

TEST(ForeignKeyManagerTest, AddForeignKeyReferenceDefaultNotNullable) {
    ForeignKeyManager manager;
    TableColumnPair parent{"parent_table", "id"};
    TableColumnPair child{"child_table", "parent_id"};

    manager.AddForeignKeyReference(parent, child);  // No nullable param

    EXPECT_FALSE(manager.IsNullable(parent, child));
}
```

### Unit Tests for DatabaseInfo

**File: `sql_util/schema/database_info_test.cpp`** (add to existing)

```cpp
TEST(DatabaseInfoTest, AddColumnForeignKeyRefNullable) {
    DatabaseInfo info("test_db");
    info.AddTable("parent_table");
    info.AddColumnPrimaryKey("parent_table", "id", DB_TYPE_BIGSERIAL);

    info.AddTable("child_table");
    info.AddColumnPrimaryKey("child_table", "id", DB_TYPE_BIGSERIAL);
    info.AddColumnForeignKeyRefNullable("parent_table", "id", "child_table", "parent_id");

    const TableInfo& childTable = info.GetTableInfo("child_table");
    const ColumnInfo& fkColumn = childTable.GetColumn("parent_id");

    EXPECT_TRUE(fkColumn.IsNullable());
    EXPECT_TRUE(info.GetForeignKeyManager().HasForeignReference({"child_table", "parent_id"}));
    EXPECT_TRUE(info.GetForeignKeyManager().IsNullable({"parent_table", "id"}, {"child_table", "parent_id"}));
}
```

### Unit Tests for DDL Generation

**File: `sql_util/database_access/db_and_table_operations_test.cpp`** (add to existing)

```cpp
TEST(DbAndTableOperationsTest, GenerateCreateTableSqlWithNullableForeignKey) {
    DatabaseInfo info("test_db");
    info.AddTable("parent_table");
    info.AddColumnPrimaryKey("parent_table", "id", DB_TYPE_BIGSERIAL);

    info.AddTable("child_table");
    info.AddColumnPrimaryKey("child_table", "id", DB_TYPE_BIGSERIAL);
    info.AddColumnForeignKeyRefNullable("parent_table", "id", "child_table", "parent_id");

    std::string sql = DbOps::GenerateCreateTableSql(info, "child_table");

    // Should NOT contain "NOT NULL" for parent_id
    EXPECT_TRUE(sql.find("parent_id BIGINT,") != std::string::npos ||
                sql.find("parent_id BIGINT DEFAULT") != std::string::npos);
    EXPECT_TRUE(sql.find("parent_id BIGINT NOT NULL") == std::string::npos);
    // Should still have FK constraint
    EXPECT_TRUE(sql.find("FOREIGN KEY(parent_id) REFERENCES parent_table(id)") != std::string::npos);
}
```

### Integration Tests for Schema Introspection

**File: `sql_util/database_access/database_metadata_test.cpp`** (add to existing)

```cpp
TEST(DatabaseMetadataTest, IsForeignKeyNullable) {
    // Requires database test helper
    TestDatabaseUtil testDb;
    testDb.RunInTransaction("IsForeignKeyNullable", [&](Transaction& transaction) {
        // Create parent table
        transaction.RunSqlStatement(
            "CREATE TABLE test_parent (id BIGSERIAL PRIMARY KEY)");

        // Create child with nullable FK
        transaction.RunSqlStatement(
            "CREATE TABLE test_child_nullable ("
            "id BIGSERIAL PRIMARY KEY, "
            "parent_id BIGINT REFERENCES test_parent(id))");

        // Create child with non-nullable FK
        transaction.RunSqlStatement(
            "CREATE TABLE test_child_notnull ("
            "id BIGSERIAL PRIMARY KEY, "
            "parent_id BIGINT NOT NULL REFERENCES test_parent(id))");

        EXPECT_TRUE(DbMeta::IsForeignKeyNullable(
            transaction, "test_parent", "id", "test_child_nullable", "parent_id"));
        EXPECT_FALSE(DbMeta::IsForeignKeyNullable(
            transaction, "test_parent", "id", "test_child_notnull", "parent_id"));
    });
}

TEST(DatabaseMetadataTest, DatabaseInfoFromDatabaseWithNullableForeignKey) {
    TestDatabaseUtil testDb;
    testDb.RunInTransaction("DatabaseInfoFromDatabaseNullableFK", [&](Transaction& transaction) {
        // Create tables with nullable FK
        transaction.RunSqlStatement(
            "CREATE TABLE test_parent (id BIGSERIAL PRIMARY KEY)");
        transaction.RunSqlStatement(
            "CREATE TABLE test_child ("
            "id BIGSERIAL PRIMARY KEY, "
            "parent_id BIGINT REFERENCES test_parent(id))");

        StringArray allowedTables = {"test_parent", "test_child"};
        DbSchema::DatabaseInfo info = DbMeta::DatabaseInfoFromDatabase(
            transaction, "test_db", allowedTables);

        // Verify FK is marked as nullable
        EXPECT_TRUE(info.GetForeignKeyManager().IsNullable(
            {"test_parent", "id"}, {"test_child", "parent_id"}));
    });
}
```

---

## Steps

- [x] Add `nullable` field to `ForeignKeyReference` struct in `foreign_key_manager.cpp` ✅ 2026-01-28
- [x] Add `nullable` parameter to `AddForeignKeyReference` in `ForeignKeyManagerImpl` ✅ 2026-01-28
- [x] Add `nullable` parameter (with default) to `AddForeignKeyReference` in `ForeignKeyManager` ✅ 2026-01-28
- [x] Add `IsNullable` method to `ForeignKeyManagerImpl` ✅ 2026-01-28
- [x] Add `IsNullable` method to `ForeignKeyManager` ✅ 2026-01-28
- [x] Add unit tests for `ForeignKeyManager` nullable support ✅ 2026-01-28
- [x] Add `AddColumnForeignKeyRefNullable` to `DatabaseInfoImpl` ✅ 2026-01-28
- [x] Add `AddColumnForeignKeyRefNullable` to `DatabaseInfo` ✅ 2026-01-28
- [x] Add unit tests for `DatabaseInfo::AddColumnForeignKeyRefNullable` ✅ 2026-01-28
- [x] Add unit test for DDL generation with nullable FK ✅ 2026-01-28
- [x] Add `IsForeignKeyNullable` to `database_metadata.h` ✅ 2026-01-28
- [x] Add `IsForeignKeyNullable` implementation to `database_metadata.cpp` ✅ 2026-01-28
- [x] Modify `AddTableToDatabaseInfo` to use `IsForeignKeyNullable` ✅ 2026-01-28
- [x] Add integration tests for schema introspection with nullable FK ✅ 2026-01-28
- [x] Update `product_prices.cpp` to use `AddColumnForeignKeyRefNullable` ✅ 2026-01-28
- [x] Update `product_entitlement_rules.cpp` to use `AddColumnForeignKeyRefNullable` ✅ 2026-01-28
- [x] Update `purchase_items.cpp` to use `AddColumnForeignKeyRefNullable` ✅ 2026-01-28
- [x] Update `payments.cpp` to use `AddColumnForeignKeyRefNullable` ✅ 2026-01-28
- [x] Update `entitlements.cpp` to use `AddColumnForeignKeyRefNullable` ✅ 2026-01-28

## Open Questions

1. ~~Should `ForeignKeyManager` track whether a FK is nullable?~~ **Resolved**: Yes, add `nullable` field and `IsNullable` method.

2. Should there be a combined method like `AddColumnForeignKeyRefOptional` that takes a nullable parameter? Or keep two separate methods for clarity?
	1. Let's keep the two separate for clarity.

3. ~~Any impact on schema introspection or admin UI that reads FK metadata?~~ **Resolved**: Yes, add `IsForeignKeyNullable` and update `DatabaseInfoFromDatabase`.

4. ~~Changes needed for DDL generation?~~ **Resolved**: No changes needed - existing logic already handles nullable columns correctly.

5. ~~Testing details?~~ **Resolved**: Added comprehensive test plan above.