---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/2/2026
Version: 0.1
tags:
---
# Overview

This document details and supports the thin slice / MUST have phase of Support for scheduled purchases.md ([[Support for scheduled purchases]]). Please go into plan mode and use this document as your planning document. Do not ask for permission to modify this document, it is your planning document. Do not attempt to work in .claude/plans. Please use the referenced document, Payment Design Document.md ([[Payment Design Document]]), and the code base for context. Please leave this Overview section in tact and do your work in the sections below.

In the Support for schedule purchases document, look for the section Implementation Plans and the MUST HAVE — Intro Workshop End-to-End section in particular. Let's create various phases to implement all of the phases defined in that section. Note that the first "pre-phase" should be modifying the existing payment tables for the changes needed for scheduling.

Please create a set of phases to implement this. Look at the layered architecture of the system. Let's have phases start with the lowest layers of the server and then build to the client. Add tests for anything that could have a unit test. Please be verbose in the implementation plan and have check boxes for the items to accomplish. As we implement phases, please check off these boxes.

# Scheduling Thin Slice Implementation Plan

**Goal**: Implement the MUST HAVE tier end-to-end — admin creates an intro workshop event, schedules sessions, sessions appear publicly, users book and pay, user sees bookings, admin sees attendees.

**Scenarios covered**: 1-4, 7, 9-11, 14, 40, 57-58, 65

## Phase 0: Modify Existing Payment Tables

Changes to `products`, `product_prices`, and `purchase_items` tables that already exist in the codebase. These must land first since all scheduling tables depend on products.

### 0a. Add new columns to `products` table

**File**: `server/knottyyoga_server/src/db_schema/products.h`
- [x] Add column constant `kProductsDefaultCapacity = "default_capacity"` (BIGINT, nullable)
- [x] Add column constant `kProductsDurationMinutes = "duration_minutes"` (BIGINT, nullable)
- [x] Add column constant `kProductsVisibilityPermissionId = "visibility_permission_id"` (BIGINT, nullable FK → permissions)
- [x] Add column constant `kProductsBookingPermissionId = "booking_permission_id"` (BIGINT, nullable FK → permissions)
- [x] Add column constant `kProductsCancellationPolicyId = "cancellation_policy_id"` (BIGINT, nullable FK — deferred until cancellation_policies table exists, add as nullable non-FK for now, convert to FK in Phase 1 after table creation)
- [x] Add column constant `kProductsRequiredRoomTypeId = "required_room_type_id"` (BIGINT, nullable — same deferral as above)
- [x] Add column constant `kProductsAdvanceBookingDays = "advance_booking_days"` (BIGINT, nullable)
- [x] Add column constant `kProductsBookingCutoffHours = "booking_cutoff_hours"` (BIGINT, nullable)
- [x] Add column constant `kProductsReminderHours = "reminder_hours"` (BIGINT, nullable)
- [x] Add column constant `kProductsMaxTimeHoleMinutes = "max_time_hole_minutes"` (BIGINT, nullable)

**File**: `server/knottyyoga_server/src/db_schema/products.cpp`
- [x] Add `AddColumnNullable` calls for each new column in `MakeProductsTable()`
- [x] Add `AddColumnForeignKeyRefNullable` for `visibility_permission_id` → `permissions.id`
- [x] Add `AddColumnForeignKeyRefNullable` for `booking_permission_id` → `permissions.id`
- [x] Note: `cancellation_policy_id` and `required_room_type_id` FKs will be added in Phase 1 after their target tables are created. For now add as nullable columns without FK constraint.

### 0b. Add `product_variant_id` to `product_prices` table

**File**: `server/knottyyoga_server/src/db_schema/product_prices.h`
- [x] Add column constant `kProductPricesProductVariantId = "product_variant_id"` (BIGINT, nullable)

**File**: `server/knottyyoga_server/src/db_schema/product_prices.cpp`
- [x] Add `AddColumnNullable` call for `product_variant_id` in `MakeProductPricesTable()`
- [x] Note: FK to `product_variants.id` will be added in Phase 1 after that table is created. For now, add as nullable column without FK constraint.
- [x] Update the unique constraint from `(product_id, price_schedule_id, permission_id)` to `(product_id, price_schedule_id, permission_id, product_variant_id)` using `AddNamedUniqueConstraint`

### 0c. Add `product_variant_id` to `purchase_items` table

**File**: `server/knottyyoga_server/src/db_schema/purchase_items.h`
- [x] Add column constant `kPurchaseItemsProductVariantId = "product_variant_id"` (BIGINT, nullable)

**File**: `server/knottyyoga_server/src/db_schema/purchase_items.cpp`
- [x] Add `AddColumnNullable` call for `product_variant_id` in `MakePurchaseItemsTable()`
- [x] Note: FK to `product_variants.id` added in Phase 1

### 0d. Add new `product_kind` enum values

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`
- [x] In `PopulateAdminEnums()`, add enum values for `product_kind`: `"event"` and `"bookable_service"` (in addition to existing `"one_time"` and `"recurring"`)

### 0e. Update admin metadata for new product columns

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`
- [x] In `PopulateAdminColumnDataInfo()`, add rows for each new product column with appropriate labels, hints, and input types
- [x] In `PopulateAdminColumnFriendlyNames()`, add friendly names for each new product column

### 0f. Update `make_database_info.cpp`

**File**: `server/knottyyoga_server/src/db_schema/make_database_info.cpp`
- [x] Verify `MakeProductsTable`, `MakeProductPricesTable`, and `MakePurchaseItemsTable` are called (they already should be — the new columns are added inside the existing Make functions)

### 0g. Tests

- [ ] Rebuild and verify existing payment tests still pass (the new nullable columns should not break existing test data creation) *(manual — Linux/Docker build required)*
- [ ] Verify the database_helper application creates the updated tables successfully *(manual — Linux/Docker build required)*

---

## Phase 1: Create All Scheduling Tables + Seed Data

Create ALL 17 new scheduling tables in the db_schema layer, register them in the database helper, update the test helper, and seed initial data. Tables not used in the thin slice are created empty.

### 1a. Create `facilities` table schema

**File**: `server/knottyyoga_server/src/db_schema/facilities.h` (NEW)
- [x] Define constants: `kFacilitiesTable`, `kFacilitiesId`, `kFacilitiesCode`, `kFacilitiesName`, `kFacilitiesAddressLine1`, `kFacilitiesAddressLine2`, `kFacilitiesCity`, `kFacilitiesState`, `kFacilitiesPostalCode`, `kFacilitiesCountry`, `kFacilitiesTimezone`, `kFacilitiesIsActive`, `kFacilitiesCreatedUs`, `kFacilitiesUpdatedUs`
- [x] Declare `MakeFacilitiesTable(DatabaseInfo databaseInfo)`

**File**: `server/knottyyoga_server/src/db_schema/facilities.cpp` (NEW)
- [x] `id` — BIGSERIAL PK
- [x] `code` — VARCHAR unique
- [x] `name` — VARCHAR simple
- [x] `address_line_1` — VARCHAR simple
- [x] `address_line_2` — VARCHAR nullable
- [x] `city` — VARCHAR simple
- [x] `state` — VARCHAR simple
- [x] `postal_code` — VARCHAR simple
- [x] `country` — VARCHAR with default `'USA'`
- [x] `timezone` — VARCHAR with default `'America/Los_Angeles'`
- [x] `is_active` — BOOL default TRUE
- [x] `created_us`, `updated_us` — BIGINT default `now_us()`

### 1b. Create `location_room_types` table schema

**File**: `server/knottyyoga_server/src/db_schema/location_room_types.h` (NEW)
- [x] Define constants: `kLocationRoomTypesTable`, `kLocationRoomTypesId`, `kLocationRoomTypesCode`, `kLocationRoomTypesName`, `kLocationRoomTypesDescription`, `kLocationRoomTypesCreatedUs`, `kLocationRoomTypesUpdatedUs`
- [x] Declare `MakeLocationRoomTypesTable(DatabaseInfo databaseInfo)`

**File**: `server/knottyyoga_server/src/db_schema/location_room_types.cpp` (NEW)
- [x] `id` — BIGSERIAL PK
- [x] `code` — VARCHAR unique
- [x] `name` — VARCHAR simple
- [x] `description` — VARCHAR nullable
- [x] `created_us`, `updated_us` — BIGINT default `now_us()`

