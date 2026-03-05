---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 1/29/2026
Version: 0.1
tags: 
---
# Overview

I'm working with you on a design document for my payment system. That document is Payment Design Document.md in this directory. I have friends reviewing the document and on reviewer suggested adding use cases / customer scenarios. I like this idea and think we could add this after section 7 Data Model and before section 8 Low Level Design. I want to workshop what to put there in this document and once I'm happy, we can have you put the contents in that section of the design document but I want to choose that. 

I'd like to start by doing an outline of all the user scenarios in this document. After we capture all the descriptions of the scenarios, we can show how our set of tables and system design enables each of these scenarios.

---

# Prioritized User Scenarios

## Must Have (Phase 1: Thin Slice)
*Core payment flow - minimum viable product*

1. User purchases a one time item for themself like an intro workshop or massage
2. User tries to pay for a service and their card is declined and they need to retry with a different card
3. Users should be able to view purchase history
4. Users should be able to view payment history

## Should Have (Phases 2-3: Multi-Seat & Permission Pricing)
*Multi-person purchases and member pricing*

5. User purchases a one time item for someone else like a massage or intro workshop
6. User purchases a one time item for themself and one or more other people like a couple's massage
7. User purchases a bundled product (e.g., massage + spa entry as a single product)
8. User who has purchased a monthly membership receives a discount for other services (permission-based pricing)
9. User reassigns seat to a different person (e.g., user buys party package and different guests show up)

## Nice to Have (Phase 4: Subscriptions)
*Recurring billing and card management*

10. User subscribes for a monthly service like Knotty Yoga Platinum
11. User pays for a monthly subscription service for another user to receive
12. User subscribes for a multi-seat monthly service (e.g., Couples Membership)
13. User elects to cancel their subscription at the end of the month
14. User has not paid for renewal - grace period before losing benefits
15. User on subscription with card declined - grace period, notified to update payment
16. New user buys membership for next month and gets remainder of current month free
17. User saves a card on file for future payment
18. User updates/removes a card on file
19. User changes default payment method
20. User receives expiring entitlement reminder
21. User receives expiring card notification

## Handle in Future (Phases 5-6: Vouchers & Refunds)
*Gift cards, coupons, and refunds*

22. User redeems a voucher/gift card and fully consumes its value
23. User redeems a voucher/gift card and has remaining balance
24. User uses a percentage-based discount coupon
25. User pays with multiple payment sources (card + voucher, two cards)
26. Studio refunds a one time purchase item
27. User cancels a monthly service and receives a prorated refund
28. Studio comps a service or good
29. Admin grants entitlement without payment (comp)
30. Admin credits user account for future purchase (as voucher)

## Low Priority (Post-MVP)
*Complex edge cases and admin features*

31. User upgrades to a higher tier membership (prorated)
32. User downgrades to a lower tier membership (effective next cycle)
33. User reactivates after cancelling membership (history preserved)
34. User is gifted something but passes it on to someone else
35. Admin grants extension of a membership as reward
36. Credit card chargeback - invalidate payment, outstanding balance
37. Guest passes - member gifts discounted/free guest pass to another person
38. Product discontinued - existing entitlements remain valid, no new purchases
39. Entitlement expires while user is mid-session

# Table Support by Scenario

## Must Have (Phase 1)

### 1. User purchases a one time item for themself
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `products` | id, code, name, kind="one_time", is_active |
| `price_schedules` | id, valid_from_us, valid_to_us, is_active |
| `product_prices` | product_id, price_schedule_id, permission_id=NULL, amount_cents |
| `purchases` | id, payer_person_id, status, total_cents, paid_cents |
| `purchase_items` | purchase_id, product_id, quantity, unit_price_cents, line_total_cents |
| `payments` | id, provider="square", status, amount_cents, payer_person_id |
| `purchase_payments` | purchase_id, payment_id, applied_cents |
| `product_entitlement_rules` | product_id, seats_default=1, validity_kind |
| `entitlements` | purchase_id, product_id, valid_from_us, valid_to_us, seats_total=1, status |
| `entitlement_assignments` | entitlement_id, person_id (same as payer) |

