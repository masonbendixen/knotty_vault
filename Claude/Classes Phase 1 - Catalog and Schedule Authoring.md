---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/23/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please use your built in tools for read only operations on the filesystem or just say yes but do NOT prompt me when performing work that only reads the filesystem. I want you to run to completion (putting questions to be answered) but DO NOT FUCKING PROMPT ME. Please leave this Overview alone and build the plan in the following sections.

Classes Phase 1 - Catalog and Schedule Authoring

Please use the code base and the document Classes, schedules, and attendance.md. Please use these documents and the code base for context as well:

- [[Payment Design Document]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Event Polish- Scheduling Should Have Items]]
- [[Multi seat and bundled products]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Product, Event, and Subscription Admin Portal]]
- [[Provider Portal]]
- [[Scheduled Jobs]]
- [[Vouchers and Refunds]]

For each document, please extract the corresponding section from [[Classes, schedules, and attendance]] and place the information from that section of the document here as well as expanded details for the implementation as well as checkboxes to mark off as we complete them. For each piece, please start with the lowest layer of the system moving to higher layers on the server (db schema, table helpers, other helpers, business logic, and endpoints) and then client side work. Make sure to test everything that you can possibly test especially changes to existing files with tests.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here

## Phase Summary

**Must-have core.** Admin can define a class (name, description, photo, defaults), bind it to a recurring schedule at a facility/room/instructor, materialize future instances into `event_sessions`, and have those classes browsable in a public catalog. No booking flow yet (Phase 2) and no skill / template / series gating yet (Phases 3 / 5 / 7).

**Prerequisites:**
- Existing `classes` table (already in DB, photo-supported)
- Existing `events`, `event_sessions`, `event_session_staffing`, `facilities`, `location_rooms`, `RecurringSessionHelper` infrastructure from [[Scheduling thin slice]]
- Existing product / pricing / permission infrastructure ([[Payment Design Document]])

**Outcome:**
- Admin can create a class definition and a `class_schedule` for it.
- Materializer creates `event_sessions` rows from a schedule + date range, idempotently and respecting room capacity.
- Public `/classes` route shows the catalog with photos.
- Public class detail page shows description + upcoming sessions + the assigned instructors.
- Admin "Class Schedule" page provides CRUD plus a "materialize through date X" action.

## Layering & Conventions

Lowest layer first per CLAUDE.md:

1. `db_schema/` — DDL + column-name constants.
2. `sql_util/table_helpers/` — CRUD wrappers via `DbCrud::*`; KeyValueTable in / out.
3. `business_logic/scheduling/` — domain helpers; no SQL of their own.
4. `endpoints/` — thin HTTP handlers (auth + parse + delegate + KVT-to-JSON).
5. Angular UI under `ui/src/app/`.
6. Admin metadata registrations in `database_helper/create_database.cpp` (all eleven steps from CLAUDE.md).
7. Tests at every layer.

**Testing rules (binding):**
- Every helper changed → test added in the same session.
- Every endpoint added under `endpoints/` (not `web_app.cpp`) → endpoint test.
- `ServerAccess.mock.spec.ts` updated when new `ServerAccess` methods land.
- Component specs for every component touched.
- Test files live next to their implementation in `src/`.
- All tables are pre-created at startup by `GlobalDatabaseTestSupport::SetupAllTables()` — do NOT call `MakePaymentTables`, `DbOps::CreateTable`, etc. in tests.
- Use `crow::query_string` for query-param handling in endpoint tests (not raw URL strings).
- `ThreadPool::Shutdown()` BEFORE the next DB read in any test that hits an endpoint queueing async work.
- Sync SQL before any `ThreadPool::Queue` inside a single transaction.

## 1. Pre-Coding Design Decisions

### 1.1 Taxonomy lock-in (resolved per parent doc P-3)
- [x] Workshops = `class_schedules` with `is_series=true` and length 1. Single code path for series + workshop. Standalone `event` Product kind stays for non-class one-offs (anniversary, party).
- [x] Recurring class instances live in `event_sessions` with `class_schedule_id` set; classic events keep `class_schedule_id = NULL`.

### 1.2 Attendance template ↔ booking semantics (resolved per parent doc §2.8)
- [x] Templates are aspirational fitness-planning only; they do NOT create bookings, do NOT consume `event_sessions.capacity`, do NOT touch the waitlist. Capacity accounting therefore lives on staff-check-in (Phase 8) and paid-booking flows (Phase 7 series, M-9 guest pass, M-12 intro workshop).
- [x] Phase 1 materializer creates `event_sessions` rows only — no implicit `bookings` writes.