### 1c. Create `location_rooms` table schema

**File**: `server/knottyyoga_server/src/db_schema/location_rooms.h` (NEW)
- [x] Define constants: `kLocationRoomsTable`, `kLocationRoomsId`, `kLocationRoomsFacilityId`, `kLocationRoomsRoomTypeId`, `kLocationRoomsName`, `kLocationRoomsDescription`, `kLocationRoomsConcurrentCapacity`, `kLocationRoomsIsActive`, `kLocationRoomsCreatedUs`, `kLocationRoomsUpdatedUs`
- [x] Declare `MakeLocationRoomsTable(DatabaseInfo databaseInfo)`

**File**: `server/knottyyoga_server/src/db_schema/location_rooms.cpp` (NEW)
- [x] `id` — BIGSERIAL PK
- [x] `facility_id` — FK → facilities.id (required)
- [x] `room_type_id` — FK → location_room_types.id (required)
- [x] `name` — VARCHAR simple
- [x] `description` — VARCHAR nullable
- [x] `concurrent_capacity` — BIGINT default 1
- [x] `is_active` — BOOL default TRUE
- [x] `created_us`, `updated_us` — BIGINT default `now_us()`

### 1d. Create `product_variants` table schema

**File**: `server/knottyyoga_server/src/db_schema/product_variants.h` (NEW)
- [x] Define constants: `kProductVariantsTable`, `kProductVariantsId`, `kProductVariantsProductId`, `kProductVariantsCode`, `kProductVariantsName`, `kProductVariantsDurationMinutes`, `kProductVariantsBufferMinutes`, `kProductVariantsSortOrder`, `kProductVariantsIsActive`, `kProductVariantsCreatedUs`, `kProductVariantsUpdatedUs`
- [x] Declare `MakeProductVariantsTable(DatabaseInfo databaseInfo)`

**File**: `server/knottyyoga_server/src/db_schema/product_variants.cpp` (NEW)
- [x] `id` — BIGSERIAL PK
- [x] `product_id` — FK → products.id (required)
- [x] `code` — VARCHAR unique
- [x] `name` — VARCHAR simple
- [x] `duration_minutes` — BIGINT simple
- [x] `buffer_minutes` — BIGINT default 0
- [x] `sort_order` — BIGINT default 0
- [x] `is_active` — BOOL default TRUE
- [x] `created_us`, `updated_us` — BIGINT default `now_us()`

### 1e. Create `provider_types` table schema

**File**: `server/knottyyoga_server/src/db_schema/provider_types.h` (NEW)
- [x] Define constants and declare `MakeProviderTypesTable`

**File**: `server/knottyyoga_server/src/db_schema/provider_types.cpp` (NEW)
- [x] `id` — BIGSERIAL PK, `code` — VARCHAR unique, `name` — VARCHAR simple, `description` — VARCHAR nullable, `created_us`, `updated_us`

### 1f. Create `provider_type_assignments` table schema

**File**: `server/knottyyoga_server/src/db_schema/provider_type_assignments.h` (NEW)
- [x] Define constants and declare `MakeProviderTypeAssignmentsTable`

**File**: `server/knottyyoga_server/src/db_schema/provider_type_assignments.cpp` (NEW)
- [x] `id` — BIGSERIAL PK, `person_id` — FK → people.id, `provider_type_id` — FK → provider_types.id, `is_accepting_bookings` — BOOL default TRUE, `created_us`, `updated_us`
- [x] Unique constraint: `(person_id, provider_type_id)`

### 1g. Create `cancellation_policies` and `cancellation_policy_windows` table schemas

**File**: `server/knottyyoga_server/src/db_schema/cancellation_policies.h` (NEW)
- [x] Define constants and declare `MakeCancellationPoliciesTable`

**File**: `server/knottyyoga_server/src/db_schema/cancellation_policies.cpp` (NEW)
- [x] `id` — BIGSERIAL PK, `name` — VARCHAR simple, `description` — VARCHAR nullable, `created_us`, `updated_us`

**File**: `server/knottyyoga_server/src/db_schema/cancellation_policy_windows.h` (NEW)
- [x] Define constants and declare `MakeCancellationPolicyWindowsTable`

**File**: `server/knottyyoga_server/src/db_schema/cancellation_policy_windows.cpp` (NEW)
- [x] `id` — BIGSERIAL PK, `cancellation_policy_id` — FK → cancellation_policies.id, `hours_before` — BIGINT simple, `refund_percent` — BIGINT simple, `created_us`, `updated_us`

### 1h. Create `event_sessions` table schema

**File**: `server/knottyyoga_server/src/db_schema/event_sessions.h` (NEW)
- [x] Define constants: `kEventSessionsTable`, `kEventSessionsId`, `kEventSessionsProductId`, `kEventSessionsFacilityId`, `kEventSessionsLocationRoomId`, `kEventSessionsStartTimeUs`, `kEventSessionsEndTimeUs`, `kEventSessionsCapacity`, `kEventSessionsBookedCount`, `kEventSessionsStatus`, `kEventSessionsShowOnHomePage`, `kEventSessionsHomePageVisibleFromUs`, `kEventSessionsShowOnUpcoming`, `kEventSessionsUpcomingVisibleFromUs`, `kEventSessionsCancellationReason`, `kEventSessionsNotes`, `kEventSessionsCreatedUs`, `kEventSessionsUpdatedUs`
- [x] Declare `MakeEventSessionsTable(DatabaseInfo databaseInfo)`

**File**: `server/knottyyoga_server/src/db_schema/event_sessions.cpp` (NEW)
- [x] `id` — BIGSERIAL PK
- [x] `product_id` — FK → products.id (required)
- [x] `facility_id` — FK nullable → facilities.id
- [x] `location_room_id` — FK nullable → location_rooms.id
- [x] `start_time_us` — BIGINT simple
- [x] `end_time_us` — BIGINT simple
- [x] `capacity` — BIGINT simple
- [x] `booked_count` — BIGINT default 0
- [x] `status` — VARCHAR simple (default "scheduled")
- [x] `show_on_home_page` — BOOL default FALSE
- [x] `home_page_visible_from_us` — BIGINT nullable
- [x] `show_on_upcoming` — BOOL default FALSE
- [x] `upcoming_visible_from_us` — BIGINT nullable
- [x] `cancellation_reason` — VARCHAR nullable
- [x] `notes` — VARCHAR nullable
- [x] `created_us`, `updated_us` — BIGINT default `now_us()`

### 1i. Create `bookings` table schema

**File**: `server/knottyyoga_server/src/db_schema/bookings.h` (NEW)
- [x] Define constants: `kBookingsTable`, `kBookingsId`, `kBookingsEventSessionId`, `kBookingsServiceSessionId`, `kBookingsPurchaseId`, `kBookingsPurchaseItemId`, `kBookingsPersonId`, `kBookingsProviderPersonId`, `kBookingsStatus`, `kBookingsWaitlistPosition`, `kBookingsCancelledUs`, `kBookingsCheckedInUs`, `kBookingsNotes`, `kBookingsCreatedUs`, `kBookingsUpdatedUs`
- [x] Declare `MakeBookingsTable(DatabaseInfo databaseInfo)`

**File**: `server/knottyyoga_server/src/db_schema/bookings.cpp` (NEW)
- [x] `id` — BIGSERIAL PK
- [x] `event_session_id` — FK nullable → event_sessions.id
- [x] `service_session_id` — FK nullable → bookable_service_sessions.id
- [x] `purchase_id` — FK → purchases.id (required)
- [x] `purchase_item_id` — FK → purchase_items.id (required)
- [x] `person_id` — FK → people.id (required, the attendee)
- [x] `provider_person_id` — FK nullable → people.id (provider for services)
- [x] `status` — VARCHAR simple (default "confirmed")
- [x] `waitlist_position` — BIGINT nullable
- [x] `cancelled_us` — BIGINT nullable
- [x] `checked_in_us` — BIGINT nullable
- [x] `notes` — VARCHAR nullable
- [x] `created_us`, `updated_us` — BIGINT default `now_us()`
- [x] Note: CHECK constraint (exactly one of event_session_id or service_session_id non-null) and partial unique constraints are deferred to business logic enforcement for now