### 2. Card declined, retry with different card
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `payments` | status="failed" for declined attempt |
| `idempotency_keys` | New key for retry attempt |

Server returns error response; client retries with new card token and new idempotency key.

### 3. View purchase history
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `purchases` | Query by payer_person_id |
| `purchase_items` | Join on purchase_id for line items |
| `products` | Join for product names |

### 4. View payment history
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `payments` | Query by payer_person_id |
| `purchase_payments` | Join to show which purchases each payment funded |

---

## Should Have (Phases 2-3)

### 5. User purchases for someone else
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `purchases` | payer_person_id = buyer |
| `entitlement_assignments` | person_id = recipient (different from payer) |

Same as scenario 1, but assignment goes to different person.

### 6. Multi-seat purchase (couple's massage)
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `product_entitlement_rules` | seats_default > 1 |
| `entitlements` | seats_total = seats_default from rule |
| `entitlement_assignments` | Multiple rows, one per person |

Trigger `enforce_entitlement_seats` prevents exceeding seats_total.

### 7. Bundled product (massage + spa combo)
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `products` | Single product representing the bundle |
| `product_entitlement_rules` | One rule for the bundle |

**Note**: Bundle is a single product. If bundle needs to grant multiple distinct entitlements, would need `product_entitlement_rules` to allow multiple rules per product (currently unique on product_id).

### 8. Permission-based pricing (member discount)
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `product_prices` | permission_id links to member permission |
| `product_entitlement_rules` | grants_permission_id on membership product |
| `entitlements` | Active membership entitlement |
| `entitlement_assignments` | User assigned to membership |

Effective permission computation finds member permission → matches product_prices row with lower amount_cents.

### 9. Reassign seat to different person
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `entitlement_assignments` | Set removed_us, removed_reason on old assignment |
| `entitlement_assignments` | Create new row for new person |

---

## Nice to Have (Phase 4)

### 10. Subscribe to monthly service
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `products` | kind="subscription" |
| `square_subscriptions` | person_id, product_id, square_subscription_id, status, current_period_start_us, current_period_end_us |
| `square_customers` | Links person to Square customer ID |
| `entitlements` | Created per billing period via webhook |

### 11. Subscribe for another user
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `square_subscriptions` | person_id = payer (subscription owner) |
| `entitlement_assignments` | person_id = beneficiary |

### 12. Multi-seat subscription (Couples Membership)
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `product_entitlement_rules` | seats_default > 1 |
| `entitlements` | seats_total from rule |
| `entitlement_assignments` | Multiple rows per entitlement |

### 13. Cancel subscription at end of month
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `square_subscriptions` | canceled_us set, status remains "active" until period ends |

Square handles not charging next period.

### 14. Grace period before losing benefits
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `square_subscriptions` | grace_period_ends_us, status="delinquent" |
| `entitlements` | Still valid while grace_period_ends_us > now |

### 15. Subscription card declined, grace period
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `square_webhook_events` | Captures payment failure event |
| `square_subscriptions` | status="delinquent", grace_period_ends_us set |

Same as 14, triggered by webhook.

### 16. Buy next month, get current month free
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `entitlements` | valid_from_us = start of current month (not next month) |

Business logic decision at entitlement creation time.

### 17. Save card on file
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `square_customers` | person_id, square_customer_id |
| `square_cards` | person_id, square_card_id, brand, last4, exp_month, exp_year, is_active |

### 18. Update/remove card on file
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `square_cards` | is_active=false to remove; update exp fields for update |

### 19. Change default payment method
**Partially Supported** ⚠️

| Table | Fields Used |
|-------|-------------|
| `square_cards` | Need to add `is_default` column |

**Gap**: No `is_default` column on `square_cards`. Add: `is_default BOOLEAN DEFAULT FALSE`.

### 20. Expiring entitlement reminder
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `entitlements` | valid_to_us queried by scheduled job |
| `entitlement_assignments` | person_id for email recipient |

### 21. Expiring card notification
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `square_cards` | exp_month, exp_year queried by scheduled job |

---

## Handle in Future (Phases 5-6)

