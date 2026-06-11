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
- [x] **Timezone fix (2026-06-10, found by Mason testing at 4:30pm):** session `start/end_time_us` are facility wall-clock **encoded as UTC** (occurrence UTC-midnight + slot minutes), but `nowUs` is real UTC — the original comparison mixed bases, so a 6-7pm Pacific class read "window closed" from 2pm local onward, and `FinalizeAttendance` would no-show paid bookings at 3pm local, *before the class ran*. Fix: `FacilityTimezone()` + `ToFacilityWallClockUs()` (Postgres `AT TIME ZONE`, the weekly-digest pattern) convert real-now to the facility's wall-clock encoding inside `IsCheckinOpen`, `DoCheckIn`'s window check, and `FinalizeAttendance`'s cutoff; `FinalizePendingSessions`' sweep bounds widened by the extreme UTC offsets (−12h/+14h) with the per-session gate as the precise filter. Audit timestamps (`checked_in_us`) stay real UTC. New tests: `IsCheckinOpenConvertsRealUtcToFacilityWallClock`, `CheckInAcceptsFacilityLocalWindow`, `FinalizeAttendanceUsesFacilityLocalCutoff` (all on an `America/Los_Angeles` facility; fixture `CreateClass`/`AddFacility` grew a timezone param, default UTC keeps the 16 existing tests as identity conversions); sweep test's "too old" session moved outside the widened bound (−65h).

### 4.2 Pre-pop list ✅
- [x] `struct CheckinCandidate` — `personId/firstName/lastName/email/source ("paid_booking"|"template"|"history")/alreadyCheckedIn/bookingId/waitlistPosition`. The plan's `missingSkillLevelIds` is replaced by the unified gate decoration: `bool meetsRequirements` + `std::vector<int64_t> failedRequirementGroupIds`.
- [x] `GetCheckinList(Transaction&, eventSessionId)` — (a) non-cancelled bookings on the session (`paid_booking`); (b) `GetTemplateIdsForSlot` → people MINUS this-occurrence `attending=false` skips (`template`); (c) `GetRecentCheckedInPersonsForClass(classId, now − weeks, now)` (`history`); deduped by person, decorated with `CheckAccess`, sorted by last then first name. Tests `GetCheckinListMergesSourcesDedupesAndSkips` + `GetCheckinListFlagsFailedRequirements`.

