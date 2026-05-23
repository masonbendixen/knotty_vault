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

Classes Phase 7 - Class Series and Workshops

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

**Should-have.** Admin creates a "class series" — a `class_schedule` with `is_series=true` covering a start/end date window, optional min attendees with a min-by date, optional auto-cancel-and-refund policy if min not met. Series purchases are one transaction (one `purchase`) covering all child instances. Workshops are a series of length 1 (per P-3). Users can buy a full series or join mid-series with pro-rated pricing using the per-instance base price for their tier. Series cancellation by admin issues full refunds; user-side cancel is non-refundable per P-6 but staff can grant a voucher.

**Prerequisites:**
- Phase 1 (class_schedules table, including `is_series` and series_* fields).
- Phase 2 (visible_event_sessions surfaces class metadata + per-user pricing).
- Phase 4 (iCal extensions — cancellation `.ics` with STATUS:CANCELLED).
- Existing `BookingHelper`, `RefundHelper`, voucher infrastructure ([[Vouchers and Refunds]]).
- Existing scheduled jobs daemon ([[Scheduled Jobs]]).

**Outcome:**
- Admin can mark a `class_schedule` as a series at creation time (Phase 1 already lets them; this phase wires the booking + purchase model).
- Users see series cards in the catalog with start/end date, # of instances, per-tier price, min/max.
- Booking endpoint handles full-series buy + mid-series pro-rated buy.
- Cancel-series admin path cancels every child instance, issues refunds, emails attendees.
- Auto-cancel daily job runs `CheckMinAttendees` for each active series and triggers cancel + refund if under the threshold past the deadline (when policy = auto_cancel_refund).

## Layering & Conventions

Lowest layer first:

1. `db_schema/` — small additions to `event_sessions` and `products` for series tagging.
2. `sql_util/table_helpers/` — extensions.
3. `business_logic/scheduling/` — `ClassSeriesHelper`.
4. `business_logic/payment/` — refund + voucher integration.
5. `endpoints/` — three new endpoints.
6. Scheduled jobs in `knottyyoga_helper`.
7. Angular UI: catalog series cards, series detail, my-bookings series rollup, admin series-create form extension.
8. Admin metadata.
9. Tests.

## 1. Pre-Coding Design Decisions

### 1.1 Locked-in
- [x] Workshop = `class_schedule` with `is_series=true` and one materialized session (P-3).
- [x] Series price flows through existing `product_prices` per-tier (P-2) — no parallel pricing system.
- [x] Pro-rated formula = `(per_instance_base_price[tier]) × remaining_sessions` (resolved per parent doc — was OQ-7).
- [x] User cannot cancel a series themselves; staff grants voucher case-by-case (resolved per parent doc — was OQ-8).
- [x] Admin-decides path: `admin_alerts` digest + email to `admin_alert_recipients` (resolved — was OQ-9).

### 1.2 Per-instance vs whole-series purchase
- [x] One `purchase` row covers the entire series buy. `bookings` for each child session reference the same `purchase_id`. This keeps refund accounting clean.
- [x] Pro-rated mid-series buys also produce one purchase, but only for the remaining instances. New `purchase_id` per buyer per join.

### 1.3 Per-instance base price source
- [ ] Series products have TWO prices per tier: a "full series" price and a "per-instance base" price. Store both in `product_prices` and disambiguate via a new `price_kind` column (`'series_total'` vs `'per_instance_base'`). Pro-rating uses `per_instance_base × remaining`. Document this in 2.2.

## 2. Database Schema

### 2.1 Reuse Phase 1 columns
- [ ] Already present on `class_schedules`: `is_series`, `series_start_date_us`, `series_end_date_us`, `series_min_attendees`, `series_min_by_us`, `series_min_not_met_policy`. No new schema work on this table.