### 1j. Create `bookable_service_sessions` table schema

**File**: `server/knottyyoga_server/src/db_schema/bookable_service_sessions.h` (NEW)
- [x] Define constants and declare `MakeBookableServiceSessionsTable`

**File**: `server/knottyyoga_server/src/db_schema/bookable_service_sessions.cpp` (NEW)
- [x] `id` — BIGSERIAL PK, `product_id` — FK → products.id, `product_variant_id` — FK → product_variants.id, `provider_person_id` — FK → people.id, `facility_id` — FK → facilities.id, `location_room_id` — FK nullable → location_rooms.id, `start_time_us`, `end_time_us`, `buffer_end_us` — BIGINT, `status` — VARCHAR, `cancellation_reason` — VARCHAR nullable, `notes` — VARCHAR nullable, `created_us`, `updated_us`

### 1k. Create remaining scheduling table schemas (used in later tiers)

**File**: `server/knottyyoga_server/src/db_schema/schedule_templates.h/.cpp` (NEW)
- [x] `id`, `provider_person_id` FK, `name`, `effective_from_us`, `effective_to_us` nullable, `is_active` default TRUE, `created_us`, `updated_us`

**File**: `server/knottyyoga_server/src/db_schema/schedule_template_entries.h/.cpp` (NEW)
- [x] `id`, `schedule_template_id` FK, `day_of_week`, `start_time_minutes`, `end_time_minutes`, `created_us`

**File**: `server/knottyyoga_server/src/db_schema/provider_availability.h/.cpp` (NEW)
- [x] `id`, `provider_person_id` FK, `facility_id` FK, `date_us`, `start_time_us`, `end_time_us`, `source`, `schedule_template_id` FK nullable, `is_blocked` default FALSE, `created_us`, `updated_us`

**File**: `server/knottyyoga_server/src/db_schema/time_off_requests.h/.cpp` (NEW)
- [x] `id`, `provider_person_id` FK, `requested_date_us`, `reason` nullable, `status`, `reviewed_by_person_id` FK nullable, `reviewed_us` nullable, `review_notes` nullable, `created_us`, `updated_us`

**File**: `server/knottyyoga_server/src/db_schema/shift_change_requests.h/.cpp` (NEW)
- [x] `id`, `request_type`, `requesting_person_id` FK, `target_person_id` FK, `requesting_availability_id` FK, `target_availability_id` FK nullable, `status`, `target_response_us` nullable, `target_accepted` nullable, `reviewed_by_person_id` FK nullable, `reviewed_us` nullable, `notes` nullable, `created_us`, `updated_us`

**File**: `server/knottyyoga_server/src/db_schema/provider_buffer_overrides.h/.cpp` (NEW)
- [x] `id`, `provider_person_id` FK, `product_variant_id` FK, `buffer_minutes`, `created_us`, `updated_us`
- [x] Unique constraint: `(provider_person_id, product_variant_id)`

### 1l. Wire up FK constraints deferred from Phase 0

Now that `cancellation_policies`, `location_room_types`, and `product_variants` tables exist:

**File**: `server/knottyyoga_server/src/db_schema/products.cpp`
- [x] Add `AddColumnForeignKeyRefNullable` for `cancellation_policy_id` → `cancellation_policies.id`
- [x] Add `AddColumnForeignKeyRefNullable` for `required_room_type_id` → `location_room_types.id`

**File**: `server/knottyyoga_server/src/db_schema/product_prices.cpp`
- [x] Add `AddColumnForeignKeyRefNullable` for `product_variant_id` → `product_variants.id`

**File**: `server/knottyyoga_server/src/db_schema/purchase_items.cpp`
- [x] Add `AddColumnForeignKeyRefNullable` for `product_variant_id` → `product_variants.id`

### 1m. Update CMakeLists.txt for db_schema

**File**: `server/knottyyoga_server/src/db_schema/CMakeLists.txt`
- [x] Add all new `.h` and `.cpp` file pairs for the 17 new tables (facilities, location_room_types, location_rooms, product_variants, provider_types, provider_type_assignments, cancellation_policies, cancellation_policy_windows, event_sessions, bookings, bookable_service_sessions, schedule_templates, schedule_template_entries, provider_availability, time_off_requests, shift_change_requests, provider_buffer_overrides)

### 1n. Update `make_database_info.cpp`

**File**: `server/knottyyoga_server/src/db_schema/make_database_info.cpp`
- [x] Include all new table headers
- [x] Call all 17 `MakeXxxTable(databaseInfo)` functions in dependency order

