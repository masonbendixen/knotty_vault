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

- [x] **Create `db_schema/vouchers.h/cpp`** — Column constants: `id`, `code` (unique, auto-generated or admin-specified), `currency`, `initial_value_cents`, `remaining_value_cents`, `issued_to_person_id` (nullable FK → people; **null = bearer voucher** anyone can redeem, **set = store credit** only that person can redeem), `issued_by_person_id` (nullable FK → people, admin who created), `is_active`, `expires_us` (nullable), `notes` (nullable), `expiry_notified_us` (nullable), `created_us`, `updated_us`. Also added `kDatabaseInfoDefaultTrue` constant.
- [x] **Register in `make_database_info.cpp`**, `create_database.cpp` (`CreateTables`, `PopulateAdminTopLevelTables`, `PopulateAdminNestedTables`, column data info, friendly names, table friendly names, display templates)
- [x] **Add to `CMakeLists.txt`** (db_schema)
- [x] **Tests** — table creation verified via existing `GlobalDatabaseTestSupport`

### 1.2 Database Schema — `voucher_redemptions` table

- [x] **Create `db_schema/voucher_redemptions.h/cpp`** — Column constants: `id`, `voucher_id` (FK → vouchers), `purchase_id` (FK → purchases), `payment_id` (FK → payments), `redeemed_cents`, `created_us`
- [x] **Register in `make_database_info.cpp`**, `create_database.cpp` (as nested table under vouchers)
- [x] **Add to `CMakeLists.txt`** (db_schema)

### 1.3 Table Helpers

- [x] **Create `sql_util/table_helpers/vouchers.h/cpp`** — `AddVoucher`, `GetVoucherByCode`, `GetVoucherById`, `UpdateRemainingValue`, `DeactivateVoucher`, `GetVouchers`, `SetExpiryNotified`
- [x] **Create `sql_util/table_helpers/voucher_redemptions.h/cpp`** — `AddRedemption`, `GetRedemptionsForVoucher`, `GetRedemptionsForPurchase`
- [x] **Add to `CMakeLists.txt`** (table_helpers)
- [x] **Tests** — `vouchers_test.cpp` (8 tests), `voucher_redemptions_test.cpp` (6 tests)

### 1.4 Business Logic — VoucherHelper

- [x] **Create `business_logic/payment/voucher_helper.h/cpp`** — Contains:
  - `GenerateVoucherCode()` — generates a unique code (format: `KY-XXXX-XXXX` using alphanumeric chars, excludes ambiguous I/O/0/1)
  - `ValidateVoucher(transaction, code, personId)` — checks exists, is_active (handles 't'/'true'), not expired, remaining > 0; if `issued_to_person_id` is set, verifies it matches `personId`; returns VoucherInfo or nullopt with error message
  - `RedeemVoucher(transaction, code, amountCents, purchaseId, paymentId)` — atomically decrements `remaining_value_cents`, creates `voucher_redemptions` row; returns redeemed amount (capped at remaining)
  - `CreateVoucher(transaction, request)` — creates a new voucher; auto-generates code if not provided, validates uniqueness if admin-specified
  - `CreateStoreCredit(transaction, personId, amountCents, currency, reason)` — creates a person-tied voucher (for refund-to-credit flow)
- [x] **Add to `CMakeLists.txt`** (business_logic/payment)
- [x] **Tests** — `voucher_helper_test.cpp` (16 tests: generate code format/uniqueness, create with auto/custom/duplicate code, validate active/inactive/depleted/nonexistent/expired/person-check, redeem full/partial/capped, create store credit)

### 1.5 Admin Endpoints — Voucher Management

- [x] **Create `endpoints/admin_vouchers.h/cpp`** — Endpoints:
  - `POST /api/admin/create_voucher` — creates a voucher (amount, currency, optional code, optional person, optional expiry, notes); sets `issued_by_person_id` from admin session
  - `GET /api/admin/vouchers` — lists all vouchers with remaining balances
  - `POST /api/admin/deactivate_voucher/:id` — deactivates a voucher
  - `GET /api/admin/voucher/:id/redemptions` — lists redemption history
- [x] **Register in `web_app.cpp`**
- [x] **Add to `CMakeLists.txt`** (endpoints)
- [x] **Tests** — `admin_vouchers_test.cpp` (6 tests: create success, create requires admin, create with custom code, get list, deactivate, get redemptions)

### 1.6 Admin UI — Voucher Management

- [x] **Create `pages/manage/vouchers/` component** — Table of vouchers (code, type [voucher/credit], initial/remaining value, status, person, expiry), Create Voucher form with dollar amount input (converts to cents), Deactivate button, expandable redemption history
- [x] **Add route** in manage routing (`/manage/vouchers`)
- [x] **Add link** in manage dashboard (card_giftcard icon, "Vouchers", "Manage gift cards and store credits")
- [x] **ServerAccess methods** — `adminCreateVoucher`, `adminGetVouchers`, `adminDeactivateVoucher`, `adminGetVoucherRedemptions` (interface, proxy, network, mock)
- [x] **Tests** — component spec (22 tests), mock spec (6 tests). All 1625 Angular tests pass.

