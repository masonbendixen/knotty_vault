---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/6/2026
Version: 0.3
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

We had a previous work item, [[Payment Should Have- Multi Seat and Bundled Pricing]] to implement the Should Have section of [[Payment Design Document]]. Unfortunately, scenarios 6 and 7 were not completed as part of that item. I would like to tackle them here.

Let's add a couple's massage product with variants for 60, 90, and 120min. We should rename the membership sharing thing to purchase sharing in the user console but keep the functionality the same. Booking a couple's massage would be a new thing under bookable services. Bookable services currently has separate providers listed with their available timeslots. For couple's massage, we would have a   similar UI but it would have slots listed where there are at least two providers available at the same time. Upon choosing a slot in the UI, they would select the other person as part of the couple's massage and, for each person, choose a provider. Note that the assignment of seats should be like membership (ie. the person booking the two person service can pay for it but could do it for two other people like a person buying it for their parents). The purchase would be under the person paying but the booking would show up under the two people recieving the service. If either user or either therapist cancels, the whole thing is cancelled for both people scheduled and both providers. If either provider cancels, the whole thing is refunded.

For bundled products, there should be base products and add ons. Add on means a discount on the price of the add on product if purchased with the base product. For each add on, we should keep track of if the discount is stackable (ie. can there be multiple discounts on this product based on it being combined with other discounts). The default should be no for stackable. Let's add a spa entry one time product. We also need to add a room to the facility called "Spa" that is a sibling to main gym. We should have different variants of spa entry for different time lengths. We should also have different variants also for early bird (ie. certain days of the week for early, less popular hours). I would like a separate product that is "Late night, post workout spa access" that is a last hour thing on certain weeknights after the last workout class finished for muscle recovery. I would like users to be able to book these at five minute intervals during available blocks. We need to have a spa capacity that is tracked. We should probably make this a property of rooms since massage room should have capacity of 1 (unless we later create a room with two tables for couple massage at a later date), the main gym should have a max capacity, and so should the spa. We should allow people to do bookings at start intervals that they like as long as there is space in the gym for the whole interval time being booked (we should not show time slots that would cause a portion of the booking to exceed capacity). We need an entry in the staff portal to check in people for their spa visit. They are allowed to check in up to a product configurable window before their booked time (default to 15min). The duration of the visit is based on their checkin time. If checking in early would exceed spa capacity, they must wait until their is space. Checking in late does not extend their visit length since that would cause complications for capacity for already booked clients. People are allowed to book slots that would not allow the full time because the space will close before the whole time is passed. There will not be a discount or prorating but they will be warned. We also need to allow drop ins where the staff portal has the ability to create a booking on the fly if there is space available as well as creating an account if the person does not have one. This will involve collecting first and last name as well as email, creating the account, and automatically generating a password that is emailed to them that they should change immediately. We need a bundled option that can specify massage as the base product and spa entry as the add on. There will be a discount for the spa entry if purchased bundled with a massage. Also, the length of the spa visit is automatically extended by the length of the massage (which needs to be factored in for the checking of spa capacity during booking). Canceling either the base product or a bundled product cancels the whole booking and all components. Refunds are based on the refund policy for each product except if the provider cancels the massage, the WHOLE bundle is fully refunded regardless of individual refund policies. I'm not really sure about the best way to expose booking this to the user. Perhaps book the massage and have an Add Ons button that bring up a modal UI where you can see the add ons and then click on one like Spa Entry and then see if it available and have options of how far before the massage to start the spa entry (which has to start at or before the time of the massage). If a certain spa entry time would not allow full duration of the spa entry length because it runs into spa closure or a period of out of capacity, we could warn the user of such but allow them to still book it if they want to.

This is a pretty big item. Please use the code base and these documents for context:

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
- [[Vouchers and Refunds]]

