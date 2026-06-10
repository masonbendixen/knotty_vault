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
- [x] **Over-capacity check-in (resolved OQ-P8-1):** membership-included flow **soft-warns but allows** (capacity is aspirational for recurring classes; staff judgment wins) — `CheckInResult.overCapacityWarning=true`, no error. Paid offerings (workshops / series) enforce real capacity as `SESSION_FULL` **at purchase time** (other phases), so there is no over-capacity new-booking-on-check-in path for them.
- [x] **Walk-in contact info (resolved OQ-P8-2):** walk-in person creation requires **both name AND email** (Mason: "name and email for everyone"). Email is NOT optional — reject with a validation error if missing.
- [x] **Per-instance exception notes (resolved OQ-P8-3):** the check-in screen shows a small panel of notes from members who marked `attending=false` for this occurrence (Phase 5 N-7); the GET check-in endpoint surfaces them.

### 1.2 Check-in window
- [x] Default −60min before / +180min after session end.
- [x] Configurable via secrets `class_checkin_window_before_minutes` and `class_checkin_window_after_minutes`.

### 1.3 No-show finalization
- [x] Hourly job calls `FinalizeAttendance(eventSessionId)` for sessions whose post-window has elapsed.
- [x] Applies to **paid** bookings only — sets `status='no_show'` if `checked_in_us IS NULL`.
- [x] Membership-included classes don't have advance bookings, so there's nothing to mark `no_show` for them — they're tracked implicitly via reliability metrics (Phase 16 / R-1).

## 2. Database Schema ✅ DONE

### 2.1 Confirm reused fields ✅
- [x] `bookings.checked_in_us BIGINT` NULL — present (`db_schema/bookings.h` `kBookingsCheckedInUs`).
- [x] `bookings.is_walkin BOOLEAN` — present (`kBookingsIsWalkin`).
- [x] `bookings.notes TEXT` — present (`kBookingsNotes`); used here for staff-override reason + walk-in info.
- [x] `bookings.status TEXT` — plain TEXT column (no enum/CHECK constraint), so it already accepts `'attended'` and `'no_show'`.

### 2.2 No new tables ✅
- [x] Verified — Phase 8 adds no `db_schema` tables.
- [x] **Schema fix surfaced during §4 (CI-4):** `bookings.purchase_id` and `bookings.purchase_item_id` were NOT NULL (`AddColumnForeignKeyRef`). Membership-included / walk-in check-in creates a booking with **no purchase**, so both were made nullable (`AddColumnForeignKeyRefNullable`, `db_schema/bookings.cpp`). Paid bookings still set them. (Caught by the §3 tests, which hit the not-null constraint.)

### 2.3 Config secrets ✅
- [x] Added via the standard two-file secrets mechanism (`util/secrets/secret_keys.h` + `secret_values.cpp` `FillInSecretsStringView`), which both seeds `config_secrets` on first run (`create_database.cpp` pulls these defaults) **and** auto-loads them into the test secrets helper:
  - `class_checkin_window_before_minutes` = `60` (`kClassCheckinWindowBeforeMinutes`)
  - `class_checkin_window_after_minutes` = `180` (`kClassCheckinWindowAfterMinutes`)
  - `class_checkin_history_weeks` = `4` (`kClassCheckinHistoryWeeks`) — used by the pre-pop list
- [x] Test: `secrets_helper_test.cpp` `ClassCheckinConfigDefaultsLoaded` asserts all three resolve to their defaults.

## 3. Table Helpers ✅ DONE

