---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/5/2026
Version: 0.1
tags: 
---
# Overview

Please go into plan mode. Please use this document for your planning document. Do not create anything in .claude/plans. Use this file for planning and do not ask me for permission to modify this file. It is your plan file. Please leave this Overview section in tact and do your work in the sections below.

We need to add instructors to the system with photo support. We need to create a new role called instructor and a permission called instructor. We need to add a table for instructors called instructors. This should have an auto incrementing 64bit serial counter for the primary key called id. We should have a foreign key reference to a person since all instructors are people. We need another column called bio that is a varchar. This should be a table that shows up in the admin console as a nested table under people. It should also have photo support. The primary key should be readonly and the bio should map a long text html control.

We need a public endpoint to enumerate the instructors with first name, last name, and an id for the photo to be able to request a scaled photo. This should be callable to all users and called get_instructors.

I want to add Instructors to the About dropdown in the main menu. It should bring up a page that says Instructors and then has a set of cards for each instructor with a decent sized photo to the left (maybe 256x256 pixels), their full name in larger print, and then their bio below their name with normal text.

Please use the codebase and this document to generate your plan. Please create phases of implementation with check boxes next to them. Please do the layered architecture with database schema changes, CRUD table helpers, business logic changes (this could probably be added to PersonHelper), the endpoint. Then the client stuff with the types, network access layer, components, and then wiring into the system.

# Implementation Plan

## Phase 1: Database Schema Layer

### 1.1 Create the `instructors` table definition
- [x] Create `server/knottyyoga_server/src/db_schema/instructors.h`
  - Constants: `kInstructorsTable = "instructors"`, `kInstructorsId = "id"`, `kInstructorsPersonId = "person_id"`, `kInstructorsBio = "bio"`
  - Declare `MakeInstructorsTable(DatabaseInfo databaseInfo)`
- [x] Create `server/knottyyoga_server/src/db_schema/instructors.cpp`
  - `AddTable(kInstructorsTable)`
  - `AddColumnPrimaryKey(kInstructorsTable, kInstructorsId, DB_TYPE_BIGSERIAL)` — auto-incrementing 64-bit serial
  - `AddColumnForeignKeyRef(kPeopleTable, kPeopleId, kInstructorsTable, kInstructorsPersonId)` — FK to people
  - `AddColumnNullable(kInstructorsTable, kInstructorsBio, DB_TYPE_STRING)` — varchar bio column
- [x] Add both files to `db_schema/CMakeLists.txt`

### 1.2 Register the table in `make_database_info.cpp`
- [x] Add `#include "instructors.h"` and call `MakeInstructorsTable(databaseInfo)` in `MakeDatabaseInfo()`

