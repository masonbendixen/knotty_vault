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

## 4. Business Logic ✅ DONE

### 4.1 New `ClassClosureHelper` (replaces the cascade-extension) ✅
- [x] `business_logic/scheduling/class_closure_helper.h/.cpp/_test.cpp` (built from scratch — the §1.1 audit found no Phase 1 stub; registered in the scheduling CMakeLists).
- [x] `CloseClassesForRange(Transaction&, const std::vector<int64_t>& classIds, int64_t fromUs, int64_t toUs, std::string_view reason)` — per class: `ClassInstances::GetActiveInstance` at the window start (none → per-class skip outcome, not an error), closure window **clamped to the instance window** (a series run ending mid-window closes for the days it covers), priority = `max(100, max overlapping impl priority + 1)` so the closure always outranks special-week impls AND repeated closures never trip the same-priority-overlap validation, then `ClassScheduleHelper::CreateImplementation` with no slots, named `"Closure: <reason>"`. Returns per-class outcomes + closedCount.
- [x] Already-purchased paid occurrences NOT auto-cancelled (documented in the header; single-cancel path is §4.2's `CancelOccurrence`).
- [x] Tests (8): closed class derives nothing in-window while days outside still derive + impl is empty and reason-named; unselected class unaffected (L-9); batch closes multiple; skips class without instance while still closing the rest; clamps to a bounded instance window (asserts stored `valid_to_us`); closure outranks a priority-150 impl; repeated closure over the same window succeeds; invalid window / empty batch rejected.

### 4.2 Extend `SessionCancellationHelper` (ensure-on-action) ✅
- [x] `CancelOccurrence(Transaction&, classScheduleSlotId, occurrenceDateUs, reason)` — `EnsureSessionExists` then the existing `CancelSession`; result gains `eventSessionId`.
- [x] **Mixed-population fix in `CancelSession`:** it previously did `std::stoll(row.at(purchase_id))`, which THROWS for zero-money (NULL-purchase) bookings — the old test-file note had deferred this. Now: paid bookings refund 100% via `RefundHelper`; zero-money bookings release capacity + get the cancellation email with no refund line; both count in `confirmedCancelled`.
- [x] Session marked `cancelled` + `cancellation_reason` (existing behavior, now asserted via the occurrence path). NOTE: the cancellation email is the existing session-cancellation mail; the `STATUS:CANCELLED` .ics attachment remains a Phase 4 hookup to wire at the endpoint/mail layer (tracked for §5).
- [x] Tests (4 new): mixed paid + zero-money (1 refund, 2 cancelled, capacity 0, 2 emails); ensure-then-cancel with no prior row (idempotent ensure returns the same id); occurrence cancel with an existing session + zero-money booking (row reused, no refund, email still sent); invalid slot fails.

### 4.3 New `InstructorSubstitutionHelper` ✅
- [x] `business_logic/scheduling/instructor_substitution_helper.h/.cpp/_test.cpp` (in CMakeLists). `SubstituteRequest` exactly as planned; result carries `eventSessionId` + `staffingId`.
- [x] `Substitute`: validates the new instructor exists → `EnsureSessionExists` → finds the teaching row (`'instructor'`, else `'substitute'` from a prior sub — `'lead_instructor'` doesn't exist in this schema) → updates `person_id` + **prepends** the audit line, or seeds a fresh `'instructor'` row carrying the new person directly when the lazily-ensured session has no staffing. The audit line is prepended (not appended) so notes **start with** "Substituted by …" — §6.5's chip detection requires that prefix. The `notes` column already existed (no schema work).
- [x] NO emails (structural — the helper takes no MailHelper); single occurrence only (OQ-P10-1).
- [x] Tests (7): seeds session+staffing when none exist (audit prefix + admin name + reason pinned); updates the existing row in place (id reuse, person swapped, original note retained under the new line); double substitution stacks the audit trail on one row; sibling occurrence of the same slot untouched (OQ-P10-1); assistant rows never touched (picks the instructor row by role, not position); invalid slot → `INVALID_OCCURRENCE`; unknown instructor → `INVALID_INSTRUCTOR` with nothing persisted.

### 4.4 Extend `ShiftChangeHelper` for class assignments ✅
- [x] `CreateClassShiftChangeRequest(transaction, requestType, requestingPersonId, targetPersonId, eventSessionStaffingId, notes)` — validates: **transfers only** (a trade needs two staffing rows and the schema carries one — design note), requester must be the currently-assigned person, target ≠ requester, staffing row + target person exist. Creates via `AddClassRequest` (+ notes).
- [x] `RespondToClassShiftRequest` — same involvement/status rules as the service path; **acceptance executes the swap immediately, never `pending_admin` (OQ-P10-3)**. The generic `RespondToRequest` now detects class-kind rows up front and routes here (this also prevents the old code's `.at(requesting_availability_id)` crash on class rows and makes the existing respond endpoint work unchanged).
- [x] No admin review path for class shifts: class requests never reach `pending_admin`, so the existing `ReviewRequest` rejects them with `invalid_status` (pinned by test).
- [x] `ExecuteClassShiftChange` — reassigns `event_session_staffing.person_id` via the §3 `UpdateStaffing`, prepends a "Shift transferred from X to Y (request #N)" audit note. NO `free_cancel_until_us`, NO booking updates, NO emails (structural — helper has no mail member).
- [x] Tests (6): create round-trip with notes; validation matrix (trade / not-owner / self-target / bad staffing / bad target); **accept with a confirmed PAID attendee executes immediately — approved + autoApproved, staffing → Tina with audit note, booking untouched incl. NULL `free_cancel_until_us`, and `ReviewRequest` rejects**; generic `RespondToRequest` routes class rows; decline + requester-cancel + self-accept-rejected leave staffing unchanged; outsider/service-row/missing-id rejections.

### 4.5 Admin "who's teaching what" ✅
- [x] `InstructorLoadRow` + `GetInstructorLoad(Transaction&, facilityId, fromUs, toUs)` in `staffing_helper.h/.cpp` — single aggregate query (`event_session_staffing ↔ event_sessions ↔ people` LEFT JOIN confirmed `bookings`), teaching roles only (`instructor`/`substitute` — assistants don't headline), `status='scheduled'` sessions starting in `[fromUs, toUs)`, `facilityId 0` = all facilities, `COUNT(DISTINCT ...)` on both aggregates so dual-role staffing can't double-count, ordered by last/first name.
- [x] Tests (4): per-instructor session + attendee counts with ordering; facility filter + facility-0 + half-open range boundary; roles & statuses (substitute counts, assistant/cancelled-session don't; only confirmed bookings count); empty range.

## 5. Endpoints ✅ DONE

### 5.0 Admin class closure (batch) ✅
- [x] `POST /api/admin/close_classes` (`endpoints/admin_close_classes.h/.cpp`) body `{ class_ids: int64[], from_us, to_us, reason? }` → `ClassClosureHelper::CloseClassesForRange`. Permission `manage_class_schedule`. Returns `{ closed_count, outcomes: [{class_id, closed, class_schedule_id?, skip_reason?}] }`. Helper validation errors (NO_CLASSES / INVALID_WINDOW) → 400.
- [x] Test (`admin_close_classes_test.cpp`, 4 cases): 403 without permission; invalid-body matrix (missing/non-array class_ids, missing window, empty batch, backwards window — sane request still 200 after); 200 closes the selected class only (empty reason-named impl, derivation dark in-window, bystander class still derives); batch reports a skipped no-instance class while closing the rest.

### 5.1 Admin instructor-substitution ✅
- [x] `POST /api/admin/class_substitute` (`endpoints/admin_class_substitute.h/.cpp`) body `{ class_schedule_slot_id, occurrence_date_us, new_instructor_person_id, reason? }`. Permission `manage_class_schedule`; `adminPersonId` from the session. INVALID_OCCURRENCE → 404, INVALID_INSTRUCTOR → 400; response via `SubstituteResultToKeyValueTable`.
- [x] Test (`admin_class_substitute_test.cpp`, 5 cases): 403; missing-field matrix; 404 unknown slot (error code pinned); 400 unknown instructor; 200 ensures the session + staffing row points at the sub with the "Substituted by <admin>" note **and the test mail helper stays empty (no email, OQ-19)**.

### 5.2 Provider portal class-shift-trade ✅
- [x] `POST /api/provider/class_shift_change_request` (`endpoints/provider_class_shift_change_request.h/.cpp`) body `{ request_type, target_person_id, event_session_staffing_id, notes? }`. **Authorization decision (departs from the plan's "permission `provider`"):** the endpoint requires a logged-in session and the HELPER enforces that the requester is the person currently assigned on the staffing row — gating on the `provider` permission would lock out instructors, who teach classes but aren't service providers. Error mapping: not_found → 404, not_authorized → 403, else 400.
- [x] `GET /api/provider/my_shift_requests` already returns class rows (no kind filter in the table helper read); extended `EnrichRequestsWithDetails` to resolve class-kind rows' session details (`class_event_session_id`, `class_session_start_us/end_us`, `class_name` from the class else the product) — `event_session_staffing_id` presence is the client's kind discriminator.
- [x] `POST /api/provider/respond_shift_request/:id` needed NO routing change — §4.4's `RespondToRequest` routes class rows internally. Added an explicit `IsClassShiftRequest` guard (made public on the helper) so the service-path acceptance email is skipped for class shifts rather than relying on a swallowed `.at()` exception; the pending_admin admin-mail block is unreachable for class rows. Admin review endpoint untouched (service-path only).
- [x] Tests (`provider_class_shift_change_request_test.cpp`, 6 cases): 401 anonymous; bad-request matrix (missing fields, trade 400, self-target 400, unknown staffing 404); 403 when the logged-in user doesn't hold the shift; 200 create + row persisted + **my_shift_requests lists it with the class enrichment fields**; target accepts via the EXISTING respond endpoint with a confirmed attendee present → approved + auto_approved immediately, staffing reassigned, booking untouched, **zero emails**; uninvolved responder 403 leaves staffing unchanged.

### 5.3 Admin "who's teaching what" ✅
- [x] `GET /api/admin/instructor_load?facility_id=&date_from=&date_to=` (`endpoints/admin_instructor_load.h/.cpp`). Permission `view_admin_instructor_load`. `date_from`/`date_to` required, strictly-parsed microsecond ints with `date_to > date_from`; `facility_id` optional (omitted/0 = all). Returns `{ instructors: [...] }` via `InstructorLoadRowsToKeyValueTableArray`.
- [x] Test (`admin_instructor_load_test.cpp`, 4 cases): 403; invalid-param matrix (missing/non-numeric/backwards/empty range, bad facility — sane request still 200 after); per-instructor counts (names + session + confirmed-attendee totals); facility filter vs all-facilities vs empty-range result. Query params set via `crow::query_string` per the memory rule; every request flushes `ThreadPool`.

### 5.4 Routing + permissions ✅
- [x] All four registered in `web_app.cpp` (includes + reference vars) and `endpoints/CMakeLists.txt` (sources + tests).
- [x] New permission `view_admin_instructor_load` (constant in `db_schema/permissions.h`, seeded at id 11 in `PopulatePermissions`, granted to **admin + Studio Manager** in `PopulateRolePermissions`).

### 5.5 KVT conversions (per the layering rule) ✅
- [x] `scheduling_key_value_table.h/.cpp`: `ClassClosureOutcomeToKeyValueTable(+Array)`, `SubstituteResultToKeyValueTable`, `InstructorLoadRowToKeyValueTable(+Array)` — 7 new cases in `scheduling_key_value_table_test.cpp` (closed/skipped outcome shapes, success/error substitute shapes, load row + arrays).

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

- [x] No new top-level tables; permission `view_admin_instructor_load` added in 5.4.

## 8. Tests-Required Summary

- [x] Table helper tests for the `shift_change_requests.event_session_staffing_id` column — done in §2.4 (7 cases incl. CHECK + FK + cascade), plus schema-DSL/DDL tests for the new check-constraint support, plus §3.4's 9 method-level cases (`AddClassRequest`/`GetClassShiftRequestsForPerson`/`UpdateStaffing`).
- [x] ~~`scheduling_exception_helper_test.cpp` extension~~ — N/A under the redesign (no class cascade); closure coverage lives in the new `class_closure_helper_test.cpp` (8 cases, §4.1).
- [x] `session_cancellation_helper_test.cpp` extension: mixed paid + zero-money attendees handled (+ 3 `CancelOccurrence` cases) — done in §4.2.
- [x] `instructor_substitution_helper_test.cpp` new — 7 cases incl. the no-cascade sibling test (OQ-P10-1) — done in §4.3.
- [x] `shift_change_helper_test.cpp` extension: 6 class-shift cases; no email (structural); no free-cancel asserted; **target-accept executes immediately with a paid attendee and ReviewRequest rejects (OQ-P10-3)** — done in §4.4.
- [x] `staffing_helper_test.cpp` extension: `GetInstructorLoad` correctness (4 cases) — done in §4.5.
- [x] Endpoint tests for all four new endpoints + the shift-trade extensions — done in §5 (19 endpoint cases + 7 KVT cases).
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
