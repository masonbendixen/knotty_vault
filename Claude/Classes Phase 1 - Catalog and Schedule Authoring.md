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
- [x] In `db_schema/classes.h`, add column-name constants:
  - `kClassesDefaultCapacity`
  - `kClassesDefaultCancellationPolicyId`
  - `kClassesDefaultRoomTypeId`
  - `kClassesIsActive`
  - `kClassesCreatedUs`
  - `kClassesUpdatedUs`
- [x] In `db_schema/classes.cpp`, extend `MakeClassesTable` DDL to add the columns. `default_capacity` is `BIGINT NOT NULL DEFAULT 0`, `default_cancellation_policy_id` and `default_room_type_id` are nullable `BIGINT` (kept plain so they don't have to be added after their would-be parent tables in the builder; validation lives at the application layer), `is_active` is `BOOLEAN NOT NULL DEFAULT TRUE`, `created_us` and `updated_us` are `BIGINT NOT NULL DEFAULT now_us()`.
- [x] Index on `is_active` for the active-catalog query. Added via `DbSchema::CreateClassesIndexes(transaction)` (raw `CREATE INDEX IF NOT EXISTS`) since the metadata schema builder does not model indexes.

### 2.2 New `class_schedules` table
- [x] New files `db_schema/class_schedules.h/.cpp` with the full column set:
  - `id BIGSERIAL PRIMARY KEY`
  - `class_id BIGINT NOT NULL` (FK → `classes(id)` via `AddColumnForeignKeyRef`)
  - `facility_id BIGINT NOT NULL` (FK → `facilities(id)`)
  - `location_room_id BIGINT NOT NULL` (FK → `location_rooms(id)`)
  - `product_id BIGINT NOT NULL` (FK → `products(id)`) — drives pricing / visibility / booking permission / cancellation policy
  - `recurrence_pattern TEXT NOT NULL` (CHECK constraint enforced in business logic; metadata builder does not expose CHECK)
  - `days_of_week TEXT NOT NULL` (comma-separated 0..6; e.g. "1,3" for Mon+Wed) — see resolved OQ-P1-1
  - `start_time_minutes BIGINT NOT NULL` (minutes-after-local-midnight in facility TZ)
  - `duration_minutes BIGINT NOT NULL`
  - `effective_from_us BIGINT NOT NULL`
  - `effective_to_us BIGINT` NULL
  - `capacity BIGINT` NULL — overrides `classes.default_capacity` when set
  - `predecessor_class_schedule_id BIGINT` NULL — plain nullable BIGINT (no self-FK so deletes don't cascade through chains); Phase 3 validates the reference at write time. No chain-length cap — see resolved OQ-P1-2
  - `is_series BOOLEAN NOT NULL DEFAULT FALSE`
  - `series_start_date_us BIGINT` NULL
  - `series_end_date_us BIGINT` NULL
  - `series_min_attendees BIGINT` NULL
  - `series_min_by_us BIGINT` NULL
  - `series_min_not_met_policy TEXT` NULL (CHECK enforced in business logic)
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us BIGINT NOT NULL DEFAULT now_us()`
  - `updated_us BIGINT NOT NULL DEFAULT now_us()`
- [x] Indexes on (`facility_id`, `is_active`), (`class_id`), (`product_id`) added via `DbSchema::CreateClassSchedulesIndexes(transaction)`.

### 2.3 Extend `event_sessions` table
- [x] Add `class_schedule_id BIGINT` NULL `REFERENCES class_schedules(id)` via `AddColumnForeignKeyRefNullable`.
- [x] Add `class_id BIGINT` NULL `REFERENCES classes(id)` via `AddColumnForeignKeyRefNullable` (denormalized convenience — saves a join in calendar query).
- [x] Index on `class_schedule_id` for "all sessions for schedule X" lookups, added via `DbSchema::CreateEventSessionsIndexes(transaction)`.
- [x] Existing event / service session rows have both NULL — backwards-compatible.

### 2.4 Wire schema into the database init pipeline
- [x] Update `make_database_info.cpp`: added `MakeClassSchedulesTable(databaseInfo)` right after `MakeCancellationPolicyWindowsTable` and just before `MakeEventSessionsTable` — at that point `classes`, `facilities`, `location_rooms`, and `products` are all already in the builder.
- [x] Update `database_helper/create_database.cpp` `CreateTables()`: added `CreateTable(DbSchema::kClassSchedulesTable)` immediately before `kEventSessionsTable`, plus `CreateClassesIndexes`, `CreateClassSchedulesIndexes`, and `CreateEventSessionsIndexes` calls.
- [x] Update `db_schema/CMakeLists.txt` with `class_schedules.h/.cpp`. `sql_util/table_helpers/CMakeLists.txt` is unchanged for this phase — no table helpers added yet (those land in Phase 3 of the plan).

## 3. Table Helpers

### 3.1 New `TableHelpers::ClassSchedules`
- [x] `sql_util/table_helpers/class_schedules.h/.cpp` and `class_schedules_test.cpp`.
- [x] Methods built on top of `DbCrud`:
  - `int64_t AddClassSchedule(Transaction&, const KeyValueTable&)`
  - `KeyValueTable GetClassSchedule(Transaction&, int64_t id)`
  - `KeyValueTableArray GetClassSchedules(Transaction&)` (added for symmetry with other helpers)
  - `void UpdateClassSchedule(Transaction&, int64_t id, const KeyValueTable& updates)` — bumps `updated_us` automatically
  - `void SetIsActive(Transaction&, int64_t id, bool isActive)`
  - `void SoftDeleteClassSchedule(Transaction&, int64_t id)` — sets `is_active=false` (the plan's intended "Delete")
  - `void DeleteClassSchedule(Transaction&, int64_t id)` — hard delete; retained for the rare admin "really delete this empty schedule" path and test cleanup
  - `KeyValueTableArray GetActiveSchedulesByFacility(Transaction&, int64_t facilityId)` — `WHERE facility_id = $1 AND is_active = true ORDER BY start_time_minutes ASC, days_of_week ASC, id ASC`
  - `std::optional<KeyValueTable> GetScheduleByEventSession(Transaction&, int64_t eventSessionId)` — INNER JOIN through `event_sessions.class_schedule_id`; `nullopt` when unlinked or unknown
  - `KeyValueTableArray GetSchedulesByClass(Transaction&, int64_t classId)` — returns active + inactive, sorted `is_active DESC, start_time_minutes ASC, id ASC`
  - `KeyValueTableArray GetSchedulesPotentiallyConflictingInRoom(Transaction&, int64_t roomId, int64_t startTimeMinutes, int64_t durationMinutes, const std::vector<int>& daysOfWeek)` — narrows by `location_room_id = $1 AND is_active = true`; when `daysOfWeek` is non-empty it adds a Postgres regex match against the comma-wrapped `days_of_week` column (`",(1|3),"`) so "1" cannot substring-match "10" or "11". Time-window overlap math is left to the materializer where it has the per-day start/end.
- [x] Tests (no fixtures, self-contained, transaction-aborted) covering:
  - Insert + Get round-trip (`AddAndGet`, `AddWithSeriesFields`, `GetClassScheduleNotFound`)
  - Update bumps `updated_us` (`UpdateUpdatesUpdatedUs`)
  - Soft + hard delete (`SoftDeleteSetsIsActiveFalse`, `HardDeleteRemovesRow`)
  - Facility filter + is_active filter + sort by `start_time_minutes` (`GetActiveSchedulesByFacilityFiltersOnFacility`, `GetActiveSchedulesByFacilityExcludesInactive`, `GetActiveSchedulesByFacilityOrderedByStartTime`)
  - `GetSchedulesByClass` returns active + inactive, sorted active-first (`GetSchedulesByClassReturnsActiveAndInactive`)
  - `GetScheduleByEventSession` join + nullopt for unlinked / unknown (`GetScheduleByEventSessionJoinsThrough`, `GetScheduleByEventSessionReturnsNulloptForUnlinked`, `GetScheduleByEventSessionReturnsNulloptForUnknown`)
  - `GetSchedulesPotentiallyConflictingInRoom` with empty filter, with day filter, substring-trap defense (1 vs 11), and `is_active=false` excluded (`ConflictsInRoomEmptyDaysReturnsAllActiveInRoom`, `ConflictsInRoomDayFilterMatchesAsWholeToken`, `ConflictsInRoomDayFilterRejectsSubstringMatch`, `ConflictsInRoomExcludesInactive`)
  - `SetIsActive` round-trips (`SetIsActiveToggles`)

### 3.2 Extend `TableHelpers::Classes`
- [x] New `sql_util/table_helpers/classes.h/.cpp` (no prior helper existed). Surfaces the new columns on read via `GetClass`/`GetClasses` (`SELECT *`), accepts them on write via the full-KVT `AddClass(Transaction&, const KeyValueTable&)` overload, and adds `GetActiveClasses` (`WHERE is_active = true ORDER BY name ASC, id ASC`). Targeted setters: `SetName`, `SetDescription`, `SetDefaultCapacity`, `SetIsActive`. `UpdateClass` bumps `updated_us`.
- [x] `classes_test.cpp` round-trips every new column (`AddWithAllNewColumns`), covers `GetActiveClasses` filtering (`GetActiveClassesFiltersInactive`, `SetIsActiveFalseExcludesFromActive`), targeted setters, and not-found / delete-not-found paths.

### 3.3 Extend `TableHelpers::EventSessions`
- [x] `class_schedule_id` and `class_id` come back from `GetEventSession` / `GetEventSessions` automatically because those queries are `SELECT *` and the new DDL columns sit on the row. Test `GetEventSessionExposesClassScheduleId` verifies they round-trip and `ClassFieldsNullForClassicEventSession` verifies they come back empty for non-class rows.
- [x] Added `GetEventSessionsByClassSchedule(Transaction&, int64_t scheduleId)` — `WHERE class_schedule_id = $1 ORDER BY start_time_us ASC, id ASC`. Tests: `GetEventSessionsByClassScheduleReturnsLinkedRows`, `GetEventSessionsByClassScheduleOrderedByStartTime`, `GetEventSessionsByClassScheduleEmptyWhenNoLinkedRows`.
- [x] Added `GetEventSessionsByClass(Transaction&, int64_t classId, int64_t fromUs, int64_t toUs)` — half-open `[fromUs, toUs)` window, `ORDER BY start_time_us ASC, id ASC`. Tests: `GetEventSessionsByClassFiltersByWindow`, `GetEventSessionsByClassExcludesOtherClassesAndUnlinked`.

## 4. Business Logic (`business_logic/scheduling/`)

### 4.1 New `ClassScheduleHelper`
- [x] Files: `class_schedule_helper.h`, `class_schedule_helper.cpp`, `class_schedule_helper_test.cpp`.
- [x] Public methods — landed as documented below, with `kClassScheduleError*` `string_view` constants exported from `class_schedule_helper.h` so endpoints and tests share the error vocabulary instead of stringly-typing. `Materialize*` and `Edit*` result structs gained `ok` + `errorCode` fields so callers can branch off a single status value:
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
- [x] `CreateClassSchedule` validations all return descriptive `errorCode` strings, covered by dedicated tests:
  - `INVALID_CLASS` — class not active or unknown (tests: `CreateRejectsInactiveClass`, `CreateRejectsUnknownClass`)
  - `INVALID_FACILITY` — facility unknown (test: `CreateRejectsUnknownFacility`)
  - `INVALID_ROOM` — room unknown or belongs to a different facility (test: `CreateRejectsRoomFromWrongFacility`)
  - `INVALID_PRODUCT` — product missing or `kind != "class"` (test: `CreateRejectsNonClassProduct`)
  - `INVALID_RECURRENCE` — pattern not one of `weekly`/`biweekly`/`custom`, or weekly/biweekly without ≥1 day, or any day not in 0..6 (tests: `CreateRejectsInvalidRecurrencePattern`, `CreateRejectsWeeklyWithNoDays`, `CreateRejectsOutOfRangeDayOfWeek`). `custom` does not require `interval_days` yet — future enhancement, materializer is a no-op for now.
  - `INVALID_TIME_BOUNDS` — duration ≤ 0, start time not in `[0, 1440)`, or `effective_to <= effective_from` (tests: `CreateRejectsZeroDuration`, `CreateRejectsStartTimeOutOfRange`, `CreateRejectsEffectiveToBeforeFrom`)
  - `INVALID_SERIES_FIELDS` — when `is_series=true`, all series fields must be present, `series_end >= series_start`, `series_min_attendees >= 0`, and `series_min_not_met_policy` must be a valid enum value (tests: `CreateRejectsSeriesMissingFields`, `CreateAcceptsValidSeries`, `CreateRejectsInvalidSeriesPolicy`)
  - `INVALID_PREDECESSOR` — added beyond the original plan since the predecessor column exists from day 1; rejects unknown / inactive references (tests: `CreateRejectsInvalidPredecessor`, `CreateAcceptsValidPredecessor`)
- [x] `MaterializeFutureSessions` is the heart of the phase. Tests: `MaterializeCreatesExpectedSessions`, `MaterializeIsIdempotent`, `MaterializeSkipsWhenRoomOversubscribed`, `MaterializeAllowsParallelWhenRoomHasCapacity`, `MaterializeRespectsCapacityOverride`, `MaterializeRejectsUnknownSchedule`, `MaterializeRejectsDeactivatedSchedule`, `MaterializeStopsAtEffectiveTo`. The room-conflict check sums `event_sessions.capacity` of overlapping rows plus 1 per overlapping `bookable_service_sessions` row (those represent single-customer slots). Effective capacity = `class_schedules.capacity` override, else `classes.default_capacity`. `ROOM_OVERSUBSCRIBED` skip reason is returned via `MaterializeSkippedDate` rows. New helpers added to support the check: `EventSessions::GetOverlappingSessionsInRoom`, `BookableServiceSessions::GetOverlappingSessionsInRoom`, `Facilities`, `LocationRooms` (table helpers were missing). Implementation flow:
  - Load schedule; verify still active.
  - Compute the set of (date, start_us, end_us) tuples between `effective_from_us` and `min(throughDateUs, effective_to_us)` from `recurrence_pattern` + `days_of_week` + `start_time_minutes` + `duration_minutes`. Wrap existing `RecurringSessionHelper::GenerateSessionDates`.
  - For each (start_us, end_us): check `event_sessions` for an existing row with `class_schedule_id = scheduleId AND start_time_us = start_us AND status != 'cancelled'`. If present → `alreadyMaterializedCount++`, skip.
  - For each new candidate, run the **room concurrent-capacity check** (per P-4): query `event_sessions` + `bookable_service_sessions` overlapping that time window in the same room. If `Σ capacity` of overlapping sessions + this session's effective capacity > `location_rooms.concurrent_capacity`, append to `skippedDates` with reason `ROOM_OVERSUBSCRIBED` and continue.
  - Otherwise create the `event_sessions` row with `class_id`, `class_schedule_id`, `capacity = scheduleCapacityOverride.value_or(classDefault.capacity)`, `status = "scheduled"`, etc.
  - Returns `MaterializeResult` so admin UI can show created + skipped + already-materialized counts.
- [x] `DeactivateClassSchedule` flips `is_active=false` and, when `cancelFutureSessions=true`, walks future-dated `event_sessions` via `eventSessions_.GetEventSessionsByClassSchedule` and forwards each to `SessionCancellationHelper::CancelSession` with reason `"class schedule deactivated"`. Requires the SquareClient + MailHelper-flavored constructor for the cancellation handoff. Tests: `DeactivateFlipsIsActive`, `DeactivateUnknownScheduleErrorCode`.
- [x] `EditClassSchedule`: applies updates via `UpdateClassSchedule`, then if `regenerateFuture`, deletes uncancelled future un-booked `event_sessions` and re-runs `MaterializeFutureSessions` from now through the previous materialization horizon (max start_time across the surviving future rows). Bookings are NEVER blown away — sessions with `booked_count > 0` are kept as-is and surfaced in `result.sessionsKeptDueToBookings` per resolved OQ-P1-3. Tests: `EditWithoutRegenerateAppliesUpdate`, `EditRegenerateDeletesUnbookedAndKeepsBooked`, `EditUnknownScheduleErrorCode`.

### 4.2 New `ClassCatalogHelper`
- [x] Files: `class_catalog_helper.h/.cpp` + `class_catalog_helper_test.cpp`.
- [x] Methods:
  - `std::vector<ClassCatalogEntry> GetActiveClasses(Transaction&)` — all active classes; no per-user filter; powers the public `/api/classes` route for logged-out visitors.
  - `std::vector<ClassCatalogEntry> GetClassesVisibleToPerson(Transaction&, int64_t personId)` — active classes that have at least one `is_active = true` class_schedule attached. Phase 2 will layer real member-vs-non-member inclusion logic on top; for Phase 1 the personId parameter is reserved (the helper signature is stable so Phase 2 doesn't need a ServerAccess API break).
  - `std::optional<ClassDetail> GetClassDetail(Transaction&, int64_t classId, int64_t personId, int64_t upcomingSessionLimit = 16)` — class info + upcoming sessions (`EventSessions::GetEventSessionsByClass` from `now_us()` forward) + facility/room names + instructor names per session via `event_session_staffing → people`. Returns `nullopt` for missing or `is_active = false` classes. Price comes from the first active `class_schedule`'s product via `Payment::CatalogHelper::GetProduct(productId, personId)`.
- [x] `ClassCatalogEntry` struct fields: classId, name, description, photoUrl (empty in Phase 1; image wiring lands later), defaultCapacity, tags (empty until Phase 13), upcomingSessionCount (cancelled rows excluded from the count).
- [x] `ClassDetail` adds: upcomingSessions, requiredSkills (empty until Phase 3), priceInfo (currency, amountCents, isIncludedInMembership=false until Phase 2).
- [x] Tests: `GetActiveClassesExcludesInactive`, `GetActiveClassesCountsUpcomingSessions`, `GetActiveClassesExcludesCancelledFromCount`, `GetClassesVisibleToPersonExcludesClassesWithoutSchedule`, `GetClassesVisibleToPersonExcludesClassesWithOnlyInactiveSchedule`, `GetClassDetailReturnsNulloptForUnknown`, `GetClassDetailReturnsNulloptForInactive`, `GetClassDetailHasUpcomingSessionsWithInstructor`, `GetClassDetailUpcomingSessionsExcludeCancelled`, `GetClassDetailRespectsUpcomingLimit`. The "non-member sees no recurring classes" branch is deferred to Phase 2 where membership-gating actually exists.

### 4.3 Integration with existing `RecurringSessionHelper`
- [x] No changes to `RecurringSessionHelper`'s `GenerateSessionDates` core — `ClassScheduleHelper` wraps it.
- [x] `ClassScheduleHelper::MaterializeFutureSessions` builds an anchored `RecurringSessionRequest` (`anchorStartUs = effective_from_us + start_time_minutes * 60us/min`) and reuses `GenerateSessionDates` for the recurrence math, then writes each session itself so it can apply idempotency + room-conflict gating before insertion.
- [x] `RecurringSessionRequest` gained optional `classId` / `classScheduleId` fields with default `0`. `RecurringSessionHelper::CreateRecurringSessions` now propagates those onto every emitted `event_sessions` row when set. Two new tests in `recurring_session_helper_test.cpp` cover both the populated and unpopulated paths: `CreatePropagatesClassIdAndClassScheduleId`, `CreateLeavesClassFieldsNullWhenUnset`.

### 4.4 KeyValueTable conversions
- [x] Added to `business_logic/scheduling/scheduling_key_value_table.h/cpp`:
  - `ClassCatalogEntryToKeyValueTable` + `ClassCatalogEntriesToKeyValueTableArray`
  - `UpcomingSessionInfoToKeyValueTable` + `UpcomingSessionsToKeyValueTableArray`
  - `ClassDetailToKeyValueTable`
  - `MaterializeSkippedDateToKeyValueTable` + `MaterializeResultToKeyValueTable`
  - `CreateClassScheduleResultToKeyValueTable` / `EditClassScheduleResultToKeyValueTable` / `DeactivateClassScheduleResultToKeyValueTable`
  Note: there is no `ClassScheduleToKeyValueTable` because `class_schedules` rows are already `KeyValueTable` (they come out of `TableHelpers::ClassSchedules::GetClassSchedule` as such — endpoints flow them through `SqlUtil::KeyValueTableToJson` directly).
- [x] Unit tests in `scheduling_key_value_table_test.cpp` — 12 new cases covering each converter and both "ok" and "error" shapes of the result converters.

### 4.5 Tests for business logic
- [x] `class_schedule_helper_test.cpp` — 22 tests covering every validation error code, materialize 4 weeks Monday-only, materialize idempotency, both directions of the room concurrent-capacity check (skip + allow), capacity override propagation, schedule-not-found and schedule-inactive paths, `effective_to` window cap, edit-without-regenerate, edit-regenerate keeping booked sessions, and unknown-id error paths on Edit/Deactivate. The Deactivate-with-cancel-future test is deferred to integration / endpoint coverage because the path requires a fully-wired SquareClient / MailHelper; the unit-level test asserts the `is_active=false` flip and the unknown-schedule error code.
- [x] `class_catalog_helper_test.cpp` — 10 tests covering catalog filter on `is_active`, upcoming-session counting (including cancelled exclusion), the "no active schedule attached" filter for logged-in users, the inactive-schedule edge case, detail fetch missing/inactive paths, instructor surfacing via `event_session_staffing → people`, cancelled-session exclusion from detail, and `upcomingSessionLimit` honouring.
- [x] No path in Phase 1 invokes `ThreadPool::Queue` — confirmed by inspection of `ClassScheduleHelper` and `ClassCatalogHelper`; the only async-write entry points in the codebase (`/api/login`, `/api/verify`, `/api/remember`, `PersonHelper::SessionUsed`) are untouched here.

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

## 12. Resolved Questions

All Phase 1 open questions are resolved. Mason accepted the recommendation on each; decisions are folded into the relevant sections above and recorded here for the project history.

- [x] **OQ-P1-1 — `days_of_week` storage format.** *Question:* comma-separated TEXT (`"1,3,5"`) or Postgres `INTEGER[]`? **Decision:** comma-separated TEXT, for portability and to match the existing convention for small enum-set columns in this codebase. *Applied in:* §2.2 (DDL) and §3.1 (`GetSchedulesPotentiallyConflictingInRoom` regex uses comma-wrapped tokens to avoid substring traps like "1" matching "10").
- [x] **OQ-P1-2 — `predecessor_class_schedule_id` chain length cap.** *Question:* reject same-day chains longer than 2 to prevent accidental "hour 1 → 2 → 3 → 4" stacks? **Decision:** no cap for now; if it becomes a hygiene issue, revisit. *Applied in:* §2.2 (DDL note) and the Phase 3 validator (it will only check that the referenced row exists, not how long the chain is).
- [x] **OQ-P1-3 — `EditClassSchedule(regenerateFuture=true)` with booked future sessions (Phase 7+).** *Question:* (a) reject the edit, (b) accept but leave bookable sessions untouched, or (c) cascade-cancel-and-refund? **Decision:** (b) — leave touched sessions as-is and surface them in the response so admin can manually reconcile. *Applied in:* §4.1 `EditClassSchedule` description.

## 13. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 1.
- Predecessor work: [[Scheduling thin slice]], [[Provider Portal]].
- Will feed into: [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 5 - Attendance Templates]], [[Classes Phase 7 - Class Series and Workshops]].
