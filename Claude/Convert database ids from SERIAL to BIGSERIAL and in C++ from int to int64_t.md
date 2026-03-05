---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 1/22/2026
Version: 0.1
tags: 
---
# Overview

I created my project with all my IDs for database tables as SERIAL which means that they are int in C++. I would like to convert them to BIGSERIAL which will mean they will be int64_t in C++. I currently have support for various database types in the project and map them to various enums and C++ native types. We need to add support for BIGSERIAL. I mean add, not replace SERIAL. This means adding the enums and mappings. I also have places that I map SERIAL to a 32bit int for foreign keys that will need to be mapped to 64bit int. All the places under db_schema where we define the id or a foreign key reference will need to be added. Then we need to move up the stack for the C++ / database layer and make sure that these are all turned to int64_t. Many places, we convert database entities to strings so these will also need to be mappings to int64_t instead of int. Generally, any place we use an int and not a size_t is suspicious. For instance, all the table helpers identify things by int and even the person identifier in the session needs to migrate. This will be a large change so please list the steps needed in this document and lets start putting together the plan.

# Steps

## Phase 1: Type System Foundation

### Files to modify:

**1. `server/knottyyoga_server/src/sql_util/database_common.h`**
- Add `DB_TYPE_BIGSERIAL` to the `DatabaseTypes` enum

**2. `server/knottyyoga_server/src/sql_util/schema/column_info.cpp`**
- Add case in `SqlTypeFromDatabaseType()`: `DB_TYPE_BIGSERIAL` -> `"BIGSERIAL"`

**3. `server/knottyyoga_server/src/sql_util/database_access/database_metadata.cpp`**
- Add to lookup table: `{"BIGSERIAL", DB_TYPE_BIGSERIAL}`, `{"SERIAL8", DB_TYPE_BIGSERIAL}`

**4. `server/knottyyoga_server/src/sql_util/schema/database_info.cpp`**
- In `AddColumnForeignKeyRef()`, add conversion: `DB_TYPE_BIGSERIAL` -> `DB_TYPE_BIGINT`

---

## Phase 2: CRUD Helper Layer

**5. `server/knottyyoga_server/src/sql_util/database_access/database_crud_helpers.h`**
- Add: `int64_t AddRowToTableFetchInt64PrimaryKey(...)`

**6. `server/knottyyoga_server/src/sql_util/database_access/database_crud_helpers.cpp`**
- Implement `AddRowToTableFetchInt64PrimaryKey()` using `std::stoll()`

---

## Phase 3: Schema Definitions (db_schema/)

Change `DB_TYPE_SERIAL` to `DB_TYPE_BIGSERIAL` in `AddColumnPrimaryKey()` calls:

| File | Table(s) |
|------|----------|
| `db_schema/people.cpp` | people |
| `db_schema/sessions.cpp` | sessions |
| `db_schema/device_tokens.cpp` | device_tokens |
| `db_schema/roles.cpp` | roles |
| `db_schema/permissions.cpp` | permissions |
| `db_schema/role_assignments.cpp` | role_assignments |
| `db_schema/role_permissions.cpp` | role_permissions |
| `db_schema/classes.cpp` | classes |
| `db_schema/email_verifications.cpp` | email_verifications |
| `db_schema/photos.cpp` | photo_instances, source_photos, scaled_photos |
| `db_schema/admin_alerts.cpp` | admin_alerts |
| `db_schema/config_secrets.cpp` | config_secrets |
| `db_schema/admin_table_friendly_names.cpp` | admin_table_friendly_names |
| `db_schema/admin_column_friendly_names.cpp` | admin_column_friendly_names |
| `db_schema/admin_column_data_info.cpp` | admin_column_data_info |

---

## Phase 4: Table Helpers (Headers)

Change `int` to `int64_t` for all ID parameters and return types:

| File | Changes |
|------|---------|
| `sql_util/table_helpers/people.h` | `AddPerson()` return, all `personId` params |
| `sql_util/table_helpers/sessions.h` | `AddSession()` return, all `id`/`personId` params |
| `sql_util/table_helpers/device_tokens.h` | All `id`/`personId` params |
| `sql_util/table_helpers/roles.h` | `AddRole()` return, all `id` params |
| `sql_util/table_helpers/permissions.h` | Same pattern |
| `sql_util/table_helpers/role_assignments.h` | All ID params |
| `sql_util/table_helpers/role_permissions.h` | All ID params |
| `sql_util/table_helpers/email_verifications.h` | All ID params |
| `sql_util/table_helpers/admin_alerts.h` | `alertId` param |

---

## Phase 5: Table Helpers (Implementations)

For each `.cpp` file corresponding to Phase 4 headers:
- Change return types and parameters to `int64_t`
- Replace `AddRowToTableFetchIntPrimaryKey` with `AddRowToTableFetchInt64PrimaryKey`
- `StringFromInt()` already has an `int64_t` overload - no changes needed

---

## Phase 6: Auth Module

**`auth/session.h`**
- Change `int GetPersonId() const` to `int64_t`
- Change `int personId_ = -1` to `int64_t personId_ = -1`

**`auth/session.cpp`**
- All `std::stoi()` for IDs -> `std::stoll()`

**`auth/person.h`**
- Change all `int id`, `int personId`, `int& outPersonId` to `int64_t`

**`auth/person.cpp`**
- All `std::stoi()` for IDs -> `std::stoll()` (~20 occurrences)
- Local `int` variables for IDs -> `int64_t`

---

## Phase 7: Endpoints

Files with `std::stoi()` for IDs to update:
- `endpoints/account_activation.cpp`
- `endpoints/logout.cpp`

---

## Phase 8: Test Files

Update all test files with `std::stoi()` for IDs:
- `auth/person_test.cpp` (~15 occurrences)
- `auth/session_test.cpp`
- `sql_util/table_helpers/*_test.cpp`
- `endpoints/*_test.cpp`
- `sql_util/json/database_rest_helper_test.cpp`

---

## Phase 9: Database Migration

For existing databases, run migration SQL:
```sql
-- For each table:
ALTER TABLE <table_name> ALTER COLUMN id TYPE BIGINT;
ALTER SEQUENCE <table_name>_id_seq AS BIGINT;

-- For foreign key columns:
ALTER TABLE sessions ALTER COLUMN person_id TYPE BIGINT;
-- (similar for all FK columns)
```

Note: Fresh databases will auto-create with BIGSERIAL from the updated schema.

---

## Verification

1. **Build**: Compile the server - type mismatches will cause compile errors
2. **Unit tests**: Run `bin/knottyyoga_tests` in the test container
3. **Integration test**:
   - Start fresh database with `clear-database.cmd`
   - Start server
   - Register a user, login, check session works
   - Verify role/permission assignment works
4. **Database check**: Verify columns are BIGINT with `\d+ <table_name>` in psql

---

## Key Files (in order of modification)

1. `sql_util/database_common.h` - enum definition
2. `sql_util/schema/column_info.cpp` - SQL type mapping
3. `sql_util/database_access/database_metadata.cpp` - reverse mapping
4. `sql_util/schema/database_info.cpp` - FK conversion
5. `sql_util/database_access/database_crud_helpers.h/cpp` - new function
6. All `db_schema/*.cpp` files - schema changes
7. All `sql_util/table_helpers/*.h/cpp` files - API changes
8. `auth/session.h/cpp` and `auth/person.h/cpp` - auth module
9. Endpoint and test files