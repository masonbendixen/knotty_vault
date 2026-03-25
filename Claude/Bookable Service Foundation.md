---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/24/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Let's make a plan to implement NICE TO HAVE - Bookable Services Foundation in the Support for scheduled purchases.md document. Please use the codebase and the following documents for context:

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

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Please number each subsection within each phase. Please always add tests for anything you chance for which testing is possible.

# Plan: NICE TO HAVE — Bookable Services Foundation

## Scope

Implements the NICE TO HAVE scenarios from *Support for scheduled purchases.md*:

| # | Scenario | Status |
|---|----------|--------|
| 19 | Admin creates a bookable service type | To implement |
| 20 | Admin configures duration variants | Already done (Phase 8) |
| 21 | Admin configures buffer time | Already done (Phase 8) |
| 24 | Admin manages location resources | Already done |
| 25 | Admin enters availability blocks for provider | To implement |
| 29 | Provider enters unavailable blocks | To implement |
| 30 | User browses available service time slots | To implement |
| 32 | User books a service appointment | To implement |
| 33 | User receives confirmation email for service | To implement |
| 34 | User cancels service within full-refund window | To implement |
| 66 | Sequential slot computation prevents schedule holes | To implement |

Already completed (no work needed): 20, 21, 24.

## Current State

**Schema exists (all tables pre-created at startup):**
- `bookable_service_sessions` — id, product_id, product_variant_id, provider_person_id, facility_id, location_room_id, start_time_us, end_time_us, buffer_end_us, status, cancellation_reason, notes
- `provider_availability` — id, provider_person_id, facility_id, date_us, start_time_us, end_time_us, source, schedule_template_id, is_blocked
- `provider_types` — id, code, name, description
- `provider_type_assignments` — id, person_id, provider_type_id, is_accepting_bookings (unique: person_id + provider_type_id)
- `provider_buffer_overrides` — id, provider_person_id, product_variant_id, buffer_minutes (unique: provider_person_id + product_variant_id)
- `product_variants` — id, product_id, code, name, duration_minutes, buffer_minutes, sort_order, is_active
- `location_rooms` — id, facility_id, room_type_id, name, description, concurrent_capacity, is_active
- `location_room_types` — id, code, name, description
- `products` — has `provider_type_id`, `required_room_type_id`, `advance_booking_days`, `booking_cutoff_hours`
- `provider_type_assignments` — has `max_time_hole_minutes` (per-provider gap tolerance, nullable)

**What doesn't exist:**
- No table helpers for: provider_availability, provider_type_assignments, bookable_service_sessions, provider_types, provider_buffer_overrides, location_room_types
- No business logic for availability computation, service booking, or service cancellation
- No endpoints for: available_service_slots, book_service, admin provider availability
- No frontend: no service types, availability calendar, or service booking UI
- No link from products to provider_types (see Open Questions)

## Architecture Notes

```
DB Schema (exists) → Table Helpers → Business Logic → Endpoints → Frontend
```

Backend work listed before frontend in each phase. The critical algorithm is **Scenario 66: Sequential Slot Computation** — computing available time slots by intersecting provider availability, subtract existing bookings + buffers, check room availability, and prevent orphaned time holes.

---

## Phase 1: Table Helpers & Provider Setup (Scenarios 19, 25, 29)

**Goal**: Create CRUD table helpers for the service-related tables. Admin can create provider types, assign providers to types, and enter availability/unavailability blocks. This is the data foundation for everything else.

### 1.1 Backend — Table Helpers Layer

- [x] **`provider_types.h/cpp`** — CRUD: AddProviderType, GetProviderType, GetProviderTypes, DeleteProviderType
- [x] **`provider_types_test.cpp`** — 4 tests: add/get, get all, delete, not found

- [x] **`provider_type_assignments.h/cpp`** — CRUD: AddAssignment, GetAssignmentsForPerson, GetAcceptingAssignmentsForProviderType (is_accepting_bookings=true), UpdateAssignment, DeleteAssignment
- [x] **`provider_type_assignments_test.cpp`** — 4 tests: add/get, accepting filter, update, delete

