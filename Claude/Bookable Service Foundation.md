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
- `products` — has `required_room_type_id`, `advance_booking_days`, `booking_cutoff_hours`, `max_time_hole_minutes`

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

- [ ] **Create `provider_types.h/cpp`** table helper
  - `AddProviderType(Transaction&, code, name, description)` → `int64_t`
  - `GetProviderType(Transaction&, int64_t id)` → `KeyValueTable`
  - `GetProviderTypes(Transaction&)` → `KeyValueTableArray`
  - `DeleteProviderType(Transaction&, int64_t id)`
- [ ] **Tests** for provider_types table helper

- [ ] **Create `provider_type_assignments.h/cpp`** table helper
  - `AddAssignment(Transaction&, int64_t personId, int64_t providerTypeId)` → `int64_t`
  - `GetAssignmentsForPerson(Transaction&, int64_t personId)` → `KeyValueTableArray`
  - `GetAssignmentsForProviderType(Transaction&, int64_t providerTypeId)` → `KeyValueTableArray` (only `is_accepting_bookings = true`)
  - `UpdateAssignment(Transaction&, int64_t id, const KeyValueTable& updates)`
  - `DeleteAssignment(Transaction&, int64_t id)`
- [ ] **Tests** for provider_type_assignments table helper

- [ ] **Create `provider_availability.h/cpp`** table helper
  - `AddAvailability(Transaction&, int64_t providerPersonId, int64_t facilityId, int64_t dateUs, int64_t startTimeUs, int64_t endTimeUs, std::string_view source, bool isBlocked)` → `int64_t`
  - `GetAvailabilityForProvider(Transaction&, int64_t providerPersonId, int64_t dateFromUs, int64_t dateToUs)` → `KeyValueTableArray` (ordered by start_time_us ASC)
  - `GetAvailabilityForProviderOnDate(Transaction&, int64_t providerPersonId, int64_t dateUs)` → `KeyValueTableArray`
  - `UpdateAvailability(Transaction&, int64_t id, const KeyValueTable& updates)`
  - `DeleteAvailability(Transaction&, int64_t id)`
- [ ] **Tests** for provider_availability table helper

- [ ] **Create `provider_buffer_overrides.h/cpp`** table helper
  - `AddOverride(Transaction&, int64_t providerPersonId, int64_t productVariantId, int64_t bufferMinutes)` → `int64_t`
  - `GetOverride(Transaction&, int64_t providerPersonId, int64_t productVariantId)` → `KeyValueTable`
  - `GetOverridesForProvider(Transaction&, int64_t providerPersonId)` → `KeyValueTableArray`
  - `DeleteOverride(Transaction&, int64_t id)`
- [ ] **Tests** for provider_buffer_overrides table helper

- [ ] **Create `bookable_service_sessions.h/cpp`** table helper
  - `AddServiceSession(Transaction&, const KeyValueTable& values)` → `int64_t`
  - `GetServiceSession(Transaction&, int64_t id)` → `KeyValueTable`
  - `GetServiceSessionsForProvider(Transaction&, int64_t providerPersonId, int64_t dateFromUs, int64_t dateToUs)` → `KeyValueTableArray`
  - `GetServiceSessionsForRoom(Transaction&, int64_t locationRoomId, int64_t dateFromUs, int64_t dateToUs)` → `KeyValueTableArray`
  - `UpdateServiceSession(Transaction&, int64_t id, const KeyValueTable& updates)`
  - `DeleteServiceSession(Transaction&, int64_t id)`
- [ ] **Tests** for bookable_service_sessions table helper

- [ ] **Create `location_room_types.h/cpp`** table helper
  - `AddRoomType(Transaction&, code, name, description)` → `int64_t`
  - `GetRoomType(Transaction&, int64_t id)` → `KeyValueTable`
  - `GetRoomTypes(Transaction&)` → `KeyValueTableArray`
- [ ] **Tests** for location_room_types table helper

- [ ] **CMakeLists.txt** — Add all new table helper files