### 2.2 New `product_prices.price_kind` column
- [ ] Add `price_kind TEXT NOT NULL DEFAULT 'standard' CHECK (price_kind IN ('standard','series_total','per_instance_base'))`.
- [ ] Standard products keep `price_kind='standard'`. Class-series products have two rows per tier — one `series_total`, one `per_instance_base`.
- [ ] Update `CatalogHelper::ResolveBestPriceForPerson` to accept a `priceKind` arg (default `'standard'`).

### 2.3 `event_sessions.series_purchase_id`
- [ ] Add `series_purchase_id BIGINT` NULL `REFERENCES purchases(id)` — marks instances tied to a paid series purchase. Used by the cancellation path to find sibling sessions.
- [ ] Index on `series_purchase_id`.

### 2.4 Wire into DB init
- [ ] `make_database_info.cpp` + `create_database.cpp` updates.

## 3. Table Helpers

### 3.1 Extend `TableHelpers::ProductPrices`
- [ ] Surface `price_kind` in reads + writes.
- [ ] `GetSeriesPricesForProduct(Transaction&, productId, asOfUs)` → returns the (tier, series_total, per_instance_base) tuples.
- [ ] Tests.

### 3.2 Extend `TableHelpers::EventSessions`
- [ ] Surface `series_purchase_id` in reads.
- [ ] `GetSessionsForSeriesPurchase(Transaction&, purchaseId)` — used by cancel-series path.
- [ ] Tests.

## 4. Business Logic

### 4.1 `ClassSeriesHelper` (`business_logic/scheduling/`)
Files: `class_series_helper.h/.cpp/_test.cpp`.

- [ ] `struct CreateSeriesRequest { ... }` — wraps Phase 1's `CreateClassScheduleRequest` plus the series-specific product (with two price kinds per tier).
- [ ] `struct CreateSeriesResult { int64_t scheduleId; int64_t productId; std::vector<int64_t> sessionIds; std::string errorCode; }`.
- [ ] `CreateSeries(Transaction&, const CreateSeriesRequest&)`:
  1. Validate (delegate to `ClassScheduleHelper::CreateClassSchedule` with `isSeries=true`).
  2. Ensure the linked product has rows of both `price_kind='series_total'` and `price_kind='per_instance_base'` for each allowed tier — else return `MISSING_TIER_PRICING`.
  3. Materialize ALL series sessions atomically (one round-trip to `MaterializeFutureSessions(throughDate=series_end_date_us)`).
  4. Return.

