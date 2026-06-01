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
- 🟡 **§4.2 `ClassScheduleHelper`** — impl + slot CRUD, `EnsureSessionExists`, `GetDerivedSessionsForRange`, sweep, **and `GetActiveScheduleView`**: all **done**.
- 🟡 **§4.5 `ClassCatalogHelper`** — recurring catalog + detail (derived sessions, instructor, price-via-instance): **done**. Workshop/series "upcoming runs" branch **deferred**.
- 🟡 **§4.6 KVT** — catalog/instance/impl/slot/derived/result/`ActiveScheduleView` converters **done**; no standalone `SweepResult` converter (folded into `EditResult`).
- ✅ **§4.3 Room concurrent-capacity check** — done 2026-05-31: `ClassScheduleHelper::GetRoomOccupancy` + `RoomHasCapacityFor` (P-4 / CAP-7, demand capped at room capacity), wired into `AddSlot` as a `ROOM_OVER_CAPACITY` guard; 13 new tests. C++ for Mason to build.
- ✅ **§5 Endpoints — COMPLETE.** §5.1 catalog, §5.2 instance endpoints (create/update/deactivate/migrate-product/list), §5.3 impl + slot endpoints, §5.4 preview, §5.5 routing — all built + tested. (Impl create lives at `POST /api/admin/class_schedule` taking `class_instance_id`, rather than the `/class_instance/<id>/schedule` shape in the plan text.)
- ✅ **§8 permission** (`manage_class_schedule`) / ✅ **§9 seed** (`PopulateClassSchedules` on the new model).
- ✅ **§6 Frontend** — public pages (pre-existing), three-level admin UI (`ClassScheduleManageComponent`), ServerAccess + types migrated to the new model, `mock.spec` extended. (Authored without `ng build`/`ng test` — verify on next UI build.)
- ❌ **§7 admin metadata** (not verified this session), §11 acceptance — pending.

Checkboxes below reflect this. Anything still `[ ]` under §6, §7, §10 (frontend) is genuine remaining work. **Server side for Phases §2–§5 is feature-complete except the §4.5 workshop/series "runs" branch** (§4.3 room-capacity landed 2026-05-31).

- ✅ **§6.6 Requirements editor (access requirement-group authoring UI)** — **DONE 2026-05-31** (both layers). Backend: read endpoint `GET /api/admin/class/<id>/requirements` + `ClassAccessHelper::GetClassRequirements`; writes reuse generic CRUD. Frontend: `ClassRequirementsEditorComponent` embedded in the class-schedules page (AND-ed group cards, OR-ed permission chips via autocomplete, plain-language rule summary, `getClassRequirements` across the ServerAccess layer). Specs green; `ng build` compiles. Per [[Permission-based class access redesign]] §4.2; shared dependency of Phase 2 / Phase 3.

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
- [x] `class_schedule_helper.h/.cpp` + `class_schedule_helper_test.cpp`. Public surface (all implemented, incl. `GetActiveScheduleView`):
  - `CreateImplementationResult CreateImplementation(Transaction&, const CreateImplementationRequest&)` — validates parent instance exists, impl window lies within the instance window, and no same-priority overlapping impl under the instance (`OVERLAPPING_SAME_PRIORITY`). Optional `copyFromImplementationId` to seed slots (OQ-CSI-21).
  - `EditResult UpdateImplementation(Transaction&, int64_t id, const KeyValueTable& updates)` — applies the update, then runs the impl-save sweep (§4.4).
  - Slot CRUD: `AddSlot` (enforces OQ-CSI-8 duplicate guard + facility/room validity), `UpdateSlot`, `DeleteSlot` (each slot mutation also runs the sweep).
  - `int64_t EnsureSessionExists(Transaction&, int64_t slotId, int64_t occurrenceDateUs)` — idempotent: looks up `(slotId, occurrenceDateUs)` via `EventSessions::LookupBySlotAndDate`; if absent, creates the `event_sessions` row with `class_id`, `class_schedule_slot_id`, `occurrence_date_us`, computed `start_time_us` / `end_time_us` from the slot, `capacity = slot.capacity_override ?? class.default_capacity`, `status='scheduled'`. Returns the row id. This is the single entry point for the seven recording triggers.
  - `std::vector<DerivedSession> GetDerivedSessionsForRange(Transaction&, int64_t classId, int64_t fromUs, int64_t toUs)` — walks each date in range; resolves active instance (`ClassInstances::GetActiveInstance`), then active impl (`ClassSchedules::GetActiveImplementation`), expands slots matching `EXTRACT(DOW)`, and left-joins persisted `event_sessions` rows (so cancellations / subs / notes override the derived defaults). Reuses `RecurringSessionHelper::GenerateSessionDates` only as a date-walking utility.
  - `ActiveScheduleView GetActiveScheduleView(Transaction&, int64_t classId, int64_t dateUs)` — backs the admin "schedule on date X" preview (active instance + active impl + resolved slot list). ✅ Implemented + tested (`GetActiveScheduleViewResolvesActiveImpl`, `...NoInstance`).