### 1.3 Room conflict policy (resolved per parent doc P-4)
- [x] Materializer respects `location_rooms.concurrent_capacity`. Parallel `event_sessions` and `bookable_service_sessions` in the same room are permitted as long as combined concurrent occupancy ≤ capacity. Hard block only when combined occupancy at the conflicting time window would exceed capacity. Document any blocked materialization rows in the response.

## 2. Database Schema

### 2.1 Extend `classes` table
- [ ] In `db_schema/classes.h`, add column-name constants:
  - `kClassesDefaultCapacity`
  - `kClassesDefaultCancellationPolicyId`
  - `kClassesDefaultRoomTypeId`
  - `kClassesIsActive`
  - `kClassesCreatedUs`
  - `kClassesUpdatedUs`
- [ ] In `db_schema/classes.cpp`, extend `MakeClassesTable` DDL to add the columns as: `BIGINT NOT NULL DEFAULT 0`, `BIGINT NULL`, `BIGINT NULL`, `BOOLEAN NOT NULL DEFAULT TRUE`, `BIGINT NOT NULL DEFAULT 0`, `BIGINT NOT NULL DEFAULT 0` respectively.
- [ ] Index on `is_active` for the active-catalog query.

### 2.2 New `class_schedules` table
- [ ] New files `db_schema/class_schedules.h/.cpp` with:
  - `id BIGSERIAL PRIMARY KEY`
  - `class_id BIGINT NOT NULL REFERENCES classes(id)`
  - `facility_id BIGINT NOT NULL REFERENCES facilities(id)`
  - `location_room_id BIGINT NOT NULL REFERENCES location_rooms(id)`
  - `product_id BIGINT NOT NULL REFERENCES products(id)` — drives pricing / visibility / booking permission / cancellation policy
  - `recurrence_pattern TEXT NOT NULL CHECK (recurrence_pattern IN ('weekly','biweekly','custom'))`
  - `days_of_week TEXT NOT NULL` (comma-separated 0..6; e.g. "1,3" for Mon+Wed)
  - `start_time_minutes BIGINT NOT NULL` (minutes-after-local-midnight in facility TZ)
  - `duration_minutes BIGINT NOT NULL`
  - `effective_from_us BIGINT NOT NULL`
  - `effective_to_us BIGINT` NULL
  - `capacity BIGINT` NULL — overrides `classes.default_capacity` when set
  - `predecessor_class_schedule_id BIGINT` NULL — for SL-11 same-day sequencing (Phase 3 wires the check; column lives here from day 1)
  - `is_series BOOLEAN NOT NULL DEFAULT FALSE`
  - `series_start_date_us BIGINT` NULL
  - `series_end_date_us BIGINT` NULL
  - `series_min_attendees BIGINT` NULL
  - `series_min_by_us BIGINT` NULL
  - `series_min_not_met_policy TEXT` NULL `CHECK (... IN ('auto_cancel_refund','proceed','admin_decides'))`
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us BIGINT NOT NULL`
  - `updated_us BIGINT NOT NULL`
- [ ] Indexes on (`facility_id`, `is_active`), (`class_id`), (`product_id`).

### 2.3 Extend `event_sessions` table
- [ ] Add `class_schedule_id BIGINT` NULL `REFERENCES class_schedules(id)`.
- [ ] Add `class_id BIGINT` NULL `REFERENCES classes(id)` (denormalized convenience — saves a join in calendar query).
- [ ] Index on `class_schedule_id` for "all sessions for schedule X" lookups.
- [ ] Existing event / service session rows have both NULL — backwards-compatible.

### 2.4 Wire schema into the database init pipeline
- [ ] Update `make_database_info.cpp`: add `MakeClassSchedulesTable(databaseInfo)` after the `classes`, `facilities`, `location_rooms`, `products` table makers (FK order).
- [ ] Update `database_helper/create_database.cpp` `CreateTables()`: add `CreateTable(transaction, kClassSchedulesTable, ...)` in FK-respecting order.
- [ ] Update `db_schema/CMakeLists.txt` and `sql_util/table_helpers/CMakeLists.txt` for the new sources.

## 3. Table Helpers

### 3.1 New `TableHelpers::ClassSchedules`
- [ ] `sql_util/table_helpers/class_schedules.h/.cpp` and `class_schedules_test.cpp`.
- [ ] Methods built on top of `DbCrud`:
  - `int64_t AddClassSchedule(Transaction&, const KeyValueTable&)`
  - `KeyValueTable GetClassSchedule(Transaction&, int64_t id)`
  - `void UpdateClassSchedule(Transaction&, int64_t id, const KeyValueTable& updates)`
  - `void DeleteClassSchedule(Transaction&, int64_t id)`  ← soft (set `is_active=false`)
  - `KeyValueTableArray GetActiveSchedulesByFacility(Transaction&, int64_t facilityId)` — ORDER BY `start_time_minutes`, day-of-week
  - `std::optional<KeyValueTable> GetScheduleByEventSession(Transaction&, int64_t eventSessionId)` — join through `event_sessions.class_schedule_id`
  - `KeyValueTableArray GetSchedulesByClass(Transaction&, int64_t classId)`
  - `KeyValueTableArray GetSchedulesPotentiallyConflictingInRoom(Transaction&, int64_t roomId, int64_t startTimeMinutes, int64_t durationMinutes, days)` — used by the materializer's room-conflict check
- [ ] Tests (no fixtures, self-contained, transaction-aborted):
  - Insert + Get round-trip
  - Update soft-delete sets `is_active=false`
  - `GetActiveSchedulesByFacility` filters by facility AND `is_active`
  - `GetSchedulesPotentiallyConflictingInRoom` returns expected rows

### 3.2 Extend `TableHelpers::Classes`
- [ ] Surface new columns in reads and accept them in writes. Add a `GetActiveClasses(Transaction&)` query.
- [ ] Add tests for the new column round-trips.

### 3.3 Extend `TableHelpers::EventSessions`
- [ ] Surface `class_schedule_id` + `class_id` in `GetEventSession` / `GetEventSessionsByFacility` etc.
- [ ] Add `GetEventSessionsByClassSchedule(Transaction&, int64_t scheduleId)` — used by `ClassScheduleHelper.DeactivateClassSchedule` and by Phase 5 template auto-cancel sweep.
- [ ] Add `GetEventSessionsByClass(Transaction&, int64_t classId, int64_t fromUs, int64_t toUs)` — used by Phase 9 attendance history.
- [ ] Tests for new methods.

## 4. Business Logic (`business_logic/scheduling/`)

### 4.1 New `ClassScheduleHelper`
- [ ] Files: `class_schedule_helper.h`, `class_schedule_helper.cpp`, `class_schedule_helper_test.cpp`.
- [ ] Public methods:
  ```cpp
  struct CreateClassScheduleRequest {
      int64_t classId = 0;
      int64_t facilityId = 0;
      int64_t locationRoomId = 0;
      int64_t productId = 0;
      std::string recurrencePattern;          // "weekly" | "biweekly" | "custom"
      std::vector<int> daysOfWeek;            // 0=Sun..6=Sat
      int64_t startTimeMinutes = 0;
      int64_t durationMinutes = 0;
      int64_t effectiveFromUs = 0;
      std::optional<int64_t> effectiveToUs;
      std::optional<int64_t> capacityOverride;
      std::optional<int64_t> predecessorClassScheduleId;
      // series fields ignored unless isSeries=true
      bool isSeries = false;
      std::optional<int64_t> seriesStartDateUs;
      std::optional<int64_t> seriesEndDateUs;
      std::optional<int64_t> seriesMinAttendees;
      std::optional<int64_t> seriesMinByUs;
      std::optional<std::string> seriesMinNotMetPolicy;
  };
  struct CreateClassScheduleResult { int64_t scheduleId = 0; bool ok = false; std::string errorCode; };
  CreateClassScheduleResult CreateClassSchedule(Transaction&, const CreateClassScheduleRequest&);

  struct MaterializeRequest { int64_t scheduleId; int64_t throughDateUs; };
  struct MaterializeResult {
      std::vector<int64_t> createdSessionIds;
      std::vector<std::pair<int64_t, std::string>> skippedDates; // (dateUs, reason)
      int64_t alreadyMaterializedCount = 0;
  };
  MaterializeResult MaterializeFutureSessions(Transaction&, const MaterializeRequest&);

  bool DeactivateClassSchedule(Transaction&, int64_t scheduleId, bool cancelFutureSessions);
  bool EditClassSchedule(Transaction&, int64_t scheduleId, const KeyValueTable& updates, bool regenerateFuture);
  ```
- [ ] `CreateClassSchedule` validations (return descriptive `errorCode` strings):
  - `INVALID_CLASS` — class not active
  - `INVALID_FACILITY` / `INVALID_ROOM` — room must belong to the facility
  - `INVALID_PRODUCT` — product must exist and be of `class` kind
  - `INVALID_RECURRENCE` — pattern + days-of-week make sense (weekly requires ≥1 day; custom requires `interval_days` in a future enhancement)
  - `INVALID_TIME_BOUNDS` — duration > 0, start time within [0, 1440), effective_to > effective_from
  - `INVALID_SERIES_FIELDS` — when `is_series=true`, all series fields must be present and consistent
- [ ] `MaterializeFutureSessions` is the heart of the phase:
  - Load schedule; verify still active.
  - Compute the set of (date, start_us, end_us) tuples between `effective_from_us` and `min(throughDateUs, effective_to_us)` from `recurrence_pattern` + `days_of_week` + `start_time_minutes` + `duration_minutes`. Wrap existing `RecurringSessionHelper::GenerateSessionDates`.
  - For each (start_us, end_us): check `event_sessions` for an existing row with `class_schedule_id = scheduleId AND start_time_us = start_us AND status != 'cancelled'`. If present → `alreadyMaterializedCount++`, skip.
  - For each new candidate, run the **room concurrent-capacity check** (per P-4): query `event_sessions` + `bookable_service_sessions` overlapping that time window in the same room. If `Σ capacity` of overlapping sessions + this session's effective capacity > `location_rooms.concurrent_capacity`, append to `skippedDates` with reason `ROOM_OVERSUBSCRIBED` and continue.
  - Otherwise create the `event_sessions` row with `class_id`, `class_schedule_id`, `capacity = scheduleCapacityOverride.value_or(classDefault.capacity)`, `status = "scheduled"`, etc.
  - Returns `MaterializeResult` so admin UI can show created + skipped + already-materialized counts.
- [ ] `DeactivateClassSchedule` flips `is_active=false`; if `cancelFutureSessions`, iterates future-dated `event_sessions` and calls `SessionCancellationHelper::CancelSession` with reason "class schedule deactivated".
- [ ] `EditClassSchedule`: applies updates, then if `regenerateFuture`, deletes uncancelled future un-booked `event_sessions` and re-runs `MaterializeFutureSessions` from now through the previous materialization horizon. Bookings (when Phase 7+ adds them) are NEVER blown away — sessions with attached bookings are kept as-is and surfaced in the result so admin can manually reconcile.

### 4.2 New `ClassCatalogHelper`
- [ ] Files: `class_catalog_helper.h/.cpp` + test.
- [ ] Methods:
  - `std::vector<ClassCatalogEntry> GetActiveClasses(Transaction&)` — all active classes; no per-user filtering (public).
  - `std::vector<ClassCatalogEntry> GetClassesVisibleToPerson(Transaction&, int64_t personId)` — joins `classes` × at-least-one-active `class_schedule` × `products` × `product_prices`, resolves user's best tier price and inclusion status (M-4 / M-5 / M-7). Used by the public catalog when the visitor is logged in.
  - `std::optional<ClassDetail> GetClassDetail(Transaction&, int64_t classId, int64_t personId /*0 for anonymous*/)` — class info + photo URL + upcoming sessions (next N from `event_sessions` JOIN `class_schedule_id` IN `(active schedules of this class)`) + the assigned instructors per session via `event_session_staffing`.
- [ ] `ClassCatalogEntry` struct fields: classId, name, description, photoUrl, defaultCapacity, tags (empty until Phase 13), upcomingSessionCount.
- [ ] `ClassDetail` adds: full description, list of UpcomingSessionInfo (sessionId, startUs, endUs, facilityName, roomName, instructorName(s)), required-skill-list placeholder (empty until Phase 3), price-info (resolved tier price OR "included in your membership" / "non-member only sees workshop offerings").
- [ ] Tests cover anonymous, logged-in-with-membership-included, and logged-in-non-member cases.

### 4.3 Integration with existing `RecurringSessionHelper`
- [ ] No changes to `RecurringSessionHelper` core — wrap it.
- [ ] In `ClassScheduleHelper::MaterializeFutureSessions`, build a `RecurringSessionRequest` from the schedule row, but include the new `class_id` / `class_schedule_id` fields on each emitted session.
- [ ] Add `class_id` / `class_schedule_id` parameters to `RecurringSessionHelper::CreateRecurringSessions` (optional, default 0/NULL) so we don't fork the helper.

### 4.4 KeyValueTable conversions
- [ ] In `business_logic/scheduling/scheduling_key_value_table.h/cpp` (existing): add
  - `ClassScheduleToKeyValueTable(const ClassScheduleInfo&)`
  - `ClassCatalogEntryToKeyValueTable(...)`
  - `ClassDetailToKeyValueTable(...)`
  - `MaterializeResultToKeyValueTable(...)`
- [ ] Unit tests in `scheduling_key_value_table_test.cpp`.

### 4.5 Tests for business logic
- [ ] `class_schedule_helper_test.cpp`:
  - Create + materialize 4 weeks → expected number of sessions, expected day-of-week distribution
  - Re-materialize same range is idempotent (alreadyMaterializedCount equals first run's createdSessionIds.size, nothing new created)
  - Room-conflict check skips a session if a parallel session would push the room over `concurrent_capacity`
  - Room-conflict check ALLOWS parallel sessions when combined occupancy fits
  - `DeactivateClassSchedule(cancelFutureSessions=true)` cancels future sessions
  - `EditClassSchedule(regenerateFuture=true)` rebuilds only sessions without bookings (Phase 7+ test)
- [ ] `class_catalog_helper_test.cpp`:
  - Anonymous visitor sees only public-visibility classes
  - Logged-in member sees included classes flagged correctly
  - Non-member sees no recurring classes (only intro workshop / series / workshops — Phase 2/7 wiring)
- [ ] Remember `ThreadPool::Shutdown()` if any path queues async work (none in Phase 1; verify).

## 5. Endpoints (`endpoints/`)

### 5.1 Public catalog endpoints
- [ ] `endpoints/get_classes.h/cpp` + test:
  - `GET /api/classes` — query params: `page`, `limit`. Returns array of `ClassCatalogEntry` JSON. Uses `ClassCatalogHelper::GetActiveClasses` for anonymous, `GetClassesVisibleToPerson(personId)` for logged-in.
- [ ] `endpoints/get_class_detail.h/cpp` + test:
  - `GET /api/classes/<id>` — public. Returns `ClassDetail` JSON. Returns 404 (`ErrorResponse::NotFound`) if class is missing or `is_active=false`.

### 5.2 Admin schedule CRUD endpoints
- [ ] `endpoints/admin_class_schedule_create.h/cpp` + test:
  - `POST /api/admin/class_schedule`
  - Permission gate: new `manage_class_schedule` (Phase 1 introduces this; Phase 12 will refine). Fallback: `manage_products`.
  - Body: JSON matching `CreateClassScheduleRequest`. Validates, calls `ClassScheduleHelper::CreateClassSchedule`. Returns 200 + scheduleId, or 400 with `errorCode`.
- [ ] `endpoints/admin_class_schedule_update.h/cpp` + test:
  - `PUT /api/admin/class_schedule/<id>`. Body: KVT updates + `regenerateFuture` bool. Calls `EditClassSchedule`.
- [ ] `endpoints/admin_class_schedule_deactivate.h/cpp` + test:
  - `DELETE /api/admin/class_schedule/<id>?cancel_future=true|false`. Calls `DeactivateClassSchedule`.
- [ ] `endpoints/admin_class_schedule_materialize.h/cpp` + test:
  - `POST /api/admin/class_schedule/<id>/materialize`. Body: `{ "through_date_us": <int64> }`. Calls `MaterializeFutureSessions`. Returns `MaterializeResult` JSON (createdSessionIds + skippedDates + alreadyMaterializedCount).
- [ ] `endpoints/admin_class_schedules_list.h/cpp` + test:
  - `GET /api/admin/class_schedules?facility_id=<id>`. Calls `GetActiveSchedulesByFacility`.

### 5.3 Routing registration
- [ ] Register all six endpoints in `endpoints/web_app.cpp`.

### 5.4 Endpoint testing patterns
- [ ] All endpoint tests use `EndpointTestHelper` + `TestDatabaseUtil::RunInTransaction`.
- [ ] Use `crow::query_string` for query params (per memory `feedback_crow_query_params_test.md`).
- [ ] Verify permission-denied paths return 403 / `ErrorResponse::NotAuthorized` for non-admin callers.
- [ ] Verify ValidationError → 400 (per memory `error_response_status_codes.md`).
- [ ] No `ThreadPool::Queue` is invoked from any of these endpoints in Phase 1 → no `Shutdown()` dance required (verify by reading helper code).

## 6. Frontend (Angular)

### 6.1 Public class catalog page
- [ ] `ui/src/app/pages/classes/class-catalog/class-catalog.component.ts` + `.html` + `.scss` + `.spec.ts`. Already partially wired per public routes.
- [ ] Loads from `ServerAccess.getClasses()`.
- [ ] Grid layout: photo card, name, description-preview, "View details" CTA.
- [ ] Empty state for "no classes yet".
- [ ] Component spec verifies the rendering of a populated grid + empty state + click navigation.

### 6.2 Public class detail page
- [ ] `class-detail/class-detail.component.*` + spec.
- [ ] Loads from `ServerAccess.getClassDetail(id)`. Returns 404 routes to a "class not found" view.
- [ ] Sections: hero photo, name, description, upcoming sessions list, instructors-who-teach, required-skills placeholder (Phase 3 wires it), pricing-tier display.
- [ ] Spec verifies all sections render with mock data.

### 6.3 Admin Class Schedule page
- [ ] `ui/src/app/pages/portal/manage/class-schedules/` directory.
- [ ] `class-schedules-list.component.*` + spec: table view, filter by facility, "Edit", "Deactivate", "Materialize" actions per row.
- [ ] `class-schedule-edit.component.*` + spec: create + edit form. Fields: class (FK picker), facility, room (filtered by facility), product, recurrence pattern (radio: weekly/biweekly/custom), day-of-week toggles, start time picker, duration picker, effective date range, capacity override, optional series fields (collapsed unless `is_series` toggled). Reactive form with validation. Use the date-picker + hour-picker per memory `feedback_date_time_pickers.md`. Mat-card border per memory `feedback_mat_card_border.md`.
- [ ] "Materialize" dialog: date picker for `through_date_us`, calls `ServerAccess.materializeClassSchedule(id, throughDateUs)`. Displays result counts (created / skipped / already-materialized).
- [ ] Routing entry under `portal/manage` lazy-loaded.
- [ ] Specs include RouterTestingModule (per memory `feedback_account_page_layout.md`).

### 6.4 `ServerAccess` extensions
- [ ] Add to `ServerAccess` interface, `ServerAccessNetwork`, AND `ServerAccessMock`:
  - `getClasses(page?: number, limit?: number): Observable<ClassCatalogEntry[]>`
  - `getClassDetail(id: number): Observable<ClassDetail>`
  - `createClassSchedule(req): Observable<{ scheduleId: number }>`
  - `updateClassSchedule(id, updates): Observable<void>`
  - `deactivateClassSchedule(id, cancelFuture: boolean): Observable<void>`
  - `materializeClassSchedule(id, throughDateUs): Observable<MaterializeResult>`
  - `listClassSchedules(facilityId): Observable<ClassScheduleInfo[]>`
- [ ] Update `ServerAccess.mock.spec.ts` per memory `feedback_always_test.md`.

### 6.5 Type definitions
- [ ] `ui/src/app/shared/types/class.types.ts`: `ClassCatalogEntry`, `ClassDetail`, `UpcomingSessionInfo`, `ClassScheduleInfo`, `MaterializeResult`.

## 7. Admin Metadata (`database_helper/create_database.cpp`)

Forgetting steps 4 or 5 below is the most common mistake (per CLAUDE.md). All eleven steps:

- [ ] **Step 1** — Confirm `db_schema/class_schedules.h` constants are referenced.
- [ ] **Step 2** — `make_database_info.cpp` already updated in 2.4.
- [ ] **Step 3** — `CreateTables()` already updated in 2.4.
- [ ] **Step 4** — `PopulateAdminTopLevelTables()`: NOT in the top-level list — `class_schedules` is nested under `classes`.
- [ ] **Step 5** — `PopulateAdminNestedTables()`: add `class_schedules` as a nested child of `classes` keyed by `class_id`. Without this, the generic CRUD endpoints reject the table with "Table is not an allowed table".
- [ ] **Step 6** — `PopulateAdminTablePermissions()`: `class_schedules` requires `manage_class_schedule`. Also: ensure the existing `classes` row uses `manage_class_schedule` (NOT just `manage_products`) so studio managers can edit class definitions.
- [ ] **Step 7** — `PopulateAdminColumnDataInfo()`: column edit types for the new columns (FKs for `class_id`/`facility_id`/`location_room_id`/`product_id`, time-of-day editor for `start_time_minutes`, JSON multi-select for `days_of_week`, etc.).
- [ ] **Step 8** — `PopulateAdminColumnFriendlyNames()`: friendly headers (e.g. "Class", "Facility", "Room", "Recurrence", "Days of Week", "Start Time", "Duration (min)").
- [ ] **Step 9** — `PopulateAdminTableFriendlyNames()`: "Class Schedules".
- [ ] **Step 10** — `PopulateAdminTableDisplayTemplates()`: FK picker for `class_schedules` shows "{class_name} @ {facility_name} {start_time}" format.
- [ ] **Step 11** — `CMakeLists.txt` for `db_schema/` and `sql_util/table_helpers/`.

## 8. Permissions

- [ ] Add `manage_class_schedule` permission seed in `create_database.cpp`.
- [ ] Add a `Studio Manager` role (or extend existing) to have `manage_class_schedule`.
- [ ] Admin role already has the master permission set; no change.

## 9. Seed Data

- [ ] In `create_database.cpp`, seed two demo classes (e.g., "Vinyasa Flow" + "Aerial 101") with photos to be uploaded later.
- [ ] Seed one demo `class_schedule` for "Vinyasa Flow" so a fresh DB shows something on the calendar.
- [ ] Seed a default cancellation policy (if not present already from earlier phases).

## 10. Tests Summary

- [ ] Table helpers: `class_schedules_test.cpp`, extended `event_sessions_test.cpp`, extended `classes_test.cpp`.
- [ ] Business logic: `class_schedule_helper_test.cpp`, `class_catalog_helper_test.cpp`, extended `scheduling_key_value_table_test.cpp`.
- [ ] Endpoint tests: one per endpoint, success + permission-denied + validation-error paths.
- [ ] Frontend: component specs for `class-catalog`, `class-detail`, `class-schedules-list`, `class-schedule-edit`, materialize dialog.
- [ ] `ServerAccess.mock.spec.ts` updated.
- [ ] Manual-testing-helper command added (if useful): `list_class_schedules`, `materialize_schedule <id> <through_date>` — see [[Manual Testing Helper Executable]] for the pattern.

## 11. Cross-Layer Acceptance Criteria

A new admin user, starting from a fresh DB built by the `knottyyoga_database_helper`, can:
- [ ] Open `portal/manage/class-schedules`, create a "Vinyasa Flow" schedule (Mon/Wed 6pm, 60min, Studio A) effective today through six months out.
- [ ] Click "Materialize through" and pick a date 8 weeks from today; see exactly N sessions created (where N = 8 × days-per-week from the recurrence), zero skipped, zero already-materialized.
- [ ] Re-run the same materialize; see the previous N already-materialized, zero new.
- [ ] Open the public `/classes` route as a logged-out visitor and see "Vinyasa Flow" with photo.
- [ ] Click into class detail and see eight Mondays + eight Wednesdays of upcoming sessions.

## 12. Open Questions

- **OQ-P1-1.** For the `days_of_week` column, use comma-separated string ("1,3,5") or a Postgres `INTEGER[]`? Recommended: comma-separated TEXT for portability and to match how other small enum-set columns are stored in the codebase (verify by grep). Pure storage choice; no behavior impact.
- **OQ-P1-2.** Should `predecessor_class_schedule_id` validation reject same-day chains longer than 2 (so we can't accidentally build "hour 1 → hour 2 → hour 3 → hour 4")? Recommended: no cap for now; if it becomes a hygiene issue, revisit.
- **OQ-P1-3.** When `EditClassSchedule(regenerateFuture=true)` encounters future sessions that have bookings (Phase 7+), should the editor (a) reject the edit, (b) accept the edit but leave the bookable sessions untouched, or (c) cascade-cancel-and-refund? Recommended (b): leave touched sessions as-is, surface them in the response; admin can manually reconcile.

## 13. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 1.
- Predecessor work: [[Scheduling thin slice]], [[Provider Portal]].
- Will feed into: [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 5 - Attendance Templates]], [[Classes Phase 7 - Class Series and Workshops]].
