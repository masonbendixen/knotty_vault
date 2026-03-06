---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/6/2026
Version: 0.1
tags: 
---
# Overview

Please go into plan mode. Please use this document for your planning document. Do not create anything in .claude/plans. Use this file for planning and do not ask me for permission to modify this file. It is your plan file. Please leave this Overview section in tact and do your work in the sections below.

Going back to improving the execution speed of my tests, I was curious if there is anyway to try and speed this up. The changes to change the ACID type setting on a transaction really didn't speed things up and even made things a bit slower. I was curious if we can change the settings on the test database creation in code (not postgres conf changes) that might speed things up. For the test execution database, I really won't have concurrent access and I don't really need things even to be durable. Things could just operate in memory and not even be committed to disk. We do need to support reads and writes but the tests are all serialized and don't run concurrently. Short of that, we could explore using a single transaction for tests and use something like checkpoints that we can revert to. I'm open to any ideas you have for speeding up test execution speed knowing that we don't need most of the ACID things we care about in production and that test execution is not in parallel. 

Please use the codebase and this document to generate your plan. Please create phases of implementation with check boxes next to them.

# Current State Analysis

## How Tests Work Today

Every test follows this pattern:

1. `TestDatabaseUtil testDb` — gets a handle to the shared `DatabaseHelperTest` (single `pqxx::connection` for entire test run)
2. `testDb.RunInTransaction(...)` — creates a `pqxx::work` transaction
3. Inside the lambda: create all needed tables (stored procedures, admin_alerts, people, etc.)
4. Run the actual test logic (usually a tiny fraction of wall time)
5. `HandleAbort` destructor calls `trans_.abort()` — all table creation and data is discarded

The database starts completely empty for every test. There is no shared state between tests — the transaction abort wipes everything.

## Key Numbers

- **102 test files** use `RunInTransaction` (818 total calls — many files have multiple tests)
- **43 test files** call `MakePaymentTables` (354 total calls) — each creates **33+ tables** + 2 stored procedures
- **24 test files** call `MakeTestPeopleTable` (134 total calls) — each creates 3 tables + 2 stored procedures
- **42 endpoint test files** create `EndpointTestHelper` (178 total calls) — each additionally creates 3 admin metadata tables + initializes full WebApp routing

## Where Time Goes Per Test

| Step | What Happens | Approx Cost |
|------|-------------|-------------|
| Create `pqxx::work` | Transaction start | ~1ms |
| `CreateStoredProceduresBeforeTables` | `CREATE FUNCTION now_us()` | ~2-5ms |
| `MakePaymentTables` (33 tables) | 33× (generate SQL string + round-trip to PG + CREATE TABLE) | ~50-150ms |
| `MakeSchedulingTables` (12 tables) | Additional tables when needed | ~20-50ms |
| `EndpointTestHelper` constructor | 3 more tables + WebApp init + route registration | ~10-30ms |
| `CreateStoredProceduresAfterTables` | `CREATE FUNCTION get_admin_alerts_in_window()` | ~2-5ms |
| Actual test logic | INSERT/SELECT/assertions | ~1-10ms |
| Transaction abort | Discard everything | ~1-5ms |

**Ratio**: 90-98% of each test's time is setup overhead that gets thrown away.

---

# Optimization Strategies

## Strategy 1: PostgreSQL Session-Level Performance Tuning

**Idea**: After creating the test database connection, execute SQL `SET` commands to disable durability guarantees that aren't needed for tests.

**Applicable SET commands** (all session-scoped, no postgresql.conf changes):

```sql
SET synchronous_commit = OFF;          -- Don't wait for WAL flush to disk
SET fsync = OFF;                       -- Skip fsync calls (session-level may not work, but worth trying)
SET full_page_writes = OFF;            -- Skip full-page writes after checkpoint
SET wal_level = minimal;               -- Minimal WAL (may not be settable per-session)
SET work_mem = '256MB';                -- More memory for sorts/hashes
SET maintenance_work_mem = '256MB';    -- More memory for CREATE INDEX, etc.
SET max_wal_size = '2GB';              -- Reduce checkpoint frequency
SET checkpoint_timeout = '30min';      -- Reduce checkpoint frequency
```

**Implementation**: Add these SET statements in `GlobalDatabaseTestSupport::InitializeInternal()` right after connecting to the test database.

