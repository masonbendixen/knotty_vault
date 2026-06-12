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

Classes Phase 10 - Scheduling Exceptions and Shift Trades

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

**Must / Should-have.** Class closures (per the redesign: create empty high-priority impls under affected classes — NOT a `scheduling_exceptions` cascade). Admin single-instance cancel (existing `SessionCancellationHelper` — extend to handle the mixed paid + zero-money attendee population, ensuring the occurrence row first). Instructor substitution (no refund, no email — homepage display only per the resolved OQ-19). Instructor-initiated shift trades / transfers (reuse `ShiftChangeHelper` from [[Provider Portal]] with a parallel path keyed off `event_session_staffing` rows). Admin "who's teaching what" grid view. Ships the closure batch UI deferred from Phase 1 (OQ-CSI-4).

**Prerequisites:**
- Phase 1 (three-level schedule model; `event_sessions` carries `class_id` + `class_schedule_slot_id` + `occurrence_date_us`; `ClassScheduleHelper::EnsureSessionExists` + `CreateImplementation`; `ClassClosureHelper`).
- Phase 2 (refund pro-rating — admin cancel of paid bookings refunds, zero-money bookings don't).
- Phase 4 (cancellation `.ics` with STATUS:CANCELLED).
- Existing `SessionCancellationHelper`, `ShiftChangeHelper`, `scheduling_exceptions` infra ([[Provider Portal]], [[Event Polish- Scheduling Should Have Items]]).

### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]] §1.7, L-8/L-9
This phase changes substantially:
- **Per-class scheduling exceptions are GONE as a separate mechanism.** A class doesn't run on a date because a higher-priority impl says so (a "Memorial Day" impl with that day's slot removed, or an empty closure impl). There is no `scheduling_exceptions` cascade for *classes*. The `scheduling_exceptions` table stays in use for **service sessions** only — untouched here.
- **No global "studio closed" lever (L-9).** Even on a closure day a workshop may run under its own class. Closures are per-class empty high-priority impls. This phase ships the **"close these N classes for this window" class-multiselect batch UI** (deferred OQ-CSI-4), backed by `ClassClosureHelper::CloseClassesForRange(classIds, fromUs, toUs)` which creates an empty high-priority impl under each selected class's active instance.
- **Ensure-on-action invariant.** Every per-occurrence write — single cancel, instructor sub, shift-trade approval — first calls `ClassScheduleHelper::EnsureSessionExists(slotId, occurrenceDateUs)` (the occurrence usually has no persisted row under the lazy model), then performs the cancel / staffing write.
- **Instructor substitution** operates on the ensured `event_sessions` row's `event_session_staffing` (overriding the slot's default `instructor_person_id`). Per ST-4, NO attendee email — the new instructor surfaces on the homepage / calendar only.

**Outcome:**
- Class closures handled via empty high-priority impls; a batch class-multiselect UI creates them in one action. Workshops on closed dates are unaffected.
- Single-instance admin cancellation ensures the occurrence row, then handles mixed paid / zero-money attendees correctly.
- Instructor substitution flow ensures the row + updates `event_session_staffing`; homepage / calendar show the new instructor; NO emails sent.
- Class instructor shift-trade flow mirrors Provider Portal's flow but operates on `event_session_staffing` rows (not `provider_availability`), ensuring the row first.
- Admin "who's teaching what" grid view (reads derived + persisted occurrences).

## Layering & Conventions

Lowest layer first:

1. `db_schema/` — minor schema extensions for shift-trade union.
2. `sql_util/table_helpers/` — extensions.
3. `business_logic/scheduling/` — extend `SessionCancellationHelper`, new `InstructorSubstitutionHelper`, extend `ShiftChangeHelper`.
4. `endpoints/` — three new endpoints.
5. Angular UI: extend scheduling-exceptions admin page; new instructor-substitution dialog; new "who's teaching what" grid; extend Provider Portal shift-trade for class assignments.
6. Tests.

## 1. Pre-Coding Design Decisions

### 1.1 Reuse audit
- [x] Class closures do NOT extend the `scheduling_exceptions` cascade. Per the redesign, a class closure is an empty high-priority impl under the class's active instance. Build/confirm `ClassClosureHelper::CloseClassesForRange` (Phase 1 stub) which creates those impls. The `scheduling_exceptions` table + its cascade remain for **service sessions** only — do not touch that path.
- [x] Verify `ShiftChangeHelper`. It's keyed off `provider_availability` for service providers. For classes, the shift is `event_session_staffing.person_id` for a specific session, NOT a `provider_availability` block. We need a parallel code path.