- [x] **`provider_availability.h/cpp`** — CRUD: AddAvailability (with is_blocked flag), GetAvailabilityForProvider (date range, ordered ASC), GetAvailabilityForProviderOnDate, UpdateAvailability, DeleteAvailability
- [x] **`provider_availability_test.cpp`** — 5 tests: add/get, get on date, blocked, update, delete

- [x] **`provider_buffer_overrides.h/cpp`** — CRUD: AddOverride, GetOverride (by provider+variant), GetOverridesForProvider, DeleteOverride
- [x] **`provider_buffer_overrides_test.cpp`** — 3 tests: add/get, get for provider, delete

- [x] **`bookable_service_sessions.h/cpp`** — CRUD: AddServiceSession, GetServiceSession, GetServiceSessionsForProvider (excludes cancelled), GetServiceSessionsForRoom (excludes cancelled), UpdateServiceSession, DeleteServiceSession
- [x] **`bookable_service_sessions_test.cpp`** — 4 tests: add/get, get for provider (excludes cancelled), get for room, update/delete

- [x] **`location_room_types.h/cpp`** — CRUD: AddRoomType, GetRoomType, GetRoomTypes
- [x] **`location_room_types_test.cpp`** — 3 tests: add/get, get all, not found

- [x] **CMakeLists.txt** — All 12 source files + 6 test files added

### 1.2 Backend — Schema Change

- [x] **`products.h`** — Added `kProductsProviderTypeId = "provider_type_id"`
- [x] **`products.cpp`** — Added nullable FK column to `provider_types.id`

### 1.3 Backend — Seed Data

- [x] **`create_database.cpp`** — Provider types already seeded: instructor, therapist. Added: personal_trainer.
- [x] **`create_database.cpp`** — Location room types already seeded: studio, massage_room. Added: treatment_room.

### 1.4 Backend — Admin Endpoints

- [x] **`admin_provider_availability.h/cpp`** — `POST /api/admin/provider_availability`, requires auth, body `{ provider_person_id, facility_id, date_us, start_time_us, end_time_us, is_blocked }`, returns created record with ID. `is_blocked=true` creates unavailable blocks (scenario 29).
- [x] **`admin_provider_availability_test.cpp`** — 4 tests: create availability block, create blocked availability, requires auth (401), missing field (400)
- [x] **Registered in `web_app.cpp`** and **`endpoints/CMakeLists.txt`**

### 1.5 Admin UI — Product Detail Page

- [x] **Product detail page** — Added `provider_type_id` dropdown (loads all provider_types, shown only for bookable_service kind)
- [x] **Product detail page** — Added `required_room_type_id` dropdown (loads all location_room_types, shown only for bookable_service kind)
- [x] **Product detail page** — Added `booking_cutoff_hours` number input (shown for all product kinds in Access & Scheduling)
- [x] **`max_time_hole_minutes` moved to provider** — Now on `provider_type_assignments` table (per-provider, not per-product). Providers control their own gap tolerance. Removed from product detail page.
- [x] **`advance_booking_days` deferred** — Will be implemented as permission-based booking window matrix in Phase 2
- [x] **Seed data** — Massage changed to `bookable_service` kind with 5-min buffers. Provider type "Therapist" renamed to "Massage Therapist".

---

## Phase 2: Availability Computation Algorithm (Scenarios 66, 30)

**Goal**: Build the core algorithm that computes available time slots for a bookable service. This is the most complex piece. The slot computation model is **sequential, not interval-based** — slots are generated from free windows between existing bookings, and only offered if they don't create unusable gaps.

### Slot Computation Model (Scenario 66)

The algorithm does NOT scan at fixed intervals. Instead, it works with **free windows** derived from provider availability minus existing bookings. For each free window, it generates the valid start times for each variant, ensuring no booking would create an orphaned gap.