### 22. Voucher fully consumed
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `vouchers` | remaining_value_cents becomes 0 |
| `voucher_redemptions` | voucher_id, purchase_id, payment_id, redeemed_cents |
| `payments` | provider="voucher" |

### 23. Voucher partial redemption
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `vouchers` | remaining_value_cents > 0 after redemption |
| `voucher_redemptions` | redeemed_cents < initial_value_cents |

### 24. Percentage-based discount coupon
**Not Supported** ✗

**Gap**: No `coupons` table. Would need:
```
coupons (id, code, discount_type ["percentage", "fixed"],
         discount_value, max_uses, current_uses,
         valid_from_us, valid_to_us, is_active)
coupon_redemptions (id, coupon_id, purchase_id, discount_cents)
```

### 25. Multiple payment sources
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `payments` | Multiple payment records |
| `purchase_payments` | Multiple rows linking payments to purchase |
| `purchases` | paid_cents accumulates until = total_cents |

### 26. Refund one-time purchase
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `payments` | New row with negative amount_cents, refund_for_payment_id, refund_reason |
| `entitlements` | revoked_us, revoked_reason set |

### 27. Prorated subscription refund
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `payments` | Negative amount_cents (prorated calculation) |
| `entitlements` | revoked_us set |
| `square_subscriptions` | status updated |

### 28. Comp service or good
**Partially Supported** ⚠️

| Table | Fields Used |
|-------|-------------|
| `payments` | provider="comp", amount_cents=0 |
| `purchases` | total_cents=0, paid_cents=0, status="funded" |

Works if we allow $0 purchases. Alternative: create entitlement directly without purchase.

### 29. Admin grants entitlement without payment
**Partially Supported** ⚠️

| Table | Fields Used |
|-------|-------------|
| `entitlements` | purchase_id is currently required FK |

**Gap**: `entitlements.purchase_id` is NOT NULL. Options:
1. Make purchase_id nullable for admin-granted entitlements
2. Create a "system" purchase with $0 total for audit trail (recommended)

### 30. Admin credits account (as voucher)
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `vouchers` | Admin creates voucher with initial_value_cents |

Admin UI would create voucher and optionally email code to user.

---

## Low Priority (Post-MVP)

### 31. Upgrade to higher tier (prorated)
**Partially Supported** ⚠️

| Table | Fields Used |
|-------|-------------|
| `square_subscriptions` | Update product_id, recalculate period |
| `entitlements` | Revoke old, create new with upgraded product |

**Gap**: Proration logic not defined. Square may handle via subscription update API.

### 32. Downgrade to lower tier
**Partially Supported** ⚠️

| Table | Fields Used |
|-------|-------------|
| `square_subscriptions` | Update product_id effective next period |
| `entitlements` | Current entitlement unchanged; next period gets lower tier |

Similar to upgrade but deferred to next billing cycle.

### 33. Reactivate after cancelling
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `square_subscriptions` | Create new subscription record or update status |
| `entitlements` | New entitlement created |

History preserved - old subscription/entitlement records remain.

### 34. Pass gift on to someone else
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `entitlement_assignments` | removed_us on original recipient |
| `entitlement_assignments` | New row for new recipient |

Same as scenario 9 (reassign seat).

### 35. Admin extends membership
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `entitlements` | Update valid_to_us to later date |

Or create new entitlement for extension period with notes explaining reason.

### 36. Credit card chargeback
**Partially Supported** ⚠️

| Table | Fields Used |
|-------|-------------|
| `payments` | status="chargedback" (new status value) |
| `entitlements` | revoked_us, revoked_reason |
| `purchases` | paid_cents reduced |

**Gap**: Need to add "chargedback" to payments.status enum. May need outstanding balance tracking if partial chargeback.

### 37. Guest passes
**Not Supported** ✗

**Gap**: No guest pass infrastructure. Would need:
```
guest_passes (id, granting_entitlement_id, owner_person_id,
              recipient_person_id, recipient_email,
              valid_from_us, valid_to_us, redeemed_us,
              created_us)
```
Plus logic to auto-grant passes when membership entitlement created.

### 38. Product discontinued
**Supported** ✓