**Audit findings (2026-06-12):**
- **`ClassClosureHelper` does NOT exist** — no `class_closure_helper.*` files, no `CloseClasses*` symbol anywhere in `src/`. The "Phase 1 stub" never materialized, so §4.1 builds it from scratch. The building blocks it composes ARE in place: `ClassScheduleHelper::CreateImplementation` (+ `CreateImplementationRequest`), `TableHelpers::ClassSchedules::AddClassSchedule`, and the priority model in `db_schema/class_schedules.h` (higher priority wins on overlap within an instance; same-priority overlap rejected in business logic, default priority 3).
- **`ShiftChangeHelper` confirmed service-path-only**, as suspected: `RespondToRequest` / `ReviewRequest` / `CountAffectedBookings` are all keyed by availability id, and the private `ExecuteShiftChange` carries the free-cancel-deadline override — exactly the behaviors the class path must NOT inherit (§1.2 / OQ-P10-3). Parallel overloads per §4.4 stand.
- **Schema correction for §2.2:** `shift_change_requests` does not have a single `provider_availability_id` — it has TWO availability columns, `requesting_availability_id` and `target_availability_id` (see `db_schema/shift_change_requests.h`). The §2.2 CHECK constraint must be restated as: class trades set `event_session_staffing_id` non-NULL with **both** availability columns NULL; service trades set `requesting_availability_id` non-NULL (target optional, per existing semantics) with `event_session_staffing_id` NULL.

### 1.2 Locked-in
- [x] Instructor substitution: NO refund (parent §2.14 ST-4) and NO email (parent OQ-19) — homepage + calendar display only.
- [x] **Substitution affects a single occurrence only (resolved OQ-P10-1):** subbing an instructor for a session that's part of a series touches just that one instance — NO implicit cascade to the rest of the series. Admin manually subs the other instances if needed (safer than an implicit cascade).
- [x] Admin single-session cancellation: full refund for paid bookings, capacity-release-only for zero-money bookings (parent SE-5 + Phase 2 refund pro-rating).
- [x] Class shift trade: no `free_cancel_until_us` extension (different from services — parent ST-5).
- [x] **Class shift trades are NOT admin-gated (resolved OQ-P10-3):** unlike the massage/provider path, class shift trades do NOT require admin approval even when there are confirmed paid attendees (workshops / series). Swapping a fitness instructor is far less invasive than swapping a massage provider and carries no refund exposure (per the no-`free_cancel` rule above). Target acceptance executes the swap immediately; the `shift_change_booking_block_days` secret and the affected-count review gate do NOT apply to class shifts.

## 2. Database Schema ✅ DONE

### 2.1 No class-closure schema (impls cover it)
- [x] No `scheduling_exceptions` extension for classes — closures are empty high-priority `class_schedules` impls (Phase 1 tables, no new columns). The `scheduling_exceptions` cascade for service sessions is unchanged. (Confirmed — nothing to do.)

### 2.2 Extend `shift_change_requests` table
- [x] Add `event_session_staffing_id BIGINT` NULL `REFERENCES event_session_staffing(id)` — for class-assignment shift trades (distinct from the availability columns for service providers). Done in `db_schema/shift_change_requests.h/.cpp`; `requesting_availability_id` changed from NOT NULL to nullable (the CHECK enforces non-NULL for the service kind; the endpoint also still validates it at the API layer).
- [x] CHECK constraint **(corrected by the §1.1 audit — the table has `requesting_availability_id` + `target_availability_id`, not a single `provider_availability_id`)**: class trades set `event_session_staffing_id` non-NULL with both availability columns NULL; service trades set `requesting_availability_id` non-NULL with `event_session_staffing_id` NULL. Named `chk_shift_change_requests_exactly_one_subject`.
- [x] Document the union semantics in the table helper file header comment (`sql_util/table_helpers/shift_change_requests.h`).

**Implementation note (2026-06-12):** the schema DSL had no CHECK-constraint support, so it was added following the unique-constraint pattern: `TableCheckConstraint {name, expression}` + `TableInfo::AddCheckConstraint/GetTableCheckConstraints` (+ `operator==` coverage) in `sql_util/schema/table_info.h/.cpp`, `DatabaseInfo::AddCheckConstraint` passthrough, and `CONSTRAINT <name> CHECK (<expr>)` emission in `GenerateCreateTableSql` (after FK + UNIQUE constraints). Available for any future table.