**Example**: Provider available 8am-2pm. Variants: 60min, 90min, 120min. Buffer: 5min.
- Booking at 8:00-9:00 (60min massage, buffer ends 9:05)
- Next available start: **9:05am** (buffer_end rounded up to 5-min boundary)
- From 9:05am, the system lists slots for each variant that fits:
  - 60min: 9:05-10:05 (buffer ends 10:10)
  - 90min: 9:05-10:35 (buffer ends 10:40)
  - 120min: 9:05-11:05 (buffer ends **11:15** — double buffer, see below)
- If someone books 10:40am, then the 9:05-10:35 window only fits:
  - 60min: 9:05-10:05 ✓ (leaves 30min gap → too small for any variant → **rejected**)
  - 90min: 9:05-10:35 ✓ (no gap, fills exactly → **offered**)
  - So only the 90min variant is offered at 9:05am

**Multi-buffer rule for long variants**: A variant whose duration is a multiple of the base variant's duration gets proportional buffers. Specifically:

`num_buffers = (duration % base_duration == 0) ? (duration / base_duration) : 1`
`total_buffer = num_buffers * buffer_minutes`

Only exact multiples of the base duration get proportional buffers. This ensures subdivisibility: a 120min slot with 2 buffers can be replaced by two 60min slots with 1 buffer each. Non-multiples (like 90min) are single blocks that can't be subdivided, so they get 1 buffer.

For the example above (base = 60min, buffer = 5min):
- 60min → 60/60 = 1, exact → 1 buffer → buffer_end = end + 5min
- 90min → 90/60 = 1.5, not exact → 1 buffer → buffer_end = end + 5min
- 120min → 120/60 = 2, exact → 2 buffers → buffer_end = end + 10min
- 180min → 180/60 = 3, exact → 3 buffers → buffer_end = end + 15min

**Why this matters**: A 120-minute slot with double buffer can always be equivalently replaced by two 60-minute slots with a buffer between them. If a 120-minute booking at 9:05 only had a single 5-minute buffer (ending at 11:10), then cancelling it and rebooking as two 60-minute massages would require: 9:05-10:05 (buffer to 10:10) + 10:10-11:10 (buffer to 11:15). The second massage's buffer extends to 11:15, which is past the original 11:10 buffer_end — creating a conflict with whatever was booked after. Double buffer ensures the time block is always subdivisible into smaller variants without overlap.

**Bidirectional hole prevention**: The hole check applies in BOTH directions — a slot can't create an unusable gap *after* it (before the next booking or window end) OR *before* it (after the previous booking's buffer_end or window start).

**Example**: Provider available 8:00am-4:00pm. Buffer: 5min. Smallest variant: 60min.
- The valid start times from the window start are: **8:00am** and **9:05am** (and onward in 65-min steps from 8:00).
- **8:00am is valid** — it's the window start, no gap before it.
- **8:05am is NOT valid** — it leaves a 5-min gap before it (8:00-8:05) which is too small for any 60-min variant.
- **8:10am, 8:15am, ... 9:00am are all NOT valid** — each leaves a gap at the start smaller than 60 minutes.
- **9:05am IS valid** — it leaves exactly 65 minutes from 8:00 (enough for a 60min + 5min buffer). So either 8:00 is booked (and 9:05 follows naturally), or the 8:00-9:05 slot can fit a 60-min variant.
- The same logic applies throughout: each candidate start time is checked to ensure the gap between the previous booking's buffer_end (or window start) and this slot's start can fit at least one variant + buffer, OR is exactly zero (the slot starts right at the boundary).

In general, for a free window, valid start times are: `window_start`, `window_start + (smallest_duration + buffer)`, `window_start + 2*(smallest_duration + buffer)`, etc. — all rounded to 5-minute boundaries. Each of these is then checked per-variant to see which durations fit without creating a hole after it.