| Table | Fields Used |
|-------|-------------|
| `products` | is_active = false |
| `entitlements` | Existing entitlements remain valid until valid_to_us |

Set `products.is_active = false`. Catalog queries filter by is_active, so product no longer appears for purchase. Existing entitlements are unaffected.

### 39. Entitlement expires while user is mid-session
**Supported** ✓ (Policy decision, not technical)

| Table | Fields Used |
|-------|-------------|
| `entitlements` | valid_to_us records exact expiration |
| Check-in system | Records check-in time |

The system records the expiration timestamp. Whether a user can finish their session after expiration is a policy decision enforced by the check-in/front desk system, not the payment system. Recommended policy: if user checked in while entitlement was valid, they can finish their session.

---

# Detailed Scenario Flows

## Must Have (Phase 1)

### Scenario 1: User purchases a one-time item for themself

**Actor**: Customer (logged in)
**Example**: User buys a 60-minute massage for $160

**Flow**:
1. User browses catalog → `GET /api/catalog_products`
   - Server queries `products` where `is_active = true`
   - Server resolves prices from `product_prices` using active `price_schedules` and user's effective permissions
   - Returns product list with resolved prices

2. User selects product and quantity → `POST /api/purchase_create`
   - Server creates `purchases` row (status="pending_payment", payer_person_id=user)
   - Server creates `purchase_items` row with snapshotted price
   - Returns purchase_id and totals

3. User enters card in Square Web Payments SDK
   - Client-side tokenization, returns source_id
   - No server involvement, card data never touches our server

4. User submits payment → `POST /api/purchase_pay_card/{purchase_id}`
   - Server checks `idempotency_keys` for duplicate
   - Server calls Square CreatePayment API
   - On success: creates `payments` row (provider="square", status="captured")
   - Creates `purchase_payments` linking payment to purchase
   - Updates `purchases.paid_cents`; if paid_cents >= total_cents, status="funded"
   - Creates `entitlements` row based on `product_entitlement_rules`
   - Creates `entitlement_assignments` row (person_id = payer_person_id)
   - Returns success + receipt

**Result**: User has entitlement for 60-minute massage they can redeem.

---

### Scenario 2: Card declined, retry with different card

**Actor**: Customer (logged in)
**Example**: User's first card is declined, they try another

**Flow**:
1. Steps 1-3 from Scenario 1 complete normally

2. User submits payment → `POST /api/purchase_pay_card/{purchase_id}`
   - Server calls Square CreatePayment API
   - Square returns CARD_DECLINED error
   - Server creates `payments` row with status="failed"
   - Returns error response: `{ "type": "payment_declined", "detail": "Your card was declined..." }`

3. Client shows error, prompts for different card
   - User enters new card in Square SDK, gets new source_id

4. User retries → `POST /api/purchase_pay_card/{purchase_id}` with NEW Idempotency-Key
   - Server processes as new payment attempt
   - Square accepts, creates successful payment
   - Flow continues as Scenario 1 step 4

**Key Point**: New idempotency key required for retry. Purchase remains in "pending_payment" until successful payment.

---

### Scenario 3: View purchase history

**Actor**: Customer (logged in)

**Flow**:
1. User navigates to purchase history → `GET /api/purchases`
   - Server queries `purchases` where `payer_person_id = current_user`
   - Joins `purchase_items` for line item details
   - Joins `products` for product names
   - Orders by created_us DESC
   - Returns list with pagination

**Response includes**: purchase_id, date, status, items purchased, totals, payment status

---

### Scenario 4: View payment history

**Actor**: Customer (logged in)

**Flow**:
1. User navigates to payment history → `GET /api/payments`
   - Server queries `payments` where `payer_person_id = current_user`
   - Joins `purchase_payments` and `purchases` to show what each payment funded
   - Orders by created_us DESC
   - Returns list with pagination

**Response includes**: payment_id, date, amount, provider, status, linked purchase(s)

---

## Should Have (Phases 2-3)

### Scenario 5: User purchases for someone else

**Actor**: Customer (logged in)
**Example**: User buys a massage gift for their friend

**Flow**:
1. Same as Scenario 1 steps 1-3

