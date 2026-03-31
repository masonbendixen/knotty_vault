---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/30/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

In the document Payment Design Document.md, there is a section Handle in Future (Phases 5-6: Vouchers & Refunds) with some remaining items to complete. Please create a design and implementation plan in this document to complete those remaining items. Please use this document for context, the source code, and these other documents:

- [[Nested item support]]
- [[Payment Design Document]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Product browsing and quoting endpoints]]
- [[Product, Event, and Subscription Admin Portal]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]
- [[Event Polish- Scheduling Should Have Items]]
- [[Bookable Service Foundation]]
- [[Provider Portal]]

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Plan: Vouchers & Refunds

This plan implements scenarios 22–30 from Payment Design Document.md (the "Handle in Future — Phases 5-6" items). Scenario 26 (studio refunds a one-time purchase) is already implemented via `RefundHelper` + Square `RefundPayment` API.

## Current State

**Already built:**
- `RefundHelper` — processes refunds via Square API, records negative payment rows, updates purchase status (scenario 26 ✓)
- `PaymentHelper` — card payments via Square, creates entitlements on full payment
- `EntitlementHelper` — creates entitlements from product rules, supports revocation
- `BookingHelper::CancelBooking()` — cancellation with refund based on cancellation policy windows
- `purchase_payments` junction table — supports multiple payments per purchase (design supports split payments)
- `payments.provider` column already accepts `"voucher"`, `"cash"`, `"comp"` (defined in `payment.types.ts`)
- Cancellation policies with tiered refund windows

**Not yet built:**
- `vouchers` and `voucher_redemptions` database tables
- Voucher creation, validation, redemption logic
- Split payment support (card + voucher in one purchase)
- Coupon/discount system
- Prorated subscription refunds
- Admin comp flow (entitlement without real payment)
- Admin voucher/credit management UI
- Frontend voucher code entry during checkout

## Remaining Scenarios

| # | Scenario | Status |
|---|----------|--------|
| 22 | User redeems voucher fully | Not started |
| 23 | User redeems voucher partially | Not started |
| 24 | Percentage-based discount coupon | Not started |
| 25 | Multiple payment sources (card + voucher) | Not started |
| 26 | Studio refunds one-time purchase | **Done** ✓ |
| 27 | Prorated subscription refund | Not started |
| 28 | Studio comps a service or good | Not started |
| 29 | Admin grants entitlement without payment | Not started |
| 30 | Admin credits user account (as voucher) | Not started |

---

## Phase 1: Voucher Foundation — Tables, Helpers, Admin CRUD (Scenarios 22, 23, 30)

### 1.1 Database Schema — `vouchers` table

- [ ] **Create `db_schema/vouchers.h/cpp`** — Column constants: `id`, `code` (unique), `currency`, `initial_value_cents`, `remaining_value_cents`, `issued_to_person_id` (nullable FK → people, for account-linked vouchers), `issued_by_person_id` (nullable FK → people, admin who created), `is_active`, `expires_us` (nullable), `notes` (nullable), `created_us`, `updated_us`
- [ ] **Register in `make_database_info.cpp`**, `create_database.cpp` (`CreateTables`, `PopulateAdminTopLevelTables`, column data info, friendly names, display templates)
- [ ] **Add to `CMakeLists.txt`** (db_schema)
- [ ] **Tests** — table creation verified via existing `GlobalDatabaseTestSupport`

### 1.2 Database Schema — `voucher_redemptions` table

- [ ] **Create `db_schema/voucher_redemptions.h/cpp`** — Column constants: `id`, `voucher_id` (FK → vouchers), `purchase_id` (FK → purchases), `payment_id` (FK → payments), `redeemed_cents`, `created_us`
- [ ] **Register in `make_database_info.cpp`**, `create_database.cpp` (as nested table under vouchers)
- [ ] **Add to `CMakeLists.txt`** (db_schema)

### 1.3 Table Helpers

- [ ] **Create `sql_util/table_helpers/vouchers.h/cpp`** — `AddVoucher`, `GetVoucherByCode`, `GetVoucherById`, `UpdateRemainingValue`, `DeactivateVoucher`
- [ ] **Create `sql_util/table_helpers/voucher_redemptions.h/cpp`** — `AddRedemption`, `GetRedemptionsForVoucher`, `GetRedemptionsForPurchase`
- [ ] **Add to `CMakeLists.txt`** (table_helpers)
- [ ] **Tests** — `vouchers_test.cpp`, `voucher_redemptions_test.cpp`

### 1.4 Business Logic — VoucherHelper