**End-of-window best-fit rule**: The last slot before the end of an availability window (or before the next booking) is a special case. A trailing gap is unavoidable if the remaining time doesn't divide evenly. In this case, the system offers only the **longest variant that fits** — not shorter ones that would leave a larger unused gap. Example: 110 minutes remaining, variants are 60/90/120min:
- 120min → doesn't fit → **rejected**
- 90min → fits, leaves 20min gap → **offered** (best fit)
- 60min → fits, but leaves 50min gap → **rejected** (90min is a better fit and should be preferred)

The logic: at the last valid start time before a window end, compute the remaining time. For each variant (sorted longest first), check if it fits. Offer the **first (longest) variant that fits**. Skip shorter variants because they'd waste more time. If no variant fits, don't offer this start time at all.

Note: this only applies to the *last* slot in a window. Interior slots use the standard bidirectional hole check (which naturally handles this via the trailing gap rule).

**Key rules:**
- Start times are on **5-minute boundaries** (configurable constant `kSlotAlignmentMinutes = 5`)
- Duration and buffer values must be multiples of 5 (validated on input)
- A slot is only offered if booking it would NOT create a gap before OR after it that is smaller than the smallest active variant's duration (+ buffer) for the product. A zero-length gap (slot flush against boundary or adjacent booking) is always fine.
- **Exception**: The last slot in a window may leave a trailing gap. In this case, only the longest fitting variant is offered (best-fit rule).
- Long variants get proportional buffers: `num_buffers = ceil(duration / base_duration)`, where base_duration is the smallest active variant's duration for the product
- The freed time after cancellation of a long variant can be split into smaller variant slots (the proportional buffers ensure this always works without overlap)
- The algorithm generates slots per-variant (the frontend shows which durations are available at each start time)

### 2.1 Backend — Business Logic Layer