- [x] Validation error codes mirror the §4.1 pattern; dedicated tests per code.
- [x] Tests: create-impl window-within-instance, same-priority-overlap rejection, copy-from seeding, multi-slot/day derivation, per-day-different-times derivation, priority-resolution derivation (override impl wins on its dates), closure (empty impl → no derived sessions), `EnsureSessionExists` idempotency, derived-vs-persisted left-join (a cancelled persisted row suppresses/overrides the derived default).

### 4.3 Room concurrent-capacity check ✅ DONE (2026-05-31) — C++ for Mason to build
- [x] Implemented as **two public methods on `ClassScheduleHelper`** (no separate helper class — it already holds every table helper needed): `GetRoomOccupancy(roomId, startUs, endUs, excludeSlotId=0) -> RoomOccupancy { roomValid, concurrentCapacity, existingDemand, overlappingSessionCount }` and `RoomHasCapacityFor(roomId, startUs, endUs, requestedCapacity, excludeSlotId=0) -> bool`. `GetRoomOccupancy` sums the **capped** simultaneous demand of every session overlapping the (room, window): persisted class sessions (`EventSessions::GetOverlappingSessionsInRoom`, capacity column), persisted service sessions (`BookableServiceSessions::GetOverlappingSessionsInRoom`, 1 each — no capacity column), and **derived** (not-yet-persisted) class-slot occurrences (via `ClassScheduleSlots::GetSlotsPotentiallyConflictingInRoom` + per-date active-instance/active-impl resolution + time-overlap), deduping derived occurrences that already have a persisted row. Per **CAP-7** ("more restrictive wins"), each session's demand is **capped at the room's `concurrent_capacity`**, so a lone oversized session fills (not overflows) the room; overflow only arises from genuine sharing (P-4). An unset (`<=0`) room capacity = unlimited.
- [x] Wired into **slot-add validation**: `AddSlot` now runs a `ROOM_OVER_CAPACITY` guard (new error code) via a private `CheckSlotRoomCapacity` that checks the slot's next derivable occurrence under its impl (skips cleanly when no future occurrence resolves — nothing derives ⇒ no conflict). `RoomHasCapacityFor` is the reusable primitive Phase 2 booking will call directly on a concrete session window.
- [x] Tests (13 new in `class_schedule_helper_test.cpp`): occupancy sums overlapping derived slots; derived + persisted both counted **without double-counting**; non-overlapping excluded; **service sessions counted**; `RoomHasCapacityFor` allows-under-capacity / blocks-on-overflow / excludes-own-slot (re-validation) / caps-demand-at-room / unset-capacity-unlimited / unknown-room-false; and `AddSlot` rejects-on-overflow / allows-parallel-under-capacity / allows-non-overlapping-same-room.

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
- [x] Add to `business_logic/scheduling/scheduling_key_value_table.h/cpp`: converters for `ClassCatalogEntry`, `ClassDetail`, `DerivedSession`/`UpcomingSessionInfo`, `ActiveScheduleView`, `CreateInstanceResult`, `CreateImplementationResult`, `EditResult`, `MigrateResult`, `SweepResult`. *(`InstanceEditResult` + `SlotResult` + `ActiveScheduleView` converters also added. No standalone `SweepResult` converter — `EditResult` carries the sweep result.)* (No `ClassScheduleToKeyValueTable` / `ClassInstanceToKeyValueTable` — those rows already come out of the table helpers as `KeyValueTable` and flow through `SqlUtil::KeyValueTableToJson` directly.)
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
- [x] `endpoints/admin_class_instance_create.h/cpp` + test: `POST /api/admin/class_instance`. Body → `CreateInstanceRequest`. 200 + `instance_id`; 400 + `error_code`. Tests: 403 missing perm, 400 missing field, 400 validation, 200 persists.
- [x] `endpoints/admin_class_instance_update.h/cpp` + test: `PUT /api/admin/class_instance/<int>`. Tests: 403, 404, 200 applies.
- [x] `endpoints/admin_class_instance_deactivate.h/cpp` + test: `DELETE /api/admin/class_instance/<int>`. Soft-delete. Tests: 403, 404, 200 flips `is_active`.
- [x] `endpoints/admin_class_migrate_product.h/cpp` + test: `POST /api/admin/class/<classId>/migrate_product`. Body `{ new_product_id, effective_at_us, copy_slots_forward }`. Delegates to `MigrateRecurringClassToNewProduct`. Tests: 403, 404 (no perpetual instance), 200 closes old + opens new.
- [x] `endpoints/admin_class_instances_list.h/cpp` + test: `GET /api/admin/class_instances?class_id=<id>`. Tests: 401, 400 missing class_id, 200 lists.

