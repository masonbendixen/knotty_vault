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

## 5. Endpoints

- [ ] `POST /api/me/signup_reminder` body `{ class_schedule_slot_id, occurrence_date_us }` — calls `RequestReminder`. Returns reminderId + notifyAtUs. Endpoint test (success + window-already-open + duplicate-no-op).
- [ ] `DELETE /api/me/signup_reminder/<eventSessionId>` — cancel a pending reminder.
- [ ] `POST /api/admin/send_signup_open_reminders` — cron-callable, idempotent. Permission `admin`. Endpoint test.

## 6. Scheduled job integration

- [ ] Add hourly job to `knottyyoga_helper`: `POST /api/admin/send_signup_open_reminders`. Idempotent.

## 7. Frontend

### 7.1 Catalog / calendar future-session hint
- [ ] In the existing class / series / workshop card render, if the user's window is closed for this session, show "Sign-ups open {local-date}" chip + a "🔔 Remind me" button.
- [ ] Clicking the button calls `requestSignupReminder(eventSessionId)`. If `WINDOW_ALREADY_OPEN`, surface a toast "Sign-ups are already open for this session".
- [ ] Spec.

### 7.2 My signup-reminders panel (optional)
- [ ] On `/my/account/notification-preferences` (created in Phase 6), add a section listing the user's pending reminders with a "Cancel" action each.
- [ ] Spec.

### 7.3 `ServerAccess` extensions
- [ ] `requestSignupReminder(eventSessionId)`, `cancelSignupReminder(eventSessionId)`, `getMyPendingSignupReminders()`.
- [ ] Update `ServerAccess.mock.spec.ts`.

### 7.4 Types
- [ ] `signup-reminder.types.ts`: `SignupReminderInfo`.

## 8. Admin Metadata (inspection only)

- [ ] `signup_open_reminders` → `admin_nested_tables` under `people` keyed by `person_id`. Permission `admin`. This table is **system/user-generated** — rows are created by the user's §7.1 "🔔 Remind me" button and consumed/cancelled by the cron + booking-dedupe paths. There is no admin *authoring* workflow here, so registering it purely for inspection in Manage Data is appropriate (per memory `feedback_manage_data_is_debug_only.md`); it must NOT become a flow where staff hand-create reminder rows.

## 9. Tests-Required Summary

- [ ] Table helper tests (CRUD + partial-unique-index + ready-to-send filter).
- [ ] `signup_reminder_helper_test.cpp`:
  - Request creates reminder at correct notifyAtUs.
  - Request rejects when window already open.
  - SendPendingReminders sends + marks notified.
  - Series reminder sends **one** email with a multi-VEVENT `.ics` (N VEVENTs = upcoming instances); single-workshop reminder sends one email with a one-VEVENT `.ics` (resolved OQ-P11-1).
  - Booking the same session cancels pending reminder.
- [ ] Endpoint tests.
- [ ] Frontend specs.

## 10. Cross-Layer Acceptance Criteria

A gold member (advance_days=42) views a "6-Week Aerial 101" series starting in 60 days:
- [ ] Catalog shows "Sign-ups open <today + 18 days>" with a Remind-me button.
- [ ] Click → reminder row created with `notify_at_us = session_start - 42 days`.
- [ ] On day 18, the hourly cron emails **one** "Sign-ups are open for 6-Week Aerial 101 starting {date}" email — carrying a multi-VEVENT `.ics` with one VEVENT per instance of the 6-week run (resolved OQ-P11-1), not six separate emails.
- [ ] If user books before day 18, the reminder is cancelled and no email goes out.

## 11. Open Questions

Both resolved (Mason, 2026-06-09) and folded into the plan above (§1.1 Locked-in + the cited sections).

- **OQ-P11-1. — RESOLVED (Mason refines the recommendation).** One email per series (single line: name + start + end + per-instance schedule summary), **but it must carry a multi-VEVENT `.ics` with one VEVENT per instance** (reuse the Phase 7 / Phase 4 iCal generator). Single workshop/intro → one-VEVENT `.ics`. Folded into §1.1, §4.3, §9, §10.
- **OQ-P11-2. — RESOLVED (Mason: "go with your recommendation").** Reminders are class series / workshops / intro only; non-class events and bookable services are out of scope for Phase 11. Folded into §1.1.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 11.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 7 - Class Series and Workshops]].
- Scheduler: [[Scheduled Jobs]].
- Booking window source: [[Event Polish- Scheduling Should Have Items]].
