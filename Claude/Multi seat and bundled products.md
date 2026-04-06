---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/3/2026
Version: 0.1
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

# Plan: Couple's Massage, Spa Capacity, Add-On Bundles

## Critique and Analysis of Requirements

Before diving into the plan, here are my observations, concerns, and suggestions about the requirements as written.

### What I Like About This Design
- **Linking add-on scheduling to the base product** (spa visit extended by massage duration) is clever and creates real customer value
- **Drop-in support** fills an important gap — not all visits are pre-planned
- **Early check-in with capacity awareness** prevents overcrowding while being flexible
- **Room-level capacity** already exists in the codebase (`location_rooms.concurrent_capacity`), so spa capacity tracking has a foundation

### Concerns and Pushback

**1. Couple's massage as multi-provider slots is a major complexity increase**

The current service booking model is fundamentally 1-provider-1-client-1-room. A couple's massage requires: finding 2 providers available simultaneously, booking 2 rooms (or 1 double room), creating 2 linked service sessions, handling partial cancellation, and a completely new slot computation algorithm. This is not an extension of the existing system — it's essentially a parallel booking engine.

**Alternative suggestion**: For v1, model couple's massage as a single booking with one "lead" provider. The admin schedules two therapists for the slot manually (or it's inherent to the room type). The system books the time slot and the room, and the provider assignment is handled operationally. This is how most spa booking systems work — the couple books a "couple's massage" slot, and the front desk assigns therapists.

Mason- If it needs to be handled operationally, how can the person book it? I suppose couple's massage isn't that important to me to be honest. I'm not putting in a room specifically for it so it really would just be two people who would be getting massages with different providers in different rooms at the same time. I'm not sure that it is worth the complexity to the system to change the model to support this when someone can pretty easily just book two massages at the same time slot (or close enough). I wanted to have a multi-seat product though. I think that this exposes a couple of things. We should allow someone to book a massage for someone else. We are already renaming Membership sharing to product sharing. I think we need to modify massage booking to allow someone to book a massage for someone else for whom they have enabled product sharing. It would also be nice to also change the membership and massage booking so that they can book this for someone else and either auto complete someone for whom they have already configured product sharing or allow them to type the email address of someone else. If this person is already a member but not configured for product sharing for this person, it would send them a request to allow product sharing and then share this product once enabled. If they aren't a member, it would send email to create an account with this email and then, after creating an account, this would prompt them to accept product sharing from this person and then after agreeing to that, share the product or membership gifted. This sounds like a lot of work but I want to enable things like gift cards, guest passes, and gifted massages as a vehicle for getting new customers.

**2. The add-on model as described creates complex scheduling dependencies**

The requirement says: "the length of the spa visit is automatically extended by the length of the massage" and canceling either component cancels everything. This creates tightly coupled scheduling where a 60-min massage + spa means the spa booking must account for the massage duration, and capacity calculations depend on services in other rooms.

**Alternative suggestion**: Keep add-on pricing simple (discounted spa when purchased with massage) but schedule them independently. The user books massage at 2pm and spa at 3pm (or whenever). The system discounts the spa price. They're linked for cancellation purposes but not for capacity calculation. This dramatically simplifies the implementation while delivering the same customer value.

Mason- my issue is that if I make spa entry three hours during off peak and then two hours during peak. It is best if someone uses the spa for a bit to have the steam room and sauna loosen them up a bit pre massage and then again post massage to make the massage easier for the provider and then to warm up again post massage (which pushes blood to the surface and makes them cold). This means they won't book the massage at the before or after the spa entry. That means that a 90min massage would consume a LOT of the spa time. Hence wanting to extend the length of the spa visit by the length of the massage.

**3. Check-in as a capacity gate adds real-time operational complexity**

"Checking in early would exceed spa capacity, they must wait" means the check-in UI needs real-time capacity tracking and the ability to queue people. This is essentially a real-time occupancy management system.

**Simpler alternative**: Check-in is just a timestamp for record-keeping. If someone arrives early, the staff makes a judgment call. Enforce capacity only at booking time, not at check-in time. Add a "current occupancy" dashboard for the spa staff but don't gate check-in on it.

Mason- the issue is that there are legal limits on capacity for fire code safety.

**4. Drop-in with account creation is a significant scope increase**