### 1o. Update `create_database.cpp` — CreateTables

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`
- [x] Include all new table headers
- [x] Add `CreateTable(DbSchema::kXxxTable)` for each new table in dependency order:
  1. `facilities`, `location_room_types`, `cancellation_policies`
  2. `location_rooms` (depends on facilities, room_types)
  3. `product_variants` (depends on products)
  4. `cancellation_policy_windows` (depends on cancellation_policies)
  5. `provider_types`
  6. `provider_type_assignments` (depends on people, provider_types)
  7. `event_sessions` (depends on products, facilities, location_rooms)
  8. `bookable_service_sessions` (depends on products, product_variants, people, facilities, location_rooms)
  9. `bookings` (depends on event_sessions, bookable_service_sessions, purchases, purchase_items, people)
  10. `schedule_templates` (depends on people)
  11. `schedule_template_entries` (depends on schedule_templates)
  12. `provider_availability` (depends on people, facilities, schedule_templates)
  13. `time_off_requests` (depends on people)
  14. `shift_change_requests` (depends on people, provider_availability)
  15. `provider_buffer_overrides` (depends on people, product_variants)

### 1p. Update `create_database.cpp` — PopulateTables (seed data)

**File**: `server/knottyyoga_server/src/database_helper/create_database.cpp`
- [x] Add `PopulateFacilities()` — seed one facility (studio address, timezone `America/Los_Angeles`)
- [x] Add `PopulateLocationRoomTypes()` — seed "studio", "massage_room"
- [x] Add `PopulateLocationRooms()` — seed "Main Gym" (studio type), "Massage Room 1" (massage_room type)
- [x] Add `PopulateProviderTypes()` — seed "instructor", "therapist"
- [x] Add `PopulateCancellationPolicies()` — seed one default policy with windows: (48h → 100%), (24h → 50%), (0h → 0%)
- [x] Update `PopulateAdminTopLevelTables()` — add scheduling tables visible to admin: `facilities`, `location_room_types`, `provider_types`, `cancellation_policies`, `event_sessions`
- [x] Update `PopulateAdminNestedTables()` — add: `location_rooms`, `product_variants`, `provider_type_assignments`, `cancellation_policy_windows`, `bookings`
- [x] Update `PopulateAdminColumnDataInfo()` — add labels/hints for key columns of event_sessions (start_time, capacity, status, etc.)
- [x] Update `PopulateAdminColumnFriendlyNames()` — add friendly names for new table columns
- [x] Update `PopulateAdminTableFriendlyNames()` — add friendly names for new tables
- [x] Update `PopulateAdminTableDisplayTemplates()` — add display templates (e.g., event_sessions: `"{product_id} — {start_time_us}"`)
- [x] Add `PopulateAdminEnums()` entries — add enums for `event_session_status` ("scheduled", "cancelled", "completed"), `booking_status` ("confirmed", "waitlisted", "cancelled", "attended", "no_show")
- [x] Update `PopulateConfigSecrets()` — add entries for `default_studio_timezone`, `event_reminder_hours`

### 1q. Update config secrets

**File**: `server/knottyyoga_server/src/secrets/secret_keys.h`
- [x] Add `kDefaultStudioTimezone = "default_studio_timezone"`
- [x] Add `kEventReminderHours = "event_reminder_hours"`

**File**: `server/knottyyoga_server/src/secrets/secret_values.cpp`
- [x] Add default values: `"America/Los_Angeles"` and `"24"`

### 1r. Update test helper

**File**: `server/knottyyoga_server/test/src/util/payment_table_test_helper.h`
- [x] Add declaration: `void MakeSchedulingTables(Transaction& transaction, TestDatabaseUtil& testDb)`

**File**: `server/knottyyoga_server/test/src/util/payment_table_test_helper.cpp`
- [x] Implement `MakeSchedulingTables()` — creates all 17 new tables in dependency order (called AFTER `MakePaymentTables`)
- [x] Include all new table headers

### 1s. Tests

- [ ] Rebuild and run all existing tests — must still pass
- [ ] Run `knottyyoga_database_helper` to verify all tables create and populate successfully
- [ ] Verify admin portal shows new tables and allows CRUD operations on facilities, event_sessions, etc.

---

## Phase 2: Table Helpers (`event_sessions`, `bookings`)

CRUD table helpers following the `Products`/`Purchases` pattern in `sql_util/table_helpers/`. Simple data access layer with no business logic.

**Key patterns to follow** (from `products.h/cpp`):
- Class with `DatabaseHelper` member in `TableHelpers` namespace
- SQL constants in anonymous namespace (`constexpr std::string_view kSql...`)
- `DbCrud` utilities for Add/Get/Update/Delete
- `allowedSqlKeywords` for SQL literals (`"true"`, `"false"`, `"now_us()"`)
- Pagination via `COUNT(*) OVER()` with `ExtractTotalCount` helper

### 2a. EventSessions table helper

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/event_sessions.h` (NEW)
- [x] Class `EventSessions` with `DatabaseHelper` constructor (following `Products` pattern)
- [x] `int64_t AddEventSession(Transaction&, const KeyValueTable&)` — insert via `DbCrud::AddRowToTableFetchInt64PrimaryKey`
- [x] `KeyValueTable GetEventSession(Transaction&, int64_t id)` — single row by ID via `DbCrud::GetRow`
- [x] `KeyValueTableArray GetEventSessions(Transaction&, int64_t pageSize = 0, int64_t pageOffset = 0, int64_t* totalCount = nullptr)` — paginated list with `COUNT(*) OVER()`
- [x] `void UpdateEventSession(Transaction&, int64_t id, const KeyValueTable& updates)` — update fields via `DbCrud::UpdateRow`
- [x] `void IncrementBookedCount(Transaction&, int64_t id)` — atomic `UPDATE event_sessions SET booked_count = booked_count + 1 WHERE id = $1`
- [x] `void DecrementBookedCount(Transaction&, int64_t id)` — atomic decrement (for cancellations)
- [x] `void DeleteEventSession(Transaction&, int64_t id)` — via `DbCrud::DeleteRow`

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/event_sessions.cpp` (NEW)
- [x] SQL constants in anonymous namespace
- [x] `allowedSqlKeywords` for `"now_us()"`, `"true"`, `"false"`, `"NULL"`
- [x] `ExtractTotalCount` helper (same pattern as `products.cpp`)
- [x] Implement all methods using `DbCrud` utilities and column constants from `db_schema/event_sessions.h`
- [x] `IncrementBookedCount`/`DecrementBookedCount`: raw SQL via `transaction.RunSqlStatementReturningKeyValueTableArray`

### 2b. Bookings table helper

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/bookings.h` (NEW)
- [x] Class `Bookings` with `DatabaseHelper` constructor
- [x] `int64_t AddBooking(Transaction&, const KeyValueTable&)` — insert via `DbCrud::AddRowToTableFetchInt64PrimaryKey`
- [x] `KeyValueTable GetBooking(Transaction&, int64_t id)` — single row by ID
- [x] `KeyValueTableArray GetBookingsBySession(Transaction&, int64_t eventSessionId)` — all bookings for an event session
- [x] `KeyValueTableArray GetBookingsByPerson(Transaction&, int64_t personId)` — all bookings for a person
- [x] `KeyValueTableArray GetBookingsByPurchase(Transaction&, int64_t purchaseId)` — bookings linked to a purchase
- [x] `void UpdateBooking(Transaction&, int64_t id, const KeyValueTable& updates)` — update fields
- [x] `void DeleteBooking(Transaction&, int64_t id)` — delete

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/bookings.cpp` (NEW)
- [x] SQL constants for each query
- [x] `allowedSqlKeywords` for `"now_us()"`, `"true"`, `"false"`, `"NULL"`
- [x] Implement all methods using `DbCrud` utilities and column constants from `db_schema/bookings.h`

### 2c. Update CMakeLists

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/CMakeLists.txt`
- [x] Add `event_sessions.h`, `event_sessions.cpp` to `knotty_yoga_core`
- [x] Add `bookings.h`, `bookings.cpp` to `knotty_yoga_core`
- [x] Add `event_sessions_test.cpp` to `knotty_yoga_tests`
- [x] Add `bookings_test.cpp` to `knotty_yoga_tests`

