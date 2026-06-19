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

Classes Phase 11 - Signup Windows and Reminders

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

**Should-have.** Per-permission advance-booking days at the product level (uses existing `product_booking_windows`). Users see future class series / workshops / intro / guest-pass-eligible sessions outside their booking window with a "Sign-ups open on {date}" hint. Users can click "Remind me when sign-ups open" → row added to `signup_open_reminders` → daily cron picks up and emails the user on the open date.

**Important scope note:** signup windows mostly apply to **paid** offerings (workshops, series, intro). Recurring class attendance is membership-included with no advance booking (Phase 5 + Phase 8), so windows are irrelevant there.

**Prerequisites:**
- Phase 1 (three-level model; products live on `class_instances`; `ClassScheduleHelper::GetDerivedSessionsForRange`).
- Phase 2 (visibility / pricing surface).
- Phase 7 (series flow uses booking windows).
- Existing `product_booking_windows` infra ([[Event Polish- Scheduling Should Have Items]]).
- Scheduled jobs daemon ([[Scheduled Jobs]]).

### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]]
Small. "Future sessions outside your booking window" are **derived** (most have no persisted `event_sessions` row) — enumerate them via `ClassScheduleHelper::GetDerivedSessionsForRange`. The per-permission advance window comes from the **active `class_instances` row's product** (`product_booking_windows`), not from a flat schedule. A `signup_open_reminders` row keys off the occurrence identity (`class_schedule_slot_id`, `occurrence_date_us`) rather than an `event_session_id`, since the occurrence usually isn't persisted until someone books it (which then ensures the row).

**Outcome:**
- Catalog / calendar shows "Sign-ups open on Jul 15" for sessions outside the user's window.
- "Remind me" button creates a reminder row.
- Daily cron emails users on the open date.
- On booking, pending reminders for that session auto-cancel.

## Layering & Conventions

Lowest layer first:

1. `db_schema/` — `signup_open_reminders` table.
2. `sql_util/table_helpers/` — new helper.
3. `business_logic/scheduling/` — `SignupReminderHelper`.
4. `endpoints/` — three new endpoints.
5. Scheduled jobs.
6. Angular: catalog / calendar future-session chip + Remind-me action.
7. Tests.

## 1. Pre-Coding Design Decisions

### 1.1 Locked-in
- [x] Per-product overrides via existing `product_booking_windows` table (parent OQ-13).
- [x] "Best window" recomputed live at booking time (parent OQ-14).
- [x] Reminder de-dup: when a user successfully books a session, mark pending reminders for that session cancelled (parent AW-5).
- [x] **Series reminder = one email, multi-VEVENT iCal (resolved OQ-P11-1).** A series run gets a single "sign-ups open" email (NOT one per instance), but it carries an `.ics` attachment with **one VEVENT per upcoming instance** of the run (reusing the Phase 7 series-confirmation / Phase 4 iCal pattern). A single workshop/intro occurrence gets a one-VEVENT `.ics`.
- [x] **Scope: class offerings only (resolved OQ-P11-2).** Reminders cover class series / workshops / intro / guest-pass-eligible sessions. Non-class events and bookable services that have advance windows are **out of scope for Phase 11** — extend opportunistically later.

## 2. Database Schema

### 2.1 `signup_open_reminders` table ✅
- [x] `db_schema/signup_open_reminders.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL REFERENCES people(id)`
  - `class_schedule_slot_id BIGINT NOT NULL REFERENCES class_schedule_slots(id)` — the occurrence's slot (the occurrence usually has no persisted `event_sessions` row yet)
  - `occurrence_date_us BIGINT NOT NULL` — the specific day
  - `notify_at_us BIGINT NOT NULL` — moment when user's window opens (occurrence start − their best `advance_days` × 86400_000_000)
  - `notified_us BIGINT NULL`
  - `cancelled_us BIGINT NULL`
  - `created_us BIGINT NOT NULL`