### 5.3 Admin implementation + slot endpoints
- [x] `endpoints/admin_class_schedule_create.h/cpp` + test: creates an implementation. Body → `CreateImplementationRequest` (incl. optional `copy_from_implementation_id`). Tests: 401, 403, 400 (validation `error_code`), 200 persists. *(Route shipped as `POST /api/admin/class_schedule` with `class_instance_id` in the body, not `/api/admin/class_instance/<instanceId>/schedule`.)*
- [x] `endpoints/admin_class_schedule_update.h/cpp` + test: `PUT /api/admin/class_schedule/<int>`. Body `{ updates, }`. Runs the sweep; response includes `{ deleted_orphan_count, blocked_rows }`. Tests: 403, 404, 200 applies + reports sweep, 409/400 when blocked by a paid booking.
- [x] `endpoints/admin_class_schedule_deactivate.h/cpp` + test: `DELETE /api/admin/class_schedule/<int>`. Soft-delete. Tests: 403, 404, 200.
- [x] `endpoints/admin_class_schedules_list.h/cpp` + test: `GET /api/admin/class_schedules?class_instance_id=<id>`. Tests: 401, 403, 400 missing param, 200 lists impls for the instance.
- [x] Slot endpoints: `POST /api/admin/class_schedule/<id>/slot`, `PUT /api/admin/class_schedule_slot/<slotId>`, `DELETE /api/admin/class_schedule_slot/<slotId>` (+ tests). Slot mutations run the sweep + return its result. Add returns 400 with `DUPLICATE_SLOT` / `INVALID_SLOT` on the OQ-CSI-8 / bounds guards.

### 5.4 Admin preview endpoint
- [x] `endpoints/admin_class_schedule_preview.h/cpp` + test: `GET /api/admin/class_schedule_preview?class_id=<id>&date_us=<t>` → `ActiveScheduleView` (active instance + active impl + resolved slots). Tests: 400 missing param, 403, 200 resolves active instance + impl + slots. *(Priority-override resolution is covered at the helper layer in `ClassScheduleHelperTest`.)*

### 5.5 Routing + testing patterns
- [x] Register all endpoints in `endpoints/web_app.cpp` (include + anonymous-namespace pointer holder) and `endpoints/CMakeLists.txt`.
- [x] **No** `materialize` endpoint exists.
- [x] Query-param endpoints use `crow::query_string` in tests.
- [x] `ValidationError → 400` confirmed via the create-endpoint tests.

## 6. Frontend (Angular)