### 1.3 Add role and permission constants
- [x] Add `constexpr std::string_view kRoleNameInstructor = "instructor"` to `roles.h`
- [x] *(No new permission constant needed — we'll add the "instructor" permission as seed data in `create_database.cpp`)*

### 1.4 Update `create_database.cpp` — table creation and seed data
- [x] Add `#include "db_schema/instructors.h"` to includes
- [x] Add `CreateTable(DbSchema::kInstructorsTable)` in `CreateTables()` (after people, since it has a FK to people)
- [x] Add `instructors` to `PopulateAdminNestedTables()` — this makes it a nested table under people in the admin console
- [x] Add `instructors` to `PopulatePhotoSupportTables()` — enables photo upload/display for instructor rows
- [x] Add column data info for instructors in `PopulateAdminColumnDataInfo()`:
  - `id` column: readonly (readonly param = "true")
  - `bio` column: htmlInputType = `"long-text-html"`, required = `"false"`
- [x] Add column friendly names in `PopulateAdminColumnFriendlyNames()`:
  - `id` → "ID", `person_id` → "Person", `bio` → "Bio"
- [x] Add table friendly name in `PopulateAdminTableFriendlyNames()`:
  - `instructors` → "Instructors", "Studio instructors with bios and photos."
- [x] Add display template in `PopulateAdminTableDisplayTemplates()`:
  - `instructors` → `"{person_id}"` (will resolve to person name via FK display)
- [x] Add `"instructor"` role in `PopulateRoles()`:
  - `AddRow(DbSchema::kRoleNameInstructor, "Instructor role for people who teach classes.")`
- [x] Add `"instructor"` permission in `PopulatePermissions()`:
  - `AddRow("instructor", "Permission identifying instructors.")`

---

## Phase 2: Table Helpers (CRUD Layer)

### 2.1 Create the `instructors` table helper
- [x] Create `server/knottyyoga_server/src/sql_util/table_helpers/instructors.h`
  - Class `Instructors` with constructor taking `DatabaseHelper`
  - Methods:
    - `int64_t AddInstructor(Transaction&, int64_t personId, std::string_view bio)` — returns new ID
    - `std::optional<KeyValueTable> GetInstructorById(Transaction&, int64_t id)`
    - `std::optional<KeyValueTable> GetInstructorByPersonId(Transaction&, int64_t personId)`
    - `KeyValueTableArray GetAllInstructors(Transaction&)` — returns all rows
    - `void UpdateInstructorBio(Transaction&, int64_t id, std::string_view bio)`
    - `void DeleteInstructor(Transaction&, int64_t id)`
- [x] Create `server/knottyyoga_server/src/sql_util/table_helpers/instructors.cpp`
  - Implement using `DbCrud::AddRowToTableFetchInt64PrimaryKey`, `DbCrud::LookupRowByValue`, `DbCrud::GetTableRows`, `DbCrud::UpdateRow`, `DbCrud::DeleteRow` (following patterns from existing helpers like `permissions.cpp`)
- [x] Add both files to `sql_util/table_helpers/CMakeLists.txt`

### 2.2 Create tests for the table helper
- [x] Create `server/knottyyoga_server/src/sql_util/table_helpers/instructors_test.cpp`
  - Tests for AddInstructor, GetInstructorById, GetInstructorByPersonId, GetAllInstructors, UpdateInstructorBio, DeleteInstructor
  - Follow existing pattern: use `testDb.MakeTestPeopleTable()` for FK dependency, then create instructors table
- [x] Add test file to test `CMakeLists.txt`

---

## Phase 3: Business Logic Layer

### 3.1 Create instructor helper in `business_logic/auth/`
- [x] Create `server/knottyyoga_server/src/business_logic/auth/instructor_helper.h`
  - Struct `InstructorInfo`:
    - `int64_t instructorId`
    - `int64_t personId`
    - `std::string firstName`
    - `std::string lastName`
    - `std::string bio`
    - `bool hasPhoto`
  - Class `InstructorHelper`:
    - Constructor taking `DatabaseHelper`
    - `std::vector<InstructorInfo> GetInstructorsForPublicDisplay(Transaction&)` — joins instructors + people to get names, checks photo existence
- [x] Create `server/knottyyoga_server/src/business_logic/auth/instructor_helper.cpp`
  - Implementation: query all instructors, for each one look up the person row for first/last name, check `ImageHelper::HasPhoto` for the instructors table
- [x] Add both files to `business_logic/auth/CMakeLists.txt`

### 3.2 Create instructor key-value table conversion
- [x] Create `server/knottyyoga_server/src/business_logic/auth/instructor_key_value_table.h`
  - `KeyValueTable InstructorInfoToKeyValueTable(const InstructorInfo&)`
  - `std::vector<KeyValueTable> InstructorInfosToKeyValueTableArray(const std::vector<InstructorInfo>&)`
- [x] Create `server/knottyyoga_server/src/business_logic/auth/instructor_key_value_table.cpp`
  - Maps fields: `instructor_id`, `person_id`, `first_name`, `last_name`, `bio`, `has_photo`
- [x] Add both files to `business_logic/auth/CMakeLists.txt`

### 3.3 Create tests
- [x] Create `server/knottyyoga_server/src/business_logic/auth/instructor_helper_test.cpp`
  - Tests: empty, basic (single instructor with name/bio/no photo), multiple instructors, empty bio
- [x] Create `server/knottyyoga_server/src/business_logic/auth/instructor_key_value_table_test.cpp`
  - Tests: basic conversion, no-photo flag, array conversion, empty array
- [x] Add test files to `business_logic/auth/CMakeLists.txt`

---

## Phase 4: Endpoint Layer

### 4.1 Create the `get_instructors` endpoint
- [x] Create `server/knottyyoga_server/src/endpoints/get_instructors.h`
  - Declare `Json::Value GetInstructors(EndpointAuthHelper&, const crow::request&, crow::response&)`
- [x] Create `server/knottyyoga_server/src/endpoints/get_instructors.cpp`
  - Route: `GET /api/get_instructors`
  - No auth required (public endpoint)
  - Calls `InstructorHelper::GetInstructorsForPublicDisplay()`
  - Converts to JSON via `InstructorInfoToKeyValueTable` → `SqlUtil::KeyValueTableToJson`
  - Returns `{ "items": [ { instructor_id, person_id, first_name, last_name, bio, has_photo }, ... ] }`
- [x] Register in `web_app.cpp`:
  - Add `#include "get_instructors.h"`
  - Add `auto g_GetInstructors = &Endpoints::GetInstructors;`
- [x] Add `.h` and `.cpp` to `endpoints/CMakeLists.txt`

### 4.2 Create endpoint tests
- [x] Create `server/knottyyoga_server/src/endpoints/get_instructors_test.cpp`
  - Test: returns empty array when no instructors
  - Test: returns instructors with name and bio data
  - Test: has_photo flag is correct
- [x] Add test file to test `CMakeLists.txt`

---

## Phase 5: Frontend — Types and Network Layer

### 5.1 Add instructor types
- [x] Create `ui/src/app/shared/types/instructor.types.ts`
  ```typescript
  export interface Instructor {
    instructor_id: number;
    person_id: number;
    first_name: string;
    last_name: string;
    bio: string;
    has_photo: boolean;
  }
  ```

### 5.2 Add `getInstructors()` to the ServerAccess interface
- [x] Update `ui/src/app/shared/types/ServerAccess.ts`:
  - Add import for `Instructor`
  - Add method: `getInstructors(): Observable<Instructor[]>`
  - Add re-export for `Instructor` type

### 5.3 Implement in ServerAccessNetwork
- [x] Update `ui/src/app/shared/services/network/ServerAccessNetwork.ts`:
  - Add `getInstructors(): Observable<Instructor[]>` — `GET /api/get_instructors`, maps `response.items`

### 5.4 Implement in ServerAccessMock
- [x] Update `ui/src/app/shared/services/network/ServerAccess.mock.ts`:
  - Add mock instructor data (a couple of sample instructors)
  - Implement `getInstructors()` returning the mock data

### 5.5 Update ServerAccessProxy
- [x] Update `ui/src/app/shared/services/network/ServerAccess.ts`:
  - Add `getInstructors()` delegation to implementation

---

## Phase 6: Frontend — Instructors Page Component

### 6.1 Create the Instructors component
- [x] Create directory `ui/src/app/pages/public/instructors/`
- [x] Create `instructors.component.ts`:
  - Standalone component importing `SharedModule`, `CommonModule`
  - Inject `ServerAccess` via `SERVER_ACCESS_TOKEN`
  - On init, call `getInstructors()` and store result
  - Build photo URLs: `/api/get_scaled_photo/instructors/{instructor_id}/256/256`
  - Fallback for instructors without photos (generic avatar/placeholder)
- [x] Create `instructors.component.html`:
  - Page title: "Instructors"
  - For each instructor, a card with:
    - Photo on the left (256x256), using the scaled photo endpoint
    - Name in larger text (e.g., `text-xl font-semibold`)
    - Bio below the name in normal text
    - Use `innerHTML` binding for bio since it's HTML content (`long-text-html` control)
  - Responsive layout: cards stack on mobile
- [x] Create `instructors.component.scss`:
  - Card border styling (per project convention: `border: 1px solid #d1d5db`)
  - Photo sizing and spacing
  - Responsive breakpoints

### 6.2 Register the route
- [x] Update `ui/src/app/pages/public/public.routes.ts`:
  - Add import for `InstructorsComponent`
  - Add route: `{ path: 'instructors', component: InstructorsComponent }`

### 6.3 Add to the About dropdown in navigation
- [x] Update `ui/src/app/shared/services/header/mockHeaderResponse.ts`:
  - Add to the `aboutDropdown.menu` array:
    ```typescript
    {
      title: 'Instructors',
      kind: HeaderButtonKind.InternalLink,
      goTo: '/instructors',
    },
    ```

### 6.4 Update the mobile menu
- [x] Check `header-mobile-menu.component.html` — the mobile menu uses the same `headerData.menu` structure, so the About dropdown addition automatically appears in both desktop and mobile.

---

## Phase 7: Testing and Polish

- [ ] Run backend tests (`knottyyoga_tests`) to verify all new tests pass
- [ ] Run frontend tests (`ng test`) to verify no regressions
- [ ] Run `ng serve -c local` to verify the instructors page renders with mock data
- [ ] Verify the admin console shows instructors as a nested table under people
- [ ] Verify photo upload works for instructor rows in the admin console
- [ ] Verify the public instructors page displays photos via the scaled photo endpoint

---

## Key Files Modified (Summary)

### New Files (Server)
| File | Purpose |
|------|---------|
| `db_schema/instructors.h/cpp` | Table definition |
| `sql_util/table_helpers/instructors.h/cpp` | CRUD helpers |
| `sql_util/table_helpers/instructors_test.cpp` | CRUD tests |
| `business_logic/auth/instructor_helper.h/cpp` | Business logic |
| `business_logic/auth/instructor_helper_test.cpp` | Business logic tests |
| `business_logic/auth/instructor_key_value_table.h/cpp` | KVT conversion |
| `business_logic/auth/instructor_key_value_table_test.cpp` | KVT tests |
| `endpoints/get_instructors.h/cpp` | Public endpoint |
| `endpoints/get_instructors_test.cpp` | Endpoint tests |

### Modified Files (Server)
| File | Change |
|------|--------|
| `db_schema/roles.h` | Add `kRoleNameInstructor` constant |
| `db_schema/make_database_info.cpp` | Register instructors table |
| `db_schema/CMakeLists.txt` | Add instructors files |
| `database_helper/create_database.cpp` | Table creation + all seed data |
| `sql_util/table_helpers/CMakeLists.txt` | Add instructors files |
| `business_logic/auth/CMakeLists.txt` | Add instructor helper files |
| `endpoints/CMakeLists.txt` | Add endpoint files |
| `endpoints/web_app.cpp` | Register endpoint |

### New Files (Frontend)
| File | Purpose |
|------|---------|
| `shared/types/instructor.types.ts` | TypeScript interface |
| `pages/public/instructors/instructors.component.ts` | Component logic |
| `pages/public/instructors/instructors.component.html` | Template |
| `pages/public/instructors/instructors.component.scss` | Styles |

### Modified Files (Frontend)
| File | Change |
|------|--------|
| `shared/types/ServerAccess.ts` | Add `getInstructors()` |
| `shared/services/network/ServerAccessNetwork.ts` | HTTP implementation |
| `shared/services/network/ServerAccess.mock.ts` | Mock implementation |
| `shared/services/network/ServerAccess.ts` | Proxy delegation |
| `pages/public/public.routes.ts` | Add `/instructors` route |
| `shared/services/header/mockHeaderResponse.ts` | Add to About dropdown |

---

## Notes and Design Decisions

1. **Photo association**: Photos are associated with the `instructors` table (not `people`). This means the scaled photo URL will be `/api/get_scaled_photo/instructors/{instructor_id}/256/256`. This is intentional — an instructor's public-facing photo may differ from their personal account photo.

2. **Bio as HTML**: The bio uses `long-text-html` in the admin console, allowing rich text formatting. On the public page, we use Angular's `[innerHTML]` binding with DomSanitizer to safely render the HTML content.

3. **No auth on endpoint**: `get_instructors` is public — anyone visiting the site can see the instructor page. The endpoint joins across tables to return denormalized data (names from people, bio from instructors, photo existence check).

4. **Nested under people**: Making instructors a nested table in the admin console means when an admin views a person, they can also see/manage instructor records for that person, including uploading a photo.

5. **Role vs Permission**: We add both an "instructor" role and an "instructor" permission. The role can be assigned to people who are instructors, and the permission can be used for access control (e.g., teacher-only features). These follow the existing pattern where roles like "admin" have corresponding permissions like "admin_portal".