- [x] Partial unique index `signup_open_reminders_pending_unique_idx` `UNIQUE (person_id, class_schedule_slot_id, occurrence_date_us) WHERE notified_us IS NULL AND cancelled_us IS NULL` — one pending reminder at a time. (Raw `CREATE UNIQUE INDEX` in `CreateSignupOpenRemindersIndexes` — partial, so it can't be a DSL constraint.)
- [x] Index `signup_open_reminders_notify_at_idx` on `notify_at_us` for the daily cron scan.

**Implementation note (2026-06-16):** the table uses only existing schema-DSL features (FK refs, simple/nullable/default-now columns), so no DSL extension was needed. Both indexes live in `CreateSignupOpenRemindersIndexes(Transaction&)` (the established `Create*Indexes` pattern — `SetupAllTables` doesn't run these, so tests that exercise the partial unique index call it in-transaction first). Test coverage in `signup_open_reminders_test.cpp` (10 cases): valid-row round-trip + defaults; person/slot FK rejection; NOT NULL rejection (occurrence_date_us, notify_at_us); both indexes created + idempotent; partial-unique blocks a duplicate pending while allowing a distinct occurrence; a notified or cancelled row frees a new pending; distinct people share an occurrence.

### 2.2 Wire into DB init ✅
- [x] `make_database_info.cpp` (`MakeSignupOpenRemindersTable`, after iCal feed tokens) + `create_database.cpp` (`CreateTable` + `CreateSignupOpenRemindersIndexes`, FK-ordered after `class_schedule_slots`). Sources added to `db_schema/CMakeLists.txt` (core + test). Admin-metadata registration is deferred to §8 (inspection only).

## 3. Table Helpers

### 3.1 `TableHelpers::SignupOpenReminders` ✅
- [x] `AddReminder(Transaction&, personId, classScheduleSlotId, occurrenceDateUs, notifyAtUs)` → reminderId — idempotent on the partial unique index. **Signature uses explicit params (house style — matches `ShiftChangeRequests::AddRequest`) rather than the plan's `KeyValueTable&`.** Uses `INSERT ... ON CONFLICT (person_id, class_schedule_slot_id, occurrence_date_us) WHERE notified_us IS NULL AND cancelled_us IS NULL DO NOTHING RETURNING id`; on conflict (a pending reminder already exists) it returns that existing row's id via a fallback `SELECT`.
- [x] `CancelReminder(Transaction&, personId, classScheduleSlotId, occurrenceDateUs)` — sets `cancelled_us = now_us()` for the pending row; no-op when none. **Keyed by (person, slot, occurrence), NOT the plan's stale `eventSessionId`** — the §2 schema + redesign note key reminders by the occurrence identity (the occurrence usually has no persisted `event_sessions` row). §4.4 booking-dedupe must call it with (slot, occurrence) too.
- [x] `GetPendingReadyToSend(Transaction&, nowUs)` → pending reminders where `notify_at_us <= nowUs` AND `notified_us IS NULL` AND `cancelled_us IS NULL`, ordered `notify_at_us ASC, id ASC`.
- [x] `MarkNotified(Transaction&, id, nowUs)` — `DbCrud::UpdateRow` sets `notified_us = nowUs`.
- [x] Added `GetReminder(Transaction&, id)` (single-row read, mirrors `ShiftChangeRequests::GetRequest`) for §4/§5 + tests.
- [x] Tests — `sql_util/table_helpers/signup_open_reminders_test.cpp` (11 cases): AddReminder creates the pending row + round-trips; idempotent returns the existing pending id with no duplicate; a notified OR a cancelled row frees a fresh AddReminder; CancelReminder retires the pending row (cancelled_us set, notified_us still NULL) and drops it from the work list; CancelReminder is a harmless no-op when nothing pending and leaves other people's reminders alone; GetPendingReadyToSend filters by `notify_at_us <= now` and orders ASC, and excludes notified/cancelled rows; MarkNotified stamps notified_us and retires the row. Each test calls `CreateSignupOpenRemindersIndexes(tx)` first (the partial index AddReminder's `ON CONFLICT` needs isn't created by `SetupAllTables`).

## 4. Business Logic — `SignupReminderHelper` ✅

Files: `business_logic/scheduling/signup_reminder_helper.h/.cpp/_test.cpp` + the email body in `signup_reminder_mail.h/.cpp/_test.cpp`. The helper takes only a `DatabaseHelper`; mail is a per-call `::Mail::MailHelper*` on `SendPendingReminders` (so it's constructible without mail, like `InstructorSubstitutionHelper`). Wired into the scheduling `CMakeLists.txt`.

### 4.1 Best-window computation ✅
- [x] `int64_t ResolveBestAdvanceDaysForPerson(Transaction&, productId, personId, asOfUs)` — **delegates to the existing `TableHelpers::ProductBookingWindows::ResolveAdvanceDaysForUser`**, whose SQL is already exactly the planned semantics: `COALESCE(MAX(advance_days), 0)` over `WHERE permission_id IS NULL OR permission_id IN (user's permissions)`. `asOfUs` is accepted for the documented contract but unused (permissions are evaluated as of now). No new SQL needed.

### 4.2 Request-reminder flow ✅
- [x] `struct RequestReminderResult { bool ok; int64_t reminderId; int64_t notifyAtUs; std::string errorCode; }`.
- [x] `RequestReminder(Transaction&, personId, classScheduleSlotId, occurrenceDateUs)` — resolves the occurrence's product by walking slot → `class_schedules` (impl) → `class_instances` (the slot's owning instance carries the product); `occurrenceStartUs = occurrenceDateUs + slot.start_time_minutes`; `notifyAtUs = occurrenceStartUs - advanceDays·day`. Returns `INVALID_OCCURRENCE` when the slot doesn't resolve to a product, `WINDOW_ALREADY_OPEN` when **`advanceDays <= 0`** (no window configured → nothing to wait for) **or** `notifyAtUs <= now`; else inserts via the §3 helper (idempotent on the partial index).
- [x] Tests — see §4 test summary below.

### 4.3 Sending pending reminders ✅
- [x] `int SendPendingReminders(Transaction&, ::Mail::MailHelper*, int64_t nowUs)` — for each `GetPendingReadyToSend(nowUs)` row: resolve class/product, resolve the person's email/name, build the iCal + email, queue it, then `MarkNotified`. Returns the count queued. Reminders that can't send (slot gone / no email on file) are retired without an email so the cron stays idempotent. No-op (0) when `mailHelper` is null.
  - **Series run** (`classes.kind == 'series'`): ONE email (OQ-P11-1) carrying a multi-VEVENT `.ics` — one VEVENT per upcoming instance derived from the instance's validity window via `ClassScheduleHelper::GetDerivedSessionsForRange`.
  - **Single workshop / intro occurrence**: ONE email with a one-VEVENT `.ics`.
  - iCal built with `ICalGenerator::GenerateICalendar(vector<ICalEvent>)`; **`floatingLocal=true`** since occurrence times are wall-clock-encoded-as-UTC (not real instants); per-occurrence UID via `BuildTemplateOccurrenceUid`. Body via `signup_reminder_mail` (FormatString + `NormalizeCrLf`); attachment `signup_reminder.ics` (contentType "calendar"); sender from `Secrets::kMailSender*`.
- [x] Tests with `TestMailHelper` — see §4 test summary.

### 4.4 Booking-side dedupe ✅
- [x] `BookingHelper::BookEvent` — after `AddBooking`, when the session row carries `class_schedule_slot_id` + `occurrence_date_us` (a class occurrence), calls `TableHelpers::SignupOpenReminders::CancelReminder(personId, slotId, occurrenceDateUs)`. **Keyed by (slot, occurrence), not the plan's stale `eventSessionId`.** No-op for classic events and when no reminder is pending. Fires for waitlist joins too (sign-ups are open either way).
- [x] Test — `booking_helper_test.cpp::BookEventCancelsPendingSignupReminder` (attaches a slot+occurrence to the priced event session so the dedupe path runs through the real `BookEvent`; asserts the pending reminder is cancelled) + the helper-level `CancelledReminderIsNotSent` (a cancelled reminder is never emailed).

### 4.5 KeyValueTable conversions ✅
- [x] `SignupReminderInfo` struct (helper header) + `SignupReminderInfoToKeyValueTable(+Array)` in `scheduling_key_value_table.h/.cpp`, with 2 KVT test cases. The producer (`GetPendingRemindersForPerson`) is deferred to §5/§7 where the listing endpoint is wired.

### §4 test summary
- `signup_reminder_helper_test.cpp` (11 cases): ResolveBestAdvanceDays delegation (0 with no window, base window applies); RequestReminder correct notifyAt; rejects with no window and with an already-open window; idempotent; INVALID_OCCURRENCE; **SendPendingReminders workshop → 1 email / 1 VEVENT**; **series → 1 email / N VEVENTs (N computed via the same derivation, ASSERT ≥ 2)**; marks-notified-and-doesn't-resend; skips reminders whose window hasn't opened; null-mail no-op; cancelled-not-sent (dedupe at the data layer).
- `signup_reminder_mail_test.cpp` (3 cases): subject names the class; single vs series body wording; CRLF endings.
- `scheduling_key_value_table_test.cpp` (2 new cases) + `booking_helper_test.cpp` (1 integration case).

## 5. Endpoints ✅

All three registered in `web_app.cpp` + `endpoints/CMakeLists.txt`. The DELETE/POST `/me` endpoints require only a logged-in session; the admin send endpoint gates on `manage_class_schedule`.

- [x] `POST /api/me/signup_reminder` (`post_signup_reminder.h/.cpp`) body `{ class_schedule_slot_id, occurrence_date_us }` → `SignupReminderHelper::RequestReminder`. **Both "created" and "already open" are 200** with `{ reminder_id, notify_at_us, window_already_open }` (reminder_id 0 + window_already_open true for the already-open case, so the SPA shows a toast rather than treating it as an error); `INVALID_OCCURRENCE` → 404; missing fields → 400. Test (`post_signup_reminder_test.cpp`, 6 cases): 401 anon; 400 missing fields; **creates with a future window**; **window-already-open returns the flag (not an error)**; **duplicate returns the same reminder_id (idempotent)**; 404 invalid occurrence.
- [x] `DELETE /api/me/signup_reminder?class_schedule_slot_id=&occurrence_date_us=` (`delete_signup_reminder.h/.cpp`) → `SignupReminderHelper::CancelReminder`. **Keyed by (slot, occurrence) via query params, NOT the plan's stale `<eventSessionId>` path** — query params keep the large `occurrence_date_us` an int64 (Crow `<int>` is 32-bit). Idempotent 200 `{ ok: true }`. Test (4 cases): 401 anon; 400 missing param; cancels a pending reminder (cancelled_us stamped); no-op 200 when nothing pending.
- [x] `POST /api/admin/send_signup_open_reminders` (`admin_send_signup_reminders.h/.cpp`) — cron-callable, idempotent; `RequirePermission(manage_class_schedule)`; computes `now_us()`, calls `SendPendingReminders(tx, GetMailHelper().get(), now)`, returns `{ sent_count }`. **Gated on `manage_class_schedule`** (the plan's `admin`; consistent with the other Phase 10/11 class-admin endpoints — §6's scheduler account must hold it). Test (2 cases): 403 without the permission; **sends one pending reminder (1 mail queued) and a second run sends 0** (idempotent).

**Shared-fixture change:** added a `productId` field to `template_test_fixture.h`'s `SlotFixture` (set by `CreateRecurringSlot`) so the booking-window setup can attach a `product_booking_windows` row — additive, existing callers unaffected. Each endpoint test that creates reminders calls `CreateSignupOpenRemindersIndexes(tx)` first (the partial index isn't created by `SetupAllTables`).

## 6. Scheduled job integration ✅

- [x] Added the hourly `send_signup_open_reminders` job to `knottyyoga_helper`: `scheduler_config.h` (`signupRemindersSeconds = 3600`), `scheduled_job.cpp` (`AppendIfEnabled` → `POST /api/admin/send_signup_open_reminders`), and `main.cpp` (`--signup_reminders_interval` flag + config wiring + the startup `LogConfigSummary` line). Idempotent and self-gating (the endpoint only emails reminders whose window has opened, via `GetPendingReadyToSend`'s `notify_at_us` filter).
- [x] **Scheduler permission fix (required for the job to authorize).** Traced that the scheduler service account holds only `manage_subscriptions` (via `EnsureSchedulerServiceAccount`), and the permission check (`Session::ActiveUserHasPermission` → direct `role_permissions` lookup) has **no implication expansion or admin bypass**. But my endpoint — and the existing class crons `finalize_class_attendance`, `send_instructor_exception_digests`, `run_series_min_attendees_check` — gate on `manage_class_schedule`, which the scheduler did **not** hold. So those crons were latently mis-authorized (they'd 403). Fixed at the single source of truth: `EnsureSchedulerServiceAccount` now also grants the scheduler role `manage_class_schedule` (best-effort — granted only when the permission row exists, which it always does in a real DB since `PopulatePermissions` seeds it before this runs; this keeps the auth unit tests that seed only `manage_subscriptions` unaffected). This makes the new cron work AND repairs the three existing class crons.
- [x] Tests:
  - `scheduled_job_test.cpp`: total job count 16 → **17**; `send_signup_open_reminders` present with the right path/POST; disabled at interval 0; interval propagated from config; all-zero (now 17 zeros) produces no jobs.
  - `scheduler_test.cpp`: `InitializeRegistersAllEnabledJobs` count 16 → 17; the three `{…16 zeros…}` aggregate inits → 17 zeros so the "0 jobs" / "1 job" assertions stay correct (the 17th member would otherwise default to 3600 and add a job).
  - `scheduler_config_test.cpp`: the all-zero validation case updated to 17 zeros.
  - `service_account_test.cpp` (2 new): the scheduler is granted `manage_class_schedule` when seeded; `Ensure` still succeeds (and grants `manage_subscriptions`) when it isn't seeded.

**Note:** the `scheduled_job.cpp` header comment still says "all jobs require manage_subscriptions" — that's now stale (the class crons require `manage_class_schedule`); left as-is to keep this change scoped, but worth a follow-up cleanup.

## 7. Frontend ✅ (button + plumbing + page-wiring + panel — all complete)

### 7.1 Catalog / calendar future-session hint ✅
- [x] Built a **reusable standalone `SignupReminderButton`** (`shared/components/signup-reminder-button/`): inputs `[classScheduleSlotId]` + `[occurrenceDateUs]`; the "🔔 Remind me when sign-ups open" button calls `requestSignupReminder` and, on the response, either flips to a confirmed "We'll remind you" state (`window_already_open=false`, success toast) or shows an **info toast "Sign-ups are already open for this session."** (`window_already_open=true`) and stays a button. Errors → error toast, stays actionable. Uses the existing `ToastService`. The button needs no up-front window state — the backend's `window_already_open` flag drives the UX.
- [x] Spec — `signup-reminder-button.component.spec.ts` (5 cases): renders; success → confirmed + success toast; already-open → info toast, stays a button; failure → error toast, stays actionable; repeat-click no-op once set.
- [x] **Backend prerequisite built** — `GET /api/me/upcoming_signup_offerings?from=&to=` (`get_my_upcoming_signup_offerings.{h,cpp}`, 366-day range guard) backed by `SignupReminderHelper::GetUpcomingSignupOfferingsForPerson` → `SignupOffering` {slot, occurrence, class, kind, start/end, `window_open`, `window_opens_at_us`, `has_pending_reminder`}. Workshops emit one offering per occurrence; **series emit one per run (earliest occurrence)**. Access-filtered via `ClassAccessHelper`. Test `get_my_upcoming_signup_offerings_test.cpp` (401, bad-range, lists workshop with window state).
- [x] **Page wired** — new member page **`upcoming-offerings`** (`pages/account/upcoming-offerings/`, route `/my/upcoming-offerings`, "Workshops & Series" dashboard card on the account page). Lists offerings for the next 120 days; window-open → "Sign-ups are open" indicator; window-closed → "Sign-ups open {date}" + the `SignupReminderButton` (or a "We'll remind you" badge when `has_pending_reminder`). Spec `upcoming-offerings.component.spec.ts` (5 cases: load, button rendered for closed, badge when pending, empty, error).

### 7.2 My signup-reminders panel ✅
- [x] **Backend prerequisite built** — `GET /api/me/signup_reminders` (`get_my_signup_reminders.{h,cpp}`) → `{reminders:[...]}` backed by `SignupReminderHelper::GetPendingRemindersForPerson` (table helper `GetPendingForPerson`, ordered by `notify_at_us`). Test `get_my_signup_reminders_test.cpp` (401, empty, lists).
- [x] Panel added to the **Notification Preferences** page (`notification-preferences`): a "Sign-up reminders" mat-card lists the user's pending reminders (class name + occurrence date) with a per-row **Cancel** button → `cancelSignupReminder` + optimistic removal. Loading/error/empty states. Spec extended (+3 cases: loads on init, cancel removes, error state).

### 7.3 `ServerAccess` extensions ✅
- [x] `requestSignupReminder(classScheduleSlotId, occurrenceDateUs)` (POST) and `cancelSignupReminder(classScheduleSlotId, occurrenceDateUs)` (DELETE, query params) across interface / network / proxy / mock. **Keyed by (slot, occurrence), not the plan's stale `eventSessionId`.**
- [x] `getMyPendingSignupReminders()` (GET `/api/me/signup_reminders`) and `getUpcomingSignupOfferings(fromUs, toUs)` (GET `/api/me/upcoming_signup_offerings`) added across interface / network / proxy / mock. **Network layer coerces `window_open` + `has_pending_reminder` (string→bool) and `Number()`s the numeric fields** (KVT→JSON leaves booleans as `"true"`/`"false"` strings).
- [x] `sendSignupOpenReminders()` (POST `/api/admin/send_signup_open_reminders` → `{sent_count}`) added across interface / network / proxy / mock. **Admin manual-trigger button** ("Send sign-up reminders now") added to the Class Schedules page header (`/manage/class-schedules`, shares the `manage_class_schedule` permission); shows a "Sent N sign-up reminder(s)." confirmation. Wraps the pre-existing endpoint — **no backend change**. Component spec +3 cases (count plural/singular, error); mock spec +1 case (retires reminders) and the logged-out batch now covers all 10 Phase 10/11 methods.
- [x] `ServerAccess.mock.spec.ts`: request/cancel cases + list/offerings cases (pending list reflects create/cancel; offerings carry window state + reflect a pending reminder); the logged-out batch now covers all 9 Phase 10/11 methods.

### 7.4 Types ✅
- [x] `signup-reminder.types.ts`: `RequestSignupReminderResult` (the POST response) + `SignupReminderInfo` (the reminders panel) + `UpcomingSignupOffering` (the offerings page), each mirroring the backend KVT.

## 8. Admin Metadata (inspection only) ✅

- [x] `signup_open_reminders` added to `admin_nested_tables` (`PopulateAdminNestedTables` in `create_database.cpp`), nesting under `people` via `person_id` — mirrors the Phase 6 `user_notification_preferences` / `ical_feed_tokens` pattern. **No `admin_table_permissions` mapping → admin-only** (which satisfies the plan's "Permission `admin`"). Inspection only — there is no admin authoring workflow (rows are created by the "Remind me" button and consumed/cancelled by the cron + booking dedupe). Seed-data registration (consistent with the other ~40 `AddRow` calls), so no dedicated unit test.

## 9. Tests-Required Summary ✅

- [x] **Table helper tests** (CRUD + partial-unique-index + ready-to-send filter) — `signup_open_reminders_test.cpp`, 11 cases (§3).
- [x] **`signup_reminder_helper_test.cpp`** (13 cases, §4 + §5):
  - [x] Request creates reminder at correct notifyAtUs.
  - [x] Request rejects when window already open (no window + already-open).
  - [x] SendPendingReminders sends + marks notified (+ skips future, null-mail no-op).
  - [x] **Series reminder → one email with a multi-VEVENT `.ics`** (N VEVENTs computed via the same derivation, ASSERT ≥ 2); **single-workshop → one email, one-VEVENT `.ics`** (resolved OQ-P11-1).
  - [x] Booking the same session cancels the pending reminder (`booking_helper_test.cpp::BookEventCancelsPendingSignupReminder` + the helper-level `CancelledReminderIsNotSent`); plus the §5 `CancelReminder` business method (2 cases).
- [x] **Endpoint tests** — `post_signup_reminder_test` (6), `delete_signup_reminder_test` (4), `admin_send_signup_reminders_test` (2).
- [x] **Schema + DDL** — `signup_open_reminders_test.cpp` schema cases (§2), `scheduling_key_value_table_test.cpp` (`SignupReminderInfo` + the today-feed fields).
- [x] **Scheduler** — `scheduled_job_test` / `scheduler_test` / `scheduler_config_test` (job wiring + counts), `service_account_test` (scheduler permission grant).
- [x] **Frontend specs** — `signup-reminder-button.component.spec.ts` (5), `ServerAccess.mock.spec.ts` (request/cancel/list/offerings + 9-method logged-out batch), `upcoming-offerings.component.spec.ts` (5), `notification-preferences.component.spec.ts` (+3 reminders-panel cases), `user.component.spec.ts` (card count 13→14 + Workshops & Series nav). **Verified green: full affected-spec run 497/497.**
- [x] **§7.1 page-wiring + §7.2 panel** — both prerequisite endpoints built (`get_my_upcoming_signup_offerings`, `get_my_signup_reminders`) and the surfaces wired (offerings page + notification-preferences panel), with specs. Backend endpoint tests `get_my_upcoming_signup_offerings_test.cpp` + `get_my_signup_reminders_test.cpp`.

## 10. Cross-Layer Acceptance Criteria

A gold member (advance_days=42) views a "6-Week Aerial 101" series starting in 60 days:
- [x] Catalog (the `/my/upcoming-offerings` page) shows "Sign-ups open <today + 18 days>" with a Remind-me button.
- [x] Click → reminder row created with `notify_at_us = session_start - 42 days`.
- [x] On day 18, the hourly cron emails **one** "Sign-ups are open for 6-Week Aerial 101 starting {date}" email — carrying a multi-VEVENT `.ics` with one VEVENT per instance of the 6-week run (resolved OQ-P11-1), not six separate emails.
- [x] If user books before day 18, the reminder is cancelled and no email goes out.

## 11. Workshop Per-Session Booking ✅ (§11.1 backend + §11.2 frontend)

**Motivation.** The public class-detail page now lists *derived* upcoming workshop occurrences (the `ClassCatalogHelper::GetClassDetail` derive-from-schedule change), so a freshly-authored workshop shows its real dates immediately. But that "Upcoming Sessions" list is **display-only** — there's no per-occurrence "Book" button. **Series** already have booking (the "Series Runs" section → `/shop/series`), and **standalone event sessions** are bookable by id via `BookEvent`. The gap is the **à-la-carte workshop occurrence**: a member can see the date but can't book that specific session. Closing it is what makes a multi-date / single-date workshop actually sellable from the catalog.

**Core problem.** A derived workshop occurrence has **no `event_sessions` row** until it's materialized (a booking/check-in/cancel/sub triggers `EnsureSessionExists`). So "book this occurrence" must **materialize first**, then run the existing event booking + payment flow against the resulting session id.

### 11.1 Backend (do first) ✅

**Built (2026-06-19).** The occurrence booking is a thin *validate → materialize → BookEvent* wrapper, so access gate, capacity/waitlist, pricing→purchase, the §4.4 reminder-cancel, and the double-book conflict guard all come from the existing `BookEvent` path; payment stays the separate `purchase_pay_card` step.
- [x] **`ClassScheduleHelper::IsDerivableOccurrence(slot, occurrenceDateUs)`** — rejects fabricated occurrences (non-midnight-aligned dates, wrong weekday, dates outside the run window, cancelled occurrences) before anything is materialized. Tests: `class_schedule_helper_test.cpp` (6 cases).
- [x] **`BookingHelper::BookClassOccurrence(BookClassOccurrenceRequest)`** — `IsDerivableOccurrence` → `EnsureSessionExists` (idempotent materialize) → `BookEvent`. New error `kErrorOccurrenceNotFound`. RSVP-vs-paid is implicit (the resolved product price drives the purchase total — $0 for a free/included workshop, >0 for paid). Refund/cancel governed by the product's `cancellation_policy_id` via the normal cancel path. Tests: `booking_helper_test.cpp` (8 cases: paid, free/$0, fabricated date, unknown slot, cancelled, idempotent+double-book conflict, waitlist-when-full, reminder-cancel, access-gate reject).
- [x] **Endpoint** `POST /api/me/book_class_occurrence` (`book_class_occurrence.{h,cpp}`, registered in `web_app.cpp` + CMake) — body `{class_schedule_slot_id, occurrence_date_us, coupon_code?, staff_override?, override_reason?}`; returns `{purchase, booking, waitlisted}`; OCCURRENCE_NOT_FOUND→404, conflict→409, missing-requirements→403, etc.; waitlist confirmation email. Tests: `book_class_occurrence_test.cpp` (401, happy path, fabricated→404, unknown slot→404, missing fields→400).
- [x] **Idempotency / double-submit** — `EnsureSessionExists` is idempotent (one `event_sessions` row per occurrence) and `BookEvent`'s overlapping-booking check returns `BOOKING_CONFLICT` (409) on a repeat by the same person.

<details><summary>Original plan (for reference)</summary>

- [ ] **Occurrence booking helper** (`business_logic/scheduling/` or extend `BookingHelper`): given `(class_schedule_slot_id, occurrence_date_us)` →
  1. `CheckAccess` (membership/skill gate) — reject if not allowed.
  2. `EnsureSessionExists(slotId, occurrenceDateUs)` to materialize the occurrence into `event_sessions` (idempotent) — book-time materialization (resolved §11.3).
  3. **Capacity / attendance cap — enforced for both included and paid bookings** (resolved §11.3) so attendance is tracked and a cap can apply (over-capacity → waitlist or reject; reuse the `BookingHelper` capacity path).
  4. **Branch on included vs paid (resolved §11.3):**
     - *Included in membership* (`priceInfo.isIncludedInMembership`) → create the booking as a **free RSVP** (no payment) so attendance is tracked under the cap.
     - *Paid* → resolve the per-session price from the workshop's active-instance product (`ResolveBestPriceForPerson`, surfaced in `ClassDetail.priceInfo`) and run the payment path.
  5. Delegate to the existing `BookingHelper` keyed by the materialized `event_session` id — booking-only for the RSVP case; `BookEvent` + `purchase_pay_card` for the paid case. The occurrence's **product refund policy** (resolved §11.3) governs later cancel/refund, not hardcoded workshop copy.
  6. On success, cancel any pending sign-up reminder for `(person, slot, occurrence)` — already wired in `booking_helper.cpp` §4.4.
- [ ] **Endpoint** — a thin `POST /api/me/book_class_occurrence` taking `class_schedule_slot_id` + `occurrence_date_us` (query/body int64-safe), returning the booking + a payment handle (or reusing the purchase/pay-card two-step the shop already uses). Keep it thin per the endpoints-layer rule; logic in the helper.
- [ ] **Idempotency** — guard double-submit (one confirmed booking per person+occurrence); the materialized `event_sessions` row + a unique booking guard.
- [ ] **Tests (required)** — helper tests (access reject, materialize-then-book, capacity, price resolution, reminder-cancel), endpoint tests (401, access 403, happy path, double-book).

</details>

### 11.2 Frontend ✅ (2026-06-19)

**Backend prerequisite shipped:** `UpcomingSessionInfo` now carries `class_schedule_slot_id` + `occurrence_date_us` (struct + `UpcomingSessionInfoToKeyValueTable` + populated from `DerivedSession` in `GetClassDetail`) so the client can name the occurrence to book. Tests updated: `scheduling_key_value_table_test`, `class_catalog_helper_test`, `get_class_detail_test`. **Needs a server rebuild.**

**Final model (redesigned 2026-06-19 per Mason):** workshops are booked **per occurrence from the class page**, not as a generic catalog product. There is no dateless Shop entry — clicking a session date materializes that specific occurrence and hands off to the **existing event-booking page** so it inherits *every* payment option (card, saved cards, vouchers, coupons, pay-for-other), instead of a parallel minimal booking page.

**Reuse-via-materialize (Approach B — replaces the earlier `ClassOccurrenceBookingComponent`).** Rather than a bespoke booking page, clicking an occurrence:
1. `materializeClassOccurrence(slotId, occurrenceDateUs)` → POST `/api/me/materialize_class_occurrence` validates derivability (`IsDerivableOccurrence`) and `EnsureSessionExists` (idempotent — one row per slot+day), returning `{event_session_id}`.
2. Client navigates to **`/shop/event/<event_session_id>`** — the standard `EventBookingComponent`, which already has the full payment surface. Because the materialized session carries `class_id`/`class_name` + the exact wall-clock-encoded-as-UTC `start_time_us`, booking targets the precise slot+date/time.

- [x] On the class-detail **"Upcoming Sessions"** list, each occurrence card for a bookable **workshop** is **clickable** (whole card, `role=button`/keyboard-enter), with a CTA hint: paid → **"Book — {price}"**, included → **"Reserve a free spot"**, logged-out → **"Log in to book"**. `canBookOccurrences` gates on `kind==='workshop'` + a bookable `pricingState`; recurring / series / members-only / unavailable are not clickable.
- [x] Clicking → `onBookOccurrence` calls `materializeClassOccurrence(slot, occurrence)` then `router.navigate(['/shop/event', event_session_id])`; logged-out → `/login`; a materialize failure shows an inline `.book-occurrence-error` and does not navigate. A `bookingOccurrence` flag guards double-click.
- [x] **`EventBookingComponent` reused as-is** for the booking + payment (card / saved cards / voucher / coupon / pay-for-other). Only fix needed: `formatTimeRange()` formats class-occurrence sessions (`session.class_id` set) in **UTC** (wall-clock encoding) instead of the facility tz, so a 10:00 slot reads 10:00. Ad-hoc events keep facility-tz formatting.
- [x] **Backend `POST /api/me/materialize_class_occurrence`** (`materialize_class_occurrence.{h,cpp}`): login→401, missing/≤0 fields→400, non-derivable→404, else `{event_session_id}`. Registered in `web_app.cpp` + `CMakeLists.txt`. Tests: `materialize_class_occurrence_test.cpp` (401, materialize+idempotent, 404 fabricated, 400 missing). **Needs a server rebuild.**
- [x] **Removed the generic dateless Shop entry:** `catalog.component` filters `kind === 'class'`, **and** the backend `CatalogHelper` now excludes any product backing a `class_instance` (`ClassInstances::GetProductIdsUsedByInstances`) so a user-created product (not kind `'class'`) still won't leak into Shop. Tests: `catalog_helper_test`, `class_instances_test`.
- [x] **"Members only" pricing bug fixed:** `ClassCatalogHelper::GetClassDetail` falls back from `GetActiveInstance(now)` to the next **upcoming** instance, so a future-dated workshop resolves its price instead of reading members-only. Test: `class_catalog_helper_test` (future-workshop-from-upcoming-instance).
- [x] **Removed** the interim `/shop/class-occurrence` page (`ClassOccurrenceBookingComponent` + route) — superseded by the event-booking reuse.
- [x] **Surfaces to the booking flow link to the class page** (`/classes/:id`): Our Schedule slot cards, Our Classes cards, and the Upcoming Workshops & Series offering titles.
- [x] `ServerAccess.materializeClassOccurrence(slotId, occurrenceDateUs)` → `Observable<{event_session_id}>`, across interface / network / proxy / mock + mock-spec (success, 404 sentinel slot, 401). (`bookClassOccurrence` from §11.1 retained as the lower-level endpoint.)
- [x] Specs — `class-detail.component.spec.ts` (non-workshop not clickable, paid click → materialize+`/shop/event` nav, included → materialize+nav, materialize-failure shows error & no nav, logged-out → /login, members-only not bookable), `event-booking.component.spec.ts` (class-occurrence time formats in UTC), `ServerAccess.mock.spec.ts` (materialize success/404/401), `catalog.component.spec.ts` (class filtered), `our-schedule` + `upcoming-offerings` link tests. **All 557 green.**

### 11.3 Decisions (resolved — Mason)
- [x] **Included (membership) workshops → free RSVP with tracked attendance + cap.** A membership-covered workshop is booked as a **free RSVP** (creates a booking, **no payment**) so attendance is tracked, and it **respects an attendance/capacity cap** (Mason: "we want to track attendance for those… there might also be an attendance cap"). So the booking path branches: included → booking-only RSVP; paid → booking + payment. Capacity is enforced for **both**. Folded into §11.1 (3–5) and §11.2.
- [x] **Materialize-on-book** (Mason: "whatever is simpler"). Keep book-time materialization via `EnsureSessionExists`; no surface needs the `event_sessions` row earlier, and it keeps the table clean. Confirmed in §11.1 (2).
- [x] **Refund/cancel policy is a product attribute** (Mason: "isn't this tied to the product?") — **confirmed:** `products.cancellation_policy_id` already exists, so the booked occurrence's product carries its own cancel/refund policy. Reuse it: surface the policy on the booking CTA / confirmation and honor it on cancel, rather than hardcoding workshop copy. No new field needed. Folded into §11.1 (5) and §11.2.

> Depends on: the §7 reminder cancel-on-book hook (done) and the class-detail derive change (done). Reuses Phase 7 series booking + the shop payment flow.

## 12. Open Questions

Both resolved (Mason, 2026-06-09) and folded into the plan above (§1.1 Locked-in + the cited sections).

- **OQ-P11-1. — RESOLVED (Mason refines the recommendation).** One email per series (single line: name + start + end + per-instance schedule summary), **but it must carry a multi-VEVENT `.ics` with one VEVENT per instance** (reuse the Phase 7 / Phase 4 iCal generator). Single workshop/intro → one-VEVENT `.ics`. Folded into §1.1, §4.3, §9, §10.
- **OQ-P11-2. — RESOLVED (Mason: "go with your recommendation").** Reminders are class series / workshops / intro only; non-class events and bookable services are out of scope for Phase 11. Folded into §1.1.

## 13. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 11.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 7 - Class Series and Workshops]].
- Scheduler: [[Scheduled Jobs]].
- Booking window source: [[Event Polish- Scheduling Should Have Items]].
