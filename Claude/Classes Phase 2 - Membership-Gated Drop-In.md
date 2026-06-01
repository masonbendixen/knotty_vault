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

Classes Phase 2 - Membership-Gated Drop-In

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

> ⚠️ **Superseded modeling (2026-05-29):** the binary `products.is_membership_included` / `price_included_in_membership` introduced below is the **wrong abstraction**. Inclusion is per-permission, not a boolean (memberships grant permissions; classes accept a *set* of permissions; attendance/skill also grant permissions per SL-10/12). See [[Permission-based class access redesign]] for the corrective plan. The tier-pricing path (M-5 via `product_prices`) is unaffected; only the binary inclusion modeling changes.

## Implementation Status (2026-05-29)

**Done — the membership/pricing surface + booking guard (the §10 acceptance pillars), with tests at every layer. Frontend verified (367 specs pass, `ng build` clean); C++ written to convention for the user to build.**

Delivered:
- **DB:** `products.is_membership_included BOOLEAN NOT NULL DEFAULT FALSE` (constant + DDL in `products.{h,cpp}`).
- **Business logic §4.1:** `Payment::CatalogHelper::ResolveBestPriceForPerson` + `PersonalizedPrice` (inclusion-wins, then lowest qualifying tier/public price, else `NO_TIER_MATCH`) + `PersonHoldsPermission`. 5 new `catalog_helper_test` cases.
- **Class catalog integration:** `ClassCatalogHelper::GetClassDetail` now sets real `isIncludedInMembership` + tier price + `isAvailable`; `GetClassesVisibleToPerson` hides deliberately members-only classes the viewer can't access (precise filter via `productIsMembershipIncluded` so unpriced classes keep prior visibility — no Phase-1 test breakage). 3 new `class_catalog_helper_test` cases.
- **Booking guard §4.3:** `BookingHelper::BookEvent` rejects a membership-included class session with `NO_ADVANCE_BOOKING_REQUIRED` (maps to 400 via the endpoint's default branch — OQ-P2-2). 2 new `booking_helper_test` cases (reject + paid-class-still-books).
- **KVT §4.4:** `ClassDetailToKeyValueTable` surfaces `price_is_available`; `price_included_in_membership` is now real. KVT tests added.
- **Endpoints §5.1 (partial):** `/api/classes/<id>` already flows the resolved price via `ClassDetailToKeyValueTable`; `book_event` maps the guard to 400.
- **Frontend §6.1:** `ClassDetail.price_is_available` added; `ServerAccessNetwork.getClassDetail` normalizes the boolean-ish price flags (the KVT→JSON layer emits `"true"/"false"` strings — also fixes a latent Phase-1 truthy-string bug); `class-detail` shows included / tier-price+no-refund-note / members-only via `pricingState`. Component spec + mock + mock.spec updated.
- **Admin §7:** friendly name + bool edit-type for `is_membership_included`.

> **✅ Superseded by [[Permission-based class access redesign]] §3 (built 2026-05-31). The binary inclusion modeling above was removed and replaced; the tier-pricing path stayed.** Concretely:
> - **DB §2.2 / Admin §7:** `products.is_membership_included` (constant, DDL, friendly name, bool edit-type) **removed**. Inclusion is now declared per-class via `class_requirement_groups` + `class_requirement_group_literals` (CNF), with a `permission_implications` hierarchy.
> - **Business logic §4.1:** `ResolveBestPriceForPerson` / `PersonalizedPrice` no longer carry `isIncluded` / `productIsMembershipIncluded`; `PersonHoldsPermission` removed. `GetEffectivePermissionIds` is now closure-expanded and the new `Scheduling::ClassAccessHelper` is the single access gate.
> - **Class catalog §4.x:** `GetClassDetail` / `GetClassesVisibleToPerson` derive inclusion from `ClassAccessHelper.CheckAccess` + `classes.kind` (recurring + passes gate ⇒ included, no drop-in price; workshop/series + passes ⇒ tier price; fails ⇒ members-only/hidden).
> - **Booking guard §4.3:** `NO_ADVANCE_BOOKING_REQUIRED` now fires on *recurring kind + viewer passes the gate*, not the flag.
> - **Frontend §6.1:** unchanged at runtime — `ClassDetail.price_included_in_membership` kept its name but is now a server-derived "included for this viewer" flag (redesign §3.3), not a product flag. The `is_membership_included` test fixtures were replaced with requirement-group fixtures (redesign §3.4).
> - **Net:** the M-5 tier-pricing path (per-permission `product_prices`, lowest-wins) is untouched; only the binary inclusion bit changed. OQ-P2-3 (drop `classes.required_permission_id`) is moot — access lives in requirement groups, not a denormalized column.

Deferred (documented, not started — each carries either regression risk to non-class flows or is a larger surface):
- **§4.2 event-session visibility projection** — surfacing class metadata on `GetVisibleEventSessions` needs the derived-session rewrite (`GetDerivedSessionsForRange`); large. The class catalog already covers the class visibility/pricing surface.
- **§4.3 cancel no-refund / admin-cancel refund split** — must be scoped to *class* bookings only; changing `CancelBooking`/`SessionCancellationHelper` globally would regress existing event refunds. Needs care + dedicated tests.
- **§5.2 `GET /api/me/visible_classes`** — new endpoint.
- ~~**§2.2 `classes.required_permission_id`, §3.1 `GetProductByClassId`, §3.2 `GetActivePricesForProduct`** — intentionally skipped~~ → **resolved/closed 2026-06-01 (see §3):** pricing/visibility resolve through the active `class_instances` product + `GetBestProductPriceByProductSchedulePermissions`; the skipped column/methods are obsolete under the access redesign. §3 is done (verification + new round-trip tests).
- **Frontend §6.2 calendar chips, §6.3 my-bookings class chrome, §6.4 `cancellation-policy.component`, §6.5 `getVisibleClasses`** — depend on §4.2/§5.2/cancel work above.

Open questions — **all resolved (2026-06-01), see §11.** OQ-P2-3 (drop `classes.required_permission_id`) was confirmed and is moot under the access redesign.

## Phase Summary

**Must-have.** Members see classes their tier includes; non-members see only workshops / series / intro workshop (per P-1 and M-6). For recurring class attendance, the user just shows up — no advance booking is created. For workshops / series / intro workshop / guest passes (Phases 7 + future), the booking flow uses the existing `BookEvent` infrastructure but at zero or paid tier price.

**Prerequisites:**
- Phase 1 complete (classes catalog + the three-level `classes` → `class_instances` → `class_schedules` → `class_schedule_slots` model + lazy-derived `event_sessions`).
- Existing payment / product / product_prices / cancellation_policies / cancellation_policy_windows infrastructure \([[Payment Design Document]], [[Event Polish- Scheduling Should Have Items]]\).

### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]]
> Phase 1 was redesigned to a three-level model with **lazy session derivation** (no materialization). For Phase 2 this means:
> - **`product_id` is on `class_instances`, not on a flat `class_schedule`.** Per-tier pricing / visibility / booking permissions / cancellation policy / advance windows all resolve from the active instance's product. Where this doc says "the `class_schedule`'s product", read "the active `class_instances` row's product".
> - **The catalog visibility query returns DERIVED sessions, not pre-materialized rows.** `GetVisibleEventSessions` (or its class-aware successor) must call `ClassScheduleHelper::GetDerivedSessionsForRange` and surface `class_id` / `class_name` / `photo_url` from the derivation, left-joining any persisted `event_sessions` rows. A future occurrence with no persisted row is normal.
> - **Booking a paid offering ensures the `event_sessions` row first.** The `BookEvent` path for workshops / series / intro / guest passes must call `ClassScheduleHelper::EnsureSessionExists(slotId, occurrenceDateUs)` to create the row before attaching the booking (recording trigger #4). Membership-included recurring attendance still creates nothing until staff check-in (Phase 8).
> - **Room-conflict / capacity checks** sum derived + persisted overlapping sessions (P-4) via the Phase 1 `RoomOccupancyHelper`.

**Outcome:**
- `GET /api/visible_event_sessions` surfaces class metadata (class_id / class_name / photo_url) for each class instance.
- Member viewing the catalog sees included-with-membership flagged correctly and the right tier price for any paid offering.
- Non-member viewing the catalog sees only workshops / series / intro workshop.
- Booking flow correctly handles paid bookings (creates purchase + booking) AND, for the workshop / series / intro / guest-pass flows that pass through here, returns zero-price errors cleanly when the offering is membership-included (which should never reach the paid endpoint in the first place).
- Per P-6, user-initiated cancel does NOT refund — capacity released, waitlist advanced.

**Important scope note (parent doc resolution):** because recurring classes under membership do NOT create advance `booking` rows (templates are planning-only and bookings are created at staff check-in per Phase 8), this phase concentrates on the **visibility / pricing surface** AND on tightening the existing event-booking flow so workshops / series / intro can ride it correctly under P-1 and P-6.

## Layering & Conventions

Lowest layer first per CLAUDE.md:

1. `db_schema/` — only minor additions (no new tables).
2. `sql_util/table_helpers/` — small reads on existing helpers.
3. `business_logic/scheduling/` and `business_logic/payment/` — pricing resolution and visibility logic.
4. `endpoints/` — wiring through `visible_event_sessions`, `book_event`, `cancel_booking`.
5. Angular UI updates.
6. Admin metadata for any new flags.
7. Tests at every layer.

Reuse existing tables — no new schema except a small set of flag columns on `products` / `event_sessions`.

## 1. Pre-Coding Design Decisions

### 1.1 "Included" semantics (resolved per parent doc M-7)
- [x] Membership-included recurring class attendance is recorded by a `booking` row only at staff check-in time. NO purchase row, NO $0 purchase. `bookings.purchase_id` is nullable for these.
- [x] Workshops / series / intro / guest pass DO create paid bookings via the existing flow (`BookEvent`).
- [x] M-7 implementation: leave `BookEvent` as-is for the paid path; add a tier-aware visibility resolver for the catalog and a "free for you" badge for membership-included classes — but NOT a server-side $0 purchase.

### 1.2 Pricing resolution
- [x] For each class, pricing comes from the active `class_instances` row's `product`. The user's effective price is the lowest `product_prices.price_cents` across all `product_prices` rows whose `permission_id` is NULL or whose `permission_id` is held by the user (per M-5).
- [x] If user holds the inclusion permission (i.e. their membership tier's `grants_permission_id` covers the class's required permission), the offering is "included" — no price shown; "Show up to attend" CTA.
- [x] If user does NOT hold any matching permission AND there is no `permission_id IS NULL` price row, the offering is not available to them (recurring classes per P-1).

### 1.3 Refund model under P-6
- [x] User-initiated cancel of a paid booking → capacity freed + waitlist advanced; NO automatic refund. Staff may issue a voucher manually (BC-6 → Phase 7+ for tooling).
- [x] Admin-initiated session cancellation → full refund for paid bookings, no refund for membership-included (zero-money) bookings (SE-5).

## 2. Database Schema

### 2.1 Confirm reused tables
- [x] No new tables. Reuse:
  - `product_prices` (per-permission pricing — already supports M-2 for paid offerings).
  - `product_booking_windows` (per-permission `advance_days` — for AW-1).
  - `product_entitlement_rules.grants_permission_id` (membership tier → permission).
  - `cancellation_policies`, `cancellation_policy_windows`.

### 2.2 Small flag additions
- [~] `classes.required_permission_id BIGINT` NULL — **skipped (OQ-P2-3):** resolve via the active instance's product instead of denormalizing.
- [x] ~~`products.is_membership_included BOOLEAN NOT NULL DEFAULT FALSE`~~ — **REMOVED by [[Permission-based class access redesign]] §3 (2026-05-31).** Do not re-add this column. Inclusion is declared per-class via `class_requirement_groups` + `class_requirement_group_literals` and resolved by `Scheduling::ClassAccessHelper` (closure-aware). See the supersession note in the Implementation Status block.

### 2.3 Wire into DB init
- [x] `make_database_info.cpp` builds `products` from `MakeProductsTable` (updated); `create_database.cpp` creates it — the new column flows automatically.
- [x] `db_schema/products.h` has `kProductsIsMembershipIncluded`. *(No `classes.h` change — see §2.2.)*

## 3. Table Helpers ✅ DONE (2026-06-01)

> **No new production table-helper code was needed — every §3 capability already exists or was obsoleted by the [[Permission-based class access redesign]].** The work this section actually required (tier-price resolution + surfacing the access-permission columns) lives in helpers built earlier; the one genuine gap was a missing round-trip *test*, now added. Details per item below.

### 3.1 Extend `TableHelpers::Products`
- [x] **Surface `booking_permission_id` / `visibility_permission_id` in reads** — already satisfied: both columns exist on `products` and `GetProduct` / `GetActiveProducts` return them (`DbCrud::GetRow`/`GetActiveProducts` = `SELECT *`). **Added round-trip tests** in `products_test.cpp` (`GetProductSurfacesPermissionColumns`, `GetProductPermissionColumnsAbsentWhenNull`, `GetActiveProductsSurfacesPermissionColumns`).
- [x] ~~Surface `is_membership_included`~~ — **column removed** by the redesign; do not re-add. Class access is decided by the requirement-group gate (`ClassAccessHelper`), not a product flag.
- [x] ~~Add `GetProductByClassId`~~ — **obsolete under the three-level model.** A class has no single product; the product lives on the active `class_instances` row. Callers resolve it via `TableHelpers::ClassInstances::GetActiveInstance(classId, atUs).product_id` (Phase 1), which `ClassCatalogHelper` already uses. (Confirms the Implementation-Status "intentionally skipped" note.)
- [x] Tests for column round-trips — **done** (the three tests above).

### 3.2 Extend `TableHelpers::ProductPrices`
- [x] ~~Add `GetActivePricesForProduct(productId, asOfUs)`~~ — **already covered, in better form.** Tier-price resolution uses `ProductPrices::GetBestProductPriceByProductSchedulePermissions(productId, scheduleId, permissionIds, variantId)` (lowest qualifying tier, NULL-permission fallback, variant-scoped), with the active schedule resolved by `CatalogHelper::GetActivePriceScheduleId`. That path is exercised by `ResolveBestPriceForPerson` and already has thorough `product_prices_test` coverage (basic lowest-wins, empty-list fallback, permission fallback, variant scoping). A separate "list all active tier prices" method has **no consumer** in Phase 2 (YAGNI; the catalog resolves the single best price) — not added.

### 3.3 Extend `TableHelpers::Classes`
- [x] ~~Surface `required_permission_id`~~ — **obsolete:** the column was permanently skipped (OQ-P2-3); access lives in `class_requirement_groups` / `class_requirement_group_literals`. `Classes::GetClass` (`SELECT *`) would surface any real column automatically; there is nothing to add.

## 4. Business Logic

> §4.2 (event-session visibility projection), the §4.3 cancel/admin-cancel refund split, and §4.4's `EventSessionInfoToKeyValueTable` class fields are **deferred** — see the Implementation Status block. §4.1, the §4.3 booking guard, and §4.4's `ClassDetail` price fields are **done**.

### 4.1 Pricing-resolution helper (extension of `CatalogHelper`)
- [x] In `business_logic/payment/catalog_helper.h/.cpp`: added `struct PersonalizedPrice` (renamed to avoid colliding with the existing `ResolvedPrice`) + `ResolveBestPriceForPerson(Transaction&, productId, personId)` (uses the currently-active price schedule, not an `asOfUs` arg) + `PersonHoldsPermission`. Tests added (included member, tier member vs public non-member, lowest-tier-wins, non-member-blocked, product-not-found). Algorithm:
    1. Load product → if `is_membership_included && user holds booking_permission_id` → `isIncluded=true`, priceCents=0.
    2. Load all active `product_prices` rows for the product at `asOfUs`.
    3. Filter to rows where `permission_id IS NULL` OR `permission_id` ∈ user's effective permission set.
    4. Pick `min(priceCents)` across that filtered set.
    5. If no rows match → `isAvailable=false`, `unavailableReason=NO_TIER_MATCH`.
  - Add tests for: included member, tier-priced member, non-member with non-member price, non-member with no non-member price (NOT_AVAILABLE).

### 4.2 Extend `EventSessionHelper::GetVisibleEventSessions`
- [ ] Already returns event sessions for the catalog; extend the projection to also surface class metadata when `class_id` is set:
  - `class_id`, `class_name`, `class_photo_url`, `is_membership_included` (resolved via product), the user's `ResolvedPrice`, and a `subtitle_label` such as `"Included with your Gold Membership"` / `"$25 / drop-in for Gold"` / `"Members only — upgrade to attend"`.
- [ ] When the request has no `personId` (anonymous), only the base `permission_id IS NULL` price is surfaced, and `is_membership_included` is left as a hint.
- [ ] Add a filter parameter `class_only` (bool) so the public `/classes/upcoming` page can request only class sessions.
- [ ] Tests covering: anonymous, member-with-included, member-with-paid-tier, non-member, no-matching-price.

### 4.3 Booking flow under P-1 / P-6
- [ ] **Membership-included recurring classes do NOT pass through `BookEvent`** at all in Phase 2 — there is no advance booking. Verify with a test that calling `POST /api/book_event/<sessionId>` on a membership-included recurring class returns either:
  - (a) 400 with `errorCode=NO_ADVANCE_BOOKING_REQUIRED` (preferred — surfaces the convention), OR
  - (b) silently succeeds and creates a zero-money booking (not preferred, conflicts with the "booking exists only at check-in" rule).
  - [x] Picked (a): `BookingHelper::BookEvent` rejects with `NO_ADVANCE_BOOKING_REQUIRED` when the session has a `class_id` and `ResolveBestPriceForPerson(...).isIncluded`. Tests added.
- [x] For workshops / series / intro (paid path), `BookEvent` runs as today — verified by `BookEventAllowsPaidClassSession` (a class session that is *not* membership-included books normally).
- [ ] **User-initiated cancellation** (`POST /api/cancel_booking/<id>`):
  - For paid bookings: free the capacity, advance the waitlist (existing `BookingHelper::CancelBooking`), but DO NOT issue a refund (per P-6). Return **`refund: { issued: false, reason: "non_refundable" }`** (OQ-P2-1) so the UI knows no refund is happening; the user can see the cancellation-policy display (BC-5) BEFORE clicking cancel.
  - For zero-money bookings (shouldn't exist post-Phase 2, but defensive): just free capacity.
- [ ] **Admin-initiated session cancellation** (`SessionCancellationHelper`):
  - Full refund for paid bookings (existing behavior).
  - No refund for zero-money bookings.

### 4.4 KeyValueTable conversions
- [~] `EventSessionInfoToKeyValueTable` `class_*` / `resolved_price.*` fields — deferred (tied to §4.2).
- [x] `ClassDetailToKeyValueTable` now surfaces `price_is_available` and the real `price_included_in_membership`. Tests in `scheduling_key_value_table_test.cpp`.

## 5. Endpoints

### 5.1 Reuse + extend existing endpoints
- [ ] `GET /api/visible_event_sessions?placement=upcoming|home_page` — already exists. Verify it correctly serializes the new class fields when `class_id` is set. Add endpoint test cases for member-with-included and non-member.
- [ ] `POST /api/book_event/<id>` — already exists. Add a guard at the top of the handler / inside `BookingHelper::BookEvent` to reject membership-included class bookings with the new `NO_ADVANCE_BOOKING_REQUIRED` error. Add endpoint test.
- [ ] `POST /api/cancel_booking/<id>` — already exists. Adjust the response payload to return **`refund: { issued: false, reason: "non_refundable" }`** (OQ-P2-1) per P-6. Add endpoint test asserting no refund was triggered (test mail helper captures no `BookingCancellationMail` with a refund line).
- [ ] `GET /api/classes` and `GET /api/classes/<id>` (introduced in Phase 1) — extend to use `ResolveBestPriceForPerson`. Add endpoint tests.

### 5.2 New visibility endpoint (helpful for the user homepage)
- [ ] `GET /api/me/visible_classes` (logged in only) — returns the union of:
  - All `class_schedules` (active, with at least one materialized future session) where the user holds the booking permission (membership-included).
  - All paid offerings (series / workshops / intro / guest-pass-flagged) visible to them at their tier price.
- [ ] Lightweight payload — drives the homepage today-classes feed + the "My Schedule" eligibility grid in Phase 5. Reuses `ClassCatalogHelper::GetClassesVisibleToPerson` plus an attendance-template join (stubbed in Phase 2; fully wired in Phase 5).

## 6. Frontend

### 6.1 Class detail page (extension)
- [x] Shows resolved price / "Included with your membership" / "Members only" via the `pricingState` getter, driven by the API's `price_*` fields (normalized to real booleans in `ServerAccessNetwork.getClassDetail`).
- [x] Included recurring classes show "Just show up — staff will check you in" (no Book CTA).
- [x] Paid offerings show the resolved tier price + the BC-5 "No refunds — staff may issue a voucher case-by-case" note inline. *(A dedicated "Reserve" button is deferred until the paid class-booking flow is wired from class-detail — Phase 7.)*
- [x] Component spec updated (included / paid+no-refund / members-only).

### 6.2 Calendar view labels
- [ ] In `pages/calendar/calendar-event/calendar-event.component.ts`, color-code or chip-label sessions by status: "Included" / "Tier-priced ($25)" / "Members only".
- [ ] Spec.

### 6.3 My-bookings page
- [ ] Confirm class instances render with class info (photo + class name) rather than generic "event" chrome.
- [ ] For class bookings, show "No refund — contact staff if you need a voucher" instead of the existing refund-window UI.
- [ ] Spec.

### 6.4 BC-5 cancellation-policy display
- [ ] Add a `cancellation-policy.component.ts` that renders the refund-window breakdown OR an explicit "non-refundable" notice from a `CancellationPolicyInfo` input.
- [ ] Use it on the class-detail page and at the cancel-confirmation dialog.
- [ ] Spec.

### 6.5 `ServerAccess` extensions
- [ ] `getVisibleClasses()` for the new endpoint.
- [ ] Update `ServerAccess.mock.spec.ts`.

## 7. Admin Metadata

- [~] Verify admin UI for `product_prices` tier pricing end-to-end — not re-verified this pass (the existing product/pricing admin already supports per-permission `product_prices`).
- [x] ~~Friendly name + bool edit-type for `products.is_membership_included`~~ — **REMOVED with the column** ([[Permission-based class access redesign]] §3, 2026-05-31). The redesign instead registered `class_requirement_groups` + `class_requirement_group_literals` (nested admin metadata under `classes`, gated by `manage_class_schedule`) and `permission_implications` (top-level, admin-only) — the friendly authoring UI for these is the §6.6 Phase-1 follow-up.

## 8. Permissions

- [x] No new permissions needed in this phase. The existing `manage_products` covers product / pricing edits; `manage_class_schedule` (introduced in Phase 1) covers schedules.

## 9. Tests-Required Summary

- [x] Table helper tests for the access-permission column round-trips — `products_test.cpp` (`GetProductSurfacesPermissionColumns` / `...AbsentWhenNull` / `GetActiveProductsSurfacesPermissionColumns`); tier-price resolution already covered by `product_prices_test.cpp` (`GetBestProductPriceByProductSchedulePermissions*`).
- [x] `catalog_helper_test.cpp`: `ResolveBestPriceForPerson` covers included→tier-price, lowest-wins, non-member-with-price, non-member-blocked, product-not-found (done in the §4.1 work + the access-redesign closure test).
- [ ] Updated `event_session_helper_test.cpp`: visible-event-sessions surfaces class metadata + resolved price.
- [ ] Updated `booking_helper_test.cpp`: reject `NO_ADVANCE_BOOKING_REQUIRED`, paid booking still works.
- [ ] Updated `event_session_cancellation_helper_test.cpp` (or whichever owns admin-cancel): full refund for paid, no refund for zero-money.
- [ ] Endpoint tests for `visible_event_sessions`, `book_event` (reject path), `cancel_booking` (no-refund path), `classes`, `classes/<id>`, `me/visible_classes`.
- [ ] Frontend specs for class-detail (included vs paid), calendar chip rendering, my-bookings class chrome, cancellation-policy component, mock service.

## 10. Cross-Layer Acceptance Criteria

A new user, given:
- An active "Gold Membership" entitlement that grants `gold_member` permission.
- A recurring class "Vinyasa Flow" with a single access requirement group holding the `gold_member` permission literal (closure-expanded, so platinum also satisfies it). *(Post-redesign: was `products.is_membership_included=true` + `booking_permission_id = gold_member` — see the supersession note.)*
- A series "6-Week Aerial 101" whose product has tier prices: gold $120, silver $180, non-member $300.

Should be able to:
- [x] See "Vinyasa Flow" with an "Included with your membership" label *(via `GetClassDetail` / `GetClassesVisibleToPerson`; covered by `class_catalog_helper_test`)*.
- [x] See "6-Week Aerial 101" at the gold tier price *(tier resolution via `ResolveBestPriceForPerson`; `catalog_helper_test`)*.
- [x] Be prevented (400) from `book_event` on a Vinyasa Flow session (`NO_ADVANCE_BOOKING_REQUIRED`) *(`booking_helper_test` + endpoint default→400)*.
- [x] Successfully book the aerial series at its tier price *(existing paid `BookEvent` path; `BookEventAllowsPaidClassSession` confirms the guard doesn't block it)*.

A non-member should:
- [x] NOT see "Vinyasa Flow" in the catalog *(`GetClassesVisibleToPersonHidesMembersOnlyFromNonMember`)*.
- [x] See "6-Week Aerial 101" at the non-member price *(`ResolveBestPriceTierForMemberPublicForNonMember`)*.

## 11. Open Questions

**All resolved (2026-06-01).** No open questions remain for Phase 2.

- **OQ-P2-1.** ✅ **RESOLVED (Mason → recommendation):** user-initiated cancel inside a non-refund window returns **200 + `refund: { issued: false, reason: "non_refundable" }`** so the UI can render the right message. *(Implementation note: this shape is the contract for the deferred §4.3 cancel path + §5.1 `cancel_booking`.)*
	- Mason- I'll go with your recommendation.
- **OQ-P2-2.** ✅ **RESOLVED (Mason → recommendation):** `NO_ADVANCE_BOOKING_REQUIRED` is a **400** (ValidationError) — matches the status-code policy in memory `error_response_status_codes.md`. *(Already implemented: the booking guard maps to 400 via the endpoint's default branch.)*
	- Mason- I'll go with your recommendation.
- **OQ-P2-3.** ✅ **RESOLVED / moot:** drop `classes.required_permission_id` (do not denormalize). Confirmed — and the [[Permission-based class access redesign]] makes it moot: access lives in `class_requirement_groups` / `class_requirement_group_literals`, resolved by `ClassAccessHelper`; pricing/visibility resolve through the active `class_instances` product. The §2.2 / §3.3 column is permanently skipped.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 2.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]].
- Successors: [[Classes Phase 5 - Attendance Templates]], [[Classes Phase 7 - Class Series and Workshops]], [[Classes Phase 8 - Staff Check-in]].
- Pricing context: [[Payment Design Document]], [[Product browsing and quoting endpoints]].
