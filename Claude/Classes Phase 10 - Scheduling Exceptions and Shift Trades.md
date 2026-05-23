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

**Must / Should-have.** Studio closures (existing `scheduling_exceptions` mechanism — extend to also cascade-cancel class instances). Admin single-instance cancel (existing `SessionCancellationHelper` — extend to handle the mixed paid + zero-money attendee population correctly). Instructor substitution (no refund, no email — homepage display only per the resolved OQ-19). Instructor-initiated shift trades / transfers (reuse `ShiftChangeHelper` from [[Provider Portal]] with a parallel path keyed off `event_session_staffing` rows). Admin "who's teaching what" grid view.

**Prerequisites:**
- Phase 1 (class_schedules, event_sessions with class linkage).
- Phase 2 (refund pro-rating — admin cancel of paid bookings refunds, zero-money bookings don't).
- Phase 4 (cancellation `.ics` with STATUS:CANCELLED).
- Existing `SessionCancellationHelper`, `ShiftChangeHelper`, `scheduling_exceptions` infra ([[Provider Portal]], [[Event Polish- Scheduling Should Have Items]]).

**Outcome:**
- Studio-closure dates cascade-cancel class instances in addition to service sessions.
- Single-instance admin cancellation handles mixed paid / zero-money attendees correctly.
- Instructor substitution flow updates `event_session_staffing`; homepage / calendar show the new instructor; NO emails sent.
- Class instructor shift-trade flow mirrors Provider Portal's flow but operates on `event_session_staffing` rows (not `provider_availability`).
- Admin "who's teaching what" grid view.

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
- [ ] Verify existing `scheduling_exceptions` cascade. It was designed for service sessions; we need it to also cancel class `event_sessions` on a date range. Read the existing `business_logic/scheduling/scheduling_exception_helper.*` (or equivalent) to confirm what it touches and where to extend.
- [ ] Verify `ShiftChangeHelper`. It's keyed off `provider_availability` for service providers. For classes, the shift is `event_session_staffing.person_id` for a specific session, NOT a `provider_availability` block. We need a parallel code path.

### 1.2 Locked-in
- [x] Instructor substitution: NO refund (parent §2.14 ST-4) and NO email (parent OQ-19) — homepage + calendar display only.
- [x] Admin single-session cancellation: full refund for paid bookings, capacity-release-only for zero-money bookings (parent SE-5 + Phase 2 refund pro-rating).
- [x] Class shift trade: no `free_cancel_until_us` extension (different from services — parent ST-5).

## 2. Database Schema

### 2.1 Extend `scheduling_exceptions` cascade
- [ ] No new columns; verify the existing CASCADE handler in the helper iterates BOTH `bookable_service_sessions` AND `event_sessions` matching the (date, facility) of the exception. If not, extend in §4.1.

### 2.2 Extend `shift_change_requests` table
- [ ] Add `event_session_staffing_id BIGINT` NULL `REFERENCES event_session_staffing(id)` — for class-assignment shift trades (distinct from `provider_availability_id` for service providers).
- [ ] CHECK constraint: exactly one of `provider_availability_id` and `event_session_staffing_id` is non-NULL.
- [ ] Document the union semantics in the table helper file header comment.

### 2.3 Wire schema into DB init
- [ ] Update `make_database_info.cpp` / `create_database.cpp` for any column additions.

## 3. Table Helpers

### 3.1 Extend `TableHelpers::SchedulingExceptions`
- [ ] If `GetExceptionsForDateRange(Transaction&, dateFromUs, dateToUs, facilityId)` exists, use it. Otherwise add it.
- [ ] Tests.

### 3.2 Extend `TableHelpers::ShiftChangeRequests`
- [ ] Surface `event_session_staffing_id`.
- [ ] `GetClassShiftRequestsForPerson(Transaction&, personId)` — filter to requests where `event_session_staffing_id IS NOT NULL` and the requesting or target person is this instructor.

### 3.3 Reuse `TableHelpers::EventSessionStaffing`
- [ ] Already exists from [[Scheduling thin slice]] / Phase 1. Confirm methods present: `GetStaffingBySession`, `AddStaffing`, `UpdateStaffing`, `DeleteStaffing`. Tests in place.

## 4. Business Logic

### 4.1 Extend `SchedulingExceptionHelper` cascade
- [ ] In the existing helper (likely `business_logic/scheduling/scheduling_exception_helper.h/.cpp`), extend the date-range scan to also pick up `event_sessions` rows in the same facility on the same date(s).
- [ ] For each picked-up `event_sessions` row: call `SessionCancellationHelper::CancelSession(sessionId, "Studio closure: <reason>")` — which handles the refund pro-rating in §4.2 below.
- [ ] Per-provider exceptions (provider_id non-NULL): leave class sessions alone (the closure is targeted at a provider, not the studio). If the class is taught by that provider, the **instructor substitution** flow handles it separately.
- [ ] Tests cover: facility-wide closure cancels class + service sessions; per-provider closure cancels only service sessions.

### 4.2 Extend `SessionCancellationHelper`
- [ ] Check that the existing helper handles mixed paid + zero-money bookings correctly. The current refund flow likely assumes every booking has a paid purchase — verify by reading.
- [ ] Extend to:
  1. For each `booking WHERE event_session_id=? AND status='confirmed'`:
     - If `purchase_id IS NOT NULL` → issue full refund via `RefundHelper::ProcessRefund(purchase_id)`. Queue cancellation email with `STATUS:CANCELLED` iCal (Phase 4 hookup).
     - If `purchase_id IS NULL` → no refund, just `booking.status='cancelled'`, decrement `booked_count`, queue cancellation email (without refund line).
  2. Mark `event_sessions.status='cancelled'` and set `cancellation_reason`.
- [ ] Tests: mixed-population cancellation refunds the paid attendees and decrements capacity for the rest.

### 4.3 New `InstructorSubstitutionHelper`
Files: `business_logic/scheduling/instructor_substitution_helper.h/.cpp/_test.cpp`.

- [ ] `struct SubstituteRequest { int64_t eventSessionId; int64_t newInstructorPersonId; std::string reason; int64_t adminPersonId; }`.
- [ ] `bool Substitute(Transaction&, const SubstituteRequest&)`:
  1. Load `event_session_staffing` for the session.
  2. Find the existing `'instructor'` or `'lead_instructor'` row (assume one primary; if multiple, take the first ordered by id).
  3. Update `person_id` to the new instructor; append substitution note ("Substituted by adminPerson reason: ...") to a `notes` column (add the column if missing — `event_session_staffing.notes TEXT`).
  4. NO emails sent (per parent OQ-19 / ST-4). Homepage will reflect the new instructor on next render.
  5. Return ok.
- [ ] Tests.

### 4.4 Extend `ShiftChangeHelper` for class assignments
- [ ] Add overload methods that take `event_session_staffing_id` instead of `provider_availability_id`.
- [ ] `CreateClassShiftChangeRequest(Transaction&, request_type, requesting_person_id, target_person_id, event_session_staffing_id, notes)` — creates a `shift_change_requests` row with `event_session_staffing_id` set.
- [ ] `RespondToClassShiftRequest(...)` — same as service path but operates on `event_session_staffing`.
- [ ] `ReviewClassShiftRequest(...)` — admin approval if affected_bookings > 0; same audit semantics as service path.
- [ ] `ExecuteClassShiftChange(...)` — reassigns `event_session_staffing.person_id`. NO `free_cancel_until_us` set on attendee bookings (parent ST-5). NO emails to attendees (parent OQ-19).
- [ ] Tests including the audit path with bookings present.

### 4.5 Admin "who's teaching what"
- [ ] Read-only in `business_logic/scheduling/staffing_helper.h/.cpp` (already exists from [[Scheduling thin slice]]). Add:
  - `struct InstructorLoadRow { int64_t personId; std::string firstName; std::string lastName; int64_t totalSessionsInRange; int64_t totalConfirmedAttendees; }`.
  - `std::vector<InstructorLoadRow> GetInstructorLoad(Transaction&, int64_t facilityId, int64_t fromUs, int64_t toUs)` — joins `event_session_staffing` ↔ `event_sessions` ↔ `bookings` with `COUNT/SUM` aggregations.
- [ ] Test.

## 5. Endpoints

### 5.1 Admin instructor-substitution
- [ ] `POST /api/admin/event_session/<id>/substitute` body `{ new_instructor_person_id, reason }`. Permission `manage_class_schedule`. Endpoint test.

### 5.2 Provider portal class-shift-trade
- [ ] `POST /api/provider/class_shift_change_request` body `{ request_type, target_person_id, event_session_staffing_id, notes }`. Permission `provider` (existing). Endpoint test.
- [ ] Extend the existing `GET /api/provider/my_shift_requests` to include class-shift requests (filter by `event_session_staffing_id IS NOT NULL`).
- [ ] Extend `POST /api/provider/respond_shift_request/:id` and `POST /api/admin/review_shift_request/:id` to handle the class-shift variant via the new `ShiftChangeHelper` overloads.

### 5.3 Admin "who's teaching what"
- [ ] `GET /api/admin/instructor_load?facility_id=&date_from=&date_to=`. Permission new `view_admin_instructor_load`. Endpoint test.

### 5.4 Routing + permissions
- [ ] All registered in `web_app.cpp`.
- [ ] New permission `view_admin_instructor_load` seeded.

## 6. Frontend

### 6.1 Admin scheduling-exceptions page (extension)
- [ ] In the existing scheduling-exceptions admin page, the "block dates" action already cascades to service sessions; ensure the affected-class-instances count and list are surfaced in the confirmation dialog.
- [ ] Spec update.

### 6.2 Admin instructor-substitution dialog
- [ ] `ui/src/app/pages/portal/manage/event-session-detail/substitute-instructor-dialog/substitute-instructor-dialog.component.*/.spec.ts`.
- [ ] Form: new instructor picker (FK to people filtered by `instructor` permission) + reason field + confirm.

### 6.3 Admin "who's teaching what" grid
- [ ] `ui/src/app/pages/portal/manage/instructor-load/instructor-load.component.*/.spec.ts`.
- [ ] Date range picker + facility filter; table with instructor name + session count + attendee count.

### 6.4 Provider portal extension
- [ ] In the existing `shift-requests` provider-portal page, surface class-shift requests alongside service-shift requests. Visually distinguish (chip "Class") via the `event_session_staffing_id` presence.
- [ ] Spec update.

### 6.5 Homepage display of substitute instructor
- [ ] Phase 5's today-classes feed already loads `event_session_staffing` for each session — when an instructor changes (substitution OR shift trade), this list reflects the change on next render. Add a subtle "Substitute: <new name>" chip when the staffing row's `notes` field starts with "Substituted by".
- [ ] Spec.

### 6.6 `ServerAccess` extensions
- [ ] `substituteInstructor(eventSessionId, newInstructorPersonId, reason)`.
- [ ] `submitClassShiftChangeRequest(req)`, etc. — likely just additions on top of the existing provider shift APIs.
- [ ] `getInstructorLoad(facilityId, dateFrom, dateTo)`.
- [ ] Update `ServerAccess.mock.spec.ts`.

## 7. Admin Metadata

- [ ] No new top-level tables; permission `view_admin_instructor_load` added in 5.4.

## 8. Tests-Required Summary

- [ ] Table helper tests for the `shift_change_requests.event_session_staffing_id` column.
- [ ] `scheduling_exception_helper_test.cpp` extension: cascade to class sessions for facility-wide closures.
- [ ] `session_cancellation_helper_test.cpp` extension: mixed paid + zero-money attendees handled.
- [ ] `instructor_substitution_helper_test.cpp` new.
- [ ] `shift_change_helper_test.cpp` extension: class-shift variant; no email; no free-cancel offering.
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
- [ ] Tina accepts → admin reviews if there are confirmed paid attendees (workshops/series only); otherwise auto-approves.
- [ ] On approval: `event_session_staffing.person_id = tinaId`; no free-cancel email to attendees; no compensation email; homepage reflects.

## 10. Open Questions

- **OQ-P10-1.** When admin substitutes an instructor for a session that's part of a series, do we substitute just that one instance, or the rest of the series? Recommended: just that one instance — admin manually substitutes others if needed; safer than implicit cascade.
- **OQ-P10-2.** For the "who's teaching what" grid, should the date range default to "this week" or "next 30 days"? Recommended: "next 30 days" — gives admin operational visibility for planning ahead.
- **OQ-P10-3.** Class shift-trade with paid attendees (workshops / series) — does the affected-count gate require admin approval, similar to the provider path? Recommended: yes, mirror the existing provider behavior (`shift_change_booking_block_days` secret).

## 11. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 10.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 4 - iCal Generator Extensions]].
- Reuse: [[Provider Portal]] (ShiftChangeHelper, scheduling_exceptions), [[Event Polish- Scheduling Should Have Items]] (SessionCancellationHelper).
- Refund integration: [[Vouchers and Refunds]].
