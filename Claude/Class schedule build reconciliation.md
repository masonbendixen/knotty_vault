---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/28/2026
Version: 0.1
tags:
---
# Overview

This is a **discussion / decision document**, not yet an implementation plan. It exists because the Classes Phase 1 redesign landed the new three-level `db_schema` (kept) while the higher layers (`business_logic/scheduling/`, `sql_util/table_helpers/`) were rolled back to their committed flat-model versions. Those two halves are now incompatible. The goal of this doc is to (1) state exactly what broke and why, (2) lay out the options for getting to a coherent, building, *correct* tree, and (3) collect the decisions needed from Mason before any more code is written.

Leave this Overview intact. Work the sections below. Related docs: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Class Schedule Implementations Redesign]].

# 1. Current state of the tree

**Kept (working-tree changes, the new model):**
- `db_schema/`: redesigned `class_schedules` (now the *implementation* layer: `class_instance_id`, `priority`, `valid_from/to_us`), new `class_instances`, new `class_schedule_slots`, `event_sessions` re-keyed (`class_schedule_slot_id` + `occurrence_date_us` + `class_id`; **dropped** `class_schedule_id`), `classes.kind`.
- `database_helper/create_database.cpp` + `make_database_info.cpp`: wiring + seed for the above.

**Rolled back to committed (the old flat model):**
- `business_logic/scheduling/` (incl. `class_schedule_helper`, `class_catalog_helper`, `recurring_session_helper`, `scheduling_key_value_table`)
- `sql_util/table_helpers/` (incl. `class_schedules`, `event_sessions`)

The old flat model assumed `class_schedules` carried `class_id`, `facility_id`, `location_room_id`, `product_id`, `recurrence_pattern`, `days_of_week`, `start_time_minutes`, `duration_minutes`, `effective_from/to_us`, `capacity`, `is_series`, `series_*`, `predecessor_*`, and that `event_sessions` carried `class_schedule_id`. **None of those columns exist anymore.**

# 2. What the build actually reported

`knotty_yoga_core` compiled all but **three** files before ninja stopped. Everything else (endpoints, table helpers, other scheduling helpers) compiled.

| File | Break | Size of fix |
|------|-------|-------------|
| `recurring_session_helper.cpp` | 1 ref: `kEventSessionsClassScheduleId` (lines ~219–222, the class_schedule_id propagation block) | **Trivial** — delete the block (class occurrences no longer link via this helper). |
| `class_catalog_helper.cpp` | 1 ref: `kClassSchedulesProductId` (pricing block ~206–220) — product moved from `class_schedules` to `class_instances` | **Small** — resolve product via the active `class_instances` row instead. |
| `class_schedule_helper.cpp` | ~30 refs across recurrence/series/facility/product/effective columns + `kEventSessionsClassScheduleId` | **Large** — the whole file is the flat-model materializer; no meaningful minimal patch. |

# 3. The trap: "compiles" ≠ "correct"

The compile break is small, but it understates the problem. Much of the old flat-model code **compiles but is runtime-broken** against the new schema because its SQL uses string literals, not the C++ column constants that the compiler can flag. Confirmed/likely runtime-broken (compiles today):

- `TableHelpers::ClassSchedules` (committed) — `GetSchedulesByClass`, `GetScheduleByEventSession` (queries the dropped `event_sessions.class_schedule_id`), `GetActiveSchedulesByFacility`, `GetSchedulesPotentiallyConflictingInRoom` all reference flat-model columns in literal SQL.
- Flat-model endpoints that compiled but call now-incompatible helpers: `admin_class_schedule_create`, `admin_class_schedule_update`, `admin_class_schedule_deactivate`, `admin_class_schedules_list`, `admin_class_schedule_materialize`.
- KVT converters tied to flat results: `MaterializeResult`, `CreateClassScheduleResult`, `EditClassScheduleResult`, `DeactivateClassScheduleResult`.
- Tests (separate `knotty_yoga_tests` target — will fail to compile, not yet built): `class_schedules_test.cpp`, `class_schedule_helper_test.cpp`, `recurring_session_helper_test.cpp`, `class_catalog_helper_test.cpp`.
- Frontend flat-model surface: `class.types.ts`, `ServerAccess*`, `class-schedule-materialize-dialog.component.ts`.