Also, please create a section noting the changes we will need to make to [[Payment Design Document]] and [[Support for scheduled purchases]]. After we have locked down the design, the first phase of implementation should be to update those documents with the changes needed to support these features. Please start a discussion with me on ideas of how to expose choosing bundled products to link them and compatibly schedule them. Also, please critique what I have written, come up with other suggestions, and list possible other complimentary work that also seems related.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Plan: Spa, Add-On Bundles, Shopping Cart, and Book-for-Others

## Scope Summary (Post-Discussion)

Based on our brainstorming discussion, here's what's in scope and what's been decided:

### In Scope
1. **Room-based bookable products with capacity tracking** — room-level capacity enforcement, configurable slot intervals, variants for different durations/times-of-day. Applies to spa, swim spa (30-min increments), rock climbing wall (capacity-limited), and future similar products.
2. **Late Night Spa** — separate product (not variant) for visibility/permission control, platinum members only
3. **Staff check-in with capacity enforcement** — fire code compliance, early check-in window, capacity gate
4. **Add-on bundles** — `product_addons` table, automatic discounts, duration extension for spa when bundled with massage
5. **Shopping cart** — multi-item purchases with implicit add-on discount detection
6. **Booking for someone else** — extends purchase sharing to bookable services, with invite flow for non-members
7. **Drop-in bookings with on-the-fly account creation** — staff portal, auto-generated password with must-change flag
8. **Linked cancellation** — booking groups, cancel-all policy, special handling when provider cancels base
9. **Rename "Membership Sharing" to "Purchase Sharing"**
10. **iCal attachments** on booking confirmation emails
11. **Occupancy dashboard** — real-time for staff and read-only for users
12. **Fix PayCardResponse TypeScript type** — entitlements array, not count

### Deferred / Dropped
- **Couple's massage** — Dropped. Not worth the multi-provider slot complexity. Users can simply book two massages at the same time. The "book for someone else" feature covers the gifting use case.
- **Multi-visit passes** ("Buy 5 spa entries, get 1 free") — Good idea but separate work item
- **Recurring spa bookings** — Interesting but separate work item
- **Spa waitlist** — Deferred until capacity becomes an issue
- **Booking modification** (reschedule without cancel+rebook) — Nice-to-have, defer unless cheap
- **Bundle analytics dashboard** — Good idea but separate work item
- **Class attendance templates** — Very different feature, separate work item

### Key Design Decisions Made

**DD1: No virtual spa "provider" account.** The provider-based availability model doesn't fit self-service rooms (spa, gym). A room with capacity > 1 needs a different availability mechanism — likely room-level operating hours rather than provider availability blocks. The room's `concurrent_capacity` gates bookings; operating hours gate when bookings are possible. This needs a new concept: **room schedules** (operating hours per room).

**DD2: Spa variants are early bird, off-peak, peak. Late night is a separate product** with visibility restricted to platinum members via `visibility_permission_id`.

**DD3: Add-on discount vs coupon — better discount wins, unused one not consumed.** If a coupon gives 20% off and the add-on bundle gives 15% off, the 20% coupon applies and the bundle discount is not consumed. The coupon IS consumed. This requires tracking which discount was actually applied.

**DD4: Provider cancels base of bundle — don't cancel the add-on.** Cancel the massage, keep the spa booked at the bundle discount, give the customer free cancellation on the spa, and email them about the change. This is fairer than yanking both bookings.

**DD5: Duration extension is base variant duration.** 90-min massage + 60-min spa = 150-min spa visit. The `extends_duration_by_base` flag on `product_addons` controls this.

**DD6: Shopping cart for bundle UX.** Users book products independently into a cart. When items in the cart have add-on relationships, the discount is shown and applied automatically. This is the most flexible approach and enables multi-product purchases generally.

---

## Current State Analysis

### What Already Exists

