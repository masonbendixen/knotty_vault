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
- [x] Create `RoomAvailabilityHelper` (`room_availability_helper.h/cpp`) with `ComputeAvailableSlots()` that:
  - Checks `requires_room_schedule = true` on the product
  - Loads room_schedules to determine operating hours per day-of-week
  - Iterates each day in the date range, finds the room's schedule windows
  - Generates slots at 5-min intervals for each variant during open hours
  - Uses `HasCapacityForFullDuration()` — checks every 5-min interval of the booking against room capacity
  - `CountOverlappingBookings()` counts non-cancelled bookable_service_sessions overlapping a time window
  - Returns `AvailableSlot` with `providerPersonId = 0` (room-based, no provider)
- [x] Added `GetDayOfWeek()` and `GetMidnightUs()` to `DateTimeUtil` for timezone-aware day/midnight calculation
- [x] Extended `GET /api/available_service_slots` endpoint: checks `requires_room_schedule` on product, delegates to `RoomAvailabilityHelper` if true, falls back to `ServiceAvailabilityHelper` otherwise
- [x] Tests: 5 tests in `room_availability_helper_test.cpp` — basic slots from schedule, capacity limits slots, no schedule → no slots, full-duration capacity check (60-min blocked by overlapping 30-min booking), cancelled booking doesn't block capacity

#### 1.4 Spa Product and Room Setup
- [x] Added "Spa", "Swim Spa", "Climbing Wall" room types in `PopulateLocationRoomTypes`
- [x] Added "Spa" room to facility with `concurrent_capacity = 15` in `PopulateLocationRooms`
- [x] Added room_schedules entries for spa: all 7 days, 6:00 AM - 10:00 PM
- [x] Created "Spa Entry" product (`bookable_service`, `requires_room_schedule = true`, `checkin_window_minutes = 15`, tiered cancellation policy) with 5 variants: 30 Min ($20), Early Bird 60 Min ($25), Off-Peak 60 Min ($30), Peak 60 Min ($40), 90 Min ($50)
- [x] Created "Late Night Recovery Spa" as separate product with `visibility_permission_id` set to platinum_fitness permission, 30 Min ($15)
- [x] Added `platinum_fitness` permission to seed data
- [x] All variant prices registered in `PopulateProductPrices`

#### 1.5 Staff Check-In Endpoint
- [x] Created `POST /api/staff/checkin/{bookingId}` in `staff_checkin.h/cpp`
- [x] Validates: booking exists, status is "confirmed", not already checked in
- [x] Check-in window: if product has `checkin_window_minutes`, rejects check-in earlier than that window
- [x] Capacity gate: if checking in before booked start time and room has a capacity limit, counts currently occupying people (checked in + end time not passed) and rejects if at capacity
- [x] Sets `bookings.checked_in_us = now_us()` — late check-in allowed, no duration extension
- [x] Registered in `web_app.cpp` and `CMakeLists.txt`
- [x] Tests: 6 tests in `staff_checkin_test.cpp` — success, already checked in, too early (outside window), capacity blocked (room full), not found (404), late check-in (allowed, time not extended)

#### 1.6 Staff Check-In UI
- [x] Created `GET /api/staff/upcoming_checkins` endpoint — returns bookings within configurable window (default 90 min), with person/product/room/facility info, plus room occupancy counts; 2 endpoint tests
- [x] Added `staffGetUpcomingCheckins()` and `staffCheckin()` to ServerAccess interface, proxy, network, mock; added `CheckinBooking`, `RoomOccupancy`, `UpcomingCheckinsResponse` types; 3 mock spec tests
- [x] Created `StaffCheckInComponent` at `/staff/check-in` with:
  - Configurable time window input (default 90 min)
  - Search filter by person name/email/product
  - Booking cards with person name, product, variant, time, room, Check In button
  - "Checked In" badge for already checked-in bookings
  - Room occupancy indicators with color coding (green < 70%, amber 70-90%, red >= 90%)
  - Auto-refresh every 30 seconds
  - Success/error messages for check-in actions
- [x] Route added to `staff.routes.ts`, dashboard card with "how_to_reg" icon added first in staff dashboard
- [x] Tests: 10 component spec tests (create, load, empty state, search filter, checked-in detection, occupancy calculation/class, check-in success/failure, error loading, heading display)