> **§6 status (updated 2026-05-29):** Implemented and verified. §6.1/§6.2 public pages already existed against the unchanged catalog API. §6.3 admin UI consolidated into a single `ClassScheduleManageComponent`. §6.4/§6.5 migrated to the three-level model with extensive `mock.spec` coverage. The component spec (26 tests) was run headless (`ChromeHeadless`) and passes; the bundle compiles, so Angular template type-checking passed.
>
> **§6.3 UI rework (2026-05-29):** The first cut of `ClassScheduleManageComponent` was rejected by the user ("the UI is terrible … doesn't look like anything else in the website"). Reworked to match the admin look-and-feel (e.g. `product-list`): Angular Material throughout, formatted DB-name terminology (*Class Instances* / *Class Schedules* / *Class Schedule Slots*), real buttons, and ID→name joins.
>
> **§6.3 UI rework #2 — view-first nesting + dialogs (2026-05-29):** The table-based second cut was *also* rejected: it defaulted to showing add/migrate controls and forced "Open"/"Edit slots" clicks to see anything. Rebuilt as a **view-first expandable-card hierarchy** using `mat-accordion`/`mat-expansion-panel` (each panel bordered per project convention): Class → **Class Instance** cards (collapsed; expand to reveal) → **Class Schedule** cards → **Class Schedule Slot** cards. Children **lazy-load on `(opened)`** (instance→schedules via `listClassSchedules`, schedule→slots via `getFilteredTableRows('class_schedule_slots', …)`). Each card *header* carries its own **edit + deactivate/delete** icon buttons (`$event.stopPropagation()`); the expanded slot body shows a `<dl>` of day/start/duration/facility/room/instructor/capacity. **Add / Edit / Migrate are now `MatDialog`s** (`SharedModule` already bundles `MatDialogModule`), not always-visible forms — four standalone dialog components under `dialogs/`: `instance-form-dialog`, `migrate-product-dialog`, `schedule-form-dialog`, `slot-form-dialog`. Dialogs use **`mat-datepicker`** for all dates, **`mat-timepicker`** for slot start time, and a **`mat-autocomplete`** instructor picker that queries `getInstructors()` and shows each person's name + bio. The "Schedule on a date" preview now uses a `mat-datepicker` with proper spacing. Shared date/time conversions live in `class-schedule-date-util.ts` (UTC-midnight-us ↔ picker `Date`, minutes ↔ time `Date`). **Verified:** 57 specs (main component + 4 dialogs + date util) pass headless, and `ng build` compiles cleanly.

### 6.1 Public class catalog page
- [x] `ClassInfoComponent` at `ui/src/app/pages/public/class-info/` (route `/classes`). Loads `ServerAccess.getClasses()`. Grid of classes. *(Pre-existing; catalog API unchanged by the redesign. "kind badge" not shown — `ClassCatalogEntry` carries no `kind` field yet.)*

### 6.2 Public class detail page
- [x] `ClassDetailComponent` at `ui/src/app/pages/public/class-detail/` (route `/classes/:id`). Recurring: hero + description + upcoming derived sessions (facility / room / instructor) + price. 404 panel. *(Pre-existing. Workshop/series "upcoming runs" view pending the deferred §4.5 backend branch.)*

