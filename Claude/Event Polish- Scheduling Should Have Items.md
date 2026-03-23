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

- [ ] **`BookingHelper::BookEvent()`** — Add overlap check before creating the booking
  - After loading the event session's `start_time_us` and `end_time_us`:
  - Query the person's existing confirmed bookings that overlap:
    ```sql
    SELECT b.id FROM bookings b
    JOIN event_sessions es ON b.event_session_id = es.id
    WHERE b.person_id = {personId}
    AND b.status = 'confirmed'
    AND es.start_time_us < {endTimeUs}
    AND es.end_time_us > {startTimeUs}
    ```
  - If any rows returned: return error with code `"booking_conflict"` and message "You already have a booking that overlaps with this event."
  - This check goes after permission checks but before purchase creation
- [ ] **Tests** in `booking_helper_test.cpp`:
  - Non-overlapping bookings: allowed
  - Overlapping bookings: rejected with `booking_conflict`
  - Cancelled overlapping booking: allowed (only checks `confirmed` status)
  - Waitlisted overlapping booking: decision — should waitlisted count? Probably yes, since the user already paid. Check `status IN ('confirmed', 'waitlisted')`.

### Frontend — Event Booking Page

- [ ] **`event-booking.component.ts`** — Handle the `booking_conflict` error code. Display: "You already have a booking that overlaps with this event time."
- [ ] **Component spec test** for booking conflict error handling

---

## Phase 3: Cancellation Policy Configuration & Refund Flow (Scenario 15)

**Goal**: Admin creates cancellation policies with tiered refund windows. When a user cancels a booking, the system calculates the refund percentage based on how far in advance they're cancelling. Refunds are recorded in the database. Square refund API integration is added.

### Backend — Table Helpers Layer

- [ ] **Create `cancellation_policies.h/cpp`** table helper in `sql_util/table_helpers/`
  - `AddCancellationPolicy(Transaction&, name, description)` → `int64_t`
  - `GetCancellationPolicy(Transaction&, int64_t id)` → `KeyValueTable`
  - `GetCancellationPolicies(Transaction&)` → `KeyValueTableArray`
  - `UpdateCancellationPolicy(Transaction&, int64_t id, const KeyValueTable& updates)`
  - `DeleteCancellationPolicy(Transaction&, int64_t id)`
- [ ] **Create `cancellation_policy_windows.h/cpp`** table helper in `sql_util/table_helpers/`
  - `AddWindow(Transaction&, int64_t policyId, int64_t hoursBefore, int64_t refundPercent)` → `int64_t`
  - `GetWindowsForPolicy(Transaction&, int64_t policyId)` → `KeyValueTableArray` (ordered by `hours_before DESC`)
  - `UpdateWindow(Transaction&, int64_t id, const KeyValueTable& updates)`
  - `DeleteWindow(Transaction&, int64_t id)`
- [ ] **Tests** for both table helpers
- [ ] **CMakeLists.txt** — Add new files

### Backend — Square Client Layer

- [ ] **Add `RefundPayment()` to `SquareClient`** virtual interface
  ```cpp
  struct RefundResult {
      std::string refundId;
      std::string status;
      int64_t amountCents = 0;
      std::string currency;
      std::string rawJson;
  };

  virtual RefundResult RefundPayment(
      const std::string& paymentId,
      int64_t amountCents,
      const std::string& currency,
      const std::string& idempotencyKey,
      const std::string& reason = "",
      const RetryPolicy& retryPolicy = RetryPolicy::Default()) = 0;
  ```
  - Calls `POST /v2/refunds` on the Square API
  - Requires `payment_id`, `amount_money`, `idempotency_key`, optional `reason`
- [ ] **Implement in `square_client.cpp`** (production implementation)
- [ ] **Add to test SquareClient** mock
- [ ] **Tests** in `square_client_test.cpp` for the new method

### Backend — Business Logic Layer (Payment)

- [ ] **Create `RefundHelper` class** in `business_logic/payment/`
  - `CalculateRefundPercent(Transaction&, int64_t cancellationPolicyId, int64_t eventStartTimeUs)` → `int64_t` (0-100)
    - Loads the policy's windows ordered by `hours_before DESC`
    - Calculates hours remaining = `(eventStartTimeUs - now_us()) / 3600000000`
    - Finds the first window where `hours_remaining >= hours_before`
    - Returns that window's `refund_percent`
    - If no window matches (too close to event): returns 0
    - If no policy assigned: returns 100 (full refund — default generous behavior)
  - `ProcessRefund(Transaction&, int64_t purchaseId, int64_t refundPercent)` → `RefundInfo`
    - Loads the purchase and its payment
    - Calculates refund amount = `payment.amount_cents * refundPercent / 100`
    - If `refundPercent == 0`: no refund, return info with zero amount
    - If purchase has no payment (e.g., $0 comp): skip Square call, just record
    - Calls `SquareClient::RefundPayment()` with the original payment's Square payment ID
    - Records a new payment row with negative amount and `refund_for_payment_id` set
    - Updates purchase status to `refunded` or `partially_refunded`
    - Returns refund details