#### 1.7 Occupancy Dashboard
- [x] Created `GET /api/room_occupancy/{roomId}` endpoint (auth required) — returns room name, capacity, current occupancy (checked-in + end time not passed), and 2-hour projected occupancy at 15-min intervals
- [x] Created `GET /api/public/room_occupancy/{roomId}` endpoint (no auth) — returns room name, capacity, current occupancy only
- [x] Added `getRoomOccupancy()` and `getPublicRoomOccupancy()` to ServerAccess interface/proxy/network/mock with types `RoomOccupancyDetail`, `PublicRoomOccupancy`, `OccupancyProjection`
- [x] Staff check-in page already shows per-room occupancy (from Phase 1.6)
- [x] Created `RoomOccupancyBadgeComponent` — standalone reusable badge, fetches public occupancy by room ID, shows "X/Y" with color coding, auto-refreshes every 60s
- [x] Tests: 5 backend endpoint tests, 3 mock spec tests

#### 1.8 Room Schedule Admin UI
- [x] Created 4 admin endpoints in `admin_room_schedules.h/cpp`: `GET /api/admin/rooms_with_schedules` (rooms with capacity > 1), `GET /api/admin/room_schedules/{roomId}`, `POST /api/admin/room_schedules` (with validation), `POST /api/admin/room_schedules/{id}/delete`. All require manage_products permission.
- [x] Added `adminGetRoomsWithSchedules`, `adminGetRoomSchedules`, `adminCreateRoomSchedule`, `adminDeleteRoomSchedule` to ServerAccess interface/proxy/network/mock with `SchedulableRoom` and `RoomScheduleEntry` types
- [x] Created `RoomScheduleEditorComponent` at `/manage/room-schedules` with room selector, 7-day weekly grid, hour:minute pickers (AM/PM, 15-min increments), delete buttons, multiple windows per day support
- [x] Route in manage routes, dashboard card with "schedule" icon
- [x] Tests: 5 backend endpoint tests, 12 component spec tests, 3 mock spec tests

#### 1.9 Room Schedule Template Redesign

The simple `room_schedules` table (day-of-week + open/close) is insufficient. Need a template-based system with date ranges and day overrides, similar to the existing provider schedule templates. Also:
- Default spa hours should be 11am-9pm (not 6am-10pm)
- Need ability to click on existing schedule entries to modify them
- Need date-specific overrides (holiday hours, maintenance days)
- Need templates with start date and optional end date
- Need "copy template" functionality to create a new template from an existing one
- Need multi-day selection: check multiple days of the week, set times, "Add days" button

**Data Model Changes:**

New tables replacing `room_schedules`:

**`room_schedule_templates`**
| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `location_room_id` | BIGINT | FK → location_rooms |
| `name` | VARCHAR | e.g. "Summer Hours", "Winter Hours" |
| `effective_from_us` | BIGINT | When this template takes effect |
| `effective_to_us` | BIGINT | Nullable — when it ends (null = indefinite) |
| `is_active` | BOOLEAN | DEFAULT TRUE |
| `created_us` | BIGINT | |
| `updated_us` | BIGINT | |

**`room_schedule_template_entries`** (the weekly pattern)
| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `template_id` | BIGINT | FK → room_schedule_templates |
| `day_of_week` | INT | 0=Sunday, 6=Saturday |
| `open_time_minutes` | INT | Minutes from midnight |
| `close_time_minutes` | INT | Minutes from midnight |
| `created_us` | BIGINT | |

**`room_schedule_overrides`** (specific date overrides)
| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `location_room_id` | BIGINT | FK → location_rooms |
| `override_date_us` | BIGINT | The specific date being overridden (midnight) |
| `open_time_minutes` | INT | Nullable — null means CLOSED for this date |
| `close_time_minutes` | INT | Nullable |
| `reason` | VARCHAR | e.g. "Holiday hours", "Maintenance" |
| `created_us` | BIGINT | |