### 1.2 Backend — Schema Change

- [ ] **Add `provider_type_id` FK to `products` table** — Links a bookable_service product to the provider type that can serve it (see Open Question #1)
  - Add `kProductsProviderTypeId` to `db_schema/products.h`
  - Add FK column in `products.cpp` DDL
  - Add to admin metadata (friendly names, display templates)

### 1.3 Backend — Seed Data

- [ ] **`create_database.cpp`** — Add seed data for provider types (e.g., "massage", "personal_training", "yoga_private")
- [ ] **`create_database.cpp`** — Add seed data for location room types (e.g., "massage_room", "treatment_room") if not already present

### 1.4 Backend — Admin Endpoints

- [ ] **Create `POST /api/admin/provider_availability`** endpoint
  - Body: `{ provider_person_id, facility_id, date_us, start_time_us, end_time_us, is_blocked }`
  - Creates an availability block (or blocked/unavailable block if `is_blocked=true`)
  - Returns the created record
  - Scenarios 25 and 29 use the same endpoint — `is_blocked` distinguishes available vs unavailable
- [ ] **Tests** for provider availability endpoint
- [ ] **Register endpoint** in `web_app.cpp` and `endpoints/CMakeLists.txt`

### 1.5 Admin UI — Product Detail Page

- [ ] **Product detail page** — Add `provider_type_id` dropdown for bookable_service products (Access & Scheduling section)
- [ ] **Product detail page** — Add `required_room_type_id` dropdown for bookable_service products
- [ ] **Product detail page** — Add `advance_booking_days` and `booking_cutoff_hours` number inputs
- [ ] **Product detail page** — Add `max_time_hole_minutes` number input

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

`effective_buffers = ceil(variant.duration_minutes / base_variant.duration_minutes) * buffer_minutes`

For the example above (base = 60min, buffer = 5min):
- 60min → 1 buffer → buffer_end = end + 5min
- 90min → 1 buffer → buffer_end = end + 5min (90 is not a multiple of 60, treated as a single block)
- 120min → 2 buffers → buffer_end = end + 10min

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

**Key rules:**
- Start times are on **5-minute boundaries** (configurable constant `kSlotAlignmentMinutes = 5`)
- Duration and buffer values must be multiples of 5 (validated on input)
- A slot is only offered if booking it would NOT create a gap before OR after it that is smaller than the smallest active variant's duration (+ buffer) for the product. A zero-length gap (slot flush against boundary or adjacent booking) is always fine.
- Long variants get proportional buffers: `num_buffers = ceil(duration / base_duration)`, where base_duration is the smallest active variant's duration for the product
- The freed time after cancellation of a long variant can be split into smaller variant slots (the proportional buffers ensure this always works without overlap)
- The algorithm generates slots per-variant (the frontend shows which durations are available at each start time)

### 2.1 Backend — Business Logic Layer

- [ ] **Create `ServiceAvailabilityHelper` class** in `business_logic/scheduling/`
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
         f. For each valid start time, for each variant:
            - Calculate proportional buffer: `num_buffers = ceil(variant.duration / base_duration)`, `total_buffer = num_buffers * effective_buffer` where effective_buffer = `MAX(variant.buffer_minutes, provider_override.buffer_minutes)`
            - The slot needs `duration + total_buffer` minutes to fit within the free window (unless the slot ends exactly at the window boundary, in which case no trailing buffer is needed within this window)
            - **Trailing hole check**: If `slot.buffer_end` to `window_end` leaves a gap smaller than `min_slot` and greater than zero → don't offer this variant at this start time
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

- [ ] **Design**: Create a `product_booking_windows` table:
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

- [ ] **Admin UI**: Add a booking window matrix to the product detail page (similar to the pricing matrix). Rows = permissions, column = advance_days. Editable inline like price cells.

- [ ] **DB schema**: Create `product_booking_windows` table in `db_schema/`
- [ ] **Table helper**: Create `product_booking_windows.h/cpp` in `sql_util/table_helpers/`
- [ ] **Tests** for table helper
- [ ] **Seed data**: Add default booking windows for seed products in `create_database.cpp`

- [ ] **Tests** in `service_availability_helper_test.cpp`:
  - Free window with no bookings → slots at valid start times (window_start, window_start + min_slot, etc.)
  - Existing booking → free window splits, next slot starts at buffer_end rounded to 5-min
  - **Trailing hole check**: 60min variant rejected when it would leave a 30min gap after, but 90min variant offered
  - **Leading hole check**: Start times like 8:05, 8:10 etc. rejected because they leave an unusable gap at the start of the window. Only 8:00 and 9:05 are valid (window_start and window_start + min_slot)
  - Multiple variants offered at same start time when all fit
  - No variants offered at start time when none fit without creating a hole in either direction
  - Proportional buffer: 120min variant gets double buffer, 60min gets single buffer
  - Provider buffer override: larger override used in MAX calculation
  - Blocked availability excluded
  - Room capacity limit enforced
  - Multiple providers → results grouped by provider
  - Booking window enforced per user permission
  - Past slots excluded
  - Provider with no availability → not shown
  - End-of-window: last slot can skip trailing buffer if it fills exactly to window end

- [ ] **CMakeLists.txt** — Add new files

### 2.2 Backend — KVT Conversion

- [ ] **Add to `scheduling_key_value_table.h/cpp`**:
  - `AvailableSlotToKeyValueTable(const AvailableSlot&)` → `KeyValueTable`
  - `AvailableSlotsToKeyValueTableArray(const std::vector<AvailableSlot>&)` → `KeyValueTableArray`
- [ ] **Tests** for KVT conversion

### 2.3 Backend — Endpoint Layer

- [ ] **Create `GET /api/available_service_slots`** endpoint
  - Query params: `product_id`, `date_from`, `date_to`, `provider_id` (optional), `facility_id` (optional)
  - Requires authentication (for pricing resolution AND booking window determination)
  - Calls `ServiceAvailabilityHelper::ComputeAvailableSlots()`
  - Returns JSON: `{ "slots": [...], "variants": [{ id, name, duration_minutes, currency, amount_cents }] }`
  - Each slot includes variant_id so the frontend knows which duration it represents
- [ ] **Tests** for the endpoint
- [ ] **Register** in `web_app.cpp` and `endpoints/CMakeLists.txt`

---

## Phase 3: Service Booking Flow (Scenarios 32, 33)

**Goal**: User selects a time slot and books a service appointment. Creates a purchase, booking, and bookable_service_session. Sends confirmation email.

### 3.1 Backend — Business Logic Layer

- [ ] **Create `ServiceBookingHelper` class** in `business_logic/scheduling/`
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

- [ ] **Tests** in `service_booking_helper_test.cpp`:
  - Successful booking creates purchase + session + booking
  - Slot no longer available → error
  - Room auto-assigned from available pool
  - Room affinity: provider stays in same room when possible
  - Room affinity: falls back to first available when prior room is full
  - Buffer end calculated correctly with provider override
  - Pricing resolved for variant
  - Booking conflict detection (existing booking at same time for same person)
  - Booking window enforced (user without advance permission can't book too far ahead)

### 3.2 Backend — Email

- [ ] **Create `ServiceBookingConfirmationMail`** in `business_logic/scheduling/`
  - `service_booking_confirmation_mail.h/cpp`
  - Struct: `{ firstName, email, serviceName, variantName, providerName, date, time, duration, facilityName, roomName, amountCents, currency, cancellationPolicyText }`
  - Blue-themed template similar to event booking confirmation but with provider and duration info
- [ ] **Tests** for mail template

### 3.3 Backend — Endpoint Layer

- [ ] **Create `POST /api/book_service`** endpoint
  - Body: `{ product_id, variant_id, provider_person_id, facility_id, start_time_us }`
  - Requires authentication
  - Calls `ServiceBookingHelper::BookService()`
  - Sends confirmation email on success
  - Returns: `{ "purchase": {...}, "booking": {...}, "service_session": {...} }`
  - Same two-step flow: create purchase → frontend handles payment
- [ ] **Tests** for the endpoint
- [ ] **Register** in `web_app.cpp` and `endpoints/CMakeLists.txt`

### 3.4 Backend — KVT Conversion

- [ ] **Add `ServiceSessionToKeyValueTable()`** to scheduling KVT
- [ ] **Tests**

### 3.5 Backend — Extend `my_bookings`

- [ ] **Update `GET /api/my_bookings`** to include service bookings
  - Currently only returns event bookings (joins on `event_session_id`)
  - Add a second query for bookings with `service_session_id`
  - Include provider name, duration, room name in the response
  - Support `type=event|service|all` query param filter

---

## Phase 4: Service Cancellation (Scenario 34)

**Goal**: User cancels a service booking. Refund calculated via the product's cancellation policy (reuses existing RefundHelper). Service session cancelled, freeing the provider time slot and room.

### 4.1 Backend — Business Logic Layer

- [ ] **Extend `BookingHelper::CancelBooking()`** to handle service bookings
  - When booking has `service_session_id` (not `event_session_id`):
    - Load the service session → get product_id, start_time_us
    - Load the product → get cancellation_policy_id
    - Calculate refund via `RefundHelper::CalculateRefundPercent()` (same as events)
    - Process refund via `RefundHelper::ProcessRefund()` (same as events)
    - Set service session status to `cancelled`
    - Freed slot becomes available automatically (availability computation ignores cancelled sessions)
  - No changes needed to `cancel_booking` endpoint — it already calls `BookingHelper::CancelBooking()`
- [ ] **Tests** — Cancel service booking with refund, cancel service booking without payment

### 4.2 Backend — Email

- [ ] **Send cancellation email** — Reuse existing `BookingCancellationMail` with service-specific details (provider, duration)
  - May need to extend the `BookingCancellationData` struct with optional service fields
- [ ] **Tests**

---

## Phase 5: Frontend — Service Browsing & Booking

**Goal**: User can browse available time slots, select a provider and time, and book a service.

### 5.1 Frontend — Types

- [ ] **`scheduling.types.ts`** — Add:
  - `AvailableSlot`, `AvailableSlotsResponse`, `BookServiceRequest`, `BookServiceResponse`

### 5.2 Frontend — ServerAccess Layer

- [ ] **`ServerAccess.ts`** (interface) — Add `getAvailableServiceSlots()` and `bookService()` methods
- [ ] **`ServerAccessNetwork.ts`** — HTTP implementations
- [ ] **`ServerAccess.mock.ts`** — Mock implementations
- [ ] **`ServerAccess.mock.spec.ts`** — Tests

### 5.3 Frontend — Service Catalog Page

- [ ] **New component: `service-catalog`** at `/services`
  - Lists bookable_service products with variant options and prices
  - Each product shows provider photos, names, and link to bio page
  - "Book" button navigates to service booking page

### 5.4 Frontend — Service Booking Page

- [ ] **New component: `service-booking`** at `/shop/service/:productId`
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

- [ ] **Update `my-events.component`** to show service bookings
  - Service bookings show: service name, variant (duration), provider name + photo, date/time, facility/room
  - Cancel button works the same (reuses existing cancel infrastructure with refund policy)

### 5.6 Frontend — Provider Bio Page

- [ ] **New component: `provider-bio`** at `/providers/:personId`
  - Provider photo, name, bio text
  - List of services they offer (provider_type_assignments → products)
  - "Book with [provider name]" button linking to service booking page filtered to this provider

### 5.7 Frontend — Tests

- [ ] **Component spec tests** for service-catalog, service-booking, provider-bio, updated my-events

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
  - `provider_type_id`, `required_room_type_id`, `advance_booking_days`, `booking_cutoff_hours`, `max_time_hole_minutes`

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