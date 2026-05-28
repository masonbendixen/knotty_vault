---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/28/2026
Version: 0.2
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

> ## ⚠️ REDESIGNED 2026-05-28 — this supersedes the original flat-model plan
> This Phase 1 plan was rewritten to the **three-level versioned-implementation model**. The design rationale and locked decisions (L-1..L-9, OQ-CSI-1..21) live in [[Class Schedule Implementations Redesign]]. The parent roadmap is [[Classes, schedules, and attendance]] (§6 Phase 1, P-7).
>
> **The original Phase 1 was implemented against a flat `class_schedules` table with a "materialize through date" button.** That implementation is being replaced wholesale (rewrite-in-place per OQ-CSI-10 — pre-deploy, no production data to migrate). Where the old code exists, this plan replaces it; checkboxes below are reset to reflect the new, not-yet-built model. A "Superseded — original flat-model plan" appendix at the bottom preserves the prior decisions for history.

## Implementation Status (updated 2026-05-28)

**Server side: built and all unit tests passing.** Frontend deferred to a separate step (per decision in [[Class schedule build reconciliation]]).

- ✅ **§2 Database schema** — `classes.kind`, new `class_instances` / `class_schedules` (impl) / `class_schedule_slots`, `event_sessions` re-keyed; wired into init pipeline. (Build + tests green.)
- ✅ **§3 Table helpers** — `ClassInstances`, `ClassSchedules`, `ClassScheduleSlots`, extended `EventSessions` (+ tests). Note: §3.4's lazy-orphan query shipped as `GetFutureClassSessionsForInstance` (the helper returns the instance's future class sessions; the business-logic sweep decides orphan-ness).
- ✅ **§4.1 / §4.4 / §4.7** — `ClassInstanceHelper`, impl-save sweep, business-logic tests.
- 🟡 **§4.2 `ClassScheduleHelper`** — impl + slot CRUD, `EnsureSessionExists`, `GetDerivedSessionsForRange`, sweep: **done**. `GetActiveScheduleView` (preview backing) **deferred** with the preview endpoint (§5.4).
- 🟡 **§4.5 `ClassCatalogHelper`** — recurring catalog + detail (derived sessions, instructor, price-via-instance): **done**. Workshop/series "upcoming runs" branch **deferred**.
- 🟡 **§4.6 KVT** — catalog/instance/impl/slot/derived/result converters **done**; `ActiveScheduleView` + `SweepResult` converters **deferred** (their features aren't built yet).
- ❌ **§4.3 Room concurrent-capacity check** — not started.
- ✅ **§5.1 catalog endpoints**, ✅ **§5.5 routing** (materialize endpoint removed).
- 🟡 **§5.2 / §5.3 admin endpoints** — implementation create/update/deactivate/list **migrated & tested** (create lives at `POST /api/admin/class_schedule` taking `class_instance_id`, not the `/class_instance/<id>/schedule` shape in the plan). Instance endpoints, migrate-product endpoint, **slot endpoints**, and **§5.4 preview** **not built** — no end-to-end authoring over HTTP yet.
- ✅ **§8 permission** (`manage_class_schedule`) / ✅ **§9 seed** (`PopulateClassSchedules` on the new model).
- ❌ **§6 Frontend**, **§7 admin metadata** (not verified this session), §10 frontend tests, §11 acceptance — pending.

Checkboxes below reflect this. Anything still `[ ]` under §4.3, §5.2 (instance/slot/preview), §6, §7, §10 (frontend) is genuine remaining work.

## Phase Summary

**Must-have core.** Admin can define a class (name, description, photo, kind, defaults), create one or more **instances** (runs) under it, attach versioned **implementations** (impls) with priority + validity window to each instance, and fill each impl with **slots** (day-of-week + start time + duration + facility + room + instructor tuples). The public catalog browses classes. Class sessions are **derived on the fly** from the active instance + active impl + slots — there is no materialization step. `event_sessions` rows persist only when something is recorded against a specific occurrence.

No booking flow yet (Phase 2), no skill / series / check-in gating yet (Phases 3 / 7 / 8). Workshops + series are *enabled* by this infrastructure (their schedule lives in the same tables) but their purchase machinery is Phase 7 (and Phase 2 for the M-12 intro workshop).

**The three-level hierarchy:**

```
classes  (marketing identity — name, description, photo, kind enum)
   └── class_instances  (a run — own product, validity window; perpetual + 1:1 for recurring)
          └── class_schedules  (versioned impls under one instance — priority + window)
                 └── class_schedule_slots  (recurring day/time/facility/room/instructor tuples)
```

**Prerequisites:**
- Existing `classes` table (photo-supported).
- Existing `events`, `event_sessions`, `event_session_staffing`, `facilities`, `location_rooms` infrastructure from [[Scheduling thin slice]].
- Existing product / pricing / permission infrastructure ([[Payment Design Document]]).
- `RecurringSessionHelper::GenerateSessionDates` is reused only as a date-walking utility inside the derived-session computation — it is NOT used to pre-create rows.

**Outcome:**
- Admin can create a class, instance(s), impl(s), and slots through a three-level admin UI with friendly-name autocompletes.
- The public `/classes` catalog shows classes with photos; recurring class detail shows derived upcoming sessions; workshop/series detail shows the list of upcoming runs.
- Calendar / catalog query a single `GetDerivedSessionsForRange` helper — correct out to any future date with no materialization.
- Admin can preview "what's the active schedule on date X" for a class.
- Editing an impl applies immediately and sweeps stale future-date admin-only rows (refusing if a paid booking would be orphaned).

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
- All tables pre-created at startup by `GlobalDatabaseTestSupport::SetupAllTables()` — do NOT call `MakePaymentTables`, `DbOps::CreateTable`, etc. in tests.
- Use `crow::query_string` for query-param handling in endpoint tests.
- `ThreadPool::Shutdown()` BEFORE the next DB read in any test that hits an endpoint queueing async work. (No Phase 1 path queues async work, but the rule stands if any is added.)

## 1. Pre-Coding Design Decisions (all RESOLVED — see [[Class Schedule Implementations Redesign]] §2.0)

### 1.1 Locked decisions carried in from the redesign
- [x] **L-1 / OQ-CSI-1** Three-level hierarchy `classes` → `class_instances` → `class_schedules` → `class_schedule_slots`. Impl + slot keep the `class_schedules` / `class_schedule_slots` names (price-schedule precedent); the new middle layer is `class_instances`.
- [x] **L-3 / OQ-CSI-2** `product_id` lives on `class_instances` (not on `classes`, not on impls). Pure price changes flow through `product_prices` × `price_schedules`; product migrations close one instance + open another.
- [x] **OQ-CSI-3** `facility_id` + `location_room_id` live on the slot (a class can run at different facilities/rooms within one impl).
- [x] **OQ-CSI-6** No `recurrence_pattern` column. Weekly is implicit in the slot rows; no biweekly/custom.
- [x] **OQ-CSI-7** Slot start times use a full HH:MM time picker (carve-out from the hour-only convention — class times aren't hour-aligned).
- [x] **OQ-CSI-8** Reject identical (`class_schedule_id`, `day_of_week`, `start_time_minutes`, `location_room_id`) slot tuples as data-entry errors.
- [x] **OQ-CSI-10** Rewrite Phase 1 in place (pre-deploy, no migration path).
- [x] **OQ-CSI-11** Skill-level requirements stay per-class (Phase 3 owns them).
- [x] **OQ-CSI-12** Lazy session derivation — no materialization, no horizon job.
- [x] **OQ-CSI-13** Keep `event_sessions` for everything; class occurrences persist there only on a recording trigger.
- [x] **OQ-CSI-14** Slot carries the default `instructor_person_id`; per-session subs ride on `event_session_staffing`.
- [x] **L-7 / OQ-CSI-15 / OQ-CSI-18** One `classes` row per offering identity; `classes.kind` enum (`recurring`|`workshop`|`series`) discriminates rendering. Workshops + series share their `classes` row across runs.
- [x] **L-5 / OQ-CSI-16** Series-bundle fields live in a Phase 7 `class_series_instances` augmentation table — NOT in Phase 1.
- [x] **L-9 / OQ-CSI-17** No global studio-closure lever. Closures are per-class empty high-priority impls. The batch UI is deferred to Phase 10 (OQ-CSI-4).
- [x] **OQ-CSI-19** Impl-save sweep: auto-delete future-date orphaned admin-only rows scoped to the impl's instance; refuse if any orphan carries a `purchase_id`. No standalone orphan-recovery view.
- [x] **OQ-CSI-20** "Migrate to new product effective DATE" action closes the perpetual instance + opens a new one, copying slots forward.
- [x] **OQ-CSI-21** "Copy slots from impl" picker on impl create.

### 1.2 Carried-over decisions still in force
- [x] Attendance templates (Phase 5) are aspirational only — they create no bookings, consume no capacity. Phase 1 creates no implicit `bookings` writes.
- [x] Room conflict policy (P-4): the room-conflict check sums derived + persisted overlapping `event_sessions` + `bookable_service_sessions` against `location_rooms.concurrent_capacity`. Hard block only on overflow.

## 2. Database Schema

### 2.1 Extend `classes` table
- [x] In `db_schema/classes.h`, keep the existing column-name constants (`kClassesDefaultCapacity`, `kClassesIsActive`, `kClassesCreatedUs`, `kClassesUpdatedUs`) and add `kClassesKind`.
- [x] In `db_schema/classes.cpp`, add `kind TEXT NOT NULL DEFAULT 'recurring'` (enum `recurring` | `workshop` | `series`, CHECK enforced at the application layer). Keep `default_capacity`, `is_active`, `created_us`, `updated_us`.
- [x] **Remove** `default_cancellation_policy_id` / `default_room_type_id` if they were added by the original flat plan — cancellation policy now comes from the instance's product, and room is on the slot. (If they were never wired anywhere, just drop them.)
- [x] No `product_id` on `classes`.
- [x] Keep the `is_active` index via `DbSchema::CreateClassesIndexes`.

### 2.2 New `class_instances` table
- [x] New files `db_schema/class_instances.h/.cpp` with columns:
  - `id BIGSERIAL PRIMARY KEY`
  - `class_id BIGINT NOT NULL` (FK → `classes(id)` via `AddColumnForeignKeyRef`)
  - `name TEXT NOT NULL`
  - `valid_from_us BIGINT NOT NULL`
  - `valid_to_us BIGINT` NULL (NULL = open-ended / perpetual)
  - `product_id BIGINT NOT NULL` (FK → `products(id)`) — pricing / visibility / booking permission / cancellation policy / advance windows
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us BIGINT NOT NULL DEFAULT now_us()`
  - `updated_us BIGINT NOT NULL DEFAULT now_us()`
- [x] Indexes on (`class_id`, `is_active`) and (`product_id`) via `DbSchema::CreateClassInstancesIndexes(transaction)`.

### 2.3 New `class_schedules` table (the implementation)
- [x] New files `db_schema/class_schedules.h/.cpp` with columns:
  - `id BIGSERIAL PRIMARY KEY`
  - `class_instance_id BIGINT NOT NULL` (FK → `class_instances(id)`)
  - `name TEXT NOT NULL` ("Default schedule", "Memorial Day", "Holiday Week")
  - `priority INTEGER NOT NULL DEFAULT 3`
  - `valid_from_us BIGINT NOT NULL`
  - `valid_to_us BIGINT` NULL
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us BIGINT NOT NULL DEFAULT now_us()`
  - `updated_us BIGINT NOT NULL DEFAULT now_us()`
- [x] No `class_id`, `product_id`, `facility_id`, `location_room_id`, `recurrence_pattern`, `days_of_week`, `start_time_minutes`, `duration_minutes`, `effective_*`, `capacity`, `is_series`, `series_*`, or `predecessor_*` columns — all dropped per the redesign.
- [x] Indexes on (`class_instance_id`, `is_active`, `priority`) via `DbSchema::CreateClassSchedulesIndexes(transaction)`.

### 2.4 New `class_schedule_slots` table
- [x] New files `db_schema/class_schedule_slots.h/.cpp` with columns:
  - `id BIGSERIAL PRIMARY KEY`
  - `class_schedule_id BIGINT NOT NULL` (FK → `class_schedules(id)`)
  - `day_of_week SMALLINT NOT NULL` (0=Sun..6=Sat; CHECK 0..6 at app layer)
  - `start_time_minutes INTEGER NOT NULL` (0..1439 minutes-after-local-midnight at the slot's facility)
  - `duration_minutes INTEGER NOT NULL DEFAULT 60` (> 0)
  - `facility_id BIGINT NOT NULL` (FK → `facilities(id)`)
  - `location_room_id BIGINT NOT NULL` (FK → `location_rooms(id)`)
  - `instructor_person_id BIGINT` NULL (FK → `people(id)`; NULL = TBD)
  - `predecessor_class_schedule_slot_id BIGINT` NULL — plain nullable BIGINT, no self-FK (avoid cascade surprises); Phase 3 (SL-11) validates the reference
  - `capacity_override INTEGER` NULL (NULL = use `classes.default_capacity`)
  - `created_us BIGINT NOT NULL DEFAULT now_us()`
  - `updated_us BIGINT NOT NULL DEFAULT now_us()`
- [x] Index on (`class_schedule_id`) and on (`location_room_id`, `day_of_week`) for the room-conflict query, via `DbSchema::CreateClassScheduleSlotsIndexes`.
- [x] Application-layer uniqueness check on (`class_schedule_id`, `day_of_week`, `start_time_minutes`, `location_room_id`) per OQ-CSI-8 (metadata builder doesn't model multi-col unique constraints; enforce in the helper / business logic).

### 2.5 Extend `event_sessions` table
- [x] Add `class_schedule_slot_id BIGINT` NULL (FK → `class_schedule_slots(id)` via `AddColumnForeignKeyRefNullable`).
- [x] Add `occurrence_date_us BIGINT` NULL (date-truncated local-midnight for the day this row pins).
- [x] Add `class_id BIGINT` NULL (FK → `classes(id)`, denormalized convenience for the 4-week-attendance and history joins).
- [x] The composite (`class_schedule_slot_id`, `occurrence_date_us`) is the natural key for the lazy derived-vs-persisted lookup. Add a (partial, where `class_schedule_slot_id IS NOT NULL`) index on it via `DbSchema::CreateEventSessionsIndexes`.
- [x] Existing event / service session rows keep all three NULL — backwards-compatible.

### 2.6 Wire schema into the database init pipeline
- [x] Update `make_database_info.cpp`: add `MakeClassInstancesTable`, `MakeClassSchedulesTable`, `MakeClassScheduleSlotsTable` in FK order — after `classes`, `facilities`, `location_rooms`, `products` are in the builder, and before `MakeEventSessionsTable`.
- [x] Update `database_helper/create_database.cpp` `CreateTables()`: `CreateTable` calls in order `class_instances` → `class_schedules` → `class_schedule_slots`, each before `kEventSessionsTable`; plus the index-creation calls.
- [x] Update `db_schema/CMakeLists.txt` with all three new header/cpp pairs.

## 3. Table Helpers (`sql_util/table_helpers/`)

### 3.1 New `TableHelpers::ClassInstances`
- [x] `class_instances.h/.cpp` + `class_instances_test.cpp`. Methods on `DbCrud`:
  - `int64_t AddClassInstance(Transaction&, const KeyValueTable&)`
  - `KeyValueTable GetClassInstance(Transaction&, int64_t id)`
  - `void UpdateClassInstance(Transaction&, int64_t id, const KeyValueTable& updates)` (bumps `updated_us`)
  - `void SetIsActive(Transaction&, int64_t id, bool)`
  - `std::optional<KeyValueTable> GetActiveInstance(Transaction&, int64_t classId, int64_t atUs)` — `WHERE class_id=$1 AND is_active AND valid_from_us<=$2 AND (valid_to_us IS NULL OR $2<valid_to_us)` (instances don't overlap → at most one)
  - `KeyValueTableArray GetInstancesByClass(Transaction&, int64_t classId)` — active + closed, sorted `valid_from_us DESC, id DESC`
  - `KeyValueTableArray GetUpcomingInstances(Transaction&, int64_t classId, int64_t asOfUs)` — for the workshop/series catalog list
- [x] Tests: add+get round-trip, perpetual (`valid_to_us=NULL`) active resolution, closed instance excluded from active, upcoming filter, update bumps `updated_us`.

### 3.2 New `TableHelpers::ClassSchedules` (instance-scoped impls)
- [x] `class_schedules.h/.cpp` + `class_schedules_test.cpp`. Methods:
  - `int64_t AddClassSchedule(Transaction&, const KeyValueTable&)`
  - `KeyValueTable GetClassSchedule(Transaction&, int64_t id)`
  - `void UpdateClassSchedule(...)` (bumps `updated_us`)
  - `void SetIsActive(...)`
  - `std::optional<KeyValueTable> GetActiveImplementation(Transaction&, int64_t classInstanceId, int64_t atUs)` — `ORDER BY priority DESC, valid_from_us DESC LIMIT 1`
  - `KeyValueTableArray GetImplementationsByInstance(Transaction&, int64_t classInstanceId)` — sorted `priority DESC, valid_from_us ASC, id ASC`
  - `KeyValueTableArray GetImplementationsOverlapping(Transaction&, int64_t classInstanceId, int64_t fromUs, int64_t toUs)` — for the same-priority-overlap validation
- [x] Tests: add+get, active-impl priority resolution (higher priority wins), tie-break by `valid_from_us`, overlap query, update/soft-delete.

### 3.3 New `TableHelpers::ClassScheduleSlots`
- [x] `class_schedule_slots.h/.cpp` + `class_schedule_slots_test.cpp`. Methods:
  - `int64_t AddSlot(Transaction&, const KeyValueTable&)`
  - `KeyValueTable GetSlot(Transaction&, int64_t id)`
  - `void UpdateSlot(...)`, `void DeleteSlot(...)`
  - `KeyValueTableArray GetSlotsByImplementation(Transaction&, int64_t classScheduleId)` — sorted `day_of_week ASC, start_time_minutes ASC, id ASC`
  - `KeyValueTableArray GetSlotsByImplementationAndDay(Transaction&, int64_t classScheduleId, int dayOfWeek)`
  - `KeyValueTableArray GetSlotsPotentiallyConflictingInRoom(Transaction&, int64_t roomId, int dayOfWeek, int64_t startTimeMinutes, int64_t durationMinutes)` — narrows by room + day; time-window overlap math done by the caller
  - `bool SlotTupleExists(Transaction&, int64_t classScheduleId, int dayOfWeek, int64_t startTimeMinutes, int64_t roomId)` — OQ-CSI-8 duplicate guard
- [x] Tests: add+get, multi-slot-per-day (morning + evening), per-day-different-times, sort order, conflict query, duplicate-tuple detection, delete.

### 3.4 Extend `TableHelpers::EventSessions`
- [x] `class_schedule_slot_id`, `occurrence_date_us`, `class_id` come back from `GetEventSession`/`GetEventSessions` automatically (`SELECT *`). Tests verify round-trip + NULL for classic event/service rows.
- [x] Add `std::optional<KeyValueTable> LookupBySlotAndDate(Transaction&, int64_t slotId, int64_t occurrenceDateUs)` — the lazy lookup. Test: returns the row when present, nullopt when not.
- [x] Add `KeyValueTableArray GetOrphanedFutureSessionsForInstance(Transaction&, int64_t classInstanceId, int64_t asOfUs)` — future-dated persisted rows whose slot is no longer in the instance's active impl; powers the impl-save sweep. Test: returns the orphan; excludes booked rows from the deletable set (caller distinguishes via `purchase_id`). *(Shipped as `GetFutureClassSessionsForInstance`; the sweep in `ClassScheduleHelper` decides orphan-ness and uses `booked_count` rather than `purchase_id`.)*

## 4. Business Logic (`business_logic/scheduling/`)

### 4.1 New `ClassInstanceHelper`
- [x] `class_instance_helper.h/.cpp` + `class_instance_helper_test.cpp`. `kClassInstanceError*` `string_view` constants exported for shared error vocabulary.
- [x] Methods:
  - `CreateInstanceResult CreateInstance(Transaction&, const CreateInstanceRequest&)` — validates class exists + active, product exists, `valid_to > valid_from`, no overlapping active instance for the class, and (for `kind='recurring'`) at most one perpetual `valid_to_us=NULL` instance. Error codes: `INVALID_CLASS`, `INVALID_PRODUCT`, `INVALID_TIME_BOUNDS`, `OVERLAPPING_INSTANCE`, `DUPLICATE_PERPETUAL`.
  - `bool UpdateInstance(Transaction&, int64_t id, const KeyValueTable& updates)`
  - `bool CloseInstance(Transaction&, int64_t id, int64_t validToUs)` — sets `valid_to_us`.
  - `MigrateResult MigrateRecurringClassToNewProduct(Transaction&, int64_t classId, int64_t newProductId, int64_t effectiveAtUs, bool copySlotsForward)` — closes the perpetual instance at `effectiveAtUs`, opens a new perpetual instance with `newProductId`, and (if `copySlotsForward`) copies the closing instance's latest active impl + slots into a new default impl on the new instance (OQ-CSI-20).
- [x] Tests: create happy path, each validation error, perpetual-dup rejection, overlap rejection, close, migrate (verify old closed + new open + slots copied).

### 4.2 New `ClassScheduleHelper`
- [x] `class_schedule_helper.h/.cpp` + `class_schedule_helper_test.cpp`. Public surface (all implemented **except** `GetActiveScheduleView`, deferred with the §5.4 preview endpoint):
  - `CreateImplementationResult CreateImplementation(Transaction&, const CreateImplementationRequest&)` — validates parent instance exists, impl window lies within the instance window, and no same-priority overlapping impl under the instance (`OVERLAPPING_SAME_PRIORITY`). Optional `copyFromImplementationId` to seed slots (OQ-CSI-21).
  - `EditResult UpdateImplementation(Transaction&, int64_t id, const KeyValueTable& updates)` — applies the update, then runs the impl-save sweep (§4.4).
  - Slot CRUD: `AddSlot` (enforces OQ-CSI-8 duplicate guard + facility/room validity), `UpdateSlot`, `DeleteSlot` (each slot mutation also runs the sweep).
  - `int64_t EnsureSessionExists(Transaction&, int64_t slotId, int64_t occurrenceDateUs)` — idempotent: looks up `(slotId, occurrenceDateUs)` via `EventSessions::LookupBySlotAndDate`; if absent, creates the `event_sessions` row with `class_id`, `class_schedule_slot_id`, `occurrence_date_us`, computed `start_time_us` / `end_time_us` from the slot, `capacity = slot.capacity_override ?? class.default_capacity`, `status='scheduled'`. Returns the row id. This is the single entry point for the seven recording triggers.
  - `std::vector<DerivedSession> GetDerivedSessionsForRange(Transaction&, int64_t classId, int64_t fromUs, int64_t toUs)` — walks each date in range; resolves active instance (`ClassInstances::GetActiveInstance`), then active impl (`ClassSchedules::GetActiveImplementation`), expands slots matching `EXTRACT(DOW)`, and left-joins persisted `event_sessions` rows (so cancellations / subs / notes override the derived defaults). Reuses `RecurringSessionHelper::GenerateSessionDates` only as a date-walking utility.
  - `ActiveScheduleView GetActiveScheduleView(Transaction&, int64_t classId, int64_t dateUs)` — backs the admin "schedule on date X" preview (active instance + active impl + resolved slot list). **⏳ DEFERRED — not built (ships with the §5.4 preview endpoint).**
- [x] Validation error codes mirror the §4.1 pattern; dedicated tests per code.
- [x] Tests: create-impl window-within-instance, same-priority-overlap rejection, copy-from seeding, multi-slot/day derivation, per-day-different-times derivation, priority-resolution derivation (override impl wins on its dates), closure (empty impl → no derived sessions), `EnsureSessionExists` idempotency, derived-vs-persisted left-join (a cancelled persisted row suppresses/overrides the derived default).

### 4.3 Room concurrent-capacity check
- [ ] `RoomOccupancyHelper` (or a method on `ClassScheduleHelper`) returns all derived + persisted sessions overlapping a given (room, time window) and sums `capacity` against `location_rooms.concurrent_capacity` (P-4). Used at slot-add validation and (Phase 2) at booking time. Reuses `EventSessions::GetOverlappingSessionsInRoom` + `BookableServiceSessions::GetOverlappingSessionsInRoom` if those exist from the prior implementation; otherwise add them.
- [ ] Tests: parallel sessions allowed under capacity; blocked on overflow; derived + persisted both counted.

### 4.4 Impl-save sweep (OQ-CSI-19)
- [x] `SweepOrphanedFutureSessions(Transaction&, int64_t classInstanceId, int64_t asOfUs) -> SweepResult { int64_t deletedCount; std::vector<int64_t> blockedRowIds; }` *(shipped as a private method on `ClassScheduleHelper` returning the count + populating a `blockedRowIds` out-param; blocks on `booked_count > 0` rather than `purchase_id`)* — finds future-dated persisted `event_sessions` under the instance whose `class_schedule_slot_id` is no longer present in the instance's active impl for that date; deletes those holding only admin actions (no `purchase_id`); collects any with a `purchase_id` into `blockedRowIds`. If `blockedRowIds` is non-empty, the caller (`UpdateImplementation` / slot mutation) aborts the save and surfaces them.
- [x] Tests: sweep deletes an orphaned note row; sweep refuses (returns blocked) when a `purchase_id` row would be orphaned; sweep no-ops when nothing orphaned.

### 4.5 New `ClassCatalogHelper`
- [x] `class_catalog_helper.h/.cpp` + `class_catalog_helper_test.cpp`. Methods (recurring catalog/detail done; workshop/series "upcoming runs" branch deferred):
  - `std::vector<ClassCatalogEntry> GetActiveClasses(Transaction&)` — active classes with at least one active instance; powers the public `/api/classes` for anonymous visitors.
  - `std::vector<ClassCatalogEntry> GetClassesVisibleToPerson(Transaction&, int64_t personId)` — Phase 2 layers membership filtering; Phase 1 reserves the param.
  - `std::optional<ClassDetail> GetClassDetail(Transaction&, int64_t classId, int64_t personId, int64_t upcomingLimit = 16)` — for `kind='recurring'`, returns derived upcoming sessions (via `GetDerivedSessionsForRange` from now forward) + facility/room/instructor names; for `kind='workshop'|'series'`, returns the list of upcoming instances (runs) with each run's window + per-tier price from its product. Price via `Payment::CatalogHelper::GetProduct(instance.product_id, personId)`.
- [x] `ClassCatalogEntry`: classId, name, description, photoUrl, kind, defaultCapacity, and (recurring) upcomingSessionCount or (workshop/series) upcomingRunCount.
- [x] `ClassDetail`: kind, upcomingSessions (recurring) or upcomingRuns (workshop/series), requiredSkills (empty until Phase 3), priceInfo.
- [x] Tests: catalog excludes classes with no active instance; recurring detail derives upcoming sessions with instructor; cancelled persisted occurrences excluded from recurring upcoming; unknown/inactive → nullopt; upcomingLimit honored. *(workshop/series "detail lists runs" not yet implemented/tested — deferred.)*

### 4.6 KeyValueTable conversions
- [x] Add to `business_logic/scheduling/scheduling_key_value_table.h/cpp`: converters for `ClassCatalogEntry`, `ClassDetail`, `DerivedSession`/`UpcomingSessionInfo`, `ActiveScheduleView`, `CreateInstanceResult`, `CreateImplementationResult`, `EditResult`, `MigrateResult`, `SweepResult`. *(`InstanceEditResult` + `SlotResult` also added; `ActiveScheduleView` and a standalone `SweepResult` converter deferred — those features aren't built. `EditResult` carries the sweep result.)* (No `ClassScheduleToKeyValueTable` / `ClassInstanceToKeyValueTable` — those rows already come out of the table helpers as `KeyValueTable` and flow through `SqlUtil::KeyValueTableToJson` directly.)
- [x] Unit tests in `scheduling_key_value_table_test.cpp` covering each converter (ok + error shapes for the result types).

### 4.7 Tests for business logic
- [x] `class_instance_helper_test.cpp`, `class_schedule_helper_test.cpp`, `class_catalog_helper_test.cpp`, extended `scheduling_key_value_table_test.cpp` — per the per-section lists above.
- [x] No Phase 1 path invokes `ThreadPool::Queue` — confirm by inspection.

## 5. Endpoints (`endpoints/`)

### 5.1 Public catalog endpoints
- [x] `endpoints/get_classes.h/cpp` + test: `GET /api/classes`. Anonymous → `GetActiveClasses`; logged-in → `GetClassesVisibleToPerson`. Returns `{ "items": [ ClassCatalogEntry, ... ] }`. Tests: empty, active-only, anonymous sees active.
- [x] `endpoints/get_class_detail.h/cpp` + test: `GET /api/classes/<int>`. Returns `ClassDetail` (recurring → upcoming_sessions; workshop/series → upcoming_runs). 404 for unknown/inactive. Tests: recurring detail with instructor + facility/room names; workshop/series detail with runs; 404 paths.

### 5.2 Admin instance endpoints
- [x] Permission: keep `DbSchema::kPermissionManageClassSchedule = "manage_class_schedule"`; admin + Studio Manager roles get it.
- [ ] `endpoints/admin_class_instance_create.h/cpp` + test: `POST /api/admin/class_instance`. Body → `CreateInstanceRequest`. 200 + `instance_id`; 400 + `error_code`. Tests: 401 anon, 403 missing perm, 400 validation, 200 persists.
- [ ] `endpoints/admin_class_instance_update.h/cpp` + test: `PUT /api/admin/class_instance/<int>`. Tests: 403, 404, 200 applies.
- [ ] `endpoints/admin_class_instance_deactivate.h/cpp` + test: `DELETE /api/admin/class_instance/<int>`. Soft-delete. Tests: 403, 404, 200 flips `is_active`.
- [ ] `endpoints/admin_class_migrate_product.h/cpp` + test: `POST /api/admin/class/<classId>/migrate_product`. Body `{ new_product_id, effective_at_us, copy_slots_forward }`. Delegates to `MigrateRecurringClassToNewProduct`. Tests: 403, 404, 200 closes old + opens new.
- [ ] `endpoints/admin_class_instances_list.h/cpp` + test: `GET /api/admin/class_instances?class_id=<id>`. Tests: 401, 403, 400 missing class_id, 200 lists.

### 5.3 Admin implementation + slot endpoints
- [x] `endpoints/admin_class_schedule_create.h/cpp` + test: creates an implementation. Body → `CreateImplementationRequest` (incl. optional `copy_from_implementation_id`). Tests: 401, 403, 400 (validation `error_code`), 200 persists. *(Route shipped as `POST /api/admin/class_schedule` with `class_instance_id` in the body, not `/api/admin/class_instance/<instanceId>/schedule`.)*
- [x] `endpoints/admin_class_schedule_update.h/cpp` + test: `PUT /api/admin/class_schedule/<int>`. Body `{ updates, }`. Runs the sweep; response includes `{ deleted_orphan_count, blocked_rows }`. Tests: 403, 404, 200 applies + reports sweep, 409/400 when blocked by a paid booking.
- [x] `endpoints/admin_class_schedule_deactivate.h/cpp` + test: `DELETE /api/admin/class_schedule/<int>`. Soft-delete. Tests: 403, 404, 200.
- [x] `endpoints/admin_class_schedules_list.h/cpp` + test: `GET /api/admin/class_schedules?class_instance_id=<id>`. Tests: 401, 403, 400 missing param, 200 lists impls for the instance.
- [ ] Slot endpoints: `POST /api/admin/class_schedule/<id>/slot`, `PUT /api/admin/class_schedule_slot/<slotId>`, `DELETE /api/admin/class_schedule_slot/<slotId>` (+ tests). Slot mutations run the sweep + return its result. Add returns 400 with `DUPLICATE_SLOT` on the OQ-CSI-8 guard.

### 5.4 Admin preview endpoint
- [ ] `endpoints/admin_class_schedule_preview.h/cpp` + test: `GET /api/admin/class_schedule_preview?class_id=<id>&date_us=<t>` → `ActiveScheduleView` (active instance + active impl + resolved slots). Tests: 403, 200 resolves the correct impl on a date covered by a high-priority override, 200 empty on a closed/dark date.

### 5.5 Routing + testing patterns
- [x] Register all endpoints in `endpoints/web_app.cpp` (include + anonymous-namespace pointer holder) and `endpoints/CMakeLists.txt`.
- [x] **No** `materialize` endpoint exists.
- [x] Query-param endpoints use `crow::query_string` in tests.
- [x] `ValidationError → 400` confirmed via the create-endpoint tests.

## 6. Frontend (Angular)

### 6.1 Public class catalog page
- [ ] `ClassInfoComponent` at `ui/src/app/pages/public/class-info/` (route `/classes`). Loads `ServerAccess.getClasses()`. Grid: photo, name, description, kind badge, and (recurring) upcoming-session-count or (workshop/series) upcoming-run-count. Empty + error states. Component spec.

### 6.2 Public class detail page
- [ ] `ClassDetailComponent` at `ui/src/app/pages/public/class-detail/` (route `/classes/:id`). Loads `getClassDetail(id)`. For recurring: hero + description + upcoming derived sessions (facility / room / instructor). For workshop/series: marketing copy + list of upcoming runs (each with window + per-tier price). 404 panel. Component spec.

### 6.3 Admin three-level schedule UI
- [ ] `ui/src/app/pages/manage/class-schedules/` area, gated by `ManageProductsGuard` (backend independently enforces `manage_class_schedule`). Discoverable from the manage dashboard tile.
- [ ] **Class list / detail** component: lists classes with kind; class detail shows the instance list + "Add instance" + (for recurring) "Migrate to new product" action.
- [ ] **Instance detail** component: name + window + product (autocomplete) + impl list + "Add implementation" (with "copy slots from impl" picker, OQ-CSI-21).
- [ ] **Implementation detail** component: priority + window + slot editor.
- [ ] **Slot editor** component: sorted list; per-row day-of-week dropdown, HH:MM time picker (OQ-CSI-7), duration input (default 60), facility / room / instructor autocompletes, optional predecessor-slot picker (scoped to same-day sibling slots). Add / remove. Rejects duplicate tuples (surfaces `DUPLICATE_SLOT`).
- [ ] **Schedule-on-date preview** component: date picker → calls preview endpoint → shows active instance + impl + slot list.
- [ ] **Impl-save sweep confirmation** modal: when a save will sweep N future admin-only rows, confirm "N future notes/subs will be removed". When blocked by paid bookings, list them with "cancel & refund first" (the cancel action is Phase 2/10; Phase 1 just surfaces + links).
- [ ] No materialize dialog. Component specs for every component.

### 6.4 `ServerAccess` extensions
- [ ] Add to `ServerAccess` interface + proxy + `ServerAccessNetwork` + `ServerAccessMock`:
  - `getClasses(): Observable<ClassCatalogEntry[]>`
  - `getClassDetail(id): Observable<ClassDetail>`
  - `listClassInstances(classId)`, `createClassInstance(req)`, `updateClassInstance(id, body)`, `deactivateClassInstance(id)`, `migrateClassProduct(classId, body)`
  - `listClassSchedules(classInstanceId)`, `createClassSchedule(instanceId, req)`, `updateClassSchedule(id, body)`, `deactivateClassSchedule(id)`
  - slot CRUD: `addClassScheduleSlot(scheduleId, slot)`, `updateClassScheduleSlot(slotId, body)`, `deleteClassScheduleSlot(slotId)`
  - `getClassSchedulePreview(classId, dateUs)`
- [ ] `ServerAccess.mock.spec.ts` updated with a block covering all new mock methods (catalog, detail-404, instance create/list/migrate, impl create with overlap rejection, slot add with duplicate rejection, sweep result shape, preview resolution).

### 6.5 Type definitions
- [ ] `ui/src/app/shared/types/class.types.ts` — `ClassCatalogEntry`, `ClassDetail`, `ClassInstanceInfo`, `ClassScheduleInfo` (impl), `ClassScheduleSlotInfo`, `UpcomingSessionInfo`, `UpcomingRunInfo`, request/response types for create/update/migrate/sweep, `ActiveScheduleView`. Re-export from `ServerAccess.ts`.

## 7. Admin Metadata (`database_helper/create_database.cpp`)

All eleven CLAUDE.md steps for EACH of `class_instances`, `class_schedules`, `class_schedule_slots`. Nesting: `class_instances` nested under `classes`; `class_schedules` nested under `class_instances`; `class_schedule_slots` nested under `class_schedules`.

- [ ] **Step 1** — confirm `db_schema/*.h` constants referenced.
- [ ] **Step 2** — `make_database_info.cpp` (done in §2.6).
- [ ] **Step 3** — `CreateTables()` (done in §2.6).
- [ ] **Step 4** — `PopulateAdminTopLevelTables()`: add `class_instances`, `class_schedules`, `class_schedule_slots` (every table with column metadata or per-table permissions MUST be here due to FK constraints — this is the most common mistake).
- [ ] **Step 5** — `PopulateAdminNestedTables()` + `PopulateAllowedTables()`: register the nesting chain above.
- [ ] **Step 6** — `PopulateAdminTablePermissions()`: map all three to `manage_class_schedule`.
- [ ] **Step 7** — `PopulateAdminColumnDataInfo()`: column edit types for every column (instance: dates as date pickers, product_id as FK picker; impl: priority numeric, dates; slot: day-of-week enum, start time, duration, facility/room/instructor FK pickers).
- [ ] **Step 8** — `PopulateAdminColumnFriendlyNames()`.
- [ ] **Step 9** — `PopulateAdminTableFriendlyNames()`: "Class Instances", "Class Schedules (Implementations)", "Class Schedule Slots".
- [ ] **Step 10** — `PopulateAdminTableDisplayTemplates()`: FK-picker display strings (instance: `"{name} ({valid_from_us}..{valid_to_us})"`; impl: `"{name} (priority {priority})"`; slot: `"{day_of_week} {start_time_minutes}min room {location_room_id}"`).
- [ ] **Step 11** — `CMakeLists.txt` updates (done in §2.6 + §3).
- [ ] Also add `classes.kind` to `PopulateAdminColumnDataInfo` as an enum and its friendly name.

## 8. Permissions
- [x] `manage_class_schedule` permission seeded in `PopulatePermissions`; granted to admin + Studio Manager roles in `PopulateRolePermissions`. (Carried from the prior implementation — unchanged by the redesign.)

## 9. Seed Data
- [~] Existing seeded classes (Knotty Yoga, Therapeutic Knotty Yoga, Partner Acrobatics, Tumbling, Handstands, Aerial Fabric) get `kind='recurring'` (all, via the DDL default) — but only **Knotty Yoga** currently gets a perpetual instance + default impl + slots. Seeding the instance/impl/slots for the other five is pending.
- [x] `PopulateClassSchedules` (kept name): for Knotty Yoga, seeds a perpetual instance + a default impl with Mon+Wed 18:00 / 60min slots in the Main Gym at Knotty Yoga Studio. **No materialize call** — the catalog detail page derives sessions on the fly.
- [x] Keep the `kind='class'` `class-dropin` product. Cancellation policies already seeded.

## 10. Tests Summary
- [x] Table helpers: `class_instances_test.cpp`, `class_schedules_test.cpp`, `class_schedule_slots_test.cpp`, extended `event_sessions_test.cpp`.
- [x] Business logic: `class_instance_helper_test.cpp`, `class_schedule_helper_test.cpp`, `class_catalog_helper_test.cpp`, extended `scheduling_key_value_table_test.cpp` (+ sweep tests). *(Room-occupancy tests pending — §4.3 not built.)*
- [~] Endpoint tests: catalog (`get_classes`, `get_class_detail`) + the four admin implementation endpoints (create/update/deactivate/list) done with success/permission/validation paths. Instance, slot, and preview endpoint tests pending (those endpoints not built).
- [ ] Frontend: component specs for catalog, detail, class list/detail, instance detail, impl detail, slot editor, preview, sweep-confirmation modal.
- [ ] `ServerAccess.mock.spec.ts` updated.
- [ ] Manual-testing-helper commands: `list_class_instances`, `list_class_schedules <instance_id>`, `preview_schedule <class_id> <date>` (no materialize command).

## 11. Cross-Layer Acceptance Criteria
A new admin on a fresh DB can:
- [ ] Open `portal/manage/class-schedules`, pick a `kind='recurring'` class, create/confirm its perpetual instance, add a default impl with three slots (Mon 6pm, Wed 6pm, Sat 9am), and see them sorted.
- [ ] Add a higher-priority "Holiday Week" impl under the same instance with the Monday slot removed; use "schedule on date X" preview to confirm the holiday-week behavior during the window and the default behavior outside it — with no materialize step anywhere.
- [ ] Open the public `/classes` route as a logged-out visitor, see the class, click in, and see derived upcoming sessions (eight Mondays + eight Wednesdays + eight Saturdays minus the holiday-week Monday).
- [ ] Create a `kind='workshop'` class, add two instances (Aug 15 2026 + Mar 22 2027) each with its own product + a single-slot bounded impl; confirm the catalog detail page lists both upcoming runs.
- [ ] Migrate the recurring class to a new product effective a future date; confirm the perpetual instance closes at that date and a new one opens with the new product and copied-forward slots.

## 12. Resolved Questions
All Phase 1 questions resolved via [[Class Schedule Implementations Redesign]] §2.0 (L-1..L-9) and §2.1 (OQ-CSI-1..21). Highlights: three-level hierarchy (L-1), `class_instances` name (L-2/OQ-CSI-1), product on instance (L-3/OQ-CSI-2), instance layer always present incl. recurring (L-4), lazy derivation (OQ-CSI-12), keep `event_sessions` (OQ-CSI-13), instructor on slot (OQ-CSI-14), `classes.kind` enum (OQ-CSI-18), sweep-on-save (OQ-CSI-19), product migration UX (OQ-CSI-20), slot copy-forward (OQ-CSI-21).

## 13. Cross-References
- Design source: [[Class Schedule Implementations Redesign]].
- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 1, P-7.
- Predecessor work: [[Scheduling thin slice]], [[Provider Portal]].
- Feeds into: [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 5 - Attendance Templates]], [[Classes Phase 7 - Class Series and Workshops]], [[Classes Phase 10 - Scheduling Exceptions and Shift Trades]].

---

# Appendix — Superseded original flat-model plan (for history)

The original Phase 1 (Version 0.1) implemented a flat single-row `class_schedules` table (class + facility + room + `recurrence_pattern` + `days_of_week` + single `start_time_minutes` + `duration_minutes` + `effective_*` + `is_series` + `series_*`), a `MaterializeFutureSessions(scheduleId, throughDateUs)` business-logic method, an admin "Materialize through date" button, and `event_sessions.class_schedule_id`. It was fully implemented and tested against that model. The 2026-05-28 redesign supersedes it: the flat table splits into the three-level hierarchy, materialization is replaced by lazy derivation, `is_series`/`series_*` move to Phase 7's `class_series_instances`, and `event_sessions` keys off `class_schedule_slot_id` + `occurrence_date_us`. The prior decisions (OQ-P1-1 days_of_week format, OQ-P1-2 predecessor chain length, OQ-P1-3 edit-regenerate booked-session handling) are subsumed by the redesign's L-/OQ-CSI- decisions and are retained only in git history.