**Availability resolution logic:**
1. Find the active template for the room whose `effective_from_us <= now` and (`effective_to_us IS NULL` or `effective_to_us > now`)
2. For the requested date, check if there's an override → use override times (or closed if times are null)
3. If no override, use the template's entry for that day of week
4. If no template entry for that day, room is closed

**UI:**
- Template list: show all templates for a room, create new, copy from existing
- Template editor: weekly grid with multi-day selection + "Add days" + click to delete
- Override list: show upcoming overrides, create new override for a specific date
- The current `room_schedules` table entries should be migrated to a template

**Implementation plan:**
- [x] Create new tables: `room_schedule_templates`, `room_schedule_template_entries`, `room_schedule_overrides`
- [x] Register in make_database_info, create_database (all admin metadata)
- [x] Create table helpers for all three tables (12 tests in `room_schedule_templates_test.cpp`)
- [x] Update `RoomAvailabilityHelper` to use templates + overrides instead of `room_schedules` (6 tests)
- [x] Update admin endpoints: template CRUD, entry CRUD with multi-day "Add days", override CRUD (13 endpoint tests)
- [x] Update `available_service_slots_test.cpp` to use templates instead of old `room_schedules`
- [x] Update seed data: create default "Standard Hours" template with 11am-9pm every day
- [x] Update frontend ServerAccess: types, interface, proxy, network, mock, mock tests (12 mock tests)
- [x] Update admin UI: template list, template editor with multi-day add, override editor (21 component tests)
- [x] Fix the service booking page to actually show spa slots (dateFrom==dateTo fix, performance optimization)

#### 1.10 Spa Product Restructure

The spa should have **separate products** (not variants) for different time-of-day tiers, each with their own availability windows. Multiple products share the same room and room capacity.

**New spa products:**

| Product | Days | Hours | Duration | Price | Visibility |
|---|---|---|---|---|---|
| Early Bird Spa | Mon-Wed | 11am-2pm | 3 hours | $40 | Public |
| Non-Peak Spa | Mon-Wed 2pm-5pm, Thu-Fri 11am-5pm | 3 hours | $60 | Public |
| Peak Spa | Mon-Fri 5pm-8pm, Sat-Sun all day (11am-9pm) | 2 hours | $60 | Public |
| Late Night Spa | Mon/Wed/Thu | 8pm-9pm | 60 min | $10 | Gold members only |

**Key design decisions:**
- Each product has ONE variant (no multi-variant per product)
- Products share room capacity — total bookings across all products must not exceed room `concurrent_capacity`
- No 30-minute or 90-minute spa options
- Room schedule templates define when the room is OPEN; product booking windows define when each product is AVAILABLE within those hours
- The old "Spa Entry" product (with 5 variants) is removed

**New table: `product_booking_windows`**

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL | PK |
| `product_id` | BIGINT | FK → products |
| `day_of_week` | INTEGER | 0=Sun, 6=Sat |
| `open_time_minutes` | INTEGER | Minutes from midnight |
| `close_time_minutes` | INTEGER | Minutes from midnight |

**Availability resolution:**
1. Get room operating hours from template (or override)
2. Get product booking windows for the requested product
3. Intersect: available slots = room hours ∩ product hours
4. Check capacity across ALL room-based products sharing that room

**Other improvements:**
- 15-minute slot alignment for room-based products (was 5-minute)
- Reduced payload: omit provider fields (provider_person_id, provider_name, buffer_end_us) for room-based slots
- Fix variant chip CSS alignment in booking UI

**Implementation plan:**
- [x] Create `product_schedule_windows` table (schema, make_database_info, create_database registration in all 8 sections)
- [x] Create table helper for product_schedule_windows (5 tests)
- [x] Replace old spa products in seed data with 4 new products (Early Bird, Non-Peak, Peak, Late Night) + booking windows
- [x] Update `RoomAvailabilityHelper` to intersect room hours with product schedule windows; use 15-min alignment (3 new tests)
- [x] Fix booking page CSS: variant chip alignment (align-items: flex-start + line-height)
- [x] Hide empty provider name card header for room-based products
- [x] Fix week availability off-by-one (last day excluded)
- [x] Performance: preload bookings in memory instead of per-slot DB queries

---

### Phase 2: Shopping Cart
*Multi-item purchase capability — prerequisite for add-on bundles*

