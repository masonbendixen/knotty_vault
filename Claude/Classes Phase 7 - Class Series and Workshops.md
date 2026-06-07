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

> **Access redesign note (2026-05-31, [[Permission-based class access redesign]] §4.6):** This phase's per-permission *pricing* path (M-2/M-5 via `product_prices`, lowest-tier-wins) is **unaffected** by the permission-based access redesign — paid offerings keep `product_prices`. The redesign changed only recurring-class *inclusion* (now requirement-group-driven; the `is_membership_included` flag was removed). Workshop/series pricing still resolves through `CatalogHelper::ResolveBestPriceForPerson`, now **closure-aware** over the tier hierarchy — a higher membership tier automatically qualifies for a lower tier's price without enumerating tiers.

## Phase Summary

**Should-have.** Admin creates a "class series" — a `class_schedule` with `is_series=true` covering a start/end date window, optional min attendees with a min-by date, optional auto-cancel-and-refund policy if min not met. Series purchases are one transaction (one `purchase`) covering all child instances. Workshops are a series of length 1 (per P-3). Users can buy a full series or join mid-series with pro-rated pricing using the per-instance base price for their tier. Series cancellation by admin issues full refunds; user-side cancel is non-refundable per P-6 but staff can grant a voucher.

**Prerequisites:**
- Phase 1 (the three-level `classes` → `class_instances` → `class_schedules` → `class_schedule_slots` model + lazy derivation). NOTE: `is_series` / `series_*` were REMOVED from `class_schedules` in the redesign — see the impact callout.
- Phase 2 (visible class sessions surface class metadata + per-user pricing).
- Phase 4 (iCal extensions — cancellation `.ics` with STATUS:CANCELLED).
- Existing `BookingHelper`, `RefundHelper`, voucher infrastructure ([[Vouchers and Refunds]]).
- Existing scheduled jobs daemon ([[Scheduled Jobs]]).

### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]] §1.5a
This phase was written against "a series is a `class_schedule` with `is_series=true`". The redesign **replaces that entirely.** The new model:
- **A series is a `classes` row of `kind='series'`** (shared marketing identity — name / description / photo) **with one `class_instances` row per run** ("Fall 2026", "Spring 2027"). The instance carries the run's `valid_from_us`/`valid_to_us`, its `product_id` (`kind='class_series'`), and the schedule lives in the instance's `class_schedules` impl(s) + slots.
- **`class_series_instances` (NEW table, this phase)** is a 1:1 augmentation of `class_instances` carrying the series-bundle fields that used to be planned on `class_schedules`: `min_attendees`, `min_by_us`, `min_not_met_policy`, `prorated_signups_allowed`. (Per L-5 / OQ-CSI-16. The `is_series` / `series_*` columns no longer exist on `class_schedules` — §2.1 below is rewritten.)
- **Holiday overrides during a run** (Mason's Labor Day Monday) are higher-priority empty/edited impls under the same instance — they reduce the derived-occurrence count, which feeds pricing.
- **Pricing = (count of derived occurrences in the instance's window) × (per-tier `per_instance_base`).** The "full series total" is computed, not stored separately — though the `price_kind` split in §2.2 is still useful for an explicit per-instance base vs an optional flat override. Derive occurrences via `ClassScheduleHelper::GetDerivedSessionsForRange` over the instance window.
- **A series purchase eagerly ensures `event_sessions` rows** for every derived occurrence (paid bookings can't be lazy — real money) via `ClassScheduleHelper::EnsureSessionExists`, setting `series_purchase_id` on each. The min-attendees auto-cancel job operates on those persisted rows.
- **Workshops** are the same shape, simpler: `classes.kind='workshop'` + one `class_instances` per run + `kind='workshop'` product + a single-slot bounded impl, NO `class_series_instances` augmentation.
Throughout this doc, read "series `class_schedule` / `scheduleId`" as "series `class_instances` / `classInstanceId`", and "materialize series sessions" as "ensure occurrence rows at purchase". The §4 helper signatures are reframed accordingly below.

**Outcome:**
- Admin creates a series run as a `class_instances` row (under a `kind='series'` class) + its `class_series_instances` augmentation + base impl; this phase wires the booking + purchase model on top.
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
- [x] Workshop = `classes.kind='workshop'` + one `class_instances` per run + a single-slot bounded impl (P-3, redesign). No `class_series_instances` augmentation.
- [x] Series = `classes.kind='series'` + one `class_instances` per run + a `class_series_instances` augmentation + base/override impls under the instance.
- [x] Series price flows through existing `product_prices` per-tier (P-2) — no parallel pricing system; total = per-instance base × derived-occurrence count.
- [x] Pro-rated formula = `(per_instance_base_price[tier]) × remaining_sessions` (resolved per parent doc — was OQ-7).
- [x] User cannot cancel a series themselves; staff grants voucher case-by-case (resolved per parent doc — was OQ-8).
- [x] Admin-decides path: `admin_alerts` digest + email to `admin_alert_recipients` (resolved — was OQ-9).

### 1.2 Per-instance vs whole-series purchase
- [x] One `purchase` row covers the entire series buy. `bookings` for each child session reference the same `purchase_id`. This keeps refund accounting clean.
- [x] Pro-rated mid-series buys also produce one purchase, but only for the remaining instances. New `purchase_id` per buyer per join.

### 1.3 Per-instance base price source
- [ ] Series products have TWO prices per tier: a "full series" price and a "per-instance base" price. Store both in `product_prices` and disambiguate via a new `price_kind` column (`'series_total'` vs `'per_instance_base'`). Pro-rating uses `per_instance_base × remaining`. Document this in 2.2.

## 2. Database Schema

### 2.1 New `class_series_instances` table (augments `class_instances` 1:1)
- [ ] `db_schema/class_series_instances.h/.cpp`:
  - `id BIGSERIAL PK`
  - `class_instance_id BIGINT NOT NULL UNIQUE REFERENCES class_instances(id)` — 1:1 augmentation
  - `min_attendees BIGINT` NULL
  - `min_by_us BIGINT` NULL
  - `min_not_met_policy TEXT` NULL (`'auto_cancel_refund' | 'proceed' | 'admin_decides'`, CHECK at app layer)
  - `prorated_signups_allowed BOOLEAN NOT NULL DEFAULT FALSE`
  - `created_us`, `updated_us`
- [ ] The run's window (`valid_from_us`/`valid_to_us`) and product live on the parent `class_instances` row (Phase 1) — NOT duplicated here. `is_series` / `series_*` no longer exist on `class_schedules`.
- [ ] Index on (`class_instance_id`).

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

- [ ] `struct CreateSeriesInstanceRequest { ... }` — the `class_id` (an existing `kind='series'` class), the run window, the `kind='class_series'` product, the base impl + slots, and the `class_series_instances` fields (min_attendees, min_by_us, min_not_met_policy, prorated_signups_allowed).
- [ ] `struct CreateSeriesInstanceResult { int64_t classInstanceId; int64_t productId; std::string errorCode; }`. (No `sessionIds` — occurrences are derived, not pre-created.)
- [ ] `CreateSeriesInstance(Transaction&, const CreateSeriesInstanceRequest&)`:
  1. Create the `class_instances` row (delegate to `ClassInstanceHelper::CreateInstance`) + the `class_series_instances` augmentation row.
  2. Create the base impl + slots (delegate to `ClassScheduleHelper::CreateImplementation` + slot adds) under the instance.
  3. Ensure the product has a `per_instance_base` price for each allowed tier — else return `MISSING_TIER_PRICING`. (No materialization — occurrences derive.)
  4. Return.

- [ ] `BookFullSeries(Transaction&, personId, classInstanceId)`:
  0. **Reject duplicate enrollment** (resolved OQ-P7-3): if the person already has an active (`status='confirmed'`) booking for ANY occurrence of this run, return `ALREADY_BOOKED`. This blocks a full-series buy after the user already joined via the prorated path mid-week; admin resolves the overlap manually.
  1. Derive the run's occurrences via `ClassScheduleHelper::GetDerivedSessionsForRange(classId, instance.valid_from_us, instance.valid_to_us)` → `occurrences`.
  2. Resolve `per_instance_base` for the user's best tier → `perInstanceCents`; `totalCents = perInstanceCents × occurrences.size()`.
  3. Create a `purchase` row + one `purchase_item` for the series product.
  4. Process payment (delegate to the existing checkout flow — Phase 7 doesn't reinvent payment).
  5. After payment success: for each occurrence, `ClassScheduleHelper::EnsureSessionExists(slotId, occurrenceDateUs)` (eager), set `event_sessions.series_purchase_id`, create a `booking` with `status='confirmed'`, `purchase_id` set, `bookings.notes='Series:<classInstanceId>'`.
  6. Send a confirmation email with a multi-VEVENT iCal (one VEVENT per occurrence).
  7. Return `{ok, purchaseId, bookingIds}`.

- [ ] `BookProratedRemainingSeries(Transaction&, personId, classInstanceId, joinDateUs)`:
  0. **Reject duplicate enrollment** (resolved OQ-P7-3, symmetric with `BookFullSeries`): if the person already has an active booking for any occurrence of this run, return `ALREADY_BOOKED`.
  1. Derive occurrences with `occurrence start > joinDateUs` → `remaining`.
  2. Resolve `per_instance_base` for the user's best tier → `perInstanceCents`; `totalCents = perInstanceCents × remaining.size()`.
  3. Same purchase + payment + ensure-occurrence + per-occurrence booking creation as `BookFullSeries`, restricted to `remaining`.
  4. Return.

- [ ] `CancelSeriesInstance(Transaction&, classInstanceId, adminPersonId, reason)`:
  1. Find all persisted `event_sessions` for this run via `series_purchase_id` (use `GetSessionsForSeriesPurchase` for each purchase tied to the instance) — these are the ensured paid occurrences.
  2. For each: call `SessionCancellationHelper::CancelSession` (refund + cancellation email per Phase 2 rules: paid bookings get full refund, zero-money bookings get capacity release only).
  3. Mark `class_instances.is_active=false` for the run.
  4. Send a "series cancelled" admin alert if any attendees had been booked.

- [ ] `CheckMinAttendees(Transaction&, classInstanceId, asOfUs)`:
  1. Load the `class_series_instances` augmentation. Skip if `min_attendees IS NULL` OR `asOfUs < min_by_us`.
  2. Count distinct `bookings.person_id` across the run's purchased occurrences with `status='confirmed'`.
  3. If count < `min_attendees`:
     - `auto_cancel_refund` → call `CancelSeriesInstance(classInstanceId, systemAdminId, "Minimum attendance not met")`.
     - `proceed` → no-op.
     - `admin_decides` → write an `admin_alerts` row + queue an email to `admin_alert_recipients`. Mark the instance "min-not-met-pending" so we don't alert again.

### 4.2 Refund integration (existing `RefundHelper`)
- [ ] Verify Phase 2's session-cancellation flow already issues correct refunds for series bookings. Add a regression test that cancelling one session of a series-bound purchase refunds 1/N of the original purchase total (where N = total series instances at purchase time, NOT remaining today).
- [ ] **Decision (resolved OQ-P7-2): ship partial-refund-per-`purchase_item` in Phase 7.** The cancel-one-session-of-a-series case requires it, so we do NOT defer to Phase 12+. If the existing `RefundHelper` only refunds at purchase-level granularity, extend it to support partial refunds keyed on `purchase_item` (refund 1/N of the line total per cancelled session, N = total series instances at purchase time). Add tests for the per-item partial-refund path.

### 4.3 KeyValueTable conversions
- [ ] `SeriesInfoToKeyValueTable(...)` — combines schedule + sessions count + per-tier pricing summary for the catalog card.
- [ ] `BookSeriesResultToKeyValueTable(...)`.

## 5. Endpoints

- [ ] `POST /api/admin/class_series_instance` — creates a series run (`class_instances` + `class_series_instances` augmentation + base impl) under a `kind='series'` class. Permission `manage_class_schedule`. Endpoint test.
- [ ] `POST /api/book_class_series/<classInstanceId>` body `{ join_date_us?: int64 }` — server decides full vs prorated based on `join_date_us` vs the instance's `valid_from_us`. Returns purchase + bookings. Endpoint test (full + prorated, payment success + failure).
- [ ] `POST /api/admin/series/<classInstanceId>/check_min_attendees` — manual run, also invoked by scheduler. Permission `manage_class_schedule`. Endpoint test.
- [ ] `POST /api/admin/series/<classInstanceId>/cancel` body `{ reason }` — explicit admin-cancel endpoint. Permission `manage_class_schedule`. Endpoint test (verifies refunds queued for each paid attendee + cancellation iCal with `STATUS:CANCELLED`).

## 6. Scheduled Jobs

- [ ] Add daily 03:00 local job in `knottyyoga_helper`: iterates active series instances and POSTs `/api/admin/series/<classInstanceId>/check_min_attendees` for each. Idempotent.
- [ ] Skip instances whose `class_series_instances.min_by_us` is far in the future (saves load); cron passes a "today" date so the endpoint can short-circuit.

## 7. Frontend

### 7.1 Catalog series cards
- [ ] Class catalog cards distinguish series by the `kind='series'` badge, and list upcoming runs ("X sessions starting {date}", per-tier price) per instance.
- [ ] Spec.

### 7.2 Series detail page
- [ ] New `series-detail.component.*/.spec.ts` (or extension of class-detail when `classes.kind='series'`). For a series class, lists upcoming runs (instances); selecting a run shows its detail.
- [ ] Sections: hero photo + name; "What you get: 6 sessions Mon/Wed 7-8pm starting July 1" (derived occurrence count for the run); per-tier pricing (your tier highlighted); min-attendees warning if `class_series_instances.min_attendees` set and current count < threshold; "Buy full series" CTA / "Join from this week — prorated $X" CTA depending on date.
- [ ] BC-5 non-refundable banner.

### 7.3 Booking flow
- [ ] Series-booking flow uses existing Square Web SDK checkout path; no new payment component.
- [ ] On success: thank-you screen + "Sessions added to your calendar" with link to my-bookings.

### 7.4 My-bookings series rollup
- [ ] My-bookings page renders series purchases as a single parent row with expandable child sessions list.
- [ ] Spec.

### 7.5 Admin series-create form
- [ ] Builds on Phase 1's instance + impl editors: creating a run under a `kind='series'` class reveals the `class_series_instances` fields (min attendees + min-by date + min-not-met policy radio buttons) and a per-tier `per_instance_base` price matrix. The run window comes from the instance; the schedule from its base impl + slots. (No `is_series` toggle — the class's `kind` already determines this.)
- [ ] Spec.

### 7.6 `ServerAccess` extensions
- [ ] `createSeriesInstance(req)`, `bookSeries(classInstanceId, joinDateUs?)`, `checkSeriesMinAttendees(classInstanceId)`, `cancelSeries(classInstanceId, reason)`.
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

All three open questions are now resolved. OQ-P7-2 and OQ-P7-3 are folded into §4.2 and §4.1 respectively. OQ-P7-1 is answered below — and yes, we can (and will) show both.

- **OQ-P7-1. — RESOLVED: show both columns.**
	- **Mason asked:** "Can't we show both? Can you explain downstream reports?"
	- **What "downstream reports" means here.** "Downstream reports" are the admin/staff-facing summaries we build *on top of* the series booking data — they don't add new booking behavior, they just read and aggregate it. For series specifically there are three that care about the purchased-vs-attended distinction:
		1. **Enrollment / roster report** — per series run, who is signed up and how many session-slots each person holds. A full-series buyer holds N slots; a prorated mid-series joiner holds only their *remaining* slots (e.g. 4 of 6). This is the report the instructor prints before a session and the one the auto-cancel min-attendees job conceptually counts against.
		2. **Attendance report** — per session, who actually checked in. Driven by the attendance-tracking records (Phase 2), NOT by how many slots were purchased. Someone can hold 6 slots but attend 4.
		3. **Revenue / reconciliation report** — per series run, money collected. Keys off `purchase` / `purchase_item` totals (full-series total vs prorated sums), independent of attendance.
	- **The original concern.** Because a prorated join produces fewer `bookings` than a full-series buy, a naïve count of `bookings` conflates "how many sessions did this person pay to be in" with "how many sessions did this person show up to." If a single report column silently mixed the two, enrollment, attendance, and revenue numbers would disagree and nobody would know which is "right."
	- **Answer — show both, as two explicit, separately-labelled columns.** We do not pick one. Every series report that lists a person carries:
		- **`instances_purchased`** — count of `bookings` rows tied to that person's series `purchase` for the run (full = N at purchase time; prorated = remaining-at-join). Derived from `bookings` joined to the series `purchase_id`.
		- **`instances_attended`** — count of attendance/check-in records for that person across the run's occurrences. Derived from the Phase 2 attendance data, never from the booking count.
	- These are always rendered as distinct labelled columns (never summed into one "sessions" number), so enrollment, attendance, and revenue each read the column they actually mean. No schema change is required for this — both are aggregations over existing `bookings` + attendance tables; it's a reporting/query concern, surfaced when the series reporting views are built (a later reporting phase, not blocking Phase 7 implementation).
- **OQ-P7-2. — RESOLVED (Mason: "go with your recommendation").** Ship partial-refund-per-`purchase_item` in Phase 7. Folded into §4.2.
- **OQ-P7-3. — RESOLVED (Mason: "go with your recommendation").** `BookFullSeries` (and, symmetrically, `BookProratedRemainingSeries`) reject with `ALREADY_BOOKED` when the user already holds an active booking for any occurrence of the run; admin resolves manually. Folded into §4.1.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 7.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 4 - iCal Generator Extensions]].
- Payment context: [[Payment Design Document]], [[Purchase creation with server-side pricing]], [[Vouchers and Refunds]].
- Scheduler: [[Scheduled Jobs]].