**Pros**: Zero changes to test code, zero changes to production code. Easy to implement.
**Cons**: Some settings may only be configurable in postgresql.conf (server-level). Moderate speedup — reduces I/O wait but doesn't eliminate the repeated DDL round-trips.
**Estimated impact**: 10-30% speedup on I/O-bound operations.

---

## Strategy 2: UNLOGGED Tables for Tests

**Idea**: PostgreSQL supports `CREATE UNLOGGED TABLE` which skips WAL (write-ahead log) entirely. These tables are faster to create and write to because they don't generate WAL records.

**Implementation**: Modify `GenerateCreateTableSql()` in `db_and_table_operations.cpp` to accept an option for `UNLOGGED`:

```cpp
// Option A: Add a bool parameter
std::string GenerateCreateTableSql(
    DbSchema::DatabaseInfo databaseInfo,
    std::string_view tableName,
    bool unlogged = false);
// Generates: "CREATE UNLOGGED TABLE ..." when unlogged=true

// Option B: Check if DatabaseHelper::IsTest()
// Pass databaseHelper to CreateTable and auto-detect
```

**Pros**: Meaningful speedup for table creation. No WAL overhead for inserts/updates during tests.
**Cons**: Requires modifying `GenerateCreateTableSql` and `CreateTable` signatures. UNLOGGED tables are wiped on crash recovery (fine for tests). Need to thread the "unlogged" flag through the call chain.
**Estimated impact**: 20-40% speedup on table creation and data insertion.

---

## Strategy 3: Savepoints Instead of Full Transaction Abort (Checkpoint/Restore Pattern)

**Idea**: Instead of creating and aborting a full transaction per test (which discards all table creation), use a single long-running transaction with SAVEPOINTs. Create all tables once at the start, then for each test:

1. `SAVEPOINT before_test`
2. Run test (inserts data, etc.)
3. `ROLLBACK TO SAVEPOINT before_test` (removes test data but keeps tables)

**Implementation**:

```
GlobalDatabaseTestSupport::Initialize():
  1. Create test database
  2. Begin a long-running transaction (never committed)
  3. Create ALL tables (stored procs, payment tables, scheduling tables, etc.)
  4. SAVEPOINT baseline

Per test:
  1. SAVEPOINT test_start
  2. Run test lambda (only inserts/queries — tables already exist)
  3. ROLLBACK TO SAVEPOINT test_start

GlobalDatabaseTestSupport::Shutdown():
  1. Abort the long-running transaction
```

**Key changes needed**:
- `GlobalDatabaseTestSupport` holds an open `pqxx::work` and `TransactionImpl`
- `DatabaseHelperTest::RunInTransaction` no longer creates a new `pqxx::work` — it uses the existing one with savepoints
- Test helpers (`MakePaymentTables`, `MakeTestPeopleTable`) must detect "tables already exist" or be called once at startup
- Tests that create specific additional tables (e.g., `instructors`, `sessions`) need handling — either create them all upfront or use `CREATE TABLE IF NOT EXISTS`

**Pros**: Eliminates ~95% of per-test overhead. Tables created once for entire test run. Each test only pays for its own data insertion + savepoint rollback.
**Cons**: Most complex to implement. Requires reworking `RunInTransaction` and table creation patterns. Tests that create different table subsets need careful handling. A failing CREATE TABLE in one test could corrupt state for subsequent tests (though savepoint rollback prevents this). Single long transaction means PostgreSQL holds more resources.
**Estimated impact**: 80-95% speedup. The dominant cost (table creation) happens once instead of hundreds of times.

---

## Strategy 4: PostgreSQL Template Database

**Idea**: PostgreSQL has a native feature where you can create a database from a template: `CREATE DATABASE test_knottyyoga TEMPLATE test_knottyyoga_template`. The template database is created once with all tables, and each test run gets an instant copy via filesystem-level copy (much faster than re-running DDL).

**Implementation**:

```
First run (or explicit setup step):
  1. Create test_knottyyoga_template database
  2. Run all table creation DDL (MakePaymentTables + MakeSchedulingTables + admin tables + stored procs)
  3. This becomes the template

Each test run:
  1. DROP DATABASE test_knottyyoga
  2. CREATE DATABASE test_knottyyoga TEMPLATE test_knottyyoga_template
  3. Connect to test_knottyyoga (all tables already exist)

Per test:
  1. Begin transaction
  2. Run test (tables exist, just insert data)
  3. Abort transaction (data cleaned up, tables remain)
```