- [ ] **Tests** in `refund_helper_test.cpp`:
  - 100% refund within full-refund window
  - 50% partial refund within partial window
  - 0% refund past all windows
  - No cancellation policy → 100% refund
  - $0 purchase → no Square call
- [ ] **CMakeLists.txt** — Add new files

### Backend — Business Logic Layer (Scheduling)

- [ ] **Update `BookingHelper::CancelBooking()`** — Integrate refund calculation
  - After loading the booking, load the event session and product
  - If product has `cancellation_policy_id`:
    - Call `RefundHelper::CalculateRefundPercent()` to get the refund percentage
    - Call `RefundHelper::ProcessRefund()` to issue the refund
  - Else (no policy): full refund via `RefundHelper::ProcessRefund(100%)`
  - Add `refundAmountCents`, `refundPercent`, `currency` to `CancelBookingResult`
  - For **waitlisted** bookings: always 100% refund (they never got in)
- [ ] **Update `CancelBookingResult`** struct — Add refund fields:
  ```cpp
  int64_t refundAmountCents = 0;
  int64_t refundPercent = 0;
  std::string currency;
  ```
- [ ] **Tests** in `booking_helper_test.cpp` — Cancel with refund policy, cancel without policy, cancel waitlisted (full refund)

### Backend — Endpoint Layer

- [ ] **Update `cancel_booking.cpp`** — Include refund info in the JSON response
  - Add `refund_amount_cents`, `refund_percent`, `currency` to the response JSON
- [ ] **Tests** in cancel_booking endpoint test (if exists)

### Frontend — Types

- [ ] **`scheduling.types.ts`** — Add refund fields to cancel booking response type:
  ```typescript
  refund_amount_cents?: number;
  refund_percent?: number;
  currency?: string;
  ```

### Frontend — My Events Page

- [ ] **`my-events.component.ts/html`** — Show refund info after successful cancellation
  - After cancel, display: "Booking cancelled. Refund of $X.XX (Y%) will be processed."
  - If 0% refund: "Booking cancelled. No refund — cancellation policy window has passed."

### Frontend — Event Booking Page

- [ ] **`event-booking.component.html`** — Show cancellation policy info on the booking page
  - Below the price, show: "Free cancellation up to 48 hours before" or the relevant policy windows
  - This requires the endpoint to return the product's cancellation policy info (add to `ResolvedEventSession`)

### Frontend — Tests

- [ ] **Component spec tests** for my-events: refund display after cancellation
- [ ] **Component spec tests** for event-booking: cancellation policy info display

### Admin — Cancellation Policy Management

- [ ] The `cancellation_policies` and `cancellation_policy_windows` tables are already exposed through the admin CRUD UI (they have admin metadata). Admin can create/edit policies and windows through the existing table editor. **No new admin UI needed** — the existing admin table UI handles this.
- [ ] To assign a policy to a product: Admin edits the product and sets `cancellation_policy_id` via the existing admin CRUD UI (which shows FK pickers for `cancellation_policy_id`). **No new UI needed.**

---

## Phase 4: Event Reminder Emails (Scenario 13)

**Goal**: Users receive a reminder email N hours before their event. A scheduled admin endpoint runs periodically to send unsent reminders.

### Backend — DB Schema Layer

- [ ] **Add `reminder_sent_us` column to `bookings` table**
  - New column in `db_schema/bookings.h`: `kBookingsReminderSentUs = "reminder_sent_us"`
  - Add to table DDL in `create_database.cpp`: `BIGINT DEFAULT NULL`
  - Purpose: Track which bookings have had reminders sent (prevents duplicate sends)

### Backend — Table Helpers Layer

- [ ] **Update `Bookings` table helper** — No new methods needed; existing `UpdateBooking()` can set `reminder_sent_us`

### Backend — Business Logic Layer

- [ ] **Create `EventReminderMail` email template** in `business_logic/scheduling/`
  - `event_reminder_mail.h/cpp`
  - Struct: `EventReminderData { firstName, email, eventName, eventDate, eventTime, facilityName, locationRoomName }`
  - Template: Friendly reminder email with event details, similar style to booking confirmation but with "Reminder: Your upcoming event" header
  - Use `FormatString()` + `NormalizeCrLf()` pattern