### 2d. Tests

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/event_sessions_test.cpp` (NEW)
- [x] Test `AddEventSession` creates a session and returns a valid ID
- [x] Test `GetEventSession` retrieves correct data by ID
- [x] Test `GetEventSessions` with pagination returns correct count and subset
- [x] Test `UpdateEventSession` modifies fields correctly
- [x] Test `IncrementBookedCount` atomically increments
- [x] Test `DecrementBookedCount` atomically decrements
- [x] Test `DeleteEventSession` removes the row
- [x] Test `GetEventSession` for non-existent ID returns empty KeyValueTable

**File**: `server/knottyyoga_server/src/sql_util/table_helpers/bookings_test.cpp` (NEW)
- [x] Test `AddBooking` creates a booking and returns a valid ID
- [x] Test `GetBooking` retrieves correct data by ID
- [x] Test `GetBookingsBySession` returns bookings for a specific session
- [x] Test `GetBookingsByPerson` returns bookings for a specific person
- [x] Test `GetBookingsByPurchase` returns bookings linked to a purchase
- [x] Test `UpdateBooking` modifies fields correctly
- [x] Test `DeleteBooking` removes the row
- [x] Test empty results for non-existent session/person/purchase

### 2e. Verification

- [ ] Rebuild and run all tests — existing and new must pass
- [ ] Verify new table helper tests pass

---

## Phase 3: Business Logic — `scheduling/` Directory

Create `scheduling/` as a sibling to `auth/` and `payment/`, containing all scheduling domain logic. This layer uses the table helpers from Phase 2 and the existing `CatalogHelper`/`PurchaseHelper` from `payment/`.

**Key patterns to follow** (from `payment/`):
- Domain structs defined in helper headers (like `PurchaseInfo`, `CatalogItem`)
- Helper classes take `DatabaseHelper` + other helpers in constructor
- `*_key_value_table.h/cpp` for domain struct → KeyValueTable conversion
- `*_mail.h/cpp` for email templates using `FormatString`
- Complex JOIN queries written directly in the helper (like `CatalogHelper` pricing resolution)
- CMakeLists with `knotty_yoga_core` and `knotty_yoga_tests` target sources

### 3a. Create scheduling directory and CMakeLists

**File**: `server/knottyyoga_server/src/scheduling/CMakeLists.txt` (NEW)
- [x] `target_sources(knotty_yoga_core PRIVATE ...)` for all scheduling source files
- [x] `target_sources(knotty_yoga_tests PUBLIC ...)` for all scheduling test files

**File**: `server/knottyyoga_server/src/CMakeLists.txt`
- [x] Add `add_subdirectory(scheduling)`

### 3b. EventSessionHelper — domain structs and session queries

**File**: `server/knottyyoga_server/src/scheduling/event_session_helper.h` (NEW)
- [x] Define `EventSessionInfo` struct: `id`, `productId`, `productName`, `productDescription`, `facilityId`, `facilityName`, `facilityTimezone`, `locationRoomId`, `locationRoomName`, `startTimeUs`, `endTimeUs`, `capacity`, `bookedCount`, `status`, `showOnHomePage`, `homePageVisibleFromUs`, `showOnUpcoming`, `upcomingVisibleFromUs`
- [x] Define `ResolvedEventSession` struct extending with pricing: `currency`, `amountCents`, `priceScheduleId`, `pricingPermissionId`
- [x] Define `AttendeeInfo` struct: `bookingId`, `personId`, `firstName`, `lastName`, `email`, `bookingStatus`, `bookedAtUs`, `checkedInUs`
- [x] Declare `EventSessionHelper` class (constructor with `DatabaseHelper`)
- [x] `std::vector<ResolvedEventSession> GetVisibleEventSessions(Transaction&, std::string_view placement, int64_t personId = 0)`
- [x] `std::optional<ResolvedEventSession> GetEventSession(Transaction&, int64_t sessionId, int64_t personId = 0)`
- [x] `std::vector<AttendeeInfo> GetAttendeesForSession(Transaction&, int64_t sessionId)`

**File**: `server/knottyyoga_server/src/scheduling/event_session_helper.cpp` (NEW)
- [x] `GetVisibleEventSessions()`:
  - SQL JOIN: `event_sessions` + `products` + `facilities` + `location_rooms`
  - Filter: `status = 'scheduled'` AND `start_time_us > now_us()`
  - If `placement = "upcoming"`: filter `show_on_upcoming = true` AND (`upcoming_visible_from_us IS NULL OR upcoming_visible_from_us <= now_us()`)
  - If `placement = "home_page"`: filter `show_on_home_page = true` AND (`home_page_visible_from_us IS NULL OR home_page_visible_from_us <= now_us()`)
  - Resolve pricing per session via `CatalogHelper::GetProduct()` with user's permissions
  - Order by `start_time_us ASC`
- [x] `GetEventSession()`: single session by ID with same joins + pricing resolution; returns `nullopt` if not found
- [x] `GetAttendeesForSession()`: `bookings` JOIN `people` for a session, all booking statuses, ordered by `bookings.created_us ASC`

### 3c. BookingHelper — booking flow and user queries

**File**: `server/knottyyoga_server/src/scheduling/booking_helper.h` (NEW)
- [x] Define `BookEventRequest` struct: `sessionId`, `personId` (attendee), `payerPersonId`
- [x] Define `BookEventResult` struct: `success`, `errorCode`, `errorMessage`, `purchase` (PurchaseInfo), `booking` (BookingInfo)
- [x] Define `BookingInfo` struct: `id`, `eventSessionId`, `purchaseId`, `purchaseItemId`, `personId`, `status`, `createdUs`
- [x] Define `UserBookingInfo` struct extending `BookingInfo` with event details: `eventName`, `eventDescription`, `startTimeUs`, `endTimeUs`, `facilityName`, `facilityTimezone`, `locationRoomName`, `capacity`, `bookedCount`
- [x] Declare `BookingHelper` class (constructor with `DatabaseHelper`)
- [x] `BookEventResult BookEvent(Transaction&, const BookEventRequest&)`
- [x] `std::vector<UserBookingInfo> GetBookingsForPerson(Transaction&, int64_t personId, std::string_view status = "")`

**File**: `server/knottyyoga_server/src/scheduling/booking_helper.cpp` (NEW)
- [x] `BookEvent()`:
  1. Load event session via EventSessions table helper — verify `status = "scheduled"`, `start_time_us > now_us()`
  2. Capacity check: `booked_count < capacity`, return `SOLD_OUT` error if full
  3. Resolve pricing via `CatalogHelper::GetProduct()` with payer's permissions
  4. Create purchase via `PurchaseHelper::CreatePurchase()` (one item: the event product, quantity 1)
  5. Create booking record via Bookings table helper (`event_session_id`, `purchase_id`, `purchase_item_id`, `person_id`, `status = "confirmed"`)
  6. Increment `booked_count` via EventSessions table helper
  7. Return purchase + booking info
  8. Note: payment happens separately via existing `POST /api/purchase_pay_card`
- [x] `GetBookingsForPerson()`:
  - SQL JOIN: `bookings` + `event_sessions` + `products` + `facilities` + `location_rooms`
  - Filter by `person_id`
  - If `status = "upcoming"`: `event_sessions.start_time_us > now_us() AND bookings.status IN ('confirmed', 'waitlisted')`
  - If `status = "past"`: `event_sessions.start_time_us <= now_us() OR bookings.status IN ('cancelled', 'attended', 'no_show')`
  - If empty: return all
  - Order by `start_time_us` (ASC for upcoming, DESC for past)

### 3d. KeyValueTable conversions

**File**: `server/knottyyoga_server/src/scheduling/scheduling_key_value_table.h` (NEW)
- [x] `KeyValueTable EventSessionInfoToKeyValueTable(const ResolvedEventSession&)` — includes `remaining_spots` (capacity - bookedCount)
- [x] `KeyValueTableArray EventSessionsToKeyValueTableArray(const std::vector<ResolvedEventSession>&)`
- [x] `KeyValueTable BookingInfoToKeyValueTable(const BookingInfo&)`
- [x] `KeyValueTable UserBookingInfoToKeyValueTable(const UserBookingInfo&)`
- [x] `KeyValueTableArray UserBookingsToKeyValueTableArray(const std::vector<UserBookingInfo>&)`
- [x] `KeyValueTable AttendeeInfoToKeyValueTable(const AttendeeInfo&)`
- [x] `KeyValueTableArray AttendeesToKeyValueTableArray(const std::vector<AttendeeInfo>&)`

**File**: `server/knottyyoga_server/src/scheduling/scheduling_key_value_table.cpp` (NEW)
- [x] Implement all conversions using `StringFromInt()` for integers, optional fields with `.has_value()` checks (following `payment_key_value_table.cpp` pattern)

### 3e. Booking confirmation email

**File**: `server/knottyyoga_server/src/scheduling/booking_confirmation_mail.h` (NEW)
- [x] Define `BookingConfirmationData` struct: `firstName`, `email`, `eventName`, `eventDate`, `eventTime`, `facilityName`, `locationRoomName`, `amountCents`, `currency`
- [x] Declare `std::string GenerateBookingConfirmationBody(const BookingConfirmationData&)`

**File**: `server/knottyyoga_server/src/scheduling/booking_confirmation_mail.cpp` (NEW)
- [x] HTML email template using `FormatString` with `{placeholder}` syntax (following `payment_confirmation_mail.cpp` pattern)
- [x] Include event details, location, date/time, amount paid

### 3f. Payment integration — products without entitlement rules

**File**: `server/knottyyoga_server/src/payment/payment_helper.cpp`
- [x] In post-payment fulfillment (where entitlements are created after purchase becomes funded): if the product has no `product_entitlement_rules` row, skip entitlement creation gracefully
- [x] This allows event products (which have no entitlement rules) to be purchased without errors

### 3g. Payment integration — send booking confirmation email

**File**: `server/knottyyoga_server/src/payment/payment_helper.cpp`
- [x] After a purchase becomes funded, check if the purchase has a linked booking (query bookings table via Bookings table helper using `GetBookingsByPurchase`)
- [x] If a booking exists, send the booking confirmation email using `GenerateBookingConfirmationBody()`
- [x] This integrates scheduling into the existing payment flow without changing the payment API

### 3h. Tests

**File**: `server/knottyyoga_server/src/scheduling/event_session_helper_test.cpp` (NEW)
- [x] Test `GetVisibleEventSessions` with `placement = "upcoming"` returns scheduled future sessions with `show_on_upcoming = true`
- [x] Test `GetVisibleEventSessions` with `placement = "home_page"` returns sessions with `show_on_home_page = true`
- [ ] Test time window filtering (`upcoming_visible_from_us` respected)
- [x] Test cancelled sessions are excluded
- [x] Test past sessions are excluded
- [x] Test pricing is resolved for the requesting user's permissions
- [x] Test `GetEventSession` returns correct session details with pricing
- [x] Test `GetEventSession` returns nullopt for non-existent session
- [x] Test `GetAttendeesForSession` returns bookings with person details
- [x] Test `GetAttendeesForSession` includes all booking statuses

**File**: `server/knottyyoga_server/src/scheduling/booking_helper_test.cpp` (NEW)
- [x] Test successful booking creates purchase + booking + increments booked_count
- [x] Test booking a sold-out session returns SOLD_OUT error
- [x] Test booking a cancelled session returns error
- [x] Test booking a past session returns error
- [ ] Test pricing is resolved correctly for the attendee's permissions
- [x] Test booked_count is correctly incremented
- [x] Test `GetBookingsForPerson` returns only bookings for the specified person
- [ ] Test upcoming filter returns only future confirmed bookings
- [ ] Test past filter returns past events and cancelled bookings
- [x] Test returns event details (name, location, times) with each booking

**File**: `server/knottyyoga_server/src/scheduling/scheduling_key_value_table_test.cpp` (NEW)
- [x] Test all KVT conversion functions produce correct keys and values
- [x] Test remaining spots calculation (capacity - bookedCount)

**File**: `server/knottyyoga_server/src/scheduling/booking_confirmation_mail_test.cpp` (NEW)
- [x] Test email body contains event name, date, time, location, amount
- [x] Test HTML is well-formed

**File**: `server/knottyyoga_server/src/payment/payment_helper_test.cpp` (EXISTING — add tests)
- [x] Test that paying for a product without `product_entitlement_rules` succeeds without creating entitlements
- [x] Test that paying for a purchase linked to a booking sends booking confirmation email

### 3i. Verification

- [ ] Rebuild and run all tests — existing and new must pass
- [ ] Verify all scheduling business logic tests pass

---

## Phase 4: Scheduling Endpoints (thin HTTP layer)

Thin endpoints that delegate to the business logic in `scheduling/`. Each endpoint only handles HTTP concerns: auth check, request parsing, call helper, KVT → JSON conversion.

**Key patterns to follow** (from `catalog_products.cpp`):
- `HandleGet`/`HandlePost` → `EndpointAuthHelper` → exported function → `RunInTransaction` → business logic → `KeyValueTableToJson`
- `SetupRouting` class inheriting `RoutingBase`
- Register in `web_app.cpp` (include + reference variable)

### 4a. `visible_event_sessions` endpoint

**File**: `server/knottyyoga_server/src/endpoints/visible_event_sessions.h` (NEW)
- [x] Declare `Json::Value GetVisibleEventSessions(EndpointAuthHelper&, const crow::request&, crow::response&)`

**File**: `server/knottyyoga_server/src/endpoints/visible_event_sessions.cpp` (NEW)
- [x] Route: `GET /api/visible_event_sessions`
- [x] Parse query param `placement` (required: `"upcoming"` or `"home_page"`) — return 400 if missing/invalid
- [x] Get `personId` from session (0 if not logged in — public endpoint)
- [x] Call `EventSessionHelper::GetVisibleEventSessions()`
- [x] Convert via `EventSessionsToKeyValueTableArray()` → `KeyValueTableArrayToJson()`
- [x] Return array of sessions

### 4b. `event_session_detail` endpoint

**File**: `server/knottyyoga_server/src/endpoints/event_session_detail.h` (NEW)
- [x] Declare `Json::Value GetEventSessionDetail(EndpointAuthHelper&, const crow::request&, crow::response&, int64_t sessionId)`

**File**: `server/knottyyoga_server/src/endpoints/event_session_detail.cpp` (NEW)
- [x] Route: `GET /api/event_session/<int>`
- [x] Public endpoint (personId from session if logged in, 0 otherwise)
- [x] Call `EventSessionHelper::GetEventSession()`
- [x] Return 404 if not found
- [x] Return session JSON with resolved pricing

### 4c. `book_event` endpoint

**File**: `server/knottyyoga_server/src/endpoints/book_event.h` (NEW)
- [x] Declare `Json::Value PostBookEvent(EndpointAuthHelper&, const crow::request&, crow::response&, int64_t sessionId)`

**File**: `server/knottyyoga_server/src/endpoints/book_event.cpp` (NEW)
- [x] Route: `POST /api/book_event/<int>`
- [x] Requires authentication — return 401 if not logged in
- [x] Parse optional `person_id` from body (for booking on behalf — defaults to logged-in user)
- [x] Call `BookingHelper::BookEvent()`
- [x] Return purchase + booking JSON (frontend uses `purchase_id` to proceed to payment)

### 4d. `my_bookings` endpoint

**File**: `server/knottyyoga_server/src/endpoints/my_bookings.h` (NEW)
- [x] Declare `Json::Value GetMyBookings(EndpointAuthHelper&, const crow::request&, crow::response&)`

**File**: `server/knottyyoga_server/src/endpoints/my_bookings.cpp` (NEW)
- [x] Route: `GET /api/my_bookings`
- [x] Requires authentication — return 401 if not logged in
- [x] Parse optional query param `status` (`"upcoming"` | `"past"`)
- [x] Call `BookingHelper::GetBookingsForPerson()` with logged-in user's personId
- [x] Convert via `UserBookingsToKeyValueTableArray()` → `KeyValueTableArrayToJson()`

### 4e. `admin_event_session_attendees` endpoint

**File**: `server/knottyyoga_server/src/endpoints/admin_event_session_attendees.h` (NEW)
- [x] Declare `Json::Value GetAdminEventSessionAttendees(EndpointAuthHelper&, const crow::request&, crow::response&, int64_t sessionId)`

**File**: `server/knottyyoga_server/src/endpoints/admin_event_session_attendees.cpp` (NEW)
- [x] Route: `GET /api/admin/event_session/<int>/attendees`
- [x] Requires admin role — return 403 if not admin
- [x] Call `EventSessionHelper::GetAttendeesForSession()`
- [x] Return array of attendees with session summary (capacity, booked_count)

### 4f. Register endpoints

**File**: `server/knottyyoga_server/src/endpoints/web_app.cpp`
- [x] Include all 5 new endpoint headers
- [x] Add reference variables in anonymous namespace

### 4g. Update CMakeLists

**File**: `server/knottyyoga_server/src/endpoints/CMakeLists.txt`
- [x] Add all 5 endpoint `.h/.cpp` files to `knotty_yoga_core`
- [x] Add all 5 test files to `knotty_yoga_tests`

### 4h. Tests

**File**: `server/knottyyoga_server/src/endpoints/visible_event_sessions_test.cpp` (NEW)
- [x] Test GET `/api/visible_event_sessions?placement=upcoming` returns 200 with sessions
- [x] Test missing `placement` parameter returns 400
- [x] Test invalid `placement` value returns 400
- [x] Test returns empty array when no matching sessions exist

**File**: `server/knottyyoga_server/src/endpoints/event_session_detail_test.cpp` (NEW)
- [x] Test GET `/api/event_session/<id>` returns 200 with session details
- [x] Test non-existent session returns 404

**File**: `server/knottyyoga_server/src/endpoints/book_event_test.cpp` (NEW)
- [x] Test POST `/api/book_event/<id>` returns 200 with purchase and booking
- [x] Test unauthenticated request returns 401
- [x] Test sold-out session returns 400 with SOLD_OUT error
- [x] Test non-existent session returns 404

**File**: `server/knottyyoga_server/src/endpoints/my_bookings_test.cpp` (NEW)
- [x] Test GET `/api/my_bookings` returns user's bookings
- [x] Test unauthenticated request returns 401
- [x] Test `status=upcoming` filter works
- [x] Test returns empty array for user with no bookings

**File**: `server/knottyyoga_server/src/endpoints/admin_event_session_attendees_test.cpp` (NEW)
- [x] Test returns attendee list for valid session
- [x] Test non-admin user gets 403
- [x] Test non-existent session returns 404

### 4i. Verification

- [ ] Rebuild and run all tests
- [ ] Manual test: create event session via admin portal, hit endpoints with curl/browser

---

## Phase 5: Angular Frontend — Types and ServerAccess

Add TypeScript types and ServerAccess methods for all new scheduling endpoints.

### 5a. Create scheduling types

**File**: `ui/src/app/shared/types/scheduling.types.ts` (NEW)
- [x] Define `EventSession` interface: `id`, `product_id`, `product_name`, `product_description`, `product_kind`, `facility_id`, `facility_name`, `facility_timezone`, `location_room_id`, `location_room_name`, `start_time_us`, `end_time_us`, `capacity`, `booked_count`, `remaining_spots`, `status`, `currency`, `amount_cents`, `price_schedule_id`, `pricing_permission_id`
- [x] Define `Booking` interface: `id`, `event_session_id`, `purchase_id`, `purchase_item_id`, `person_id`, `status`, `created_us`
- [x] Define `UserBooking` interface: extends `Booking` with `event_name`, `event_description`, `start_time_us`, `end_time_us`, `facility_name`, `facility_timezone`, `location_room_name`, `capacity`, `booked_count`
- [x] Define `Attendee` interface: `booking_id`, `person_id`, `first_name`, `last_name`, `email`, `booking_status`, `booked_at_us`, `checked_in_us`
- [x] Define `BookEventResponse` interface: `purchase` (Purchase), `booking` (Booking)
- [x] Define `AdminEventSessionAttendeesResponse` interface: `session_id`, `attendees` (Attendee[])
- [x] Add utility function: `formatSessionTimeRange(startUs, endUs, timezone)`
- [x] Add status types: `EventSessionStatus`, `EventSessionPlacement`, `BookingStatus`

### 5b. Update ServerAccess interface

**File**: `ui/src/app/shared/types/ServerAccess.ts`
- [x] Add `getVisibleEventSessions(placement: EventSessionPlacement): Observable<EventSession[]>`
- [x] Add `getEventSessionDetail(sessionId: number): Observable<EventSession>`
- [x] Add `bookEvent(sessionId: number): Observable<BookEventResponse>`
- [x] Add `getMyBookings(status?: string): Observable<UserBooking[]>`
- [x] Add `getAdminEventSessionAttendees(sessionId: number): Observable<AdminEventSessionAttendeesResponse>`
- [x] Add re-exports for all scheduling types

### 5c. Implement in ServerAccessNetwork

**File**: `ui/src/app/shared/services/network/ServerAccessNetwork.ts`
- [x] Implement `getVisibleEventSessions` — `GET /api/visible_event_sessions?placement={placement}`, unwraps `{sessions: [...]}`
- [x] Implement `getEventSessionDetail` — `GET /api/event_session/{sessionId}`
- [x] Implement `bookEvent` — `POST /api/book_event/{sessionId}`
- [x] Implement `getMyBookings` — `GET /api/my_bookings?status={status}`, unwraps `{bookings: [...]}`
- [x] Implement `getAdminEventSessionAttendees` — `GET /api/admin/event_session/{sessionId}/attendees`

### 5d. Implement in ServerAccessMock

**File**: `ui/src/app/shared/services/network/ServerAccess.mock.ts`
- [x] Add mock event session data (2 upcoming sessions)
- [x] Add mock booking/userBooking state
- [x] Implement all 5 new methods with in-memory state
- [x] Booking mock checks capacity and returns error if sold out
- [x] Auth checks on bookEvent, getMyBookings, getAdminEventSessionAttendees

### 5e. Update ServerAccessProxy

**File**: `ui/src/app/shared/services/network/ServerAccess.ts`
- [x] Add proxy passthrough for all 5 new methods

### 5f. Tests

**File**: `ui/src/app/shared/services/network/ServerAccess.mock.spec.ts` (ADD)
- [x] Test `getVisibleEventSessions` returns future scheduled sessions
- [x] Test `getEventSessionDetail` returns session by ID and 404 for missing
- [x] Test `bookEvent` creates a booking and purchase
- [x] Test `bookEvent` returns 404 for non-existent session
- [x] Test `bookEvent` fails when not logged in
- [x] Test `getMyBookings` returns bookings for the logged-in user
- [x] Test `getMyBookings` returns empty when no bookings exist
- [x] Test `getMyBookings` fails when not logged in
- [x] Test `getAdminEventSessionAttendees` returns attendees after booking
- [x] Test `getAdminEventSessionAttendees` returns 404 for non-existent session
- [x] Test `getAdminEventSessionAttendees` fails when not logged in
- [x] All 454 Angular tests pass (11 new scheduling tests added)

---

## Phase 6: Angular Frontend — Upcoming Events Page

Public-facing page displaying upcoming event sessions.

### 6a. Create the upcoming events page component

**File**: `ui/src/app/pages/public/upcoming-events/upcoming-events.component.ts` (NEW)
- [x] Standalone component with SharedModule
- [x] On init, fetch `serverAccess.getVisibleEventSessions('upcoming')`
- [x] Handle loading/error/empty states
- [x] Store sessions in component state
- [x] `formatPrice()` using `formatCents()` from payment.types
- [x] `formatTimeRange()` using `formatSessionTimeRange()` from scheduling.types

**File**: `ui/src/app/pages/public/upcoming-events/upcoming-events.component.html` (NEW)
- [x] Loading spinner while fetching
- [x] Responsive grid of event cards (1-3 columns via Tailwind grid)
- [x] Each card shows: event name, description, date/time (formatted in facility timezone), location + room, remaining spots / capacity, price
- [x] "Book Now" link on each card → navigates to `/shop/event/{sessionId}`
- [x] "Sold Out" badge when remaining spots = 0
- [x] Empty state: "No upcoming events at this time."
- [x] Error state: "Unable to load events. Please try again later."

**File**: `ui/src/app/pages/public/upcoming-events/upcoming-events.component.scss` (NEW)
- [x] Card styling with visible border (`border: 1px solid #d1d5db`)
- [x] Sold out badge styling (red pill badge)
- [x] Page container layout matching catalog pattern

