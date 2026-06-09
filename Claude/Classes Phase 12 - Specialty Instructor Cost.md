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

## 2. Database Schema

### 2.1 `specialty_instructor_costs` table
- [ ] `db_schema/specialty_instructor_costs.h/.cpp`:
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
- [ ] Unique on `(class_instance_id, class_schedule_slot_id, instructor_person_id, price_schedule_id)`.

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

### 2.3 Ensure-time rate snapshot
- [ ] Add `event_sessions.specialty_instructor_cost_id BIGINT` NULL `REFERENCES specialty_instructor_costs(id)` — set when the occurrence row is ensured (no materialization) so payroll later reads the snapshot rather than the live rate.

### 2.4 Ensure-time hook
- [ ] In `ClassScheduleHelper::EnsureSessionExists` (Phase 1), when creating a new occurrence row, look up the active `specialty_instructor_costs` row for the occurrence's `class_instance_id` (+ matching `class_schedule_slot_id` if a per-slot override exists), the slot's `instructor_person_id`, and `asOfUs` via the active `price_schedule`. If found, set `event_sessions.specialty_instructor_cost_id` on the new row.
- [ ] If no cost row, leave NULL (non-specialty instructors). Idempotent: ensuring an existing row doesn't re-stamp.

### 2.5 Wire into DB init
- [ ] `make_database_info.cpp` + `create_database.cpp`.