### 4.3 Check-in action ✅
- [x] `struct CheckInRequest { classScheduleSlotId; occurrenceDateUs; personId; staffPersonId; skillOverride; overrideReason; }` / `struct CheckInResult { ok; eventSessionId; bookingId; createdNewBooking; overCapacityWarning; errorCode; }`.
- [x] `CheckIn` — window check from slot+date (`CHECKIN_NOT_OPEN`), `EnsureSessionExists`, existing-booking → `MarkCheckedIn`, else access gate (`NOT_ELIGIBLE` unless overridden), soft-warn capacity (OQ-P8-1), create `purchase_id=NULL` attended booking + increment `booked_count`. Tests: open-class create, existing→attended (no double-count), closed window, invalid slot, gated-block, gated-override-allowed, override-ignored-without-permission, over-capacity-soft-warn.
- [x] `WalkInCheckIn(Transaction&, eventSessionId, WalkInRequest, staffPersonId)` — requires non-empty name + well-formed email (`MISSING_WALKIN_CONTACT_INFO`), reuses an existing person by email or creates one, then checks in with `is_walkin=true`. (`is_walkin` lives on `bookings`, not `people` — the plan's "people.is_walkin" was a mis-statement; there is no such column.) Tests: create+check-in, missing/malformed email, reuse-by-email.
- [x] **Walk-in account fix (2026-06-11, found by Mason):** the original walk-in created the person with an **empty password hash and no email** — the person could never log in. Now a brand-new walk-in gets a real **quick account** via the new `Auth::QuickAccountHelper` (`business_logic/auth/quick_account_helper.{h,cpp}` + 4 tests): random temporary password, `must_change_password=true`, pending gift-invitation processing, and the welcome email with the temporary password — the same flow as `/api/staff/create_quick_account`, whose endpoint was refactored to delegate to the helper (it previously held all this logic inline, against the layering rules; response shape unchanged). `ClassCheckinHelper` grew a `(db, secrets, mail)` ctor (walk-in endpoint passes `GetMailHelper()`; `::Mail::` globally qualified — `Scheduling::Mail` shadows it); `WalkInCheckIn` now checks the window **before** account creation so a refused check-in never creates an account or sends mail. **Follow-up fix (runtime 500 "key not allowed"):** `People::UpdatePerson` deliberately allowlists only profile fields, so the helper now uses a new dedicated `People::SetMustChangePassword(tx, personId, bool)` (mirrors `UpdatePassword`; +2 tests in `people_test.cpp` incl. the allowlist still rejecting the flag); `update_user_password.cpp`'s flag-clear was switched from a direct endpoint-level `DbCrud::UpdateRow` to the same setter (existing `ClearFlag` endpoint test covers it). New tests: helper (`WalkInSendsWelcomeEmailAndCreatesUsableAccount`, `WalkInReuseSendsNoWelcomeEmail`, `WalkInClosedWindowCreatesNoAccountOrEmail`), endpoint (`WalkInSucceeds` now asserts the welcome email), and `quick_account_helper_test.cpp` incl. extracting the temp password from the email body and `VerifyPassword`-ing it.
- [x] `UndoCheckIn` — a **purchase-less** booking (walk-in OR membership-included; broadened from the plan's "is_walkin AND purchase_id NULL" since membership check-in is `is_walkin=false, purchase_id=NULL` yet was still created by the check-in) is deleted + `booked_count` decremented; a paid booking is reset to `confirmed` with `checked_in_us` cleared and an audit note. Tests: delete-membership, reset-paid, false-when-no-booking.

### 4.4 Finalize attendance (hourly job) ✅
- [x] `int FinalizeAttendance(Transaction&, eventSessionId, nowUs)` — returns 0 before `end + after`; else flips every `confirmed` + unchecked + PAID booking to `no_show` (membership/walk-in purchase-less rows are left alone). Idempotent. Test `FinalizeAttendanceMarksUncheckedPaidNoShow`.
- [x] `int FinalizePendingSessions(Transaction&, nowUs)` — sweeps `GetClassSessionsEndingInWindow(now − 48h, now)` and finalizes each. Test `FinalizePendingSessionsSweepsRecentlyEnded` (in-window finalized, out-of-window skipped).

### 4.5 Per-instance exception notes (resolved OQ-P8-3) ✅
- [x] `struct ExceptionNote { personId; firstName; lastName; note; }` + `GetExceptionNotesForOccurrence(slot, occurrenceDateUs)` — `GetExceptionsForSlotOccurrence` filtered to `attending=false` with a non-empty note, resolved via `template → person`. No SQL in business logic. Test `GetExceptionNotesForOccurrenceReturnsSkipNotes`.

### 4.6 KeyValueTable conversions ✅
- [x] `CheckinCandidateToKeyValueTable` (+ array), `CheckInResultToKeyValueTable` (incl. `over_capacity_warning`), `ExceptionNoteToKeyValueTable` (+ array) added to `scheduling_key_value_table.h/.cpp`. `failed_requirement_group_ids` is a comma-delimited id list. Tests in `scheduling_key_value_table_test.cpp` (7 cases: scalar fields, waitlist omitted when unset, failed-group list, array conversions, error-code surfacing).

## 5. Endpoints ✅ DONE

Files: `endpoints/staff_class_checkin.{h,cpp}` (+ `_test.cpp`, 10 cases) and
`endpoints/admin_finalize_class_attendance.{h,cpp}` (+ `_test.cpp`, 3 cases).

**Reconciliations vs the plan:**
- **People search reuses the existing endpoint.** `GET /api/staff/search_people?q=` already exists (`staff_search_people.cpp`, `staff_access`-gated) — reused, not recreated (the plan allowed "reuse if existing"). The frontend points at that path, not the plan's `POST /api/staff/people/search`.
- **Permission is `staff_access`.** The Phase-1.7/3.6 security review mandates every `/api/staff/*` route require `staff_access` (`RequirePermission`, which there's no "any-of" variant of), so the GET uses `staff_access` rather than the plan's loose "staff OR manage_classes". The override-on-check-in path still separately checks `manage_class_schedule` inside the helper.
- **Admin finalize uses `manage_class_schedule`.** The codebase has no generic "admin" gate for jobs; sibling job endpoints (e.g. instructor-digest) gate on `manage_class_schedule`, which the scheduler service account holds. Used here too.
- **Lazy-derivation bridge added.** Action endpoints are keyed by a persisted `eventSessionId`, but a purely-derived occurrence has none until first recording. Added `POST /api/staff/checkin/ensure {class_schedule_slot_id, occurrence_date_us}` → `{event_session_id}` (idempotent `EnsureSessionExists`) so the frontend can obtain an id for a derived occurrence before GET/check-in. Check-in via id uses the new `ClassCheckinHelper::CheckInByEventSession` wrapper (reads the session's slot+date, delegates to `CheckIn`).

### 5.1 Staff endpoints ✅
- [x] `GET /api/staff/checkin/<eventSessionId>` → `{ window_open, session_info, candidates, exception_notes }`. `session_info` from `EventSessionHelper::GetEventSession`; `candidates`/`exception_notes` from `ClassCheckinHelper`. `staff_access`. Tests: returns candidates+notes+window, 401 anon, 403 non-staff.
- [x] `POST /api/staff/checkin/<eventSessionId>/person/<personId>` body `{ skill_override?, override_reason? }` → `CheckInResult` JSON (200; `ok`/`error_code`/`over_capacity_warning` carried in the body). staffPersonId = the logged-in session. Tests: creates booking, 403 non-staff.
- [x] `DELETE /api/staff/checkin/<eventSessionId>/person/<personId>` → `{ ok }`. Test: undo after check-in.
- [x] People search — reused `GET /api/staff/search_people` (see above). No new endpoint.
- [x] `POST /api/staff/checkin/<eventSessionId>/walkin` body `{ first_name, last_name, email }` — all required; returns **400** when contact info missing/malformed, else `CheckInResult`. Tests: success + missing-email 400.
- [x] `POST /api/staff/checkin/ensure` — lazy-derivation bridge (see above). Tests: returns id + 400 on missing body.

### 5.2 Admin/scheduler endpoint ✅
- [x] `POST /api/admin/finalize_class_attendance` — `FinalizePendingSessions(now)` over the last 48h. Idempotent. `manage_class_schedule`. Returns `{ no_show_count }`. Tests: 401 anon, 403 without permission, finalizes + idempotent second run.

### 5.3 Routing ✅
- [x] All six functions registered in `web_app.cpp` (includes + reference vars) and `endpoints/CMakeLists.txt` (sources + tests).

### 5.4 Async-write safety ✅
- [x] These endpoints do NOT call `ThreadPool::Queue` (check-in writes are synchronous: `MarkCheckedIn`/`AddBooking`/`UpdateBooking`), so the queue-race doesn't arise. The endpoint tests still apply the `ThreadPool::Shutdown()` discipline (flush after each `handle_full`) to drain any session-touch writes before the next request / DB read, per `feedback_sync_sql_before_threadpool_queue.md`.

## 6. Scheduled job integration ✅ DONE

- [x] Add hourly job in `knottyyoga_helper`: `POST /api/admin/finalize_class_attendance`. Idempotent.
  - `scheduler_config.h` — added `JobIntervals::finalizeAttendanceSeconds = 3600` (hourly).
  - `scheduled_job.cpp` `BuildStandardJobs` — registers `finalize_class_attendance` → `POST /api/admin/finalize_class_attendance` (via `AppendIfEnabled`, so a 0 interval disables it).
  - `main.cpp` — `--finalize_attendance_interval` ABSL flag (default 3600), wired into `BuildConfigFromFlags` and the startup `LogConfigSummary` line.
  - Validation: follows the recent-sibling convention (instructor/weekly/series jobs aren't in `ValidateSchedulerConfig` either); `BuildStandardJobs` already defends against `<= 0`.
  - Tests: `scheduled_job_test.cpp` — job count 15→16, `EachJobHasPostMethodAndExpectedPath` spot-checks the new path, `ZeroIntervalDisablesIndividualJob` 13→14, added `FinalizeAttendanceCanBeDisabled` + `IntervalFromConfigPropagatedForFinalizeAttendance`, all-zeros initializer extended to 16. `scheduler_config_test.cpp` + `scheduler_test.cpp` — all-zeros initializers extended to 16, `InitializeRegistersAllEnabledJobs` 15→16.

## 7. Frontend ✅ DONE

**Reconciliations vs the plan:**
- **Path is `pages/staff/`, not `pages/portal/staff/`** — the staff portal lives at `ui/src/app/pages/staff/` (route prefix `/staff`); the new page is `pages/staff/class-checkin/` at `/staff/class-checkin`, registered in `staff.routes.ts` + a "Class Check-In" dashboard card (`fact_check` icon). The pre-existing `/staff/check-in` page (paid service/event booking check-in from the Provider Portal era) is untouched — different feature.
- **Session list reuses `getTodayClasses()`** (Phase 5 §4.5) — it already returns today's derived+persisted class occurrences with `event_session_id` (0 when purely derived), names, rooms, instructors. No new listing endpoint needed.
- **Search reuses `staffSearchPeople()`** (`GET /api/staff/search_people`), per the §5 reconciliation — not the plan's `/api/staff/people/search`.
- **Field names are snake_case** matching the wire format (codebase convention): `exception_notes` / `over_capacity_warning`, not the plan's camelCase prose names. `ServerAccessNetwork` normalizes KVT string-booleans → real booleans and splits the comma-delimited `failed_requirement_group_ids` → `number[]`.
- **Inline panels, not MatDialog** — the walk-in form and override confirmation are inline bordered panels (same pattern as the existing staff check-in page's inline walk-in flow); spec-friendlier and consistent.
- **Timezone fix #2 (2026-06-10, 5:30pm repro):** the session list called `getTodayClasses()` with no `tz`, and the server defaults its "today" window to UTC — so the list emptied at exactly 5pm Pacific (UTC midnight). Now passes the browser IANA timezone (`Intl.DateTimeFormat().resolvedOptions().timeZone`), same as the homepage today-feed; spec pins the argument.
- **Timezone fix (2026-06-10):** class times (`TodayClassEntry.start_us/end_us` and `EnsureSessionExists`-persisted `event_sessions.start_time_us`) are **studio wall-clock encoded as UTC** (occurrence UTC-midnight + `start_time_minutes`, `GetDerivedSessionsForRange`), NOT real instants — they must be formatted with `timeZone: 'UTC'` or a Pacific browser shows a 6pm class as 11am. Fixed in `class-checkin.component.ts` (list + header times) **and** the pre-existing `account/today-classes` page which had the same latent bug; pinned by timezone-proof regression tests in both specs (assert 6:00 PM on a Pacific test host).

### 7.1 Staff check-in page ✅
- [x] `ui/src/app/pages/staff/class-checkin/class-checkin.component.{ts,html,scss,spec.ts}` (standalone, SharedModule).
- [x] Lists today's sessions (sorted by `start_us`); click → check-in screen. A derived occurrence (`event_session_id=0`) is resolved via `ensureCheckinSession(slot, occurrenceDate)` first (§5.1 bridge).
- [x] Check-in screen:
  - Session header: class name, time range (facility timezone), room, facility, **attended count / capacity** (red over capacity), **window open/closed badge**.
  - **Exception-notes panel** (OQ-P8-3): collapsible amber panel from `exception_notes`; hidden when empty.
  - Search bar with autocomplete (≥2 chars, 300ms debounce, `staffSearchPeople`); "Check in" per result (`checkInPerson`, then list refresh).
  - Pre-pop list grouped Template Attendees / Paid Bookings / Recent Attendees with checkbox toggles; toggles disabled when the window is closed.
  - "Add walk-in" → inline panel, **first/last/email all required** (OQ-P8-2): submit disabled + field error until valid email; server 400 surfaced inline; autofocus on first field.
  - Yellow "Requirements not met" chip (`meets_requirements=false`); clicking the toggle opens the **reason-required** override confirmation → `checkInPerson(…, skillOverride=true, reason)`.
  - `over_capacity_warning` (OQ-P8-1) → non-blocking warning toast "Over capacity — checked in anyway." (check-in retained).
- [x] Optimistic toggle updates with rollback on `ok=false` AND on transport error (both directions: check-in + undo).
- [x] Spec: 24 cases — list sort/empty, persisted-vs-derived open (ensure bridge), header render, grouping, notes panel render/hide/collapse, yellow flag, check-in + rollback (`ok=false` and throw), undo + rollback, over-capacity toast (check-in AND walk-in), override open/disabled-until-reason/confirm args, walk-in email validation (invalid + missing) / submit+refresh / 400 surface, window-closed badge + disabled toggles + no server call, search debounce + <2-char skip + search-result check-in, back-to-list reload.

### 7.2 `ServerAccess` extensions ✅
- [x] Interface + `ServerAccessNetwork` + `ServerAccessMock` + `ServerAccessProxy`: `ensureCheckinSession(slotId, occurrenceDateUs)`, `getCheckinList(eventSessionId)` (carries `exception_notes`), `checkInPerson(eventSessionId, personId, skillOverride?, overrideReason?)` (result carries `over_capacity_warning`), `undoCheckIn(eventSessionId, personId)`, `walkInCheckin(eventSessionId, firstName, lastName, email)` (**email required**). People search = existing `staffSearchPeople`.
- [x] Mock state mirrors the server: lazy ensure map (slot+date → session 501), three-source candidate list incl. a gated person, exception note, capacity-3 session, created-by-check-in bookings deleted on undo vs paid reset, walk-in reuse-by-email.
- [x] `ServerAccess.mock.spec.ts`: 24 new cases — ensure (resolve/idempotent/400/401), list (three sources + notes + gated flags + 404), check-in (create/reuse/gated-reject/blank-reason-reject/override-allow/**over-capacity soft-warn**/window-closed/idempotent/401), undo (delete-created/reset-paid/never-checked-in), walk-in (create+check-in/**missing-email 400**/malformed-email 400/blank-name 400/reuse-by-email/401).

### 7.3 Types ✅
- [x] `shared/types/checkin.types.ts`: `CheckinCandidate` (+`CheckinCandidateSource`), `CheckInResult` (with `over_capacity_warning`), `CheckinSessionInfo`, `ExceptionNote`, `CheckinListResponse` (carries `exception_notes`), `WalkInCheckinRequest`, `CheckinErrorCodes` const map. Re-exported from `shared/types/ServerAccess.ts`.
- [x] Updated `staff-dashboard.component.spec.ts` for the new card (index + count assertions).

## 8. Admin Metadata ✅ DONE

- [x] No new permissions and no new admin tables. Verified: the endpoints gate on the pre-existing `staff_access` (staff routes) and `manage_class_schedule` (finalize job + override path) — both already seeded (`db_schema/permissions.h`); the §5 reconciliation already corrected the plan's original `manage_classes` mention. Nothing to register in `create_database.cpp`.

## 9. Recent-Attendee Quick Check-in + Test-Helper Attendance Seeding

**Request (Mason, 2026-06-11):** Show people who have taken a given class over the last month as automatically-added candidates (when they aren't already there via template) so checking them in is one click — many people won't use the schedule template. Also add test-helper commands to enumerate classes with an active schedule and their slots, search for a person (autocomplete), and mark that person as having attended that class slot the previous week, so the feature can be validated in the UI.

**Status check — the core behavior already shipped in Phase 8.** The pre-pop list's `history` source (§3.1 `GetRecentCheckedInPersonsForClass` + §4.2 `GetCheckinList` step (c)) pulls everyone with a `checked_in_us` booking on this class in the last `class_checkin_history_weeks` (secret, default **4 weeks ≈ "last month"**), dedupes against template/paid entries, and the frontend (§7.1) renders them as the **Recent Attendees** group with one-click check-in. It has been invisible in dev only because the database has no historical attendance — which is exactly what the §9.5 seeding commands fix. So §9 = (a) verification of that path end-to-end once seeded, (b) one small polish item (surface *when* they last attended), and (c) the new test-helper tooling.

### 9.1 Database Schema ✅ DONE
- [x] No new tables or columns. Verified:
  - `bookings.checked_in_us` + `event_sessions.class_id` (denormalized; only recorded/attended occurrences have it) is exactly the "took this class" join.
  - `class_checkin_history_weeks` secret already seeded (default 4 weeks ≈ "last month"); operator-configurable, default left as-is.

### 9.2 Table Helpers ✅ DONE
- [x] Verified — `TableHelpers::Bookings::GetRecentCheckedInPersonsForClass` already returns `(person_id, last_checked_in_us)` (per-person MAX, window-filtered, NULL-check-in excluded — tested in `bookings_test.cpp`). No changes needed.

### 9.3 Business Logic ✅ DONE
- [x] `CheckinCandidate.lastCheckedInUs` (`std::optional<int64_t>`) — set only for `history`-source rows in `GetCheckinList` (header comment flags it as a REAL UTC instant, unlike session times). Absent for `template`/`paid_booking` rows.
- [x] `scheduling_key_value_table.cpp`: emits `last_checked_in_us` when set, omitted when unset (the `waitlist_position` pattern). Tests: `CheckinCandidateSurfacesLastCheckedIn` (new) + omission assertions added to both existing candidate cases.
- [x] `class_checkin_helper_test.cpp` — `GetCheckinListMergesSourcesDedupesAndSkips` extended with five new behaviors: (a) history row carries the exact `lastCheckedInUs`; (a') with TWO attendances, it carries the LATEST one (MAX, not first-found); (b) template+history person resolves as `template`, exactly once, with NO history decoration; (c) attendee whose last check-in is 35 days old is excluded entirely; (d) paid/template rows carry no `lastCheckedInUs`.

### 9.4 Endpoints ✅ DONE
- [x] No new endpoints. `GetReturnsCandidatesNotesAndWindow` extended: seeds a real attendance on last week's occurrence of the same slot (`EnsureSessionExists` + attended booking), then asserts the JSON history candidate has `source="history"` + the exact `last_checked_in_us`, and the template candidate has NO `last_checked_in_us` key.

### 9.5 Test-Helper Changes ✅ DONE
New file `test_helper/commands/checkin_commands.{h,cpp}` (category "Attendance"), registered in `command_runner.cpp` `RegisterAllCommands` + `test_helper/CMakeLists.txt`.
- [x] `list_active_class_slots` (`lacs`) — sweeps active classes, resolves each through `ClassScheduleHelper::GetActiveScheduleView(now)` (same activity logic as the admin preview), and prints every slot of every class active **today**: slot id, class, day, start–end, room/facility names. Ends with a copy-paste hint for `sca`.
- [x] `find_person` (`fp`) — `--q=<substring>`: parameterized case-insensitive ILIKE over first/last/email, first 20 matches with a narrow-the-search note.
- [x] `seed_class_attendance` (`sca`) — `--slot_id=` + (`--person_id=` or `--email=`) + `--weeks_ago=1` (1–52): resolves the slot's most recent occurrence on-or-before today minus N weeks, `EnsureSessionExists` (idempotent), then a purchase-less **attended** booking with backdated `checked_in_us` + `IncrementBookedCount`. Idempotent — an existing non-cancelled booking for (session, person) reports and skips. Notes column marks the row "Seeded by knottyyoga_test_helper". **Implementation deviations from the sketch:** the booking is created directly with `status=attended`/`checked_in_us` set (no `MarkCheckedIn` call — that would append a bogus "Checked in by person 0" note), and `checked_in_us` is converted **wall-clock → real UTC instant through the facility timezone** (it's the one real-instant field; storing the wall-clock encoding would have been our fourth timezone bug).
- [x] Replxx `--email` value-completion: skipped per the plan's "only if cheap" — `find_person` covers the workflow.
- [x] §10's outstanding "manual-testing-helper commands" item: `sca` covers the seeding need; the `checkin`/`walkin_checkin`/`finalize_attendance` wrappers are dropped (the UI exercises those flows interactively).
- Testing note: test-helper commands follow the established convention of no gtest coverage (none of the ~25 existing commands have tests; the tool is itself a manual-QA aid). Every load-bearing operation `sca` performs is already test-covered at its home layer (`EnsureSessionExists`, `AddBooking`, `IncrementBookedCount`, and the §9.3 history-window behavior they feed).

### 9.6 Frontend
- [ ] Verify end-to-end with seeded data: seed one person via `sca`, open the next occurrence of that slot in `/staff/class-checkin`, confirm they appear under **Recent Attendees** and one-click check-in works (this is the §9 acceptance test).
- [ ] `checkin.types.ts`: `last_checked_in_us?: number` on `CheckinCandidate`; `ServerAccessNetwork.normalizeCheckinCandidate` coerces it with `Number()` when present; mock seeds it on its `history` candidates (+ mock spec assertion).
- [ ] `class-checkin.component`: show a dim "Last attended <Mon DD>" subtitle on history-group rows. **Timezone care:** `checked_in_us` is a REAL UTC instant (unlike every other timestamp on this page, which is wall-clock-encoded and formatted with `timeZone: 'UTC'`) — format this one in browser-local time, and say so in a comment or the next reader will "fix" it.
- [ ] Component spec: history row renders the last-attended label; rows without the field render no label.

## 10. Tests-Required Summary

- [x] Table helper tests for `GetRecentCheckedInPersonsForSchedule`, `MarkCheckedIn`, `MarkNoShow` (done in §3).
- [x] `class_checkin_helper_test.cpp` (done in §4):
  - Pre-pop list combines template + paid + history correctly without duplicates.
  - Check-in creates booking for membership-included class with `purchase_id=NULL`.
  - Check-in on existing paid booking sets `checked_in_us`.
  - Walk-in flow creates person + booking **with email persisted**; **rejects when email is missing/blank (`MISSING_WALKIN_CONTACT_INFO`)** (resolved OQ-P8-2).
  - Over-capacity membership check-in **succeeds with `overCapacityWarning=true`** and no error (resolved OQ-P8-1).
  - `GetExceptionNotesForOccurrence` returns the `attending=false` notes for the occurrence, excludes empty notes (resolved OQ-P8-3).
  - Skill override requires `manage_classes`; rejects without permission.
  - Undo deletes walk-in booking; resets `checked_in_us` for paid.
  - `FinalizeAttendance` marks `no_show` on paid + unchecked.
- [x] Endpoint tests for all six endpoints (success + permission-denied + validation-error), incl. the walk-in **missing-email 400** and the GET endpoint returning `exception_notes` (done in §5).
- [x] Frontend spec for check-in page covering search, pre-pop, walk-in (incl. **email-required validation**), skill-override, undo, **exception-notes panel**, and the **over-capacity soft-warn** toast (done in §7 — `class-checkin.component.spec.ts`, 24 cases; plus 24 mock cases in `ServerAccess.mock.spec.ts` and the dashboard spec update).
- [x] Manual-testing-helper commands — **resolved by §9.5** (`seed_class_attendance` / `list_active_class_slots` / `find_person`, shipped 2026-06-11). The originally-sketched `checkin`/`walkin_checkin`/`finalize_attendance` wrappers are intentionally dropped: the staff UI exercises those flows interactively, and seeding history was the actual gap.

## 11. Cross-Layer Acceptance Criteria

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

## 12. Open Questions

All three resolved (Mason, 2026-06-09) and folded into the plan above (§1.1 Locked-in + the cited sections).

- **OQ-P8-1. — RESOLVED (Mason: "go with your recommendation").** Over-capacity membership-included check-in **soft-warns but allows** (`CheckInResult.overCapacityWarning`, no error); paid-offering capacity stays a real `SESSION_FULL` enforced at purchase time. Folded into §1.1, §4.3, §7.1, §9, §10.
- **OQ-P8-2. — RESOLVED (Mason: "name and email for everyone").** Walk-in person creation requires **both name AND email** — email is NOT optional; reject with `MISSING_WALKIN_CONTACT_INFO` if missing/malformed. Folded into §1.1, §4.3, §5.1, §7.1–7.3, §9, §10.
- **OQ-P8-3. — RESOLVED (Mason: "go with your recommendation").** A small exception-notes panel on the check-in screen shows notes from members who marked `attending=false` for the occurrence, via `GetExceptionNotesForOccurrence` + the GET endpoint's `exception_notes`. Folded into §1.1, §4.5, §5.1, §7.1–7.3, §9, §10.

## 13. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 8.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 3 - Skill Levels]], [[Classes Phase 5 - Attendance Templates]].
- Scheduler: [[Scheduled Jobs]].
- Provider portal patterns: [[Provider Portal]] (the staff-portal patterns we extend here).
