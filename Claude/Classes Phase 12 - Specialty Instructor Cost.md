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
- Phase 1 (class_schedules, classes).
- Phase 2 (product_prices per tier).
- Phase 7 (series + per-instance-base pricing in product_prices).
- Existing `price_schedules` infrastructure ([[Payment Design Document]]).

**Outcome:**
- `specialty_instructor_costs` table tied to a class / series via `price_schedule_id`, NOT per-instance (parent SI-1 / SI-6).
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
- [x] Cost lives at the schedule / series level, NOT per-instance (parent SI-1).
- [x] Cost rows reference a `price_schedule_id` so rate changes roll forward via a new schedule row (parent SI-6 / P-2 / OQ-42).
- [x] Per-instructor / per-class-type max attendees in a separate `instructor_class_preferences` table (parent OQ-23).
- [x] Materialized sessions snapshot the rate at materialization time (parent OQ-42).

## 2. Database Schema

### 2.1 `specialty_instructor_costs` table
- [ ] `db_schema/specialty_instructor_costs.h/.cpp`:
  - `id BIGSERIAL PK`
  - `class_schedule_id BIGINT NOT NULL REFERENCES class_schedules(id)`
  - `instructor_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `price_schedule_id BIGINT NOT NULL REFERENCES price_schedules(id)`
  - `base_rate_cents BIGINT NOT NULL`
  - `per_student_bonus_cents BIGINT NOT NULL DEFAULT 0`
  - `bonus_threshold_count BIGINT NULL` — bonus applies for attendees beyond this count
  - `notes TEXT NOT NULL DEFAULT ''`
  - `created_us`, `updated_us`
- [ ] Unique on `(class_schedule_id, instructor_person_id, price_schedule_id)`.

### 2.2 `instructor_class_preferences` table
- [ ] `db_schema/instructor_class_preferences.h/.cpp`:
  - `id BIGSERIAL PK`
  - `instructor_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `class_id BIGINT NOT NULL REFERENCES classes(id)`
  - `min_attendees BIGINT NULL`
  - `max_attendees BIGINT NULL`
  - `notes TEXT NOT NULL DEFAULT ''`
  - `created_us`, `updated_us`
- [ ] Unique on `(instructor_person_id, class_id)`.

### 2.3 Materialization snapshot
- [ ] Add `event_sessions.specialty_instructor_cost_id BIGINT` NULL `REFERENCES specialty_instructor_costs(id)` — set at materialization time (Phase 1 materializer extension) so payroll later reads the snapshot rather than the live rate.

### 2.4 Materialization hook update
- [ ] In `ClassScheduleHelper::MaterializeFutureSessions` (Phase 1), look up the active `specialty_instructor_costs` row for `(class_schedule_id, instructor_person_id, asOfUs)` via the active `price_schedule`. If found, set `event_sessions.specialty_instructor_cost_id` on each created session.
- [ ] If no cost row, leave NULL (non-specialty instructors).

### 2.5 Wire into DB init
- [ ] `make_database_info.cpp` + `create_database.cpp`.

## 3. Table Helpers

### 3.1 `TableHelpers::SpecialtyInstructorCosts`
- [ ] Full CRUD + `GetActiveCostForSchedule(Transaction&, classScheduleId, asOfUs)` — joins to `price_schedules` to filter to the active window.
- [ ] Tests.

### 3.2 `TableHelpers::InstructorClassPreferences`
- [ ] Full CRUD + `GetMaxForInstructorAndClass(Transaction&, instructorPersonId, classId)`.
- [ ] Tests.

## 4. Business Logic — `SpecialtyCostHelper`

Files: `business_logic/scheduling/specialty_cost_helper.h/.cpp/_test.cpp`.

### 4.1 Pay computation
- [ ] `int64_t ComputeInstructorPayCents(Transaction&, int64_t eventSessionId, int64_t attendeeCount)`:
  1. Load `event_sessions.specialty_instructor_cost_id`. If NULL → 0.
  2. Load the cost row.
  3. `pay = base_rate_cents`. If `bonus_threshold_count` set and `attendeeCount > bonus_threshold_count`: `pay += per_student_bonus_cents × (attendeeCount - bonus_threshold_count)`. Else if `bonus_threshold_count IS NULL` and `per_student_bonus_cents > 0`: `pay += per_student_bonus_cents × attendeeCount`.
  4. Return `pay`.