#### 2.1 Cart Service (Frontend)
- [x] Create `CartService` — in-memory cart state management (array of cart items)
- [x] Each cart item: `{ productId, variantId?, quantity, scheduledStartTimeUs?, providerPersonId?, facilityId?, productName, variantName?, priceCents, currency, durationMinutes? }`
- [x] Methods: `addItem()`, `removeItem()`, `clearCart()`, `getItems()`, `getItemCount()`
- [x] Observable `cartItems$`, `itemCount$`, `subtotalCents$` for reactive UI updates
- [x] Persist cart in localStorage for page refresh survival
- [x] Tests: 12 service spec tests (add, merge duplicates, scheduled items, remove, clear, persistence, observables)

#### 2.2 Cart UI Component
- [x] Cart page at `/shop/cart` with item list, remove buttons, subtotal, checkout
- [x] Cart item count badge (red) in header nav — only shows when items in cart
- [x] Shows scheduled service details (time, duration, provider)
- [x] Empty state with "Browse Services" CTA
- [x] Tests: 7 component spec tests

#### 2.3 Cart Checkout Page
- [x] Single-item cart routes to existing checkout flow
- [ ] Multi-item cart checkout with single purchase (Phase 3 when add-on bundles need it)
- [ ] Coupon/voucher applied to whole purchase
- [ ] Payment method

#### 2.4 Add to Cart Buttons
- [x] Service booking: "Add to Cart" alongside "Book and Pay" on confirm step
- [x] Product detail: "Add to Cart" alongside "Purchase"
- [x] Event booking: "Add to Cart" alongside "Book and Pay"
- [ ] Add-on suggestions (Phase 3)

---

### Phase 3: Add-On Bundles
*Product relationships with automatic discounts*

#### 3.1 `product_addons` Table
- [x] Create `db_schema/product_addons.h/cpp` — base_product_id, addon_product_id, discount_type, discount_value, is_stackable, extends_duration_by_base, sort_order, is_active
- [x] Register in all 8 admin metadata sections
- [x] Table helper: AddProductAddon, GetAddonsForProduct, GetBaseProductsForAddon, DeactivateAddon, DeleteAddon
- [x] Tests: 6 table helper tests

#### 3.2 `booking_groups` and `booking_group_members` Tables
- [x] Create `db_schema/booking_groups.h/cpp` — purchase_id FK
- [x] Create `db_schema/booking_group_members.h/cpp` — booking_group_id FK, booking_id FK, role (base/addon)
- [x] Register in all 8 admin metadata sections
- [x] Table helpers: BookingGroups (CreateGroup, GetGroupById, DeleteGroup), BookingGroupMembers (AddMember, GetMembersForGroup, GetGroupForBooking, DeleteMember)
- [x] Tests: 5 table helper tests

#### 3.3 Add-On Discount Logic
- [x] Created `addon_helper.h/cpp` with GetAddonsForProduct, CalculateAddonDiscount, DetectAddonDiscounts
- [x] Percentage and fixed discount types, capped at price
- [x] Non-stackable: skipped when coupon present; stackable: both apply
- [x] 10 unit tests

#### 3.4 Purchase Creation with Add-On Discounts
- [x] Extended `PurchaseHelper::CreatePurchase` to auto-detect and apply per-item add-on discounts
- [x] Stackability logic with coupons
- [x] 3 purchase_helper tests (addon applied, non-stackable+coupon, stackable+coupon)

#### 3.5 Duration Extension for Bundled Bookings
- [x] `cart_checkout_helper` extends addon session end_time_us by base duration when `extends_duration_by_base = true`
- [x] Creates booking_group with base + addon members
- [x] Test: DurationExtensionAndBookingGroup

#### 3.6 Linked Cancellation
- [x] `BookingHelper::CancelBooking` checks booking groups — cancels all members
- [x] Group deleted first to prevent recursion, then members cancelled
- [x] Test: CancelBookingGroupCancelsAllMembers
- [ ] Provider-cancel-base-only (keep addon with free cancel window) — deferred