2. User submits payment with beneficiary specified → `POST /api/purchase_pay_card/{purchase_id}`
   - Request includes `beneficiary_person_id` or `beneficiary_email`
   - If email provided and person doesn't exist, create pending invitation

3. On successful payment:
   - Creates `entitlements` row (purchase.payer_person_id = buyer)
   - Creates `entitlement_assignments` row with `person_id = beneficiary` (not payer)

**Result**: Buyer paid, but friend has the entitlement to redeem the massage.

---

### Scenario 6: Multi-seat purchase (couple's massage)

**Actor**: Customer (logged in)
**Example**: User buys couples massage for themselves and partner

**Flow**:
1. Product "Couples Massage" has `product_entitlement_rules.seats_default = 2`

2. User purchases product (Scenario 1 flow)

3. On successful payment:
   - Creates `entitlements` row with `seats_total = 2`
   - Creates first `entitlement_assignments` for buyer
   - User is prompted to assign second seat

4. User assigns second seat → `POST /api/entitlement_assign/{entitlement_id}`
   - Request includes `person_id` or `email` for partner
   - Server checks: count of active assignments < seats_total
   - Creates second `entitlement_assignments` row
   - Trigger `enforce_entitlement_seats` prevents over-assignment

**Result**: Both buyer and partner have assignments; both can redeem.

---

### Scenario 7: Bundled product purchase

**Actor**: Customer (logged in)
**Example**: User buys "Massage + Spa Day Combo" for $200 (normally $160 + $60 = $220)

**Flow**:
1. Bundle exists as single `products` row: "Massage + Spa Day Combo"
2. Single `product_prices` row with amount_cents = 20000
3. Single `product_entitlement_rules` row defining what access it grants

4. Purchase flow identical to Scenario 1

**Note**: If bundle needs to grant multiple distinct entitlements (massage AND spa access), would need to either:
- Create single entitlement that grants access to both (simpler)
- Modify schema to allow multiple entitlement rules per product (more complex)

---

### Scenario 8: Permission-based pricing (member discount)

**Actor**: Customer with active Platinum membership
**Example**: Platinum member buys massage at $140 instead of $160

**Flow**:
1. User has active Platinum membership
   - `entitlements` row exists: product_id = Platinum, status = "active", valid dates include now
   - `entitlement_assignments` links user to this entitlement
   - `product_entitlement_rules` for Platinum has `grants_permission_id` = "platinum_member"

2. User browses catalog → `GET /api/catalog_products`
   - Server computes effective permissions (role permissions ∪ entitlement-derived permissions)
   - Finds user has "platinum_member" permission
   - For massage product, finds `product_prices` rows:
     - permission_id = NULL, amount_cents = 16000 (public price)
     - permission_id = "platinum_member", amount_cents = 14000 (member price)
   - Returns massage at $140

3. Purchase flow continues with member price locked in
   - `purchase_items.pricing_permission_id` records which permission matched

---

### Scenario 9: Reassign seat to different person

**Actor**: Customer (logged in)
**Example**: User bought party package for 5, but one guest can't make it

**Flow**:
1. User views their entitlement → `GET /api/entitlements`
   - Returns entitlement with list of current assignments

2. User removes original guest → `POST /api/entitlement_unassign/{assignment_id}`
   - Server sets `entitlement_assignments.removed_us = now`
   - Server sets `removed_reason = "Reassigned by owner"`

3. User assigns new guest → `POST /api/entitlement_assign/{entitlement_id}`
   - Server checks seats_total > active assignments
   - Creates new `entitlement_assignments` row

**Result**: Original guest no longer has access; new guest does.

---

## Nice to Have (Phase 4)

### Scenario 10: Subscribe to monthly service

**Actor**: Customer (logged in)
**Example**: User subscribes to Knotty Yoga Platinum for $99/month

**Flow**:
1. User selects subscription product → `POST /api/subscription_start`
   - Server creates/retrieves `square_customers` record for user
   - Redirects to Square-hosted checkout or returns client token

2. User completes Square checkout
   - Square creates subscription, charges first month
   - Square sends webhook to `POST /api/webhook_square`

