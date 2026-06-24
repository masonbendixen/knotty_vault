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

Classes Phase 12 - Specialty Instructor Cost

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

**Nice-to-Have.** Admin records a specialty instructor's compensation rate for a class / series via the same `price_schedules` infrastructure that drives student pricing (P-2). Pricing assistant suggests per-tier prices to cover the cost on a break-even attendance and profit beyond. Per-class-type max attendees per instructor stored in a separate `instructor_class_preferences` table (parent OQ-23). Cost vs revenue panel on admin session detail. Full payroll export deferred to [[Classes Phase 16 - Stretch Items]] (PR-1..PR-4).

**Prerequisites:**
- Phase 1 (three-level model — `classes` / `class_instances` / `class_schedules` / `class_schedule_slots`; lazy derivation; `ClassScheduleHelper::EnsureSessionExists`).
- Phase 2 (product_prices per tier).
- Phase 7 (series + per-instance-base pricing in product_prices; `class_series_instances`).
- Existing `price_schedules` infrastructure ([[Payment Design Document]]).

### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]]
A specialty instructor is hired for a specific **run**, so cost keys off **`class_instance_id`**, not a flat `class_schedule_id`. And there is no materialization, so the rate-snapshot moves to **ensure-time**: when an occurrence `event_sessions` row is ensured (booking / check-in / sub), `EnsureSessionExists` (or a small hook it calls) stamps `event_sessions.specialty_instructor_cost_id` from the active cost row for the occurrence's instance. Optional per-slot cost override if the specialty teacher only covers some days of the run. Rate changes still roll forward via `price_schedules` (P-2).

**Outcome:**
- `specialty_instructor_costs` table tied to a run via `class_instance_id` + `price_schedule_id`, NOT per flat schedule (parent SI-1 / SI-6).
- `instructor_class_preferences` table (per `(instructor, class)` min/max preferences).
- Admin pricing-assistant endpoint returns per-tier suggestion.
- Admin session-detail cost vs revenue panel.

## Layering & Conventions

Lowest layer first:

1. `db_schema/` — two new tables.
2. `sql_util/table_helpers/` — two new helpers.
3. `business_logic/scheduling/` (or new `business_logic/payroll/` namespace) — `SpecialtyCostHelper`.
4. `endpoints/` — pricing-assistant + cost/revenue endpoints.
5. Angular: admin session-detail extension.
6. Admin metadata.
7. Tests.

## 1. Pre-Coding Design Decisions

### 1.1 Locked-in
- [x] Cost lives at the **run (`class_instances`) level** (a specialty instructor is hired for a specific run), NOT a flat schedule and NOT per individual occurrence (parent SI-1; redesign).
- [x] Cost rows reference a `price_schedule_id` so rate changes roll forward via a new schedule row (parent SI-6 / P-2 / OQ-42).
- [x] Per-instructor / per-class-type max attendees in a separate `instructor_class_preferences` table (parent OQ-23).
- [x] Occurrences snapshot the rate when their `event_sessions` row is **ensured** (no materialization step — redesign).
- [x] **Profit margin is configurable (resolved OQ-P12-1).** The pricing-assistant margin is NOT hard-coded. It comes from a `non_member_profit_margin_pct` config secret (default 50%) and can be overridden per request from the Suggest-pricing dialog — globally and/or per tier. (Mason: "I want this to be configurable.")
- [x] **The snapshot rate is authoritative for payroll (resolved OQ-P12-2).** `ComputeInstructorPayCents` always reads the cost row referenced by `event_sessions.specialty_instructor_cost_id`, even after that cost's `price_schedule` window has closed — pay is computed from the rate in effect when the occurrence was ensured, never the current live rate. (This was never really an open question — it's a requirement; covered by the §8 regression test.)

## Post-acceptance refinements (2026-06-23, from Mason's testing)

Three fixes after the first end-to-end run:

1. **Stacked action icons (UI).** The specialty-cost row's edit/delete `mat-icon-button`s wrapped vertically (doubling row height). Wrapped them in an `inline-flex` `.action-buttons` container + `width:1%` on the cell so they never get squeezed. Frontend only.