### 2.3 Wire schema into DB init
- [x] Update `make_database_info.cpp` / `create_database.cpp` for any column additions. **No changes needed:** the table definition lives in `MakeShiftChangeRequestsTable` (already called after `MakeEventSessionStaffingTable`, satisfying the new FK's ordering); `CreateTables()` already creates `event_session_staffing` (line ~243) before `shift_change_requests` (~260); the table is already registered in both `admin_top_level_tables` and `admin_nested_tables`, and it has no per-column admin metadata to extend.

### 2.4 Tests ✅
- [x] Schema DSL: 6 new `TableInfoTest` cases (named/unnamed store, empty-expression throws, empty/multiple getters, equality with differing/missing constraints) + 2 `DatabaseInfoTest` cases (passthrough, unknown-table throws).
- [x] DDL generation: 3 new `DbOpsTest` SQL-string cases (named, unnamed, ordering after FK + UNIQUE constraints) + 2 live round-trip cases (valid row inserts; violating row throws against a real Postgres table).
- [x] `shift_change_requests_test.cpp`: 7 new cases — class row with staffing-only accepted (availability columns NULL); service path still works and carries NULL staffing; CHECK rejections (staffing+requesting, staffing+target, neither); FK rejection for unknown staffing id; ON DELETE CASCADE from `event_session_staffing` removes the class request row.

## 3. Table Helpers ✅ DONE

### 3.1 Reuse Phase 1 `TableHelpers::ClassSchedules` / `ClassInstances`
- [x] Class closures use the Phase 1 impl-create path (`ClassSchedules::AddClassSchedule` with an empty slot set + high priority under the class's active instance). No `SchedulingExceptions` work for classes. The `SchedulingExceptions` helper stays as-is for the service-session path. **Confirmed (2026-06-12):** `AddClassSchedule` takes a generic `KeyValueTable` (priority + validity window included), an "empty slot set" is simply not creating slot rows, and `GetImplementationsOverlapping` supports the same-priority-overlap validation §4.1 needs. No code.

### 3.2 Extend `TableHelpers::ShiftChangeRequests`
- [x] Surface `event_session_staffing_id` — `AddClassRequest(transaction, requestType, requestingPersonId, targetPersonId, eventSessionStaffingId)` inserts the class-kind row (staffing set, both availability columns NULL, status `pending`); reads are `SELECT *` so the column flows out of `GetRequest`/`GetRequestsForPerson` automatically.
- [x] `GetClassShiftRequestsForPerson(Transaction&, personId)` — `event_session_staffing_id IS NOT NULL AND (requesting_person_id = $1 OR target_person_id = $1)`, ordered `created_us DESC`.

### 3.3 Reuse `TableHelpers::EventSessionStaffing`
- [x] Audit result (2026-06-12): `AddStaffing`, `GetStaffing`, `GetStaffingForSession` (the plan's "GetStaffingBySession"), `GetStaffingForPerson`, `DeleteStaffing` all existed with tests — but **`UpdateStaffing` did NOT exist** and §4.3 (substitution) / §4.4 (`ExecuteClassShiftChange`) both need to reassign `person_id` and append notes. Added `UpdateStaffing(Transaction&, id, const KeyValueTable&)` via `DbCrud::UpdateRow` (silent no-op on missing id, matching `UpdateRequest`). The `notes` column §4.3 wants already exists — no schema work.

### 3.4 Tests ✅
- [x] `event_session_staffing_test.cpp` — 4 new `UpdateStaffing` cases: person reassignment + note (the substitution shape, other columns untouched), role-only update, missing-id no-op leaves existing rows alone, unknown-person FK violation throws.
- [x] `shift_change_requests_test.cpp` — 5 new cases: `AddClassRequest` round-trip (staffing set, availabilities NULL, pending), unknown-staffing FK throw, class-only read filters out the same person's service request while the generic read returns both, target matches / uninvolved person sees nothing, most-recent-first ordering (created_us pinned explicitly — `now_us()` can tie within one transaction).

## 4. Business Logic

### 4.1 New `ClassClosureHelper` (replaces the cascade-extension)
- [ ] `business_logic/scheduling/class_closure_helper.h/.cpp/_test.cpp`.
- [ ] `CloseClassesForRange(Transaction&, const std::vector<int64_t>& classIds, int64_t fromUs, int64_t toUs, std::string_view reason)` — for each class, find its active `class_instances`, create an empty high-priority `class_schedules` impl (no slots) over `[fromUs, toUs)` under it. Derivation then yields no occurrences for those classes in the window. Workshops/series under their own classes are untouched unless explicitly included in `classIds`.
- [ ] Already-purchased paid occurrences in the window (rare for recurring classes; possible for a series instance) are NOT auto-cancelled — the empty impl just stops *new* derivation. To refund + cancel a specific booked occurrence, admin uses the single-cancel path (§4.2). The impl-save sweep (Phase 1) refuses to silently drop a `purchase_id`-bearing row.
- [ ] Tests: closing class A for a week yields no derived occurrences for A in that window; a workshop under class B in the same window still derives.

### 4.2 Extend `SessionCancellationHelper` (ensure-on-action)
- [ ] Single-occurrence cancel takes the occurrence identity (`class_schedule_slot_id`, `occurrence_date_us`) and first calls `ClassScheduleHelper::EnsureSessionExists` to get/create the `event_sessions` row (the occurrence usually has no persisted row under the lazy model). Then:
  1. For each `booking WHERE event_session_id=? AND status='confirmed'`:
     - If `purchase_id IS NOT NULL` → full refund via `RefundHelper::ProcessRefund(purchase_id)`. Queue cancellation email with `STATUS:CANCELLED` iCal (Phase 4 hookup).
     - If `purchase_id IS NULL` → no refund; `booking.status='cancelled'`, decrement `booked_count`, queue cancellation email (no refund line).
  2. Mark `event_sessions.status='cancelled'` + set `cancellation_reason`.
- [ ] Tests: ensure-then-cancel on an occurrence with no prior row; mixed-population cancellation refunds paid attendees and decrements capacity for the rest.

### 4.3 New `InstructorSubstitutionHelper`
Files: `business_logic/scheduling/instructor_substitution_helper.h/.cpp/_test.cpp`.

- [ ] `struct SubstituteRequest { int64_t classScheduleSlotId; int64_t occurrenceDateUs; int64_t newInstructorPersonId; std::string reason; int64_t adminPersonId; }`.
- [ ] `bool Substitute(Transaction&, const SubstituteRequest&)`:
  1. `eventSessionId = ClassScheduleHelper::EnsureSessionExists(classScheduleSlotId, occurrenceDateUs)` (recording trigger #2 — the occurrence may have had no persisted row).
  2. Load `event_session_staffing` for the session; find the existing `'instructor'` / `'lead_instructor'` row, or seed one from the slot's default `instructor_person_id` if none exists yet.
  3. Update `person_id` to the new instructor; append a substitution note ("Substituted by adminPerson reason: ...") to `event_session_staffing.notes` (add the column if missing — `event_session_staffing.notes TEXT`).
  4. NO emails sent (per parent OQ-19 / ST-4). Homepage / calendar reflect the new instructor on next render (overriding the slot default).
  5. Return ok.
- [ ] **Single occurrence only (resolved OQ-P10-1):** `Substitute` operates on exactly the one `(classScheduleSlotId, occurrenceDateUs)` passed in — even when that occurrence belongs to a series run, it does NOT cascade to the run's other instances. Subbing the rest is a separate per-occurrence call by the admin.
- [ ] Tests (incl. a series-occurrence sub that leaves the sibling occurrences' staffing untouched).

### 4.4 Extend `ShiftChangeHelper` for class assignments
- [ ] Add overload methods that take `event_session_staffing_id` instead of `provider_availability_id`.
- [ ] `CreateClassShiftChangeRequest(Transaction&, request_type, requesting_person_id, target_person_id, event_session_staffing_id, notes)` — creates a `shift_change_requests` row with `event_session_staffing_id` set.
- [ ] `RespondToClassShiftRequest(...)` — same as service path but operates on `event_session_staffing`. **When the target accepts, it executes the swap immediately (resolved OQ-P10-3) — there is NO admin-approval gate, even when the session has confirmed paid attendees.** It calls `ExecuteClassShiftChange` directly.
- [ ] **No `ReviewClassShiftRequest` for class shifts (resolved OQ-P10-3).** The affected-count → admin-approval gate and the `shift_change_booking_block_days` secret apply to the **service/provider** path only; class shift trades skip review entirely. (Do not add an admin review path for class shifts.)
- [ ] `ExecuteClassShiftChange(...)` — reassigns `event_session_staffing.person_id`. NO `free_cancel_until_us` set on attendee bookings (parent ST-5). NO emails to attendees (parent OQ-19).
- [ ] Tests: target-accept executes the swap with paid attendees present and **without** any admin review step; audit note recorded; no free-cancel / no email.

### 4.5 Admin "who's teaching what"
- [ ] Read-only in `business_logic/scheduling/staffing_helper.h/.cpp` (already exists from [[Scheduling thin slice]]). Add:
  - `struct InstructorLoadRow { int64_t personId; std::string firstName; std::string lastName; int64_t totalSessionsInRange; int64_t totalConfirmedAttendees; }`.
  - `std::vector<InstructorLoadRow> GetInstructorLoad(Transaction&, int64_t facilityId, int64_t fromUs, int64_t toUs)` — joins `event_session_staffing` ↔ `event_sessions` ↔ `bookings` with `COUNT/SUM` aggregations.
- [ ] Test.

## 5. Endpoints

### 5.0 Admin class closure (batch)
- [ ] `POST /api/admin/close_classes` body `{ class_ids: int64[], from_us, to_us, reason }` → `ClassClosureHelper::CloseClassesForRange`. Permission `manage_class_schedule`. Endpoint test (verifies empty high-priority impls created; verifies a non-selected class still derives).

### 5.1 Admin instructor-substitution
- [ ] `POST /api/admin/class_substitute` body `{ class_schedule_slot_id, occurrence_date_us, new_instructor_person_id, reason }`. Permission `manage_class_schedule`. Endpoint test (verifies ensure-session + staffing override, no email). (Takes the occurrence identity, not an `event_session_id`, since the row may not exist yet.)

### 5.2 Provider portal class-shift-trade
- [ ] `POST /api/provider/class_shift_change_request` body `{ request_type, target_person_id, event_session_staffing_id, notes }`. Permission `provider` (existing). Endpoint test.
- [ ] Extend the existing `GET /api/provider/my_shift_requests` to include class-shift requests (filter by `event_session_staffing_id IS NOT NULL`).
- [ ] Extend `POST /api/provider/respond_shift_request/:id` to handle the class-shift variant via the new `ShiftChangeHelper` overloads — **target acceptance executes the swap immediately (resolved OQ-P10-3)**. Do NOT extend `POST /api/admin/review_shift_request/:id` for class shifts: there is no admin-review step for them (the admin review endpoint stays service-path only). Endpoint test asserts an accept with paid attendees executes without a review call.

### 5.3 Admin "who's teaching what"
- [ ] `GET /api/admin/instructor_load?facility_id=&date_from=&date_to=`. Permission new `view_admin_instructor_load`. Endpoint test.

### 5.4 Routing + permissions
- [ ] All registered in `web_app.cpp`.
- [ ] New permission `view_admin_instructor_load` seeded.

## 6. Frontend

### 6.1 Admin class-closure batch UI (new — deferred OQ-CSI-4)
- [ ] New "Close classes for a date range" admin action: class-multiselect + from/to date pickers + reason. Calls `close_classes`. Shows which classes will be dark for the window (and notes that workshops under their own classes are unaffected unless selected). This is NOT the service-session `scheduling_exceptions` page — that stays separate for the service path.
- [ ] Spec.

### 6.2 Admin instructor-substitution dialog
- [ ] `ui/src/app/pages/portal/manage/event-session-detail/substitute-instructor-dialog/substitute-instructor-dialog.component.*/.spec.ts`.
- [ ] Form: new instructor picker (FK to people filtered by `instructor` permission) + reason field + confirm.

### 6.3 Admin "who's teaching what" grid
- [ ] `ui/src/app/pages/portal/manage/instructor-load/instructor-load.component.*/.spec.ts`.
- [ ] Date range picker + facility filter; table with instructor name + session count + attendee count.
- [ ] **Default date range = "next 30 days" (resolved OQ-P10-2)** — on first load the picker is pre-filled today → today+30d (forward-looking operational visibility for planning), and the grid loads that range before the admin touches the picker. Spec asserts the default range.

### 6.4 Provider portal extension
- [ ] In the existing `shift-requests` provider-portal page, surface class-shift requests alongside service-shift requests. Visually distinguish (chip "Class") via the `event_session_staffing_id` presence.
- [ ] Spec update.

### 6.5 Homepage display of substitute instructor
- [ ] Phase 5's today-classes feed already loads `event_session_staffing` for each session — when an instructor changes (substitution OR shift trade), this list reflects the change on next render. Add a subtle "Substitute: {new name}" chip when the staffing row's `notes` field starts with "Substituted by".
- [ ] Spec.

### 6.6 `ServerAccess` extensions
- [ ] `closeClasses(classIds, fromUs, toUs, reason)`.
- [ ] `substituteInstructor(classScheduleSlotId, occurrenceDateUs, newInstructorPersonId, reason)`.
- [ ] `submitClassShiftChangeRequest(req)`, etc. — likely just additions on top of the existing provider shift APIs.
- [ ] `getInstructorLoad(facilityId, dateFrom, dateTo)`.
- [ ] Update `ServerAccess.mock.spec.ts`.

## 7. Admin Metadata

- [ ] No new top-level tables; permission `view_admin_instructor_load` added in 5.4.

## 8. Tests-Required Summary

- [x] Table helper tests for the `shift_change_requests.event_session_staffing_id` column — done in §2.4 (7 cases incl. CHECK + FK + cascade), plus schema-DSL/DDL tests for the new check-constraint support, plus §3.4's 9 method-level cases (`AddClassRequest`/`GetClassShiftRequestsForPerson`/`UpdateStaffing`).
- [ ] `scheduling_exception_helper_test.cpp` extension: cascade to class sessions for facility-wide closures.
- [ ] `session_cancellation_helper_test.cpp` extension: mixed paid + zero-money attendees handled.
- [ ] `instructor_substitution_helper_test.cpp` new — incl. a series-occurrence sub that does NOT cascade to sibling occurrences (resolved OQ-P10-1).
- [ ] `shift_change_helper_test.cpp` extension: class-shift variant; no email; no free-cancel offering; **target-accept executes immediately with paid attendees present and no admin-review step (resolved OQ-P10-3)**.
- [ ] `staffing_helper_test.cpp` extension: `GetInstructorLoad` correctness.
- [ ] Endpoint tests for all three new endpoints + extensions to existing shift-trade endpoints.
- [ ] Frontend specs for scheduling-exceptions update, substitution dialog, instructor-load grid, provider-shift class variant, homepage substitute chip, mock service.

## 9. Cross-Layer Acceptance Criteria

A studio-wide closure for July 4 — admin blocks the date facility-wide:
- [ ] All class `event_sessions` on July 4 (any facility specified) flip to `status='cancelled'`.
- [ ] All paid attendees receive a cancellation email + STATUS:CANCELLED iCal + a refund queued via `RefundHelper`.
- [ ] All membership-included attendees receive a no-refund cancellation email + STATUS:CANCELLED iCal.

Admin substitutes Sara → Maya for the Tuesday 7pm Vinyasa:
- [ ] `event_session_staffing` row updated; notes contain audit trail.
- [ ] Members on their template / on the calendar see "Maya" listed for that session on next page load.
- [ ] **No email sent to attendees** (verified via TestMailHelper assertion).

Instructor-initiated shift trade (Sara wants Tina to take next Tuesday's class):
- [ ] Sara files request via provider portal → `shift_change_requests` row created with `event_session_staffing_id=...`.
- [ ] Tina accepts → the swap executes **immediately, with no admin-review step — even if the session has confirmed paid attendees** (resolved OQ-P10-3).
- [ ] On execution: `event_session_staffing.person_id = tinaId`; no free-cancel email to attendees; no compensation email; homepage reflects.

## 10. Open Questions

All three resolved (Mason, 2026-06-09) and folded into the plan above (§1.2 Locked-in + the cited sections).

- **OQ-P10-1. — RESOLVED (Mason: "go with your recommendation").** Substitution touches the single passed-in occurrence only — no cascade to the rest of a series; admin subs others manually if needed. Folded into §1.2, §4.3, §8.
- **OQ-P10-2. — RESOLVED (Mason: "go with your recommendation").** The "who's teaching what" grid defaults to a forward-looking **next-30-days** range. Folded into §6.3.
- **OQ-P10-3. — RESOLVED (Mason departs from the recommendation): NO admin gate on class shift trades.** Swapping a fitness instructor is far less invasive than swapping a massage provider and carries no refund exposure, so class shift trades are NOT gated even with confirmed paid attendees — "just let people swap shifts." Target acceptance executes immediately; the `shift_change_booking_block_days` secret + affected-count review gate stay service/provider-path only. Folded into §1.2, §4.4, §5.2, §8, §9.

## 11. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 10.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 4 - iCal Generator Extensions]].
- Reuse: [[Provider Portal]] (ShiftChangeHelper, scheduling_exceptions), [[Event Polish- Scheduling Should Have Items]] (SessionCancellationHelper).
- Refund integration: [[Vouchers and Refunds]].