So whichever path we pick, "make `knotty_yoga_core` compile" is a much smaller milestone than "the class-schedule feature is correct on the new schema."

# 4. Options

### Option A — Minimal compile fix only (unblock, defer the real migration)
Touch just the 3 failing files so `knotty_yoga_core` builds; accept that flat-model class paths are dead/runtime-broken until a later, deliberate migration.
- `recurring_session_helper.cpp`: drop the `class_schedule_id` write block.
- `class_catalog_helper.cpp`: resolve price via the active `class_instances` product (or, if we want truly minimal, return no price for now).
- `class_schedule_helper.cpp`: this is the blocker — there is no honest 5-line fix. To keep Option A truly minimal we'd have to **stub** its bodies (return empty/false results) or exclude it + its endpoints from the build.
- **Pros:** fastest; lets you keep iterating on `db_schema`/`database_helper`. **Cons:** leaves a misleading "green" build with dead flat-model code and broken tests; the stubbing is throwaway work.

### Option B — Forward-migrate the higher layers to the three-level model
Rebuild `class_schedule_helper` (impl + slot CRUD + lazy `GetDerivedSessionsForRange` + sweep), add `class_instance_helper`, the `class_instances`/`class_schedule_slots` table helpers, adapt `class_catalog_helper`/`recurring_session_helper`, update KVT + tests; then the flat-model endpoints/frontend get migrated too. (This is essentially §3–§5 of [[Classes Phase 1 - Catalog and Schedule Authoring]] — the work that was started and rolled back.)
- **Pros:** the correct end state; matches the redesign. **Cons:** large; spans table-helpers → business-logic → endpoints → frontend → tests; must be done as deliberate, reviewed phases (the rollback happened because it ran ahead unreviewed).

### Option C — Defer the redesign entirely (revert db_schema too)
Roll `db_schema` + `database_helper` back to committed as well, returning the whole tree to the flat model; revisit the three-level redesign later as one coherent branch.
- **Pros:** instantly green and self-consistent; zero half-migrated state. **Cons:** throws away the schema work you just chose to keep; re-does it later.

# 5. Recommendation (for discussion)

A clean sequence that avoids another unreviewed sprint:

1. **Now:** Option A, but scoped honestly — fix `recurring_session_helper.cpp` (trivial) and `class_catalog_helper.cpp` (small, via `class_instances`), and for `class_schedule_helper.cpp` decide between *stub* vs *exclude from build*. This gets `knotty_yoga_core` green so `db_schema`/`database_helper` work isn't blocked.
2. **Then, as planned phases (Option B):** migrate the higher layers deliberately, lowest-layer-first, with review between layers — table helpers (`class_instances`, `class_schedule_slots`, rework `class_schedules`/`event_sessions`) → business logic → endpoints → frontend → tests at each step.

Option C only if you'd rather not carry the new schema while the higher layers catch up.

# 6. Open questions for Mason

- **OQ-1.** Which option / sequence do you want? (Recommendation: A-now-then-B.)
	- Mason- I'm compiling is just part of it. I need unit tests to pass so option B really is the only option.
- **OQ-2.** For `class_schedule_helper.cpp` under Option A: **stub the bodies** (keep the file + endpoints compiling, methods return empty results) or **exclude the helper + its flat-model endpoints from CMake** (cleaner, but removes routes until Phase B)?
	- Mason- We need to go with option B. I'm not interested in just getting things to compile with a broken code base.
- **OQ-3.** Is a "green core build with dead flat-model class paths" acceptable as an intermediate state, or do you want the tests target green too before we call it done?
	- Mason- I will not check in to version control without all unit tests passing.
- **OQ-4.** When we do Option B, confirm the rollback lesson: land it **one layer at a time with your review between layers**, not as a single sweep?
	- Mason- I need all the tests to pass. I didn't realize what you were doing before. I thought you were moving on to the next phases. I didn't realize this was all because of the model changes. 
- **OQ-5.** Frontend flat-model surface (`class-schedule-materialize-dialog`, `class.types.ts`, ServerAccess class-schedule methods) — leave untouched until the backend migration is done, or remove the materialize UI now since the new model has no materialization step?
	- Mason- let's tackle that as a separate step after completing the server side.

# 7. Notes / scratch
(Use this section as we work through the decisions.)