### 3.1 Extend `TableHelpers::Bookings` ✅
- [x] `GetRecentCheckedInPersonsForClass(Transaction&, int64_t classId, int64_t fromUs, int64_t toUs)` → list of `(person_id, last_checked_in_us)` joining `bookings` ↔ `event_sessions` on the denormalized `event_sessions.class_id`. Used to seed the pre-pop list. **Naming note:** implemented as `...ForClass` (keyed by `class_id`), not the doc's original `...ForSchedule` — per the Class Schedule Redesign only persisted/attended `event_sessions` carry `class_id`, which is exactly the join we want, and §4.2's history step already calls `GetRecentCheckedInPersonsForClass(classId, ...)`.
- [x] `GetBookingsForSession` — already present as `GetBookingsBySession` (SELECT * in `bookings.cpp`), so it exposes `checked_in_us`, `status`, `is_walkin`. Verified.
- [x] `MarkCheckedIn(Transaction&, int64_t bookingId, int64_t staffPersonId, int64_t nowUs)` — sets `checked_in_us=nowUs`, `status='attended'`, appends `"Checked in by person <staffPersonId>"` to `notes` (CASE-guarded so the first line isn't prefixed with a blank), bumps `updated_us`.
- [x] `MarkNoShow(Transaction&, int64_t bookingId, int64_t nowUs)` — sets `status='no_show'`, bumps `updated_us`. No-op when already attended (`WHERE status <> 'attended' AND checked_in_us IS NULL`).
- [x] Tests — `bookings_test.cpp`: `MarkCheckedInSetsAttendedAndAppendsNote` (single + double check-in note append), `MarkNoShowOnlyAffectsUncheckedBookings` (confirmed→no_show; attended→no-op), `GetRecentCheckedInPersonsForClassDedupesAndFilters` (per-person latest via MAX, window filtering, NULL-check-in excluded, other-class excluded).

### 3.2 Reuse `TableHelpers::AttendanceTemplateEntries` ✅
- [x] Reuse target is `GetTemplateIdsForSlot` (Phase 5) — template entries are keyed by `class_schedule_slot_id`, so the doc's `GetTemplateIdsForSchedule` name maps to the existing slot-keyed reader. No new code needed.

## 4. Business Logic — `ClassCheckinHelper` ✅ DONE

Files: `business_logic/scheduling/class_checkin_helper.h/.cpp/_test.cpp` (16 tests).

**Cross-cutting reconciliations vs the original prose:**
- **Secrets via DI, `now` from the DB.** The window/history config is injected once (`ClassCheckinHelper(DatabaseHelper, Secrets::SecretsHelperPtr)`), not threaded per-call as sketched. There's also a mail-less/secret-less `ClassCheckinHelper(DatabaseHelper)` ctor that falls back to the documented defaults (60/180/4). The action methods read `now` from `SELECT now_us()` internally, so `CheckIn`/`UndoCheckIn` take no clock; `IsCheckinOpen`/`FinalizeAttendance` keep an explicit `nowUs` so the job + tests stay deterministic.
- **One access gate.** Under the permission-based access redesign, "eligible" and "skill requirements" are the SAME check (`ClassAccessHelper::CheckAccess`). A blocked new-booking returns `NOT_ELIGIBLE`; the single `skillOverride` flag (with a reason, by a staffer holding `manage_class_schedule`) overrides it and records `booking_requirement_overrides` via `RecordOverride`. There is no separate `MISSING_SKILL_REQUIREMENTS` path.
- **New table-helper for the sweep.** `TableHelpers::EventSessions::GetClassSessionsEndingInWindow(from, to)` (+ test) added so `FinalizePendingSessions` finds candidates without SQL in business logic.

### 4.1 Window check ✅
- [x] `bool IsCheckinOpen(Transaction&, int64_t eventSessionId, int64_t nowUs)` — loads the session and tests `nowUs ∈ [start − before, end + after]` using the secret window (defaults 60/180). Test `IsCheckinOpenRespectsWindow` (boundaries + missing session).

### 4.2 Pre-pop list ✅
- [x] `struct CheckinCandidate` — `personId/firstName/lastName/email/source ("paid_booking"|"template"|"history")/alreadyCheckedIn/bookingId/waitlistPosition`. The plan's `missingSkillLevelIds` is replaced by the unified gate decoration: `bool meetsRequirements` + `std::vector<int64_t> failedRequirementGroupIds`.
- [x] `GetCheckinList(Transaction&, eventSessionId)` — (a) non-cancelled bookings on the session (`paid_booking`); (b) `GetTemplateIdsForSlot` → people MINUS this-occurrence `attending=false` skips (`template`); (c) `GetRecentCheckedInPersonsForClass(classId, now − weeks, now)` (`history`); deduped by person, decorated with `CheckAccess`, sorted by last then first name. Tests `GetCheckinListMergesSourcesDedupesAndSkips` + `GetCheckinListFlagsFailedRequirements`.

### 4.3 Check-in action ✅
- [x] `struct CheckInRequest { classScheduleSlotId; occurrenceDateUs; personId; staffPersonId; skillOverride; overrideReason; }` / `struct CheckInResult { ok; eventSessionId; bookingId; createdNewBooking; overCapacityWarning; errorCode; }`.
- [x] `CheckIn` — window check from slot+date (`CHECKIN_NOT_OPEN`), `EnsureSessionExists`, existing-booking → `MarkCheckedIn`, else access gate (`NOT_ELIGIBLE` unless overridden), soft-warn capacity (OQ-P8-1), create `purchase_id=NULL` attended booking + increment `booked_count`. Tests: open-class create, existing→attended (no double-count), closed window, invalid slot, gated-block, gated-override-allowed, override-ignored-without-permission, over-capacity-soft-warn.
- [x] `WalkInCheckIn(Transaction&, eventSessionId, WalkInRequest, staffPersonId)` — requires non-empty name + well-formed email (`MISSING_WALKIN_CONTACT_INFO`), reuses an existing person by email or creates one, then checks in with `is_walkin=true`. (`is_walkin` lives on `bookings`, not `people` — the plan's "people.is_walkin" was a mis-statement; there is no such column.) Tests: create+check-in, missing/malformed email, reuse-by-email.
- [x] `UndoCheckIn` — a **purchase-less** booking (walk-in OR membership-included; broadened from the plan's "is_walkin AND purchase_id NULL" since membership check-in is `is_walkin=false, purchase_id=NULL` yet was still created by the check-in) is deleted + `booked_count` decremented; a paid booking is reset to `confirmed` with `checked_in_us` cleared and an audit note. Tests: delete-membership, reset-paid, false-when-no-booking.

### 4.4 Finalize attendance (hourly job) ✅
- [x] `int FinalizeAttendance(Transaction&, eventSessionId, nowUs)` — returns 0 before `end + after`; else flips every `confirmed` + unchecked + PAID booking to `no_show` (membership/walk-in purchase-less rows are left alone). Idempotent. Test `FinalizeAttendanceMarksUncheckedPaidNoShow`.
- [x] `int FinalizePendingSessions(Transaction&, nowUs)` — sweeps `GetClassSessionsEndingInWindow(now − 48h, now)` and finalizes each. Test `FinalizePendingSessionsSweepsRecentlyEnded` (in-window finalized, out-of-window skipped).

### 4.5 Per-instance exception notes (resolved OQ-P8-3) ✅
- [x] `struct ExceptionNote { personId; firstName; lastName; note; }` + `GetExceptionNotesForOccurrence(slot, occurrenceDateUs)` — `GetExceptionsForSlotOccurrence` filtered to `attending=false` with a non-empty note, resolved via `template → person`. No SQL in business logic. Test `GetExceptionNotesForOccurrenceReturnsSkipNotes`.

### 4.6 KeyValueTable conversions ✅
- [x] `CheckinCandidateToKeyValueTable` (+ array), `CheckInResultToKeyValueTable` (incl. `over_capacity_warning`), `ExceptionNoteToKeyValueTable` (+ array) added to `scheduling_key_value_table.h/.cpp`. `failed_requirement_group_ids` is a comma-delimited id list.

## 5. Endpoints

### 5.1 Staff endpoints
- [ ] `GET /api/staff/checkin/<eventSessionId>` → `{ window_open, session_info, candidates: [...], exception_notes: [...] }`. `exception_notes` comes from `GetExceptionNotesForOccurrence` (resolved OQ-P8-3). Permission `staff` OR `manage_classes`. Endpoint test (asserts exception notes returned for an occurrence with an `attending=false` note).
- [ ] `POST /api/staff/checkin/<eventSessionId>/person/<personId>` body `{ skill_override?, override_reason? }`. Returns `CheckInResult`. Endpoint test.
- [ ] `DELETE /api/staff/checkin/<eventSessionId>/person/<personId>` → undo. Endpoint test.
- [ ] `POST /api/staff/people/search?q=...` — autocomplete. Permission `staff`. Endpoint test (reuse if existing; otherwise create).
- [ ] `POST /api/staff/checkin/<eventSessionId>/walkin` body `{ first_name, last_name, email }` — **all three required** (resolved OQ-P8-2) → creates the person + check-in atomically; returns 400 `MISSING_WALKIN_CONTACT_INFO` if any is missing or `email` is malformed. Endpoint test (success + missing-email validation error).

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
  - **Exception-notes panel** (resolved OQ-P8-3): a small collapsible panel listing members who marked `attending=false` for this occurrence and their notes, from the GET endpoint's `exception_notes`. Hidden when empty.
  - Search bar at top with autocomplete (≥2 chars; debounced; uses `/api/staff/people/search`).
  - Pre-pop list grouped by source (Template Attendees, Paid Bookings, Recent Attendees) with checkboxes for "Attended" — click flips state via API.
  - "Add walk-in" button → dialog with **first / last / email fields, all required** (resolved OQ-P8-2 — email is required, not optional; disable submit and show a field error until a valid email is entered).
  - Yellow-flag chip next to anyone with missing skill requirements; clicking "Check in anyway" pops a reason-required confirmation dialog.
  - When a check-in returns `over_capacity_warning` (resolved OQ-P8-1), the check-in still succeeds; surface a non-blocking toast/badge ("Over capacity — checked in anyway").
- [ ] Optimistic UI updates with rollback on error.
- [ ] Spec covers all flows: regular check-in, walk-in (incl. **email-required validation**), skill-override, undo, **exception-notes panel render**, and the **over-capacity soft-warn** toast.

### 7.2 `ServerAccess` extensions
- [ ] `getCheckinList(eventSessionId)` (response now includes `exceptionNotes`), `checkInPerson(eventSessionId, personId, skillOverride?, overrideReason?)` (result includes `overCapacityWarning`), `undoCheckIn(eventSessionId, personId)`, `walkInCheckin(eventSessionId, firstName, lastName, email)` (**email required — resolved OQ-P8-2**), `searchPeople(query)`.
- [ ] Update `ServerAccess.mock.spec.ts` (incl. a walk-in case that rejects a missing/blank email, and the exception-notes + over-capacity-warning fields).

### 7.3 Types
- [ ] `checkin.types.ts`: `CheckinCandidate`, `CheckInResult` (with `overCapacityWarning`), `CheckinSessionInfo`, `ExceptionNote`, and a `CheckinListResponse` that carries `exceptionNotes`.

## 8. Admin Metadata

- [ ] New permissions used: `manage_classes` (reused; introduced earlier). No new admin tables.

## 9. Tests-Required Summary

- [ ] Table helper tests for `GetRecentCheckedInPersonsForSchedule`, `MarkCheckedIn`, `MarkNoShow`.
- [ ] `class_checkin_helper_test.cpp`:
  - Pre-pop list combines template + paid + history correctly without duplicates.
  - Check-in creates booking for membership-included class with `purchase_id=NULL`.
  - Check-in on existing paid booking sets `checked_in_us`.
  - Walk-in flow creates person + booking **with email persisted**; **rejects when email is missing/blank (`MISSING_WALKIN_CONTACT_INFO`)** (resolved OQ-P8-2).
  - Over-capacity membership check-in **succeeds with `overCapacityWarning=true`** and no error (resolved OQ-P8-1).
  - `GetExceptionNotesForOccurrence` returns the `attending=false` notes for the occurrence, excludes empty notes (resolved OQ-P8-3).
  - Skill override requires `manage_classes`; rejects without permission.
  - Undo deletes walk-in booking; resets `checked_in_us` for paid.
  - `FinalizeAttendance` marks `no_show` on paid + unchecked.
- [ ] Endpoint tests for all six endpoints (success + permission-denied + validation-error), incl. the walk-in **missing-email 400** and the GET endpoint returning `exception_notes`.
- [ ] Frontend spec for check-in page covering search, pre-pop, walk-in (incl. **email-required validation**), skill-override, undo, **exception-notes panel**, and the **over-capacity soft-warn** toast.
- [ ] Manual-testing-helper commands: `checkin <event_session_id> <person_id>`, `walkin_checkin <event_session_id> <first> <last> <email>`, `finalize_attendance`.

## 10. Cross-Layer Acceptance Criteria

Tuesday 6:55pm (5min before session start of "Vinyasa Flow at Studio A 7-8pm"):
- [ ] Staff opens `/portal/staff/class-checkin`, clicks on the session.
- [ ] The pre-pop list shows: 3 template attendees, 0 paid bookings (it's membership-included), and 5 recent-history attendees (last 4 weeks).
- [ ] Staff checks in template attendee #1 → creates a `booking` with `purchase_id=NULL`, `checked_in_us=now`, `status='attended'`, `event_sessions.booked_count++`.
- [ ] Staff types "Jor" → autocomplete returns "Jordan Smith" (recent history). Staff clicks → booking created + checked in.
- [ ] Walk-in "Maya Patel" (not in system) → staff must enter **name and email** (both required); person created + booking created + checked in with `is_walkin=true`. Submitting without an email is blocked with a field error.
- [ ] Tries to check in "Alex" who has a skill requirement they don't meet → yellow flag appears; "Check in anyway" → reason dialog → submit → booking created with override note. Reject if staff lacks `manage_classes`.
- [ ] The check-in screen shows an exception-notes panel: "Priya — out sick this week" because Priya marked `attending=false` with that note for this occurrence.
- [ ] The membership-included session is already at capacity; staff checks in one more walk-in → check-in succeeds with a non-blocking "Over capacity — checked in anyway" toast (no `SESSION_FULL` block).

The next morning at 11pm window-close:
- [ ] Hourly `finalize_class_attendance` job runs; for a separate paid workshop where 2 attendees never checked in, those bookings flip to `status='no_show'`.

## 11. Open Questions

All three resolved (Mason, 2026-06-09) and folded into the plan above (§1.1 Locked-in + the cited sections).

- **OQ-P8-1. — RESOLVED (Mason: "go with your recommendation").** Over-capacity membership-included check-in **soft-warns but allows** (`CheckInResult.overCapacityWarning`, no error); paid-offering capacity stays a real `SESSION_FULL` enforced at purchase time. Folded into §1.1, §4.3, §7.1, §9, §10.
- **OQ-P8-2. — RESOLVED (Mason: "name and email for everyone").** Walk-in person creation requires **both name AND email** — email is NOT optional; reject with `MISSING_WALKIN_CONTACT_INFO` if missing/malformed. Folded into §1.1, §4.3, §5.1, §7.1–7.3, §9, §10.
- **OQ-P8-3. — RESOLVED (Mason: "go with your recommendation").** A small exception-notes panel on the check-in screen shows notes from members who marked `attending=false` for the occurrence, via `GetExceptionNotesForOccurrence` + the GET endpoint's `exception_notes`. Folded into §1.1, §4.5, §5.1, §7.1–7.3, §9, §10.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 8.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 3 - Skill Levels]], [[Classes Phase 5 - Attendance Templates]].
- Scheduler: [[Scheduled Jobs]].
- Provider portal patterns: [[Provider Portal]] (the staff-portal patterns we extend here).