| Component | Status | Relevant For |
|-----------|--------|--------------|
| `location_rooms.concurrent_capacity` | Exists | Spa capacity tracking |
| `product_variants` with `duration_minutes`, `buffer_minutes` | Exists | Massage/spa variants |
| `bookable_service_sessions` with `location_room_id` | Exists | Room assignment |
| `bookings.checked_in_us` | Column exists, no endpoint | Check-in tracking |
| `bookings.status` includes "attended", "no_show" | Exists | Attendance tracking |
| Service availability helper (5-min slot alignment) | Exists | Spa slot computation |
| `FindAvailableRoom()` with provider affinity | Exists | Room assignment |
| Cancellation with tiered refund policies | Exists | Cancel/refund |
| Gift permissions / seat assignment | Exists | Purchase sharing |
| Product booking windows (permission-based advance booking) | Exists | Early bird access |
| Coupon system with multi-product restriction | Exists | Discount infrastructure |
| `CreatePurchaseRequest` accepts `items[]` array | Exists | Multi-item purchases (backend ready) |
| `CatalogHelper::GetQuote` resolves prices for multiple items | Exists | Multi-item pricing |
| Account creation / registration flow | Exists | Drop-in account creation base |

### What Needs to Be Built

| Gap | Feature Area |
|-----|-------------|
| Room-level operating hours schedule | Spa availability (replaces virtual provider) |
| Room capacity check during slot computation | Spa overbooking prevention |
| Check-in endpoint + capacity enforcement | Staff check-in with fire code compliance |
| Must-change-password flag on accounts | Drop-in account creation |
| `product_addons` table + discount logic | Bundle pricing |
| `booking_groups` + `booking_group_members` | Linked cancellation |
| Shopping cart frontend | Multi-item purchase UX |
| Book-for-others flow with invite | Gifting / sharing |
| iCal generation + email attachment | Calendar integration |
| Occupancy dashboard (staff + public) | Real-time capacity view |

---

## Data Model Changes

### New Tables

#### `room_schedules`
Operating hours for rooms that can be self-service booked (spa, gym). Replaces the "virtual provider" concept.

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `location_room_id` | BIGINT | FK → location_rooms |
| `day_of_week` | SMALLINT | 0=Sunday, 6=Saturday |
| `open_time_minutes` | INT | Minutes from midnight (e.g., 360 = 6:00 AM) |
| `close_time_minutes` | INT | Minutes from midnight (e.g., 1320 = 10:00 PM) |
| `is_active` | BOOLEAN | DEFAULT TRUE |
| `created_us` | BIGINT | |
| `updated_us` | BIGINT | |

This table defines when a room is bookable. Multiple entries per room per day are allowed (e.g., 6am-12pm and 2pm-10pm with a maintenance gap).

#### `product_addons`
Defines which products can be add-ons to which base products, and the discount.

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `base_product_id` | BIGINT | FK → products |
| `addon_product_id` | BIGINT | FK → products |
| `discount_type` | VARCHAR(32) | "percentage" or "fixed" |
| `discount_value` | BIGINT | Percentage 0-100 or cents |
| `is_stackable` | BOOLEAN | DEFAULT FALSE |
| `extends_duration_by_base` | BOOLEAN | DEFAULT FALSE |
| `is_active` | BOOLEAN | DEFAULT TRUE |
| `created_us` | BIGINT | |
| `updated_us` | BIGINT | |

**Unique constraint**: `(base_product_id, addon_product_id)`

#### `booking_groups`
Links bookings that are part of the same logical unit (bundle).

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `group_type` | VARCHAR(32) | "bundle" |
| `purchase_id` | BIGINT | FK → purchases |
| `created_us` | BIGINT | |

#### `booking_group_members`
| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `booking_group_id` | BIGINT | FK → booking_groups |
| `booking_id` | BIGINT | FK → bookings |
| `role` | VARCHAR(32) | "base" or "addon" |
| `created_us` | BIGINT | |

### Modified Tables

#### `products`
| New Column | Type | Notes |
|------------|------|-------|
| `checkin_window_minutes` | BIGINT | Nullable. Minutes before booked time check-in is allowed. |
| `requires_room_schedule` | BOOLEAN | DEFAULT FALSE. If true, availability is driven by room_schedules instead of provider_availability. |