2. **Instructor pay showed $0 when the cost was configured AFTER someone booked (live-cost fallback).** Root cause: the snapshot (`event_sessions.specialty_instructor_cost_id`) is stamped at ensure-time, so a session materialized by a booking placed *before* any cost existed has a NULL snapshot, and a later-added cost never back-fills it. Fix: `ComputeInstructorPayCents` now resolves the cost via `ResolveSessionCost` — **snapshot if present (still authoritative, OQ-P12-2), else the run's current active cost** for the session's slot+instructor at the session's start time. So a rate configured after signup now applies to those pre-existing sessions. Requires the **slot to have an instructor assigned** that matches the cost's instructor (pay can't be attributed otherwise) — documented for the user. Added `classSchedules_` member. Tests: fallback-applies-late-cost (base + threshold bonus), fallback-zero-when-slot-has-no-instructor, snapshot-still-wins.

3. **Cost/revenue is now reported PER RUN, not per session/date (Mason: "better to do this reporting per instance").** The old per-date widget showed a single occurrence's view but summed the *whole bundled series purchase* as that date's revenue (a series is one purchase spanning N sessions → every session reported the full total). New design:
   - **Backend:** `Bookings::GetRunAttendanceRevenue` (distinct-person counts + revenue summing each `purchase_item` **once** via slot→schedule→instance join, so a bundle isn't multiplied across sessions), `EventSessions::GetSessionIdsForInstance`, `SpecialtyCostHelper::GetRunCostRevenue` (sums each session's pay using the fallback above + run revenue), `RunCostRevenueReport` + converter, `GET /api/admin/class_instance/<id>/cost_revenue`. Tests at each layer incl. the bundled-series-counted-once case.
   - **Frontend:** a **Cost & revenue (this run)** block in the Specialty instructor cost panel (sessions / attendees / instructor pay / revenue / margin, red when negative). The per-date widget is now **suggest-pricing only** (pick a date → materialize → open the suggest dialog); the per-session actuals panel (`session-cost-revenue-panel`) was deleted. `ServerAccess.getRunCostRevenue` across all layers + mock-spec. Full UI suite green (2724).

## 2. Database Schema

### 2.1 `specialty_instructor_costs` table ✅ (2026-06-19)
- [x] `db_schema/specialty_instructor_costs.h/.cpp`:
  - `id BIGSERIAL PK`
  - `class_instance_id BIGINT NOT NULL REFERENCES class_instances(id)` — the run the instructor is hired for
  - `class_schedule_slot_id BIGINT NULL REFERENCES class_schedule_slots(id)` — optional per-slot override (specialty teacher only covers certain days of the run)
  - `instructor_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `price_schedule_id BIGINT NOT NULL REFERENCES price_schedules(id)`
  - `base_rate_cents BIGINT NOT NULL`
  - `per_student_bonus_cents BIGINT NOT NULL DEFAULT 0`
  - `bonus_threshold_count BIGINT NULL` — bonus applies for attendees beyond this count
  - `notes TEXT NOT NULL DEFAULT ''`
  - `created_us`, `updated_us`
- [x] Unique on `(class_instance_id, class_schedule_slot_id, instructor_person_id, price_schedule_id)` (named `uq_specialty_instructor_costs_run_slot_instructor_schedule`). **Schema note:** because Postgres treats NULLs as distinct, whole-run rows (NULL slot) are NOT deduped by this constraint — dedup of whole-run rows is the helper's job (§3/§4). Documented + locked in by `WholeRunRowsAreNotDedupedByNullSlot`.
- [x] Schema test `specialty_instructor_costs_test.cpp`: valid-insert-with-defaults, per-slot+bonus, the three NOT-NULL FK rejections (instance/instructor/price_schedule), slot-FK rejection, base-rate NOT NULL, unique-blocks-duplicate / allows-new-window, NULL-slot-not-deduped.

### 2.2 `instructor_class_preferences` table ✅ (2026-06-19)
- [x] `db_schema/instructor_class_preferences.h/.cpp`:
  - `id BIGSERIAL PK`
  - `instructor_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `class_id BIGINT NOT NULL REFERENCES classes(id)`
  - `min_attendees BIGINT NULL`
  - `max_attendees BIGINT NULL`
  - `notes TEXT NOT NULL DEFAULT ''`
  - `created_us`, `updated_us`
- [x] Unique on `(instructor_person_id, class_id)` (named `uq_instructor_class_preferences_instructor_class`).
- [x] Schema test `instructor_class_preferences_test.cpp`: valid-insert-with-defaults, min/max+notes, instructor-FK + class-FK rejections, unique-blocks-duplicate-pair / allows-different-class.

### 2.3 Ensure-time rate snapshot ✅ (2026-06-19)
- [x] Added `event_sessions.specialty_instructor_cost_id BIGINT` NULL `REFERENCES specialty_instructor_costs(id)` (`kEventSessionsSpecialtyInstructorCostId`) — set when the occurrence row is ensured (no materialization) so payroll later reads the snapshot rather than the live rate. event_sessions is now created AFTER specialty_instructor_costs in both the builder and `CreateTables` so the FK resolves.

### 2.4 Ensure-time hook ✅ (2026-06-19, alongside §3.1)
- [x] In `ClassScheduleHelper::EnsureSessionExists`, when creating a NEW occurrence row, if the slot has an `instructor_person_id` we look up `specialtyCosts_.GetActiveCostForOccurrence(instanceId, slotId, instructorPersonId, asOfUs=startUs)` and, when found, set `event_sessions.specialty_instructor_cost_id` on the new row. `asOfUs` is the occurrence's **start time** (the rate in effect for that session). A per-slot override wins over a whole-run row (handled by the table helper's ORDER BY).
- [x] No cost row (or no slot instructor) → column stays NULL. Idempotent for free: the stamp only runs on the new-row branch; an already-persisted occurrence returns early before this code, so re-ensuring never re-stamps.
- [x] Wired a `TableHelpers::SpecialtyInstructorCosts specialtyCosts_` member into `ClassScheduleHelper` (no ad-hoc SQL in business logic — `feedback_no_sql_in_business_logic` satisfied).
- [x] Tests in `class_schedule_helper_test.cpp`: stamps the snapshot when an active cost exists; leaves NULL when the instructor has no cost row; leaves NULL when the slot has no instructor (even if a cost exists for someone else).

