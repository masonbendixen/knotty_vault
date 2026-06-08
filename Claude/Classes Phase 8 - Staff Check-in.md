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

Classes Phase 8 - Staff Check-in

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

> **Access redesign note (2026-05-31, [[Permission-based class access redesign]] §4.5):** Check-in is where attendance facts are created (P-5: staff-only attribution; CI-4: check-in creates the booking for membership-included recurring classes — there is no advance booking for those, per P-1). Those facts are exactly what **SL-10** counts toward the attendance-threshold permission. Also: when a person is **blocked by the access gate**, the logged staff **override** writes `booking_requirement_overrides` via `ClassAccessHelper::RecordOverride` (built, redesign §3.2) — the check-in UI is a natural home for that override action. No access-model change; note the dependency + the override surface.

## Phase Summary

**Must-have, benefits from earlier phases.** Staff opens a check-in screen for a class session within the configurable window (default −1h to +3h around session start), sees a pre-populated list (template attendees + paid bookings + last-4-weeks attendance), can autocomplete-search additional people, and marks them attended with one click. For membership-included recurring classes, the check-in itself **creates** the `booking` row (`purchase_id IS NULL`); for paid bookings, the check-in updates the existing booking's `checked_in_us`.

Per P-5: staff is the only role that records attendance. NO kiosk / self-check-in (rejected — see §4 Alternatives Considered in parent doc).

**Prerequisites:**
- Phase 1 (the three-level schedule model; `event_sessions` carries `class_id` + `class_schedule_slot_id` + `occurrence_date_us`; `ClassScheduleHelper::EnsureSessionExists`).
- Phase 2 (visibility / pricing — knows whether a class is membership-included).
- Phase 3 (skill levels — staff override path needs the skill check).
- Phase 5 (attendance templates — pre-pop list reads from template entries, now slot-keyed).
- Existing `BookingHelper` (waitlist, status flags).
- Existing `EventReminderHelper` pattern for hourly job (`FinalizeAttendance`).

### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]]
Small but load-bearing under the lazy model: **check-in is recording trigger #6.** A membership-included class occurrence usually has NO persisted `event_sessions` row when staff opens the check-in screen — it's derived. So:
- The check-in screen opens against a **derived occurrence** identified by (`class_schedule_slot_id`, `occurrence_date_us`), not necessarily an existing `event_session_id`.
- Checking someone in calls `ClassScheduleHelper::EnsureSessionExists(slotId, occurrenceDateUs)` first (idempotent — returns the existing row if a prior check-in / note already created it), then creates the `booking` with `checked_in_us` set + `purchase_id IS NULL`.
- The pre-pop "template attendees" list joins `attendance_template_entries` by **`class_schedule_slot_id`** (slot-keyed, per the Phase 5 redesign), minus skip-exceptions keyed by (`class_schedule_slot_id`, `occurrence_date_us`).
- The "last-4-weeks attendance" lookup still joins via the denormalized `event_sessions.class_id` (only persisted/attended rows have that, which is exactly what we want).

**Outcome:**
- Staff portal: per-session check-in page with autocomplete + pre-pop list.
- Walk-in flow: typed-name autocomplete; if no person, create on the spot (existing pattern).
- Configurable secrets for the pre-window and post-window.
- Hourly job marks `no_show` on unchecked paid bookings whose session ended.
- Skill-level override path with logged reason.

## Layering & Conventions

Lowest layer first:

1. `db_schema/` — no new tables.
2. `sql_util/table_helpers/` — small extensions for the 4-week-history query.
3. `business_logic/scheduling/` — `ClassCheckinHelper`.
4. `endpoints/` — four new endpoints.
5. Scheduled jobs.
6. Angular: staff portal check-in page.
7. Tests.

## 1. Pre-Coding Design Decisions

### 1.1 Locked-in
- [x] Membership-included recurring class attendance: NO advance booking; check-in creates the booking (parent §2.4 / CI-4).
- [x] Paid bookings (workshops / series / intro / guest-pass): booking already exists; check-in sets `checked_in_us`.
- [x] No homepage check-in badge (rejected — see Alternatives Considered).
- [x] No kiosk / self-check-in (P-5).
- [x] Instructor change is NOT emailed (homepage display only) — confirmed Phase 10.

### 1.2 Check-in window
- [x] Default −60min before / +180min after session end.
- [x] Configurable via secrets `class_checkin_window_before_minutes` and `class_checkin_window_after_minutes`.

### 1.3 No-show finalization
- [x] Hourly job calls `FinalizeAttendance(eventSessionId)` for sessions whose post-window has elapsed.
- [x] Applies to **paid** bookings only — sets `status='no_show'` if `checked_in_us IS NULL`.
- [x] Membership-included classes don't have advance bookings, so there's nothing to mark `no_show` for them — they're tracked implicitly via reliability metrics (Phase 16 / R-1).