#### `people`
| New Column | Type | Notes |
|------------|------|-------|
| `must_change_password` | BOOLEAN | DEFAULT FALSE. If true, force password change on next login. |

---

## Implementation Phases

### Phase 0: Quick Fixes and Foundation
*Small, high-impact cleanups*

#### 0.1 Rename "Membership Sharing" to "Purchase Sharing"
- [x] Update `gift-permissions.component.html` — title and description text
- [x] Update account dashboard card in `user.component.html` — title and subtitle
- [x] Tests: updated `user.component.spec.ts` assertion for card text

#### 0.2 Fix PayCardResponse TypeScript Type
- [x] `payment.types.ts`: Changed `PayCardResponse.entitlements_created: number` to `entitlements: Entitlement[]`
- [x] Same for `PayVoucherResponse`
- [x] `ServerAccessNetwork.ts`: Removed lossy mapping that was converting entitlements array to count
- [x] `ServerAccessMock`: Updated `purchasePayCard()` and `purchasePayVoucher()` to return entitlement arrays
- [x] Fixed all tests: mock spec, checkout spec, event-booking spec, service-booking spec (14 occurrences)

#### 0.3 Add `must_change_password` to People
- [x] Add column to `db_schema/people.h/cpp` (BOOLEAN, DEFAULT FALSE)
- [x] Register in admin column data info and column friendly names in `create_database.cpp`
- [x] Include `must_change_password` in `get_user_info` endpoint response
- [x] Clear `must_change_password` flag in `update_user_password` endpoint after successful password change
- [x] Add `must_change_password` to frontend `UserInfo` type and `AuthData` type
- [x] Propagate flag in `AuthService.udpateAuthData()`
- [x] Login component: redirect to change password page if `mustChangePassword` is true
- [x] AuthGuard: redirect to change password page if `mustChangePassword` is true (except when already on that page)
- [x] Tests: `get_user_info_test.cpp` — default false + flag set returns true; `update_user_password_test.cpp` — flag cleared after password change

---

### Phase 1: Room Schedules and Spa Infrastructure
*The foundation for capacity-tracked room booking*

#### 1.1 `room_schedules` Table
- [x] Create `db_schema/room_schedules.h/cpp` — id, location_room_id (FK), day_of_week (INT), open_time_minutes (INT), close_time_minutes (INT), is_active, created_us, updated_us
- [x] Register in `make_database_info.cpp`, `create_database.cpp` (all admin sections)
- [x] Add to db_schema and table_helpers CMakeLists
- [x] Create table helper `room_schedules.h/cpp`: AddRoomSchedule, GetSchedulesForRoom, GetScheduleForRoomAndDay, GetScheduleById, DeleteSchedule
- [x] Tests: 5 tests in `room_schedules_test.cpp`

#### 1.2 Add `checkin_window_minutes` and `requires_room_schedule` to Products
- [x] Add `checkin_window_minutes` (nullable BIGINT) and `requires_room_schedule` (BOOLEAN, DEFAULT FALSE) to `products.h/cpp`
- [x] Register in admin column data info and column friendly names

#### 1.3 Room-Based Availability Computation
- [ ] Create `RoomAvailabilityHelper` (or extend service availability)
- [ ] For products where `requires_room_schedule = true`:
  - Use `room_schedules` to determine open hours (not provider_availability)
  - Compute available slots at 5-min intervals during open hours
  - For each slot, count overlapping active bookings in the room
  - Exclude slots where booking count would reach `concurrent_capacity` for any moment during the booking duration
- [ ] Endpoint: extend `GET /api/available_service_slots` to handle room-schedule-based products
- [ ] Tests: extensive availability tests — open hours, capacity limits, overlapping bookings

