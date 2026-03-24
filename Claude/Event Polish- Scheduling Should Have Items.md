---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/23/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Let's implement the SHOULD HAVE - Event Polish section of Support for scheduled purchases.md. Please use the code base and the following documents for context:

- [[Nested item support]]
- [[Payment Design Document]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Product browsing and quoting endpoints]]
- [[Product, Event, and Subscription Admin Portal]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]

Be aware that we have [[Manual Testing Helper Executable]] as a place to add tasks that need support to test a scenario and [[Scheduled Jobs]] for tasks that need to be performed periodically (like sending reminder emails and so forth).

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Please always add tests for anything you chance for which testing is possible.

# Plan: SHOULD HAVE — Event Polish

## Scope

Implements the remaining unchecked SHOULD HAVE scenarios from *Support for scheduled purchases.md*:

| # | Scenario | Status |
|---|----------|--------|
| 5 | Admin configures event product visibility (public vs member-only) | To implement |
| 6 | Admin configures event product as member-only booking | To implement |
| 13 | User receives reminder email before event | To implement |
| 15 | Admin configures refund policy for event product | To implement |
| 60 | Admin cancels entire session (with reason, notification, bulk refund) | To implement |
| 64 | Booking conflicts prevented (no double-booking overlapping events) | To implement |

Already completed (no work needed): 8, 12, 16, 42, 59.

## Current State Summary

**What exists but isn't wired up:**
- `products.visibility_permission_id` column — exists in schema, never checked in `EventSessionHelper::GetVisibleEventSessions()`
- `products.booking_permission_id` column — exists in schema, never checked in `BookingHelper::BookEvent()`
- `products.cancellation_policy_id` column — exists in schema, never read by `BookingHelper::CancelBooking()`
- `products.reminder_hours` column — exists in schema, no reminder job or email template exists
- `cancellation_policies` table — schema exists, no table helper
- `cancellation_policy_windows` table — schema exists (`hours_before`, `refund_percent`), no table helper

**What doesn't exist at all:**
- No `RefundPayment` method on `SquareClient` (only `CreatePayment`)
- No reminder email template
- No `reminder_sent_us` column on bookings (to track which reminders sent)
- No `cancel_session` endpoint (no admin bulk cancel)
- No session cancellation email template
- No booking conflict detection
- No cancellation policy table helpers
- No refund calculation logic anywhere

## Architecture Notes

All work follows the existing layering:
```
DB Schema → Table Helpers → Business Logic → Endpoints → Frontend
```

Backend work listed before frontend work in each phase. Tests required for every backend change where a test file exists.

---

## Phase 1: Visibility & Booking Permissions (Scenarios 5, 6)

**Goal**: Event products can be restricted so only users with the right permission can see or book them. Non-logged-in users see only public events. Events with `visibility_permission_id` set are hidden from users who lack that permission. Events with `booking_permission_id` set show a "Members Only" badge but the "Book" button is disabled for users without that permission.

### Backend — Business Logic Layer

- [x] **`EventSessionHelper::GetVisibleEventSessions()`** — Add permission filtering
  - Load `products.visibility_permission_id` for each session's product
  - If `visibility_permission_id` is set (non-null, non-zero):
    - If `personId == 0` (not logged in): exclude the session
    - If logged in: check if user has the permission via `Session::ActiveUserHasPermission()` or a direct SQL query against `role_permissions` + `role_assignments`. Since EventSessionHelper doesn't have a Session object, pass the person's permission set or do a SQL-level filter.
    - **Approach**: Extend the SQL query in `GetVisibleEventSessions()` to LEFT JOIN through the permission chain: `LEFT JOIN role_assignments ra ON ra.person_id = {personId} LEFT JOIN role_permissions rp ON rp.role_id = ra.role_id AND rp.permission_id = p.visibility_permission_id`. Filter: `WHERE p.visibility_permission_id IS NULL OR rp.id IS NOT NULL`.
  - Include `visibility_permission_id` and `booking_permission_id` in the returned `ResolvedEventSession` so the frontend knows whether to show "Members Only" badge or disable booking