3. Server handles webhook:
   - Validates signature
   - Checks `square_webhook_events` for idempotency
   - Creates `square_subscriptions` row (status="active", period dates)
   - Creates `payments` row for first charge
   - Creates `entitlements` row for current billing period
   - Creates `entitlement_assignments` for subscriber

4. Each month, Square auto-charges and sends webhook
   - Server creates new payment and extends/creates entitlement for new period

---

### Scenario 11: Subscribe for another user

**Actor**: Customer (logged in)
**Example**: Parent buys Platinum membership for their adult child

**Flow**:
1. Same as Scenario 10, but with beneficiary specified
   - `square_subscriptions.person_id` = payer (parent)
   - `entitlement_assignments.person_id` = beneficiary (child)

2. Parent is billed; child receives benefits

3. Parent can manage (cancel, update payment) via their account

---

### Scenario 12: Multi-seat subscription

**Actor**: Customer (logged in)
**Example**: User subscribes to Couples Platinum for themselves and partner

**Flow**:
1. Product "Couples Platinum" has `product_entitlement_rules.seats_default = 2`

2. Subscription flow as Scenario 10

3. On entitlement creation:
   - `entitlements.seats_total = 2`
   - First assignment created for subscriber
   - User prompted to assign second seat (or can do later)

4. Each billing period, entitlement created with 2 seats

---

### Scenario 13: Cancel subscription at end of month

**Actor**: Customer (logged in)
**Example**: User cancels Platinum, wants to keep access until period ends

**Flow**:
1. User requests cancellation → `POST /api/subscription_cancel/{subscription_id}`
   - Server calls Square Cancel Subscription API with `cancel_at_period_end = true`
   - Updates `square_subscriptions.canceled_us = now`
   - Status remains "active" until period ends

2. User retains access through `current_period_end_us`

3. When period ends, Square sends webhook
   - Server updates status to "canceled"
   - Entitlement naturally expires (valid_to_us = period end)

---

### Scenario 14: Grace period before losing benefits

**Actor**: System / Customer
**Example**: Subscription renewal fails, user has 7 days to fix payment

**Flow**:
1. Square attempts renewal charge, fails
2. Square sends payment.failed webhook

3. Server handles webhook:
   - Updates `square_subscriptions.status = "delinquent"`
   - Sets `grace_period_ends_us = now + 7 days`
   - Sends "payment failed" email to user

4. During grace period:
   - User's entitlement remains valid (permission check includes grace period logic)
   - User can update payment method

5. If user fixes payment within grace:
   - Square retries charge successfully
   - Server clears `grace_period_ends_us`, sets status = "active"

6. If grace period expires:
   - Scheduled job or next check finds expired grace
   - User loses access (entitlement no longer considered valid)

---

### Scenario 15: Subscription card declined, grace period triggered

**Flow**: Same as Scenario 14 - this is the trigger for the grace period.

---

### Scenario 16: Buy next month, get current month free

**Actor**: New customer
**Example**: User signs up Jan 15, pays for February, gets rest of January free

**Flow**:
1. User subscribes (Scenario 10 flow)

2. Business logic decision at entitlement creation:
   - Instead of `valid_from_us = Feb 1`, set `valid_from_us = now`
   - `valid_to_us = end of February` (next full billing period)
   - User effectively gets Jan 15-31 free

3. This is a pricing/promotional decision, not a schema change

---

### Scenario 17: Save card on file

**Actor**: Customer (logged in)
**Example**: User saves card for faster future checkout

**Flow**:
1. User goes to payment methods → initiates add card

2. Client uses Square Web Payments SDK to tokenize card

3. Client sends token → `POST /api/cards`
   - Server creates/retrieves `square_customers` record
   - Server calls Square CreateCard API with customer_id and source_id
   - Server creates `square_cards` row with card details (brand, last4, exp)

4. Card available for future purchases without re-entering

---

### Scenario 18: Update/remove card on file

**Actor**: Customer (logged in)

**Update Flow**:
1. User can't really "update" a card - they add a new card and remove the old one
2. For expiration updates, Square may handle automatically