---

## Phase 2: Voucher Payment & Split Payments (Scenarios 22, 23, 25)

### 2.1 PaymentHelper — Voucher Payment Method

- [x] **Update `PaymentHelper`** — Add `PayWithVoucher(transaction, purchaseId, voucherCode)`:
  - Validates voucher via VoucherHelper
  - Calculates amount to redeem (min of voucher balance, remaining purchase balance)
  - Creates payment row with `provider="voucher"`, `provider_payment_id=voucher_code`
  - Calls `VoucherHelper::RedeemVoucher`
  - Links payment to purchase via `purchase_payments`
  - Updates `purchases.paid_cents`
  - If fully paid, triggers entitlement creation
- [x] **Tests** — full voucher payment, partial voucher payment, voucher + card split, invalid code, currency mismatch, wrong person store credit, missing code (7 tests in `payment_helper_test.cpp`)

### 2.2 Split Payment Support

- [x] **Create `POST /api/purchase_pay_voucher/{purchaseId}`** endpoint — accepts `{ "voucher_code": "..." }`, applies voucher to purchase
- [x] **Update purchase status logic** — `partially_funded` when voucher covers part, `funded` when fully covered (handled by existing `PurchaseHelper::RecordPayment`)
- [x] **Allow card payment on partially-funded purchase** — fixed `purchase_pay_card` to accept both `pending` and `partially_funded` status
- [x] **Tests** — voucher covers full amount, partial payment, voucher then card split, invalid code, missing code, not authenticated, expired voucher (7 tests in `purchase_pay_voucher_test.cpp`)

### 2.3 Frontend — Voucher Code Entry in Checkout

- [x] **Update payment control** — add "Apply Voucher" section with code input field and Apply button in checkout component
- [x] **Show applied voucher** — display redeemed amount, remaining purchase balance with green banner
- [x] **ServerAccess methods** — `purchasePayVoucher(purchaseId, voucherCode)` and `checkVoucher(code)` added to interface, proxy, network, mock
- [x] **Conditional card payment** — if voucher covers full amount, hide card entry and pay button, go straight to success; otherwise show card form for remaining balance
- [x] **Tests** — 4 checkout component spec tests (full cover, partial, error, hide card form); 6 mock spec tests

### 2.4 Frontend — Voucher Balance Check

- [x] **Create `GET /api/check_voucher/{code}`** endpoint — returns voucher status, remaining balance (requires login, validates person for store credits). Returns `{ valid: true/false, ... }` with HTTP 200 (not error)
- [x] **ServerAccess + mock** — `checkVoucher(code)` added to all layers
- [x] **Tests** — 6 endpoint tests (valid, nonexistent, expired, wrong person, not auth, correct person store credit); mock spec tests

### 2.5 Refund to Store Credit

- [x] **Update `RefundHelper`** — added `refundAsCredit` parameter to `ProcessRefund`; when true, creates a person-tied voucher via `VoucherHelper::CreateStoreCredit` instead of Square RefundPayment, records payment with provider="voucher"
- [x] **Update cancellation endpoints** — added `refund_as_credit` body parameter to `cancel_booking`; threaded through `BookingHelper::CancelBooking` to all `ProcessRefund` calls; response includes `store_credit_voucher_id` and `store_credit_voucher_code`
- [x] **Update cancellation UI** — my-events component now shows "Refund as store credit instead of card refund" checkbox in cancel confirmation; refund message includes store credit voucher code when applicable
- [x] **Tests** — 2 refund_helper tests (store credit creates voucher, records voucher payment); 1 cancel_booking endpoint test (refund_as_credit=true)

---

## Phase 3: Comps & Admin-Granted Entitlements (Scenarios 28, 29)

### 3.1 System Purchase for Comps

- [x] **Create comp purchase flow** — creates a `$0 system purchase`:
  - `purchases` row: `total_cents=0`, `paid_cents=0`, `status="funded"`, `payer_person_id=target person`
  - `payments` row: `provider="comp"`, `amount_cents=0`, `status="COMPLETED"`, unique `provider_payment_id`
  - `purchase_payments` linking row
  - Creates entitlement directly via `CreateEntitlement` (bypasses purchase_items since no price schedule for comps)
  - Auto-assigns entitlement to target person
- [x] **Created `CompHelper`** — `CreateComp(transaction, request)` orchestrates the above. Two constructors (with/without mail). Validates product exists and is active.
- [x] **Tests** — 7+ tests in `comp_helper_test.cpp`: success, invalid product, inactive product, invalid person, sends email, works without mail helper, default notes

### 3.2 Admin Comp Endpoint

- [x] **Create `POST /api/admin/comp`** endpoint — accepts `{ "product_id": N, "person_id": N, "notes": "reason" }`; creates $0 purchase + payment + entitlement
- [x] **Register in `web_app.cpp`**
- [x] **Tests** — 6 endpoint tests in `admin_comp_test.cpp`: success, requires admin, product not found, not authenticated, missing product_id, missing person_id

### 3.3 Comp Notification Email

