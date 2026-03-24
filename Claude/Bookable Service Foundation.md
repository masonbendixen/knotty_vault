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

**Goal**: Build the core algorithm that computes available time slots for a bookable service. This is the most complex piece — it must intersect provider availability, subtract existing bookings + buffers, check room availability, and prevent orphaned time holes.

### 2.1 Backend — Business Logic Layer

- [ ] **Create `ServiceAvailabilityHelper` class** in `business_logic/scheduling/`
  - `service_availability_helper.h/cpp`
  - Dependencies: `DatabaseHelper`

  - **`ComputeAvailableSlots(Transaction&, request)` → `std::vector<AvailableSlot>`**
    - Request: `{ productId, productVariantId, providerPersonId (optional), facilityId, dateFromUs, dateToUs }`
    - Steps:
      1. Load the product variant to get `duration_minutes` and `buffer_minutes`
      2. Load the product to get `required_room_type_id`, `provider_type_id`, `advance_booking_days`, `booking_cutoff_hours`, `max_time_hole_minutes`
      3. If no specific provider requested: find all providers assigned to the product's provider type who are accepting bookings
      4. For each provider:
         a. Load provider availability blocks for the date range (exclude `is_blocked = true`)
         b. Load existing bookable_service_sessions for the provider in the date range
         c. Load provider buffer override for this variant (if any) — effective buffer = `MAX(variant.buffer_minutes, override.buffer_minutes)`
         d. For each availability window, subtract booked sessions (using `buffer_end_us` for gap calculation)
         e. Generate candidate slots at 15-minute intervals within the remaining free windows
         f. **Scenario 66**: Filter out slots that would create orphaned gaps smaller than the smallest active variant's `duration_minutes` for this product
      5. Load room availability: for the product's `required_room_type_id`, find rooms at the facility, check `concurrent_capacity` against overlapping service sessions
      6. Filter slots to only those where a room is available
      7. Apply `advance_booking_days` and `booking_cutoff_hours` time filters
      8. Return list of available slots

  - **`AvailableSlot` struct**:
    ```cpp
    struct AvailableSlot {
        int64_t providerPersonId = 0;
        std::string providerName;
        int64_t facilityId = 0;
        std::string facilityName;
        int64_t startTimeUs = 0;
        int64_t endTimeUs = 0;
        int64_t bufferEndUs = 0;
    };
    ```

- [ ] **Tests** in `service_availability_helper_test.cpp`:
  - Provider with open availability → slots generated at 15-min intervals
  - Provider with existing booking → gap created, slots around it
  - Buffer time respected between consecutive slots
  - Provider buffer override takes precedence when larger
  - Blocked availability window excluded
  - Room capacity limit respected (concurrent_capacity)
  - No orphaned gaps smaller than minimum variant duration (scenario 66)
  - Multiple providers → slots for each
  - advance_booking_days and booking_cutoff_hours filters applied
  - No slots outside provider availability windows
  - No slots in the past

- [ ] **CMakeLists.txt** — Add new files

### 2.2 Backend — KVT Conversion

- [ ] **Add to `scheduling_key_value_table.h/cpp`**:
  - `AvailableSlotToKeyValueTable(const AvailableSlot&)` → `KeyValueTable`
  - `AvailableSlotsToKeyValueTableArray(const std::vector<AvailableSlot>&)` → `KeyValueTableArray`
- [ ] **Tests** for KVT conversion

### 2.3 Backend — Endpoint Layer

- [ ] **Create `GET /api/available_service_slots`** endpoint
  - Query params: `product_id`, `variant_id`, `date_from`, `date_to`, `provider_id` (optional), `facility_id` (optional)
  - Requires authentication (need to resolve pricing for the user)
  - Calls `ServiceAvailabilityHelper::ComputeAvailableSlots()`
  - Resolves pricing via CatalogHelper for the variant
  - Returns JSON: `{ "slots": [...], "variant": { duration_minutes, name }, "currency", "amount_cents" }`
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
      2. Resolve pricing via CatalogHelper (variant-aware)
      3. Create purchase + purchase_item
      4. Find an available room of the required type → auto-assign (first available by ID)
      5. Calculate endTimeUs = startTimeUs + duration_minutes * 60 * 1000000
      6. Calculate bufferEndUs = endTimeUs + effective_buffer_minutes * 60 * 1000000
      7. Create bookable_service_session record
      8. Create booking record (with `service_session_id`, not `event_session_id`)
      9. Return result

- [ ] **Tests** in `service_booking_helper_test.cpp`:
  - Successful booking creates purchase + session + booking
  - Slot no longer available → error
  - Room auto-assigned from available pool
  - Buffer end calculated correctly with provider override
  - Pricing resolved for variant
  - Booking conflict detection (existing booking at same time for same person)

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

4. **Provider selection UX**: When browsing slots, should the user:
   - **Option A**: See all providers' availability merged, assigned a provider at booking time
   - **Option B**: Select a specific provider first, then see their availability
   - **Option C**: See slots grouped by provider, pick a specific slot+provider
   - **Recommendation**: Option C — show slots grouped by provider name. User picks both time and provider. "Any provider" option could be a stretch feature.

5. **Date range for availability queries**: How far ahead should the availability endpoint look?
   - **Recommendation**: Accept `date_from` and `date_to` in the query, let the frontend control the range (typically 1-2 weeks). Default to 7 days if not specified.

6. **Same cancel endpoint for services?**: Should cancelling a service booking use the same `cancel_booking` endpoint as events?
   - **Recommendation**: Yes, same endpoint. `BookingHelper::CancelBooking()` can handle both via `event_session_id` vs `service_session_id`. The cancellation policy comes from the product regardless.

7. **Scenario 66 — time hole enforcement**: The `max_time_hole_minutes` on products prevents orphaned gaps. Should this be:
   - **Option A**: Enforced at slot computation time (don't show slots that would create too-small gaps)
   - **Option B**: Enforced at booking time (reject bookings that create too-small gaps)
   - **Recommendation**: Option A — enforce at slot computation time. Don't show the slot to the user if booking it would create an unusable gap. Better UX than "you can see this slot but can't book it."

8. **How do we handle the case where a provider has no availability entered?** Should they show up in search results with zero slots, or be completely hidden?
   - **Recommendation**: Completely hidden — only providers with at least one non-blocked availability window in the date range should appear in results.

9. **Should the service booking page show the provider's photo?** We have the photo system — should we link provider photos to people records?
   - **Recommendation**: Nice-to-have stretch. For now, show provider name only. Photo integration can come when the provider portal is built.

10. **Timezone handling for availability**: Provider availability is stored as microsecond timestamps. Should the admin enter availability in the facility's timezone? How does the frontend handle display?
    - **Recommendation**: Store as UTC microseconds (consistent with events). Admin enters in facility timezone, frontend converts. The facility record already has a `timezone` field. The existing `formatSessionTimeRange()` utility already handles timezone conversion for display.