### 6b. Add route

**File**: `ui/src/app/pages/public/public.routes.ts`
- [x] Add route `/events` → `UpcomingEventsComponent` (public, no auth required)

### 6c. Home page integration

**File**: `ui/src/app/pages/public/home-page/home-page.component.ts`
- [x] Replace mock calendar data (`EXAMPLE_CALENDAR_EVENTS`) with `serverAccess.getVisibleEventSessions('home_page')`
- [x] Show first upcoming event in the events section
- [x] Removed `CalendarEventComponent` dependency
- [x] Added `formatEventTime()` and `formatEventPrice()` methods

**File**: `ui/src/app/pages/public/home-page/home-page.component.html`
- [x] Display next upcoming event with name, date/time, location, price
- [x] "Book Now" link to `/shop/event/{id}`
- [x] "View All Events" link to `/events`
- [x] Loading and empty states

### 6d. Timezone formatting

- [x] Uses `Intl.DateTimeFormat` with facility's IANA timezone string via `formatSessionTimeRange()` (already created in Phase 5 in `scheduling.types.ts`)
- [x] Both upcoming-events and home-page components use the utility function

### 6e. Tests

**File**: `ui/src/app/pages/public/upcoming-events/upcoming-events.component.spec.ts` (NEW)
- [x] Test component creates successfully
- [x] Test displays event cards from mock server access
- [x] Test shows event names, prices, facility, spots remaining
- [x] Test shows "Sold Out" badge when remaining spots = 0
- [x] Test shows "Book Now" for available sessions
- [x] Test "Book Now" links to `/shop/event/{id}`
- [x] Test empty state message when no events
- [x] Test error state on server failure