**Key changes**:
- `GlobalDatabaseTestSupport::InitializeInternal()` checks if template exists. If not, creates it with all tables. Then creates the test DB from template.
- Test helpers like `MakePaymentTables` become no-ops (or are removed from individual tests)
- Tests that need specific tables beyond the standard set would need those added to the template

**Pros**: PostgreSQL handles the optimization natively. Fast database creation. Clean per-test isolation (transaction abort only removes data, tables persist). Template database survives across test runs (no re-creation unless schema changes).
**Cons**: Template must be rebuilt when schema changes (need a mechanism for this — e.g., version stamp, or always rebuild). Requires managing two databases. `CREATE DATABASE ... TEMPLATE` cannot be done inside a transaction. Individual tests still pay for transaction start/abort + data insertion. Doesn't help tests that need different table subsets (but most tests use MakePaymentTables anyway).
**Estimated impact**: 60-80% speedup. Eliminates per-test-run DDL, but per-test transaction overhead remains.

---

## Strategy 5: Persistent Tables + DELETE/TRUNCATE Between Tests

**Idea**: Create all tables once during `GlobalDatabaseTestSupport::Initialize()` (in a committed transaction), then between tests use `TRUNCATE` or `DELETE` to clear data rather than recreating tables.

**Implementation**:

```
GlobalDatabaseTestSupport::Initialize():
  1. Create test database
  2. Connect, run committed transaction that creates ALL tables
  3. Tables persist across tests

Per test:
  1. Begin transaction
  2. Run test
  3. Abort transaction (data cleaned up automatically)
  OR
  1. Run test (auto-commit)
  2. TRUNCATE all tables between tests
```

**With transaction abort approach** (simplest):
- Create tables once in a committed transaction at startup
- Each test's `RunInTransaction` + abort pattern still works — it just doesn't need to CREATE tables anymore
- Test helpers (`MakePaymentTables`, `MakeTestPeopleTable`) are called once at startup instead of per-test

**Key changes**:
- Add `SetupAllTables()` method to `GlobalDatabaseTestSupport` that commits table creation
- `MakePaymentTables` and friends either become no-ops when tables exist, or are only called at startup
- Tests stop calling `MakePaymentTables` / `MakeTestPeopleTable` — they just use the pre-existing tables

**Pros**: Simpler than savepoints. Tables survive test failures. No template database management. Transaction abort still provides per-test data isolation.
**Cons**: Requires either modifying every test to remove table creation calls, or making `MakePaymentTables` detect existing tables. Tests that need only a subset of tables now get all tables (minor — no real downside). Need to handle tests that create additional tables not in the standard set.
**Estimated impact**: 70-90% speedup. One committed transaction creates everything, then abort-pattern cleans up data per test.

---

## Strategy 6: Batch DDL Execution

**Idea**: Instead of executing 33+ separate `CREATE TABLE` statements as individual round-trips, concatenate them into a single SQL string and execute once.

**Implementation**: Modify `MakePaymentTables` to build a single SQL batch:

```cpp
void MakePaymentTables(Transaction& transaction, TestDatabaseUtil& testDb) {
    auto dbInfo = testDb.GetDatabaseInfo();
    std::string batchSql;
    batchSql += GenerateCreateFunctionSql("now_us", ...);
    batchSql += GenerateCreateTableSql(dbInfo, "admin_alerts") + ";";
    batchSql += GenerateCreateTableSql(dbInfo, "people") + ";";
    // ... all 33 tables ...
    batchSql += GenerateCreateFunctionSql("get_admin_alerts_in_window", ...);
    transaction.RunSqlStatement(batchSql);
}
```

**Pros**: Reduces 35+ network round-trips to 1. Easy to implement alongside other strategies. Works with existing transaction pattern.
**Cons**: Only reduces network latency, not PostgreSQL's actual DDL execution time. If any statement fails, the entire batch fails (harder to debug). Still creates tables per-test unless combined with a persistence strategy.
**Estimated impact**: 20-40% speedup (depends on network latency to PostgreSQL — bigger impact in Docker).