### 2.6 Config secret (resolved OQ-P12-1)
- [ ] Seed `non_member_profit_margin_pct` in the `config_secrets` defaults (in `create_database.cpp`), default `50` (percent). This is the fallback the pricing assistant uses when the Suggest-pricing request supplies no per-tier / request-level margin override. Read via `SecretsHelper` in §4.2 (no secrets lookup inside table helpers).

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
- [ ] `struct PricingSuggestion { int64_t permissionId; std::string permissionName; int64_t suggestedPriceCents; double appliedMarginPct; std::string rationale; }`. (`appliedMarginPct` echoes the margin actually used for that tier so the UI can show it.)
- [ ] `struct PricingSuggestionRequest { int64_t classScheduleId; int64_t targetAttendees; std::vector<int64_t> allowedPermissionIds; std::optional<double> profitMarginPct; std::map<int64_t, double> perTierMarginPct; }`. **Margin is configurable (resolved OQ-P12-1):** `perTierMarginPct[permissionId]` wins if present, else the request-level `profitMarginPct`, else the `non_member_profit_margin_pct` secret default (§2.6). Member tiers ignore margin (they break even).
- [ ] `std::vector<PricingSuggestion> SuggestPricesForBreakeven(Transaction&, const PricingSuggestionRequest&, const SecretsHelper&)`:
  1. Resolve the default margin from the `non_member_profit_margin_pct` secret (default 50%).
  2. Load active specialty cost for the schedule.
  3. Compute total cost at the target attendee count: `totalCost = ComputeInstructorPayCents(at attendees=target)`.
  4. For each tier in `allowedPermissionIds`:
     - If `tier` is a member tier (you can detect this via the permission's existing-membership-grant linkage; if the user holds an active membership granting that permission, they should "cover cost" only — break-even price = `totalCost / target`; `appliedMarginPct = 0`).
     - If `tier` is a non-member tier: resolve the effective margin (`perTierMarginPct` → request `profitMarginPct` → secret default) → `suggestedPriceCents = (totalCost / target) × (1 + margin)`; record `appliedMarginPct`.
  5. Return suggestions with rationale strings that name the margin used ("Break-even for Gold tier at 8 attendees: $X" / "Non-member at 8 attendees with 50% margin: $Y").

### 4.3 Cost / revenue report
- [ ] `struct SessionCostRevenueReport { int64_t eventSessionId; int64_t attendeeCount; int64_t paidAttendeeCount; int64_t instructorPayCents; int64_t revenueCents; int64_t marginCents; }`.
- [ ] `SessionCostRevenueReport GetSessionCostRevenue(Transaction&, int64_t eventSessionId)`:
  1. Compute pay via `ComputeInstructorPayCents`.
  2. Query paid revenue: SUM of `purchase_items.line_total_cents` across `bookings` for this session where `purchase_id IS NOT NULL`.
  3. Return.

### 4.4 KeyValueTable conversions
- [ ] `PricingSuggestionToKeyValueTable`, `SessionCostRevenueReportToKeyValueTable`.

## 5. Endpoints

- [ ] `POST /api/admin/session/<id>/suggest_pricing` body `{ target_attendees, tiers: [permission_id...], profit_margin_pct?, per_tier_margin_pct?: { permission_id: pct } }` → list of suggestions (each echoing `applied_margin_pct`). The margin fields are optional; when omitted the server falls back to the `non_member_profit_margin_pct` secret (resolved OQ-P12-1). Permission `manage_class_schedule`. Endpoint test (incl. a request that overrides the margin and asserts the suggested price reflects it, and one that omits it and falls back to the secret default).
- [ ] `GET /api/admin/session/<id>/cost_revenue` → `SessionCostRevenueReport`. Permission `manage_class_schedule`. Endpoint test.

#### Specialty-cost authoring endpoints (bespoke — NOT generic CRUD / Manage Data)
> **Manage Data is debug-only** (memory `feedback_manage_data_is_debug_only.md`). Hiring a specialty instructor and setting their pay rate is a core admin workflow, so it gets a bespoke `/manage` flow with dedicated endpoints — never "go to Manage Data and add a row." The endpoints below are thin (parse → `SpecialtyCostHelper` / table helper → KeyValueTable→JSON) per the layering rule.
- [ ] `GET /api/admin/class_instance/<id>/specialty_costs` → list the specialty-cost rows for a run (with resolved instructor name + active `price_schedule` window). Permission `manage_class_schedule`. Endpoint test.
- [ ] `POST /api/admin/specialty_cost` body `{ class_instance_id, class_schedule_slot_id?, instructor_person_id, base_rate_cents, per_student_bonus_cents, bonus_threshold_count?, notes }` — creates the cost row (and its backing `price_schedule` row, mirroring how product pricing snapshots roll forward). Returns the created row. Permission `manage_class_schedule`. Endpoint test (403 / 400-missing-field / 200+persist).
- [ ] `PUT /api/admin/specialty_cost/<id>` — opens a new `price_schedule` window for a rate change (roll-forward, per SI-6 / P-2) rather than mutating history. Endpoint test.
- [ ] `DELETE /api/admin/specialty_cost/<id>` — soft-delete / close the cost. Endpoint test.
- [ ] `GET /api/admin/instructor/<id>/class_preferences` + `POST /api/admin/instructor_class_preference` (body `{ instructor_person_id, class_id, min_attendees?, max_attendees?, notes }`) + `PUT`/`DELETE` — bespoke CRUD for `instructor_class_preferences`. Permission `manage_class_schedule`. Endpoint tests at each verb.
- [ ] These authoring endpoints live in `endpoints/` as thin handlers delegating to `SpecialtyCostHelper` + the two table helpers; conversions go in `scheduling_key_value_table.*`. (Implementation note: they MAY instead be served by the generic CRUD REST endpoints called *from the bespoke §6 forms* — the established Phase 1 `class-requirements-editor` pattern — but the **authoring UI must be the bespoke `/manage` surface below, never the Manage Data table editor**.)

## 6. Frontend

### 6.1 Admin session detail extension
- [ ] On the admin session-detail page (likely under `portal/manage/event-session-detail/`), add:
  - "Specialty cost" panel: shows rate + bonus + computed pay at current attendee count.
  - "Cost vs revenue" panel: revenue, cost, margin.
  - "Suggest pricing" button → opens a dialog with target-attendees input + allowed-tiers multiselect + a **profit-margin input** (pre-filled from the `non_member_profit_margin_pct` secret default, editable; optional per-tier margin override) → calls the suggest-pricing endpoint → displays table of suggestions per tier, showing the `applied_margin_pct` used for each (resolved OQ-P12-1).
- [ ] Specs (incl. editing the margin re-queries and the per-tier suggestion price changes accordingly).

### 6.2 Specialty-cost authoring on the run (bespoke Manage UI — NOT Manage Data)
> Authoring specialty-instructor pay is done in the dedicated `/manage` portal, mirroring the Phase 7 `series-run-form-dialog` on **Manage Products ▸ Class Schedules**. The Manage Data generic editor is debug-only and must never be the path an admin uses to set up a specialty instructor's rate.
- [ ] On **Manage Products ▸ Class Schedules**, when a run (`class_instances`) is selected, add a **"Specialty instructor cost"** section/dialog: instructor picker (people filtered to the `instructor` permission), base-rate + per-student-bonus (money inputs), optional bonus-threshold, optional per-slot scope (the slots of the run), notes. Save → `POST /api/admin/specialty_cost`; edit → `PUT` (roll-forward new rate); remove → `DELETE`. List existing cost rows for the run with their active window.
- [ ] New `specialty-cost-form-dialog.component.*/.spec.ts` under `manage/class-schedules/dialogs/` (sibling of `series-run-form-dialog`). Spec covers defaults, money formatting, per-slot scope toggle, validation paths, and the normalized save result. Use money inputs + date pickers per `feedback_date_time_pickers.md`; mat-card border + RouterTestingModule per `feedback_account_page_layout.md`.

### 6.3 Instructor class-preferences authoring (bespoke Manage UI — NOT Manage Data)
- [ ] On **Manage ▸ Instructors** (`manage/instructors/instructors-admin.component`), add a **"Class preferences"** section per instructor: a small editable table of `(class, min_attendees, max_attendees, notes)` rows backed by the §5 `instructor_class_preference` endpoints (add/edit/delete inline). This replaces the generic-CRUD path the doc previously assumed.
- [ ] Extend `instructors-admin.component.spec.ts` (or a new `instructor-class-preferences` sub-component spec) covering add/edit/remove + validation.

### 6.4 `ServerAccess` extensions
- [ ] `suggestPricingForSession(sessionId, targetAttendees, tiers, profitMarginPct?, perTierMarginPct?)`, `getSessionCostRevenue(sessionId)`.
- [ ] `getSpecialtyCostsForRun(classInstanceId)`, `createSpecialtyCost(req)`, `updateSpecialtyCost(id, req)`, `deleteSpecialtyCost(id)`.
- [ ] `getInstructorClassPreferences(instructorPersonId)`, `createInstructorClassPreference(req)`, `updateInstructorClassPreference(id, req)`, `deleteInstructorClassPreference(id)`.
- [ ] Update `ServerAccess.mock.spec.ts` (per memory `feedback_always_test.md` — every new `ServerAccess` method needs a mock-spec case).

### 6.5 Types
- [ ] `specialty-cost.types.ts`: `PricingSuggestion` (incl. `appliedMarginPct`), `PricingSuggestionRequest` (incl. optional `profitMarginPct` + `perTierMarginPct`), `SessionCostRevenueReport`, `SpecialtyCost`, `CreateSpecialtyCostRequest`, `InstructorClassPreference`, `CreateInstructorClassPreferenceRequest`.

## 7. Admin Metadata (debug-only fallback — NOT the authoring workflow)

> Per memory `feedback_manage_data_is_debug_only.md`: the real authoring workflow for both tables is the bespoke `/manage` UI in §6. The Manage Data registration below is a **debug / raw-data escape hatch only** — it must never be the path an admin uses to set up a specialty cost or instructor preference. Registering the tables is still required so the generic CRUD endpoints (if the bespoke forms reuse them) accept the table and so the rows are inspectable.
- [ ] `specialty_instructor_costs` → `admin_top_level_tables`. Permission `manage_class_schedule`. (Inspection / debug only.)
- [ ] `instructor_class_preferences` → `admin_top_level_tables`. Permission `manage_class_schedule`. (Inspection / debug only.)
- [ ] Friendly names, column data info, display templates — so the rows render legibly in the debug editor; money columns get the cents edit type.

## 8. Tests-Required Summary

- [ ] Table helper tests for both new tables.
- [ ] `specialty_cost_helper_test.cpp`:
  - Pay computation with / without bonus, with / without threshold.
  - Pricing suggestions cover break-even for member, profit margin for non-member.
  - **Configurable margin (resolved OQ-P12-1):** request-level `profitMarginPct` override changes the non-member suggestion; per-tier `perTierMarginPct` wins over the request-level value; omitting both falls back to the `non_member_profit_margin_pct` secret; member tiers ignore margin. `appliedMarginPct` is echoed per suggestion.
  - **Snapshot authoritative (resolved OQ-P12-2):** `ComputeInstructorPayCents` uses the snapshotted cost row even when its `price_schedule` window has closed / a newer rate is active — regression test asserts pay reflects the old snapshot rate, not the live one.
  - Cost vs revenue reports include only paid attendees in revenue.
- [ ] Endpoint tests — suggest_pricing, cost_revenue, AND the bespoke authoring endpoints (`specialty_cost` GET/POST/PUT/DELETE, `instructor_class_preference` GET/POST/PUT/DELETE): 403 / validation / persist at each verb.
- [ ] Frontend specs — session-detail panels, **`specialty-cost-form-dialog` spec**, **instructor class-preferences spec**, and the `ServerAccess.mock.spec.ts` cases for every new mock method.

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