**File**: `ui/src/app/pages/public/home-page/home-page.component.spec.ts` (UPDATED)
- [x] Updated to remove CalendarEventComponent import, add RouterTestingModule
- [x] All 465 Angular tests pass (11 new tests added)

---

## Phase 7: Angular Frontend — Event Booking Page

The booking page where users select an event and pay.

### 7a. Create event booking component

**File**: `ui/src/app/pages/shop/event-booking/event-booking.component.ts` (NEW)
- [x] Standalone component, route param `sessionId`
- [x] On init, fetch `serverAccess.getEventSessionDetail(sessionId)`
- [x] Handle states: loading → event_loaded → processing → success / error / sold_out / event_not_found
- [x] Two-step flow mirroring existing checkout:
  1. Display event details + price → user clicks "Book and Pay"
  2. Call `serverAccess.bookEvent(sessionId)` to create purchase
  3. Call `squarePayment.tokenizeCard()` to tokenize card
  4. Call `serverAccess.purchasePayCard(purchaseId, { source_id, idempotency_key })` to pay
  5. On success, show confirmation with event details
- [x] If user is not logged in, AuthGuard redirects to login

**File**: `ui/src/app/pages/shop/event-booking/event-booking.component.html` (NEW)
- [x] Event details section: name, description, date/time, location, remaining spots
- [x] Price display using `formatCents()`
- [x] Card form container for Square Web Payments SDK
- [x] "Book and Pay" button
- [x] Processing spinner during payment
- [x] Success state with booking confirmation details
- [x] Error state with user-friendly messages
- [x] "Sold Out" state if no remaining spots

**File**: `ui/src/app/pages/shop/event-booking/event-booking.component.scss` (NEW)
- [x] Styling consistent with existing checkout component

### 7b. Add route

**File**: `ui/src/app/pages/shop/shop.routes.ts` (UPDATED)
- [x] Add route `/shop/event/:sessionId` → `EventBookingComponent` (auth required via AuthGuard)