#### 1.4 Spa Product and Room Setup
- [ ] Add "Spa" room type in database seed data
- [ ] Add "Spa" room to facility with `concurrent_capacity` (e.g., 15)
- [ ] Add room_schedules entries for spa operating hours
- [ ] Create "Spa Entry" product (`kind = "bookable_service"`, `requires_room_schedule = true`) with variants:
  - Early Bird 60 Min ($25)
  - Off-Peak 60 Min ($30)
  - Peak 60 Min ($40)
  - 30 Min ($20)
  - 90 Min ($50)
- [ ] Create "Late Night Recovery Spa" as separate product with `visibility_permission_id` set to platinum member permission
  - 30 Min ($15)
- [ ] Set `checkin_window_minutes = 15` on both spa products
- [ ] Tests: products created correctly

#### 1.5 Staff Check-In Endpoint
- [ ] Create `POST /api/staff/checkin/{bookingId}` endpoint
- [ ] Validates: booking exists, booking is confirmed, checked_in_us not already set
- [ ] Check-in window: if `checkin_window_minutes` is set, allow check-in that many minutes before start time
- [ ] **Capacity gate for early check-in**: if checking in before booked start time, query current occupancy of the room. If at capacity, return error "Room is at capacity — please wait"
- [ ] Sets `bookings.checked_in_us = now_us()`
- [ ] Register in `web_app.cpp` and `CMakeLists.txt`
- [ ] Tests: success, already checked in, too early, capacity blocked, late check-in (allowed, no extension)

#### 1.6 Staff Check-In UI
- [ ] Create check-in page in staff portal at `/staff/checkin`
- [ ] Show upcoming bookings for the next configurable window (default 90 min, configurable via input)
- [ ] Person autocomplete search to find a specific person's bookings
- [ ] Each booking row: person name, product, variant, time, status, "Check In" button
- [ ] Current occupancy indicator per room: "Spa: 8/15" with color coding (green < 70%, amber < 90%, red >= 90%)
- [ ] Route and staff dashboard card
- [ ] Tests: component spec

#### 1.7 Occupancy Dashboard
- [ ] Create `GET /api/room_occupancy/{roomId}` endpoint — returns current count, capacity, and upcoming 2-hour window with projected counts
- [ ] Staff dashboard widget showing all rooms with current occupancy
- [ ] Public-facing endpoint (no auth required): `GET /api/public/room_occupancy/{roomId}` — returns just current count and capacity
- [ ] Frontend: occupancy indicator on the spa service booking page so users can see how busy it is before booking
- [ ] Tests: endpoint tests, component spec

---

### Phase 2: Shopping Cart
*Multi-item purchase capability — prerequisite for add-on bundles*

#### 2.1 Cart Service (Frontend)
- [ ] Create `CartService` — in-memory cart state management (array of cart items)
- [ ] Each cart item: `{ productId, variantId?, quantity, scheduledStartTimeUs?, providerPersonId?, facilityId?, forPersonId? }`
- [ ] Methods: `addItem()`, `removeItem()`, `clearCart()`, `getItems()`, `getItemCount()`
- [ ] Observable `cartItems$` for reactive UI updates
- [ ] Persist cart in localStorage for page refresh survival
- [ ] Tests: service spec

#### 2.2 Cart UI Component
- [ ] Create `CartComponent` (standalone) — slide-out panel or dropdown from header
- [ ] Shows cart items with product name, variant, price, scheduled time (if applicable)
- [ ] Remove item button per row
- [ ] Subtotal display
- [ ] "Checkout" button → navigates to cart checkout page
- [ ] Cart item count badge in header nav
- [ ] Tests: component spec

#### 2.3 Cart Checkout Page
- [ ] Create `CartCheckoutComponent` at `/shop/cart`
- [ ] Shows all items with line totals
- [ ] Detects add-on relationships and shows bundle discount applied
- [ ] Coupon code input (applied to the whole purchase)
- [ ] Voucher code input
- [ ] Payment method
- [ ] Creates a single multi-item purchase via `createPurchase()`
- [ ] Tests: component spec

