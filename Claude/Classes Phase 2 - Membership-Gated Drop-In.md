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
- [ ] No new tables. Reuse:
  - `product_prices` (per-permission pricing — already supports M-2 for paid offerings).
  - `product_booking_windows` (per-permission `advance_days` — for AW-1).
  - `product_entitlement_rules.grants_permission_id` (membership tier → permission).
  - `cancellation_policies`, `cancellation_policy_windows`.

### 2.2 Small flag additions
- [ ] `classes.required_permission_id BIGINT` NULL — the permission a user must hold to attend this class. NULL = no permission required (rare; only for the studio's first generic offerings). Resolved via the class's product (`products.booking_permission_id`), so this is a denormalized convenience for catalog filtering. Optional — could compute on the fly. Recommend YES because the catalog filter query is hot.
- [ ] `products.is_membership_included BOOLEAN NOT NULL DEFAULT FALSE` — if true AND user holds the booking permission, attendance is included with their membership (M-1). Drives the "Included" badge.

### 2.3 Wire into DB init
- [ ] Update `make_database_info.cpp` and `create_database.cpp` to add these columns to existing tables.
- [ ] Update `db_schema/classes.h` and `db_schema/products.h` (or wherever they live) with column-name constants.

## 3. Table Helpers

### 3.1 Extend `TableHelpers::Products`
- [ ] Surface `is_membership_included`, `booking_permission_id`, `visibility_permission_id` in reads.
- [ ] Add `GetProductByClassId(Transaction&, int64_t classId)` if not already present — used by pricing resolution.
- [ ] Tests for column round-trips.

### 3.2 Extend `TableHelpers::ProductPrices`
- [ ] If not already present, add `GetActivePricesForProduct(Transaction&, int64_t productId, int64_t asOfUs)` — returns rows where the product's active `price_schedules` row covers `asOfUs`. Used for tier-price resolution.
- [ ] Tests.

### 3.3 Extend `TableHelpers::Classes`
- [ ] Surface the new `required_permission_id` column.

## 4. Business Logic

### 4.1 Pricing-resolution helper (extension of `CatalogHelper`)
- [ ] In `business_logic/payment/catalog_helper.h/.cpp`:
  - Add `struct ResolvedPrice { int64_t priceCents = 0; std::string currency; int64_t permissionId = 0; bool isIncluded = false; bool isAvailable = true; std::string unavailableReason; }`.
  - Add `ResolvedPrice ResolveBestPriceForPerson(Transaction&, int64_t productId, int64_t personId, int64_t asOfUs)`. Algorithm:
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
  - Pick (a). Add the explicit reject path in `BookingHelper::BookEvent` when the session's class has `is_membership_included && user holds booking permission`.
- [ ] For workshops / series / intro (paid path), `BookEvent` runs as today — creates purchase + booking, charges Square, etc.
- [ ] **User-initiated cancellation** (`POST /api/cancel_booking/<id>`):
  - For paid bookings: free the capacity, advance the waitlist (existing `BookingHelper::CancelBooking`), but DO NOT issue a refund (per P-6). Update return JSON so the UI knows no refund is happening; the user can see the cancellation-policy display (BC-5) BEFORE clicking cancel.
  - For zero-money bookings (shouldn't exist post-Phase 2, but defensive): just free capacity.
- [ ] **Admin-initiated session cancellation** (`SessionCancellationHelper`):
  - Full refund for paid bookings (existing behavior).
  - No refund for zero-money bookings.

### 4.4 KeyValueTable conversions
- [ ] Extend `EventSessionInfoToKeyValueTable` to include the new `class_*` fields and `resolved_price.*` sub-table.
- [ ] Tests in `scheduling_key_value_table_test.cpp`.

## 5. Endpoints

### 5.1 Reuse + extend existing endpoints
- [ ] `GET /api/visible_event_sessions?placement=upcoming|home_page` — already exists. Verify it correctly serializes the new class fields when `class_id` is set. Add endpoint test cases for member-with-included and non-member.
- [ ] `POST /api/book_event/<id>` — already exists. Add a guard at the top of the handler / inside `BookingHelper::BookEvent` to reject membership-included class bookings with the new `NO_ADVANCE_BOOKING_REQUIRED` error. Add endpoint test.
- [ ] `POST /api/cancel_booking/<id>` — already exists. Adjust the response payload so the `refund_*` fields explicitly reflect "no refund issued" per P-6. Add endpoint test asserting no refund was triggered (test mail helper captures no `BookingCancellationMail` with a refund line).
- [ ] `GET /api/classes` and `GET /api/classes/<id>` (introduced in Phase 1) — extend to use `ResolveBestPriceForPerson`. Add endpoint tests.

### 5.2 New visibility endpoint (helpful for the user homepage)
- [ ] `GET /api/me/visible_classes` (logged in only) — returns the union of:
  - All `class_schedules` (active, with at least one materialized future session) where the user holds the booking permission (membership-included).
  - All paid offerings (series / workshops / intro / guest-pass-flagged) visible to them at their tier price.
- [ ] Lightweight payload — drives the homepage today-classes feed + the "My Schedule" eligibility grid in Phase 5. Reuses `ClassCatalogHelper::GetClassesVisibleToPerson` plus an attendance-template join (stubbed in Phase 2; fully wired in Phase 5).

## 6. Frontend

### 6.1 Class detail page (extension)
- [ ] Show resolved price / "Included with your membership" / "Members only" badge from the API.
- [ ] Hide the "Book this class" CTA for membership-included recurring classes; show "Just show up — your membership includes this" with a hint that staff will check them in.
- [ ] For paid offerings: show "Reserve" CTA + the resolved tier price + BC-5 cancellation-policy text inline ("No refunds — staff may issue a voucher case-by-case").
- [ ] Component spec.

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

- [ ] Verify admin UI for `product_prices` allows tier-per-product pricing end-to-end for class products. Add a test if the existing admin CRUD has a gap.
- [ ] Friendly name for `products.is_membership_included` ("Included with membership").

## 8. Permissions

- [ ] No new permissions needed in this phase. The existing `manage_products` covers product / pricing edits; `manage_class_schedule` (introduced in Phase 1) covers schedules.

## 9. Tests-Required Summary

- [ ] Table helper tests for the new column round-trips.
- [ ] `catalog_helper_test.cpp`: `ResolveBestPriceForPerson` covers included, paid tier, multi-permission lowest-wins, non-member-with-price, non-member-blocked.
- [ ] Updated `event_session_helper_test.cpp`: visible-event-sessions surfaces class metadata + resolved price.
- [ ] Updated `booking_helper_test.cpp`: reject `NO_ADVANCE_BOOKING_REQUIRED`, paid booking still works.
- [ ] Updated `event_session_cancellation_helper_test.cpp` (or whichever owns admin-cancel): full refund for paid, no refund for zero-money.
- [ ] Endpoint tests for `visible_event_sessions`, `book_event` (reject path), `cancel_booking` (no-refund path), `classes`, `classes/<id>`, `me/visible_classes`.
- [ ] Frontend specs for class-detail (included vs paid), calendar chip rendering, my-bookings class chrome, cancellation-policy component, mock service.

## 10. Cross-Layer Acceptance Criteria

A new user, given:
- An active "Gold Membership" entitlement that grants `gold_member` permission.
- A class "Vinyasa Flow" whose product has `is_membership_included=true` and `booking_permission_id = gold_member`.
- A series "6-Week Aerial 101" whose product has tier prices: gold $120, silver $180, non-member $300.

Should be able to:
- [ ] See "Vinyasa Flow" in the catalog with "Included with your Gold Membership" label.
- [ ] See "6-Week Aerial 101" at $120 in the catalog.
- [ ] Be prevented (400) from POSTing `book_event` on a Vinyasa Flow session ("`NO_ADVANCE_BOOKING_REQUIRED`").
- [ ] Successfully book the aerial series via the workshop / series flow at $120.

A non-member should:
- [ ] NOT see "Vinyasa Flow" in the catalog at all.
- [ ] See "6-Week Aerial 101" at the non-member $300 price.

## 11. Open Questions

- **OQ-P2-1.** Cancel-during-non-refund-window response: should the API return 200 + `refund: { issued: false, reason: "policy" }`, or 200 + nothing about refund? Recommended: explicit `refund: { issued: false, reason: "non_refundable" }` so the UI can render the right message.
- **OQ-P2-2.** Should `NO_ADVANCE_BOOKING_REQUIRED` be a 400 or a 409? Recommended: 400 ValidationError (clearly user-side wrong) — matches existing error-status policy in memory `error_response_status_codes.md`.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 2.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]].
- Successors: [[Classes Phase 5 - Attendance Templates]], [[Classes Phase 7 - Class Series and Workshops]], [[Classes Phase 8 - Staff Check-in]].
- Pricing context: [[Payment Design Document]], [[Product browsing and quoting endpoints]].