- [x] **Create `business_logic/payment/comp_notification_mail.h/cpp`** — email template notifying recipient that a product has been comped to them (product name, notes from admin). Follows payment_confirmation_mail pattern with FormatString + NormalizeCrLf.
- [x] **Send email from comp flow** — CompHelper sends notification after entitlement creation
- [x] **Tests** — 5 tests in `comp_notification_mail_test.cpp`: contains product name, first name, notes when present, omits notes when empty, valid HTML structure

### 3.4 Admin UI — Comp

- [x] **Created standalone comp page** at `/manage/comps` with product dropdown (active products only), person autocomplete picker, optional notes field, and submit button. Dashboard card with "redeem" icon added.
- [x] **ServerAccess methods** — `adminComp(productId, personId, notes)` added to interface, proxy, network, mock
- [x] **Tests** — 11 component spec tests (create, load products, render form, submit validation, success/error messages, person autocomplete), 2 mock spec tests (success, auth), manage route and dashboard card

---

## Phase 4: Prorated Subscription Refunds (Scenario 27)

### 4.1 Prorated Refund Calculation

- [ ] **Add `CalculateProratedRefund` to `RefundHelper`** — given a subscription billing period (start/end dates) and cancellation date, calculate the unused portion:
  - `refundCents = (daysRemaining / totalDays) * periodAmountCents`
  - Round down to avoid over-refunding
- [ ] **Tests** — mid-period cancellation, last day, first day, already expired

### 4.2 User Subscription Cancellation (No Refund, Access Through Period End)

- [ ] **Update user subscription cancellation flow** — when user cancels mid-period:
  - Mark subscription as cancelled (no new billing)
  - Entitlement remains active until current period ends (no revocation)
  - No refund issued
  - Optionally offer refund-as-credit if desired (future enhancement)
- [ ] **Tests** — user cancel mid-period: subscription cancelled, entitlement still active, no refund

### 4.3 Admin Subscription Cancellation (Immediate Revoke + Prorated Refund)

- [ ] **Update admin subscription cancellation flow** — admin override option:
  - Admin can choose "Cancel immediately with prorated refund"
  - Calculates prorated refund via `CalculateProratedRefund`
  - Processes refund via `RefundHelper::ProcessRefund` (to card or as store credit)
  - Revokes entitlement immediately
- [ ] **Update admin subscription UI** — add "Cancel with refund" option alongside existing cancel
- [ ] **Tests** — admin cancel with prorated refund, entitlement revoked, refund amount correct

---

## Phase 5: Coupons (Scenario 24)

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

## Phase 6: Voucher Expiry Notifications

### 6.1 Expiry Notification Email

- [ ] **Create `business_logic/payment/voucher_expiry_mail.h/cpp`** — email template warning user their voucher/credit is about to expire (code, remaining balance, expiry date)
- [ ] **Tests** — email body generation test

### 6.2 Scheduled Expiry Check

- [ ] **Create expiry check logic** — query for vouchers where `expires_us` is within a notification window (e.g., 7 days) and `is_active = true` and `remaining_value_cents > 0`
- [ ] **Send notifications** — for each expiring voucher, send email to the holder (bearer vouchers: `issued_by_person_id` or skip; store credits: `issued_to_person_id`)
- [ ] **Track notification sent** — add `expiry_notified_us` column to `vouchers` to avoid duplicate notifications
- [ ] **Trigger mechanism** — either a cron endpoint (`POST /api/admin/process_voucher_expiry`) or integrate into an existing periodic task
- [ ] **Tests** — expiry within window gets notification, already notified skipped, expired vouchers not notified

### 6.3 Expiry Enforcement

- [ ] **On redemption, check expiry** — `ValidateVoucher` already checks `expires_us`; expired vouchers return error
- [ ] **Deactivate expired vouchers** — the scheduled check can also set `is_active = false` on vouchers past `expires_us` (balance forfeited)
- [ ] **Tests** — expired voucher cannot be redeemed

---

## Resolved Questions

| # | Question | Decision |
|---|----------|----------|
| 1 | **Voucher code format** | Both. Auto-generate a code (e.g., `KY-XXXX-XXXX`) but let the admin replace it with a custom one. Server validates uniqueness on custom codes. |
| 2 | **Voucher scope** | Vouchers are **bearer** (not tied to a person — anyone with the code can use it). Store credits are **person-tied** (`issued_to_person_id` required). These are two distinct use cases on the same `vouchers` table, distinguished by whether `issued_to_person_id` is set. |
| 3 | **Refund to voucher** | Support both: refund to card (existing) **and** refund as store credit (creates a person-tied voucher). The refund flow will offer a choice. |
| 4 | **Subscription cancellation timing** | **User cancel**: access continues through end of period, no refund (option B). **Admin cancel**: option to revoke immediately with prorated refund (option A) as an override. |
| 5 | **Coupon priority** | Coupons are in scope — keep Phase 5, not optional. |
| 6 | **Comp notification** | Yes. Send an email to the recipient when an admin comps a product. |
| 7 | **Voucher expiry behavior** | Notify user before expiry (scheduled job). If they don't use it, the remaining balance is forfeited at expiry. |