## 2. Database Schema

### 2.1 Confirm reused fields
- [ ] `bookings.checked_in_us BIGINT` NULL — already present from [[Scheduling thin slice]].
- [ ] `bookings.is_walkin BOOLEAN` — already present.
- [ ] `bookings.notes TEXT` — already present (used here for "staff override reason" and walk-in info).
- [ ] `bookings.status TEXT` — must accept `'attended'` and `'no_show'` (already supported).

### 2.2 No new tables
- [ ] Verified — Phase 8 is pure business-logic + endpoint + UI work.

### 2.3 Config secrets
- [ ] Add to `config_secrets` defaults (seeded in `create_database.cpp`):
  - `class_checkin_window_before_minutes` = `60`
  - `class_checkin_window_after_minutes` = `180`
  - `class_checkin_history_weeks` = `4` (used by the pre-pop list)

## 3. Table Helpers

### 3.1 Extend `TableHelpers::Bookings`
- [ ] `GetRecentCheckedInPersonsForSchedule(Transaction&, int64_t classScheduleId, int64_t fromUs, int64_t toUs)` → list of `(person_id, last_checked_in_us)` joining `bookings` ↔ `event_sessions`. Used to seed the pre-pop list.
- [ ] `GetBookingsForSession(Transaction&, int64_t eventSessionId)` — already present from Phase 1 / [[Scheduling thin slice]]; verify it exposes `checked_in_us`, `status`, `is_walkin`.
- [ ] `MarkCheckedIn(Transaction&, int64_t bookingId, int64_t staffPersonId, int64_t nowUs)` — sets `checked_in_us=nowUs`, `status='attended'`, appends to `notes` ("checked-in by staffPerson").
- [ ] `MarkNoShow(Transaction&, int64_t bookingId, int64_t nowUs)` — sets `status='no_show'`. No-op if already attended.
- [ ] Tests.

### 3.2 Reuse `TableHelpers::AttendanceTemplateEntries`
- [ ] `GetTemplateIdsForSchedule` already exists (Phase 5). Re-used here.

## 4. Business Logic — `ClassCheckinHelper`

Files: `business_logic/scheduling/class_checkin_helper.h/.cpp/_test.cpp`.

### 4.1 Window check
- [ ] `bool IsCheckinOpen(Transaction&, int64_t eventSessionId, int64_t nowUs, const SecretsHelper&)` — pulls the window secrets and compares against `event_sessions.start_time_us` and `end_time_us`.

### 4.2 Pre-pop list
- [ ] `struct CheckinCandidate { int64_t personId; std::string firstName; std::string lastName; std::string email; std::string source; /* "template" | "paid_booking" | "history" */ bool alreadyCheckedIn; int64_t bookingId; /* 0 if no booking yet */ std::optional<int64_t> waitlistPosition; std::vector<int64_t> missingSkillLevelIds; }`.
- [ ] `std::vector<CheckinCandidate> GetCheckinList(Transaction&, int64_t eventSessionId)`. Algorithm:
  1. Load session → `classScheduleId`, `classId`, `startTimeUs`, `endTimeUs`.
  2. (a) **Paid + waitlisted bookings already on the session** (only present if the occurrence row was already ensured): `bookings WHERE event_session_id = ?` (look up the persisted row by (`class_schedule_slot_id`, `occurrence_date_us`) first; may be absent) with `source='paid_booking'`.
  3. (b) **Template attendees**: join `attendance_template_entries WHERE class_schedule_slot_id=?` → people, MINUS those with an `attendance_template_exceptions.attending=false` for this (`class_schedule_slot_id`, `occurrence_date_us`). Skip people already in (a). Source = `'template'`.
  4. (c) **4-week history**: `GetRecentCheckedInPersonsForClass(classId, now-28d, now)` via the denormalized `event_sessions.class_id`. Skip people already in (a) or (b). Source = `'history'`.
  5. Decorate each with `missingSkillLevelIds` from `SkillLevelHelper::PersonMeetsClassRequirements(personId, classId)` — UI shows a yellow flag.
  6. Sort: alphabetical by lastName, firstName.
  7. Return.