- [ ] **Create `EventReminderHelper` class** in `business_logic/scheduling/`
  - `event_reminder_helper.h/cpp`
  - `SendPendingReminders(Transaction&, DatabaseHelper&, MailHelperPtr&)` → `ReminderResult { int64_t sent, int64_t skipped }`
  - Logic:
    1. Load products' `reminder_hours` (default to a configurable secret, e.g., 24 hours)
    2. Find confirmed bookings where:
       - `b.status = 'confirmed'`
       - `b.reminder_sent_us IS NULL`
       - `es.start_time_us - now_us() <= reminder_hours * 3600000000` (event is within reminder window)
       - `es.start_time_us > now_us()` (event hasn't passed)
    3. For each matching booking: build `EventReminderData`, send email, set `reminder_sent_us = now_us()`
- [ ] **Tests** in `event_reminder_helper_test.cpp`:
  - Sends reminder when event within window
  - Doesn't re-send if `reminder_sent_us` already set
  - Doesn't send for cancelled bookings
  - Doesn't send for past events
  - Respects per-product `reminder_hours`
- [ ] **CMakeLists.txt** — Add new files

### Backend — Endpoint Layer

- [ ] **Create `admin_send_event_reminders.h/cpp`** endpoint
  - Route: `POST /api/admin/send_event_reminders`
  - Requires authentication + admin/scheduling permission
  - Calls `EventReminderHelper::SendPendingReminders()`
  - Returns JSON: `{ "reminders_sent": N, "reminders_skipped": N }`
  - Follow the `admin_process_waitlist_refunds` pattern
- [ ] **Tests** for the endpoint

### Scheduled Jobs Integration

- [ ] **Update `Scheduled Jobs.md`** — Mark the `send_event_reminders` endpoint as implemented. This endpoint will be called by the scheduled jobs helper executable when it's built (Phase 8 of that plan). For now it can be triggered manually via the admin API or the test helper.

### Test Helper Integration

- [ ] **Add `send_event_reminders` command** to `test_helper/commands/booking_commands.cpp`
  - Calls `EventReminderHelper::SendPendingReminders()` directly
  - Prints count of reminders sent

---

## Phase 5: Admin Cancels Entire Session (Scenario 60)

**Goal**: Admin can cancel an entire event session. All confirmed attendees receive refunds (per cancellation policy, or 100% if studio-initiated). All attendees (confirmed + waitlisted) receive a cancellation notification email. All bookings are marked cancelled.

### Backend — Business Logic Layer

- [ ] **Create `SessionCancellationMail` email template** in `business_logic/scheduling/`
  - `session_cancellation_mail.h/cpp`
  - Struct: `SessionCancellationData { firstName, email, eventName, eventDate, eventTime, facilityName, cancellationReason, refundAmountCents, currency }`
  - Template: "Event Cancelled" email with reason, refund info ("A refund of $X.XX has been issued" or "No payment was collected — no refund needed" for waitlisted)
  - Red-themed header (unlike the green confirmation / orange waitlist)

- [ ] **Create `SessionCancellationHelper` class** in `business_logic/scheduling/`
  - `session_cancellation_helper.h/cpp`
  - Dependencies: `DatabaseHelper`, `SquareClientPtr`, `SecretsHelperPtr`, `MailHelperPtr`
  - `CancelSession(Transaction&, int64_t sessionId, const std::string& reason)` → `SessionCancellationResult`
  - Logic:
    1. Load event session; verify status is `scheduled`
    2. Set session status to `cancelled`, set `cancellation_reason`
    3. Load all bookings for the session (`confirmed` + `waitlisted`)
    4. For each **confirmed** booking:
       - Issue 100% refund (studio-initiated cancellation = full refund regardless of policy)
       - Set booking status to `cancelled`, set `cancelled_us` and `notes`
       - Decrement `booked_count`
       - Send cancellation email with refund info
    5. For each **waitlisted** booking:
       - Cancel the pending purchase (via `PurchaseHelper::CancelPurchase`)
       - Set booking status to `cancelled`
       - Send cancellation email (no refund info — they were waitlisted)
    6. Return result: `{ confirmed_cancelled, waitlisted_cancelled, refunds_processed, total_refunded_cents }`
  - `SessionCancellationResult` struct:
    ```cpp
    bool success = false;
    std::string errorMessage;
    int64_t confirmedCancelled = 0;
    int64_t waitlistedCancelled = 0;
    int64_t refundsProcessed = 0;
    int64_t totalRefundedCents = 0;
    ```

- [ ] **Tests** in `session_cancellation_helper_test.cpp`:
  - Cancel session with confirmed + waitlisted bookings
  - Cancel session with only confirmed bookings
  - Cancel empty session (no bookings)
  - Cancel already-cancelled session (error)
  - Verify emails sent to all attendees
  - Verify refund amounts

### Backend — Endpoint Layer

- [ ] **Create `admin_cancel_session.h/cpp`** endpoint
  - Route: `POST /api/admin/cancel_session/<int>`
  - Requires authentication + admin permission
  - Request body: `{ "reason": "..." }` (required)
  - Calls `SessionCancellationHelper::CancelSession()`
  - Returns JSON: `{ "success": true, "confirmed_cancelled": N, "waitlisted_cancelled": N, "refunds_processed": N, "total_refunded_cents": N }`
- [ ] **Tests** for the endpoint
- [ ] **CMakeLists.txt** — Add new files

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

- [ ] **Add `cancel_session` command** to `test_helper/commands/booking_commands.cpp`
  - Flags: `--session_id` (required), `--reason` (optional, default "Cancelled by admin via test helper")
  - Calls `SessionCancellationHelper::CancelSession()` directly
  - Prints results

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