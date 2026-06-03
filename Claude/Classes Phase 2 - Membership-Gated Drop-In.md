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
- **§4.3 cancel refund split** — **user-cancel P-6 DONE 2026-06-01** (class bookings non-refundable, `refundReason="non_refundable"`, free-cancel override still refunds, non-class unchanged). The **zero-money (NULL `purchase_id`) branch is deferred to Phase 8** — blocked on M-7 (purchase_id is currently NOT NULL, so the row can't exist/be tested).
- **§4.2 event-session visibility projection** — **partial (§4.2):** `class_id`/`class_name` now surfaced; the per-viewer `subtitle_label` + derived-session rewrite is **descoped** (obsolete `is_membership_included`; large; redundant with the class catalog).
- **§5.2 `GET /api/me/visible_classes`** — new endpoint (still deferred).
- ~~**§2.2 `classes.required_permission_id`, §3.1 `GetProductByClassId`, §3.2 `GetActivePricesForProduct`** — intentionally skipped~~ → **resolved/closed 2026-06-01 (see §3):** pricing/visibility resolve through the active `class_instances` product + `GetBestProductPriceByProductSchedulePermissions`; the skipped column/methods are obsolete under the access redesign. §3 is done (verification + new round-trip tests).
- **Frontend §6** — **§6.1 class-detail DONE; §6.3 my-bookings cancel-messaging DONE (2026-06-01, class-aware non-refundable).** Remaining deferred: §6.2 calendar chips (calendar is mock-only, no live session data), §6.3 class photo/name on the booking *card* (needs a backend `UserBookingInfo` extension), §6.4 shared `cancellation-policy.component` (display already inline in both consumers — optional refactor), §6.5 `getVisibleClasses` (blocked on §5.2 → Phase 5).

Open questions — **all resolved (2026-06-01), see §12.** OQ-P2-3 (drop `classes.required_permission_id`) was confirmed and is moot under the access redesign.

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

> **Status (2026-06-01):** §4.1 (pricing), the §4.3 booking guard, the **§4.3 cancel/admin-cancel refund split (P-6 / SE-5)**, and §4.4 (`ClassDetail` price fields + `EventSessionInfoToKeyValueTable` `class_*` fields) are **DONE**. §4.2 is **partially done** — the class-metadata surfacing (`class_id` / `class_name`) landed; the per-viewer `subtitle_label` / derived-session rewrite is **descoped** (obsolete `is_membership_included`, large, redundant with the class catalog — see the note in §4.2).

### 4.1 Pricing-resolution helper (extension of `CatalogHelper`)
- [x] In `business_logic/payment/catalog_helper.h/.cpp`: added `struct PersonalizedPrice` (renamed to avoid colliding with the existing `ResolvedPrice`) + `ResolveBestPriceForPerson(Transaction&, productId, personId)` (uses the currently-active price schedule, not an `asOfUs` arg) + `PersonHoldsPermission`. Tests added (included member, tier member vs public non-member, lowest-tier-wins, non-member-blocked, product-not-found). Algorithm:
    1. Load product → if `is_membership_included && user holds booking_permission_id` → `isIncluded=true`, priceCents=0.
    2. Load all active `product_prices` rows for the product at `asOfUs`.
    3. Filter to rows where `permission_id IS NULL` OR `permission_id` ∈ user's effective permission set.
    4. Pick `min(priceCents)` across that filtered set.
    5. If no rows match → `isAvailable=false`, `unavailableReason=NO_TIER_MATCH`.
  - Add tests for: included member, tier-priced member, non-member with non-member price, non-member with no non-member price (NOT_AVAILABLE).

### 4.2 Extend `EventSessionHelper::GetVisibleEventSessions` — PARTIAL (2026-06-01)
- [x] **Surface `class_id` + `class_name`** when the session belongs to a class. The three visibility queries (`upcoming` / `home_page` / by-id) now `LEFT JOIN classes c ON es.class_id = c.id` and select `c.name AS class_name`; `EventSessionInfo` gained `classId` / `className`, parsed in `EventSessionInfoFromRow`. Tests in `event_session_helper_test.cpp` (`VisibleSessionSurfacesClassMetadata`, `StandaloneSessionHasNoClassMetadata`).
- [~] **Descoped: `is_membership_included` / `subtitle_label` / derived-session rewrite.** `is_membership_included` is an obsolete product flag (removed by the redesign — inclusion is the requirement-group gate, not a product bit). The per-viewer subtitle + the "return DERIVED sessions via `GetDerivedSessionsForRange`" rewrite is large and **redundant with the class catalog** (`ClassCatalogHelper::GetClassesVisibleToPerson`/`GetClassDetail` already deliver per-viewer class visibility + pricing via the access gate). `canBook` + `amount_cents` already encode "members only" / tier price on the event-session feed. The subtitle is a presentation concern the deferred §6.2 calendar frontend can compute from `class_name` + `amount_cents` + `can_book`. Revisit only if a concrete calendar consumer needs more.
- [x] `class_photo_url` — not stored as a column; the frontend derives `/api/get_scaled_photo/classes/<class_id>/…` from the surfaced `class_id` (same pattern as elsewhere). No backend field needed.

### 4.3 Booking flow under P-1 / P-6
- [ ] **Membership-included recurring classes do NOT pass through `BookEvent`** at all in Phase 2 — there is no advance booking. Verify with a test that calling `POST /api/book_event/<sessionId>` on a membership-included recurring class returns either:
  - (a) 400 with `errorCode=NO_ADVANCE_BOOKING_REQUIRED` (preferred — surfaces the convention), OR
  - (b) silently succeeds and creates a zero-money booking (not preferred, conflicts with the "booking exists only at check-in" rule).
  - [x] Picked (a): `BookingHelper::BookEvent` rejects with `NO_ADVANCE_BOOKING_REQUIRED` when the session has a `class_id` and `ResolveBestPriceForPerson(...).isIncluded`. Tests added.
- [x] For workshops / series / intro (paid path), `BookEvent` runs as today — verified by `BookEventAllowsPaidClassSession` (a class session that is *not* membership-included books normally).
- [x] **User-initiated cancellation** (`BookingHelper::CancelBooking`) — DONE (2026-06-01):
  - A **class booking** (the session has `class_id`) takes the **non-refundable** path under P-6: capacity is freed + waitlist advances, but no money is returned. `CancelBookingResult.refundReason = "non_refundable"` so the endpoint can surface **`refund: { issued: false, reason: "non_refundable" }`** (OQ-P2-1). A studio-caused **free-cancel override** still refunds (100%). A standalone (non-class) event keeps the existing policy-based refund.
  - Tests: `booking_helper_test.cpp` — `CancelClassBookingIssuesNoRefundP6`, `CancelNonClassBookingHasNoNonRefundableReason`, `CancelClassBookingFreeCancelOverrideIsRefundable`.
- [~] **Zero-money (NULL `purchase_id`) bookings — deferred to Phase 8.** The doc's "defensive: just free capacity" handling is **blocked on M-7**: `bookings.purchase_id` (and `purchase_item_id`) are currently **NOT NULL** (`AddColumnForeignKeyRef`), so a zero-money booking cannot exist or be constructed in a test. Staff check-in (Phase 8) is what both makes `purchase_id` nullable and creates such bookings; the cancel/admin-cancel zero-money handling + its tests land there. Not implemented now (would be untestable defensive code against an impossible state).
- [~] **Admin-initiated session cancellation** (`SessionCancellationHelper::CancelSession`) — unchanged: paid bookings get the existing 100% refund. The zero-money branch is the same Phase-8 item above (no NULL-purchase booking can reach it under the current schema).

### 4.4 KeyValueTable conversions
- [x] `EventSessionInfoToKeyValueTable` now emits `class_id` + `class_name` when the session belongs to a class (omitted for standalone events). `resolved_price.*` is already emitted (`amount_cents` / `currency` / `pricing_permission_id` / `can_book`). Tests: `scheduling_key_value_table_test.cpp` (`EventSessionInfoSurfacesClassMetadata`, `EventSessionInfoOmitsClassMetadataForStandaloneEvent`). *(The obsolete `is_membership_included` field is intentionally not emitted — see §4.2.)*
- [x] `ClassDetailToKeyValueTable` surfaces `price_is_available` and the (now gate-derived) `price_included_in_membership`. Tests in `scheduling_key_value_table_test.cpp`.

## 5. Endpoints

### 5.1 Reuse + extend existing endpoints
- [x] `GET /api/visible_event_sessions?placement=upcoming|home_page` — now serializes `class_id` + `class_name` when the session belongs to a class (auto via the §4.2 projection + KVT). Endpoint test `visible_event_sessions_test.cpp::SurfacesClassMetadata`.
- [x] `POST /api/book_event/<id>` — the `BookingHelper::BookEvent` guard rejects a membership-included recurring class with `NO_ADVANCE_BOOKING_REQUIRED`; the endpoint now maps it to a **machine-detectable 400** (`ErrorResponse::Create("no_advance_booking_required", …)`) instead of a generic BadRequest. Test `book_event_test.cpp::BookRecurringClassReturnsNoAdvanceBookingRequired`.
- [x] `POST /api/cancel_booking/<id>` — response now includes the structured **`refund: { issued, reason, amount_cents, currency }`** object (OQ-P2-1); a class booking cancelled by the user is `issued:false, reason:"non_refundable"` (P-6). Flat `refund_*` fields kept for back-compat. Tests `cancel_booking_test.cpp::CancelClassBookingReturnsNonRefundable` + `CancelNonClassBookingRefundHasNoReason`.
- [x] `GET /api/classes` and `GET /api/classes/<id>` — **already viewer-aware (verified):** `/api/classes` calls `GetClassesVisibleToPerson(personId)` when logged in (`GetActiveClasses` for anonymous); `/api/classes/<id>` (`get_class_detail`) resolves price via `GetClassDetail` → `ResolveBestPriceForPerson` and emits the `price_*` fields. Price *resolution* is thoroughly covered at the helper layer (`catalog_helper_test`, `class_catalog_helper_test`). The lightweight catalog *list* intentionally carries no per-card price (price lives on the detail). No endpoint change needed.

### 5.2 New visibility endpoint (helpful for the user homepage) — DEFERRED to Phase 5
- [~] **`GET /api/me/visible_classes` — deferred (redundant now; distinct value is Phase 5).** Its core ("classes visible to this viewer — included + paid offerings at their tier") is **already served by `GET /api/classes`**, which calls `ClassCatalogHelper::GetClassesVisibleToPerson(personId)` for a logged-in user. A separate authenticated route only becomes non-redundant once Phase 5 adds its distinct payload — the **attendance-template join** (explicitly "stubbed in Phase 2; fully wired in Phase 5") driving the homepage today-classes feed + the "My Schedule" eligibility grid. Building it now would be a redundant endpoint with no consumer (the §6 homepage feed is also deferred). Land it in Phase 5 alongside the attendance-template data it's meant to carry.

## 6. Frontend

### 6.1 Class detail page (extension)
- [x] Shows resolved price / "Included with your membership" / "Members only" via the `pricingState` getter, driven by the API's `price_*` fields (normalized to real booleans in `ServerAccessNetwork.getClassDetail`).
- [x] Included recurring classes show "Just show up — staff will check you in" (no Book CTA).
- [x] Paid offerings show the resolved tier price + the BC-5 "No refunds — staff may issue a voucher case-by-case" note inline. *(A dedicated "Reserve" button is deferred until the paid class-booking flow is wired from class-detail — Phase 7.)*
- [x] Component spec updated (included / paid+no-refund / members-only).

### 6.2 Calendar view labels — DEFERRED (calendar is mock-only)
- [~] **Deferred.** The calendar (`pages/calendar/`) is **entirely mock-driven** — `CalendarService` returns `mockCalendarResponse()` with a `TODO replace with API call`, and `CalendarEvent` carries only `{id,title,startTime,endTime,location}` (no class / price / membership fields). It is not wired to `visible_event_sessions` or any backend. Chip-labeling sessions by "Included / Tier-priced / Members only" first requires connecting the calendar to the real session API — a calendar↔API integration well outside Phase 2's membership-gating scope. Revisit when the calendar is wired to live data. (The membership pricing/visibility surface is already delivered via the class catalog + `visible_event_sessions`, which now carry `class_*` + `can_book` + price.)

### 6.3 My-bookings page — PARTIAL (cancel messaging DONE 2026-06-01)
- [x] **Cancel flow is class-aware (P-6).** In `my-events.component`: when the cancel-confirm fetches the session and it belongs to a class (`EventSession.class_id`, surfaced by §4.2), it shows **"Class bookings are non-refundable. Contact staff if you need a voucher."** and sets the no-refund acknowledgment instead of computing a refund-window breakdown. After cancel, when the response carries `refund.reason === "non_refundable"` (§5.1 / OQ-P2-1), the result message says the booking is non-refundable (not the misleading "policy window has passed"). Added `class_id`/`class_name` to the `EventSession` TS type and the `refund` object to `CancelBookingResponse`. Tests: `my-events.component.spec.ts` (class non-refundable pre-cancel notice, non-class keeps the window text, post-cancel `refund.reason` message) — 42 specs green.
- [~] **Class photo + class name on the booking *card* — deferred.** The booking list (`getMyBookings` → `UserBooking`) shows the product name; surfacing the class photo/name requires adding `class_id`/`class_name` to the backend `UserBookingInfo` / `GetBookingsForPerson` (a backend change, not frontend-only). Land it with that backend extension.

### 6.4 BC-5 cancellation-policy display — DEFERRED (display already inline; component is an optional refactor)
- [~] **Deferred (optional DRY refactor).** The cancellation-policy *display* is already delivered in both places it's needed: the **class-detail** page shows the BC-5 "No refunds — staff may issue a voucher case-by-case" note inline (§6.1, done), and **my-events** renders the refund-window breakdown / non-refundable notice inline (§6.3). There is no separate cancel-confirmation *dialog* (my-events uses an inline confirm panel). Extracting a shared `cancellation-policy.component` would be a cosmetic refactor of two working consumers, not new capability — deferred as a tidy-up.

### 6.5 `ServerAccess` extensions — DEFERRED (blocked on §5.2)
- [~] **Deferred.** `getVisibleClasses()` would call `GET /api/me/visible_classes`, which was **deferred to Phase 5** (§5.2 — redundant with `GET /api/classes` today; its distinct attendance-template payload is a Phase-5 deliverable). No endpoint to call ⇒ no client method to add yet. Lands with §5.2 in Phase 5.

## 7. Admin Metadata

- [~] Verify admin UI for `product_prices` tier pricing end-to-end — not re-verified this pass (the existing product/pricing admin already supports per-permission `product_prices`).
- [x] ~~Friendly name + bool edit-type for `products.is_membership_included`~~ — **REMOVED with the column** ([[Permission-based class access redesign]] §3, 2026-05-31). The redesign instead registered `class_requirement_groups` + `class_requirement_group_literals` (nested admin metadata under `classes`, gated by `manage_class_schedule`) and `permission_implications` (top-level, admin-only) — the friendly authoring UI for these is the §6.6 Phase-1 follow-up.

## 8. Permissions

- [x] No new permissions needed in this phase. The existing `manage_products` covers product / pricing edits; `manage_class_schedule` (introduced in Phase 1) covers schedules.

## 9. Tests-Required Summary

- [x] Table helper tests for the access-permission column round-trips — `products_test.cpp` (`GetProductSurfacesPermissionColumns` / `...AbsentWhenNull` / `GetActiveProductsSurfacesPermissionColumns`); tier-price resolution already covered by `product_prices_test.cpp` (`GetBestProductPriceByProductSchedulePermissions*`).
- [x] `catalog_helper_test.cpp`: `ResolveBestPriceForPerson` covers included→tier-price, lowest-wins, non-member-with-price, non-member-blocked, product-not-found (done in the §4.1 work + the access-redesign closure test).
- [x] Updated `event_session_helper_test.cpp`: visible-event-sessions surfaces class metadata (`class_id`/`class_name`) + resolved price (`VisibleSessionSurfacesClassMetadata`, `StandaloneSessionHasNoClassMetadata`).
- [x] Updated `booking_helper_test.cpp`: reject `NO_ADVANCE_BOOKING_REQUIRED`, paid booking still works, **plus the §4.3 cancel matrix** (class non-refundable / non-class refundable / free-cancel override / zero-money no-throw).
- [~] `session_cancellation_helper_test.cpp` (admin-cancel): full refund for paid bookings (existing). Zero-money no-refund test deferred to Phase 8 (NULL purchase_id not constructible yet — M-7).
- [x] Endpoint tests: `visible_event_sessions` (class metadata), `book_event` (NO_ADVANCE_BOOKING_REQUIRED → 400 reject path), `cancel_booking` (P-6 `refund.issued=false/reason=non_refundable` + non-class contrast). `classes` / `classes/<id>` are already viewer-aware (covered at the helper layer); `me/visible_classes` deferred to Phase 5.
- [x] Frontend specs: class-detail (included / paid+no-refund / members-only — §6.1) and my-bookings cancel messaging (class non-refundable pre-cancel + post-cancel `refund.reason` — §6.3, 42 specs green). Deferred: calendar chip rendering (§6.2, mock-only calendar), shared cancellation-policy component (§6.4, optional), `getVisibleClasses` mock (§6.5, blocked on §5.2).

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

## 11. Weekly "Our Schedule" view + requirement inline descriptions

> **Added 2026-06-01 (Mason).** Two related additions:
> 1. A human-readable **`inline_description`** on each access requirement group — a free-text blurb describing that requirement (e.g. "Gold or Platinum membership", "Intermediate Acro skill").
> 2. A public **"Our Schedule"** weekly page (Sun–Sat) showing every active class's slots for the coming week, each with a class **photo thumbnail, class name, instructor, duration, the requirements inline description, and any "requires attending" predecessor**.
>
> Builds on existing infrastructure: Phase 1's lazy derived-session model (`ClassScheduleHelper::GetDerivedSessionsForRange`), the requirement-group access model ([[Permission-based class access redesign]]), SL-11 predecessor resolution, the photo whitelist (`classes` is photo-supported), and the dynamic "Our Classes" nav. Bottom-up build order: **DB → table helpers → business logic → endpoints → frontend.**

**Design decisions:**
- **`inline_description` lives on `class_requirement_groups`** (per-group blurb), not a single class-level field — so the description tracks the actual CNF groups and the §6.6 Requirements editor edits it inline beside each group. Display surfaces the **AND-join** of a class's non-empty group descriptions (empty ⇒ "open to everyone"). *(Alternative — one class-level field — rejected for that reason.)*
- The **"Our Schedule" page is public / anonymous-capable** (like `/api/classes`): it shows *what runs each day*, not a booking surface. Recurring classes have no advance booking (P-1) and there is no Reserve flow yet, so this is purely informational.
- **"Valid over the next week" = a 7-day Sun–Sat window.** Default = the current week (the UTC-midnight Sunday on/before today through the following Saturday, matching the Phase-1 occurrence-date convention); an optional `week_start_us` pages forward/back. Per day, resolve each active class's active instance → active impl → slots whose `day_of_week` matches (exactly `GetDerivedSessionsForRange`).

### 11.1 Database ✅ DONE (2026-06-02) — C++ for Mason to build
- [x] Added **`inline_description TEXT NOT NULL DEFAULT ''`** to `class_requirement_groups` — `db_schema/class_requirement_groups.h` (`kClassRequirementGroupsInlineDescription` constant + comment) and `.cpp` (`AddColumnNotNullableWithDefault(..., DB_TYPE_STRING, "''")`, placed after `label`). No new table, no FK. Pre-deploy (no migration).
- [x] `create_database.cpp` admin metadata: `PopulateAdminColumnDataInfo` row (`text` edit type, optional) + `PopulateAdminColumnFriendlyNames` row ("Requirement Description"). (The table is already a nested admin-CRUD table under `classes` from the redesign §3.1.)
- [x] Test: `class_requirement_groups_test.cpp::InlineDescriptionDefaultsEmptyAndRoundTrips` — defaults to `''` on add and round-trips through `UpdateGroup`. (Covers the §11.2 round-trip bullet too.)

### 11.2 Table Helpers ✅ DONE (2026-06-02) — C++ for Mason to build
- [x] `TableHelpers::ClassRequirementGroups`: `inline_description` round-trips automatically via `GetGroup`/`GetGroupsByClass` (`DbCrud::GetRow` / `GetRowsByValuesWithOrderBy` = `SELECT *`). Added a 4-arg **`AddGroup(tx, classId, label, inlineDescription)`** overload (the existing 3-arg delegates with `""`). Tests in `class_requirement_groups_test.cpp`: `InlineDescriptionDefaultsEmptyAndRoundTrips` (default `''` + `UpdateGroup` round-trip, from §11.1) and `AddGroupWithInlineDescription` (overload sets it on create).
- [x] **No new query for the weekly view** — confirmed it reuses Phase 1 reads: `ClassInstances::GetActiveInstance`, `ClassSchedules::GetActiveImplementation`, `ClassScheduleSlots::GetSlotsByImplementationAndDay`, and the SL-11 predecessor read `GetSlotsByImplementationWithPredecessor`. Verified the latter's SQL already selects `predecessor_class_name` / `predecessor_start_time_minutes` / `predecessor_day_of_week`, so §11.3 can resolve "requires attending" with no new query.

### 11.3 Business Logic ✅ DONE (2026-06-02) — C++ for Mason to build
- [x] New **`WeeklyScheduleHelper`** (`business_logic/scheduling/weekly_schedule_helper.{h,cpp}`): `ScheduleSlotView` (classId, className, classHasPhoto, dayOfWeek 0=Sun, startUs/endUs, durationMinutes, instructorPersonId?+instructorName, requirementsDescription, requiresAttending + predecessor className/dayOfWeek/startTimeMinutes) and `WeeklySchedule { weekStartUs; slots }` (flat, each carries dayOfWeek; endpoint groups Sun–Sat).
- [x] `GetWeeklySchedule(tx, weekStartUs)` — `weekStartUs<=0` ⇒ current week (UTC-midnight Sunday on/before `now_us()` via local `TruncToUtcMidnight`/`DayOfWeekUtc`). For each `ClassCatalogHelper::GetActiveClasses`, enriches `className` (catalog entry), `classHasPhoto` (`TableItemPhotos::HasPhoto("classes", classId)`), `durationMinutes`, `instructorName` (`People::GetPersonById`), `requirementsDescription`, and predecessor (slot → `predecessor_class_schedule_slot_id` → walk slot→impl→instance→class for the cross-class name + day/start). No CRUD SQL in business logic — all via table helpers (only `SELECT now_us()`, the accepted scalar pattern).
	- ⚠️ **Revised (2026-06-02) — marketing recurring-pattern, not per-day occurrences.** The first cut used `GetDerivedSessionsForRange(classId, weekStart, +7d)`, which resolves the active instance/impl **per day** and so dropped any weekday earlier than the instance's `valid_from_us` (which defaults to creation time). Symptom Mason hit: "Knotty Yoga runs Mon + Wed but only Wed shows once Monday has passed" — the instance was entered mid-week, so Monday < valid_from. First fix resolved the active impl **once** at `now` and projected its slots — but that still lost a weekday when the impl active *at that instant* didn't own the earlier slot.
	- ✅ **Final (2026-06-02) — per-day UNION of the active impl's slots, projected onto each slot's own weekday.** For each class, loop the 7 days; for any day with an active instance+impl (`GetActiveScheduleView(classId, weekStart + dow·day)`), collect its slots into a set de-duped by slot id; then project each unioned slot onto `slot.day_of_week` (`startUs = weekStart + dayOfWeek·day + startMinutes·min`). This shows the full Sun–Sat pattern — a class entered mid-week still surfaces its earlier weekdays (the Monday slot is collected from a later day's active impl and projected back onto Monday), and a mid-week impl change shows both patterns. A class dark on **every** day of the week contributes nothing. Slots are sorted by (day, start, class name). Triggered by Mason's second report: the 7pm predecessor showed on Monday but the 6pm target class did not. Trade-off unchanged: per-occurrence cancellations are not reflected (a public "what we run each week" view).
	- ✅ **Instructor thumbnail (2026-06-02, Mason request).** Added `instructorHasPhoto` + `instructorId` to `ScheduleSlotView`; serialized as `instructor_has_photo` / `instructor_id` in `ScheduleSlotViewToKeyValueTable`. Frontend renders a small round 20px avatar beside the instructor name, falling back to the person icon when absent.
		- ⚠️ **Corrected (2026-06-02): photo lives on `instructors`, not `people`.** First cut keyed the thumbnail on `people/<person_id>` — but instructor photos are stored on the **`instructors`** table by its own id (the public Instructors page uses `instructors/<instructor_id>`), so nothing showed. Fix: the weekly helper resolves the slot's person → instructor via `Instructors::GetInstructorByPersonId`, exposes `instructorId`, and sets `instructorHasPhoto` from `HasPhoto("instructors", instructorId)`. Frontend URL is now `/api/get_scaled_photo/instructors/<instructor_id>/64/64`.
		- ✅ **Public photo access (answers "don't we have an unauthenticated photo path?").** `get_scaled_photo` only treated `home_page_photos` as public (the carousel carve-out); `classes` + `instructors` required login, so the public Classes / Class-Detail / Instructors / Our-Schedule pages showed broken images for anonymous visitors. Added **`classes` and `instructors` to `IsPublicScaledPhotoTable`** — deliberate marketing-imagery carve-outs (no private data; dimension clamp + `public, max-age=3600` edge cache still apply). New endpoint tests: `GetScaledPhotoClassPhotoServesAnonymous`, `GetScaledPhotoInstructorPhotoServesAnonymous` (and the existing `people`-stays-401 test still pins the closed allow-list).
- [x] `requirementsDescription`: AND-join (`" - "`, plain ASCII to avoid an MSVC narrow-literal charset risk) the class's non-empty `class_requirement_groups.inline_description`s; empty when no groups. *(The frontend can re-render the separator as a "·" if desired.)* Added **`inlineDescription` to `RequirementGroupView`** + populated it in `ClassAccessHelper::GetClassRequirements` (one read shared by the schedule + the §6.6 editor); the weekly helper reads the groups directly via `ClassRequirementGroups::GetGroupsByClass`. Test: `class_access_helper_test::GetClassRequirementsSurfacesInlineDescription`.
- [x] Tests (`weekly_schedule_helper_test.cpp`, 11 cases): Mon+Wed land on the right days with duration/className; instructor name resolved (+ `instructorHasPhoto` false when no photo) / empty when none; requirements description joined from groups (empty group skipped) / empty for an open class; cross-class predecessor resolved (`requires_attending`); `weekStartUs<=0` defaults to the current-week Sunday (UTC-midnight, ≤ now); a future-windowed instance contributes no slots to the queried week (dark); **`EarlierWeekdayShownEvenWhenInstanceStartedMidWeek`** (the union fix); **`SlotsSortedByDayThenStartTime`** (out-of-order Mon 7pm + Mon 6pm sort by start). Registered both files in `CMakeLists.txt`.

### 11.4 Endpoints ✅ DONE (2026-06-02) — C++ for Mason to build
- [x] **`GET /api/schedule/week[?week_start_us=<us>]`** — `endpoints/schedule_week.{h,cpp}`, public (anonymous OK). Thin handler → `WeeklyScheduleHelper::GetWeeklySchedule`. Emits `{ week_start_us, days: [ { day_of_week, date_us, slots: [...] } ] }` — **always 7 day buckets** (Sun..Sat, `date_us = weekStart + dow days`); each slot from `ScheduleSlotViewToKeyValueTable` (new KVT converter) with the **`requires_attending` predecessor object nested by the endpoint** (present only when set), same parent-KVT-then-nested pattern as `get_class_detail`. `week_start_us` should be a Sunday-midnight; absent/≤0 ⇒ current week. Registered in `web_app.cpp` + `endpoints/CMakeLists.txt`.
- [x] Endpoint tests (`endpoints/schedule_week_test.cpp`, 3 cases): anonymous 200 with a Monday slot on the right day + `requirements_description` + empty Sunday; cross-class `requires_attending` nested object; no-param defaults to the current week with 7 days.
- [x] **§6.6 read** surfaces the new per-group `inline_description`: added to `RequirementGroupToKeyValueTable` (the `inlineDescription` field was added to `RequirementGroupView` + `GetClassRequirements` in §11.3), so `GET /api/admin/class/<id>/requirements` now returns it. Tests: `scheduling_key_value_table_test::RequirementGroupConvertsScalarFields` (+ new `ScheduleSlotViewConvertsScalarFields` / `...OmitsInstructorIdWhenNone`) and `admin_class_requirements_list_test::Returns200GroupTreeWithResolvedNames` (asserts the group's `inline_description`).

### 11.5 Frontend ✅ DONE (2026-06-02)
- [x] **"Our Schedule" nav entry** — prepended a fixed **"Our Schedule"** item to the TOP of the dynamic "Our Classes" top-level menu (above "All Classes"), linking to `/schedule`. (`shared/services/header/mockHeaderResponse.ts` — the menu is already populated dynamically from active classes; the fixed item is now first.)
- [x] **Public route `/schedule`** → new `OurScheduleComponent` (`pages/public/our-schedule/`, standalone, `SharedModule`; route registered in `pages/public/public.routes.ts`). Renders a **vertical scrollable Sun–Sat list** — each day is a stacked row whose classes are roomy **horizontal cards** (thumbnail + details), so the rich info fits without a horizontal grid or table. Each card shows: class **photo thumbnail** (`/api/get_scaled_photo/classes/<id>/…` when `has_photo`, with a placeholder otherwise), **class name**, **time + duration**, **instructor** (person icon), **requirements inline description** (lock icon — the membership/skill blurb), and a **"Requires attending: {class} · {day} {time}"** line (event_repeat icon) when set. Prev/next/today week navigation via `week_start_us`. Per-day "No classes" note + empty-week message.
	- *Refinement (2026-06-02, Mason):* switched from the original 7-column horizontal grid to the vertical list above — the grid was too narrow for the per-class membership / skill / instructor / predecessor detail.
- [x] **`ServerAccess`**: `getWeeklySchedule(weekStartUs?)` across the interface + proxy + `ServerAccessNetwork` (`GET /api/schedule/week`) + `ServerAccessMock` (derives 7 days from seeded slots). New `WeeklySchedule` / `ScheduleDay` / `ScheduleSlot` types in `class.types.ts` (re-exported via `ServerAccess.ts`). `has_photo` normalized via `toBool` per the KVT-string convention.
- [x] **§6.6 Requirements editor** (`class-requirements-editor.component`): added a per-group **inline description** text field (`updateGroupDescription`), persisted via the generic-CRUD `updateItem` on `class_requirement_groups` (same path as the group label; no-ops when unchanged). `inline_description` added to the `RequirementGroup` type and the mock `class_requirement_groups` table + `getClassRequirements`.
- [x] Specs: `our-schedule.component.spec.ts` (8 tests — days/slots; photo / instructor / duration / requirements / predecessor; week nav; empty), `ServerAccess.mock.spec.ts` weekly-schedule block (2 tests), `mockHeaderResponse.spec.ts` updated for the prepended "Our Schedule" item, and `class-requirements-editor.component.spec.ts` (4 new tests — field renders, persists, blank clears, no-op). All **388** frontend specs pass.

### 11.6 Tests rollup
- [x] **Frontend** (verified, 388 pass): Our Schedule component, ServerAccess mock `getWeeklySchedule`, header nav "Our Schedule", and Requirements-editor inline-description field.
- [ ] **C++ (for Mason to build/run)**: DB/table-helper inline_description round-trip · `WeeklyScheduleHelper` (days / enrichment / predecessor / requirements join / dark days / week-start default / **mid-week-`valid_from` regression** — `EarlierWeekdayShownEvenWhenInstanceStartedMidWeek`) · `/api/schedule/week` endpoint + the §6.6 inline_description read. *(Tests written; not built locally per the no-build-server rule.)*

## 12. Instructors (admin management + people thumbnails)

> **Added 2026-06-02 (Mason).** An admin-portal area to manage **Instructors**: view the list, full CRUD, pick a **person** from the system to promote to instructor, edit a **bio**, and upload/see a **photo**. Plus: people listings in the portal show a **thumbnail** per person (useful for the class instructor picker and the staff page).

**What already exists (do NOT rebuild):**
- `instructors` table (`id`, `person_id`→`people`, `bio`) + `TableHelpers::Instructors` (`AddInstructor`/`GetInstructorById`/`GetInstructorByPersonId`/`GetAllInstructors`/`UpdateInstructorBio`/`DeleteInstructor`).
- Public read **`GET /api/get_instructors`** → `{ items: [{ instructor_id, person_id, first_name, last_name, bio, has_photo }] }` via `Auth::InstructorHelper::GetInstructorsForPublicDisplay` — **already joins the person name + `has_photo`**. Reusable as the admin list source (verify it returns ALL instructors, not a public-only subset).
- Generic admin CRUD: `instructors` is registered top-level + nested, with a `bio` column data-info (long-text), friendly names, and **photo support** (`photo_support_tables`). The `people` display template is name-based (`{first_name} {last_name} - {email}`), so a `person_id` FK picker shows names.
- Photo endpoints: `POST /api/upload_photo/<table>/<id>/<type>`, `DELETE /api/delete_photo/<table>/<id>`, `Images::ImageHelper`. Frontend: `uploadPhoto`, `hasPhoto`, the `controls/photo-upload` component, and `getFkOptions(table, search, n)`.
- **`people` is photo-support registered ⇒ the generic admin `people` table view already renders a 50×50 thumbnail per row** (`controls/table-view-control`).
- Public Instructors page (`pages/public/instructors`) renders `instructors/<id>` photos; `instructors` is now on the public scaled-photo allow-list (§11.3 follow-up).

**Design decision — a dedicated Instructors admin page (not the raw generic CRUD).** The generic table CRUD *can* edit instructors, but (a) its list shows `person_id` as a raw number — the instructors display template `{person_id}` can't resolve to the person's name (the resolver only fills own-row columns, no join), and (b) photo upload is a separate thumbnail-click, not part of add/edit. A small bespoke page delivers the asked-for UX — a list of **name + thumbnail + bio**, an add flow that **searches people** and captures a bio, and **inline photo upload** — while reusing the existing backend wholesale.

### 12.1 Backend ✅ DONE (2026-06-02) — C++ for Mason to build
- [x] **Prevent a duplicate instructor per person.** Added `databaseInfo.AddUniqueConstraint(kInstructorsTable, kInstructorsPersonId)` in `MakeInstructorsTable` (`db_schema/instructors.cpp`) — `person_id` is now FK + UNIQUE (one-to-one with a person). Pre-deploy, no migration. Tests in `instructors_test.cpp`: `AddInstructorDuplicatePersonRejected` (second `AddInstructor` for the same person throws) + `AddInstructorDistinctPersonsAllowed` (two different people both succeed).
- [x] **Confirmed `GET /api/get_instructors` returns ALL instructors** — `Auth::InstructorHelper::GetInstructorsForPublicDisplay` calls `Instructors::GetAllInstructors` and only skips an orphan (person row missing, which the FK makes near-impossible); there is **no public/active filter**, so it is correct to reuse for the admin list. Added identity-field coverage the admin page depends on: `instructor_helper_test` Basic now pins `instructorId` (the photo-path key, distinct from `personId`); `get_instructors_test` Basic now asserts `instructor_id` + `person_id` in the JSON. (Existing empty/multiple/empty-bio/`has_photo:false` coverage retained.)
- [x] (Reuse — no new endpoints) create = generic `add_item` on `instructors {person_id, bio}`; edit bio = `update_item`; delete = `delete_item`; photo = `upload_photo` / `delete_photo`. All already admin-gated. *(The UNIQUE constraint is the backstop for the generic `add_item` create path.)*

### 12.2 ServerAccess layer (frontend) ✅ DONE (2026-06-02)
- [x] **Reused existing methods — no interface change.** The page composes `getInstructors()`, `getFkOptions('people', …)`, `addItemFetchPrimaryKey`, `updateItem`, `deleteItem`, `uploadPhoto`, `hasPhoto` directly; a typed `createInstructor` wrapper wasn't worth a new surface. **Mock made coherent:** added a generic `instructors` table to `ServerAccess.mock` (`["bio","id","person_id"]`, seeded for persons 1–2) and rewrote `getInstructors()` to **derive** from it joined with `people` (+ `has_photo` from the in-memory photo map), so create/edit/delete via the generic CRUD helpers reflect in mock mode. Mock spec block "Instructors (Classes Phase 2 §12)" — derivation, add reflects, edit+delete reflect.

### 12.3 Frontend — Instructors admin page ✅ DONE (2026-06-02)
- [x] New standalone `InstructorsAdminComponent` (`pages/admin/instructors/`, SharedModule + `PhotoUploadComponent`), registered as the `instructors` child route (before the legacy `:tableName` catch-all) + a **"Manage Instructors"** button on the admin dashboard (`goToInstructors()`).
- [x] **List**: a row per instructor with a round thumbnail (`/api/get_scaled_photo/instructors/<id>/64/64`, person-icon fallback), `First Last`, a plain-text bio snippet (HTML stripped + truncated), and Edit + Delete actions.
- [x] **Add**: a searchable **people picker** (`mat-autocomplete` over `getFkOptions('people', …)`, `displayWith` for clean labels, excludes people already instructors) + a bio textarea → `addItemFetchPrimaryKey('instructors', { person_id, bio })`; on success the panel reveals the `photo-upload` component bound to the new instructor id. Guarded: Create disabled / errors without a chosen person.
- [x] **Edit**: a bio textarea + the `photo-upload` component (`tableName='instructors'`, `tableItemId=instructor_id`) to add/replace the picture; Save → `updateItem`.
- [x] **Delete**: inline two-step confirm → `deleteItem('instructors','id', id)`, then reload. (Photo cleanup handled by existing `delete_photo`/cascade semantics.)
- [x] Specs: `instructors-admin.component.spec.ts` (11 tests — load/list, thumbnail vs icon, empty state, bio-snippet, people search + exclusion, blank-search no-op, create guard, create writes + reveals photo upload, edit saves, delete confirm/cancel, load error); dashboard `goToInstructors` nav test.

### 12.4 People thumbnails in the portal ✅ DONE (2026-06-02)
- [x] **Confirmed** the generic admin `people` table view already renders a per-row 50×50 thumbnail — `controls/table-view-control` shows a `.photo-thumbnail` (with placeholder) for any photo-support table, and `people` is photo-support registered. No change needed. (`people` photos stay on the **private** scaled-photo path — login required — which is correct for the admin portal; public pages use the public `instructors` path.)
- [x] Internal **staff portal** (`pages/staff/…`) is workflow pages (dashboard, check-in, provider bookings/schedule/time-off/shift-requests) — there is **no people/team listing** there, so nothing to add. The public "meet the team" surface is the Instructors page (`pages/public/instructors`), which already renders photos.

### 12.5 Tests rollup
- [x] **Backend (12.1)** *(for Mason to build/run)*: `instructors_test` unique-person guard (`AddInstructorDuplicatePersonRejected`, `AddInstructorDistinctPersonsAllowed`); `instructor_helper_test` + `get_instructors_test` pin `instructor_id` / `person_id` shape.
- [x] **Frontend (verified, 2275 specs pass)**: `InstructorsAdminComponent` (11 tests), `ServerAccess.mock` Instructors derivation/CRUD block (3 tests), admin dashboard `goToInstructors` nav test, and confirmation that the generic people view already shows thumbnails.

## 13. Open Questions

**All resolved (2026-06-01).** No open questions remain for Phase 2.

- **OQ-P2-1.** ✅ **RESOLVED (Mason → recommendation):** user-initiated cancel inside a non-refund window returns **200 + `refund: { issued: false, reason: "non_refundable" }`** so the UI can render the right message. *(Implementation note: this shape is the contract for the deferred §4.3 cancel path + §5.1 `cancel_booking`.)*
	- Mason- I'll go with your recommendation.
- **OQ-P2-2.** ✅ **RESOLVED (Mason → recommendation):** `NO_ADVANCE_BOOKING_REQUIRED` is a **400** (ValidationError) — matches the status-code policy in memory `error_response_status_codes.md`. *(Already implemented: the booking guard maps to 400 via the endpoint's default branch.)*
	- Mason- I'll go with your recommendation.
- **OQ-P2-3.** ✅ **RESOLVED / moot:** drop `classes.required_permission_id` (do not denormalize). Confirmed — and the [[Permission-based class access redesign]] makes it moot: access lives in `class_requirement_groups` / `class_requirement_group_literals`, resolved by `ClassAccessHelper`; pricing/visibility resolve through the active `class_instances` product. The §2.2 / §3.3 column is permanently skipped.

## 14. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 2.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]].
- Successors: [[Classes Phase 5 - Attendance Templates]], [[Classes Phase 7 - Class Series and Workshops]], [[Classes Phase 8 - Staff Check-in]].
- Pricing context: [[Payment Design Document]], [[Product browsing and quoting endpoints]].