#### 2.4 Add to Cart Buttons
- [ ] Update service booking flow: after selecting a slot, "Add to Cart" button (in addition to "Book and Pay Now")
- [ ] Update catalog/product browse: "Add to Cart" on products
- [ ] Update event booking: "Add to Cart" option
- [ ] If user adds a product that has add-ons, show a suggestion: "Save X% — add Spa Entry to your cart"
- [ ] Tests: integration tests

---

### Phase 3: Add-On Bundles
*Product relationships with automatic discounts*

#### 3.1 `product_addons` Table
- [ ] Create `db_schema/product_addons.h/cpp`
- [ ] Register in `make_database_info.cpp`, `create_database.cpp` (all admin metadata, nested under products)
- [ ] Add to `CMakeLists.txt`
- [ ] Create table helper: `AddProductAddon`, `GetAddonsForProduct`, `GetBaseProductsForAddon`
- [ ] Tests: table helper CRUD

#### 3.2 `booking_groups` and `booking_group_members` Tables
- [ ] Create `db_schema/booking_groups.h/cpp` and `booking_group_members.h/cpp`
- [ ] Register in `make_database_info.cpp`, `create_database.cpp`
- [ ] Create table helpers
- [ ] Tests: table helper CRUD

#### 3.3 Add-On Discount Logic
- [ ] Create `business_logic/payment/addon_helper.h/cpp`
- [ ] `GetAvailableAddons(productId)` — returns addon products with discount info
- [ ] `CalculateAddonDiscount(baseProductId, addonProductId, addonPriceCents)` — computes discounted price
- [ ] Discount vs coupon logic: if `is_stackable = false`, compare add-on discount to coupon discount. Apply the better one. The unused discount is not consumed (coupon use count not incremented if add-on discount was better).
- [ ] Tests: percentage discount, fixed discount, stackability, coupon interaction

#### 3.4 Purchase Creation with Add-On Discounts
- [ ] Extend purchase creation: when a purchase contains items that have add-on relationships, automatically apply the add-on discount
- [ ] The detection is based on `product_addons` table: if item A is a base and item B is its addon, apply discount to B
- [ ] Track which discount was applied per item (add-on vs coupon)
- [ ] Tests: purchase with add-on priced correctly, coupon vs add-on interaction

#### 3.5 Duration Extension for Bundled Bookings
- [ ] When booking base + add-on together and `extends_duration_by_base = true`:
  - Adjust add-on booking end time: `addon_end = addon_start + addon_variant_duration + base_variant_duration`
  - Factor extended duration into room capacity check
- [ ] Create booking_group linking both bookings
- [ ] Tests: extended duration calculated correctly, capacity check uses extended time

#### 3.6 Linked Cancellation
- [ ] Update `BookingHelper::CancelBooking` to check booking groups
- [ ] When user cancels ANY member of a bundle group: cancel ALL members, refund per individual product policy
- [ ] When **provider** cancels the **base** product:
  - Cancel the base booking with full refund
  - Do NOT cancel the add-on — keep it booked at the bundle discount price
  - Give the add-on a `free_cancel_until_us` override (e.g., 24 hours from now) so the customer can cancel with full refund if they want
  - Send email: "Your massage was cancelled by the provider. Your spa visit is still booked. You may cancel with a full refund if you prefer."
- [ ] Tests: user cancel → all cancelled, provider cancel base → base cancelled + addon kept + free cancel window

#### 3.7 Admin Add-On Configuration
- [ ] Use existing nested admin table CRUD (`product_addons` registered as nested under products)
- [ ] Verify admin can create/view/edit add-on relationships through the dashboard
- [ ] Tests: can CRUD add-on relationships

#### 3.8 Add-On Suggestions in Shopping Cart
- [ ] When an item is added to cart, check if it has add-ons
- [ ] Show suggestion card: "Add Spa Entry and save 20%! [Add to Cart]"
- [ ] When add-on is in cart alongside its base, show the discounted price on the add-on line item with strikethrough on original price
- [ ] Tests: component spec

---