#### 3.7 Admin Bundle Management
- [x] Created dedicated `/manage/bundles` page with full CRUD UI
- [x] Dashboard card for "Bundles" in manage section
- [x] Search bundles by product name (base or addon)
- [x] Create: select base + addon products, discount type/value, stackable, extends duration
- [x] Edit inline: update discount, stackable, extends duration
- [x] Delete (deactivate) with confirmation
- [x] Backend: GET/POST/POST update/POST delete endpoints with search support
- [x] 4 endpoint tests + 9 component spec tests + 9 mock spec tests

#### 3.8 Add-On Suggestions in Shopping Cart
- [x] `GET /api/addon_suggestions?product_ids=1,2,3` endpoint returns available add-ons with pricing
- [x] Cart page loads suggestions when items change
- [x] Green "Bundle and Save" section shows suggestions with original/discounted price and strikethrough
- [x] "Add" button adds suggestion to cart
- [x] Doesn't suggest products already in cart
- [x] 2 endpoint tests + 2 cart component spec tests + 1 mock spec test

---

### Phase 4: Book for Someone Else
*Extends purchase sharing to bookable services with invite flow*

#### 4.1 Service Booking for Another Person
- [x] Cart checkout accepts `for_person_id` to book for someone else
- [x] Server validates accepted gift permission (grantor→grantee) before allowing
- [x] Booking `person_id` = recipient, purchase `payer_person_id` = logged-in user
- [x] Booking for self (own ID or omitted) works without gift permission
- [x] Cart UI: "Who is this booking for?" expansion panel with radio (Myself/Someone else)
- [x] Loads accepted grantees from `searchGiftableUsers`
- [x] Selected person shown as badge in panel header
- [x] 3 backend tests (with permission, without permission, self)
- [x] 4 frontend tests (default self, panel visible, select user, clear on switch back)

#### 4.2 Invite Non-Member via Email
- [x] If user enters an email address of someone who isn't a member:
  - System sends an invitation email: "X has booked a Massage for you at Knotty Yoga! Create your account to view your booking."
  - The invitation includes a link to register
  - After registration, the person is prompted to accept purchase sharing from the gifter
  - Once accepted, the booking becomes visible in their "My Bookings"
- [x] If the email belongs to an existing member but purchase sharing isn't set up:
  - Send a purchase sharing request
  - Once accepted, the booking becomes visible
- [x] Tests: invite flow for non-member, request flow for existing member

#### 4.3 Same Patterns for Event Booking
- [x] Extend event booking with the same "Who is this for?" flow (for_person_id with gift permission validation, intended_for_email, invitation after payment)
- [x] Tests: event booking for another person (with permission, without permission, self, pending permission fails, combined with intended_for_email; frontend: forPersonId passed, inviteEmail passed, invitation sent after payment, defaults)

#### 4.4 Payer Cancellation, Transfer, and Wrong-Email Recovery
- [x] **Payer can cancel bookings they paid for**: The person who paid (payer) should be able to cancel a booking even if the booking's `person_id` is someone else. Currently only the booking's `person_id` can cancel. Add `payer_person_id` check to the cancellation endpoint so the payer can also initiate cancellation. Refund goes to the payer's original payment method. Also fixes security issue: strangers can no longer cancel arbitrary bookings.
- [x] **Booking transfer on sharing acceptance**: When a gift permission is accepted, check if the grantor has any bookings that were intended for the grantee (tracked via a new `intended_for_email` column on `purchases`). Reassign those bookings' `person_id` from the payer to the grantee.
- [x] **Wrong email recovery**: Allow the payer to change the `intended_for_email` on an untransferred purchase via `POST /api/purchase/{id}/change_recipient`. Resends invitation to the new email. Only allowed before the booking has been transferred (intended_for_email cleared).
- [x] **UI: "Booked for someone else" section in My Purchases**: Show purchases where the logged-in user is the payer but the booking is intended for someone else. Display intended_for_email. Provide "Change Recipient" inline edit action.
- [x] Tests: payer cancellation (owner, payer, stranger, unauth), transfer on acceptance (4 tests), email change (7 endpoint tests), UI states (7 component tests), mock spec (3 tests)

---

### Phase 5: Drop-In Bookings
*Staff-initiated on-the-fly bookings for walk-in customers*