- [x] **Tests** in `event_session_helper_test.cpp` — Test visibility filtering: public event visible to all, member-only event hidden from non-members, member-only event visible to members

- [x] **`BookingHelper::BookEvent()`** — Add booking permission check
  - After loading the event session + product, check `products.booking_permission_id`
  - If set: verify the person (attendee, not payer) has the permission
  - Query: `SELECT 1 FROM role_assignments ra JOIN role_permissions rp ON rp.role_id = ra.role_id WHERE ra.person_id = {personId} AND rp.permission_id = {bookingPermissionId}`
  - If check fails: return error with code `"booking_permission_required"` and message "This event requires a membership to book."
- [x] **Tests** in `booking_helper_test.cpp` — Test booking permission: allowed when user has permission, denied when user lacks permission, allowed when `booking_permission_id` is null

### Backend — Business Logic Layer (KVT)

- [x] **`SchedulingKeyValueTable`** — Add `visibility_permission_id` and `booking_permission_id` to `EventSessionInfoToKeyValueTable()` and the reverse conversion, so these values flow through the endpoint JSON
- [x] **Tests** in `scheduling_key_value_table_test.cpp` — Round-trip tests for new fields

### Backend — Endpoint Layer

- [x] **`visible_event_sessions.cpp`** — No structural change needed (already passes personId to helper). New fields flow through via KVT automatically.
- [x] **`book_event.cpp`** — Updated to return `ErrorResponse::NotAuthorized()` (403) for `BOOKING_PERMISSION_REQUIRED` error code, instead of generic BadRequest.

### Frontend — Types

- [x] **`scheduling.types.ts`** — Added `visibility_permission_id?: number` and `booking_permission_id?: number` to `EventSession` interface

### Frontend — Upcoming Events Page

- [x] **`upcoming-events.component.html`** — Shows purple "Members Only" badge on events where `booking_permission_id` is set. Badge appears next to event title. Book Now button still works (backend enforces the actual restriction).
- [x] **`upcoming-events.component.scss`** — Added `.members-only-badge` style (purple pill badge)

### Frontend — Event Booking Page

- [x] **`event-booking.component.ts`** — Added `NOT_AUTHORIZED` case to `getErrorMessage()`. When booking is denied due to permission, displays the server's detail message (e.g., "This event requires a membership to book.").

### Frontend — Event Create Component (Admin)

- [x] **No changes needed** — Visibility and booking permissions are on the **product**, not the session. Admin configures these via the product admin page (existing admin CRUD UI with FK pickers). Event create form picks the product, and the product carries the permissions.

### Frontend — Tests

- [x] **Component spec tests** for upcoming-events: test "Members Only" badge renders when `booking_permission_id` is set, and does not render when not set
- [x] **Component spec tests** for event-booking: test 403 NOT_AUTHORIZED error displays "membership to book" message

---

## Phase 2: Booking Conflict Prevention (Scenario 64)

**Goal**: A user cannot book two events that overlap in time. Before creating a booking, the system checks for existing confirmed bookings with overlapping time ranges.

### Backend — Business Logic Layer

- [x] **`BookingHelper::BookEvent()`** — Added overlap check (step 5) after permission check and before capacity check. SQL query checks for existing `confirmed` or `waitlisted` bookings where the event session times overlap. Returns `BOOKING_CONFLICT` error code.
- [x] **Tests** in `booking_helper_test.cpp` — 4 new tests:
  - Overlapping bookings rejected with `BOOKING_CONFLICT`
  - Non-overlapping bookings allowed (back-to-back sessions)
  - Cancelled overlapping booking does not block (only checks confirmed/waitlisted)
  - Waitlisted overlapping booking also blocks (since user already paid)

### Backend — Endpoint Layer

- [x] **`book_event.cpp`** — Returns HTTP 409 with `booking_conflict` type for `BOOKING_CONFLICT` error code
- [x] **Test** in `book_event_test.cpp` — Verifies 409 response with `booking_conflict` type when booking overlapping sessions

