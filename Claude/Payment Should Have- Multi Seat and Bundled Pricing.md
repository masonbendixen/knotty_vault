---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/13/2026
Version: 0.1
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

### What Already Exists (Scenario 6)

The multi-seat infrastructure is **almost entirely built** from the subscription work. What exists:

| Layer | Component | Status |
|-------|-----------|--------|
| DB Schema | `product_entitlement_rules.seats_default` | Exists, supports > 1 |
| DB Schema | `entitlements.seats_total` | Exists |
| DB Schema | `entitlement_assignments` table | Exists with unique constraint |
| DB Trigger | `enforce_entitlement_seats` | Exists, prevents over-assignment |
| Backend | `EntitlementHelper::CreateEntitlementsForPurchase()` | Creates entitlements with `seats_total` from rule |
| Backend | `PayWithCard` auto-assigns payer | Assigns payer to 1 seat per entitlement |
| Backend | Entitlement assignment endpoints | `POST /assign`, `DELETE /remove` |
| Backend | Gift permission system | Full workflow for authorizing cross-user assignment |
| Frontend | `SeatAssignmentComponent` | Standalone, reusable — takes entitlement inputs |
| Frontend | `EntitlementAssignment` types | Fully defined in `payment.types.ts` |
| Frontend | `ServerAccess` entitlement methods | `assignEntitlementSeat()`, `removeEntitlementAssignment()`, `getEntitlementAssignments()` |
| Frontend | `PayCardResponse` | Includes `entitlements` array with seat info |

### What's Missing (Scenario 6)

| Gap | Description |
|-----|-------------|
| **Admin UI** | No admin metadata for `product_entitlement_rules` — admins can't set `seats_default > 1` |
| **Catalog display** | No indication in catalog/product detail that a product is multi-seat |
| **Checkout success** | After payment, shows generic "Payment Successful!" — no entitlement/seat info, no seat assignment prompt |
| **Purchase detail page** | No dedicated page to view a purchase's entitlements with seat assignment (purchase history shows "X/Y seats" text but no assignment UI) |

### What Already Exists (Scenario 7)

| Layer | Component | Status |
|-------|-----------|--------|
| DB Schema | `products` table | Any product can represent a bundle |
| DB Schema | `product_entitlement_rules` | One rule per product (unique constraint on `product_id`) |
| Backend | Full purchase/payment/entitlement flow | Works for any product |
| Frontend | Catalog, checkout, purchase history | Works for any product |

### What's Missing (Scenario 7)

| Gap | Description |
|-----|-------------|
| **Bundle structure** | No way to express what's "in" a bundle other than the description text |
| **Savings display** | No way to show "Save $X vs buying separately" |
| **Admin workflow** | No special tooling for creating bundles — admin just creates a product |

---

## Design Decisions

### D1: Multi-seat purchase flow

After payment, if any created entitlement has `seats_total > seats_used`, the checkout success screen should show seat assignment inline using the existing `SeatAssignmentComponent`. The payer is already auto-assigned to one seat by `PayWithCard`. For a couple's massage (`seats_default = 2`), the user sees "1/2 seats assigned" and can immediately assign the second seat.

**Alternative considered**: Redirect to a separate purchase detail page. Rejected because forcing navigation after payment is a worse UX — the user should be able to assign seats right there. They can always come back via purchase history later.

### D2: Purchase detail page

Create a purchase detail page at `/my/purchases/:id` that shows the full purchase with items, entitlements, and seat assignment. This mirrors the subscription detail page pattern. The checkout success screen links here for later management. Purchase history list links here for each purchase.

### D3: Bundle approach — simple product (v1)

Per the Payment Design Document's design decision: a bundle is a single product. The admin creates a product named "Massage + Spa Combo" with a description listing what's included and sets a discounted price. One `product_entitlement_rules` entry grants a single permission. Staff validates the bundle entitlement at check-in.

No new tables needed for v1. A structured `product_bundle_items` table is deferred to v2 (if ever needed).

### D4: Catalog display for multi-seat products

Add `seats_default` to the catalog response. The frontend shows a "X seats" badge on multi-seat products so users understand what they're buying before checkout.

### D5: Admin metadata for product_entitlement_rules

Add admin table metadata so admins can view and edit entitlement rules (especially `seats_default`) through the existing admin dashboard. This is required for admins to create multi-seat products without direct SQL.

---

## Part A: Multi-Seat One-Time Purchases (Scenario 6)

### A.1 Admin Configuration for Entitlement Rules

**Goal**: Admins can configure `seats_default` and other entitlement rule properties through the admin dashboard.

**Backend changes**:
- Add admin metadata entries for `product_entitlement_rules` table in `create_database.cpp`:
  - `admin_table_display_templates` entry
  - `admin_column_friendly_names` for: product_id, grants_permission_id, seats_default, validity_kind, validity_days
  - `admin_column_data_info` for foreign key columns (product_id → products, grants_permission_id → permissions)
  - `admin_top_tables` or `admin_nested_tables` entry (nested under products is cleanest)

**No frontend changes** — the existing admin CRUD components render from metadata automatically.

**Tests**: Verify the table appears in admin schema and is CRUD-able through endpoints.

### A.2 Catalog Enhancement — Multi-Seat Indicator

**Goal**: Catalog displays seat count so users know a product is multi-seat before purchasing.

**Backend changes** (`catalog_helper.h/cpp`):
- Add `seatsDefault` field to `CatalogItem` (or embed in `ProductInfo`)
- Populate from `product_entitlement_rules.seats_default` when building catalog response
- Add `seats_default` to `CatalogItemToKeyValueTable()` in `payment_key_value_table.cpp`

**Frontend changes** (`payment.types.ts`, `catalog.component.ts`):
- Add `seats_default?: number` to `CatalogProduct` type
- In catalog component template: if `seats_default > 1`, show badge like "2 seats" or "For 2 people"
- In product detail / checkout: show seat count info ("This product includes 2 seats — you can assign seats after purchase")

**Tests**:
- Backend: `catalog_helper_test.cpp` — verify multi-seat product includes `seats_default` in response
- Backend: `payment_key_value_table_test.cpp` — verify KV conversion includes field
- Frontend: `catalog.component.spec.ts` — verify badge renders for multi-seat products

### A.3 Purchase Detail Page

**Goal**: Dedicated page to view a purchase with its entitlements and manage seat assignments.

**Backend** — `GET /api/purchases/<int>`:
- New endpoint (or enhance existing if one exists)
- Returns purchase with items, entitlements, and entitlement assignments
- Each entitlement includes its assignments (reuse `EntitlementHelper::GetAssignments()`)
- Only accessible by the purchase's payer (or admin)

**Frontend** — `PurchaseDetailComponent`:
- Route: `/my/purchases/:id`
- Shows: purchase summary (items, totals, payment status, date)
- For each entitlement: status, validity, seat progress, `SeatAssignmentComponent` (inline)
- Navigation: purchase history list → purchase detail

**Tests**:
- Backend: `purchases_test.cpp` — endpoint tests for purchase detail with entitlements and assignments
- Frontend: `purchase-detail.component.spec.ts` — renders entitlements, shows seat assignment for multi-seat

### A.4 Checkout Success Enhancement

**Goal**: After payment, show created entitlements with seat assignment capability.

**Frontend changes** (`checkout.component.ts`):
- On payment success, use the `PayCardResponse.entitlements` array (currently returned but ignored)
- If any entitlement has `seats_total > seats_used`: show seat assignment section inline using `SeatAssignmentComponent`
- Add link: "Manage this purchase" → `/my/purchases/:id`
- Keep existing "Continue Shopping" and "View My Purchases" buttons

**The checkout component needs the entitlement assignments** to pass to `SeatAssignmentComponent`. Two options:
1. Enhance `PayCardResponse` to include assignments (the payer was just auto-assigned)
2. After payment, call `getEntitlementAssignments()` for each entitlement

Option 1 is cleaner. The backend already refreshes entitlements after auto-assignment in `PayWithCard` (line 266 of `payment_helper.cpp`), but doesn't include assignments in the response. Add assignments to the entitlement response from `PayWithCard`.

**Backend changes** (`payment_helper.cpp`):
- After auto-assigning, include assignments in the returned `EntitlementInfo` objects
- Add `assignments` to `EntitlementInfoToKeyValueTable` output (as a nested array)
- Or: add a separate `entitlement_assignments` array to `PayCardResult`

**Simpler alternative**: Don't change the backend response. After payment success in the frontend, call `getEntitlementAssignments(entitlementId)` for each multi-seat entitlement. This keeps the backend simpler and uses existing endpoints.

**Recommendation**: Use the simpler alternative. The checkout success page can lazily load assignments for any entitlement where `seats_total > 1`.

**Tests**:
- Frontend: `checkout.component.spec.ts` — verify entitlement section shown after payment, seat assignment rendered for multi-seat products

### A.5 Purchase History Enhancement

**Goal**: Link from purchase history list to purchase detail page.

**Frontend changes** (`purchase-history.component.ts`):
- Each purchase row becomes clickable → routes to `/my/purchases/:purchaseId`
- Keep the expansion panel for quick preview, but add a "View Details" link
- For multi-seat entitlements, the expansion preview shows "X/Y seats — Manage seats" link

**Tests**:
- Frontend: `purchase-history.component.spec.ts` — verify click navigation and manage seats link

---

## Part B: Bundled Products (Scenario 7)

### B.1 Simple Bundles (v1) — Admin Workflow

**Goal**: Admin can create bundle products through the admin dashboard.

This requires **no code changes**. The admin:
1. Creates a product with name "Massage + Spa Combo" and description "Includes: 60-min Swedish massage and full-day spa access"
2. Sets product kind to `one_time`
3. Creates a `product_prices` entry with the discounted bundle price
4. Creates a `product_entitlement_rules` entry with appropriate permission and seats

The bundle appears in the catalog like any other product. Users purchase it through the normal checkout flow.

**Documentation**: Add a section to the admin guide explaining how to create bundle products.

### B.2 Bundle Display Enhancement (v1)

**Goal**: Catalog and checkout show bundle pricing context.

**Frontend changes**:
- If a product description contains structured content (e.g., bullet points), render it with formatting in the product detail / checkout page
- Consider a `is_bundle` flag on products or a naming convention to trigger bundle-specific UI (like a "Bundle" badge)

**Simple approach**: Use a product kind value. Currently products have `kind` = `one_time`, `subscription`, or `event`. Adding `bundle` as a fourth kind would allow the frontend to display a "Bundle" badge without any structural changes. However, a bundle IS a one-time purchase functionally.

**Recommendation**: Don't add a new kind. Instead, use a display hint. Add an optional `display_tags` text column to products (comma-separated tags like "bundle,featured,new"). The frontend parses tags and renders badges. This is more flexible than a boolean and avoids proliferating product kinds.

**Alternative recommendation (simpler)**: Just use the product description. If the admin writes a clear description, no code changes are needed. Defer display enhancements until there's real demand for bundle-specific UI.

### B.3 Structured Bundles (v2 — deferred)

**Goal**: Structured tracking of what's in a bundle for automated savings display and per-component entitlements.

**New table**: `product_bundle_items`
```
id BIGSERIAL PRIMARY KEY
bundle_product_id BIGINT FK → products (the bundle)
component_product_id BIGINT FK → products (what's in it)
quantity INT DEFAULT 1
created_us BIGINT
```

**Backend**:
- Table helper for `product_bundle_items`
- `CatalogHelper::GetProduct()` returns component list for bundles
- Savings calculation: sum of component prices vs bundle price
- Option: `CreateEntitlementsForPurchase` creates per-component entitlements for bundles