**Remove Flow**:
1. User requests removal → `DELETE /api/cards/{card_id}`
   - Server calls Square DisableCard API
   - Server sets `square_cards.is_active = false`

---

### Scenario 19: Change default payment method

**Actor**: Customer (logged in)

**Flow**:
1. User selects card to make default → `POST /api/cards/{card_id}/make_default`
   - Server sets `is_default = false` on all user's cards
   - Server sets `is_default = true` on selected card

**Note**: Requires adding `is_default` column to `square_cards` table.

---

### Scenario 20: Expiring entitlement reminder

**Actor**: System (scheduled job)
**Example**: User's membership expires in 7 days, send reminder email

**Flow**:
1. Scheduled job runs daily

2. Query: find entitlements where `valid_to_us` is within 7 days AND status = "active" AND no renewal pending

3. For each:
   - Look up `entitlement_assignments` to find beneficiaries
   - Send reminder email to each beneficiary

---

### Scenario 21: Expiring card notification

**Actor**: System (scheduled job)
**Example**: User's saved card expires next month

**Flow**:
1. Scheduled job runs monthly

2. Query: find `square_cards` where `exp_year/exp_month` is current or next month AND `is_active = true`

3. For each:
   - Send email to card owner: "Your card ending in {last4} expires soon. Please update your payment method."

---

# Design Decisions

## Add-ons / Bundles (Scenario 7)
**Question**: Is the spa entry a separate product at a discounted price, or included in the massage product?

**Recommendation**: Create bundled products rather than conditional discounts on one-time purchases.

| Approach                                                          | Pros                                                         | Cons                                                                             |
| ----------------------------------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| **Bundled product** (e.g., "Massage + Spa Combo")                 | Simple to implement; clear pricing; no permission complexity | More products to manage; less flexible                                           |
| **Conditional discount** (permission granted by massage purchase) | Flexible; fewer products                                     | Complex timing (permission only valid same day?); harder to explain to customers |

**Decision**: Use bundled products. Simpler, and bundles are a well-understood retail concept.

## Vouchers vs Coupons
**Vouchers/Gift Cards**: Stored monetary value, can be partially redeemed, tracked in `vouchers` table
**Coupons**: Percentage or fixed discount, single-use or limited-use, would need a separate `coupons` table

For Phase 5, focus on vouchers first. Coupons can be added later if needed.

## Guest Passes (Scenario 37)
**Question**: Are guest passes a separate entitlement type, or limited-use vouchers?

| Approach                                            | Pros                                              | Cons                                           |
| --------------------------------------------------- | ------------------------------------------------- | ---------------------------------------------- |
| **Separate entitlement type** with `uses_remaining` | Clear model; easy to query "how many passes left" | New column on entitlements; mixed semantics    |
| **Limited-use vouchers** auto-granted by membership | Reuses voucher infrastructure; flexible value     | Vouchers are monetary; passes are access-based |
| **Dedicated `guest_passes` table**                  | Clean separation; can track who used each pass    | Another table; more complexity                 |

**Recommendation**: Dedicated `guest_passes` table (Low Priority). Guest passes have unique properties:
- Granted automatically by membership entitlement
- Limited quantity per period (e.g., 2/month)
- Can be gifted to specific person
- Track redemption (who, when, for what)

This doesn't fit cleanly into vouchers (which are monetary) or entitlements (which are access for the holder). Defer to post-MVP.

---

# Deferred Considerations

These came up during analysis but are out of scope for now:

- **Price increases for existing subscribers (grandfathered pricing)** - Not supporting this.

- **Product discontinued** - Now scenario 38. Handled by setting `products.is_active = false`.

- **Entitlement expires while in use** - Now scenario 39. Policy decision for check-in system, not payment system.

---

# Next Steps
1. ~~Review prioritized scenarios above~~ ✓ Done
2. ~~Confirm priority assignments make sense~~ ✓ Done
3. ~~Write detailed flows for Must Have, Should Have, and Nice to Have~~ ✓ Done (scenarios 1-21)
4. Review detailed flows for accuracy
5. ~~Transfer to Payment Design Document~~ ✓ Done - Added as Section 8 with subsections 8.1-8.5