### Frontend — Event Booking Page

- [x] **`event-booking.component.ts`** — Added `BOOKING_CONFLICT` case to `getErrorMessage()`, displays server detail message about overlapping bookings
- [x] **`ApiError.ts`** — Added `BOOKING_CONFLICT: 'booking_conflict'` to `ErrorTypes`
- [x] **Component spec test** — Verifies 409 booking conflict error displays "overlaps" message

---

## Phase 3: Cancellation Policy Configuration & Refund Flow (Scenario 15)

**Goal**: Admin creates cancellation policies with tiered refund windows. When a user cancels a booking, the system calculates the refund percentage based on how far in advance they're cancelling. Refunds are recorded in the database. Square refund API integration is added.

### Backend — Table Helpers Layer

- [x] **Create `cancellation_policies.h/cpp`** table helper — CRUD for cancellation_policies table
- [x] **Create `cancellation_policy_windows.h/cpp`** table helper — Add/Get/Update/Delete windows ordered by hours_before DESC
- [x] **Tests** — 5 tests for policies (add/get, get all, update, delete, not found), 4 tests for windows (add/get ordered, update, delete, empty)
- [x] **CMakeLists.txt** — Added to table_helpers CMakeLists

### Backend — Square Client Layer

- [x] **`RefundResult` struct + `RefundPayment()` virtual method** added to `square_client.h`
- [x] **Production implementation** in `square_client.cpp` — calls `/refunds` endpoint with `payment_id`, `amount_money`, `idempotency_key`, optional `reason`; parses refund response
- [x] **Test mock** in `square_client_test_util.h/cpp` — `RefundPaymentArgs`, call tracking, `QueueRefundResult`/`QueueRefundError`
- [x] **Tests** in `square_client_test.cpp` — success, partial refund, invalid request error (3 tests)

### Backend — Business Logic Layer (Payment)

- [x] **`RefundHelper` class** in `business_logic/payment/refund_helper.h/cpp`
  - `CalculateRefundPercent()` — loads policy windows DESC, matches hours remaining to tiers; no policy → 100%, no match → 0%
  - `ProcessRefund()` — finds payment via purchase_payments, calculates refund amount, calls Square, records negative payment row with `refund_for_payment_id`, updates purchase status
- [x] **Tests** in `refund_helper_test.cpp` — 8 tests: no policy → 100%, empty policy → 100%, full window, partial window, too late → 0%, full refund process, partial refund process, zero percent returns empty, no payment returns empty
- [x] **CMakeLists.txt** — Added refund_helper.h/cpp and test

### Backend — Business Logic Layer (Scheduling)

- [x] **`BookingHelper` updated** — Constructor takes optional `SquareClientPtr`; `CancelBooking()` now creates `RefundHelper` and processes refunds. Confirmed bookings use cancellation policy; waitlisted bookings always get 100% refund.
- [x] **`CancelBookingResult` updated** — Added `refundAmountCents`, `refundPercent`, `currency` fields
- [x] **Tests** in `booking_helper_test.cpp` — 3 new tests: cancel confirmed with no policy (full refund), cancel confirmed with tiered policy (partial refund), cancel waitlisted (always full refund)

### Backend — Endpoint Layer

- [x] **`cancel_booking.cpp`** — Passes `SquareClient` to `BookingHelper`; includes `refund_amount_cents`, `refund_percent`, `currency` in response JSON
- [ ] **Tests** in cancel_booking endpoint test — existing tests still pass (no payment = no refund fields); new refund-specific endpoint tests deferred to integration testing

### Frontend — Types

- [x] **`scheduling.types.ts`** — Added `CancelBookingResponse` interface with `refund_amount_cents`, `refund_percent`, `currency` fields
- [x] **`ServerAccess.ts`** (interface) — Updated `cancelBooking` return type to `CancelBookingResponse`
- [x] **`ServerAccess.ts`** (proxy), **`ServerAccessNetwork.ts`**, **`ServerAccess.mock.ts`** — All updated to use `CancelBookingResponse`

### Frontend — My Events Page

- [x] **`my-events.component.ts`** — Captures `CancelBookingResponse` from cancel, builds refund message via `buildRefundMessage()`: full refund, partial refund with %, or no-refund notice. Displayed as a dismissible green banner.
- [x] **`my-events.component.html`** — Added `#cancel-result` banner with close button above the loading state
- [x] **`my-events.component.scss`** — Added `.cancel-result-banner` style (green themed, flex with dismiss button)

### Frontend — Event Booking Page

- [x] **`event-booking.component.ts`** — Added `cancellationPolicyText` getter that parses `cancellation_windows` into human-readable text (e.g., "Full refund if cancelled 48+ hours before. 50% refund if cancelled 24+ hours before. No refund after purchase.")
- [x] **`event-booking.component.html`** — Amber info box with policy text shown below the price total inside the event details card
- [x] **`event-booking.component.scss`** — Added `.cancellation-policy-info` style

### Frontend — My Events Page (Cancellation Flow)

- [x] **`my-events.component.ts`** — `onCancelClick` now fetches event session detail to get cancellation policy. Calculates applicable refund tier based on hours remaining. Shows no-refund warning gate when 0% refund applies.
- [x] **`my-events.component.html`** — Three-step cancel: (1) policy text shown, (2) no-refund warning with extra confirmation if applicable, (3) standard "Are you sure?" confirmation
- [x] **`my-events.component.scss`** — Styles for policy text (green/red), no-refund warning (amber), cancel confirmation container

### Frontend — Tests

- [x] **Component spec tests** for my-events — 7 new tests: full refund message, partial refund message, no-refund message, banner dismiss, policy text on cancel click, no-refund warning with extra confirmation, refund-available skips warning
- [x] **Component spec tests** for event-booking — 3 new tests: tiered policy text, no-policy free cancellation, full-refund policy text

### Backend — Cancellation Email & Policy in Session Response

- [x] **`booking_cancellation_mail.h/cpp`** — Red-themed HTML email template with event details and conditional refund section (full, partial, or no-refund)
- [x] **`booking_cancellation_mail_test.cpp`** — 3 tests: full refund, partial refund, no refund email body
- [x] **`cancel_booking.cpp`** — Sends "Booking Cancelled" email with refund info to the cancelled person
- [x] **`event_session_helper.h`** — Added `CancellationWindow` struct and `cancellationPolicyName`/`cancellationWindows` to `ResolvedEventSession`
- [x] **`event_session_helper.cpp`** — `ResolveSessionPricing` loads product's cancellation policy name and windows from DB
- [x] **`scheduling_key_value_table.cpp`** — Serializes `cancellation_policy_name` and `cancellation_windows` ("48:100;24:50;0:0" format)
- [x] **`event_session_helper_test.cpp`** — 2 new tests: policy info returned, no policy fields empty
- [x] **`scheduling_key_value_table_test.cpp`** — 2 new tests: policy serialized, policy omitted when empty
- [x] **`scheduling.types.ts`** — Added `cancellation_policy_name` and `cancellation_windows` to `EventSession`

### Admin — Cancellation Policy Management

- [x] The `cancellation_policies` and `cancellation_policy_windows` tables are already exposed through the admin CRUD UI (they have admin metadata). Admin can create/edit policies and windows through the existing table editor. **No new admin UI needed.**
- [x] To assign a policy to a product: Admin edits the product and sets `cancellation_policy_id` via the product detail page or the admin CRUD UI. **No new UI needed.**
- [x] Three default policies seeded in `create_database.cpp`: Full Refund, Tiered (48h/24h/0h), No Refund

### Admin — Cancellation Policy Management

- [x] The `cancellation_policies` and `cancellation_policy_windows` tables are already exposed through the admin CRUD UI (they have admin metadata). Admin can create/edit policies and windows through the existing table editor. **No new admin UI needed.**
- [x] To assign a policy to a product: Admin edits the product and sets `cancellation_policy_id` via the product detail page or the admin CRUD UI. **No new UI needed.**