---

## Strategy 7: `CREATE TABLE IF NOT EXISTS` + Committed Setup Transaction

**Idea**: A hybrid approach — make table creation idempotent with `IF NOT EXISTS`, create all tables in a committed transaction at startup, and let individual test calls to `MakePaymentTables` be harmless no-ops.

**Implementation**:
1. Change `GenerateCreateTableSql` to generate `CREATE TABLE IF NOT EXISTS`
2. In `GlobalDatabaseTestSupport::InitializeInternal()`, after creating the database, run a committed transaction that calls `MakePaymentTables` + `MakeSchedulingTables` + all admin tables + stored procedures
3. Individual tests still call `MakePaymentTables` etc. but they're instant no-ops because tables already exist
4. The abort-per-test pattern continues to work (only data is rolled back)

**Pros**: **Minimal test code changes** — existing tests continue to work unmodified. Gradual migration — tests naturally benefit. If schema changes, just re-run the database helper to recreate. Idempotent table creation means tests that create additional tables (instructors, sessions) work fine.
**Cons**: `IF NOT EXISTS` has a small overhead per call (PostgreSQL still checks). Stored procedures need `CREATE OR REPLACE FUNCTION`. Need a committed transaction at startup (new concept for the test infrastructure).
**Estimated impact**: 70-85% speedup with minimal code changes.

---

# Recommended Implementation Plan

I recommend a phased approach, starting with the easiest wins and building toward the biggest gains. Each phase is independently valuable.

## Phase 1: Session-Level PostgreSQL Tuning (Easy Win)
- [x] Add performance-oriented `SET` commands after test database connection in `GlobalDatabaseTestSupport::InitializeInternal()`
- [x] Commands to add:
  ```sql
  SET synchronous_commit = OFF;
  SET work_mem = '256MB';
  SET maintenance_work_mem = '256MB';
  ```
- [ ] Verify these settings take effect (write a test that checks `SHOW synchronous_commit`)
- [ ] Measure before/after test suite timing

## Phase 2: UNLOGGED Tables for Tests (Moderate Win)
- [ ] Add `bool unlogged` parameter (default `false`) to `GenerateCreateTableSql()` in `db_and_table_operations.cpp`
- [ ] When `unlogged` is true, generate `CREATE UNLOGGED TABLE` instead of `CREATE TABLE`
- [ ] Add `bool unlogged` parameter to `CreateTable()` and `DropIfExistsAndCreateTable()`
- [ ] In test infrastructure, pass `unlogged=true` when calling `CreateTable` — detect via `DatabaseHelper::IsTest()` or explicit parameter
- [ ] Verify all tests still pass with UNLOGGED tables
- [ ] Measure before/after

## Phase 3: Committed Table Setup at Startup (Big Win — Strategy 7)
This is the highest-impact change: create all tables once in a committed transaction during `GlobalDatabaseTestSupport::Initialize()`, then let the per-test abort pattern clean up only data.

### 3.1 Make DDL idempotent
- [ ] Change `GenerateCreateTableSql()` to support `IF NOT EXISTS` (add parameter or always use it in test mode)
- [ ] Change stored procedure creation (`CreateNowUs`, `CreateGetAdminAlertsInWindow`) to use `CREATE OR REPLACE FUNCTION`
- [ ] Verify existing tests still pass with `IF NOT EXISTS` / `CREATE OR REPLACE`

### 3.2 Add committed setup transaction to GlobalDatabaseTestSupport
- [ ] Add a new method `SetupAllTables()` to `GlobalDatabaseTestSupport` that:
  1. Opens a `pqxx::work` transaction (NOT aborted — will be committed)
  2. Calls `StoredProcedures::CreateStoredProceduresBeforeTables`
  3. Calls `MakePaymentTables` equivalent (all tables from payment_table_test_helper)
  4. Calls `MakeSchedulingTables` equivalent
  5. Creates admin metadata tables (allowed_tables, admin_top_level_tables, admin_nested_tables)
  6. Creates any other commonly used tables (sessions, device_tokens, email_verifications, instructors)
  7. Calls `StoredProcedures::CreateStoredProceduresAfterTables`
  8. **Commits** the transaction
- [ ] Call `SetupAllTables()` from `InitializeInternal()` after creating the database connection
- [ ] Verify all tests pass (existing `MakePaymentTables` calls are harmless with `IF NOT EXISTS`)