Creating accounts on the fly during a drop-in (collect name/email, auto-generate password, email them) is an entirely separate workflow that touches auth, registration, and email. It's useful but orthogonal to the booking features.

**Suggestion**: Phase drop-ins separately. For v1, staff can only do drop-in bookings for existing accounts. Account creation during drop-in comes in a later phase.

Mason- I want to make collecting new customers to be something easy. We already have account creation support. This would be auto generating a password and emailing it to them. Since we don't support booking as a guest, since we want to funnel people through the account creation process, we should make creating an account for a new person as easy as possible. 

### Suggested Additional Work Items

1. **Staff check-in endpoint** — `POST /api/staff/checkin/{bookingId}` — the `checked_in_us` column exists but there's no endpoint to set it
2. **Spa occupancy dashboard** — real-time view of current and upcoming capacity for staff
3. **Product add-on admin UI** — admin interface for configuring which products can be add-ons to which
4. **iCal attachment for booking emails** — mentioned in Support for scheduled purchases as a future item, relevant here
5. **Rename "Membership Sharing" to "Purchase Sharing"** — simple UI text change in gift-permissions component

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
| Gift permissions / seat assignment | Exists | Couple's booking recipient selection |
| Product booking windows (permission-based advance booking) | Exists | Early bird variants |
| `SeatAssignmentComponent` (standalone) | Exists | Assigning second person |
| Coupon system with multi-product restriction | Exists | Add-on discounts |

### What's Missing

| Gap | Impact |
|-----|--------|
| No concept of linked/grouped bookings | Couple's massage, bundle cancellation |
| No room capacity enforcement during booking | Spa capacity |
| No check-in endpoint | Staff check-in |
| No drop-in booking flow | Walk-in customers |
| No add-on/bundle product relationship table | Bundle pricing |
| No multi-provider simultaneous slot computation | Couple's massage availability |
| No account-on-the-fly creation | Drop-in for new customers |
| Service availability doesn't check room capacity | Spa overbooking prevention |

---

## Data Model Changes

### New Tables

#### `product_addons`
Defines which products can be add-ons to which base products, and the discount.

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `base_product_id` | BIGINT | FK → products (the main product, e.g., massage) |
| `addon_product_id` | BIGINT | FK → products (the add-on, e.g., spa entry) |
| `discount_type` | VARCHAR(32) | "percentage" or "fixed" |
| `discount_value` | BIGINT | Percentage 0-100 or cents |
| `is_stackable` | BOOLEAN | DEFAULT FALSE — can this discount combine with other discounts? |
| `extends_duration_by_base` | BOOLEAN | DEFAULT FALSE — does base product duration extend the add-on? |
| `is_active` | BOOLEAN | DEFAULT TRUE |
| `created_us` | BIGINT | |
| `updated_us` | BIGINT | |

**Unique constraint**: `(base_product_id, addon_product_id)` — each pairing defined once.