- [x] **`ServiceAvailabilityHelper` class** created in `business_logic/scheduling/`
  - `service_availability_helper.h/cpp`
  - Dependencies: `DatabaseHelper`

  - **`ComputeAvailableSlots(Transaction&, request)` → `std::vector<AvailableSlot>`**
    - Request: `{ productId, providerPersonId (optional), facilityId, dateFromUs, dateToUs, personId (for permission-based booking window) }`
    - Note: returns slots for ALL active variants, not a single variant. The frontend groups by start time and shows which durations are available.
    - Steps:
      1. Load the product to get `provider_type_id`, `required_room_type_id`, `advance_booking_days`, `booking_cutoff_hours`
      2. Load all active product variants for this product → get duration_minutes and buffer_minutes for each. Determine `base_duration` = smallest variant's duration_minutes. Determine `min_slot` = base_duration + buffer (the smallest bookable unit of time).
      3. Determine the user's booking window based on their permissions (see Phase 2 booking window design below)
      4. If no specific provider requested: find all providers assigned to the product's provider type who are accepting bookings
      5. For each provider:
         a. Load provider availability blocks for the date range (exclude `is_blocked = true`)
         b. Load existing bookable_service_sessions for the provider in the date range (ordered by start_time_us)
         c. Load provider buffer overrides for each variant
         d. Compute **free windows**: subtract booked sessions (using `buffer_end_us`) from availability windows
         e. For each free window, generate **valid start times**:
            - First start = window_start (rounded up to 5-min boundary)
            - Subsequent starts = previous_start + min_slot, rounded up to 5-min boundary
            - This ensures no gap before any start time is smaller than min_slot (bidirectional hole prevention)
         f. For each valid start time, determine if this is the **last start time** in the window (no room for another min_slot after it). Then for each variant (sorted **longest first**):
            - Calculate proportional buffer: `num_buffers = ceil(variant.duration / base_duration)`, `total_buffer = num_buffers * effective_buffer` where effective_buffer = `MAX(variant.buffer_minutes, provider_override.buffer_minutes)`
            - Check if `start + duration + total_buffer <= window_end` (or `start + duration <= window_end` if this is the very end of the window and no trailing buffer is needed)
            - If this is NOT the last start time: **trailing hole check** — if `slot.buffer_end` to `window_end` leaves a gap smaller than `min_slot` and greater than zero → reject this variant
            - If this IS the last start time: **best-fit rule** — offer only the **longest variant that fits**. Skip shorter variants (they'd waste more time). This is the one case where a trailing gap is acceptable.
            - If valid, add to results
      6. Check room availability: for the product's `required_room_type_id`, find rooms at the facility. For each slot, verify at least one room has capacity (check `concurrent_capacity` against overlapping sessions)
      7. Apply booking window filter (based on user's permissions)
      8. Filter out slots in the past
      9. Return results grouped by provider

  - **`AvailableSlot` struct**:
    ```cpp
    struct AvailableSlot {
        int64_t providerPersonId = 0;
        std::string providerName;
        int64_t facilityId = 0;
        std::string facilityName;
        int64_t startTimeUs = 0;
        int64_t endTimeUs = 0;       // start + duration (no buffer)
        int64_t bufferEndUs = 0;     // end + buffer
        int64_t productVariantId = 0;
        std::string variantName;
        int64_t durationMinutes = 0;
    };
    ```

  - **`FreeWindow` helper struct** (internal):
    ```cpp
    struct FreeWindow {
        int64_t startUs;  // Start of free time
        int64_t endUs;    // End of free time (next booking start or availability end)
    };
    ```

### Permission-Based Booking Windows (from Open Question #5)

Members get earlier access to service booking. This uses a **booking window matrix** that mirrors the pricing matrix — configurable per product/permission combination, managed on the product detail page alongside pricing.

- [x] **Design**: Create a `product_booking_windows` table:
  ```
  product_booking_windows
    id              BIGSERIAL PRIMARY KEY
    product_id      FK → products (required)
    permission_id   FK → permissions (nullable — NULL = base window for anyone)
    advance_days    BIGINT (how many days ahead this tier can book)
    created_us      BIGINT
    updated_us      BIGINT
    UNIQUE(product_id, permission_id)
  ```
  - Same pattern as `product_prices`: one row per product/permission combination
  - `permission_id = NULL` → base window for users without any matching permission (the "public" row)
  - The availability endpoint resolves the user's best booking window: find all `product_booking_windows` rows for this product where the user has the permission, take the maximum `advance_days`
  - The availability endpoint filters: `slot.startTimeUs <= now + best_advance_days * 86400000000`
  - Example: Massage product has rows: `(permission=NULL, advance_days=7)`, `(permission=gold_membership, advance_days=30)`, `(permission=platinum, advance_days=60)`. A Gold member can book 30 days ahead; a non-member can book 7 days ahead.

- [x] **Admin UI**: Booking window matrix on product detail page — table with "Anyone" base row + one row per permission. Click-to-edit inline cells showing advance days. Creates/updates/deletes `product_booking_windows` rows. Reuses `pricing-matrix` CSS classes. Help text explains that higher-tier members get the best window across all their permissions.

- [x] **DB schema**: `product_booking_windows.h/cpp` — id, product_id (FK), permission_id (FK nullable), advance_days, unique(product_id, permission_id). Added to make_database_info, db_schema CMakeLists, and create_database table creation.
- [x] **Table helper**: `product_booking_windows.h/cpp` — AddWindow, GetWindowsForProduct (DESC), GetWindow (handles NULL permission), ResolveAdvanceDaysForUser (SQL MAX across base + user's permissions via role chain), UpdateWindow, DeleteWindow
- [x] **Tests**: 11 tests — CRUD, multiple windows ordering, get by product+permission, resolve for base user, gold member, platinum (max of all tiers), no windows returns 0, user without permission gets base only
- [x] **Seed data**: Massage product gets base booking window of 7 days

- [x] **Tests** in `service_availability_helper_test.cpp` — 35+ tests covering:
  - Pure algorithm (15): empty/small window, exact fit, multiple variants, bidirectional hole prevention, proportional buffers (120min double, 90min single), end-of-window best-fit, trailing hole, alignment, buffer override
  - Doc scenarios (9): free window after booking with all variants, booking at 10:40 → only 90min, double buffer subdivisibility, five massages in 5h20m, multiple free windows, perfect fit, buffer end values match doc, no invalid start times
  - Critical business (6): 120min OR two 60min in 130min window, 90min best-fit rejects 60min, interior/last slot variant interplay, cancelled double booking fits two 60s, 160min window variant mix, 195min window three 60s or 120+60
  - Edge cases (13): 59min window, exact 60min, two variants only, single 120min, buffer override MAX, zero buffer, full day coverage, consistent spacing, 180min triple buffer, alignment edge cases
  - Integration (6): basic availability, blocked excluded, booking splits window, provider hidden, buffer override, room capacity

- [x] **CMakeLists.txt** — Added service_availability_helper.h/cpp and test

### 2.2 Backend — KVT Conversion

- [x] **`scheduling_key_value_table.h/cpp`** — Added `AvailableSlotToKeyValueTable` and `AvailableSlotsToKeyValueTableArray`
- [x] **Tests** — 2 tests: single slot with all fields, array conversion

### 2.3 Backend — Endpoint Layer

- [x] **`GET /api/available_service_slots`** — params: `product_id`, `date_from`, `date_to`, `provider_id` (opt), `facility_id` (opt). Requires auth. Returns `{ "slots": [...], "variants": [{ id, name, duration_minutes, currency, amount_cents }] }`. Pricing resolved per-variant.
- [x] **Registered** in `web_app.cpp` and `endpoints/CMakeLists.txt`

---

## Phase 3: Service Booking Flow (Scenarios 32, 33)

**Goal**: User selects a time slot and books a service appointment. Creates a purchase, booking, and bookable_service_session. Sends confirmation email.

### 3.1 Backend — Business Logic Layer

- [x] **`ServiceBookingHelper` class** in `business_logic/scheduling/`
  - `service_booking_helper.h/cpp`

  - **`BookService(Transaction&, request)` → `BookServiceResult`**
    - Request: `{ productId, productVariantId, providerPersonId, facilityId, startTimeUs, personId, payerPersonId }`
    - Steps:
      1. Validate the slot is still available (re-compute to prevent race conditions)
      2. Verify the user's booking window permits this date (permission-based advance booking)
      3. Resolve pricing via CatalogHelper (variant-aware)
      4. Create purchase + purchase_item
      5. **Room auto-assignment**: Find an available room of the required type. Prefer the room the provider was already using earlier that day (provider room affinity). If no prior room or it's unavailable, use the first available room by ID.
      6. Calculate endTimeUs = startTimeUs + duration_minutes * 60 * 1000000
      7. Calculate bufferEndUs = endTimeUs + effective_buffer_minutes * 60 * 1000000
      8. Create bookable_service_session record
      9. Create booking record (with `service_session_id`, not `event_session_id`)
      10. Return result

- [x] **Tests** in `service_booking_helper_test.cpp` — 6 tests: successful booking (purchase+session+booking, correct times, room assigned), slot no longer available, room auto-assigned, room affinity prefers same room, buffer end correct for 90min (single buffer), product not found

### 3.2 Backend — Email

- [x] **`service_booking_confirmation_mail.h/cpp`** — Email with service name, variant, provider, date/time, facility/room, amount, optional cancellation policy
- [x] **Tests** — 2 tests: all fields, optional fields omitted

### 3.3 Backend — Endpoint Layer

- [x] **`POST /api/book_service`** — Body: `{ product_id, variant_id, provider_person_id, facility_id, start_time_us }`. Requires auth. Returns `{ purchase, booking, service_session }`. Sends confirmation email.
- [x] **Tests** — 3 tests: success, auth required (401), missing field (400)
- [x] **Registered** in `web_app.cpp` and `endpoints/CMakeLists.txt`

### 3.4 Backend — KVT Conversion

- [x] **Service session uses raw KeyValueTable** from `GetServiceSession()` → `KeyValueTableToJson()` directly. No separate struct needed.

### 3.5 Backend — Extend `my_bookings`

- [x] **Update `GET /api/my_bookings`** to include service bookings
  - Currently only returns event bookings (joins on `event_session_id`)
  - Add a second query for bookings with `service_session_id`
  - Include provider name, duration, room name in the response
  - Support `type=event|service|all` query param filter

---

## Phase 4: Service Cancellation (Scenario 34)

**Goal**: User cancels a service booking. Refund calculated via the product's cancellation policy (reuses existing RefundHelper). Service session cancelled, freeing the provider time slot and room.

### 4.1 Backend — Business Logic Layer

- [x] **Extend `BookingHelper::CancelBooking()`** to handle service bookings
  - When booking has `service_session_id` (not `event_session_id`):
    - Load the service session → get product_id, start_time_us
    - Load the product → get cancellation_policy_id
    - Calculate refund via `RefundHelper::CalculateRefundPercent()` (same as events)
    - Process refund via `RefundHelper::ProcessRefund()` (same as events)
    - Set service session status to `cancelled`
    - Freed slot becomes available automatically (availability computation ignores cancelled sessions)
  - No changes needed to `cancel_booking` endpoint — it already calls `BookingHelper::CancelBooking()`
- [x] **Tests** — Cancel service booking with refund, cancel service booking without payment

### 4.2 Backend — Email

- [x] **Send cancellation email** — Reuse existing `BookingCancellationMail` with service-specific details (provider, duration)
  - May need to extend the `BookingCancellationData` struct with optional service fields
- [x] **Tests**

---

## Phase 5: Frontend — Service Browsing & Booking

**Goal**: User can browse available time slots, select a provider and time, and book a service.

### 5.1 Frontend — Types

- [x] **`scheduling.types.ts`** — Add:
  - `AvailableSlot`, `AvailableSlotsResponse`, `BookServiceRequest`, `BookServiceResponse`

### 5.2 Frontend — ServerAccess Layer

- [x] **`ServerAccess.ts`** (interface) — Add `getAvailableServiceSlots()` and `bookService()` methods
- [x] **`ServerAccessNetwork.ts`** — HTTP implementations
- [x] **`ServerAccess.mock.ts`** — Mock implementations
- [x] **`ServerAccess.mock.spec.ts`** — Tests

### 5.3 Frontend — Service Catalog Page

- [x] **New component: `service-catalog`** at `/shop/services`
  - Lists bookable_service products with variant options and prices
  - Each product shows provider photos, names, and link to bio page
  - "Book" button navigates to service booking page

### 5.4 Frontend — Service Booking Page

- [x] **New component: `service-booking`** at `/shop/service/:productId`
  - Step 1: Select date (date picker — week view)
  - Step 2: View available slots grouped by provider. Each provider shows:
    - Provider photo and name (linked to bio page)
    - Available start times, each showing which variant durations are available
    - Example: "9:05 AM — 60min, 90min, 120min" or "11:10 AM — 90min only"
  - Step 3: Select a specific start time + duration → shows price, cancellation policy
  - Step 4: Confirm & Pay (payment method + book via existing PaymentMethodComponent)
  - Shows cancellation policy info (reuse existing `cancellationPolicyText` pattern)
  - Success confirmation with provider, duration, time, location, room
  - If user's booking window doesn't reach the selected date, show message explaining when booking opens (e.g., "Gold members can book 30 days ahead — upgrade your membership for earlier access")

### 5.5 Frontend — My Bookings Integration

- [x] **Update `my-events.component`** to show service bookings
  - Service bookings show: service name, variant (duration), provider name + photo, date/time, facility/room
  - Cancel button works the same (reuses existing cancel infrastructure with refund policy)

### 5.6 Frontend — Provider Bio Page

- [x] **New component: `provider-bio`** at `/providers/:personId`
  - Provider photo, name, bio text
  - List of services they offer (provider_type_assignments → products)
  - "Book with [provider name]" button linking to service booking page filtered to this provider

### 5.7 Frontend — Tests

- [x] **Component spec tests** for service-catalog, service-booking, provider-bio, updated my-events

---

## Phase 6: Admin Provider Management UI

**Goal**: Admin manages providers, availability, and service configuration.

### 6.1 Frontend — Provider Availability Admin

- [ ] **New component: `provider-availability`** at `/manage/providers/:personId/availability`
  - Weekly view of provider's availability blocks and booked sessions
  - Add/edit/delete availability blocks

### 6.2 Frontend — Provider List

- [ ] **New component: `provider-list`** at `/manage/providers`
  - Lists people with provider_type_assignments
  - Link to availability management

### 6.3 Admin — Product Detail Updates

- [ ] **Product detail page** — For bookable_service products, add dropdowns/inputs for:
  - `provider_type_id`, `required_room_type_id`, `booking_cutoff_hours`
  - Note: `max_time_hole_minutes` is per-provider on `provider_type_assignments`, managed in the provider management UI. `advance_booking_days` is managed through the booking window matrix.

### 6.4 Test Helper Commands

- [ ] **Add `list_providers` command** — Lists people with provider_type_assignments
- [ ] **Add `add_provider_availability` command** — Creates availability blocks for a provider
- [ ] **Add `list_available_slots` command** — Computes and displays available service slots

---

## Dependencies & Ordering

```
Phase 1 (Table Helpers & Provider Setup) — Foundation, must be first
Phase 2 (Availability Algorithm) — Depends on Phase 1
Phase 3 (Booking Flow) — Depends on Phase 2
Phase 4 (Cancellation) — Depends on Phase 3
Phase 5 (Frontend Booking) — Depends on Phases 2-4 for backend APIs
Phase 6 (Admin UI) — Can start after Phase 1, parallel with Phases 3-5
```

---

## Resolved Questions

1. **Product → Provider Type linkage**: **Option A** — Add `provider_type_id` FK to the `products` table. One provider type per product.

2. **Room auto-assignment strategy**: **Option A with affinity** — First available room of the required type, but prefer the room the provider was already using earlier that day. Falls back to first available by ID if prior room is full.

3. **Slot granularity**: **5-minute boundaries** with sequential (not interval-based) slot generation. Slots are NOT generated at fixed intervals. Instead, the algorithm computes free windows from provider availability minus existing bookings, and generates valid start times at 5-minute boundaries. Only slots that don't create orphaned gaps are offered. Duration and buffer values must be multiples of 5 minutes (validated on input). This gives providers flexible buffers (5, 10, 15, 20 min) while keeping start times clean (no 10:47am). Five 60-min massages with a 5-min buffer = 5h 20min instead of 6h with 15-min boundaries.

4. **Provider selection UX**: **Option C** — Slots grouped by provider. User picks both time and provider. Provider shows photo, name, and link to bio.

5. **Booking windows**: **Permission-based booking window matrix** — Uses a dedicated `product_booking_windows` table (same pattern as `product_prices`). One row per product/permission combination, each specifying `advance_days`. `permission_id = NULL` is the base window for anyone. Admin configures the matrix on the product detail page alongside the pricing matrix. The availability endpoint resolves the user's best window (max `advance_days` across all their permissions) and filters slots accordingly. Frontend shows an upgrade prompt when dates are outside the user's window.

6. **Cancel endpoint**: **Same endpoint** — `cancel_booking` handles both event and service bookings via `event_session_id` vs `service_session_id`.

7. **Time hole enforcement**: **Option A** — Enforced at slot computation time. Don't show slots that would create unusable gaps.

8. **Empty availability**: **Hidden** — Providers with no availability in the date range don't appear in results.

9. **Provider photos and bios**: **Yes** — Show provider photo, name, and link to bio page on the service catalog and booking pages. New `/providers/:personId` bio page.

10. **Timezone handling**: **UTC microseconds** — Store as UTC, admin enters in facility timezone, frontend converts using facility's timezone field.