### Phase 4: Book for Someone Else
*Extends purchase sharing to bookable services with invite flow*

#### 4.1 Service Booking for Another Person
- [ ] Extend service booking UI: "Who is this booking for?" selector
  - Default: "Myself"
  - Option: search from existing purchase sharing contacts (gift permission grantees)
  - Option: enter email address of someone else
- [ ] If person has purchase sharing enabled → book directly for them
- [ ] The booking shows up in the recipient's "My Bookings"
- [ ] The purchase is under the payer's account
- [ ] Tests: book for self, book for sharing contact, purchase attributed to payer

#### 4.2 Invite Non-Member via Email
- [ ] If user enters an email address of someone who isn't a member:
  - System sends an invitation email: "X has booked a Massage for you at Knotty Yoga! Create your account to view your booking."
  - The invitation includes a link to register
  - After registration, the person is prompted to accept purchase sharing from the gifter
  - Once accepted, the booking becomes visible in their "My Bookings"
- [ ] If the email belongs to an existing member but purchase sharing isn't set up:
  - Send a purchase sharing request
  - Once accepted, the booking becomes visible
- [ ] Tests: invite flow for non-member, request flow for existing member

#### 4.3 Same Patterns for Event Booking
- [ ] Extend event booking with the same "Who is this for?" flow
- [ ] Tests: event booking for another person

---

### Phase 5: Drop-In Bookings
*Staff-initiated on-the-fly bookings for walk-in customers*

#### 5.1 Staff Drop-In Endpoint
- [ ] Create `POST /api/staff/dropin_booking` endpoint
- [ ] Accepts: productId, variantId, personId, facilityId, roomId
- [ ] Creates booking starting now (rounded to next 5-min boundary)
- [ ] Checks room capacity before creating
- [ ] Creates purchase at standard product price
- [ ] Tests: success, capacity exceeded

#### 5.2 On-the-Fly Account Creation
- [ ] Create `POST /api/staff/create_quick_account` endpoint
- [ ] Accepts: firstName, lastName, email
- [ ] Creates account with auto-generated password
- [ ] Sets `must_change_password = true`
- [ ] Sends welcome email with temporary password and "please change immediately" message
- [ ] Returns the new person_id for immediate use in drop-in booking
- [ ] Tests: success, duplicate email, email sent

#### 5.3 Staff Drop-In UI
- [ ] Add to staff check-in page: "Walk-In Booking" section
- [ ] Search for existing person or "New Customer" button
- [ ] New Customer: collect first name, last name, email → create account → proceed to booking
- [ ] Select product/variant, show current room capacity
- [ ] Create booking button
- [ ] Tests: component spec

---

### Phase 6: iCal and Email Enhancements

#### 6.1 iCal Generation Utility
- [ ] Create `util/ical/ical_generator.h/cpp`
- [ ] Generate `.ics` file content from: title, start time, end time, location, description
- [ ] Timezone-aware using facility timezone
- [ ] Tests: generated ICS is valid format

#### 6.2 Attach iCal to Booking Confirmation Emails
- [ ] Update `ServiceBookingConfirmationMail` to attach .ics
- [ ] Update `WaitlistConfirmationBody` / event booking email to attach .ics
- [ ] Update any spa booking confirmation to attach .ics
- [ ] The mail helper needs to support attachments — check if mailio supports this, extend if needed
- [ ] Tests: email body includes attachment

#### 6.3 Confirmation Email for Bundles
- [ ] When a bundle is booked (base + add-on), send a single confirmation showing both components
- [ ] Include note about linked cancellation policy
- [ ] Include iCal with multiple events (or two separate .ics files)
- [ ] Tests: email content

---

### Phase 7: Polish

#### 7.1 Purchase Detail Page
- [ ] Create `GET /api/purchases/{id}` endpoint with entitlements + assignments
- [ ] Create `PurchaseDetailComponent` at `/my/purchases/:id`
- [ ] Integrate `SeatAssignmentComponent` for multi-seat entitlements
- [ ] Link from purchase history and checkout success