#### `booking_groups`
Links bookings that are part of the same logical unit (couple's massage, base + add-on bundle).

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `group_type` | VARCHAR(32) | "couple" or "bundle" |
| `purchase_id` | BIGINT | FK → purchases |
| `cancel_policy` | VARCHAR(32) | "cancel_all" (default) — canceling any member cancels all |
| `created_us` | BIGINT | |

#### `booking_group_members`
Junction table linking individual bookings to a group.

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `booking_group_id` | BIGINT | FK → booking_groups |
| `booking_id` | BIGINT | FK → bookings |
| `role` | VARCHAR(32) | "base" or "addon" (for bundles), "primary" or "secondary" (for couples) |
| `created_us` | BIGINT | |

### Modified Tables

#### `products` — add columns
| Column | Type | Notes |
|--------|------|-------|
| `checkin_window_minutes` | BIGINT | Nullable, default NULL. How many minutes before booked time can check-in happen. NULL = no early check-in. |

#### `location_rooms` — already has `concurrent_capacity`
No schema change needed. Existing `concurrent_capacity` field will be used for spa capacity tracking.

---

## Design Decisions

### DD1: Couple's Massage — Booking Model

**Decision**: Model as **two separate linked bookings** in a booking group, not as a single multi-seat booking.

**Rationale**: 
- Each person gets their own booking (visible in their "My Bookings")
- Each person can be assigned a different provider
- Cancellation of either cancels both (via booking_group)
- The payer pays once (single purchase with two purchase items)
- Seat assignment uses the existing gift permission / seat assignment flow

**Flow**:
1. User selects "Couple's Massage" product (which has `seats_default = 2`)
2. UI shows available time slots where 2+ providers have overlapping availability and 2+ rooms of the required type are free
3. User selects a time slot
4. User assigns person 2 (self, or via gift permission search)
5. For each person, user selects a preferred provider from available ones
6. System creates: 1 purchase (2 items), 2 service sessions, 2 bookings, 1 booking group
7. Both people see the booking in their "My Bookings"

### DD2: Spa Capacity Enforcement

**Decision**: Enforce capacity at **booking time** using the room's `concurrent_capacity`.

**How**: When computing available spa slots, query all bookings overlapping each potential slot's time window. If the count of overlapping active bookings equals `concurrent_capacity`, the slot is unavailable. This check happens at 5-minute intervals (existing slot alignment).

**Check-in**: Check-in at the staff portal is a simple timestamp update. If the user arrives early, staff decides — no system gate on capacity at check-in time. This keeps the operational flow simple. A "Current Occupancy" indicator on the check-in screen shows staff the current count vs capacity as a visual aid.

### DD3: Add-On Bundle — Purchase and Scheduling

**Decision**: Add-ons are **separate products linked by the `product_addons` table** with an automatic discount applied at purchase creation time.

**Booking flow**:
1. User books the base product (e.g., massage) normally
2. After selecting a time slot, a "Add-Ons" section appears showing available add-ons
3. User clicks an add-on (e.g., "Spa Entry")
4. System shows available spa time slots that are compatible:
   - For `extends_duration_by_base = true`: spa slot starts at or before massage start, spa duration = spa variant duration + massage duration
   - Available slots checked against spa room capacity
5. User selects a spa time slot
6. System creates a single purchase with 2 items: base product (full price) + add-on (discounted price)
7. Both bookings are linked via a booking group
8. Canceling either cancels both; if provider cancels base, full refund for everything

### DD4: Check-In Window

**Decision**: Use a `checkin_window_minutes` column on `products` (not on variants, since check-in policy is per product type).

Default behavior: if `checkin_window_minutes` is NULL, no early check-in allowed (or staff discretion). If set to 15, the staff check-in endpoint allows check-in up to 15 minutes before the booked start time.

### DD5: Drop-In Bookings

**Decision**: Phase separately. For v1, drop-in only works for existing accounts. Staff searches for the person and creates a booking on-the-fly if there's capacity. Account-on-the-fly creation comes in a later phase.

### DD6: "Late Night Post-Workout Spa" and "Early Bird"

**Decision**: These are just **product variants** with specific pricing and availability windows.

- "Early Bird Spa (Mon-Wed 6am-9am)" — a variant of the Spa Entry product with lower price and restricted availability (the availability blocks for this variant only exist during early morning hours)
- "Late Night Post-Workout Spa (Thu 9pm-10pm)" — another variant with specific availability

Provider availability determines when these are bookable. The admin creates availability blocks for the spa "provider" (or treats the spa as a self-service room) during these specific windows.

**Challenge**: The current provider availability model is person-based. A spa is a room, not a person. 

**Solution**: Create a virtual "Spa" provider (a staff account) that represents the spa. The spa's availability determines when spa bookings can be made. The room capacity limits concurrent bookings. This reuses the existing provider availability infrastructure without modification.

---

## Documents to Update

### Payment Design Document
- Add scenario for couple's service purchase (multi-provider, linked booking)
- Add scenario for add-on bundle purchase
- Add scenario for drop-in booking
- Add scenario for check-in workflow
- Update data model section with new tables: `product_addons`, `booking_groups`, `booking_group_members`
- Update `products` table with `checkin_window_minutes`

### Support for scheduled purchases
- Add new scenarios:
  - Staff checks in attendee (scenario 37 — will be partially implemented)
  - Couple books a service together
  - User purchases base product with add-on
  - User does a drop-in booking
  - Spa capacity management
- Update scenario 23 (reminder window for services — may be partially addressed)
- Move relevant items from COULD HAVE / STRETCH as they get implemented

---

## Implementation Phases

### Phase 0: Rename and Quick Fixes
*Prerequisite cleanup*

#### 0.1 Rename "Membership Sharing" to "Purchase Sharing"
- [ ] Update `gift-permissions.component.html` — change title text and card descriptions
- [ ] Update the account dashboard card in `user.component.html`
- [ ] Tests: component spec

#### 0.2 Fix PayCardResponse TypeScript Type
- [ ] `payment.types.ts`: Change `PayCardResponse.entitlements_created: number` to `entitlements: Entitlement[]`
- [ ] Same for `PayVoucherResponse`
- [ ] Update mock, fix any broken tests

---

### Phase 1: Spa Product and Capacity Infrastructure
*Database, business logic, then UI — bottom up*

#### 1.1 Create Spa Product, Room, and Variants
- [ ] Add "Spa" room type to database seed data
- [ ] Add "Spa" room to existing facility with `concurrent_capacity` (e.g., 15)
- [ ] Create "Spa Entry" product (`kind = "bookable_service"`) with variants:
  - 30 Min Spa ($20)
  - 60 Min Spa ($35)
  - 90 Min Spa ($50)
  - Early Bird 60 Min Spa ($25) — lower price variant
  - Late Night Recovery 30 Min ($15) — lower price variant
- [ ] Create virtual "Spa Attendant" provider with availability blocks for spa hours
- [ ] Tests: verify products and rooms are created

#### 1.2 Room Capacity Check in Service Availability
- [ ] Update `ServiceAvailabilityHelper::ComputeAvailableSlots()` to check room capacity
- [ ] For each potential slot, count overlapping active bookings in the same room
- [ ] If count >= `concurrent_capacity`, exclude the slot
- [ ] The existing `FindAvailableRoom()` already checks room availability for 1:1 sessions — extend it to support concurrent rooms (capacity > 1)
- [ ] Tests: `service_availability_helper_test.cpp` — slots excluded when room at capacity

#### 1.3 Add `checkin_window_minutes` to Products
- [ ] Add column to `db_schema/products.h/cpp` (nullable BIGINT)
- [ ] Register in admin column metadata
- [ ] Add to `ProductInfo` struct if needed
- [ ] Tests: schema creation

#### 1.4 Staff Check-In Endpoint
- [ ] Create `POST /api/staff/checkin/{bookingId}` endpoint
- [ ] Validates: booking exists, booking is confirmed, checked_in_us is not already set
- [ ] If product has `checkin_window_minutes`: validates current time is within window before booking start time
- [ ] Sets `bookings.checked_in_us = now_us()`
- [ ] Returns updated booking
- [ ] Register in `web_app.cpp` and `CMakeLists.txt`
- [ ] Tests: success, already checked in, too early, not found

#### 1.5 Staff Check-In UI
- [ ] Create check-in page in staff portal at `/staff/checkin`
- [ ] Search for bookings by person name/email or upcoming bookings for today
- [ ] Show booking details with "Check In" button
- [ ] Show current spa occupancy indicator (count of checked-in-but-not-ended bookings vs capacity)
- [ ] Route and dashboard card
- [ ] Tests: component spec

---

### Phase 2: Add-On Products and Bundle Pricing
*The add-on relationship and discounted bundle purchases*

#### 2.1 `product_addons` Table
- [ ] Create `db_schema/product_addons.h/cpp`
- [ ] Register in `make_database_info.cpp`, `create_database.cpp` (all admin metadata)
- [ ] Add to `CMakeLists.txt`
- [ ] Create table helper: `AddProductAddon`, `GetAddonsForProduct`, `GetAddonRelationship`
- [ ] Tests: table helper CRUD

#### 2.2 `booking_groups` and `booking_group_members` Tables
- [ ] Create `db_schema/booking_groups.h/cpp` and `booking_group_members.h/cpp`
- [ ] Register in `make_database_info.cpp`, `create_database.cpp`
- [ ] Add to `CMakeLists.txt`
- [ ] Create table helpers
- [ ] Tests: table helper CRUD

#### 2.3 Add-On Discount Logic
- [ ] Create `business_logic/payment/addon_helper.h/cpp`
- [ ] `GetAvailableAddons(productId)` — returns addon products with discount info
- [ ] `CalculateAddonDiscount(baseProductId, addonProductId, addonPriceCents)` — computes discounted price
- [ ] Stackability check: if `is_stackable = false` and the purchase already has a coupon discount, skip the add-on discount (or apply the better one)
- [ ] Tests: percentage discount, fixed discount, stackability

#### 2.4 Purchase Creation with Add-Ons
- [ ] Extend `CreatePurchaseRequest` with optional `addon_for_purchase_item_id` on each item (links add-on items to their base)
- [ ] In `PurchaseHelper::CreatePurchase`, if an item has `addon_for_purchase_item_id`, look up the add-on relationship and apply discount
- [ ] Modify `purchase_items` with optional `addon_for_purchase_item_id` FK
- [ ] Tests: purchase with add-on has correct discounted price

#### 2.5 Linked Booking Creation for Bundles
- [ ] Extend `ServiceBookingHelper` to support creating a booking group when add-ons are included
- [ ] When booking base + add-on:
  - Create base booking normally
  - Create add-on booking (e.g., spa session) with start time from request
  - If `extends_duration_by_base`: adjust add-on end time = addon_duration + base_duration
  - Create booking_group linking both
- [ ] Tests: bundle booking created correctly, durations correct

#### 2.6 Linked Cancellation
- [ ] Update `BookingHelper::CancelBooking` to check for booking groups
- [ ] If booking is part of a group with `cancel_policy = "cancel_all"`: cancel all members
- [ ] If provider cancels the base product: full refund for all group members (override individual policies)
- [ ] Tests: cancel one → cancels all, provider cancel → full refund

#### 2.7 Add-On UI in Service Booking
- [ ] After user selects a base service slot, show "Add-Ons" section
- [ ] Load available add-ons via `getAvailableAddons(productId)`
- [ ] For each add-on, show: name, original price, discounted price, "Save X%"
- [ ] On selecting an add-on: show available time slots (spa slots with capacity check)
- [ ] For `extends_duration_by_base`: show the total duration and note the extension
- [ ] Warn if selected spa slot doesn't allow full duration
- [ ] Create combined booking (base + add-on) in single payment flow
- [ ] Tests: component spec

#### 2.8 Admin Add-On Configuration UI
- [ ] Create admin page for managing add-on relationships
- [ ] Or: use existing nested admin table CRUD (product_addons shows under products)
- [ ] Tests: can create and view add-on relationships

---

### Phase 3: Couple's Massage
*Multi-provider, multi-person bookable service*

#### 3.1 Couple's Massage Product Setup
- [ ] Create "Couple's Massage" product (`kind = "bookable_service"`, `seats_default = 2`)
- [ ] Create variants: 60 Min ($200), 90 Min ($280), 120 Min ($350)
- [ ] Assign same provider type as regular massage
- [ ] Create a "Couple's Room" room type with capacity 2 (or use two individual rooms)

#### 3.2 Multi-Provider Slot Computation
- [ ] Create `CoupleAvailabilityHelper` (or extend `ServiceAvailabilityHelper`)
- [ ] Compute slots where 2+ providers of the required type are simultaneously available
- [ ] Also require 2 rooms of the required type (or 1 room with capacity >= 2)
- [ ] Return slots with list of available provider pairs
- [ ] Tests: extensive slot computation tests

#### 3.3 Multi-Provider Slot Endpoint
- [ ] Create `GET /api/available_couple_slots` (or extend existing slots endpoint with `?seats=2`)
- [ ] Returns: time slots with available provider pairs
- [ ] Tests: endpoint tests

#### 3.4 Couple's Booking Endpoint
- [ ] Extend `POST /api/book_service` or create `POST /api/book_couple_service`
- [ ] Accepts: productId, variantId, startTimeUs, person1Id, person2Id, provider1Id, provider2Id
- [ ] Creates: 1 purchase (2 items), 2 service sessions, 2 bookings, 1 booking group (type=couple)
- [ ] Auto-assigns each person to their respective booking
- [ ] Payer may be different from either recipient
- [ ] Tests: success, provider conflict, room conflict, auth

#### 3.5 Person Selection in Couple's Booking UI
- [ ] After slot selection, show person assignment step
- [ ] Default: person 1 = payer, person 2 = search (uses gift permission grantees or direct email)
- [ ] Allow both persons to be different from payer (gift scenario)
- [ ] Provider selection per person from available providers at that slot
- [ ] Tests: component spec

#### 3.6 Couple's Cancellation
- [ ] Canceling either booking cancels the group (via Phase 2.6 infrastructure)
- [ ] If either provider cancels: full refund for both
- [ ] Both users see cancellation in their "My Bookings"
- [ ] Tests: cascading cancellation

---

### Phase 4: Drop-In Bookings
*Staff-initiated on-the-fly bookings*

#### 4.1 Staff Drop-In Endpoint
- [ ] Create `POST /api/staff/dropin_booking` endpoint
- [ ] Accepts: productId, variantId, personId (existing account), facilityId
- [ ] Creates booking starting now (or at specified time)
- [ ] Checks room capacity before creating
- [ ] Creates purchase with immediate payment (or deferred payment?)
- [ ] Tests: success, capacity exceeded, invalid product

#### 4.2 Staff Drop-In UI
- [ ] Add to staff check-in page: "Walk-In Booking" section
- [ ] Search for person, select product/variant
- [ ] Show current capacity — if room, show "X/Y occupied"
- [ ] Create booking button
- [ ] Tests: component spec

#### 4.3 Drop-In Account Creation (v2 — later phase)
- [ ] Extend drop-in UI with "New Customer" option
- [ ] Collect: first name, last name, email
- [ ] Create account with auto-generated password
- [ ] Email the password with "please change immediately" message
- [ ] Then create the booking for this new account
- [ ] Tests: account creation + booking in one flow

---

### Phase 5: Polish and Integration
*Tying everything together*

#### 5.1 Purchase Detail Page
- [ ] Create `/my/purchases/:id` with items, payments, entitlements, seat assignment
- [ ] (As described in [[Payment Should Have- Multi Seat and Bundled Pricing]])

#### 5.2 Checkout Success Enhancement
- [ ] Show entitlements and seat assignment after payment
- [ ] For couple's bookings: show both bookings

#### 5.3 Booking Confirmation Emails for Bundles
- [ ] Bundle booking sends confirmation showing both components
- [ ] Include note about linked cancellation policy

#### 5.4 My Bookings Enhancement
- [ ] Show linked bookings grouped together
- [ ] "Part of bundle with: Spa Entry" label
- [ ] Cancellation warning: "Canceling will also cancel: Spa Entry"

#### 5.5 Update Payment Design Document and Support for Scheduled Purchases
- [ ] Add new scenarios to both documents
- [ ] Update data model sections with new tables
- [ ] Update completion checkboxes

---

## Open Questions

### OQ1: Should the virtual "Spa Attendant" provider be a real person account or a system concept?

Spa doesn't have a "provider" in the traditional sense — it's a self-service space. But the availability system is provider-based.

**My suggestion**: Use a real staff account labeled "Spa" that represents the spa facility. Its availability blocks represent spa operating hours. This avoids changes to the availability system.

### OQ2: What happens to the spa booking if the massage runs long?

If massage takes 70 min instead of 60 min, does the spa time shift?

**My suggestion**: No — bookings have fixed times. The extension is calculated at booking time. If the massage runs over, that's an operational issue, not a system issue.

### OQ3: Should "early bird" and "late night" be separate products or variants of the same product?

Separate products are simpler for admin but clutter the catalog. Variants keep it clean but limit pricing flexibility.

**My suggestion**: Variants of the same "Spa Entry" product. The admin configures different availability windows for each variant's "provider" schedule.

### OQ4: How should the add-on discount interact with coupons?

If someone has a 20% coupon AND the spa add-on has a 15% bundle discount, do they stack?

**My suggestion**: Default: non-stackable (`is_stackable = false`). The better discount wins. If admin marks the add-on as stackable, both apply (coupon on the already-discounted price).

### OQ5: Can a spa entry be booked standalone (without massage)?

The requirements describe it as both an add-on AND a standalone product.

**My suggestion**: Yes — the spa entry product exists independently. The add-on relationship just provides a discount when purchased with massage.

### OQ6: For couple's massage, can the two people choose different variants (e.g., person 1 gets 60 min, person 2 gets 90 min)?

This complicates pricing and scheduling significantly.

**My suggestion**: For v1, both get the same variant. The "Couple's Massage" product has its own variants with combined pricing. Different-variant couples massage is a v2 feature.

### OQ7: What's the refund policy when the provider cancels the base of a bundle?

Requirements say full refund. But does that include add-ons that might have a no-refund policy?

**My suggestion**: Yes — if the provider cancels the base, the entire bundle is fully refunded regardless of individual component policies. This is fair because the customer didn't choose to cancel.

### OQ8: Should the duration extension for add-ons be based on the base product's variant duration, or a fixed amount?

If someone books a 90-min massage + 60-min spa, the spa becomes 150 min? That seems like a lot.

**My suggestion**: Use the base variant's `duration_minutes`. For a 90-min massage + 60-min spa, the spa visit is 150 min total. If this feels too long, the admin can create a "Spa Add-On" product with shorter variants (30 min) that only extends by the base duration.

### OQ9: Do we need the "Membership Sharing" rename to "Purchase Sharing" right away, or can it wait?

It's a simple text change but touches the user-facing UI.

**My suggestion**: Do it in Phase 0 as a quick win. It's a 2-file change.

### OQ10: For drop-ins: should the system create a purchase at full price, or should there be a drop-in pricing tier?

Drop-in customers at spas often pay a premium (no reservation discount).

**My suggestion**: For v1, use the product's standard pricing. Drop-in-specific pricing can be added later via a "drop-in" permission tier in the existing pricing system.

---

## Discussion: How to Expose Add-On Selection to Users

This is the trickiest UX question in the entire feature set. Here are several approaches I considered:

### Approach A: Post-Slot Add-On Modal (Your Suggestion)
After the user selects a massage time slot, an "Add-Ons" button appears. Clicking it opens a modal showing available add-ons (e.g., Spa Entry) with discounted pricing. Selecting an add-on shows compatible spa time slots.

**Pros**: Clean separation of base booking from add-on. Familiar e-commerce pattern.
**Cons**: Two-step slot selection (massage time, then spa time) could be confusing. What if no spa slots are available at the chosen massage time?

### Approach B: Combined Availability View
Show a single booking page where the user first picks a massage slot, and the add-on slots are immediately shown below (filtered to compatible times). No modal — it's all one flow.

**Pros**: User sees the full picture at once. No surprise about unavailable spa times.
**Cons**: More complex UI. What if there are 5 add-ons? The page gets long.

### Approach C: Bundle Product with Embedded Scheduling
Create a "Massage + Spa Bundle" product that, when booked, asks for both time slots in sequence. The product IS the bundle — not a base with add-ons.

**Pros**: Simplest for the user — they're booking one thing.
**Cons**: Combinatorial explosion of bundle products (60-min massage + 30-min spa, 60-min massage + 60-min spa, 90-min + 30-min, ...). Admin has to create each combination.

### Approach D: Cart-Based Bundling
The user adds a massage to a shopping cart, then adds a spa entry. The system detects the add-on relationship and applies the discount. Scheduling happens per-item in the cart.

**Pros**: Maximum flexibility. Reuses general multi-item purchase infrastructure.
**Cons**: Requires building a shopping cart system (major feature). The "detect and apply discount" logic is implicit, which may confuse users who don't realize they're getting a discount.

### My Recommendation: Approach A with enhancements from B

Use the post-slot modal (Approach A), but show a preview of add-on availability before the user opens it. After selecting a massage slot, the booking page shows a card like:

```
🧖 Add-Ons Available
  Spa Entry — $35 → $25 (save 29%)
  ✓ Slots available before/after your massage
  [Add to Booking]
```

If no compatible spa slots exist, the card shows "No available spa times for this date" in gray. This gives the user the key information (discount, availability) without forcing a modal interaction.

Clicking "Add to Booking" expands into an inline section (not a modal) showing compatible spa time slots. The user picks one, and both bookings are confirmed together.

---

## Complementary Work Items

These are related features that would pair well with this implementation:

1. **Shopping cart** — Multi-item purchases are already supported on the backend. A cart UI would enable buying massage + spa + other products without the formal add-on relationship.

2. **Package deals / multi-visit passes** — "Buy 5 spa entries, get 1 free" type deals. Uses the existing entitlement system with seats_total = 6, seats_used decremented per visit.

3. **Recurring spa bookings** — "Every Tuesday 6pm spa entry" as a subscription-like recurring booking.

4. **Waitlist for spa** — If spa is at capacity, allow users to join a waitlist for a specific time window.

5. **Real-time occupancy API** — WebSocket or polling endpoint showing current spa occupancy for a dashboard.

6. **Booking modification** — Reschedule a booking without cancel+rebook (scenario 17 from Support for scheduled purchases).

7. **iCal email attachments** — Send `.ics` files with booking confirmations so users can add to their calendar.

8. **Admin dashboard for bundle analytics** — Track how often add-ons are purchased, revenue from bundle discounts vs individual sales.