### 4.2 Pricing assistant
- [ ] `struct PricingSuggestion { int64_t permissionId; std::string permissionName; int64_t suggestedPriceCents; std::string rationale; }`.
- [ ] `struct PricingSuggestionRequest { int64_t classScheduleId; int64_t targetAttendees; std::vector<int64_t> allowedPermissionIds; }`.
- [ ] `std::vector<PricingSuggestion> SuggestPricesForBreakeven(Transaction&, const PricingSuggestionRequest&)`:
  1. Load active specialty cost for the schedule.
  2. Compute total cost at the target attendee count: `totalCost = ComputeInstructorPayCents(at attendees=target)`.
  3. For each tier in `allowedPermissionIds`:
     - If `tier` is a member tier (you can detect this via the permission's existing-membership-grant linkage; if the user holds an active membership granting that permission, they should "cover cost" only — break-even price = `totalCost / target`).
     - If `tier` is a non-member tier: profit margin applied (parameterize via secret `non_member_profit_margin_pct` default 50%) → `suggestedPriceCents = (totalCost / target) × (1 + margin)`.
  4. Return suggestions with rationale strings ("Break-even for Gold tier at 8 attendees: $X").

### 4.3 Cost / revenue report
- [ ] `struct SessionCostRevenueReport { int64_t eventSessionId; int64_t attendeeCount; int64_t paidAttendeeCount; int64_t instructorPayCents; int64_t revenueCents; int64_t marginCents; }`.
- [ ] `SessionCostRevenueReport GetSessionCostRevenue(Transaction&, int64_t eventSessionId)`:
  1. Compute pay via `ComputeInstructorPayCents`.
  2. Query paid revenue: SUM of `purchase_items.line_total_cents` across `bookings` for this session where `purchase_id IS NOT NULL`.
  3. Return.

### 4.4 KeyValueTable conversions
- [ ] `PricingSuggestionToKeyValueTable`, `SessionCostRevenueReportToKeyValueTable`.

## 5. Endpoints

- [ ] `POST /api/admin/session/<id>/suggest_pricing` body `{ target_attendees, tiers: [permission_id...] }` → list of suggestions. Permission `manage_class_schedule`. Endpoint test.
- [ ] `GET /api/admin/session/<id>/cost_revenue` → `SessionCostRevenueReport`. Permission `manage_class_schedule`. Endpoint test.
- [ ] Admin CRUD for `specialty_instructor_costs` + `instructor_class_preferences` uses the generic CRUD endpoints once metadata is registered.

## 6. Frontend

### 6.1 Admin session detail extension
- [ ] On the admin session-detail page (likely under `portal/manage/event-session-detail/`), add:
  - "Specialty cost" panel: shows rate + bonus + computed pay at current attendee count.
  - "Cost vs revenue" panel: revenue, cost, margin.
  - "Suggest pricing" button → opens a dialog with target-attendees input + allowed-tiers multiselect → calls the suggest-pricing endpoint → displays table of suggestions per tier.
- [ ] Specs.

### 6.2 `ServerAccess` extensions
- [ ] `suggestPricingForSession(sessionId, targetAttendees, tiers)`, `getSessionCostRevenue(sessionId)`.
- [ ] Update `ServerAccess.mock.spec.ts`.

### 6.3 Types
- [ ] `specialty-cost.types.ts`: `PricingSuggestion`, `SessionCostRevenueReport`.

## 7. Admin Metadata

- [ ] `specialty_instructor_costs` → `admin_top_level_tables`. Permission `manage_class_schedule`.
- [ ] `instructor_class_preferences` → `admin_top_level_tables`. Permission `manage_class_schedule`.
- [ ] Friendly names, column data info, display templates.

## 8. Tests-Required Summary

- [ ] Table helper tests for both new tables.
- [ ] `specialty_cost_helper_test.cpp`:
  - Pay computation with / without bonus, with / without threshold.
  - Pricing suggestions cover break-even for member, profit margin for non-member.
  - Cost vs revenue reports include only paid attendees in revenue.
- [ ] Endpoint tests.
- [ ] Frontend specs.

## 9. Cross-Layer Acceptance Criteria

Admin sets up a "Hands Balancing Workshop" with specialty instructor "Visiting Maya" at $400 base + $25 per student past 6:
- [ ] Materializing the workshop session snapshots `specialty_instructor_cost_id`.
- [ ] At 10 attendees: pay = $400 + 4×$25 = $500.
- [ ] Suggested member price (gold tier, 8 target) = $400/8 = $50 break-even.
- [ ] Suggested non-member price = $50 × 1.5 = $75 (with 50% margin).
- [ ] Cost vs revenue at 10 attendees (8 gold @ $50, 2 non-member @ $75): revenue = 8×$50 + 2×$75 = $550; cost $500; margin $50.

## 10. Open Questions

- **OQ-P12-1.** Should the pricing-assistant accept a target *profit margin* per tier instead of a flat "members break-even, non-members 50%"? Recommended: hard-coded for now; if admin wants more flexibility, add a `non_member_profit_margin_pct` secret and a "profit_margin_pct" input on the dialog. Defer until requested.
- **OQ-P12-2.** When the rate snapshot in `event_sessions.specialty_instructor_cost_id` references a cost row whose `price_schedule` is no longer active, payroll calculations should still use the snapshot. Add a regression test.

## 11. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 12.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 7 - Class Series and Workshops]].
- Will feed into: [[Classes Phase 16 - Stretch Items]] (PR-1..PR-4 payroll).
- Pricing infra: [[Payment Design Document]].