### 7c. Tests

**File**: `ui/src/app/pages/shop/event-booking/event-booking.component.spec.ts` (NEW)
- [x] Test component loads event session details
- [x] Test "Book and Pay" button triggers booking + payment flow
- [x] Test sold-out event shows appropriate message
- [x] Test event-not-found for invalid/missing session
- [x] Test success state displays confirmation details
- [x] Test error handling for payment failures (tokenization + declined card)
- [x] Test purchase reuse on retry
- [x] Test squarePayment.destroy() on component destroy
- [x] Test button disabled during processing
- [x] All 478 Angular tests pass (13 new tests added)

---

## Phase 8: Angular Frontend — User Portal (My Events)

User's view of their upcoming and past booked events.

### 8a. Create My Events component

**File**: `ui/src/app/pages/account/my-events/my-events.component.ts` (NEW)
- [x] Standalone component with explicit Material module imports
- [x] Fetch `serverAccess.getMyBookings('upcoming')` and `serverAccess.getMyBookings('past')` on init
- [x] Two sections: "Upcoming Events" and "Past Events"
- [x] Status badge helpers (getStatusClass, getStatusLabel) for booking statuses

**File**: `ui/src/app/pages/account/my-events/my-events.component.html` (NEW)
- [x] Upcoming events: cards with event name, date/time (in facility timezone), location, booking status
- [x] Past events: expansion panels showing event details and status (attended, no_show, cancelled)
- [x] Empty state when no bookings exist with link to /events
- [x] Loading and error states
- [x] "Browse upcoming events" link when bookings exist

**File**: `ui/src/app/pages/account/my-events/my-events.component.scss` (NEW)
- [x] Consistent styling with existing purchase history component (max-width 800px, card borders, status badges)

### 8b. Add route

**File**: `ui/src/app/pages/account/account.routes.ts` (UPDATED)
- [x] Add route `events` → `MyEventsComponent`

### 8c. Add navigation links

**File**: `ui/src/app/pages/account/profile/user.component.ts` (UPDATED)
- [x] Added `onEvents()` navigation method
- [x] Added "My Events" dashboard card with event icon

**File**: `ui/src/app/pages/account/profile/user.component.html` (UPDATED)
- [x] Added "My Events" card to dashboard grid

**File**: `ui/src/app/shared/services/header/mockHeaderResponse.ts` (UPDATED)
- [x] Added "My Events" link to user dropdown menu

### 8d. Tests

**File**: `ui/src/app/pages/account/my-events/my-events.component.spec.ts` (NEW)
- [x] Test component creates
- [x] Test loading state
- [x] Test displays upcoming bookings with name, facility, room
- [x] Test displays past bookings
- [x] Test booking status badges
- [x] Test empty state when no bookings
- [x] Test error state on failure
- [x] Test browse more events link
- [x] Test status labels
- [x] Test shows only upcoming section when no past bookings
- [x] Test shows only past section when no upcoming bookings

**File**: `ui/src/app/pages/account/profile/user.component.spec.ts` (UPDATED)
- [x] Updated dashboard cards test (3 → 4 cards)
- [x] Added test for events card navigation
- [x] All 490 Angular tests pass (12 new tests added)

---

## Phase 9: Angular Frontend — Admin Attendee List

Admin view of all attendees for a specific event session.

### 9a. Create admin attendee list component

**File**: `ui/src/app/pages/admin/event-attendees/event-attendees.component.ts` (NEW)
- [x] Standalone component, route param `sessionId`
- [x] Fetch `getEventSessionDetail(sessionId)` + `getAdminEventSessionAttendees(sessionId)` via `forkJoin` on init
- [x] States: loading → loaded / not_found / error
- [x] Status chip helpers (getStatusClass, getStatusLabel) for booking statuses
- [x] Timestamp formatting for booked_at and checked_in

**File**: `ui/src/app/pages/admin/event-attendees/event-attendees.component.html` (NEW)
- [x] Session header card: event name, date/time, location, capacity/booked count
- [x] Attendee count heading
- [x] Attendee table using MatTable: columns for name, email, status, booked_at, checked_in
- [x] Status chips (confirmed = green, cancelled = grey, waitlisted = orange, attended = blue, no_show = red)
- [x] Dash indicator for attendees not yet checked in
- [x] Empty attendees message
- [x] Not-found state for invalid session IDs

**File**: `ui/src/app/pages/admin/event-attendees/event-attendees.component.scss` (NEW)
- [x] Table styling, status chip colors, session header card with border, responsive table wrapper

### 9b. Add route and navigation

**File**: `ui/src/app/pages/admin/admin.routes.ts` (UPDATED)
- [x] Add route `event-session/:sessionId/attendees` → `EventAttendeesComponent` (admin guard inherited from app.routes.ts)

- [ ] (Future) Add a way to navigate from the event_sessions table view to the attendee list (requires custom action support in the generic table-view-control)

### 9c. Tests

**File**: `ui/src/app/pages/admin/event-attendees/event-attendees.component.spec.ts` (NEW)
- [x] Test session header with event name, facility, room, booked count
- [x] Test attendee table with names and emails
- [x] Test attendee count display
- [x] Test status chips render with correct CSS classes
- [x] Test checked-in time displayed for checked-in attendees
- [x] Test dash shown for not-checked-in attendees
- [x] Test not-found state for invalid session ID
- [x] Test not-found state when fetch fails
- [x] Test empty attendees message
- [x] Test status label helper
- [x] All 500 Angular tests pass (10 new tests added)

---

## Phase 10: Polish

Final polish pass for the thin slice.

### 10a. Sold out handling

- [x] "Sold Out" badge on event cards when `remaining_spots = 0` (Phase 6 — upcoming-events.component.html)
- [x] "Book Now" button hidden / "Sold Out" badge shown when capacity reached (Phase 6)
- [x] Event booking page shows `sold_out` state when `remaining_spots <= 0` (Phase 7)
- [x] Server booking endpoint validates capacity (Phase 3/4)

### 10b. Auto-hide past events

- [x] Server endpoint filters `start_time_us > now` for upcoming/home_page placements (Phase 3/4)
- [x] Frontend displays only what server returns — past events automatically excluded

### 10c. Non-logged-in flow

- [x] Upcoming events page is public (in `public.routes.ts`, no auth required) (Phase 6)
- [x] "Book Now" links to `/shop/event/{sessionId}` which has AuthGuard (Phase 6/7)
- [x] **AuthGuard now passes `returnUrl` query param** when redirecting to `/login` (Phase 10)
- [x] **LoginComponent now reads `returnUrl` and redirects there after login** instead of always going to `/` (Phase 10)

**Files modified for returnUrl flow:**

**File**: `ui/src/app/core/guards/auth-guards.ts` (UPDATED)
- [x] AuthGuard passes `returnUrl={encodedUrl}` to `/login` redirect

**File**: `ui/src/app/pages/auth/login/login.component.ts` (UPDATED)
- [x] Reads `returnUrl` from `ActivatedRoute.snapshot.queryParamMap`
- [x] Uses `router.navigateByUrl(returnUrl)` instead of `router.navigate(['/'])`

**File**: `ui/src/app/pages/auth/login/login.component.spec.ts` (UPDATED)
- [x] Refactored to `setup(returnUrl?)` helper with mock ActivatedRoute
- [x] Test: navigates to `/` when no returnUrl
- [x] Test: navigates to returnUrl (`/shop/event/1`) when provided
- [x] Test: auth data correct after login

### 10d. Timezone display consistency

- [x] All `formatSessionTimeRange()` calls pass `facility_timezone` in: upcoming events, home page, event booking, my events, admin attendees
- [x] Timezone parameter flows through to `Intl.DateTimeFormat` for correct display

### 10e. Final test pass

- [x] All 501 Angular tests pass
- [ ] Server tests: `bin/knottyyoga_tests` (requires Linux container)
- [ ] Manual end-to-end test:
  1. Admin creates event product in admin portal
  2. Admin creates event session with dates, capacity, visibility settings
  3. Session appears on upcoming events page
  4. User books event and pays
  5. User receives booking confirmation email
  6. User sees booking in "My Events"
  7. Admin sees attendee in attendee list
  8. Second user tries to book sold-out event and sees "Sold Out"