### 6.3 Admin three-level schedule UI
- [x] `ui/src/app/pages/manage/class-schedules/` area, gated by `ManageProductsGuard` (route unchanged; backend independently enforces `manage_class_schedule`). The flat-model components (list/edit/materialize-dialog) were removed.
- [x] `ClassScheduleManageComponent` — a **view-first expandable-card hierarchy** (`mat-accordion`/`mat-expansion-panel`, bordered per convention): **Class** `mat-select` → **Class Instance** cards → **Class Schedule** cards → **Class Schedule Slot** cards. Children lazy-load on panel `(opened)`. Each card header carries edit + deactivate/delete icon buttons (`$event.stopPropagation()`); the expanded slot body is a `<dl>` of day/start/duration/facility/room/instructor/capacity.
- [x] **Add / Edit / Migrate are `MatDialog`s** under `dialogs/` (`instance-form-dialog`, `migrate-product-dialog`, `schedule-form-dialog`, `slot-form-dialog`) — not always-visible forms. Dates use **`mat-datepicker`**, slot start time uses **`mat-timepicker`** (OQ-CSI-7), and the instructor field is a **`mat-autocomplete`** over `getInstructors()` showing name + bio. Duplicate slot tuples surface a friendly `DUPLICATE_SLOT` message. Date/time conversions live in `class-schedule-date-util.ts`.
- [x] **Schedule-on-date preview**: `mat-datepicker` (proper spacing) → `getClassSchedulePreview` → active instance + schedule + resolved slot list.
- [~] **Impl-save sweep**: surfaced inline (deactivate/delete report `swept_orphan_count`; a `SWEEP_BLOCKED_BY_BOOKING` result lists the blocked rows with a "cancel & refund first" message) rather than as a dedicated confirmation modal.
- [x] No materialize dialog. Specs pass headless; `ng build` compiles clean.
- [x] **Class CRUD on the page (2026-05-29)** — the class picker now has **Add / Edit / Delete class** buttons backed by a `class-form-dialog` (name, description, default capacity, kind, active). No new backend: `classes` is already a registered generic-CRUD table, so this uses `addItemFetchPrimaryKey` / `updateItem` / `deleteItem` with string values. The page now sources its class list from `getTableRows('classes')` (all columns incl. `is_active`, so inactive classes show with an "(inactive)" tag and can be reactivated) instead of the active-only `getClasses()` catalog read. Delete surfaces the FK error ("remove its class instances first") when the class still has instances.
- [x] **Class properties panel + photo editing (2026-05-29)** — selecting a class shows a read-only properties card (photo thumbnail, description, default capacity, kind, active). The **Edit class** dialog embeds the shared `PhotoUploadComponent` (`tableName="classes"`, `[tableItemId]="id"`, direct-upload mode — same control as the profile editor); the panel thumbnail re-checks `hasPhoto` and refreshes when the dialog closes. Photo edits apply immediately (independent of the field Save/Cancel), matching the profile-editor UX.
  - **Bugfix:** `PhotoUploadComponent.uploadFile` had no `.catch` on the `prepareImageForUpload(...)` chain — an undecodable image (e.g. **HEIC**, which browsers can't render) left it stuck on "Processing…" with no upload request and no error. Added a catch that clears the spinner and shows "Try a JPEG or PNG (HEIC not supported)". New `photo-upload.component.spec.ts` covers upload (table + user mode), the decode-failure path, the upload-error path, deferred upload, and delete.
  - **Note:** `/api/upload_photo` requires an **admin** session (`session.IsAdmin`), not just `manage_class_schedule`. A non-admin manager will now get a clear error rather than a hang; allowing class-photo upload under the manage permission would be a separate server change.
- [x] **SL-11 predecessor ("Requires attending")** — the slot dialog has a *Requires attending* select populated by querying eligible same-day predecessors (slots ending within the hour before this slot's start) as the day/start-time change; the picked predecessor is saved to `predecessor_class_schedule_slot_id` and shown in the slot detail. The edited slot is excluded from its own candidate list.
- [x] **Cross-class predecessors (2026-05-29)** — predecessor candidates now span **all classes** (active class/instance/schedule), not just the same schedule, so e.g. a 7pm Partner Acrobatics slot can require a 6pm Knotty Yoga slot. The candidate query joins `class_schedules→class_instances→classes` and returns `class_id`+`class_name`; the dialog labels options "Class · time–time". The slot list endpoint (`GetSlotsByImplementationWithPredecessor`) LEFT JOINs the predecessor slot's class so the read-only slot detail shows "Knotty Yoga · Mon · 6:00 PM" even for a cross-class predecessor. `GetSlotsByImplementation` stays plain (copy-forward re-inserts those rows). New frontend type `PredecessorCandidate`; `getPredecessorSlotCandidates` returns it. Also fixed the predecessor field's wrapping hint overlapping the capacity field (`subscriptSizing="dynamic"`).
- [x] **Dedicated list-slots endpoint added** — `GET /api/admin/class_schedule/<id>/slots` now backs slot listing (replacing the generic-CRUD read), and the *same* endpoint serves predecessor candidates via `?predecessor_for_day=&before_start_minutes=[&exclude_slot_id=]`. See §5.3 backend note. The earlier generic-CRUD gap is closed.

### 6.4 `ServerAccess` extensions
- [x] Added to interface + proxy + `ServerAccessNetwork` + `ServerAccessMock`: `getClasses`, `getClassDetail`, `listClassInstances`, `createClassInstance`, `updateClassInstance`, `deactivateClassInstance`, `migrateClassProduct`, `listClassSchedules(classInstanceId)`, `createClassSchedule(req)` *(req carries `class_instance_id`, not a separate `instanceId` arg)*, `updateClassSchedule(id, body)`, `deactivateClassSchedule(id)`, `addClassScheduleSlot`, `updateClassScheduleSlot`, `deleteClassScheduleSlot`, `getClassSchedulePreview`. Flat `materialize`/`facility`-scoped methods removed.
- [x] **(2026-05-29)** Added `listClassScheduleSlots(scheduleId)` and `getPredecessorSlotCandidates(scheduleId, dayOfWeek, beforeStartMinutes, excludeSlotId?)` — both hit `GET /api/admin/class_schedule/<id>/slots`; the network layer normalizes the raw KVT rows (NULL → undefined) into `ClassScheduleSlotInfo`. Backend: new table-helper `ClassScheduleSlots::GetPredecessorCandidates` (SQL: same impl+day, `(start+duration) <= $3 AND ($3-(start+duration)) < 60`, with a C++ exclude filter) + new thin endpoint `admin_class_schedule_slots_list` (registered in `web_app.cpp` + CMake), with table-helper and endpoint tests. The slot's `predecessor_class_schedule_slot_id` was already persisted by `AddSlot`/`UpdateSlot`.
- [x] `ServerAccess.mock.spec.ts` updated: instance create/list/update/deactivate/migrate (+ INVALID_WINDOW/INVALID_PRODUCT/INSTANCE_NOT_FOUND), impl create/list/update/deactivate (+ INVALID_WINDOW/SCHEDULE_NOT_FOUND), slot add/update/delete (+ DUPLICATE_SLOT/INVALID_SLOT), and preview resolution (active + dark).

### 6.5 Type definitions
- [x] `ui/src/app/shared/types/class.types.ts` — `ClassCatalogEntry`, `ClassDetail`, `UpcomingSessionInfo`, `ClassInstanceInfo`, `ClassScheduleInfo` (impl), `ClassScheduleSlotInfo`, `ActiveScheduleView`, `ClassUpdateBody`, and request/response types for instance/impl/slot create + migrate + sweep (`EditResult`). Re-exported from `ServerAccess.ts`. *(`UpcomingRunInfo` deferred with the workshop/series branch.)*

### 6.6 Access requirement-group authoring UI — the "Requirements" editor ✅ DONE (2026-05-31)

> **Added 2026-05-31 per [[Permission-based class access redesign]] §4.2.** Shared dependency of Phase 2 (membership inclusion) and Phase 3 (skill / attendance prerequisites). The permission-based access model's **schema + gate + consumers are built** (redesign §3.1/§3.2/§3.3): `permission_implications`, `class_requirement_groups`, `class_requirement_group_literals`, `booking_requirement_overrides`, `Scheduling::ClassAccessHelper`, and the closure-expanded `GetEffectivePermissionIds`. What is **missing is the authoring surface**: today a class can only be tied to a permission through the **generic admin table editor** (raw FK IDs, group/literal rows edited by hand). This deliverable gives the class-schedules page a first-class editor for a class's access rule. Backend before frontend (per convention).

**Backend ✅ DONE (2026-05-31) — C++ for Mason to build:**
- [x] **Write path = generic CRUD (no new write endpoints).** The two access tables are already generic-CRUD registered (redesign §3.1, gated by `manage_class_schedule`), so the frontend will create/update/delete groups + literals via `addItemFetchPrimaryKey` / `updateItem` / `deleteItem` exactly like the §6.3 class CRUD. Decision recorded; nothing to build server-side for writes.
- [x] **Read endpoint** `GET /api/admin/class/<classId>/requirements` (gated by `manage_class_schedule`) — `endpoints/admin_class_requirements_list.{h,cpp}` (registered in `web_app.cpp` + CMake). Returns `{ class_id, groups: [ { id, label, literal_count, literals: [ { id, kind, permission_id/permission_name/permission_description | skill_level_id/skill_level_name } ] } ] }`; empty `groups` ⇒ open to everyone. Backed by a new `ClassAccessHelper::GetClassRequirements(classId) -> ClassRequirementsView` that joins groups + literals and **resolves each permission literal's name/description** (skill literals carry their id with an empty name until Phase 3). KVT converters (`RequirementGroupToKeyValueTable` / `RequirementLiteral{,s}ToKeyValueTable{,Array}`) added to `scheduling_key_value_table`. Tests: 5 helper-level (`class_access_helper_test`: open class, resolved permission name, all-groups/OR-literals, skill-literal empty name, unknown class), 4 KVT-converter (`scheduling_key_value_table_test`), and 5 endpoint (`admin_class_requirements_list_test`: 401 anon, 403 no-perm, 400 bad id, 200 empty/open, 200 group-tree with resolved permission + skill literal).
- [x] `permission_implications` (the tier hierarchy) stays in the generic admin editor — admin-only global data, not per-class authoring. Unchanged.

**Frontend ✅ DONE (2026-05-31) — specs green, `ng build` compiles:**
- [x] **`ClassRequirementsEditorComponent`** (`class-requirements-editor.component.{ts,html,scss}`), a standalone component embedded in `ClassScheduleManageComponent` under the §6.3 class-properties panel (`[classId]` input, reloads on change). Renders the class's requirement groups as AND-ed cards; each card shows its (inline-editable) label and its OR-ed literals as removable `mat-chip`s. Empty state: "Open to everyone — no requirements."
- [x] Add / rename (inline `(blur)`) / delete a **group**; add / remove a **literal**. Permission literals are added via a `mat-autocomplete` over the permissions list (loaded from `getTableRows('permissions')`) — never a raw ID; the picker excludes permissions already used in the group. Skill literals are *displayed* if present but not addable here (deferred to Phase 3 `skill_levels`). Writes use the generic admin CRUD (`addItemFetchPrimaryKey` / `updateItem` / `deleteItem`); delete-group removes its literals first (FK order).
- [x] **Plain-language `ruleSummary()`** — e.g. *"(knotty_yoga_gold (Gold Member) or staff_comp) AND acro_6_recent"*; "Open to everyone (no requirements)." when empty. Subtitle copy notes a higher tier covers the tiers below it (closure).
- [x] **`getClassRequirements(classId)`** added to the `ServerAccess` interface + proxy + `ServerAccessNetwork` (GET `/api/admin/class/<id>/requirements`) + `ServerAccessMock` (builds the tree from the generic `class_requirement_groups` / `class_requirement_group_literals` / `permissions` tables so editor writes round-trip in mock mode). New types `ClassRequirements` / `RequirementGroup` / `RequirementLiteral` in `class.types.ts`. Added `MatChipsModule` to the shared Material imports. Specs: `class-requirements-editor.component.spec.ts` (11 cases) + `ServerAccess.mock.spec.ts` requirements suite (4 cases); manage-component spec mock extended with `getClassRequirements` + a `permissions` table.
- [x] Cross-reference noted in code comments: the booking-time **staff override** (`booking_requirement_overrides`) is a booking-flow surface (Phase 2 / Phase 8 check-in), NOT this authoring editor.

## 7. Admin Metadata (`database_helper/create_database.cpp`)

All eleven CLAUDE.md steps for EACH of `class_instances`, `class_schedules`, `class_schedule_slots`. Nesting: `class_instances` nested under `classes`; `class_schedules` nested under `class_instances`; `class_schedule_slots` nested under `class_schedules`.

> **§7 status (verified 2026-05-29):** DONE in `create_database.cpp`. All three tables are registered across `PopulateAdminTopLevelTables`, `PopulateAdminNestedTables`, `PopulateAllowedTables`, `PopulateAdminTablePermissions` (→ `manage_class_schedule`), `PopulateAdminColumnDataInfo`, `PopulateAdminColumnFriendlyNames`, `PopulateAdminTableFriendlyNames`, and `PopulateAdminTableDisplayTemplates`; `classes.kind` has column-data-info + a friendly name. This is what makes the generic CRUD endpoints work on these tables (which the §6.3 class-schedules UI and the class CRUD rely on). Two cosmetic deviations from the original wording, both intentional / non-blocking — see Step 7 and Step 9.

- [x] **Step 1** — `db_schema/*.h` constants referenced.
- [x] **Step 2** — `make_database_info.cpp` (done in §2.6).
- [x] **Step 3** — `CreateTables()` (done in §2.6).
- [x] **Step 4** — `PopulateAdminTopLevelTables()`: `class_instances`, `class_schedules`, `class_schedule_slots` registered.
- [x] **Step 5** — `PopulateAdminNestedTables()` + `PopulateAllowedTables()`: nesting chain registered.
- [x] **Step 6** — `PopulateAdminTablePermissions()`: all three mapped to `manage_class_schedule`.
- [x] **Step 7** — `PopulateAdminColumnDataInfo()`: edit types present for every column. *(Refinement: FK columns — `product_id`, facility/room/instructor — and `day_of_week` / `classes.kind` use `number`/`text` edit types rather than dedicated FK-picker / enum widgets in the generic admin editor. Non-blocking: the dedicated §6.3 class-schedules UI provides the real FK dropdowns, date/time pickers, and the instructor autocomplete, so the generic editor is the fallback path only.)*
- [x] **Step 8** — `PopulateAdminColumnFriendlyNames()`.
- [x] **Step 9** — `PopulateAdminTableFriendlyNames()`: "Class Instances", "Class Schedules", "Class Schedule Slots". *(Used "Class Schedules" rather than "Class Schedules (Implementations)" — matches the jargon-free terminology adopted in §6.3.)*
- [x] **Step 10** — `PopulateAdminTableDisplayTemplates()`: instance `"{name}"`, schedule `"{name} (priority {priority})"`, slot `"day {day_of_week} {start_time_minutes}min"`.
- [x] **Step 11** — `CMakeLists.txt` updates (done in §2.6 + §3).
- [x] `classes.kind` added to `PopulateAdminColumnDataInfo` + a friendly name. *(Edit type is `text`, not a dedicated enum widget — see Step 7.)*

## 8. Permissions
- [x] `manage_class_schedule` permission seeded in `PopulatePermissions`; granted to admin + Studio Manager roles in `PopulateRolePermissions`. (Carried from the prior implementation — unchanged by the redesign.)

## 9. Seed Data
- [~] Existing seeded classes (Knotty Yoga, Therapeutic Knotty Yoga, Partner Acrobatics, Tumbling, Handstands, Aerial Fabric) get `kind='recurring'` (all, via the DDL default) — but only **Knotty Yoga** currently gets a perpetual instance + default impl + slots. Seeding the instance/impl/slots for the other five is pending.
- [x] `PopulateClassSchedules` (kept name): for Knotty Yoga, seeds a perpetual instance + a default impl with Mon+Wed 18:00 / 60min slots in the Main Gym at Knotty Yoga Studio. **No materialize call** — the catalog detail page derives sessions on the fly.
- [x] Keep the `kind='class'` `class-dropin` product. Cancellation policies already seeded.

## 10. Tests Summary

> **§10 status (verified 2026-05-29):** Complete. The endpoint + frontend lines, marked pending when this was written, were delivered in Phases 5 and 6; the manual-testing-helper commands were added 2026-05-29.

- [x] Table helpers: `class_instances_test.cpp`, `class_schedules_test.cpp`, `class_schedule_slots_test.cpp`, extended `event_sessions_test.cpp`.
- [x] Business logic: `class_instance_helper_test.cpp`, `class_schedule_helper_test.cpp`, `class_catalog_helper_test.cpp`, extended `scheduling_key_value_table_test.cpp` (+ sweep tests). *(Room-occupancy tests added 2026-05-31 — §4.3: 13 cases covering occupancy summation, dedup, service sessions, capacity comparison, and the AddSlot `ROOM_OVER_CAPACITY` guard.)*
- [x] Endpoint tests: catalog (`get_classes`, `get_class_detail`) + **all** admin endpoints — instance create/update/deactivate/list, schedule create/update/deactivate/list, slot create/update/delete, slots-list (+ predecessor candidates), and preview — each with success/permission/validation paths.
- [x] Frontend: specs for the public catalog (`class-info`) + detail (`class-detail`), and the consolidated admin UI — `class-schedule-manage.component.spec.ts`, the five dialog specs (instance / migrate / schedule / slot / class form), and `class-schedule-date-util.spec.ts`. *(The originally-planned separate "instance detail / impl detail / slot editor / preview / sweep-modal" components were consolidated into one component + dialogs in §6.3; the sweep is surfaced inline rather than as a modal.)*
- [x] `ServerAccess.mock.spec.ts` updated (class catalog, instance/schedule/slot CRUD, preview, slot listing, cross-class predecessor candidates).
- [x] Manual-testing-helper commands (added 2026-05-29) — new `test_helper/commands/class_commands.cpp` registers (category "Classes"): `list_class_instances` (`lci`, `--class_id`), `list_class_schedules` (`lcs`, `--instance_id`), and `preview_schedule` (`ps`, `--class_id` + `--date=YYYY-MM-DD` or `--date_us`). `preview_schedule` reuses `ClassScheduleHelper::GetActiveScheduleView` for accurate active-instance/active-schedule resolution and lists the resolved slots; no materialize command. Wired in `command_runner.cpp` + `CMakeLists.txt`.

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
- Access model + the §6.6 Requirements-editor follow-up: [[Permission-based class access redesign]] (§4.2).

---

# Appendix — Superseded original flat-model plan (for history)

The original Phase 1 (Version 0.1) implemented a flat single-row `class_schedules` table (class + facility + room + `recurrence_pattern` + `days_of_week` + single `start_time_minutes` + `duration_minutes` + `effective_*` + `is_series` + `series_*`), a `MaterializeFutureSessions(scheduleId, throughDateUs)` business-logic method, an admin "Materialize through date" button, and `event_sessions.class_schedule_id`. It was fully implemented and tested against that model. The 2026-05-28 redesign supersedes it: the flat table splits into the three-level hierarchy, materialization is replaced by lazy derivation, `is_series`/`series_*` move to Phase 7's `class_series_instances`, and `event_sessions` keys off `class_schedule_slot_id` + `occurrence_date_us`. The prior decisions (OQ-P1-1 days_of_week format, OQ-P1-2 predecessor chain length, OQ-P1-3 edit-regenerate booked-session handling) are subsumed by the redesign's L-/OQ-CSI- decisions and are retained only in git history.