- [ ] **Create `business_logic/payment/voucher_helper.h/cpp`** — Contains:
  - `ValidateVoucher(transaction, code)` — checks exists, is_active, not expired, remaining > 0; returns voucher details or error
  - `RedeemVoucher(transaction, code, amountCents, purchaseId, paymentId)` — atomically decrements `remaining_value_cents`, creates `voucher_redemptions` row; returns redeemed amount
  - `CreateVoucher(transaction, request)` — admin creates a new voucher; generates unique code if not provided
  - `GetVoucherBalance(transaction, code)` — returns remaining balance
- [ ] **Add to `CMakeLists.txt`** (business_logic/payment)
- [ ] **Tests** — `voucher_helper_test.cpp` (validate active/inactive/expired/depleted, full redemption, partial redemption, over-redemption capped at remaining)

### 1.5 Admin Endpoints — Voucher Management

- [ ] **Create `endpoints/admin_vouchers.cpp`** — Endpoints:
  - `POST /api/admin/create_voucher` — creates a voucher (amount, currency, optional code, optional person, optional expiry, notes)
  - `GET /api/admin/vouchers` — lists all vouchers with remaining balances
  - `POST /api/admin/deactivate_voucher/:id` — deactivates a voucher
  - `GET /api/admin/voucher/:id/redemptions` — lists redemption history
- [ ] **Register in `web_app.cpp`**
- [ ] **Add to `CMakeLists.txt`** (endpoints)
- [ ] **Tests** — `admin_vouchers_test.cpp`

### 1.6 Admin UI — Voucher Management

- [ ] **Create `pages/manage/vouchers/` component** — Table of vouchers (code, initial/remaining value, status, person, expiry), Create Voucher form, Deactivate button, expandable redemption history
- [ ] **Add route** in manage routing
- [ ] **Add link** in manage dashboard
- [ ] **ServerAccess methods** — `adminCreateVoucher`, `adminGetVouchers`, `adminDeactivateVoucher`, `adminGetVoucherRedemptions`
- [ ] **Tests** — component spec, mock spec

---

## Phase 2: Voucher Payment & Split Payments (Scenarios 22, 23, 25)

### 2.1 PaymentHelper — Voucher Payment Method

- [ ] **Update `PaymentHelper`** — Add `PayWithVoucher(transaction, purchaseId, voucherCode)`:
  - Validates voucher via VoucherHelper
  - Calculates amount to redeem (min of voucher balance, remaining purchase balance)
  - Creates payment row with `provider="voucher"`, `provider_payment_id=voucher_code`
  - Calls `VoucherHelper::RedeemVoucher`
  - Links payment to purchase via `purchase_payments`
  - Updates `purchases.paid_cents`
  - If fully paid, triggers entitlement creation
- [ ] **Tests** — full voucher payment, partial voucher payment, voucher + remaining balance

### 2.2 Split Payment Support

- [ ] **Create `POST /api/purchase_pay_voucher/{purchaseId}`** endpoint — accepts `{ "voucher_code": "..." }`, applies voucher to purchase
- [ ] **Update purchase status logic** — `partially_funded` when voucher covers part, `funded` when fully covered
- [ ] **Allow card payment on partially-funded purchase** — existing `purchase_pay_card` should work if `paid_cents < total_cents`; verify and fix if needed
- [ ] **Tests** — voucher then card, voucher covers full amount, invalid/expired voucher errors

### 2.3 Frontend — Voucher Code Entry in Checkout

- [ ] **Update payment control** — add "Apply Voucher" section with code input field and Apply button
- [ ] **Show applied voucher** — display redeemed amount, remaining purchase balance
- [ ] **ServerAccess methods** — `purchasePayVoucher(purchaseId, voucherCode)`
- [ ] **Conditional card payment** — if voucher covers full amount, skip card entry; otherwise show card form for remainder
- [ ] **Tests** — component spec

### 2.4 Frontend — Voucher Balance Check

- [ ] **Create `GET /api/check_voucher/{code}`** endpoint — returns voucher status, remaining balance (public, no admin required)
- [ ] **ServerAccess + mock** — `checkVoucher(code)`
- [ ] **Tests** — endpoint test, mock spec

---

## Phase 3: Comps & Admin-Granted Entitlements (Scenarios 28, 29)

### 3.1 System Purchase for Comps

- [ ] **Create comp purchase flow** — rather than making `purchase_id` nullable on entitlements (which breaks audit trail), create a `$0 system purchase`:
  - `purchases` row: `total_cents=0`, `paid_cents=0`, `status="funded"`, `payer_person_id=target person`
  - `payments` row: `provider="comp"`, `amount_cents=0`, `status="COMPLETED"`
  - `purchase_payments` linking row
  - Then create entitlement normally with this purchase
- [ ] **Add to `PaymentHelper` or create `CompHelper`** — `CreateComp(transaction, productId, personId, notes)` orchestrates the above

### 3.2 Admin Comp Endpoint

- [ ] **Create `POST /api/admin/comp`** endpoint — accepts `{ "product_id": N, "person_id": N, "notes": "reason" }`; creates $0 purchase + payment + entitlement
- [ ] **Register in `web_app.cpp`**
- [ ] **Tests** — endpoint test (comp creates entitlement, $0 purchase recorded)