**Frontend**:
- Catalog/checkout shows component breakdown with individual vs bundle pricing
- "You save $X!" callout

**This is deferred** — only implement when there's a real need for structured bundle metadata. The simple approach (v1) covers the scenario adequately.

---

## Implementation Order

| Step | Section | Description | Dependencies | Effort |
|------|---------|-------------|--------------|--------|
| 1 | A.1 | Admin metadata for product_entitlement_rules | None | Small |
| 2 | A.2 | Catalog seats_default in response | None | Small |
| 3 | A.3 | Purchase detail endpoint + page | A.2 (for display) | Medium |
| 4 | A.4 | Checkout success seat assignment | A.3 (for navigation) | Medium |
| 5 | A.5 | Purchase history → detail links | A.3 | Small |
| 6 | B.1 | Bundle admin workflow (documentation only) | A.1 (admin can create products) | Trivial |
| 7 | B.2 | Bundle display enhancement | — | Small (or deferred) |

**Critical path**: A.1 → A.2 → A.3 → A.4 → A.5 is the main sequence for Scenario 6. Steps 1 and 2 can be done in parallel. Step 6 (Scenario 7 v1) requires no code — just admin documentation.

---

## Implementation Checklist

### A.1 — Admin Configuration
- [ ] Add `admin_table_display_templates` entry for `product_entitlement_rules` in `create_database.cpp`
- [ ] Add `admin_column_friendly_names` entries
- [ ] Add `admin_column_data_info` entries (FK references to products, permissions)
- [ ] Add `admin_nested_tables` entry (nested under products)
- [ ] Verify admin dashboard renders the table correctly

### A.2 — Catalog Multi-Seat Indicator
- [ ] Add `seatsDefault` to `ProductInfo` or `CatalogItem` in `catalog_helper.h`
- [ ] Populate `seatsDefault` from `product_entitlement_rules` in `CatalogHelper`
- [ ] Add to `CatalogItemToKeyValueTable()` in `payment_key_value_table.cpp`
- [ ] Add `seats_default` to Angular `CatalogProduct` type
- [ ] Show multi-seat badge in catalog component
- [ ] Show seat info in checkout component before payment
- [ ] Tests: catalog_helper_test, payment_key_value_table_test, catalog.component.spec

### A.3 — Purchase Detail Page
- [ ] Create `GET /api/purchases/<int>` endpoint (or verify existing) with entitlements + assignments
- [ ] Endpoint declaration in header file
- [ ] Register route in web_app.cpp
- [ ] Add `getPurchaseDetail(id)` to `ServerAccess` interface
- [ ] Implement in `ServerAccessNetwork` and `ServerAccessMock`
- [ ] Create `PurchaseDetailComponent` with template and styles
- [ ] Integrate `SeatAssignmentComponent` for multi-seat entitlements
- [ ] Add route `/my/purchases/:id` to portal routing
- [ ] Tests: endpoint tests, ServerAccess.mock.spec, purchase-detail.component.spec

### A.4 — Checkout Success Enhancement
- [ ] Use `PayCardResponse.entitlements` in checkout success state
- [ ] For multi-seat entitlements (`seats_total > 1`), load assignments via `getEntitlementAssignments()`
- [ ] Render `SeatAssignmentComponent` inline for each multi-seat entitlement
- [ ] Add "Manage this purchase" link to purchase detail page
- [ ] Tests: checkout.component.spec

### A.5 — Purchase History Enhancement
- [ ] Make purchase rows clickable → navigate to `/my/purchases/:id`
- [ ] Add "View Details" or "Manage seats" link for multi-seat entitlements
- [ ] Tests: purchase-history.component.spec

### B.1 — Simple Bundles (Documentation)
- [ ] Document bundle creation workflow for admin
- [ ] Verify a bundle product can be created, priced, purchased, and entitled through existing flow

### B.2 — Bundle Display (Optional)
- [ ] Add bundle indicator to catalog display (badge or tag)
- [ ] Format bundle descriptions with component listing