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

## 2. Database Schema (DONE)

### 2.1 New `class_series_instances` table (augments `class_instances` 1:1) (DONE)
- [x] `db_schema/class_series_instances.h/.cpp`: `id` PK, `class_instance_id` (FK → class_instances, **UNIQUE** → 1:1 + index), `min_attendees`/`min_by_us` (nullable BIGINT), `min_not_met_policy` (nullable TEXT; allowed-value constants `kSeriesMinNotMetPolicy*` in the header — app-layer CHECK, matching the codebase's no-DB-CHECK convention), `prorated_signups_allowed` (BOOL NOT NULL DEFAULT FALSE), `created_us`/`updated_us`.
- [x] Run window + product live on the parent `class_instances` (not duplicated). `is_series`/`series_*` are not added to `class_schedules`.
- [x] Index on `class_instance_id` is provided by the UNIQUE constraint (no separate `CREATE INDEX` needed — same as `attendance_templates`).

### 2.2 New `product_prices.price_kind` column (DONE)
- [x] Added `price_kind TEXT NOT NULL DEFAULT 'standard'` + value constants `kProductPriceKind{Standard,SeriesTotal,PerInstanceBase}`. CHECK is enforced at the app layer (the schema builder has no CHECK support).
- [x] **Extended the unique constraint** to include `price_kind` (renamed `uq_product_prices_..._variant_kind`) so a series product can hold both a `series_total` and a `per_instance_base` row for the same (product, schedule, permission, variant) tuple. Without this the two-rows-per-tier requirement would violate the old constraint.
- [ ] `CatalogHelper::ResolveBestPriceForPerson` `priceKind` arg → deferred to §3/§4 (business-logic layer), not §2.

### 2.3 `event_sessions.series_purchase_id` (DONE)
- [x] Added nullable FK `series_purchase_id → purchases(id)`.
- [x] Partial index `event_sessions_series_purchase_idx` (WHERE series_purchase_id IS NOT NULL) added to `CreateEventSessionsIndexes`.

### 2.4 Wire into DB init (DONE)
- [x] `make_database_info.cpp` (include + `MakeClassSeriesInstancesTable` after `MakeClassInstancesTable`) and `create_database.cpp` `CreateTables()` (include + `CreateTable(kClassSeriesInstancesTable)` after `class_instances`). Verified FK creation order on **both** the test `SetupAllTables` batch (insertion order: `purchases`@173 < `event_sessions`@206; `class_instances` < `class_series_instances`) and the real `CreateTables` (`purchases`@203 < `event_sessions`@237; `class_instances`@231 < new table). `db_schema/CMakeLists.txt` updated.

## 3. Table Helpers (DONE)

### 3.1 Extend `TableHelpers::ProductPrices` (DONE)
- [x] `price_kind` surfaced in reads (the `SELECT *` getters already return it) and writes (`AddProductPrice` gained a trailing `priceKind` param, default `kProductPriceKindStandard`).
- [x] `GetSeriesPricesForProduct(tx, productId, asOfUs)` → `std::vector<SeriesTierPrice>` (new struct: `permissionId` tier + `currency` + optional `seriesTotalCents`/`perInstanceBaseCents`). Resolves the effective active price schedule at `asOfUs` (CTE: most-recent active schedule with `valid_from_us <= asOfUs` and `valid_to_us` open), then pivots the `series_total`/`per_instance_base` rows per tier with `MAX(CASE …)` (the extended unique constraint guarantees ≤1 row per (permission, kind)). Empty when no schedule is effective or the product has no series prices.
- [x] Tests: price_kind default, explicit two-rows-per-tier round-trip (validates the extended unique constraint), `GetSeriesPricesForProduct` per-tier (full + partial tier, standard price ignored, NULLS-FIRST order), empty-before-schedule / no-series-prices.
- [~] **Deferred to §4 → sidestepped:** filtering the *best-price* path (`GetBestProductPriceByProductSchedulePermissions` / `CatalogHelper::ResolveBestPriceForPerson`) by `price_kind` so a series product's cheap `per_instance_base` row isn't mistaken for a standard price. **§4 resolves series pricing through a dedicated `ClassSeriesHelper::ResolvePerInstanceCents` (over `GetSeriesPricesForProduct`), so the generic best-price path is never asked for a series product** — the `priceKind` arg on `ResolveBestPriceForPerson` was not needed and remains unbuilt. No impact today — every existing product is `'standard'`; revisit if the generic catalog ever has to render series products.

### 3.1b NEW `TableHelpers::ClassSeriesInstances` (DONE — added during §4)
- [x] §3 didn't originally list a CRUD wrapper for `class_series_instances`, but the no-SQL-in-business-logic rule requires one. Added `sql_util/table_helpers/class_series_instances.h/.cpp/_test.cpp`: `AddClassSeriesInstance`, `GetClassSeriesInstance`, `GetByClassInstanceId` (empty when none — e.g. workshops), `UpdateClassSeriesInstance` (bumps `updated_us`), `DeleteClassSeriesInstance`. Registered in `table_helpers/CMakeLists.txt`. Tests cover add/get, null policy fields, get-by-instance, the UNIQUE(class_instance_id) constraint, and update/delete.

### 3.2 Extend `TableHelpers::EventSessions` (DONE)
- [x] `series_purchase_id` surfaced in reads (`SELECT *` getters) and writable via the existing generic `UpdateEventSession` (no dedicated setter needed).
- [x] `GetSessionsForSeriesPurchase(tx, seriesPurchaseId)` → sessions ordered by `start_time_us ASC, id ASC`.
- [x] Tests: lookup returns only matching sessions in start order (unrelated purchase → empty); `series_purchase_id` is NULL by default, then surfaces in reads + the lookup after `UpdateEventSession`.

## 4. Business Logic

### 4.1 `ClassSeriesHelper` (`business_logic/scheduling/`)
Files: `class_series_helper.h/.cpp/_test.cpp`.

- [x] `struct CreateSeriesInstanceRequest { ... }` — the `class_id` (an existing `kind='series'` class), the run window, the `kind='class_series'` product, the base impl + slots, and the `class_series_instances` fields (min_attendees, min_by_us, min_not_met_policy, prorated_signups_allowed).
- [x] `struct CreateSeriesInstanceResult { int64_t classInstanceId; int64_t productId; std::string errorCode; }`. (No `sessionIds` — occurrences are derived, not pre-created.)
- [x] `CreateSeriesInstance(Transaction&, const CreateSeriesInstanceRequest&)`:
  1. Create the `class_instances` row (delegate to `ClassInstanceHelper::CreateInstance`) + the `class_series_instances` augmentation row.
  2. Create the base impl + slots (delegate to `ClassScheduleHelper::CreateImplementation` + slot adds) under the instance.
  3. Ensure the product has a `per_instance_base` price for each allowed tier — else return `MISSING_TIER_PRICING`. (No materialization — occurrences derive.)
  4. Return.

- [x] `BookFullSeries(Transaction&, personId, classInstanceId)`: *(payment processed separately by checkout, matching `BookingHelper::BookEvent`; confirmation email deferred to the payment step — see §4 implementation note)*
  0. **Reject duplicate enrollment** (resolved OQ-P7-3): if the person already has an active (`status='confirmed'`) booking for ANY occurrence of this run, return `ALREADY_BOOKED`. This blocks a full-series buy after the user already joined via the prorated path mid-week; admin resolves the overlap manually.
  1. Derive the run's occurrences via `ClassScheduleHelper::GetDerivedSessionsForRange(classId, instance.valid_from_us, instance.valid_to_us)` → `occurrences`.
  2. Resolve `per_instance_base` for the user's best tier → `perInstanceCents`; `totalCents = perInstanceCents × occurrences.size()`.
  3. Create a `purchase` row + one `purchase_item` for the series product.
  4. Process payment (delegate to the existing checkout flow — Phase 7 doesn't reinvent payment).
  5. After payment success: for each occurrence, `ClassScheduleHelper::EnsureSessionExists(slotId, occurrenceDateUs)` (eager), set `event_sessions.series_purchase_id`, create a `booking` with `status='confirmed'`, `purchase_id` set, `bookings.notes='Series:<classInstanceId>'`.
  6. Send a confirmation email with a multi-VEVENT iCal (one VEVENT per occurrence).
  7. Return `{ok, purchaseId, bookingIds}`.

- [x] `BookProratedRemainingSeries(Transaction&, personId, classInstanceId, joinDateUs)`:
  0. **Reject duplicate enrollment** (resolved OQ-P7-3, symmetric with `BookFullSeries`): if the person already has an active booking for any occurrence of this run, return `ALREADY_BOOKED`.
  1. Derive occurrences with `occurrence start > joinDateUs` → `remaining`.
  2. Resolve `per_instance_base` for the user's best tier → `perInstanceCents`; `totalCents = perInstanceCents × remaining.size()`.
  3. Same purchase + payment + ensure-occurrence + per-occurrence booking creation as `BookFullSeries`, restricted to `remaining`.
  4. Return.

- [x] `CancelSeriesInstance(Transaction&, classInstanceId, adminPersonId, reason)`: *(refunds each occurrence's 1/N share via `RefundHelper::ProcessPartialRefundCents` rather than reusing `SessionCancellationHelper::CancelSession`, which would 100%-refund the shared series purchase once per session and double-refund)*
  1. Find all persisted `event_sessions` for this run via `series_purchase_id` (use `GetSessionsForSeriesPurchase` for each purchase tied to the instance) — these are the ensured paid occurrences.
  2. For each: call `SessionCancellationHelper::CancelSession` (refund + cancellation email per Phase 2 rules: paid bookings get full refund, zero-money bookings get capacity release only).
  3. Mark `class_instances.is_active=false` for the run.
  4. Send a "series cancelled" admin alert if any attendees had been booked.

- [x] `CheckMinAttendees(Transaction&, classInstanceId, asOfUs)`: *(admin_decides writes an `admin_alerts` row feeding the existing digest; no `admin_alert_recipients` table exists yet and no "min-not-met-pending" column was added — re-alert dedup deferred to avoid premature schema churn)*
  1. Load the `class_series_instances` augmentation. Skip if `min_attendees IS NULL` OR `asOfUs < min_by_us`.
  2. Count distinct `bookings.person_id` across the run's purchased occurrences with `status='confirmed'`.
  3. If count < `min_attendees`:
     - `auto_cancel_refund` → call `CancelSeriesInstance(classInstanceId, systemAdminId, "Minimum attendance not met")`.
     - `proceed` → no-op.
     - `admin_decides` → write an `admin_alerts` row + queue an email to `admin_alert_recipients`. Mark the instance "min-not-met-pending" so we don't alert again.

### 4.2 Refund integration (existing `RefundHelper`)
- [x] Verify Phase 2's session-cancellation flow already issues correct refunds for series bookings. Add a regression test that cancelling one session of a series-bound purchase refunds 1/N of the original purchase total (where N = total series instances at purchase time, NOT remaining today). *(covered by `RefundHelperTest::PartialRefundCentsRefundsOneOfN` + `ClassSeriesHelperTest::CancelOneSessionRefundsOneOfN`)*
- [x] **Decision (resolved OQ-P7-2): ship partial-refund-per-`purchase_item` in Phase 7.** The cancel-one-session-of-a-series case requires it, so we do NOT defer to Phase 12+. If the existing `RefundHelper` only refunds at purchase-level granularity, extend it to support partial refunds keyed on `purchase_item` (refund 1/N of the line total per cancelled session, N = total series instances at purchase time). Add tests for the per-item partial-refund path. *(added `RefundHelper::ProcessPartialRefundCents` — cents-based, capped at remaining paid; refactored the shared refund-recording tail into `RecordRefund`. The series purchase carries one `purchase_item` quantity=N unit=per_instance, so per-item granularity = per-occurrence 1/N.)*

### 4.3 KeyValueTable conversions
- [x] `SeriesInfoToKeyValueTable(...)` — combines schedule + sessions count + per-tier pricing summary for the catalog card. *(added `SeriesInfo` struct + converter in `scheduling_key_value_table.h/.cpp`; the producer that fills `SeriesInfo` from the DB is catalog/detail work, §5/§7.)*
- [x] `BookSeriesResultToKeyValueTable(...)`.

## 5. Endpoints

- [x] `POST /api/admin/class_series_instance` — creates a series run (`class_instances` + `class_series_instances` augmentation + base impl) under a `kind='series'` class. Permission `manage_class_schedule`. Endpoint test. *(`admin_class_series_instance_create.h/.cpp/_test.cpp`; parses a `slots` array; tests cover 403/400-missing-field/400-missing-pricing/200+persist.)*
- [x] `POST /api/book_class_series/<classInstanceId>` body `{ join_date_us?: int64 }` — server decides full vs prorated based on `join_date_us` vs the instance's `valid_from_us`. Returns purchase + bookings. *(`book_class_series.h/.cpp/_test.cpp`; tests cover 401/full/prorated/409-already-booked/404. Payment is a separate step — `purchase_pay_card` — matching `BookEvent`, so there is no inline "payment success/failure" here; the prorated path is gated on `prorated_signups_allowed`, added to `BookProratedRemainingSeries` with error `PRORATION_NOT_ALLOWED`.)*
- [x] `POST /api/admin/series/<classInstanceId>/check_min_attendees` — manual run, also invoked by scheduler. Permission `manage_class_schedule`. Endpoint test. *(`admin_series_check_min_attendees.h/.cpp/_test.cpp`; optional `as_of_us` body (defaults to now); tests cover 403/skip/admin_alerted/auto_cancelled+deactivated/404.)*
- [x] `POST /api/admin/series/<classInstanceId>/cancel` body `{ reason }` — explicit admin-cancel endpoint. Permission `manage_class_schedule`. Endpoint test (verifies refunds queued for each paid attendee~~ + cancellation iCal with `STATUS:CANCELLED`~~). *(`admin_series_cancel.h/.cpp/_test.cpp`; tests cover 403/400-missing-reason/404/200 with a simulated payment asserting `refunds_processed == N` and the purchase fully refunded + run deactivated. Cancellation iCal is deferred with the booking/confirmation email work — see §4 note.)*

**§5 wiring:** all four registered in `web_app.cpp` (includes + reference vars) and `endpoints/CMakeLists.txt` (sources + tests). Result→JSON conversions (`CreateSeriesInstanceResult` / `CheckMinAttendeesResult` / `CancelSeriesResult` ToKeyValueTable) were added to `scheduling_key_value_table.h/.cpp` with tests (kept out of the endpoint files per the layering rule).

## 6. Scheduled Jobs

- [x] Add daily job in `knottyyoga_helper`: ~~iterates active series instances and POSTs `/api/admin/series/<classInstanceId>/check_min_attendees` for each~~. *(The scheduler is a dumb HTTP cron with no DB access — every existing job (e.g. `send_weekly_digests`) POSTs a single endpoint that fans out server-side. So §6 adds **one** job `run_series_min_attendees_check` (daily, 86400s) → `POST /api/admin/run_series_min_attendees_check`, which enumerates active runs via `ClassSeriesInstances::GetActiveSeriesClassInstanceIds` and calls `ClassSeriesHelper::RunMinAttendeesCheckForActiveRuns` → `CheckMinAttendees` per run. Idempotent. Wired in `scheduler_config.h` (`seriesMinAttendeesSeconds`), `main.cpp` (`--series_min_attendees_interval` flag + log summary), and `scheduled_job.cpp`.)* The per-instance `/api/admin/series/<id>/check_min_attendees` endpoint from §5 remains for manual admin runs. The "03:00 local" time is approximated by a daily interval (the scheduler is interval-based, like all existing jobs).
- [x] Skip instances whose `class_series_instances.min_by_us` is far in the future. *(Handled by `CheckMinAttendees` itself — it returns `checked=false` when `asOf < min_by_us`, so far-future runs take no action. The endpoint uses `now` (or an optional `as_of_us` body so the cron can pass "today"). The active-runs query already excludes ended runs (`valid_to_us <= asOf`); far-future runs are considered-but-skipped rather than query-filtered, matching the self-gating pattern of the weekly/instructor digests.)*

**§6 tests:** table-helper test for `GetActiveSeriesClassInstanceIds` (active+ongoing only); `ClassSeriesHelper` sweep tests (due-vs-future skip across two classes; auto-cancel path); `RunMinAttendeesSummaryToKeyValueTable` converter test; endpoint test (403 + 200 sweep); scheduler `scheduled_job_test` updated (job count 14→15, disable/propagate cases for the new job).

## 7. Frontend

> **§7 backend prerequisite (built this pass):** the series detail UI needs a *reader* for per-run data (occurrence count + viewer-resolved per-instance/total price + min policy), which §4.3 deferred ("producer is §5/§7 work"). Added `ClassSeriesHelper::GetUpcomingSeriesRuns(tx, classId, personId, asOfUs)` → `vector<SeriesInfo>` + `SeriesInfosToKeyValueTableArray` + **`GET /api/class_series_runs/<classId>`** (public; pricing resolves for the logged-in viewer or public). Helper + endpoint tests added. ServerAccess exposes it as `getSeriesRuns(classId)`.

### 7.1 Catalog series cards
- [x] **Done.** Added `kind` to `ClassCatalogEntry` (struct + `BuildEntry` + `ClassCatalogEntryToKeyValueTable` + TS type) so the public catalog (**Our Classes ▸ All Classes**, `class-info`) renders a **Series** / **Workshop** badge next to the class name. Specs added (badge shows for series/workshop, hidden for recurring).

### 7.2 Series detail page + ### 7.3 Booking flow
- [x] Implemented as an **extension of `class-detail`** (per the plan's "or extension" option). On load it calls `getSeriesRuns(classId)`; when runs come back it renders a **Series Runs** section: per run the name, "N sessions · {date range}", resolved **full-series total** + **per-session** price, a **min-attendees warning** when `min_attendees` is set, the **non-refundable banner**, and CTAs. Logged-out viewers see "Log in to book". `class-detail.component.{ts,html,scss}` + extended `.spec.ts` (renders section, price, min warning, login-gated, books full series → success message, friendly error on failure, prorated CTA once the run has started).
- [x] Booking calls `bookSeries(classInstanceId, joinDateUs?)` — full when not started, **prorated** ("Join from today") once started and `prorated_signups_allowed`. On success it shows "Booked! N sessions added — pay at checkout to confirm". *(Payment remains the existing `purchase_pay_card` step, matching `BookEvent`; a dedicated thank-you screen wiring into the checkout/my-bookings link is the remaining polish.)*

### 7.4 My-bookings series rollup
- [x] **Done.** Corrected an earlier error: the bookings page **does** exist (`MyEventsComponent`, avatar ▸ **My Bookings**, `/my/events`). The upcoming list now groups event bookings sharing one `purchase_id` (2+) into a single expandable **series rollup panel** ("N sessions · date range") with the child session cards inside; standalone bookings render as before. The card body was extracted into an `ng-template` reused by both paths so the cancel flow is unchanged. Spec added (grouping + panel render + single stays single).

### 7.5 Admin series-create form
- [x] **Done.** New **`series-run-form-dialog`** (run name, product, start/end dates, optional min-attendees + decide-by date + min-not-met policy select, prorated checkbox, and a repeatable slot editor: day/time/duration/facility/room). Surfaced as an **"Add series run"** button on **Manage Products ▸ Class Schedules** (shown only when the selected class is `kind='series'`), wired to `createSeriesInstance`. Dialog spec covers defaults, slot add/remove, room-by-facility filtering, all validation paths, and the normalized save result. *(The "per-tier price matrix" is handled by setting `per_instance_base` rows in the §8 admin Price Kind dropdown rather than inside this dialog.)*

### 7.6 `ServerAccess` extensions
- [x] `getSeriesRuns(classId)`, `createSeriesInstance(req)`, `bookSeries(classInstanceId, joinDateUs?)`, `checkSeriesMinAttendees(classInstanceId, asOfUs?)`, `cancelSeries(classInstanceId, reason)` — added to the interface, `ServerAccessProxy`, `ServerAccessNetwork` (with string→bool normalization), and `ServerAccessMock`.
- [x] `ServerAccess.mock.spec.ts` — new `describe('class series')` block: getSeriesRuns, create (success + INVALID_WINDOW/INVALID_MIN_NOT_MET_POLICY + 401), book (full/prorated/ALREADY_BOOKED/INSTANCE_NOT_FOUND/401), check-min (skip-before-deadline + admin_alerted), cancel (+401).

### 7.7 Types
- [x] `series.types.ts`: `SeriesRun`, `SeriesSlotInput`, `CreateSeriesInstanceRequest`/`Response`, `BookSeriesResult`, `CheckMinAttendeesResult`, `CancelSeriesResult`.

## 8. Admin Metadata

- [x] No new top-level tables.
- [x] `product_prices.price_kind` shows up in the admin column data info with friendly name "Price Kind" and a dropdown editor for the three values — added the column-data-info row, the friendly name, a new `product_price_kind` enum (standard / series_total / per_instance_base) and the column binding in `create_database.cpp`.

# Mason- Issues Noted
- I don’t know that we should have slot generation in the Add series run
- When I try and add a class instance that is a single day to essentially say that the studio is closed that day, nothing happens and I see:
	- (2026-06-08 21:43:39) [INFO    ] Request: 127.0.0.1:4088 000002BC01509150 HTTP/1.1 POST /api/admin/class_schedule
	- (2026-06-08 21:43:39) [ERROR   ] ErrorResponse 400 Bad Request: INVALID_WINDOW
- I added the class for the series and it doesn’t show up as an option under Our classes or show up for All Classes. When I go to our schedule and scroll forward to the next month where the series is scheduled, the classes show up but it should still show up even if it is not this month. Actually, it doesn’t show up even if I make it start this month.
- When I created a series run, I initially had it start in July. When I would view our schedule, the class would show up when I would scroll forward to that week in July. When I went back and edited the series to now start in June and went and looked at our schedule, the classes in June don’t show up so the edit didn’t change anything.
- Can I add another class schedule in a class instance for a series that essentially “cancels” one instance of a class and have that reflect in the paymentWhen I am creating a class schedule and do the from date and set a date in the next month, when I choose the valid to date, the calendar control defaults to this month. I’d like it to move forward to the same month as the valid to
- In the pricing grid for a product for permissions, the column width for each permission is too narrow so it is essentially eligible. I think there are two issues here:
	- Instead of having a grid with permissions on top with narrow columns and then row for price schedules, I think it would be better to have the rows be permissions and the columns be price schedules since there are far fewer price schedules
	- I don’t think that every permission should show up for pricing. I think we should add a table that shows which permissions are available for pricing and only show those permissions in the places where we do class restrictions and pricing. Some public permissions table.


## 9. Live-Test Follow-Up Work Items (found 2026-06-08)

Distilled from **Mason- Issues Noted** above. Each is a discrete work item; tags: **[bug]** broken behavior, **[enh]** improvement, **[q]** decision needed before building. Respect layering (lowest layer first) and add tests per item.

### 9.1 Series/workshop classes don't appear in the catalog [bug]
- [ ] A `kind='series'` class with a run does **not** show under **Our Classes** / **All Classes** — even when the run starts this month. Only **Our Schedule** shows it (when you scroll to the run's week), because that view derives from the run window, not the catalog.
- [ ] Root cause: the catalog (`ClassCatalogHelper::GetClassesVisibleToPerson`) only includes a class with an instance **active at `now`** (`GetActiveInstance(classId, now)`); a series whose run is upcoming (or starts later this month) has no currently-active instance, so it's filtered out.
- [ ] Fix: include classes with an **upcoming** run (active + not-yet-ended as of `now`, like `GetUpcomingInstances`) instead of "covers now"; keep the access-gate filter. Decide whether the catalog card shows the run window / "starts {date}".
- [ ] Tests: catalog includes a series whose only run is future-dated; still excludes a class with no runs.

### 9.2 Editing a series run's window doesn't move the schedule [bug]
- [ ] Editing a run's start (e.g. July → June) did not move the derived sessions — **Our Schedule** still showed July, not June.
- [ ] Root cause: the run's **implementation** (`class_schedules`) window is separate from the instance window. "Add series run" sets the impl `valid_from` to the run start, but later editing the **instance** doesn't update the impl, so occurrences still derive from the stale impl window (and the augmentation min dates may also need to move).
- [ ] Fix: a series-run edit must keep the base implementation's `valid_from/valid_to` (and slot applicability) in sync with the instance window — likely a dedicated "edit series run" path rather than the generic instance edit.
- [ ] Tests: edit a run window → `GetDerivedSessionsForRange` reflects the new window.

### 9.3 Single-day "studio closed" implementation rejected (INVALID_WINDOW) [bug]
- [ ] Adding a single-day `class_schedule` to mark the studio closed fails: `POST /api/admin/class_schedule → 400 INVALID_WINDOW`, and the UI shows **nothing** (the 400 isn't surfaced).
- [ ] Root cause: impl-window validation requires `valid_to > valid_from` strictly; a one-day window has `to == from`.
- [ ] Fix (backend): allow a single-day implementation (treat `valid_to == valid_from` as an inclusive one-day window) **or** add a dedicated "close this day" / holiday-override action that builds the empty higher-priority impl.
- [ ] Fix (frontend): surface backend 400s (INVALID_WINDOW, etc.) in the schedule dialogs instead of silently no-op'ing.
- [ ] Interacts with 9.6 (holiday override reduces occurrence count → pricing).

### 9.4 "Valid to" datepicker should open on the "valid from" month [enh]
- [ ] When "valid from" is in a future month, opening the "valid to" picker shows the **current** month; it should open on the same month as "valid from".
- [ ] Fix: bind `[startAt]` on the "valid to" `mat-datepicker` to the "valid from" value in the instance-form, schedule-form, and series-run-form dialogs.

### 9.5 Pricing grid layout + permission scope [enh]
- [ ] (a) **Transpose** the product pricing matrix: rows = permissions (tiers), columns = price schedules (far fewer schedules), so columns aren't too narrow to read.
- [ ] (b) Don't show **every** permission in pricing. Introduce a "pricing-eligible / public permissions" set (a table, or a flag on `permissions`) marking which permissions are offered for pricing **and** class restrictions, and only show those in the pricing matrix and the class-requirements editor.
- [ ] Backend: new table/flag + table helper + a read endpoint for the eligible set. Frontend: pricing matrix + requirements editor consume it.
- Mason- This is a pretty large work item. I'd like to wrap up all the other stuff and then do this separately so please move this to it's own section 10 and move the existing 10 and all the following sections down one.

### 9.6 Per-occurrence cancellation within a run, reflected in pricing/refunds [q]
- [ ] Mason: *"Can I add another class schedule in a class instance for a series that essentially 'cancels' one instance of a class and have that reflect in the payment?"* The redesign envisioned holiday overrides as higher-priority empty impls that reduce the derived-occurrence count (and thus the series total).
- [ ] Decide the policy: does removing an occurrence **after** purchase trigger a partial refund (per the §4.2 per-occurrence path)? Does it affect already-sold runs or only future pricing? Then implement.
- Mason- Let's trigger a partial refund. If I end up needing to remove a day from a series because there is studio maintenance or the instructor will be out of town, students should be refunded.

### 9.7 Should "Add series run" generate slots at all? [q]
- [ ] Mason: *"I don't know that we should have slot generation in the Add series run."*
- [ ] Option A: keep slot entry in the dialog (one-stop run creation). Option B: the dialog creates only the instance + augmentation + base impl, and the admin authors slots via the existing slot editor under the run (consistent with recurring classes).
- [ ] Decide; if B, remove the slot editor from `series-run-form-dialog` and document the two-step flow.

## 10. Tests-Required Summary

- [ ] Table helper tests for `price_kind` round-trip + `GetSeriesPricesForProduct`.
- [ ] `class_series_helper_test.cpp` covering all five methods + the three `series_min_not_met_policy` branches.
- [ ] Endpoint tests for all four new endpoints.
- [ ] Mail helper test: cancellation email queued for each attendee on `CancelSeries`.
- [ ] Frontend specs for series-detail, my-bookings series rollup, admin series-create form, mock service.
- [ ] Manual-testing-helper commands: `create_series <...>`, `book_series <person_id> <schedule_id>`, `simulate_series_under_min <schedule_id>`.

## 11. Cross-Layer Acceptance Criteria

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

## 12. Open Questions

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

## 13. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 7.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 4 - iCal Generator Extensions]].
- Payment context: [[Payment Design Document]], [[Purchase creation with server-side pricing]], [[Vouchers and Refunds]].
- Scheduler: [[Scheduled Jobs]].