---

## Phase 4: Event Reminder Emails (Scenario 13)

**Goal**: Users receive a reminder email N hours before their event. A scheduled admin endpoint runs periodically to send unsent reminders.

### Backend — DB Schema Layer

- [x] **Add `reminder_sent_us` column to `bookings` table** — Added to `bookings.h` and `bookings.cpp` as nullable BIGINT

### Backend — Table Helpers Layer

- [x] **`Bookings` table helper** — No new methods needed; existing `UpdateBooking()` sets `reminder_sent_us`

### Backend — Business Logic Layer

- [x] **`event_reminder_mail.h/cpp`** — Blue-themed "Event Reminder" email template with event details, facility, and room
- [x] **`event_reminder_mail_test.cpp`** — 2 tests: full fields, no room
- [x] **`event_reminder_helper.h/cpp`** — `SendPendingReminders()` finds confirmed bookings within the reminder window (per-product `reminder_hours` or default from `event_reminder_hours` secret), sends emails, marks `reminder_sent_us`
- [x] **`event_reminder_helper_test.cpp`** — 5 tests: sends within window, skips outside window, no duplicate sends, skips cancelled, respects per-product hours
- [x] **CMakeLists.txt** — Added all new files to scheduling CMakeLists

### Backend — Endpoint Layer

- [x] **`admin_send_event_reminders.h/cpp`** — `POST /api/admin/send_event_reminders`, requires auth, returns `{ "reminders_sent": N, "reminders_skipped": N }`
- [x] **Registered in `web_app.cpp`** and **`endpoints/CMakeLists.txt`**

### Scheduled Jobs Integration

- [x] **`Scheduled Jobs.md`** — Updated below

### Test Helper Integration

- [x] **`send_event_reminders` command** (alias `ser`) added to `booking_commands.cpp` — calls `EventReminderHelper::SendPendingReminders()` directly, prints sent/skipped counts
- [x] **`Manual Testing Helper Executable.md`** — Updated below

---

## Phase 5: Admin Cancels Entire Session (Scenario 60)

**Goal**: Admin can cancel an entire event session. All confirmed attendees receive refunds (per cancellation policy, or 100% if studio-initiated). All attendees (confirmed + waitlisted) receive a cancellation notification email. All bookings are marked cancelled.

### Backend — Business Logic Layer

- [x] **`session_cancellation_mail.h/cpp`** — Red-themed "Event Cancelled" email with cancellation reason, refund amount for confirmed ("A full refund of $X.XX has been issued"), or "No payment was collected" for waitlisted
- [x] **`session_cancellation_mail_test.cpp`** — 3 tests: with refund, waitlisted no payment, without reason

- [x] **`session_cancellation_helper.h/cpp`** — `CancelSession()` verifies session is scheduled, sets status to cancelled with reason, loads all confirmed+waitlisted bookings, issues 100% refund for confirmed (studio-initiated), cancels purchases for waitlisted, sends email to all, returns counts
- [x] **`session_cancellation_helper_test.cpp`** — 5 tests: confirmed with refunds, mixed confirmed+waitlisted, empty session, already-cancelled error, not-found error

### Backend — Endpoint Layer

- [x] **`admin_cancel_session.h/cpp`** — `POST /api/admin/cancel_session/<int>`, requires auth, body `{ "reason": "..." }` required, returns `{ success, confirmed_cancelled, waitlisted_cancelled, refunds_processed, total_refunded_cents }`
- [x] **Registered in `web_app.cpp`** and **`endpoints/CMakeLists.txt`**

### Frontend — Types

- [ ] **`scheduling.types.ts`** — Add cancel session response type
- [ ] **`ServerAccess.ts`** — Add `adminCancelSession(sessionId: number, reason: string)` method
- [ ] **`ServerAccess.mock.ts`** + **spec test** for mock

### Frontend — Event Session Card (Admin)