### 4.3 Check-in action
- [ ] `struct CheckInRequest { int64_t classScheduleSlotId; int64_t occurrenceDateUs; int64_t personId; int64_t staffPersonId; bool skillOverride; std::string overrideReason; }`. (Identifies the occurrence by slot + date; the `event_sessions` row may not exist yet.)
- [ ] `struct CheckInResult { bool ok; int64_t eventSessionId; int64_t bookingId; bool createdNewBooking; std::string errorCode; }`.
- [ ] `CheckIn(Transaction&, const CheckInRequest&)`:
  1. Verify `IsCheckinOpen` (compute occurrence start from the slot + date) → else `CHECKIN_NOT_OPEN`.
  2. `eventSessionId = ClassScheduleHelper::EnsureSessionExists(classScheduleSlotId, occurrenceDateUs)` (idempotent — recording trigger #6).
  3. Look up an existing booking for `(eventSessionId, personId)`.
  4. If exists → call `MarkCheckedIn`. Return.
  5. If none exists:
     - Verify the user is eligible for the class (membership-included via `CatalogHelper`) → else `NOT_ELIGIBLE`.
     - Verify skill requirements: if missing AND `skillOverride=true` AND staff has `manage_classes`, append override reason to `bookings.notes`; else `MISSING_SKILL_REQUIREMENTS`.
     - Verify capacity: query `event_sessions.booked_count + 1 ≤ capacity`; if not → `SESSION_FULL` (staff can still proceed via a separate flow, see CI-6 in parent + open questions).
     - Create new `booking` with `purchase_id = NULL`, `checked_in_us = now`, `status = 'attended'`, `is_walkin = false`. Increment `event_sessions.booked_count`.
     - Return `createdNewBooking=true`.

- [ ] `WalkInCheckIn(Transaction&, eventSessionId, walkInPersonRequest, staffPersonId)`:
  - If `walkInPersonRequest.personId == 0` (no existing person), create a `people` row using the existing person-creation pattern with `is_walkin=true` and just `firstName/lastName/(optional email)`.
  - Then call `CheckIn` with `is_walkin=true`.

- [ ] `UndoCheckIn(Transaction&, eventSessionId, personId, staffPersonId)`:
  1. Find the booking.
  2. If `is_walkin=true AND purchase_id IS NULL` → soft-delete the booking entirely (decrement `booked_count`).
  3. Otherwise reset `checked_in_us=NULL`, `status='confirmed'` (or `'waitlisted'`).
  4. Audit-trail entry in `bookings.notes`.

### 4.4 Finalize attendance (hourly job)
- [ ] `int FinalizeAttendance(Transaction&, int64_t eventSessionId, int64_t nowUs, const SecretsHelper&)`:
  1. Compute the post-window cutoff = `event_sessions.end_time_us + class_checkin_window_after_minutes * 60_000_000`.
  2. If `nowUs < cutoff` → return 0 (too early).
  3. For each `booking WHERE event_session_id=? AND status='confirmed' AND checked_in_us IS NULL AND purchase_id IS NOT NULL`: set `status='no_show'`. Return count.
- [ ] Walks all eligible sessions in a separate sweep method `FinalizePendingSessions(Transaction&, nowUs, SecretsHelper&)` — used by the hourly job.

### 4.5 KeyValueTable conversions
- [ ] `CheckinCandidateToKeyValueTable(...)`, `CheckInResultToKeyValueTable(...)`.

## 5. Endpoints

### 5.1 Staff endpoints
- [ ] `GET /api/staff/checkin/<eventSessionId>` → `{ window_open, session_info, candidates: [...] }`. Permission `staff` OR `manage_classes`. Endpoint test.
- [ ] `POST /api/staff/checkin/<eventSessionId>/person/<personId>` body `{ skill_override?, override_reason? }`. Returns `CheckInResult`. Endpoint test.
- [ ] `DELETE /api/staff/checkin/<eventSessionId>/person/<personId>` → undo. Endpoint test.
- [ ] `POST /api/staff/people/search?q=...` — autocomplete. Permission `staff`. Endpoint test (reuse if existing; otherwise create).
- [ ] `POST /api/staff/checkin/<eventSessionId>/walkin` body `{ first_name, last_name, email? }` → creates the person + check-in atomically. Endpoint test.

### 5.2 Admin/scheduler endpoint
- [ ] `POST /api/admin/finalize_class_attendance` — runs `FinalizePendingSessions(now)` over all sessions in the last 48h. Idempotent. Permission `admin`. Endpoint test.

### 5.3 Routing
- [ ] All registered in `web_app.cpp`.

### 5.4 Async-write safety
- [ ] These endpoints DO queue async writes (booking-confirmation emails are not relevant here, but the `bookings.notes` audit append may invoke logging). Apply the `ThreadPool::Shutdown()` discipline per memory `feedback_sync_sql_before_threadpool_queue.md` and CLAUDE.md.

## 6. Scheduled job integration

- [ ] Add hourly job in `knottyyoga_helper`: `POST /api/admin/finalize_class_attendance`. Idempotent.

## 7. Frontend

### 7.1 Staff check-in page
- [ ] `ui/src/app/pages/portal/staff/class-checkin/class-checkin.component.*/.spec.ts`.
- [ ] Lists today's sessions for the staff member's facility (sorted by start time); click into one → check-in screen for that session.
- [ ] On the check-in screen:
  - Session header: class name, room, instructor, current attended count / capacity, window open/closed badge.
  - Search bar at top with autocomplete (≥2 chars; debounced; uses `/api/staff/people/search`).
  - Pre-pop list grouped by source (Template Attendees, Paid Bookings, Recent Attendees) with checkboxes for "Attended" — click flips state via API.
  - "Add walk-in" button → dialog with first/last/email fields.
  - Yellow-flag chip next to anyone with missing skill requirements; clicking "Check in anyway" pops a reason-required confirmation dialog.
- [ ] Optimistic UI updates with rollback on error.
- [ ] Spec covers all four flows: regular check-in, walk-in, skill-override, undo.

### 7.2 `ServerAccess` extensions
- [ ] `getCheckinList(eventSessionId)`, `checkInPerson(eventSessionId, personId, skillOverride?, overrideReason?)`, `undoCheckIn(eventSessionId, personId)`, `walkInCheckin(eventSessionId, firstName, lastName, email?)`, `searchPeople(query)`.
- [ ] Update `ServerAccess.mock.spec.ts`.

### 7.3 Types
- [ ] `checkin.types.ts`: `CheckinCandidate`, `CheckInResult`, `CheckinSessionInfo`.

## 8. Admin Metadata

- [ ] New permissions used: `manage_classes` (reused; introduced earlier). No new admin tables.

## 9. Tests-Required Summary

- [ ] Table helper tests for `GetRecentCheckedInPersonsForSchedule`, `MarkCheckedIn`, `MarkNoShow`.
- [ ] `class_checkin_helper_test.cpp`:
  - Pre-pop list combines template + paid + history correctly without duplicates.
  - Check-in creates booking for membership-included class with `purchase_id=NULL`.
  - Check-in on existing paid booking sets `checked_in_us`.
  - Walk-in flow creates person + booking.
  - Skill override requires `manage_classes`; rejects without permission.
  - Undo deletes walk-in booking; resets `checked_in_us` for paid.
  - `FinalizeAttendance` marks `no_show` on paid + unchecked.
- [ ] Endpoint tests for all six endpoints (success + permission-denied + validation-error).
- [ ] Frontend spec for check-in page covering search, pre-pop, walk-in, skill-override, undo.
- [ ] Manual-testing-helper commands: `checkin <event_session_id> <person_id>`, `walkin_checkin <event_session_id> <first> <last>`, `finalize_attendance`.

## 10. Cross-Layer Acceptance Criteria

Tuesday 6:55pm (5min before session start of "Vinyasa Flow at Studio A 7-8pm"):
- [ ] Staff opens `/portal/staff/class-checkin`, clicks on the session.
- [ ] The pre-pop list shows: 3 template attendees, 0 paid bookings (it's membership-included), and 5 recent-history attendees (last 4 weeks).
- [ ] Staff checks in template attendee #1 → creates a `booking` with `purchase_id=NULL`, `checked_in_us=now`, `status='attended'`, `event_sessions.booked_count++`.
- [ ] Staff types "Jor" → autocomplete returns "Jordan Smith" (recent history). Staff clicks → booking created + checked in.
- [ ] Walk-in "Maya Patel" (not in system) → person created + booking created + checked in with `is_walkin=true`.
- [ ] Tries to check in "Alex" who has a skill requirement they don't meet → yellow flag appears; "Check in anyway" → reason dialog → submit → booking created with override note. Reject if staff lacks `manage_classes`.

The next morning at 11pm window-close:
- [ ] Hourly `finalize_class_attendance` job runs; for a separate paid workshop where 2 attendees never checked in, those bookings flip to `status='no_show'`.

## 11. Open Questions

- **OQ-P8-1.** When a check-in pushes the session over capacity (membership-included flow), do we hard-block, soft-warn the staff, or silently exceed? Recommended: soft-warn but allow — the studio is membership-based and capacity is aspirational for recurring classes; staff judgment wins. For paid offerings (workshops / series) where capacity is real, hard-block with `SESSION_FULL`.
- **OQ-P8-2.** Should walk-in account creation require any contact info (email at minimum) or accept name-only? Recommended: name-only is fine for check-in; capture email as a "Save for future" optional field.
- **OQ-P8-3.** Where does the staff portal expose the per-instance exception notes (Phase 5 N-7)? Recommended: a small panel on the check-in screen showing notes from members who marked attending=false for this session.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 8.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 3 - Skill Levels]], [[Classes Phase 5 - Attendance Templates]].
- Scheduler: [[Scheduled Jobs]].
- Provider portal patterns: [[Provider Portal]] (the staff-portal patterns we extend here).