#### 5.1 Staff Drop-In Endpoint
- [x] Create `POST /api/staff/dropin_booking` endpoint
- [x] Accepts: productId, variantId, personId, facilityId
- [x] Creates booking starting now (rounded to next 5-min boundary)
- [x] Checks room capacity via ServiceBookingHelper (reuses existing infrastructure)
- [x] Creates purchase at standard product price
- [x] Tests: success, not authenticated, missing fields, product not found, non-service product rejected (5 tests)

#### 5.2 On-the-Fly Account Creation
- [x] Create `POST /api/staff/create_quick_account` endpoint
- [x] Accepts: firstName, lastName, email
- [x] Creates account with auto-generated password (12 chars, alphanumeric + symbols)
- [x] Sets `must_change_password = true`
- [x] Account is fully validated (no email verification step)
- [x] Sends welcome email with temporary password and "please change immediately" message
- [x] Returns the new person_id for immediate use in drop-in booking
- [x] Duplicate email returns existing person (doesn't error)
- [x] Processes pending gift permission invitations for the email
- [x] Tests: success, welcome email sent, duplicate email returns existing, not authenticated, missing fields, account fully validated (7 endpoint tests + 5 email template tests)

#### 5.3 Staff Drop-In UI
- [x] Add to staff check-in page: "Walk-In Booking" expansion panel
- [x] Search for existing person via server autocomplete, or "New Customer" form
- [x] New Customer: collect first name, last name, email → create quick account → auto-select
- [x] ServerAccess: staffDropinBooking, staffCreateQuickAccount, staffSearchPeople added to all 4 layers
- [x] Server: facility_id defaults to first active facility when not specified
- [x] Server: admin bypass for for_person_id gift permission check (cart_checkout + book_event)
- [x] **Inline slot selection**: Product/variant dropdowns → date picker + hour pickers → "Show Available" button → clickable slot buttons with provider/time
- [x] **Inline payment**: PaymentMethodComponent shown after slot selection with price and "Book and Pay" button
- [x] **Book and pay**: `cartCheckout` (services) or `bookEvent` (events) with `for_person_id` + `purchasePayCard`. Admin bypass for gift permission.
- [x] Tests: component spec (10 tests), server admin bypass (2 tests)

#### 5.4 Walk-In Payment for Customer Account
- [x] **Reuse existing admin card endpoints** — `GET/POST /api/admin/person/{personId}/cards` already exist. Added `staffGetCustomerCards` and `staffCreateCustomerCard` to all 4 ServerAccess layers.
- [x] **PaymentMethodComponent: forPersonId input** — Loads customer's saved cards via `staffGetCustomerCards`. Card-on-file enabled for staff mode. "Manage cards" link hidden.
- [x] **Save card for customers** — `allowSaveCard` input shows checkbox + card name field. Saves to customer's account via `staffCreateCustomerCard`.
- [x] **Walk-in UI integration** — `[forPersonId]="walkinPersonId" [allowSaveCard]="true"` on the PaymentMethodComponent.
- [ ] **Coupon/voucher validation against customer** — Deferred (coupons are product-specific, voucher person-scoping is separate).
- [x] Tests: PaymentMethodComponent spec (11 tests), mock spec (3 staff card tests)

---

### Phase 6: iCal and Email Enhancements

#### 6.1 iCal Generation Utility
- [x] Create `util/ical_generator.h/cpp` — RFC 5545 compliant .ics generation
- [x] Generate `.ics` file content from: title, start time (UTC microseconds), end time (UTC microseconds), location, description
- [x] Times stored as UTC with Z suffix (no VTIMEZONE needed)
- [x] Proper escaping of commas, semicolons, newlines, backslashes
- [x] CRLF line endings per RFC 5545
- [x] Tests: 13 tests covering structure, timestamps, escaping, optional fields, CRLF

#### 6.2 Attach iCal to Booking Confirmation Emails
- [x] Added `MailAttachment` struct and `AddAttachment`/`GetAttachments` to `MailMessage`
- [x] Updated `MailHelperImpl::SendMail` to use mailio's `msg.attach()` for MIME attachments
- [x] Updated `cart_checkout.cpp` to generate and attach iCal for all bookable service items with a time slot
- [x] iCal end time resolved from `bookable_service_sessions.end_time_us` (falls back to 1 hour default)
- [x] iCal title includes variant name when present (e.g. "Peak Spa - 60 Min Session")
- [x] Tests: FullCartWithAllProductTypes verifies iCal on spa/massage/workshop emails, ServiceBookingEmailHasICalWithCorrectTimes verifies content, NoICalAttachmentForItemWithoutTimeslot

#### 6.3 Confirmation Email for Bundles
- [x] Created `bundle_booking_confirmation_mail.h/cpp` — consolidated email template showing all bundle components with pricing, discount, and cancellation policy note
- [x] Updated `cart_checkout.cpp` — detects bundle groups (base + addon) and sends a single combined email instead of separate emails per item; standalone items still get individual emails
- [x] Bundle email includes linked cancellation policy: "If you cancel one, the bundle discount will be removed and the remaining booking will be charged at full price"
- [x] Two separate `.ics` file attachments (one per booking) on the bundle email
- [x] Tests: `BundleBookingConfirmationMailTest` (6 tests: both items, discount, no-discount, cancellation policy, provider omission, HTML structure), `BundleDiscountAppearsInEmail` updated (verifies consolidated email with both products, discount, policy, 2 iCal attachments)

---

### Phase 7: Polish

#### 7.1 Purchase Detail Page
- [x] Created `GET /api/purchases/{id}` endpoint — returns purchase with items, entitlements (with assignments), payments, booking_ids. Auth: owner or admin.
- [x] Created `PurchaseDetailComponent` at `/my/purchases/:id` — full page view with items, entitlements, payments, summary
- [x] Integrated `SeatAssignmentComponent` for multi-seat entitlements — renders inline for entitlements with seats_total > 1
- [x] Added `getPurchase(id)` to all 4 ServerAccess layers (interface, proxy, network, mock)
- [x] Linked from purchase history ("View full details" link in each accordion panel)
- [x] Linked from service booking and event booking success pages ("Purchase Details" button)
- [x] Server tests: 5 tests (success with entitlements/payments, not authenticated, not found, not authorized, admin access)
- [x] Client tests: 10 tests (component creation, load/display, invalid id, 404, server error, items/entitlements/payments display, status labels, multi-seat detection)
- [x] Mock spec tests: 2 tests (getPurchase success, getPurchase 404)

#### 7.2 Checkout Success Enhancement
- [x] Service booking, event booking, and cart success screens now capture `PayCardResponse.entitlements`
- [x] Multi-seat entitlements show inline `SeatAssignmentComponent` on the success screen
- [x] All three flows link to purchase detail page (done in 7.1)
- [x] Cart success links to specific purchase detail instead of generic purchase list

#### 7.3 My Bookings Enhancement for Bundles
- [x] Server: `my_bookings` endpoint detects addon discount relationships between bookings sharing a purchase, populates `bundled_with_product_name` on each bundled booking
- [x] Client: "Bundled with: [product name]" label shown on both upcoming and past booking cards
- [x] Cancellation warning when cancelling a bundled booking: "Cancelling will remove the bundle discount from your [other product] booking, which will be charged at full price"
- [x] Server test: `GetMyBookingsBundleInfo` — verifies both bookings in a bundle have correct `bundled_with_product_name`
- [x] Client tests: 3 tests (bundle label shown, cancellation warning, no label for non-bundled)

#### 7.4 Catalog Multi-Seat Badge
- [x] Server: catalog endpoint looks up `seats_default` from `product_entitlement_rules` for each product, includes in response when > 1
- [x] Client: added `seats_default` to `CatalogProduct` interface and `ServerAccessNetwork` mapping
- [x] Catalog UI: "For N people" badge with group icon on multi-seat product cards
- [x] Checkout UI: "For N people — assign seats after purchase" note on multi-seat products
- [x] Server test: `GetCatalogProductsIncludesSeatsDefault` — verifies multi-seat product has `seats_default: 4`, single-seat product omits it
- [x] Client tests: 2 tests (badge shown for multi-seat, not shown for single-seat)

#### 7.5 Update Design Documents
- [ ] Update Payment Design Document with new scenarios and tables
- [ ] Update Support for scheduled purchases with new scenarios
- [ ] Mark completed checkboxes

---

### Phase 8: Automatic Comps, Vouchers, and Coupons
*Auto-detect and offer applicable discounts at checkout — no manual code entry needed*

#### 8.1 Backend: User-Compatible Discount Lookup
- [x] Created `GET /api/my_applicable_discounts?product_id=X` endpoint
- [x] Queries active coupons: within validity window, not maxed out, either unrestricted or restricted to the product via `coupon_products`
- [x] Queries user's vouchers: active, with balance, not expired, either unrestricted or restricted to the product
- [x] Comps are just vouchers — included automatically
- [x] Returns `{ coupons: [{code, discount_type, discount_value, description}], vouchers: [{code, remaining_value_cents, currency, description}] }`
- [x] Added `getMyApplicableDiscounts(productId)` to all 4 ServerAccess layers
- [x] Added TypeScript types: `ApplicableDiscountsResponse`, `ApplicableCoupon`, `ApplicableVoucher`
- [x] 12 server tests: empty, unrestricted coupon, product-specific coupon, wrong product coupon excluded, expired coupon excluded, user voucher, other user's voucher excluded, wrong product voucher excluded, matching product voucher, zero balance excluded, both coupons and vouchers, auth required

#### 8.2 Frontend: Auto-Suggest Discounts
- [x] On payment page load, calls `getMyApplicableDiscounts(productId)` for all three single-product flows
- [x] Panel auto-expands when discounts found, shows "Discounts available" badge in header
- [x] Clickable suggestion buttons above manual code entry with coupon/voucher icons and descriptions
- [x] Clicking a suggestion auto-fills the code and triggers the apply action
- [x] Implemented in: service booking, event booking, checkout (cart skipped — multi-product, handled in 8.3)
- [x] Suggestions hidden after a coupon or voucher is applied
- [x] Tests: 4 service-booking spec tests (loads suggestions, no suggestions, coupon click fills code, voucher click fills code)

#### 8.3 Shopping Cart Compatibility
- [ ] When shopping cart is implemented (Phase 2), check discounts for each item in the cart
- [ ] Aggregate applicable discounts across all cart items
- [ ] Show per-item discount suggestions in the cart summary
- [ ] Handle multi-product coupons that apply to multiple cart items

---

### Phase 9: Save Card Option on All Payment Flows
*Offer "Save card for future visits" on every payment page that accepts credit cards (except subscriptions, which already save cards automatically)*

#### 9.1 Identify All Payment Flows
- [ ] Service booking (`service-booking.component`) — user books a service and pays
- [ ] Event booking (`event-booking.component`) — user books an event and pays
- [ ] Cart / checkout (`cart.component`, `checkout.component`) — user checks out a cart and pays
- [ ] All of the above currently use `PaymentMethodComponent` without `allowSaveCard`

#### 9.2 Enable Save Card on Self-Service Payment
- [ ] Add `[allowSaveCard]="true"` to `PaymentMethodComponent` in each payment flow listed above
- [ ] The component already supports this — when `allowSaveCard` is true and mode is credit-card, it shows a "Save card for future visits" checkbox and card name field
- [ ] On save, uses the logged-in user's own person ID (not `forPersonId`) — need to add this path to `getPaymentSource()` which currently only saves for `forPersonId` (staff walk-in). Extend to also call the user's own `createCard` endpoint when `forPersonId` is null
- [ ] After saving, use `saved_card_id` for payment (same nonce-reuse pattern as walk-in flow)

#### 9.3 Server: Self-Service Card Save Endpoint
- [ ] Verify existing `POST /api/cards` endpoint supports saving a card from a nonce (it may already — check `createCard` in `ServerAccess`)
- [ ] If not, add or update to accept `source_id` (nonce) + `friendly_name` and create a saved card for the logged-in user
- [ ] Tests for the endpoint

#### 9.4 Tests
- [ ] Component spec tests for each payment flow verifying `allowSaveCard` is wired
- [ ] Integration test: user saves card during payment, card appears in saved cards list
- [ ] Verify subscriptions are NOT affected (they handle card saving separately)

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