- [ ] `BookFullSeries(Transaction&, personId, scheduleId)`:
  1. Look up product + resolve `series_total` price for the user's best tier.
  2. Create a `purchase` row + one `purchase_item` for the series product.
  3. Process payment (delegate to `PaymentHelper::PayWithCard` or whatever the existing checkout flow is — Phase 7 doesn't reinvent payment).
  4. After payment success: iterate the series's `event_sessions`, create a `booking` per session with `status='confirmed'`, `purchase_id` set, `bookings.notes='Series:<scheduleId>'`. Set `event_sessions.series_purchase_id` on each.
  5. Send a confirmation email with a multi-VEVENT iCal (one VEVENT per child session).
  6. Return `{ok, purchaseId, bookingIds}`.

- [ ] `BookProratedRemainingSeries(Transaction&, personId, scheduleId, joinDateUs)`:
  1. Count `event_sessions` for this series with `start_time_us > joinDateUs AND status='scheduled'` → `remainingCount`.
  2. Resolve `per_instance_base` price for the user's best tier → `perInstanceCents`.
  3. Compute `totalCents = perInstanceCents * remainingCount`.
  4. Same purchase + payment + per-instance booking creation as `BookFullSeries`, restricted to the remaining sessions.
  5. Return.

- [ ] `CancelSeries(Transaction&, scheduleId, adminPersonId, reason)`:
  1. Find all `event_sessions` with `class_schedule_id=scheduleId AND status='scheduled'`.
  2. For each: call `SessionCancellationHelper::CancelSession` (which already does refund + cancellation email per Phase 2's refund-pro-rating rules). Phase 2 already established: paid bookings get full refund, zero-money bookings get capacity release only.
  3. Mark the `class_schedule.is_active=false`.
  4. Send a "series cancelled" admin alert via the existing admin alerts infrastructure if any attendees had been booked.

- [ ] `CheckMinAttendees(Transaction&, scheduleId, asOfUs)`:
  1. Load `class_schedule`. Skip if `series_min_attendees IS NULL` OR `asOfUs < series_min_by_us`.
  2. Count distinct `bookings.person_id` for the series's first instance with `status='confirmed'`.
  3. If count < `series_min_attendees`:
     - If `series_min_not_met_policy='auto_cancel_refund'` → call `CancelSeries(scheduleId, systemAdminId, "Minimum attendance not met")`.
     - If `series_min_not_met_policy='proceed'` → no-op.
     - If `series_min_not_met_policy='admin_decides'` → write an `admin_alerts` row + queue an email to `admin_alert_recipients`. Mark the schedule as "min-not-met-pending" so we don't alert again for the same series.

### 4.2 Refund integration (existing `RefundHelper`)
- [ ] Verify Phase 2's session-cancellation flow already issues correct refunds for series bookings. Add a regression test that cancelling one session of a series-bound purchase refunds 1/N of the original purchase total (where N = total series instances at purchase time, NOT remaining today).
- [ ] If the existing logic only refunds at purchase-level granularity, extend `RefundHelper` to support partial refunds per `purchase_item`. Otherwise leave as full-series refund on `CancelSeries`.

### 4.3 KeyValueTable conversions
- [ ] `SeriesInfoToKeyValueTable(...)` — combines schedule + sessions count + per-tier pricing summary for the catalog card.
- [ ] `BookSeriesResultToKeyValueTable(...)`.

## 5. Endpoints

- [ ] `POST /api/admin/class_series` — extends Phase 1's `class_schedule` create with series-specific fields. Permission `manage_class_schedule`. Endpoint test.
- [ ] `POST /api/book_class_series/<scheduleId>` body `{ join_date_us?: int64 }` — server decides full vs prorated based on `join_date_us` vs `series_start_date_us`. Returns purchase + bookings. Endpoint test (full + prorated, payment success + failure).
- [ ] `POST /api/admin/series/<scheduleId>/check_min_attendees` — manual run, also invoked by scheduler. Permission `manage_class_schedule`. Endpoint test.
- [ ] `POST /api/admin/series/<scheduleId>/cancel` body `{ reason }` — explicit admin-cancel endpoint. Permission `manage_class_schedule`. Endpoint test (verifies refunds queued for each paid attendee + cancellation iCal with `STATUS:CANCELLED`).

## 6. Scheduled Jobs

- [ ] Add daily 03:00 local job in `knottyyoga_helper`: iterates active series schedules and POSTs `/api/admin/series/<id>/check_min_attendees` for each. Idempotent.
- [ ] Skip series whose `series_min_by_us` is far in the future (saves load); cron passes a "today" date so the endpoint can short-circuit.

## 7. Frontend

### 7.1 Catalog series cards
- [ ] Class catalog cards distinguish series by an `Is Series` badge, show "X sessions starting <date>", per-tier price.
- [ ] Spec.

### 7.2 Series detail page
- [ ] New `series-detail.component.*/.spec.ts` (or extension of class-detail when `is_series`).
- [ ] Sections: hero photo + name; "What you get: 6 sessions Mon/Wed 7-8pm starting July 1"; per-tier pricing (your tier highlighted); min-attendees warning if `series_min_attendees` set and current count < threshold; "Buy full series" CTA / "Join from this week — prorated $X" CTA depending on date.
- [ ] BC-5 non-refundable banner.

### 7.3 Booking flow
- [ ] Series-booking flow uses existing Square Web SDK checkout path; no new payment component.
- [ ] On success: thank-you screen + "Sessions added to your calendar" with link to my-bookings.

### 7.4 My-bookings series rollup
- [ ] My-bookings page renders series purchases as a single parent row with expandable child sessions list.
- [ ] Spec.

### 7.5 Admin series-create form
- [ ] Extends Phase 1's class-schedule-edit form: when `is_series` toggled, reveal series-specific fields (start/end date, optional min attendees + min-by date + min-not-met policy radio buttons) and a tier-pricing matrix with two columns per tier (full-series price + per-instance-base price).
- [ ] Spec.

### 7.6 `ServerAccess` extensions
- [ ] `createSeries(req)`, `bookSeries(scheduleId, joinDateUs?)`, `checkSeriesMinAttendees(scheduleId)`, `cancelSeries(scheduleId, reason)`.
- [ ] Update `ServerAccess.mock.spec.ts`.

### 7.7 Types
- [ ] `series.types.ts`: `SeriesSummary`, `SeriesDetail`, `SeriesPricing`, `BookSeriesResult`.

## 8. Admin Metadata

- [ ] No new top-level tables (uses Phase 1's `class_schedules`).
- [ ] `product_prices.price_kind` shows up in the admin column data info with friendly name "Price Kind" and a dropdown editor for the three values.

## 9. Tests-Required Summary

- [ ] Table helper tests for `price_kind` round-trip + `GetSeriesPricesForProduct`.
- [ ] `class_series_helper_test.cpp` covering all five methods + the three `series_min_not_met_policy` branches.
- [ ] Endpoint tests for all four new endpoints.
- [ ] Mail helper test: cancellation email queued for each attendee on `CancelSeries`.
- [ ] Frontend specs for series-detail, my-bookings series rollup, admin series-create form, mock service.
- [ ] Manual-testing-helper commands: `create_series <...>`, `book_series <person_id> <schedule_id>`, `simulate_series_under_min <schedule_id>`.

## 10. Cross-Layer Acceptance Criteria

A gold member buys a 6-week aerial series the day before it starts:
- [ ] One `purchase` row + one `purchase_item` at gold-tier full-series price.
- [ ] Six `bookings` rows, all tied to the same `purchase_id`.
- [ ] One confirmation email with a multi-VEVENT `.ics` (6 events).
- [ ] Each `event_sessions.series_purchase_id` is set.

A silver member joins after week 2 has happened:
- [ ] Server computes 4 remaining sessions × silver per-instance-base price.
- [ ] Single new `purchase` for that amount; 4 `bookings`.

Admin cancels the series at week 3 due to low enrollment:
- [ ] All future sessions go to `status='cancelled'`.
- [ ] Each attendee with a paid booking gets a refund queued (per the existing refund flow); each receives a cancellation iCal with `STATUS:CANCELLED`.

Daily auto-cancel job runs at 03:00 local, finds a series past `series_min_by_us` with policy `auto_cancel_refund`:
- [ ] Series auto-cancelled, refunds issued, attendees emailed.

## 11. Open Questions

- **OQ-P7-1.** When prorated mid-series booking creates fewer bookings than the original purchaser's full-series count, do downstream reports key off "instances purchased" vs "instances attended"? Define both columns explicitly when reporting.
- **OQ-P7-2.** If `RefundHelper` only does full-purchase refunds today, do we ship partial-refund-per-purchase-item in Phase 7 or leave that to Phase 12+? Recommended: ship in Phase 7 — the cancel-one-session-of-a-series case needs it.
- **OQ-P7-3.** Should `BookFullSeries` reject the booking if the user already has an active booking for any series instance (e.g. they came via the prorated path mid-week)? Recommended: yes, return `ALREADY_BOOKED`; admin can resolve manually.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 7.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 4 - iCal Generator Extensions]].
- Payment context: [[Payment Design Document]], [[Purchase creation with server-side pricing]], [[Vouchers and Refunds]].
- Scheduler: [[Scheduled Jobs]].