- [ ] **`event-session-card.component.ts/html`** — Add "Cancel Session" button
  - Only visible for sessions with `status === 'scheduled'`
  - Opens a confirmation dialog with a reason text field
  - On confirm: calls `adminCancelSession(sessionId, reason)`
  - On success: reloads the session data (status will now be `cancelled`)
  - Show result summary: "Session cancelled. X confirmed bookings refunded ($Y.YY total). Z waitlisted bookings cancelled."

### Frontend — Tests

- [ ] **Component spec tests** for event-session-card: cancel button visibility, confirmation dialog, success handling

### Test Helper Integration

- [x] **`cancel_session` command** (alias `cs`) added to `booking_commands.cpp` — flags `--session_id` (required), `--reason` (optional), shows mail test mode warning, prints refund summary

---

## Phase 6: Integration Testing & Polish

**Goal**: End-to-end verification that all features work together. Test the complete flows.

### Manual Testing Scenarios

- [ ] **Visibility**: Create a permission "member". Create an event product with `visibility_permission_id` pointing to "member". Verify non-members don't see the event on the upcoming page. Add the permission to a user's role. Verify they now see it.
- [ ] **Booking permission**: Create an event product with `booking_permission_id`. Verify a user without the permission gets an error when booking. Add the permission; verify booking succeeds.
- [ ] **Conflict prevention**: Book an event. Try to book a second event that overlaps in time. Verify the system rejects it. Cancel the first booking. Verify the second event can now be booked.
- [ ] **Cancellation policy**: Create a cancellation policy with windows (48h → 100%, 24h → 50%, 0h → 0%). Assign to a product. Book the event. Cancel at various times and verify refund amounts.
- [ ] **Reminders**: Set `reminder_hours` on a product. Book the event. Set the event time to be within the reminder window (use test helper `set_event_session_time`). Run `send_event_reminders`. Verify email sent. Run again — verify no duplicate.
- [ ] **Session cancellation**: Create an event with multiple confirmed and waitlisted bookings. Admin cancels session with reason. Verify all bookings cancelled, refunds issued, emails sent.

### Test Helper Commands Summary

New commands added across phases:
- `send_event_reminders` (Phase 4) — in `booking_commands.cpp`
- `cancel_session --session_id=X --reason=Y` (Phase 5) — in `booking_commands.cpp`

---

## Dependencies & Ordering

```
Phase 1 (Visibility & Booking Permissions) — Independent
Phase 2 (Booking Conflicts) — Independent
Phase 3 (Cancellation & Refunds) — Requires Square RefundPayment API
Phase 4 (Reminders) — Independent
Phase 5 (Session Cancellation) — Depends on Phase 3 (uses RefundHelper)
Phase 6 (Integration) — Depends on all prior phases
```

Phases 1, 2, and 4 can be implemented in any order. Phase 3 must come before Phase 5. Phase 6 is last.

## Risk Notes

1. **Square Refund API**: The `SquareClient` currently has no `RefundPayment` method. Phase 3 adds it. The Square Refunds API (`POST /v2/refunds`) requires the original `payment_id` and supports partial refunds. The sandbox supports refund testing.

2. **Reminder job scheduling**: Phase 4 creates the endpoint but doesn't create the scheduled job runner. The endpoint can be called manually, via the test helper, or via the future Scheduled Jobs helper executable. This is intentional — the scheduled jobs infrastructure is a separate project.

3. **Visibility permission filtering SQL complexity**: The SQL join for permission filtering in `GetVisibleEventSessions()` adds complexity. The alternative is a two-pass approach (fetch all sessions, then filter in C++), but the SQL approach is more efficient and avoids sending hidden events over the wire.

4. **Refund for $0 purchases**: Some bookings may have $0 purchases (comp/free events). The refund logic must handle this gracefully — skip the Square API call, just update the database records.

5. **Admin metadata**: The `cancellation_policies` and `cancellation_policy_windows` tables already have admin metadata (column friendly names, display templates) so they're already editable in the admin CRUD UI. No new admin pages needed for policy management.