### 2.5 Wire into DB init ✅ (2026-06-19)
- [x] `make_database_info.cpp` (both `Make*Table` calls added before `MakeEventSessionsTable`, with includes) + `create_database.cpp` `CreateTables()` (both `CreateTable` calls before `kEventSessionsTable`, with includes). Also added both files to `db_schema/CMakeLists.txt` (sources + the two `_test.cpp` to the test target).

### 2.6 Config secret (resolved OQ-P12-1) ✅ (2026-06-19)
- [x] Seeded `non_member_profit_margin_pct`, default `50` (percent), via the established secrets-default mechanism: `kNonMemberProfitMarginPct` in `secret_keys.h` + `kNonMemberProfitMarginPctValue` ("50") + `addSecret(...)` in `secret_values.cpp`. This auto-loads into the DB on first run AND into the test secrets helper (so §4.2 / §8 tests get the default for free). Read via `SecretsHelper` in §4.2 (no secrets lookup inside table helpers). *(Used `secret_keys.h`/`secret_values.cpp` rather than hand-seeding in `create_database.cpp` — that's the canonical path and the only one that also covers the test helper.)*

## 3. Table Helpers

### 3.1 `TableHelpers::SpecialtyInstructorCosts` ✅ (2026-06-19)
- [x] Full CRUD (`AddCost` / `GetCost` / `UpdateCost` (bumps `updated_us`) / `DeleteCost`) + `GetCostsForInstance(Transaction&, classInstanceId)` (all rows for a run, newest first).
- [x] **`GetActiveCostForOccurrence(Transaction&, classInstanceId, classScheduleSlotId, instructorPersonId, asOfUs)`** — joins to `price_schedules` and filters to the active window (`is_active` + `valid_from_us <= asOfUs < valid_to_us|∞`); a **per-slot override** (`class_schedule_slot_id = slotId`) wins over a whole-run row (slot NULL) via `ORDER BY (class_schedule_slot_id IS NULL) ASC`. *(Renamed from the doc's stale pre-redesign `GetActiveCostForSchedule(classScheduleId, …)` — cost keys off the run + occurrence slot, and the §2.4 hook needs the instructor + per-slot-override semantics.)*
- [x] Tests `specialty_instructor_costs_test.cpp` (table-helper layer): add/get, costs-for-instance (scoped + newest-first), active whole-run match, per-slot-override-wins, window respected (roll-forward old vs new), inactive-schedule excluded, instructor filter, none→nullopt, update, delete.

### 3.2 `TableHelpers::InstructorClassPreferences` ✅ (2026-06-19)
- [x] Full CRUD (`AddPreference` / `GetPreference` / `UpdatePreference` (bumps `updated_us`) / `DeletePreference`) + `GetPreferencesForInstructor(Transaction&, instructorPersonId)` + `GetForInstructorAndClass(...)` + **`GetMaxForInstructorAndClass(Transaction&, instructorPersonId, classId)`** (empty when no row or NULL max).
- [x] Tests `instructor_class_preferences_test.cpp` (table-helper layer): add/get, get-for-(instructor,class)/nullopt, get-max (value / NULL-max / no-row), get-for-instructor scoping, update, delete.

## 4. Business Logic — `SpecialtyCostHelper`

Files: `business_logic/scheduling/specialty_cost_helper.h/.cpp/_test.cpp`.

**Built (2026-06-19).** `business_logic/scheduling/specialty_cost_helper.{h,cpp,_test.cpp}` — pure orchestration of table helpers; no SQL in business logic (`feedback_no_sql_in_business_logic`). Two existing table helpers gained the cross-table queries it needs (each with its own added tests):
- **`Bookings::GetSessionAttendanceRevenue(Transaction&, eventSessionId)`** → one-row aggregate `{attendee_count, paid_count, revenue_cents}`. "Held seat" = status not in (`cancelled`,`waitlisted`); paid rows also carry a purchase; revenue = `SUM(purchase_items.line_total_cents)` over paid held seats (LEFT JOIN so free RSVPs still count as attendees). Tests in `bookings_test.cpp`.
- **`ProductEntitlementRules::IsPermissionGrantedBySubscriptionProduct(Transaction&, permissionId)`** → member-tier detection: EXISTS an **active `subscription`** product granting the permission. Tests in `product_entitlement_rules_test.cpp` (subscription→true, one_time→false, no-rule→false, inactive→false).

### 4.1 Pay computation ✅
- [x] `int64_t ComputeInstructorPayCents(Transaction&, int64_t eventSessionId, int64_t attendeeCount)`: reads `event_sessions.specialty_instructor_cost_id` (NULL → 0), loads the cost row (missing → 0), then `pay = base_rate_cents`; with a threshold the per-student bonus applies only to attendees beyond it, else (no threshold + positive bonus) to every attendee. Reads the SNAPSHOT row, authoritative even after its window closes (OQ-P12-2). Tests: base-only, bonus-no-threshold, bonus-with-threshold (4/6/10), no-cost→0, missing-cost→0, snapshot-not-live regression.

### 4.2 Pricing assistant ✅
- [x] `PricingSuggestion { permissionId; permissionName; suggestedPriceCents; appliedMarginPct; rationale; }` and `PricingSuggestionRequest { eventSessionId; targetAttendees; allowedPermissionIds; std::optional<double> profitMarginPct; std::map<int64_t,double> perTierMarginPct; }`. *(Keyed on `eventSessionId`, not the doc's stale pre-redesign `classScheduleId` — the §5 endpoint is `/api/admin/session/<id>/suggest_pricing`, and the cost is the session's snapshot, so the assistant reuses `ComputeInstructorPayCents` directly.)*
- [x] `SuggestPricesForBreakeven(Transaction&, request, const Secrets::SecretsHelper&)`: default margin from the `non_member_profit_margin_pct` secret (50% fallback); `totalCost = ComputeInstructorPayCents(target)`; `breakEvenPerAttendee = round(totalCost / target)`; per tier — member tier (`IsPermissionGrantedBySubscriptionProduct`) → break-even, `appliedMarginPct=0`; non-member → `round(breakEven × (1 + margin/100))` with margin precedence `perTierMarginPct[id]` > `profitMarginPct` > secret. Rationale names the margin. Tests: member break-even, non-member default-from-secret, secret-changed, request override, per-tier-wins, member-ignores-margin, mixed tiers, bonus-in-breakeven.

### 4.3 Cost / revenue report ✅
- [x] `SessionCostRevenueReport { eventSessionId; attendeeCount; paidAttendeeCount; instructorPayCents; revenueCents; marginCents; }`.
- [x] `GetSessionCostRevenue(Transaction&, eventSessionId)`: pulls `Bookings::GetSessionAttendanceRevenue`, computes pay at `attendeeCount`, `marginCents = revenue − pay` (can be negative). Tests: paid/free/cancelled/waitlisted mix; zero-cost session.

### 4.4 KeyValueTable conversions ✅
- [x] `PricingSuggestionToKeyValueTable` (+ array) and `SessionCostRevenueReportToKeyValueTable` in `scheduling_key_value_table.*`. `applied_margin_pct` formats with `%g` so whole margins (50) coerce to a JSON number and fractional (32.5) stay precise. Tests in `scheduling_key_value_table_test.cpp` (whole + fractional margin, array, report incl. negative margin).

## 5. Endpoints ✅ (2026-06-19)

**Built — 10 thin handlers, each gated on `manage_class_schedule`, registered in `web_app.cpp` + `endpoints/CMakeLists.txt`, one route per file.** Authoring endpoints delegate to `SpecialtyCostHelper` (§4 + new create/update/close methods); list/preference-CRUD endpoints call the table helpers directly (the established Phase 1 `admin_class_instances_list` pattern). Two new join reads added to the table helpers (`SpecialtyInstructorCosts::GetCostsForInstanceWithDetails`, `InstructorClassPreferences::GetPreferencesForInstructorWithClassName`) + `SpecialtyCostMutationResultToKeyValueTable` converter. **Roll-forward uses the `price_schedule.is_active` toggle** (deactivate old + new active schedule) rather than wall-clock `valid_to` windows — keeps history intact for snapshots, needs no time source, and matches `GetActiveCostForOccurrence`'s `is_active` filter.

- [x] `POST /api/admin/session/<id>/suggest_pricing` body `{ target_attendees, tiers: [permission_id...], profit_margin_pct?, per_tier_margin_pct?: { permission_id: pct } }` → `{ suggestions: [...] }` (each echoing `applied_margin_pct`). Margin fields optional → fall back to the `non_member_profit_margin_pct` secret. Tests: 403, 400-missing-target, member+non-member fall-back-to-secret (50%→$75 vs break-even $50), request-margin-override (30%→$65).
- [x] `GET /api/admin/session/<id>/cost_revenue` → `SessionCostRevenueReport`. Tests: 403, 200 (asserts pay/margin).

#### Specialty-cost authoring endpoints (bespoke — NOT generic CRUD / Manage Data)
> **Manage Data is debug-only** (memory `feedback_manage_data_is_debug_only.md`). Hiring a specialty instructor and setting their pay rate is a core admin workflow, so it gets a bespoke `/manage` flow with dedicated endpoints — never "go to Manage Data and add a row." The endpoints below are thin (parse → `SpecialtyCostHelper` / table helper → KeyValueTable→JSON) per the layering rule.
- [x] `GET /api/admin/class_instance/<id>/specialty_costs` → `{ items: [...] }` (each with resolved `instructor_name` + `price_schedule_*` window). Tests: 403, 200 (asserts instructor_name "Visiting Maya").
- [x] `POST /api/admin/specialty_cost` body `{ class_instance_id, class_schedule_slot_id?, instructor_person_id, base_rate_cents, per_student_bonus_cents?, bonus_threshold_count?, notes? }` → creates the cost + backing active `price_schedule`. Tests: 403, 400-missing-field, 400 INSTANCE_NOT_FOUND, 200+persist.
- [x] `PUT /api/admin/specialty_cost/<id>` body `{ base_rate_cents, per_student_bonus_cents?, bonus_threshold_count?, notes? }` — roll-forward (deactivate old schedule + new schedule/cost, same run/slot/instructor). Tests: 403, 404, 200 (new cost id, base 60000).
- [x] `DELETE /api/admin/specialty_cost/<id>` — soft-close (deactivates the schedule; cost row preserved). Tests: 403, 404, 200 (schedule inactive).
- [x] `GET /api/admin/instructor/<id>/class_preferences` (→ items with `class_name`), `POST /api/admin/instructor_class_preference`, `PUT`/`DELETE /api/admin/instructor_class_preference/<id>` — bespoke CRUD via the `InstructorClassPreferences` table helper. Tests at each verb (403 / 400-missing or 404 / 200+persist).
- [x] Thin handlers in `endpoints/`; the create/update/close orchestration (cost + price_schedule roll-forward) lives in `SpecialtyCostHelper` (`SpecialtyCostRequest` / `SpecialtyCostMutationResult` + error codes), with its own helper tests. *(Chose bespoke endpoints over generic CRUD because the POST/PUT need the price_schedule orchestration and the GETs need joins — generic CRUD can't express either.)*

## 6. Frontend ✅ (2026-06-20)

**Built — full §6.4/§6.5 foundation + §6.2/§6.3 authoring UIs wired into their hosts + §6.1 dialog/panel as self-contained components.** Whole frontend suite green (**2720 tests**). Components are self-contained sub-components (testable in isolation, minimal host coupling); specs instantiate them directly with `ServerAccessMock` + dialog spies.

### 6.1 Admin session detail extension ✅ (components built; host page deferred)
- [x] `session-cost-revenue-panel.component` (`manage/session-pricing/`): `@Input() sessionId` → loads `getSessionCostRevenue`; shows attendees / instructor pay / revenue / margin (red when negative); "Suggest pricing" button opens the dialog. Spec: load, negative-margin flag, $ formatting, dialog-open args.
- [x] `suggest-pricing-dialog.component`: target-attendees + tiers multiselect + profit-margin input (pre-filled from `defaultMarginPct`) + per-tier overrides → calls `suggestPricingForSession`, renders a per-tier price/margin table. Spec: validation, member-breakeven vs non-member-margin, margin edit re-prices, per-tier override wins, $ formatting.
- **Note:** there is **no admin session-detail page** in the app (the doc's `portal/manage/event-session-detail/` doesn't exist). The panel + dialog are self-contained and take their inputs (`sessionId`, `tiers`, `defaultMarginPct`) from a host, so they drop in unchanged once that page is built. Host wiring is the only deferred piece of §6.

### 6.2 Specialty-cost authoring on the run ✅
- [x] `specialty-cost-form-dialog.component` (+ spec) under `manage/class-schedules/dialogs/` — instructor picker + optional per-slot scope + money inputs (dollars → cents) + bonus threshold + notes; edit mode shows only rate fields (roll-forward). Spec: validation, $→cents, whole-run vs slot scope, negative-bonus reject, edit prefill keeps instructor.
- [x] `specialty-cost-section.component` (+ spec) — lists the run's **active** cost rows, opens the dialog for add/edit, calls create/update(roll-forward)/delete(close). Wired into `class-schedule-manage` under each run (`[classInstanceId]`, `[instructors]`, `[slots]`), with `instructorOptions`/`slotOptionsForInstance` accessors added to the host. Host spec mock got `getSpecialtyCostsForRun`.

### 6.3 Instructor class-preferences authoring ✅
- [x] `instructor-class-preferences.component` (+ spec) — inline add (class autocomplete via `getFkOptions('classes')`) / edit / delete of `(class, min, max, notes)` rows. Wired into the `instructors-admin` edit panel (`[instructorPersonId]="editing.person_id"`); host spec mock got `getInstructorClassPreferences`. Spec: load, require-class, add/edit/remove round-trip, empty-search clears options.

### 6.4 `ServerAccess` extensions ✅
- [x] All 10 methods (`suggestPricingForSession`, `getSessionCostRevenue`, `getSpecialtyCostsForRun`, `create/update/deleteSpecialtyCost`, `getInstructorClassPreferences`, `create/update/deleteInstructorClassPreference`) across interface / network / proxy / mock. The network layer normalizes the KVT wire format (booleans, NULL/fractional → real bool/number/undefined).
- [x] `ServerAccess.mock.spec.ts` — 14 cases (suggest member/non-member/override/401, cost_revenue, specialty-cost create→list / roll-forward / COST_NOT_FOUND / delete / 401, preference round-trip / 404 / 401).

### 6.5 Types ✅
- [x] `specialty-cost.types.ts`: `PricingSuggestion`, `PricingSuggestionRequest`, `SessionCostRevenueReport`, `SpecialtyCost`, `Create/UpdateSpecialtyCostRequest`, `SpecialtyCostMutationResult`, `InstructorClassPreference`, `Create/UpdateInstructorClassPreferenceRequest`.

## 7. Admin Metadata (debug-only fallback) ✅ (2026-06-20)

- [x] `specialty_instructor_costs` + `instructor_class_preferences` registered in `create_database.cpp` across all 6 Populate* functions: top-level tables, `manage_class_schedule` permission, column data info, column friendly names, table friendly names, display templates. Money columns (`base_rate_cents`, `per_student_bonus_cents`) use the `"number"` edit type labeled "(cents)" — matching the codebase convention (`product_prices.amount_cents` etc.; there is no dedicated cents edit type). created/updated are read-only. (Schema/make_database_info/CreateTables/CMake were already done in §2.)

## 8. Tests-Required Summary ✅

- [x] Table helper tests for both new tables (§3 — `specialty_instructor_costs_test.cpp`, `instructor_class_preferences_test.cpp`).
- [x] `specialty_cost_helper_test.cpp`: pay with/without bonus + threshold; member break-even vs non-member margin; configurable margin (request override, per-tier wins, secret fallback, member ignores); snapshot-authoritative regression; cost vs revenue counts only paid held seats. (§4)
- [x] Endpoint tests — suggest_pricing, cost_revenue, and the bespoke authoring endpoints at each verb (403 / validation / persist). (§5)
- [x] Frontend specs — the §6.1 panel + dialog, §6.2 dialog + section, §6.3 preferences, and the `ServerAccess.mock.spec.ts` cases for every new mock method. **Full UI suite: 2720 green.**
- **Build note:** the C++ tests (§3/§4/§5) compile + run on the user's machine; the frontend suite was run here and is green.

## 9. Cross-Layer Acceptance Criteria

Admin sets up a "Hands Balancing Workshop" with specialty instructor "Visiting Maya" at $400 base + $25 per student past 6:
- [ ] Materializing the workshop session snapshots `specialty_instructor_cost_id`.
- [ ] At 10 attendees: pay = $400 + 4×$25 = $500.
- [ ] Suggested member price (gold tier, 8 target) = $400/8 = $50 break-even.
- [ ] Suggested non-member price = $50 × 1.5 = $75 at the default 50% margin (from the `non_member_profit_margin_pct` secret); admin overrides the margin to 30% in the dialog → suggestion re-computes to $50 × 1.3 = $65, with `applied_margin_pct=30` shown (resolved OQ-P12-1).
- [ ] Cost vs revenue at 10 attendees (8 gold @ $50, 2 non-member @ $75): revenue = 8×$50 + 2×$75 = $550; cost $500; margin $50.
- [ ] A month later the rate is raised via a new `price_schedule` window; payroll for the already-ensured October sessions still computes at the **old** snapshotted $400 base, not the new rate (resolved OQ-P12-2).

## 10. Open Questions

Both resolved (Mason, 2026-06-09) and folded into the plan above (§1.1 Locked-in + the cited sections).

- **OQ-P12-1. — RESOLVED (Mason: "I want this to be configurable").** Build it now (don't defer): a `non_member_profit_margin_pct` secret (default 50%) supplies the fallback, overridable per request and per tier from the Suggest-pricing dialog; each suggestion echoes the `applied_margin_pct`. Folded into §1.1, §2.6, §4.2, §5, §6.1, §6.4–6.5, §8, §9.
- **OQ-P12-2. — RESOLVED (Mason: "Is there a question here?" — correct, it's a requirement, not a question).** The snapshot rate is authoritative: `ComputeInstructorPayCents` always reads the cost row referenced by `event_sessions.specialty_instructor_cost_id`, even after its `price_schedule` has closed. Folded into §1.1 (locked-in) with a regression test in §8 + acceptance check in §9.
	- Mason- Is there a question here?

## 11. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 12.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 7 - Class Series and Workshops]].
- Will feed into: [[Classes Phase 16 - Stretch Items]] (PR-1..PR-4 payroll).
- Pricing infra: [[Payment Design Document]].
