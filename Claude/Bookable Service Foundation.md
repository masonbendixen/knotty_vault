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
  - 120min: 9:05-11:05 (buffer ends 11:10)
- If someone books 10:40am, then the 9:05-10:35 window only fits:
  - 60min: 9:05-10:05 ✓ (leaves 30min gap → too small for any variant → **rejected**)
  - 90min: 9:05-10:35 ✓ (no gap, fills exactly → **offered**)
  - So only the 90min variant is offered at 9:05am

**Key rules:**
- Start times are on **5-minute boundaries** (configurable constant `kSlotAlignmentMinutes = 5`)
- Duration and buffer values should be multiples of 5 (validated on input)
- A slot is only offered if booking it would NOT create a gap smaller than the smallest active variant's duration for the product
- A 120min booking always means the buffer applies after the full duration — it's treated as one block, not two 60min blocks. However, the freed time after cancellation can be split into smaller slots.
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
      2. Load all active product variants for this product → get duration_minutes and buffer_minutes for each
      3. Determine the user's booking window based on their permissions (see Phase 2 booking window design below)
      4. If no specific provider requested: find all providers assigned to the product's provider type who are accepting bookings
      5. For each provider:
         a. Load provider availability blocks for the date range (exclude `is_blocked = true`)
         b. Load existing bookable_service_sessions for the provider in the date range (ordered by start_time_us)
         c. Load provider buffer overrides for each variant
         d. Compute **free windows**: subtract booked sessions (using `buffer_end_us`) from availability windows
         e. For each free window, for each variant:
            - Calculate effective_buffer = `MAX(variant.buffer_minutes, provider_override.buffer_minutes)`
            - The slot needs `duration + effective_buffer` minutes to fit (the buffer must fit within the window, or the slot ends at the window boundary with no gap after)
            - Start time = window start, rounded up to nearest 5-minute boundary
            - **Hole check**: If booking this slot would leave a gap after buffer_end that is smaller than the smallest variant's duration → don't offer this slot for this variant (offer a longer variant instead, or don't offer this start time at all)
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

Members get earlier access to service booking. This is controlled by `advance_booking_days` on the product (base window for anyone) plus permission-based overrides.

- [ ] **Design**: Add a `booking_window_overrides` table (or reuse the existing permission infrastructure):
  - Option: Add `booking_advance_days` column to `product_prices` — each permission tier already has a price row, so we can add a booking window column. If the user has a permission that maps to a price row with `booking_advance_days > 0`, they can book that many days ahead. Users without any matching permission use the product's default `advance_booking_days`.
  - The availability endpoint checks: `slot.startTimeUs <= now + user_advance_days * 86400000000`
  - Example: Product has `advance_booking_days = 7` (anyone can book 7 days ahead). Gold members have a product_price row with `booking_advance_days = 30` (they can book 30 days ahead).

- [ ] **Tests** in `service_availability_helper_test.cpp`:
  - Free window with no bookings → slots for all variants at 5-min aligned start times
  - Existing booking → free window splits, slots generated from buffer_end of booking
  - Hole check: 60min variant rejected when it would leave a 30min gap but 90min variant offered
  - Multiple variants offered at same start time when all fit
  - No variants offered at start time when none fit without creating a hole
  - Buffer respected: next slot starts at buffer_end, rounded up to 5-min boundary
  - Provider buffer override: larger override used
  - Blocked availability excluded
  - Room capacity limit enforced
  - Multiple providers → results grouped by provider
  - Booking window enforced per user permission
  - Past slots excluded
  - Provider with no availability → not shown

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
  - "Book" button navigates to service booking page

### 5.4 Frontend — Service Booking Page

- [ ] **New component: `service-booking`** at `/shop/service/:productId`
  - Step 1: Select variant (duration + price)
  - Step 2: Select date (date picker)
  - Step 3: Select time slot (fetches available slots, grouped by provider)
  - Step 4: Confirm & Pay (payment method + book)
  - Shows cancellation policy info
  - Success confirmation with all details

### 5.5 Frontend — My Bookings Integration

- [ ] **Update `my-events.component`** to show service bookings
  - Service bookings show: service name, variant, provider, date/time, facility/room
  - Cancel button works the same

### 5.6 Frontend — Tests

- [ ] **Component spec tests** for service-catalog, service-booking, updated my-events

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

## Open Questions

1. **Product → Provider Type linkage**: The `products` table has no `provider_type_id` column. How does the system know which providers can serve a given bookable_service product? Options:
   - **Option A**: Add `provider_type_id` FK to the `products` table (simplest — one provider type per product)
   - **Option B**: Create a many-to-many `product_provider_types` join table (allows a product to be served by multiple provider types)
   - **Option C**: Infer from product code/name convention (fragile, not recommended)
   - **Recommendation**: Option A — add `provider_type_id` to products. A massage product maps to the "massage" provider type. If we need multi-type products later, we can add the join table.
   - Mason- Let's go with Option A

2. **Room auto-assignment strategy**: When a slot is booked, should the system:
   - **Option A**: Assign the first available room of the required type (simplest)
   - **Option B**: Assign the room that minimizes fragmentation (pack bookings into fewer rooms)
   - **Recommendation**: Option A for now — first available room sorted by ID.
   - Mason- Let's go with option A but let's have logic so that if a provider was in a previous room one the given day, try to keep them in that room if possible.

3. **Availability slot granularity**: Should slots be computed at a fixed interval or at exact availability boundaries?
   - **Option A**: Fixed 15-minute intervals (cleaner UI — slots start at :00, :15, :30, :45)
   - **Option B**: Exact boundaries (maximizes availability but messy start times like 10:47 AM)
   - **Recommendation**: Option A — 15-minute intervals. Standard for salon/spa booking.
   - Mason- I'm on the fence about this one. If the provider wants a buffer between massages, it forces them to 15min. That means that five massages is six hours. I'm thinking it might make sense to make the buffer and start times need to be on 5min boundaries. It avoids really weird start times like 10:47am. I don't know that 2:05pm is that much different than 2:15pm. It would also let a provider specify a 5min buffer which would mean 5:20 for five massages instead of six hours. What do you think? I don't want to deviate too much from industry standard but I also want to enable the providers to have flexibility.

4. **Provider selection UX**: When browsing slots, should the user:
   - **Option A**: See all providers' availability merged, assigned a provider at booking time
   - **Option B**: Select a specific provider first, then see their availability
   - **Option C**: See slots grouped by provider, pick a specific slot+provider
   - **Recommendation**: Option C — show slots grouped by provider name. User picks both time and provider. "Any provider" option could be a stretch feature.
   - Mason- Let's go with Option C

5. **Date range for availability queries**: How far ahead should the availability endpoint look?
   - **Recommendation**: Accept `date_from` and `date_to` in the query, let the frontend control the range (typically 1-2 weeks). Default to 7 days if not specified.
   - Mason- I want to be able to have various permissions have different booking windows that are allowed and then have a no permission window that allows anyone to book that window. I feel like getting first dibs on massage and service booking will be a selling point for various memberships. I do want the client to be able to specify a range they would like to see but there might not be availability in that time based on their permission. Please modify requirements and this document accordingly (including the document upon which this is based).

6. **Same cancel endpoint for services?**: Should cancelling a service booking use the same `cancel_booking` endpoint as events?
   - **Recommendation**: Yes, same endpoint. `BookingHelper::CancelBooking()` can handle both via `event_session_id` vs `service_session_id`. The cancellation policy comes from the product regardless.
   - Mason- I'll go with your recommendation.

7. **Scenario 66 — time hole enforcement**: The `max_time_hole_minutes` on products prevents orphaned gaps. Should this be:
   - **Option A**: Enforced at slot computation time (don't show slots that would create too-small gaps)
   - **Option B**: Enforced at booking time (reject bookings that create too-small gaps)
   - **Recommendation**: Option A — enforce at slot computation time. Don't show the slot to the user if booking it would create an unusable gap. Better UX than "you can see this slot but can't book it."
   - Mason- Option A

8. **How do we handle the case where a provider has no availability entered?** Should they show up in search results with zero slots, or be completely hidden?
   - **Recommendation**: Completely hidden — only providers with at least one non-blocked availability window in the date range should appear in results.
   - Mason- I'll go with the recommendation.

9. **Should the service booking page show the provider's photo?** We have the photo system — should we link provider photos to people records?
   - **Recommendation**: Nice-to-have stretch. For now, show provider name only. Photo integration can come when the provider portal is built.
   - Mason- Yes, I want to show photos. I also want to have bios and a link to their bio.

10. **Timezone handling for availability**: Provider availability is stored as microsecond timestamps. Should the admin enter availability in the facility's timezone? How does the frontend handle display?
    - **Recommendation**: Store as UTC microseconds (consistent with events). Admin enters in facility timezone, frontend converts. The facility record already has a `timezone` field. The existing `formatSessionTimeRange()` utility already handles timezone conversion for display.
    - Mason- I'll go with your recommendation.