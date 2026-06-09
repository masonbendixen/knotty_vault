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

### 2.1 `signup_open_reminders` table
- [ ] `db_schema/signup_open_reminders.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL REFERENCES people(id)`
  - `class_schedule_slot_id BIGINT NOT NULL REFERENCES class_schedule_slots(id)` — the occurrence's slot (the occurrence usually has no persisted `event_sessions` row yet)
  - `occurrence_date_us BIGINT NOT NULL` — the specific day
  - `notify_at_us BIGINT NOT NULL` — moment when user's window opens (occurrence start − their best `advance_days` × 86400_000_000)
  - `notified_us BIGINT NULL`
  - `cancelled_us BIGINT NULL`
  - `created_us BIGINT NOT NULL`
- [ ] Partial unique index `UNIQUE (person_id, class_schedule_slot_id, occurrence_date_us) WHERE notified_us IS NULL AND cancelled_us IS NULL` — one pending reminder at a time.
- [ ] Index on `notify_at_us` for the daily cron scan.

### 2.2 Wire into DB init
- [ ] `make_database_info.cpp` + `create_database.cpp`.

## 3. Table Helpers

### 3.1 `TableHelpers::SignupOpenReminders`
- [ ] `AddReminder(Transaction&, KeyValueTable&)` — idempotent on the partial unique index.
- [ ] `CancelReminder(Transaction&, personId, eventSessionId)` — sets `cancelled_us=now`.
- [ ] `GetPendingReadyToSend(Transaction&, nowUs)` → list of pending reminders where `notify_at_us <= nowUs` AND `notified_us IS NULL` AND `cancelled_us IS NULL`.
- [ ] `MarkNotified(Transaction&, id, nowUs)`.
- [ ] Tests.

## 4. Business Logic — `SignupReminderHelper`

Files: `business_logic/scheduling/signup_reminder_helper.h/.cpp/_test.cpp`.

### 4.1 Best-window computation
- [ ] `int64_t ResolveBestAdvanceDaysForPerson(Transaction&, int64_t productId, int64_t personId, int64_t asOfUs)`:
  1. Load all `product_booking_windows` rows for the product.
  2. Filter to rows where `permission_id IS NULL` OR `permission_id` ∈ user's permission set.
  3. Return `max(advance_days)` across the filtered set. If no matches → 0 (booking always open today, or never open — caller distinguishes).

### 4.2 Request-reminder flow
- [ ] `struct RequestReminderResult { bool ok; int64_t reminderId; int64_t notifyAtUs; std::string errorCode; }`.
- [ ] `RequestReminder(Transaction&, personId, classScheduleSlotId, occurrenceDateUs)`:
  1. Resolve the occurrence's product from the slot's active `class_instances` row; compute `occurrenceStartUs` from the slot + date.
  2. Compute `advanceDays = ResolveBestAdvanceDaysForPerson(productId, personId, now)`.
  3. Compute `notifyAtUs = occurrenceStartUs - advanceDays * 86400_000_000`.
  4. If `notifyAtUs <= now` → return `WINDOW_ALREADY_OPEN` (the user can book right now; no reminder needed).
  5. Insert `signup_open_reminders` row (slot + occurrence-date keyed; no `event_sessions` row need exist yet).
- [ ] Tests.

### 4.3 Sending pending reminders
- [ ] `int SendPendingReminders(Transaction&, MailHelper*, int64_t nowUs)`:
  1. `GetPendingReadyToSend(nowUs)`.
  2. For each pending reminder, resolve the occurrence's product/class via the slot's active `class_instances` row and build the email (resolved OQ-P11-1):
     - **Series run** (`classes.kind='series'`): a **single** email — subject/body list the series as one line (name + start + end + per-instance schedule summary), and attach a **multi-VEVENT `.ics`** with one VEVENT per upcoming instance of the run. Derive the run's occurrences via `ClassScheduleHelper::GetDerivedSessionsForRange(classId, runStartUs, runEndUs)` and emit the iCal with the existing Phase 7 series-confirmation / Phase 4 multi-VEVENT generator. **One email per series, not per instance.**
     - **Single workshop / intro occurrence**: body "Sign-ups are open for {className} on {date}" + a **one-VEVENT `.ics`** for that occurrence.
     - Wrap the body with `NormalizeCrLf()` (mailio CRLF rule) and queue via `MailHelper`.
  3. `MarkNotified`.
  4. Return count.
- [ ] Tests with `TestMailHelper`: a series reminder queues exactly **one** email whose `.ics` contains **N VEVENTs** (N = upcoming instances); a single-workshop reminder queues one email with a one-VEVENT `.ics`.

### 4.4 Booking-side dedupe
- [ ] In `BookingHelper::BookEvent` (and the series-booking flow), after creating the booking, call `SignupOpenReminders::CancelReminder(personId, eventSessionId)` — no-op if no pending reminder.
- [ ] Test the dedupe: request a reminder, then book the same session, then run `SendPendingReminders` → zero sent.

### 4.5 KeyValueTable conversions
- [ ] `SignupReminderInfoToKeyValueTable(...)`.

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