#### 7.2 Checkout Success Enhancement
- [ ] Show entitlements and seat assignment after payment
- [ ] Link to purchase detail page

#### 7.3 My Bookings Enhancement for Bundles
- [ ] Show linked bookings grouped together
- [ ] "Bundled with: Spa Entry" label
- [ ] Cancellation warning: "Canceling will also cancel your Spa Entry booking"

#### 7.4 Catalog Multi-Seat Badge
- [ ] Add `seats_default` to catalog response
- [ ] Show "For N people" badge on multi-seat products
- [ ] Show seat info in checkout

#### 7.5 Update Design Documents
- [ ] Update Payment Design Document with new scenarios and tables
- [ ] Update Support for scheduled purchases with new scenarios
- [ ] Mark completed checkboxes

---

## Documents to Update

### Payment Design Document
- Add scenario: User purchases a bundled product (add-on with discount)
- Add scenario: User books a service for someone else (book-for-others)
- Add scenario: Staff creates drop-in booking for walk-in customer
- Add scenario: Staff checks in attendee with capacity enforcement
- Add scenario: User views real-time room occupancy
- Update data model with: `room_schedules`, `product_addons`, `booking_groups`, `booking_group_members`
- Update `products` table with `checkin_window_minutes`, `requires_room_schedule`
- Update `people` table with `must_change_password`

### Support for scheduled purchases
- Update scenario 37 (Staff checks in attendee) — will be fully implemented
- Add scenario: Room capacity enforcement during booking
- Add scenario: Drop-in booking with account creation
- Add scenario: Bundle booking with linked cancellation
- Add scenario: Shopping cart multi-item purchase

---

## Resolved Questions

### RQ1: Room-schedule-based product UI

**Decision**: Option (a) — flat time slot grid, no provider grouping. This applies to spa, swim spa, rock climbing wall, and any future room-based bookable product. The room-schedule concept is general purpose — it's for anything that is bookable by time slot with capacity, without a specific provider. The room assignment is automatic.

**Additional scope note**: The swim spa (bookable in 30-min increments) and rock climbing wall (capacity-limited, no attendant) are additional products that will use this same `requires_room_schedule` infrastructure. All three (spa, swim spa, climbing wall) will need rooms with capacity and operating hour schedules.

### RQ2: Add-on relationships are bidirectional

**Decision**: Yes. Create entries in both directions if desired (Massage→Spa and Spa→Massage). With the cart-based approach, directionality matters less — the cart detects any add-on relationship between items regardless of which was added first.

### RQ3: Room capacity check covers full booking duration

**Decision**: Check every 5-minute interval of the full booking duration (including duration extension from bundles). If any interval would exceed room capacity, the slot is unavailable. This is required for fire code compliance.

### RQ4: "Add to Cart" on all purchasing flows except subscriptions

**Decision**: All products show both "Book and Pay Now" (existing express checkout) and "Add to Cart" — except subscriptions, which require a card on file and monthly billing, so they stay as their own dedicated flow.

### RQ5: Cart persists with configurable TTL

**Decision**: Cart persists in localStorage. TTL is controlled by a configuration secret. Scheduled items are re-validated on cart load — if the slot was taken, warn the user and remove the item.

### RQ6: Unclaimed bookings remain valid

**Decision**: Bookings exist and are valid regardless of whether the recipient has created an account. The payer sees it in their purchases. When the recipient creates an account with the matching email, the booking automatically appears in their "My Bookings." Normal cancellation applies if the payer cancels before signup.

### RQ7: Occupancy = checked-in with end time not passed

**Decision**: Option (c). A person counts as "occupying" a room from check-in until their booked end time. Not checked in yet = not occupying. End time passed = not occupying. This gives the most accurate real-time picture for staff and public dashboards.

### RQ8: Product catalog finalized during implementation

**Decision**: Exact variants and pricing will be finalized during Phase 1.4 seed data creation. Placeholder prices in the plan are approximate.