### 3.3 Measure and document improvement
- [ ] Time full test suite before and after
- [ ] Document the new test infrastructure pattern

## Phase 4: Batch DDL Execution (Optional Polish)
- [ ] Modify the startup `SetupAllTables()` to concatenate all DDL into a single SQL statement
- [ ] Execute as one round-trip instead of 50+
- [ ] This further reduces the one-time startup cost

## Phase 5: Savepoint Pattern (Advanced — Optional)
Only pursue this if Phase 3 doesn't provide sufficient speedup, or if the abort pattern itself is slow.

- [ ] Investigate whether `pqxx::subtransaction` (wraps SAVEPOINT) works with the existing `TransactionImpl` abstraction
- [ ] If viable, modify `DatabaseHelperTest::RunInTransaction` to use savepoints within a long-running transaction instead of creating/aborting full transactions
- [ ] This would eliminate per-test transaction creation overhead entirely
- [ ] Requires careful handling of test failures (a failed savepoint doesn't corrupt the parent transaction)

---

# Notes and Considerations

1. **Schema migration**: When the database schema changes (new tables, new columns), the committed tables from Phase 3 become stale. The simplest approach: the `GlobalDatabaseTestSupport::Initialize()` always drops and recreates the test database (which it already does), then re-runs `SetupAllTables()`. This is fine since it only happens once per test run.

2. **Tests with unique table needs**: Some tests create tables not in the standard set (e.g., `instructors`, `sessions`). With `IF NOT EXISTS`, these still work — the per-test creation is just a no-op for already-existing tables, and the few tables not pre-created are created normally. Over time, popular tables can be added to `SetupAllTables()`.

3. **No parallel test execution needed**: Since tests are serialized and share one connection, we don't need to worry about concurrent access. This simplifies the savepoint approach considerably.

4. **Docker network latency**: If tests run inside Docker with PostgreSQL in another container, every SQL round-trip pays network overhead. Batch DDL (Phase 4) is especially valuable in this scenario.

5. **The `CREATE OR REPLACE FUNCTION` change**: PostgreSQL supports this natively for functions. The stored procedures (`now_us()` and `get_admin_alerts_in_window()`) can be changed from `CREATE FUNCTION` to `CREATE OR REPLACE FUNCTION` with no behavioral difference.

6. **Backward compatibility**: All phases maintain backward compatibility — existing test code continues to work. Phase 3 specifically uses `IF NOT EXISTS` so that tests calling `MakePaymentTables` see no change in behavior, just faster execution.

---

# Execution Time Log

| Run | Description | Time (ms) | Time (min:sec) | Change |
|-----|-------------|-----------|----------------|--------|
| 1 | Baseline — no optimizations | 643,287 | 10:43 | — |

---

# Tests with Custom Table Definitions

Several test files create their own versions of tables (especially `people` and `orders`) with non-standard columns, rather than using the standard `DbSchema::MakePeopleTable` definitions. These were written before the standard table definitions were established and many could likely be migrated. This matters because a "create all standard tables once at startup" strategy won't cover these custom tables — they need separate handling or migration.

## Files with Custom Table Definitions

### 1. `sql_util/database_access/database_util_test.cpp`
- **Custom `MakePeopleTable`**: Uses `person_id` as column name (instead of standard `id`), no timestamp columns, no stored procedures. A simplified version for testing low-level database utility operations.
- **Migration candidate**: Partial — some tests use the custom schema intentionally, but many could work with the standard schema. However, since this file also contains meta-database operations (see below), it may need to stay custom.

### 2. `sql_util/database_access/database_crud_helpers_test.cpp`
- **Custom `MakePeopleTable`**: Same simplified schema as database_util_test (person_id, first_name, last_name, email).
- **Custom `MakePeopleTableWithTimestamp`**: Adds created_at/modified_at columns for testing timestamp behavior.
- **Custom `MakeOrdersTable`**: A simple `orders` table with FK to people, for testing CRUD join operations.
- **Migration candidate**: Yes — these tests exercise generic CRUD helpers and would likely work with standard DbSchema tables, just with different column names in assertions.

### 3. `sql_util/database_access/database_metadata_test.cpp`
- **Custom `MakePeopleTable`** and **custom `MakeOrdersTable`**: Simplified schemas for testing database metadata introspection (information_schema queries).
- **Migration candidate**: Partial — some tests verify specific column names/types in the metadata, so migrating would require updating expected values. But this is straightforward.

### 4. `sql_util/database_access/transaction_impl_test.cpp`
- **Custom `MakePeopleTable`** and **custom `MakeOrdersTable`**: Same simplified schemas for testing transaction commit/abort/error handling behavior.
- **Migration candidate**: Yes — these tests care about transaction behavior, not specific table structure.

### 5. `sql_util/database_access/database_rest_helper_test.cpp`
- **Custom people table without primary key**: Intentionally creates a malformed table for negative testing (testing error handling when PK is missing).
- **Migration candidate**: No — the custom schema IS the test. This must remain custom.

### 6. `sql_util/database_access/db_and_table_operations_test.cpp`
- **Minimal test tables**: Creates bare-minimum tables for testing DDL generation (`GenerateCreateTableSql` output verification).
- **Migration candidate**: No — these tests verify SQL generation for specific table configurations and need precise control over the table definition.

## Recommendation

For Phase 3 (committed table setup), these files should be handled as follows:
- **Migrate where possible** (files 2, 3, 4): Rewrite to use standard `DbSchema` table definitions. This reduces the number of custom table creation calls and lets more tests benefit from pre-created tables.
- **Leave custom where intentional** (files 5, 6): These tests are testing specific table configurations as part of their purpose.
- **Evaluate case-by-case** (file 1): Some tests in database_util_test could migrate, others are tightly coupled to the custom schema.

Even without migration, these tests work fine with `IF NOT EXISTS` — their custom table creation runs in the per-test transaction and gets aborted as usual. They just don't benefit from the pre-created tables optimization. With only ~6 files affected, this is a small fraction of the total test suite.

---

# Meta-Database Operation Tests

These test files perform database-level operations (CREATE DATABASE, DROP DATABASE, information_schema queries) that are fundamentally different from normal CRUD tests. They will remain separate and somewhat slower under any optimization scheme because they need control over the database itself, not just tables within it.

## `sql_util/database_access/database_util_test.cpp` — 4 meta-database tests

These tests use `MakeNoDatabaseHelper()` to get a connection without a specific database, then CREATE/DROP entire databases:

| Test | What It Does |
|------|-------------|
| `CreateDatabaseAndDropDatabase` | Creates a test DB, verifies it exists, drops it, verifies it's gone |
| `CreateDatabaseThatAlreadyExists` | Creates a DB twice, verifies the second call is handled |
| `DropDatabaseThatDoesNotExist` | Drops a non-existent DB, verifies error handling |
| `DoesDatabaseExist` | Tests the database existence check function |

These tests **cannot** run inside the normal per-test transaction pattern because `CREATE DATABASE` and `DROP DATABASE` cannot run inside a transaction. They use their own connection management.

## `sql_util/database_access/database_metadata_test.cpp` — 19 metadata introspection tests

These tests query `information_schema` to verify table/column metadata. They create custom tables and then inspect them via metadata functions:

- `DoesTableExist` / `DoesColumnExist` / `DoesConstraintExist`
- `GetColumnNamesForTable` / `GetForeignKeyInfoForTable`
- `GetPrimaryKeyColumnName` / `HasPrimaryKey`
- `GetColumnDbType` / `IsColumnNullable` / `IsColumnAutoIncrement`
- Various edge cases (table not found, column not found, etc.)

These tests DO run inside transactions (and could benefit from pre-created tables), but they use custom table definitions because they test specific metadata properties (column types, nullable flags, FK relationships). Migrating them to standard tables is possible but would require updating all expected metadata values.

## Impact on Optimization

- **4 meta-database tests** in `database_util_test.cpp` will always be slower — they manage entire databases and can't use the shared transaction infrastructure. This is unavoidable but a tiny fraction of the total test suite.
- **19 metadata tests** in `database_metadata_test.cpp` could potentially benefit from pre-created tables if migrated to standard schemas, but this is low priority given the small count.
- **Total**: 23 tests (~2.8% of test suite assuming ~818 total `RunInTransaction` calls) that may not benefit from the Phase 3 optimization. The other 97%+ will see the full speedup.