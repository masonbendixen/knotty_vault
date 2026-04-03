---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/3/2026
Version: 0.2
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

In Payment Design Document.md there is a section Under 8.1 Prioritized Scenarios with a sub section Should Have (Phases 2-3: Multi-Seat & Permission Pricing). Let's make an implementation plan for the incomplete work items under that section.

Please use the code base and these documents for context:
- [[Payment Design Document]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]

# Plan: Multi-Seat & Bundled Product Purchases

## Incomplete Scenarios

From Payment Design Document section 8.1:

- **Scenario 6**: User purchases a one-time item for themselves and one or more other people (e.g., couple's massage)
- **Scenario 7**: User purchases a bundled product (e.g., massage + spa entry as a single product)

---

## Current State Analysis

### Infrastructure That Already Exists

The system has remarkably complete multi-seat infrastructure from the subscription and entitlement work. Here's the full inventory:

**Database Layer:**
- `product_entitlement_rules` table with `seats_default` (default 1), `grants_permission_id`, `validity_kind`, `validity_days` — unique on `product_id`
- `entitlements` table with `seats_total`, `seats_used` (computed)
- `entitlement_assignments` table — links entitlements to people with unique constraint
- `enforce_entitlement_seats` database trigger — prevents over-assignment
- Admin metadata already registered: nested table under products, column friendly names, column data info, display template, manage_products permission

**Business Logic:**
- `EntitlementHelper::CreateEntitlementsForPurchase()` — creates one entitlement per purchase quantity unit, each with `seats_total = seats_default` from the rule
- `EntitlementHelper::AssignEntitlement()` — validates seats available, supports gift permissions
- `PaymentHelper::PayWithCard()` — after payment, creates entitlements AND auto-assigns payer to one seat each
- Gift permission system — full workflow for authorizing cross-user assignment

**Endpoints:**
- `POST /api/entitlement_assign/{id}` — assign a seat
- `DELETE /api/entitlement_unassign/{id}` — remove assignment
- `GET /api/entitlement_assignments/{id}` — list assignments for an entitlement
- `GET /api/purchases` — returns purchases with nested entitlements array (includes `seats_total`, `seats_used`)
- `POST /api/purchase_pay_card/{id}` — returns full `entitlements` array (not just a count)

**Frontend:**
- `SeatAssignmentComponent` (standalone) — takes `entitlementId`, `seatsTotal`, `seatsUsed`, `assignments[]` as inputs; supports assign/remove with gift permission filtering
- `EntitlementAssignment` and `EntitlementWithAssignments` types — fully defined
- `ServerAccess` methods: `assignEntitlementSeat()`, `removeEntitlementAssignment()`, `getEntitlementAssignments()`
- `Purchase` type includes optional `entitlements?: Entitlement[]` and `payments?: Payment[]`
- Subscription detail page already uses `SeatAssignmentComponent` for subscription entitlements

### What's Actually Missing

The gaps are surprisingly small — mostly UI plumbing and one type mismatch:

| # | Gap | Effort |
|---|-----|--------|
| 1 | **TypeScript type mismatch**: `PayCardResponse.entitlements_created` is typed as `number` but C++ returns `entitlements: Entitlement[]`. Frontend ignores the actual entitlements data. | Small fix |
| 2 | **Catalog doesn't expose `seats_default`**: users can't see "this is a 2-person product" before purchasing | Small backend + small frontend |
| 3 | **Checkout success shows nothing after payment**: just "Payment Successful!" with navigation buttons, no entitlements or seat assignment | Medium frontend |
| 4 | **No individual purchase detail page**: `/my/purchases` is a list, no `/my/purchases/:id` for managing a specific purchase's entitlements | Medium full-stack |
| 5 | **Purchase history has no seat management**: shows "X/Y seats used" text but no link to assign seats | Small frontend |
| 6 | **Bundle products**: no structural way to express what's in a bundle | Design decision needed |

---

## Design Decisions & Alternatives

### D1: How Should Multi-Seat Products Work at Checkout?

**The question**: When a user buys a 2-seat product, should they assign the second person during checkout, or after payment?

**Option A — Assign after payment (recommended)**:
The user purchases the product normally. After payment succeeds, the success screen shows the created entitlement(s) with seat assignment inline via `SeatAssignmentComponent`. The payer is already auto-assigned. They can assign remaining seats now or come back later via a purchase detail page.

*Pros*: Simpler checkout flow, no risk of payment failure after complex seat selection, consistent with subscription seat assignment pattern already working.
*Cons*: User might forget to assign. Second person isn't validated before payment.

**Option B — Select people during checkout**:
The checkout shows a "Who is this for?" section before payment. User selects themselves + N-1 other people from their gift permission contacts. The purchase is created with pre-selected beneficiaries.

*Pros*: Everything configured in one flow. Second person gets notified immediately.
*Cons*: Significant checkout complexity. What if the user doesn't have gift permissions set up yet? What if payment fails after they've done all the selection? Couples-massage use case: the user might not know the other person's email yet.

**Option C — Hybrid: show seat info but defer assignment**:
Checkout shows "This includes 2 seats — you'll assign the second person after purchase." No selection UI. After payment, show assignment inline.

*Pros*: User is informed without complexity. Assignment happens in a calm post-payment state.
*Cons*: Extra text in checkout without actionability.

**Recommendation**: Option A with a touch of Option C — inform the user about seats during checkout, but actual assignment happens post-payment. This matches the existing subscription pattern and avoids checkout complexity.

### D2: Do We Need a Purchase Detail Page?

**The question**: Should there be a dedicated `/my/purchases/:id` page, or can we handle everything in the existing purchase history list?

**Option A — New purchase detail page (recommended)**:
Create `/my/purchases/:id` that shows the full purchase with items, payments, entitlements, and seat assignment via `SeatAssignmentComponent`. Purchase history links to it. Checkout success links to it.

*Pros*: Clean URL for bookmarking/sharing. Natural place for seat management. Mirrors subscription detail pattern. Can show all purchase data without cramming into a list item.
*Cons*: New page to build and test.

**Option B — Expand purchase history rows inline**:
Add `SeatAssignmentComponent` directly into the purchase history expansion panel for each entitlement.

*Pros*: No new page needed.
*Cons*: Crowded UX if there are multiple entitlements. Doesn't have a clean URL. Assignment UI inside an accordion is awkward.

**Option C — Generic entitlement management page**:
Create `/my/entitlements/:id` that works for any entitlement (subscription-based, purchase-based, comp-based).

*Pros*: Single place for all entitlement management. 
*Cons*: Entitlement without purchase context is confusing. Subscriptions already have their own detail page. Doesn't reduce the need for a purchase detail page.

**Recommendation**: Option A. The purchase detail page is the natural container — it shows what was purchased, how it was paid, and what access was granted, all in one view. Subscriptions have their own detail page because the subscription lifecycle is different, but one-time purchases should have their own too.

### D3: How Should Bundle Products Work?

**The question**: What does "bundled product" mean for the system?

After deep analysis, I see three possible interpretations of "bundle" and they have very different implementation implications:

**Interpretation 1 — Marketing bundle (single product, single entitlement)**:
A "Massage + Spa Combo" is just a product. The admin creates it, describes what's included in the description, sets a discounted price, and creates one entitlement rule. Staff validates the bundle entitlement at check-in. The system treats it identically to any other one-time product.

*This requires zero code changes.* The admin can do this today.

**Interpretation 2 — Structured bundle (single purchase, multiple products)**:
The user adds a "Massage + Spa Combo" to their cart, and the system knows it contains Product A (Massage) and Product B (Spa Entry). The purchase has two `purchase_items`, each with its own product reference, but the total price is discounted.

*This would require*: a `product_bundle_items` junction table, changes to the catalog to show component breakdown, changes to purchase creation to resolve bundle pricing (sum of components vs bundle price), and changes to entitlement creation to create separate entitlements per component.

**Interpretation 3 — Multi-item cart (shopping cart with multiple products)**:
No "bundle" concept at all. The user simply adds Massage ($80) and Spa Entry ($40) to a multi-item cart and gets both. The `purchase_items` table already supports multiple items per purchase. Discount would come from a coupon.

*This partially exists*: `CreatePurchaseRequest` accepts an `items[]` array with multiple products. But the current checkout UI always creates a single-item purchase for the one product being viewed.

**Recommendation**: Start with Interpretation 1 (zero code, just admin documentation). If there's demand, move to Interpretation 3 (multi-item cart) which is more generally useful than structured bundles. Interpretation 2 (structured bundles) is the most complex and least likely to be needed.

**Important insight**: The multi-item cart capability (Interpretation 3) is already 90% built on the backend. `PurchaseHelper::CreatePurchase` accepts multiple items. `CatalogHelper::GetQuote` resolves prices for multiple products. What's missing is only the frontend: a cart component that holds multiple products before creating the purchase. This might be more valuable than a formal "bundle" concept.

### D4: What About Event/Service Bookings with Multi-Seat?

**The question**: If someone buys a couples-massage (2-seat product), how does that interact with service booking?

Currently, `BookEvent` and `BookService` create a purchase internally for a single person. They don't support multi-seat.

**Options**:
1. **Multi-seat products are only for one-time purchases** — event/service bookings are always single-seat. If two people want a couples massage, each books separately. A "couples" product variant is just a different duration/room type.
2. **Extend booking endpoints to accept seat count** — the booking creates a multi-seat entitlement and the payer assigns the second person.
3. **Two-step flow** — purchase the multi-seat product first (creates entitlement), then book the session using the entitlement's seats.

**Recommendation**: Option 1 for now. A "couples massage" is better modeled as a product variant (60 min couple's massage @ $120) that happens to book both people into the same time slot. Multi-seat entitlements are about access grants (like memberships), not about scheduling two people into one appointment slot. If the scheduling case becomes important, Option 2 can be added later.

---

## Implementation Plan

### Phase 1: Fix the Foundation (Small, High-Impact)

#### 1.1 Fix TypeScript PayCardResponse Type

The C++ backend returns `"entitlements": [...]` as a full array, but the TypeScript type says `entitlements_created: number`. This means the frontend has been ignoring entitlement data from every payment response.

**Changes:**
- `payment.types.ts`: Change `PayCardResponse.entitlements_created: number` to `entitlements: Entitlement[]`
- Same for `PayVoucherResponse`
- Update the mock to return entitlement arrays
- Verify no frontend code depends on the old `entitlements_created` field

**Tests:** Mock spec tests, checkout spec tests, any tests that construct PayCardResponse

#### 1.2 Add `seats_default` to Catalog Response

**Backend:**
- `catalog_helper.h`: Add `seatsDefault` field to `ProductInfo`
- `catalog_helper.cpp`: In `ProductInfoFromKeyValueTable()` or `GetProduct()`, look up the product's `product_entitlement_rules.seats_default` and populate it (default to 1 if no rule exists)
- `payment_key_value_table.cpp`: Add `seats_default` to `ProductInfoToKeyValueTable()`

**Frontend:**
- `payment.types.ts`: Add `seats_default?: number` to `CatalogProduct`

**Tests:** `catalog_helper_test.cpp`, `payment_key_value_table_test.cpp`

---

### Phase 2: Purchase Detail Page (Medium, Core Feature)

#### 2.1 Purchase Detail Endpoint

**Create `GET /api/purchases/{id}` endpoint:**
- Returns a single purchase with items, entitlements (with assignments), and payments
- Each entitlement includes its `assignments[]` array (reuse existing `GetEntitlementAssignments`)
- Only accessible by the purchase's payer or an admin
- Returns 404 if not found, 403 if wrong person

**Endpoint:** `endpoints/purchase_detail.h/cpp`

**Backend types:** Create a `PurchaseDetailResult` that wraps `PurchaseInfo` + `vector<EntitlementWithAssignments>` + `vector<PaymentInfo>`

**Tests:** `purchase_detail_test.cpp` — detail returns entitlements with assignments, auth checks, 404 for missing

#### 2.2 ServerAccess Method

**Add `getPurchaseDetail(purchaseId: number)` to all ServerAccess layers:**
- Interface, proxy, network, mock
- Returns a type that includes entitlements with assignments

**Tests:** Mock spec tests

#### 2.3 Purchase Detail Component

**Create `PurchaseDetailComponent` at `pages/account/purchase-detail/`:**
- Route: `/my/purchases/:id`
- Displays: purchase summary card (items, totals, date, status)
- Payment section: each payment with provider, amount, date
- Entitlements section: for each entitlement, show status, validity, and **`SeatAssignmentComponent`** if `seats_total > 1`
- For single-seat entitlements: just show status and who it's assigned to
- Back link to `/my/purchases`

**Tests:** `purchase-detail.component.spec.ts`

#### 2.4 Purchase History Links

**Update `PurchaseHistoryComponent`:**
- Each purchase row becomes clickable → routes to `/my/purchases/:purchaseId`
- For multi-seat entitlements in the expansion panel, show "Manage seats →" link
- Keep the existing expansion panel for quick preview

**Tests:** `purchase-history.component.spec.ts`

---

### Phase 3: Checkout & Booking Success Enhancement (Medium)

#### 3.1 Checkout Success — Show Entitlements

**Update `CheckoutComponent` success state:**
- After payment, use the `PayCardResponse.entitlements` array (now correctly typed from Phase 1.1)
- Show each entitlement: product info, validity, seat status
- If any entitlement has `seats_total > seats_used`: show `SeatAssignmentComponent` inline with a header like "Assign seats for your purchase"
- Add link: "View purchase details" → `/my/purchases/:id`
- Keep existing "Continue Shopping" and "View My Purchases" buttons

**Tests:** `checkout.component.spec.ts` — entitlements shown, seat assignment renders for multi-seat

#### 3.2 Event Booking Success — Show Entitlements

**Update `EventBookingComponent` success state:**
- After booking+payment, if the response includes entitlements with multi-seat, show seat assignment
- More likely scenario: event bookings are typically 1-seat, so this may just show "Booking confirmed — 1 seat assigned to you"
- Add link to purchase detail page

**Tests:** `event-booking.component.spec.ts`

#### 3.3 Service Booking Success — Show Entitlements

**Same pattern as 3.2 for `ServiceBookingComponent`.**

**Tests:** `service-booking.component.spec.ts`

---

### Phase 4: Catalog Display Enhancement (Small)

#### 4.1 Multi-Seat Badge in Catalog

**Update catalog/product browse component:**
- If `seats_default > 1`: show a badge like "For 2 people" or an icon with "2 seats"
- In product detail/checkout info section: "This product includes {N} seats — you can assign participants after purchase"

**Tests:** `catalog.component.spec.ts`

---

### Phase 5: Bundle Products — Admin Documentation (Trivial)

#### 5.1 Document Bundle Creation Workflow

No code changes. Write admin documentation explaining:
1. Create a product with a descriptive name (e.g., "Massage + Spa Day Combo")
2. Write a description listing what's included
3. Set pricing to the bundle price (discounted vs individual)
4. Optionally create an entitlement rule if the bundle grants access
5. The product appears in the catalog like any other product

#### 5.2 Verify End-to-End Bundle Flow

Manually verify: admin creates bundle product → appears in catalog → user purchases → entitlement created → works at check-in.

---

## Implementation Order

| Step | Section | Description | Effort | Dependencies |
|------|---------|-------------|--------|--------------|
| 1 | 1.1 | Fix PayCardResponse TypeScript type | Small | None |
| 2 | 1.2 | Add seats_default to catalog response | Small | None |
| 3 | 2.1 | Purchase detail endpoint | Medium | None |
| 4 | 2.2 | ServerAccess getPurchaseDetail | Small | 2.1 |
| 5 | 2.3 | Purchase detail component | Medium | 2.2 |
| 6 | 2.4 | Purchase history links | Small | 2.3 |
| 7 | 3.1 | Checkout success entitlements | Medium | 1.1, 2.3 |
| 8 | 3.2 | Event booking success | Small | 1.1 |
| 9 | 3.3 | Service booking success | Small | 1.1 |
| 10 | 4.1 | Catalog multi-seat badge | Small | 1.2 |
| 11 | 5.1 | Bundle documentation | Trivial | None |

**Steps 1-2 can be done in parallel. Steps 3-5 are the critical path for multi-seat. Steps 7-9 depend on the type fix. Step 10 depends on the catalog enhancement. Step 11 is independent.**

---

## Implementation Checklist

### Phase 1.1 — Fix PayCardResponse Type
- [ ] Update `PayCardResponse` in `payment.types.ts`: change `entitlements_created: number` to `entitlements: Entitlement[]`
- [ ] Update `PayVoucherResponse` similarly (C++ returns entitlements array from voucher payment too)
- [ ] Update `ServerAccessMock` to return entitlement arrays in `purchasePayCard()` and `purchasePayVoucher()`
- [ ] Update `ServerAccessNetwork` if any response mapping is needed
- [ ] Fix any tests that construct `PayCardResponse` with `entitlements_created`
- [ ] Verify checkout, event-booking, service-booking components don't break

### Phase 1.2 — Catalog seats_default
- [ ] Add `seatsDefault` to `ProductInfo` in `catalog_helper.h`
- [ ] Look up `product_entitlement_rules.seats_default` in `CatalogHelper::GetProduct()` / `GetCatalog()` 
- [ ] Add `seats_default` to `ProductInfoToKeyValueTable()` in `payment_key_value_table.cpp`
- [ ] Add `seats_default` to `CatalogProduct` in `payment.types.ts`
- [ ] Tests: `catalog_helper_test.cpp`, `payment_key_value_table_test.cpp`

### Phase 2.1 — Purchase Detail Endpoint
- [ ] Create `endpoints/purchase_detail.h/cpp` — `GET /api/purchases/{id}`
- [ ] Return purchase with items, entitlements (with assignments for each), and payments
- [ ] Auth: only payer or admin
- [ ] Register in `web_app.cpp` and `CMakeLists.txt`
- [ ] Tests: `purchase_detail_test.cpp` — success, 404, 403, includes assignments

### Phase 2.2 — ServerAccess getPurchaseDetail
- [ ] Add `PurchaseDetailResponse` type to `payment.types.ts` (purchase + entitlements with assignments + payments)
- [ ] Add `getPurchaseDetail(purchaseId)` to ServerAccess interface
- [ ] Implement in proxy, network, mock
- [ ] Tests: `ServerAccess.mock.spec.ts`

### Phase 2.3 — Purchase Detail Component
- [ ] Create `pages/account/purchase-detail/purchase-detail.component.ts/html/scss`
- [ ] Route at `/my/purchases/:id` in account routes
- [ ] Show purchase summary, items, payments, entitlements
- [ ] Integrate `SeatAssignmentComponent` for multi-seat entitlements
- [ ] Tests: `purchase-detail.component.spec.ts`

### Phase 2.4 — Purchase History Enhancement
- [ ] Make purchase rows clickable → `/my/purchases/:purchaseId`
- [ ] Show "Manage seats" link for multi-seat entitlements
- [ ] Tests: `purchase-history.component.spec.ts`

### Phase 3.1 — Checkout Success Entitlements
- [ ] After payment, use `PayCardResponse.entitlements` array
- [ ] Show entitlement cards with status, validity, seats
- [ ] For multi-seat: show `SeatAssignmentComponent` inline
- [ ] Add "View purchase details" link
- [ ] Tests: `checkout.component.spec.ts`

### Phase 3.2 — Event Booking Success
- [ ] Show entitlement info after successful booking+payment
- [ ] For multi-seat: show seat assignment inline
- [ ] Tests: `event-booking.component.spec.ts`

### Phase 3.3 — Service Booking Success
- [ ] Same as 3.2 for service booking
- [ ] Tests: `service-booking.component.spec.ts`

### Phase 4.1 — Catalog Multi-Seat Badge
- [ ] Show "For N people" badge on catalog products where `seats_default > 1`
- [ ] Show seat info text on checkout page
- [ ] Tests: catalog spec tests

### Phase 5.1 — Bundle Documentation
- [ ] Document the admin workflow for creating bundle products
- [ ] Verify end-to-end flow

---

## Open Questions

| # | Question | Context | Recommendation |
|---|----------|---------|----------------|
| 1 | **Should the purchase detail page show the full payment history or just the summary?** | Purchases can have multiple payments (voucher + card split). Showing each payment row is useful for troubleshooting. | Show full payment history — it's already returned by the API. |
| 2 | **Should multi-seat booking flows work for events and services?** | Currently booking creates 1-seat purchases. A "couples massage" could be a product variant rather than a multi-seat entitlement. | Defer. Model couples services as product variants for now. Multi-seat booking can be added if there's demand. |
| 3 | **Should we build a shopping cart?** | The backend already supports multi-item purchases via `CreatePurchaseRequest.items[]`. A cart UI would enable buying multiple products at once, which is more generally useful than structured bundles. | Defer to a separate planning document. It's a valuable feature but a distinct work item from multi-seat. |
| 4 | **Should voucher payment success also show entitlements?** | When a voucher fully covers a purchase, `PayVoucherResponse` also creates entitlements. The current type has `entitlements_created: number` like `PayCardResponse`. | Yes — fix `PayVoucherResponse` in Phase 1.1 alongside `PayCardResponse`. Same pattern. |
| 5 | **What about the event/service booking responses?** | `BookEventResponse` and `BookServiceResponse` don't include entitlements at all — they return `{purchase, booking}`. Should they? | Defer. Event/service bookings typically don't create entitlements (the booking IS the access). Only subscription products and one-time "membership" products create entitlements. |
| 6 | **Is there a need for a maximum seats limit per product?** | Currently `seats_default` is fixed. What if you want "up to 5 people" where the user chooses how many? | Defer. The current model (fixed seats per product) covers couples massage and small-group scenarios. A variable-seat model would need purchase-time seat selection, which is significantly more complex. |

---

## Design Debates

### Debate 1: Post-Payment vs In-Checkout Seat Assignment

The plan recommends post-payment assignment (D1, Option A). But there's a counterargument worth considering:

**For in-checkout assignment**: If someone is buying a "Couple's Massage" as a gift, they might want to assign it to two OTHER people (not themselves). Currently, `PayWithCard` auto-assigns the payer. The payer would need to remove themselves and add the intended recipients. This is a bit awkward.

**Counter-counter**: The payer can always remove themselves after the fact. And the gift permission system already handles the "assign to someone else" flow elegantly. The subscription flow already works this way (payer is auto-assigned, then re-assigns).

**Resolution**: Keep post-payment for v1. If the "buy as gift" flow is too awkward, consider adding a "This is a gift" checkbox at checkout that skips auto-assignment.

### Debate 2: Single Purchase Detail Page vs Per-Entitlement Pages

We could have `/my/entitlements/:id` as a standalone page instead of (or in addition to) `/my/purchases/:id`.

**For purchase-centric**: A purchase is the natural unit of "I bought something." Users think in terms of purchases, not entitlements. Purchase detail naturally groups related entitlements.

**For entitlement-centric**: An entitlement is the thing you actually USE. You want to manage seats on the entitlement, not on the purchase. Subscriptions already have their own detail pages separate from purchases.

**Resolution**: Purchase detail page for now, with `SeatAssignmentComponent` embedded for each entitlement. If there's demand for standalone entitlement pages (e.g., for comp'd entitlements that have no purchase), add them later.

### Debate 3: Bundle — Product Kind vs Display Tag vs Nothing

The original document discussed adding a `bundle` product kind or display tags. After analysis:

**Just use description (v1)**: Admin writes "Includes: 60-min massage + full-day spa access" in the product description. No code changes. This handles 100% of the actual use case.

**Why not a new kind?**: Product `kind` drives actual behavior (one_time, subscription, event, bookable_service). A "bundle" isn't a different behavior — it's a one-time product. Adding a kind that doesn't change behavior creates confusion.

**Why not display tags?**: Adding a `display_tags` column is a general-purpose feature that might be useful later (featured, new, limited, etc.) but isn't needed specifically for bundles. If there's a future need to flag products for catalog presentation, it should be designed holistically.

**Resolution**: No code changes for bundles in v1. Document the admin workflow. Revisit if there's real demand for structured bundle metadata.