### 3.3 Admin UI — Comp

- [ ] **Add comp action** to admin product or user management — "Comp this product to a user" with person picker and notes field
- [ ] **ServerAccess methods** — `adminComp(productId, personId, notes)`
- [ ] **Tests** — component spec, mock spec

---

## Phase 4: Prorated Subscription Refunds (Scenario 27)

### 4.1 Prorated Refund Calculation

- [ ] **Add `CalculateProratedRefund` to `RefundHelper`** — given a subscription billing period (start/end dates) and cancellation date, calculate the unused portion:
  - `refundCents = (daysRemaining / totalDays) * periodAmountCents`
  - Round down to avoid over-refunding
- [ ] **Tests** — mid-period cancellation, last day, first day, already expired

### 4.2 Subscription Cancellation with Refund

- [ ] **Update subscription cancellation flow** — when user cancels mid-period:
  - Calculate prorated refund for remaining days
  - Process refund via `RefundHelper::ProcessRefund`
  - Revoke entitlement (or let it expire at period end — this is an open question)
- [ ] **Tests** — cancel mid-period gets prorated refund, cancel on last day gets nothing

### 4.3 Admin Subscription Refund

- [ ] **Update admin subscription management** — option to refund when cancelling a subscription
- [ ] **Tests** — admin cancel with refund

---

## Phase 5: Coupons (Scenario 24) — *Optional / Lower Priority*

### 5.1 Database Schema — `coupons` and `coupon_redemptions`

- [ ] **Create `db_schema/coupons.h/cpp`** — `id`, `code` (unique), `discount_type` ("percentage" or "fixed"), `discount_value` (percentage 0-100 or cents), `max_uses` (nullable), `current_uses`, `valid_from_us`, `valid_to_us`, `is_active`, `applies_to_product_id` (nullable, for product-specific coupons), `created_us`, `updated_us`
- [ ] **Create `db_schema/coupon_redemptions.h/cpp`** — `id`, `coupon_id` (FK), `purchase_id` (FK), `discount_cents`, `created_us`
- [ ] **Register tables** in `make_database_info.cpp`, `create_database.cpp`

### 5.2 Coupon Business Logic

- [ ] **Create `business_logic/payment/coupon_helper.h/cpp`** — `ValidateCoupon`, `ApplyCoupon` (modifies purchase total), `GetCouponDiscount` (calculates discount for a given cart)
- [ ] **Integration with purchase creation** — apply coupon code during `purchase_create` to reduce `total_cents`
- [ ] **Tests** — percentage discount, fixed discount, expired, max uses reached, product-specific

### 5.3 Coupon Endpoints & UI

- [ ] **Admin CRUD** — create/list/deactivate coupons
- [ ] **Checkout integration** — coupon code input, shows discount applied
- [ ] **Tests**

---

## Open Questions

1. **Voucher code format**: Should voucher codes be auto-generated (e.g., `KY-XXXX-XXXX`) or admin-specified? Or both (admin can optionally provide a custom code)? This affects the `CreateVoucher` API and admin UI.
   - Mason: I like the idea of both. Maybe auto generate one but let the admin replace the suggestion with their own creation. Only issue is checking for uniqueness if the admin specifies one.

2. **Voucher scope**: Should vouchers be tied to a specific person (like store credit) or bearer instruments (anyone with the code can use it)? The schema has `issued_to_person_id` as nullable — if set, only that person can redeem. Gift cards are typically bearer; store credit is person-tied.
   - Mason: - voucher should not be tied to a person. A credit should be tied to a person.

3. **Refund to voucher**: When a refund is processed, should there be an option to refund as store credit (create a new voucher) instead of refunding to the card? This is common in retail and would add a `refund_as_credit` flag to cancellation flows.
   - Mason: Sure, let's do that but we should also be able to refund to a card.

4. **Subscription cancellation timing**: When a user cancels mid-period, should the entitlement be revoked immediately or remain active until the period ends? Options: (a) immediate revocation + prorated refund, or (b) access through period end + no refund. Industry standard is (b), but studios sometimes prefer (a).
   - Mason: Let's have user cancel be B but have an admin override to support A.

5. **Coupon priority**: Is the coupon system (scenario 24) needed now, or can it be deferred? The plan includes it as Phase 5 / lower priority. Coupons add complexity to pricing; vouchers cover most use cases.
   - Mason: Let's support coupons.

6. **Comp notification**: When an admin comps a product, should the recipient receive an email notification? Similar to gift permission notification.
   - Mason:

7. **Voucher expiry behavior**: When a voucher expires, should the remaining balance be forfeited silently, or should the user be notified before expiry? Notification would need a scheduled job.
   - Mason: