---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/13/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Please use the code base and these documents for context:

- [[Payment Design Document]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]

We currently have an admin portal that exposes select tables to the user via Manage Data. See [[Nested item support]] for more information. Although one can kind of navigate products, events, and subscriptions through the raw database tables, it is hardly intuitive.

I'd like a separate page or set of pages to be able to do things based on a new permission, manage_products, that admins have but can be granted to employees other than full admins to do things like create an event, product, or subscription. You should be able to perform operations like:

- Create / edit a price schedule
- Create / edit an event
- Create / edit a product
- Create / edit a subscription
- Bind events, products, and subscriptions to different price schedules and create prices per price schedule and permission. Ad admin should be able to configure if the price for various permissions including whether or not it is available to user's with no permission.
- The admin should be able to perform CRUD operations on permissions
- For subscriptions, the admin should be able to specify various permission that a given subscription grants.
- The admin UI should be able to do CRUD operations for the various locations / facilities / rooms
- The admin UI should be able to define that various events or services need a given room type
- The admin UI should be able to create instantiations of events with a given start / end time, number of seats, facilities, and the other properties of and event through a user friendly UI.
- The UI should be able to enumerate various event instances to see who is signed up, payments for the event, and how much space is in the event. 
- The UI should be entitlement aware and be able to navigate doing CRUD style operations on the entitlements associated for product creation
- The UI should be able to enumerate the various events, products, and subscriptions and see for each instance which entitlements are granted and to whom.

Please start by listing these requirements and help me brainstorm possible other ones and then we can work on design and implementation plan.

# Requirements

## R1: Stated Requirements

From the Overview, organized by domain:

### Products
- R1.1: Create / edit a product
- R1.2: Bind products to different price schedules and create prices per price schedule and permission
- R1.3: Configure whether a product is available to users with no permission (i.e., permission-gated pricing)
- R1.4: The UI should be entitlement aware — CRUD operations on entitlement rules associated with product creation
- R1.5: Enumerate products and see which entitlements are granted and to whom

### Events
- R1.6: Create / edit an event
- R1.7: Create instantiations of events (event sessions) with start/end time, number of seats, facilities, and other properties through a user-friendly UI
- R1.8: Define that various events or services need a given room type
- R1.9: Enumerate event instances to see who is signed up, payments for the event, and how much space is in the event

### Subscriptions
- R1.10: Create / edit a subscription (subscription-type product)
- R1.11: Specify which permission(s) a subscription grants
- R1.12: Enumerate subscriptions and see which entitlements are granted and to whom

### Pricing
- R1.13: Create / edit a price schedule
- R1.14: Create prices per price schedule and permission for products/events/subscriptions

### Infrastructure
- R1.15: New `manage_products` permission — admins have it, can be granted to non-admin employees
- R1.16: CRUD operations on permissions
- R1.17: CRUD operations for locations / facilities / rooms

## R2: Brainstormed Additional Requirements

### Product Management
- R2.1: **Product activation/deactivation toggle** — quickly mark a product as inactive without deleting it, hiding it from the catalog while preserving history
- R2.2: **Product duplication** — clone an existing product with its prices and entitlement rules to create a similar offering quickly
- R2.3: **Product kind filtering** — filter product lists by kind (one_time, subscription, event, bookable_service) since these have different management workflows
- R2.4: **Product variant management** — create/edit product variants (e.g., different durations or group sizes of the same service) as nested items under a product

### Event Management
- R2.5: **Recurring event session creation** — create multiple event sessions at once (e.g., "every Tuesday at 6pm for the next 8 weeks") rather than one at a time
- R2.6: **Event session cancellation** — cancel an event session with notifications to booked attendees
- R2.7: **Event session status management** — mark sessions as completed, cancelled, or back to scheduled
- R2.8: **Attendee status management** — mark attendees as attended/no-show from the event detail view for attendance tracking
- R2.9: **Waitlist management** — view and manage waitlisted bookings, promote from waitlist when cancellations open seats
- R2.10: **Event calendar view** — see upcoming events in a calendar-style layout (weekly/monthly) in addition to a list

### Subscription Management
- R2.11: **Admin subscription creation** — create a subscription on behalf of a customer (e.g., walk-in sign-up)
- R2.12: **Subscription status dashboard** — at-a-glance view of active, past_due, cancelled, and expired subscription counts
- R2.13: **Billing status visibility** — see next billing date, last charge status, grace period info for each subscription

### Pricing
- R2.14: **Price schedule overlap warning** — warn when creating a schedule whose date range overlaps an existing active schedule
- R2.15: **Bulk price creation** — when creating a new price schedule, offer to copy prices from an existing schedule as a starting point
- R2.16: **Price comparison view** — side-by-side view of prices across schedules and permissions for a given product

### Entitlements
- R2.17: **Entitlement search** — search entitlements by person name/email to quickly look up what a customer has access to
- R2.18: **Manual entitlement creation** — admin can create an entitlement for a person without going through the purchase flow (comp/courtesy access)
- R2.19: **Entitlement revocation with reason** — revoke an entitlement with a tracked reason

### Facilities
- R2.20: **Room availability check** — when creating an event session, show which rooms are available at the requested time (no double-booking)
- R2.21: **Facility schedule view** — see all event sessions using a given facility/room over a time range

### Audit & History
- R2.22: **Change log** — track who created/modified products, prices, events (the `created_us`/`updated_us` fields exist but aren't surfaced)
- R2.23: **Purchase/payment history per product** — from a product view, see all purchases and revenue for that product

---

# Current State Analysis

## What the Existing Admin System Provides

The current admin portal is **metadata-driven** — adding entries to database metadata tables automatically generates CRUD UIs:

| Metadata Table | Purpose |
|----------------|---------|
| `admin_top_level_tables` | Tables shown in the admin dropdown menu |
| `admin_nested_tables` | Child tables shown within a parent record |
| `admin_column_data_info` | Per-column form metadata (label, hint, type, required, hidden, readonly) |
| `admin_column_friendly_names` | Display names for columns |
| `admin_table_display_templates` | FK picker display format (e.g., `{first_name} {last_name}`) |
| `admin_enums` | Dropdown values for enum columns |
| `admin_table_friendly_names` | Display name for table |
| `admin_table_descriptions` | Description text for table |

The system already handles:
- Paginated table views with sorting
- FK picker with search/autocomplete and display templates
- Create/edit/delete with type-appropriate form controls (text, bool, date, enum, long-text, FK picker, photo upload)
- Nested table navigation (parent → child relationships via binding stack)
- Photo support for tables that have photos

### Currently Configured Admin Tables

**Top-level** (16): classes, config_secrets, home_page_photos, people, permissions, price_schedules, product_prices, products, roles, facilities, location_room_types, provider_types, cancellation_policies, event_sessions

**Nested** (13): purchases→purchase_items/payments/purchase_payments, people→role_assignments/instructors/provider_type_assignments, facilities→location_rooms, products→product_variants/product_prices, cancellation_policies→cancellation_policy_windows, event_sessions→bookings

### What's Missing from the Metadata System

| Table | Current Status | Gap |
|-------|---------------|-----|
| `product_entitlement_rules` | Not in admin metadata | Admins can't configure entitlement rules (seats, validity, granted permission) |
| `role_permissions` | Not in admin metadata | Admins can't see/edit which permissions belong to which roles |
| `entitlements` | Not in admin metadata | No admin view of active entitlements |
| `entitlement_assignments` | Not in admin metadata | No admin view of seat assignments |
| `subscriptions` | Not in admin metadata | Only visible through custom admin endpoints |
| `subscription_charges` | Not in admin metadata | No visibility into billing history |
| `saved_cards` | Not in admin metadata | No admin visibility into stored payment methods |
| `gift_permissions` | Not in admin metadata | No admin visibility into gift relationships |

## Access Control Architecture

Currently, admin access is binary: `IsAdmin(transaction)` checks if the user has the admin role. All admin tables are either accessible or not based on this single check. There is no concept of partial admin access.

The `EndpointAuthHelper::GetAllowedTables()` method:
1. Starts with public tables (classes, products)
2. If user is admin: adds ALL `admin_top_level_tables` and `admin_nested_tables`
3. Returns combined list

To support `manage_products`, this needs to become permission-aware rather than role-aware.

---

# Design Decisions

## D1: Two-Track Approach — Enhanced Metadata + Custom Pages

**Decision**: Use a two-track approach:

1. **Track A — Metadata enhancements**: Add missing tables to the existing metadata-driven admin CRUD system. This gives immediate, no-frontend-code coverage for straightforward CRUD (permissions, facilities, rooms, price schedules, entitlement rules, role_permissions).

2. **Track B — Custom admin pages**: Build purpose-built pages for workflows that require more than raw table CRUD. These pages compose multiple related tables into unified views and add workflow-specific UX (e.g., creating an event session with room selection and time picker, viewing a product with all its prices and entitlement rules together).

**Rationale**: The metadata system is powerful and already handles 80% of CRUD needs. Building custom UIs for everything would be massive waste. But for complex workflows (event scheduling, product pricing matrix, subscription management), the raw table view is genuinely unintuitive and a custom UI provides dramatically better UX.

## D2: `manage_products` Permission Implementation

**Decision**: Introduce a tiered permission system for admin table access:

1. Add a `manage_products` permission to the permissions table in `create_database.cpp`
2. Add a new metadata table `admin_table_permissions` that maps table names to required permissions
3. Modify `EndpointAuthHelper::GetAllowedTables()` to check per-table permissions:
   - If user is admin → all admin tables (unchanged behavior)
   - If user has `manage_products` → only tables mapped to that permission
   - Tables without a permission mapping require full admin

**Schema for `admin_table_permissions`**:
```sql
CREATE TABLE admin_table_permissions (
    id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    required_permission TEXT NOT NULL,
    UNIQUE(table_name, required_permission)
);
```

**Alternative considered**: Using role-based access instead of permission-based. Rejected because the user specifically wants `manage_products` as a permission that can be granted flexibly to non-admin employees through any role.

## D3: Custom Page Routing

**Decision**: Add custom admin pages as siblings to the existing table CRUD routes under `/admin/`:

```
/admin                          → Admin dashboard (existing)
/admin/tables/:name/...         → Metadata-driven CRUD (existing)
/admin/products                 → Product management dashboard (new)
/admin/products/:id             → Product detail (prices, entitlements, variants)
/admin/events                   → Event management dashboard (new)
/admin/events/new               → Create event session (new)
/admin/events/:sessionId        → Event session detail + attendees (enhance existing)
/admin/subscriptions            → Subscription management dashboard (new)
/admin/subscriptions/:id        → Subscription detail (new)
/admin/entitlements             → Entitlement search/browse (new)
/admin/pricing                  → Pricing overview across products/schedules (new)
```

These custom pages are protected by the `manage_products` permission (or admin).

## D4: Custom Pages Reuse Existing Components

**Decision**: Custom pages should reuse existing controls wherever possible:
- `TableViewControlComponent` for embedded table views (e.g., showing bookings within an event)
- `CompositeRowControlComponent` for inline edit forms
- `FkPickerComponent` for selecting related entities (products, facilities, permissions)
- `SimpleDate`, `SimpleText`, `SimpleBool`, etc. for form fields

Custom pages compose these existing controls with layout and workflow logic, rather than rebuilding form controls from scratch.

## D5: Product Detail as Hub Page

**Decision**: The product detail page (`/admin/products/:id`) serves as the central hub for managing a product's complete configuration:

| Section | Data Source | Operations |
|---------|-------------|------------|
| Product info | `products` | Edit name, code, description, kind, is_active |
| Pricing matrix | `product_prices` | Add/edit/delete prices per schedule × permission |
| Entitlement rules | `product_entitlement_rules` | Edit seats_default, validity_kind, validity_days, grants_permission_id |
| Variants | `product_variants` | CRUD variants |
| Event sessions | `event_sessions` (if kind=event) | List sessions, link to event detail |
| Active subscriptions | `subscriptions` (if kind=subscription) | List active subscriptions |
| Purchase history | `purchase_items` by product_id | View purchases |

This eliminates the need to navigate multiple raw tables to understand a product.

## D6: Event Session Creation Workflow

**Decision**: Event session creation uses a purpose-built form (not raw table CRUD) with:

1. Product selector (FK picker, filtered to kind=event)
2. Date/time pickers for start and end (not raw microsecond timestamps)
3. Facility/room selector with availability check
4. Capacity field (pre-populated from product.default_capacity if set)
5. Visibility toggles (show_on_home_page, show_on_upcoming)
6. Notes field

The form creates an `event_sessions` row. Optionally, a "recurring" mode creates multiple sessions at once (R2.5).

**Date handling**: The existing admin system stores dates as microsecond timestamps (`_us` suffix). The custom event form should present standard date/time pickers and convert to/from microseconds internally using the existing `dateFromUs`/`usFromDate` utility functions.

## D7: Entitlement Visibility Strategy

**Decision**: Rather than building a standalone entitlement management page immediately, surface entitlements contextually:

1. **On product detail**: Show entitlement rule configuration and a count of active entitlements for this product
2. **On event session detail**: Show which attendees have valid entitlements
3. **On subscription detail**: Show current period entitlement and seat assignments
4. **Entitlement search page**: Simple search-by-person to answer "what does this customer have access to?"

This is more useful than a raw `entitlements` table view because entitlements are always understood in context.

---

# Implementation Plan

## Phase 1: Permission & Access Control Foundation

### 1.1 Create `manage_products` Permission

**Backend** (`create_database.cpp`):
- [x] Add `manage_products` to the initial permissions insert
- [x] Add it to the admin role's permissions (so existing admins automatically have it)

**No frontend changes** — the permission is just a database row.

### 1.2 Create `admin_table_permissions` Table

**Backend**:
- [x] Add `admin_table_permissions` to `db_schema/` (new file: `admin_table_permissions.h/.cpp`)
- [x] Add table helper in `sql_util/table_helpers/` (new file: `admin_table_permissions.h/.cpp`)
- [x] Create the table in `create_database.cpp`
- [x] Populate initial mappings — all product/event/subscription-related tables map to `manage_products`:
  ```
  products → manage_products
  product_prices → manage_products
  product_variants → manage_products
  product_entitlement_rules → manage_products
  price_schedules → manage_products
  permissions → manage_products
  event_sessions → manage_products
  bookings → manage_products
  facilities → manage_products
  location_rooms → manage_products
  location_room_types → manage_products
  cancellation_policies → manage_products
  cancellation_policy_windows → manage_products
  entitlements → manage_products
  entitlement_assignments → manage_products
  subscriptions → manage_products
  subscription_charges → manage_products
  ```

### 1.3 Update Access Control Logic

**Backend** (`endpoint_auth_helper.cpp`):
- [x] Modify `GetAllowedTables()`:
  1. Start with public tables (unchanged)
  2. If admin: add all admin tables (unchanged)
  3. **New**: If not admin but has permissions, query `admin_table_permissions` and add tables where the user has the required permission
- This means a user with `manage_products` permission (but not admin) can access product-related tables through the existing metadata CRUD system

**Backend** (`session.h/cpp`):
- [x] Add `GetActiveUserPermissions(transaction)` → returns set of permission names for the logged-in user (if not already available)

**Tests**: Endpoint tests verifying that:
- [x] Admin user can access all tables (unchanged behavior)
- [x] User with `manage_products` can access product tables but not people/roles/secrets
- [x] User with no permissions sees only public tables

### 1.4 Frontend Permission Check

**Frontend**:
- [x] Add `manage_products` to the permission checking in auth/user service
- [x] Custom admin pages check for `manage_products` OR admin permission before rendering
- [x] Admin nav shows "Manage Products" link when user has the permission
- **Important**: Users with only `manage_products` (not admin) do NOT see the "Manage Data" dropdown. That is reserved for full admins. `manage_products` users only see the custom product/event/subscription admin pages built in Phases 3-7. The existing metadata CRUD system (`/admin/tables/...`) remains admin-only.

---

## Phase 2: Metadata Enhancements (Track A)

Add missing tables to the existing metadata-driven admin system. This gives immediate CRUD coverage with zero frontend code.

### 2.1 Add `product_entitlement_rules` to Admin

**Backend** (`create_database.cpp`):
- [x] `admin_nested_tables`: Add as nested under `products`
- [x] `admin_column_data_info` entries:
  - `product_id`: FK picker → products (readonly on edit — rule is 1:1 with product)
  - `grants_permission_id`: FK picker → permissions (nullable — some products don't grant permissions)
  - `seats_default`: number input, default 1
  - `validity_kind`: enum (instant, days_from_activation, calendar_month)
  - `validity_days`: number input (nullable — only relevant for days_from_activation)
- [x] `admin_column_friendly_names`: Product, Granted Permission, Default Seats, Validity Type, Validity Days
- [x] `admin_table_display_templates`: `Product: {product_id}, Permission: {grants_permission_id}`

### 2.2 Add `role_permissions` to Admin

**Backend** (`create_database.cpp`):
- [x] `admin_nested_tables`: Add as nested under `roles`
- [x] `admin_column_data_info`: role_id (FK→roles, readonly on edit), permission_id (FK→permissions)
- [x] `admin_column_friendly_names`: Role, Permission
- This allows admins to see and edit which permissions belong to which roles

### 2.3 Add `entitlements` to Admin

**Backend** (`create_database.cpp`):
- [x] `admin_top_level_tables`: Add `entitlements`
- [x] `admin_column_data_info`:
  - purchase_id (FK→purchases, readonly), purchase_item_id (readonly)
  - product_id (FK→products, readonly)
  - valid_from_us (date, readonly), valid_to_us (date, readonly)
  - seats_total (number), status (enum: active/expired/revoked)
  - revoked_us (date, readonly/hidden), revoked_reason (text, readonly/hidden)
- [x] `admin_nested_tables`: `entitlement_assignments` nested under `entitlements`
- [x] `admin_column_friendly_names` for both tables

### 2.4 Add `subscriptions` to Admin

**Backend** (`create_database.cpp`):
- [x] `admin_top_level_tables`: Add `subscriptions`
- [x] `admin_column_data_info`:
  - person_id (FK→people), product_id (FK→products), saved_card_id (FK→saved_cards, nullable)
  - status (enum), billing_anchor_day (number)
  - current_period_start_us (date, readonly), current_period_end_us (date, readonly)
  - next_billing_us (date, readonly), grace_period_ends_us (date, readonly)
  - cancelled_us (date, readonly), cancel_reason (text, readonly)
- [x] `admin_nested_tables`: `subscription_charges` nested under `subscriptions`
- [x] `admin_table_display_templates`: `{person_id} - {product_id} ({status})`

### 2.5 Add `gift_permissions` to Admin

**Backend** (`create_database.cpp`):
- [x] `admin_top_level_tables`: Add `gift_permissions`
- [x] `admin_column_data_info`: grantor/grantee person IDs (FK→people), status (enum)

### 2.6 Tests for Phase 2

- [x] `get_table_rows_test.cpp`: Verify each newly configured table appears in admin schema and is queryable
- [x] Verify nested table relationships work (product → entitlement_rules, role → role_permissions, etc.)

---

## Phase 3: Product Management Pages (Track B)

### 3.1 Product List Page (`/admin/products`)

**Frontend** — New `ProductListComponent`:

**Layout**:
- [x] Filter bar: kind dropdown (all/one_time/subscription/event/bookable_service), active/inactive toggle
- [x] Product table: name, code, kind, is_active, price (current default schedule), action buttons
- [x] "Create Product" button → inline form or modal
- [x] Each row clickable → routes to `/admin/products/:id`

**Backend** — May reuse existing `GET /api/catalog_products` or `GET /api/get_table_rows/products`. The catalog endpoint already resolves prices, but the admin view might need all products (including inactive). Options:
- Add admin flag to catalog endpoint to include inactive products
- Or use the generic `get_filtered_table_rows/products` and enrich client-side

**Recommendation**: Use `get_filtered_table_rows/products` for the list (it already supports filtering and pagination through the metadata system), then load prices separately. The product list page is essentially a filtered view of the existing `products` admin table with enhanced columns.

### 3.2 Product Detail Page (`/admin/products/:id`)

**Frontend** — New `ProductDetailComponent`:

**Sections**:

1. **Product Info** (top card):
   - [x] Editable fields: name, code, description, kind, is_active, default_capacity, duration_minutes
   - [x] Save button calls `updateItem`
   - Uses existing form controls (SimpleText, SimpleBool, SimpleEnum)

2. **Entitlement Configuration** (card):
   - [x] Shows `product_entitlement_rules` for this product (embedded TableViewControl)
   - [x] Fields: granted permission (FK picker → permissions), seats_default (number), validity_kind (enum), validity_days (number)
   - [x] "Create Rule" if none exists, "Edit" if one does
   - Calls `addItem` / `updateItem` on `product_entitlement_rules` table

3. **Pricing Matrix** (card):
   - [x] Table: rows = price schedules, columns = permissions (+ "no permission" column)
   - [x] Each cell shows the price in cents, editable inline
   - [x] Empty cells mean "not available for this permission/schedule combo"
   - [x] "Add Price" button for adding new cells (click empty cell to add)
   - This is the key view that makes pricing intuitive — currently spread across raw `product_prices` rows

   **Data loading**:
   - GET all `product_prices` filtered by product_id
   - GET all active `price_schedules`
   - GET all `permissions`
   - Build the matrix client-side

4. **Product Variants** (card, if applicable):
   - [x] Embedded `TableViewControlComponent` for `product_variants` filtered to this product
   - Uses existing nested table mechanism

5. **Context-specific section**:
   - [x] If kind=event: "Event Sessions" card showing upcoming sessions, link to create new session
   - [x] If kind=subscription: "Active Subscriptions" card showing count and list
   - [x] If kind=one_time: "Recent Purchases" card showing recent purchase_items for this product

6. **Entitlement Summary** (card):
   - [x] Count of active entitlements for this product
   - [x] List of people who have active entitlements (from entitlement_assignments joined through entitlements)
   - [ ] Links to entitlement search filtered by this product

### 3.3 Product Creation Flow

**Frontend** — New `ProductCreateComponent` (or modal within ProductListComponent):

**Form fields**:
- [x] Name (required), code (optional), description (textarea), kind (enum dropdown)
- [x] is_active (default true)
- [x] For event kind: default_capacity, duration_minutes
- [x] For subscription kind: duration_minutes (billing period)

**After creation**:
- [x] Navigate to product detail page
- [ ] Prompt: "Would you like to configure pricing and entitlements?"
- This naturally leads into the detail page sections

**Backend**: Uses existing `addItemFetchPrimaryKey` on the `products` table.

### 3.4 Product Duplication

**Goal**: Allow admins to clone an existing product with its prices and entitlement rule, then modify the copy. This is especially important for tiered memberships where each tier is additive (Silver → Gold → Platinum).

**Frontend** — "Duplicate" button on the product detail page and product list row actions:
- [x] "Duplicate" button on product detail page creates copy with "(Copy)" suffix and navigates to new product
- [x] Admin edits the name, description, prices, and entitlement configuration before saving
- [x] On save: creates the new product, copies `product_prices` entries (with new product_id), copies `product_entitlement_rules` entry

**Backend** — New endpoint `POST /api/admin/duplicate_product`:
- [x] Input: source product_id
- [x] Creates a new product row (clone of source with name suffixed " (Copy)")
- [x] Copies all `product_prices` rows for the source to the new product
- [x] Copies `product_entitlement_rules` row for the source to the new product
- [x] Returns the new product ID
- [x] All in a single transaction

**Tiered membership workflow**:
1. Create Silver membership with base price, permission, and entitlement rule
2. Duplicate Silver → rename to Gold, adjust price upward, potentially add/modify entitlement permissions
3. Duplicate Gold → rename to Platinum, adjust further
4. Each tier's entitlement rule can grant a different permission (silver_member, gold_member, platinum_member)

---

## Phase 4: Event Management Pages

### 4.1 Event Session List Page (`/admin/events`)

**Frontend** — New `EventListComponent`:

**Layout**:
- [x] Filter bar: date range picker, status dropdown (all/scheduled/cancelled/completed), product filter
- [x] Default view: upcoming scheduled events sorted by start_time
- [x] Table columns: product name, date/time, facility/room, capacity (booked/total), status, actions
- [x] Color-coded capacity: green (<50%), yellow (50-90%), red (>90%), full
- [x] "Create Event Session" button → routes to `/admin/events/new`
- [x] Each row clickable → routes to `/admin/events/:sessionId`

**Backend**: Uses `get_filtered_table_rows/event_sessions` with FK display resolution for product_id and facility_id. The existing metadata system handles pagination and sorting.

### 4.2 Staff Binding to Event Sessions

**Goal**: Associate staff members (instructors, assistants) with event sessions so that scheduling and public-facing information reflect who is running each class.

**New tables**:

1. **`event_session_staffing`** — Links staff to event sessions (many-to-many):
   - `id` (SERIAL, PK)
   - `event_session_id` (FK → `event_sessions`)
   - `person_id` (FK → `people`)
   - `role` (VARCHAR) — e.g., "instructor", "assistant", "substitute" (optional, defaults to "instructor")
   - `notes` (TEXT, nullable) — optional notes for this assignment

2. **`facility_staff`** — Which staff work at which facilities (many-to-many):
   - `id` (SERIAL, PK)
   - `facility_id` (FK → `facilities`)
   - `person_id` (FK → `people`)
   - `is_primary` (BOOL) — whether this is the person's primary facility (for default suggestions)

**Staff eligibility**: Only people with the `instructor` permission are eligible for staff assignments. Add a new `instructor` permission and bind it to the existing `kRoleNameInstructor` role. The `facility_staff` table and staff picker filter to people who have this permission.

**Backend**:
- [x] Add `instructor` permission to initial data in `create_database.cpp` and bind to the instructor role
- [x] Add `event_session_staffing` and `facility_staff` tables to `db_schema/` and `create_database.cpp`
- [x] Table helpers for both new tables
- [x] Admin metadata entries so both tables are CRUD-able and appear as nested items under event sessions / facilities
- [x] Endpoint or FK picker support: `GET /api/admin/facility/<id>/staff` and `GET /api/admin/event_session/<id>/staff` with JOIN queries
- [x] Include staff info in event session detail response (join or nested query)
- [ ] When creating recurring event sessions (4.3), copy staff assignments from the template session to all generated sessions

**Frontend — Network layer and types**:
- [x] TypeScript types for `StaffMember`, `FacilityStaffMember`, and response types in `scheduling.types.ts`
- [x] `ServerAccess` interface methods: `getAdminEventSessionStaff(sessionId)`, `getAdminFacilityStaff(facilityId)`
- [x] `ServerAccessNetwork`, `ServerAccessProxy`, and `ServerAccessMock` implementations
- [x] Mock spec tests for both new methods

**Frontend — Staff section in event session detail**:
- [x] `EventSessionStaffComponent` at `/admin/event-session/:sessionId/staff` showing staff with role chips, notes, and session header
- [x] Component spec tests (10 tests)
- [ ] Staff picker in event creation form: auto-complete dropdown filtered to staff at the selected facility (deferred to 4.3 purpose-built event creation form)
- [ ] Support adding multiple staff to a session (instructor + assistant) (deferred to 4.3)
- [ ] Role selector per staff member (instructor / assistant / substitute) (deferred to 4.3)
- [ ] When facility changes in event creation form, re-filter the staff auto-complete suggestions (deferred to 4.3)
- [x] Staff assignment is optional — sessions can be created without staff (e.g., open gym)

**Staff-centric views**:
- [ ] Admin view: "show all sessions for instructor X" — filtered event list by staff member, useful for scheduling oversight
- [ ] Staff portal: staff member can see their own upcoming sessions they are assigned to (accessible to people with `instructor` permission)

**Deferred**: Staff schedule / availability windows — will be a later phase. For now, staff are assigned to sessions manually without availability conflict checking.

**Resolved decisions**:
- Staff eligibility is gated by the `instructor` permission (bound to the existing instructor role)
- Staff assignment is optional on event sessions
- Recurring session creation copies staff assignments from the template
- Staff schedule / availability is deferred to a future phase
- A staff-centric view (admin + staff portal) is in scope

### 4.3 Event Session Creation Page (`/admin/events/new`)

**Frontend** — New `EventCreateComponent`:

**Form** (purpose-built, not raw table CRUD):
1. - [x] **Product** (required): FK picker filtered to products where kind='event'. Shows product name and description.
2. - [x] **Date & Time**: Date picker + time pickers for start and end. Pre-calculates duration from product's `duration_minutes` if set. End time auto-fills when start is selected.
3. - [x] **Facility & Room**: Two cascading dropdowns — select facility first, then room filters to rooms in that facility. Shows room type for reference.
4. - [x] **Capacity**: Number input. Pre-filled from product's `default_capacity` if set.
5. - [x] **Visibility**: Checkboxes for `show_on_home_page` and `show_on_upcoming`, with numeric fields for `upcoming_visible_days_before` and `home_page_visible_days_before`.
6. - [x] **Notes**: Textarea for internal notes.

- [x] **Conversion**: The form converts human-readable date/time to microsecond timestamps before submitting via `addItemFetchPrimaryKey` on `event_sessions`.

**Recurring mode** (R2.5 — in scope):
- [ ] Toggle "Recurring" reveals additional fields:
  - Recurrence pattern: weekly, biweekly, or custom interval (every N days)
  - Day(s) of week: multi-select (e.g., Monday + Wednesday + Friday)
  - End condition: "Until date" (date picker) or "Number of occurrences" (count)
- [ ] Preview: shows a list of all sessions that will be created with dates/times, allowing the admin to review before confirming
- [ ] On confirm: creates all event_sessions in a single transaction
- [ ] Backend: new endpoint `POST /api/admin/create_recurring_sessions`
  - Input: base session template (product_id, facility_id, room_id, capacity, start_time, end_time, visibility settings) + recurrence config (pattern, days_of_week, end_date_or_count)
  - Generates all session dates from recurrence config
  - Creates multiple `event_sessions` rows in a transaction
  - Returns array of created session IDs
  - Validates room availability for each generated date/time slot before creating (see 4.6)

### 4.4 Event Session Detail Page (`/admin/events/:sessionId`)

**Frontend** — Enhance existing `EventAttendeesComponent` (route: `/admin/event-session/:sessionId/attendees`) or create new `EventDetailComponent`:

**Sections**:

1. **Session Info** (top card):
   - [x] Product name (linked to product detail), date/time, facility/room, capacity, status
   - [x] Edit button for editable fields (capacity, times, visibility, notes)
   - [x] Status actions: Cancel (with reason), Mark Complete
   - Cancellation triggers notification to booked attendees (deferred — R2.6)

2. **Attendees / Bookings** (main card):
   - [x] Table: person name, email, booking status, payment status, entitlement status
   - [ ] For each booking: link to the purchase, link to the person
   - [ ] Status actions per attendee: mark attended, mark no-show, cancel booking
   - [x] Capacity bar: visual progress bar (booked / capacity)

3. **Waitlist**: See Phase 9

4. **Revenue Summary** (card):
   - [ ] Total revenue from this session's bookings (sum of purchase_items.line_total_cents for this session's bookings)
   - [ ] Payment status breakdown (completed/pending/failed)

**Backend**:
- [ ] Enhance `GET /api/event_sessions/:id/attendees` (if exists) or create it to return bookings with person info and payment info
- Or use existing `get_filtered_table_rows/bookings` filtered by event_session_id with FK resolution

### 4.5 Room Type Requirements for Events

**Current state**: `event_sessions` already has `facility_id` and `location_room_id` FK columns. Products have no direct room type requirement.

**Design for R1.8** ("define that events need a given room type"):
- [ ] Add optional `required_room_type_id` FK column to `products` table (for event-type products)
- [ ] When creating an event session, the room picker filters to rooms matching the required type
- [ ] Admin metadata entry for the new column with FK picker → location_room_types

**Alternative (simpler)**: Don't add a column. Instead, use convention — the admin picks an appropriate room when creating the session. The system shows room type in the picker to guide selection. Defer the enforcement to a later phase.

**Recommendation**: Start with the simpler alternative. Add `required_room_type_id` only if admins find they're frequently assigning wrong room types.

### 4.6 Room Availability Checking (Double-Booking Prevention)

**Goal**: Prevent double-booking rooms when creating event sessions. When an admin selects a room and time slot, the system checks for conflicts with existing sessions.

**Backend** — New endpoint or business logic method:
- [ ] `GET /api/admin/room_availability?facility_id=X&room_id=Y&start_us=T1&end_us=T2`
  - Queries `event_sessions` for any non-cancelled sessions that overlap the requested time range in the same room
  - Returns: `{ available: bool, conflicts: EventSession[] }`
  - Overlap check: `existing.start_time_us < requested.end_time_us AND existing.end_time_us > requested.start_time_us`

**Backend validation** — Add conflict check to event session creation:
- [ ] In the `addItem` flow (or in a new business logic helper), validate no room conflict before inserting
- [ ] Return a clear error if a conflict exists: "Room X is already booked for [Event Name] from [time] to [time]"
- [ ] Also enforce this in the recurring session creation endpoint — if any date/time has a conflict, report which ones and don't create any (atomic)

**Frontend integration**:
- [ ] When admin selects a room and sets start/end time, call the availability check endpoint
- [ ] If conflict: show a warning with the conflicting session name and time, disable the submit button
- [ ] In recurring mode: after generating the preview list, check availability for all dates and mark conflicting ones in red
- [ ] Admin can remove conflicting dates from the recurring batch before creating

**Database enforcement** (optional additional safety):
- [ ] Consider a stored procedure or trigger that prevents inserting overlapping sessions in the same room
- This provides a safety net even if the UI check is bypassed
- Could be deferred if the API-level validation is sufficient

---

## Phase 5: Client-Side CRUD Extensions — Defaults and Computed Fields

### Problem Statement

When creating an event session from a product page, the admin must manually enter values that the system already knows:
- **Capacity** must be typed in, even though the product defines `default_capacity`
- **End time** must be entered manually, even though the product defines `duration_minutes` and the end time can be computed from `start_time + duration`

This is tedious and error-prone. Rather than complicating the server with formulas in database metadata, we solve this entirely on the client by extending the generic CRUD table pages. The server stays clean — this is purely a client UI friendliness concern. The caller (e.g., the product detail page) already has all the context it needs and passes it to the CRUD form via Angular route state.

### Design: `CrudFormAssist` via Angular Route State

A single unified object, `CrudFormAssist`, carries both default values and computed field rules. It is passed via Angular's `router.navigate` `state` option, which uses the Navigation API to transfer data without putting it in the URL.

#### Interface

```typescript
interface CrudFormAssist {
  defaults?: Record<string, string>;
  computedDates?: ComputedDateRule[];
}

interface ComputedDateRule {
  source: string;        // Column name of the source date field
  dest: string;          // Column name of the destination date field
  offsetMinutes: number; // Minutes to add to source to compute dest
  autoByDefault: boolean; // Whether auto-compute is on initially
}
```

#### Example — Product detail navigates to new event session

```typescript
const formAssist: CrudFormAssist = {
  defaults: {
    capacity: this.product.default_capacity,  // e.g., '20'
  },
  computedDates: [{
    source: 'start_time_us',
    dest: 'end_time_us',
    offsetMinutes: parseInt(this.product.duration_minutes, 10),  // e.g., 60
    autoByDefault: true,
  }],
};

this.router.navigate(
  ['/admin/tables', 'event_sessions', 'new'],
  {
    queryParams: {
      returnUrl: this.router.url,
      ctx: serializeBindingStack(this.productBindingStack),
    },
    state: { formAssist },
  }
);
```

The CRUD form reads `history.state?.formAssist` on init.

#### 5.1 Default Values Behavior

- On form load in create mode, each key in `defaults` is matched to a form field by column name
- The field is pre-filled with the provided value
- The user can freely change the value — defaults are suggestions, not constraints
- In edit mode, `defaults` is ignored (existing row values take precedence)
- Fields not present in `defaults` behave exactly as they do today

#### 5.2 Computed Date Fields Behavior

- When the form loads, each `ComputedDateRule` is registered
- If `autoByDefault` is true, the destination field shows a toggle/checkbox indicating it is auto-computed (e.g., "Auto from start time + 60 min")
- While auto-compute is on:
  - The destination field is read-only and displays the computed value
  - When the source field changes, the destination updates automatically
  - The toggle/checkbox lets the user switch to manual mode
- When auto-compute is off:
  - The destination field becomes a normal editable date/time input
  - The user can enter any value
  - A toggle lets them switch back to auto-compute mode
- In edit mode, `autoByDefault` is ignored — if the existing dest value matches `source + offset`, auto-compute is on; otherwise it's off (preserving whatever the user previously set)
- Microsecond arithmetic: `dest_us = source_us + (offsetMinutes * 60 * 1_000_000)`

#### Extensibility

The `CrudFormAssist` interface is designed to grow. If we later need other types of computed fields (e.g., numeric calculations, string concatenation), we add sibling arrays like `computedNumbers` rather than building a generic formula engine. Each computed type has its own clear, typed structure.

### Why Client-Side Only

- **The server doesn't need to know about UI convenience features.** Default capacity and duration-based end times are presentation concerns — the server just stores whatever values the client sends.
- **No formulas in database metadata.** Expressing `end_time = start_time + duration * 60 * 1000000` in a metadata table is fragile, hard to debug, and only the client needs it.
- **The caller has the context.** The product detail page already has the product row with `default_capacity` and `duration_minutes`. Passing these via route state is trivial — no need for the CRUD form to fetch parent data or understand parent-child field mappings.
- **Generalizable without over-engineering.** Any page that navigates to CRUD "new" mode can pass a `CrudFormAssist`. If subscriptions or other tables need similar behavior later, they just populate the same interface — no new metadata tables, no server endpoints.

### Why Route State Instead of Query Params

- **Clean URLs.** JSON in query params produces ugly, percent-encoded URLs. Route state keeps the URL clean.
- **No encoding concerns.** Route state is passed as a typed JavaScript object — no JSON serialization/parsing, no URL-safety issues.
- **Unified object.** A single `formAssist` state property carries both defaults and computed rules, rather than splitting across multiple query params.
- **Acceptable trade-off.** Route state is lost on page refresh, but this is fine — nobody bookmarks or shares a link to a CRUD create form. The user always navigates there from a parent page.

### Where This Lives in the Codebase

The CRUD form logic is in the admin table entry form components. Changes are isolated to:
- **Route state reading**: Read `history.state?.formAssist` on form init
- **Default application**: Apply `defaults` to form controls before the user interacts
- **Computed field wiring**: Subscribe to source field value changes, compute dest values, manage auto/manual toggle state
- **UI for computed fields**: Toggle or checkbox per computed field to switch between auto and manual mode
- **Type definition**: `CrudFormAssist` and `ComputedDateRule` interfaces in a shared types file

### Implementation Plan

**5.1 Default Values**:
- [x] Define `CrudFormAssist` and `ComputedDateRule` interfaces in shared types
- [x] Read `formAssist` from route state in the CRUD form component on init
- [x] In create mode, apply each `defaults` entry to the matching form control
- [x] Update product detail's `onCreateEventSession()` to pass `formAssist` with `defaults.capacity`
- [x] Tests: defaults applied in create mode, user can override

**5.2 Computed Date Fields**:
- [x] For each `computedDates` rule: subscribe to source field changes, auto-compute dest value when auto mode is on
- [x] Add toggle UI per computed field (auto-compute on/off)
- [x] Handle microsecond timestamp arithmetic: `dest_us = source_us + (offsetMinutes * 60 * 1_000_000)`
- [ ] In edit mode: infer auto/manual state from whether existing dest matches computed value
- [x] Update product detail's `onCreateEventSession()` to include `computedDates` with duration rule
- [x] Tests: auto-compute on source change, toggle to manual, toggle back to auto

**5.3 Wire Up Event Session Creation**:
- [x] Product detail page passes `formAssist` with both `defaults` and `computedDates` via route state
- [x] Event list page's "Create Event Session" button — no `formAssist` needed (no product context)
- [ ] Verify the full flow: product page → CRUD new → capacity pre-filled, end time auto-computed → save → return to product page

---

## Phase 6: Subscription Management Pages

### 6.1 Subscription List Page (`/admin/subscriptions`)

**Frontend** — New `SubscriptionListComponent`:

**Layout**:
- [ ] Status summary cards at top: Active (count), Past Due (count), Cancelled (count), Expired (count)
- [ ] Filter bar: status dropdown, person search, product filter
- [ ] Table: person name, product name, status, current period, next billing date, saved card last4
- [ ] Status indicators: green=active, yellow=past_due, red=expired, grey=cancelled
- [ ] Each row clickable → `/admin/subscriptions/:id`

**Backend**:
- [ ] Existing `GET /api/admin/subscriptions` returns all subscriptions — reuse or enhance
- [ ] Ensure response includes person name and product name (not just IDs)
- Or use `get_filtered_table_rows/subscriptions` with FK resolution

### 6.2 Subscription Detail Page (`/admin/subscriptions/:id`)

**Frontend** — New `SubscriptionDetailComponent`:

**Sections**:

1. **Subscription Info** (top card):
   - [ ] Person (linked), product (linked to product detail), status, billing anchor day
   - [ ] Current period: start → end dates
   - [ ] Next billing date, grace period end (if past_due)
   - [ ] Saved card: brand, last4, expiration
   - [ ] Cancel reason (if cancelled)

2. **Actions** (card):
   - [ ] Change product (upgrade/downgrade) — calls existing `POST /api/subscriptions/:id/change_product`
   - [ ] Cancel subscription — calls existing `POST /api/subscriptions/:id/cancel`
   - [ ] Retry billing (if past_due) — calls existing `POST /api/subscriptions/:id/retry_billing`
   - [ ] Expire grace period (admin override)

3. **Current Entitlement** (card):
   - [ ] Shows the entitlement for the current billing period
   - [ ] Seat assignments with `SeatAssignmentComponent` (reuse from customer portal)
   - [ ] Granted permission name

4. **Billing History** (card):
   - [ ] Embedded `TableViewControlComponent` for `subscription_charges` filtered by subscription_id
   - [ ] Shows each billing cycle: period dates, status (completed/failed), charge amount, payment link

5. **Entitlement History** (card):
   - [ ] All entitlements created for this subscription across billing cycles
   - [ ] Status, validity period, seats used/total

### 6.3 Admin Subscription Creation

**Goal**: Admins can create subscriptions on behalf of customers — a common use case during new member intro workshops.

**Use case**: Admin signs up a new member, sets up their subscription to start billing next month, and gives them the rest of the current month free.

**Frontend** — "Create Subscription" button on subscription list page, opens a creation form:

1. - [ ] **Person selector**: FK picker to select the customer (search by name/email)
2. - [ ] **Product selector**: FK picker filtered to subscription-type products
3. - [ ] **Saved card**:
   - If the person already has saved cards: dropdown to select one
   - If no saved cards: option to add a card using the Square Web Payments SDK card form (same as the customer-facing card setup)
   - This sets the `saved_card_id` on the subscription for automatic billing
4. - [ ] **Start date**: Date picker. Two options:
   - "Start billing next month" — subscription starts on the 1st of next month, but entitlement starts immediately (rest of current month is free)
   - "Start billing now" — charges immediately for the current period
5. - [ ] **Charge now toggle**: Whether to charge the first billing cycle immediately

**Backend**:
- [ ] Reuse existing `POST /api/subscriptions` endpoint with admin authorization or create `POST /api/admin/subscriptions`
  - Takes person_id, product_id, saved_card_id, start_date, charge_now
  - Validates admin or manage_products permission
  - Calls `SubscriptionHelper::CreateSubscription()` with `created_by_person_id` set to the admin
  - If "rest of month free": creates an entitlement valid from now to end of month with no charge, sets `next_billing_us` to the 1st of next month

**Card setup for new customers**:
- [ ] New endpoint or enhance existing: `POST /api/admin/save_card` that takes person_id and card nonce
- [ ] Creates a Square customer for the person (if not exists), attaches the card, saves to `saved_cards`
- [ ] Reuses existing card-saving business logic from the customer portal

### 6.4 Subscription Permission Configuration

**For R1.11** ("specify which permission a subscription grants"):
- This is configured through `product_entitlement_rules.grants_permission_id` on the subscription's product
- No separate subscription-level configuration needed — the product's entitlement rule defines what permission is granted
- The product detail page (Phase 3) already surfaces this configuration
- For subscriptions with multi-seat, `seats_default` in the entitlement rule controls how many seats each billing cycle creates

---

## Phase 7: Pricing Management Page

### 7.1 Pricing Overview Page (`/admin/pricing`)

**Frontend** — New `PricingOverviewComponent`:

**Layout**:
- [ ] Tabs or toggle: by Product | by Price Schedule
- [ ] **By Product**: Select a product → shows pricing matrix (schedules × permissions) — same as product detail pricing section but standalone
- [ ] **By Price Schedule**: Select a schedule → shows all products priced in that schedule with their prices per permission

**Price Schedule Management** (sub-section):
- [ ] List of all price schedules with active/inactive status, date range
- [ ] Create new schedule with name, valid_from, valid_to, is_active
- [ ] Edit existing schedule
- [ ] When creating a new schedule: option to "Copy prices from schedule X" (R2.15)

This page is primarily a navigation aid — it helps admins see the big picture of pricing across the system. The actual price editing happens through the product detail page's pricing matrix.

---

## Phase 8: Entitlement Management Page

### 8.1 Entitlement Search Page (`/admin/entitlements`)

**Frontend** — New `EntitlementSearchComponent`:

**Layout**:
- [ ] Search bar: search by person name or email
- [ ] Filter: by status (active/expired/revoked), by product, by date range
- [ ] Results table: person name, product name, status, valid from/to, seats (used/total), source (purchase link)
- [ ] Each row expandable: shows seat assignments inline

**Backend**:
- [ ] New endpoint: `GET /api/admin/entitlements?person_id=X&product_id=Y&status=Z`
  - Returns entitlements with joined person info (via entitlement_assignments), product info, and assignment details
  - Paginated
- Or leverage existing `get_filtered_table_rows/entitlements` with FK resolution for a simpler initial implementation

**Manual entitlement creation** (R2.18 — deferred):
- [ ] "Create Entitlement" button → form with product picker, person picker, validity dates, seats
- [ ] Calls a new admin endpoint that creates an entitlement without a purchase
- This is a comp/courtesy mechanism — track separately from purchases

---

## Phase 9: Waitlist Management

Extracted from Phase 4.4 — waitlist functionality is a distinct feature with its own UI, backend, and notification concerns.

### 9.1 Waitlist Display

**Frontend** — Add a waitlist card to the Event Session Detail Page:
- [ ] Show waitlisted bookings sorted by creation time (first come, first serve)
- [ ] Display: person name, email, booking time, position in queue
- [ ] Only visible when there are waitlisted bookings for the session

### 9.2 Waitlist Promotion

- [ ] "Promote to confirmed" action per waitlisted booking when seats are available
- [ ] Auto-suggest promotion when a confirmed booking is cancelled and waitlisted bookings exist
- [ ] Notification to the promoted person (email) — ties into R2.6 notification system

### 9.3 Waitlist Backend

- [ ] Endpoint or business logic to promote a waitlisted booking to confirmed status
- [ ] Validation: only promote if capacity allows
- [ ] Update booking status, create/update entitlement if needed
- [ ] Tests: waitlist promotion, capacity validation, ordering

---

## Phase 10: Facilities & Infrastructure Pages

### 10.1 Facilities Already Configured

The existing metadata system already has:
- `facilities` as a top-level admin table
- `location_rooms` nested under facilities
- `location_room_types` as a top-level table

These are already CRUD-able through the existing admin portal. No additional work needed unless we want enhanced UIs.

### 10.2 Facility Schedule View (R2.21 — deferred)

A calendar view showing all event sessions at a facility. Deferred until there's demand — admins can filter the event list by facility for now.

---

## Implementation Phases & Dependencies

| Phase | Description | Dependencies | Effort | Priority |
|-------|-------------|--------------|--------|----------|
| 1 | Permission & access control | None | Medium | **Must Have** |
| 2 | Metadata enhancements | Phase 1 (for permission gating) | Small | **Must Have** |
| 3 | Product management pages + duplication | Phase 2 (for entitlement rules metadata) | Large | **Must Have** |
| 4 | Event management pages + staff binding + recurring + room availability | Phase 3 (product detail links) | Large | **Must Have** |
| 5 | Client-side CRUD extensions — defaults and computed fields via route state | Phase 4 (event session workflow) | Medium | **Should Have** |
| 6 | Subscription management pages + admin creation | Phase 2 (metadata for subscriptions) | Large | **Must Have** |
| 7 | Pricing overview page | Phase 3 (product detail pricing) | Small | **Nice to Have** |
| 8 | Entitlement management page | Phase 2 (metadata for entitlements) | Medium | **Should Have** |
| 9 | Waitlist management — display, promotion, notifications | Phase 4 (event detail page) | Medium | **Should Have** |
| 10 | Facilities enhancements | Phase 2 | Small | **Deferred** |

**Critical path**: Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5

Phases 6, 7, 8 can be done in parallel after Phase 2. Phase 6 is now **Must Have** due to admin subscription creation being a key workflow for new member onboarding.

---

## Implementation Checklist

### Phase 1 — Permission & Access Control
- [ ] Add `manage_products` permission to initial data in `create_database.cpp`
- [ ] Assign `manage_products` to admin role in initial role_permissions data
- [ ] Create `admin_table_permissions` schema in `db_schema/`
- [ ] Create `admin_table_permissions` table helper
- [ ] Create table and populate initial mappings in `create_database.cpp`
- [ ] Modify `EndpointAuthHelper::GetAllowedTables()` to check `admin_table_permissions`
- [ ] Add `GetActiveUserPermissions()` to Session if not already available
- [ ] Tests: endpoint access control with manage_products permission
- [ ] Frontend: surface `manage_products` in user/auth service
- [ ] Frontend: add "Manage Products" nav link gated by permission

### Phase 2 — Metadata Enhancements
- [ ] Add `product_entitlement_rules` metadata (nested under products)
- [ ] Add `role_permissions` metadata (nested under roles)
- [ ] Add `entitlements` as top-level admin table with metadata
- [ ] Add `entitlement_assignments` metadata (nested under entitlements)
- [ ] Add `subscriptions` as top-level admin table with metadata
- [ ] Add `subscription_charges` metadata (nested under subscriptions)
- [ ] Add `gift_permissions` as top-level admin table with metadata
- [ ] Tests: verify each table appears in schema, is CRUD-able via endpoints

### Phase 3 — Product Management Pages
- [x] Create `ProductListComponent` with filtering and product table
- [x] Create `ProductDetailComponent` with info, entitlement, pricing, variants sections (embedded TableViewControls)
- [ ] Implement pricing matrix view in product detail (currently using embedded TableViewControl)
- [x] Create product creation form/flow
- [x] Implement product duplication endpoint (`POST /api/admin/duplicate_product`) with `ProductHelper` business logic layer — copies product, prices, and entitlement rules
- [x] Wire `duplicateProduct` across all ServerAccess layers (interface, network, mock, proxy) and update `ProductDetailComponent` to use backend endpoint
- [x] Add "Duplicate" button to product detail
- [x] Add product manage routes to `manage.routes.ts`
- [x] Wire product list to manage navigation (dashboard card)
- [x] Tests: ProductListComponent, ProductCreateComponent, ProductDetailComponent specs
- [x] Tests: ProductHelper (5 tests), AdminDuplicateProduct endpoint (3 tests), duplicateProduct mock spec (3 tests)

#### Price Schedule Management (first-class entity)
- [x] Create `ScheduleListComponent` with active/inactive toggle filter
- [x] Create `ScheduleCreateComponent` with name, validFrom, validTo, isActive form
- [x] Create `ScheduleDetailComponent` with edit form, duplicate button, embedded product_prices TableViewControl
- [x] Add schedule routes to `manage.routes.ts` (`/manage/schedules`, `/manage/schedules/new`, `/manage/schedules/:id`)
- [x] Add "Price Schedules" card to manage dashboard
- [x] Tests: ScheduleListComponent (12 tests), ScheduleCreateComponent (11 tests), ScheduleDetailComponent (16 tests)

### Phase 4 — Event Management Pages
- [ ] Create `EventListComponent` with filtering and capacity display
- [ ] Add `event_session_staffing` and `facility_staff` tables to `db_schema/` and `create_database.cpp`
- [ ] Create table helpers for `event_session_staffing` and `facility_staff`
- [ ] Add admin metadata for both staffing tables (nested under event sessions / facilities)
- [ ] Implement staff auto-complete picker filtered by facility
- [ ] Add staff section to event session creation and detail pages
- [ ] Create `EventCreateComponent` with user-friendly form (single session)
- [ ] Implement recurring event creation UI (recurrence pattern, preview, batch create)
- [ ] Create `POST /api/admin/create_recurring_sessions` backend endpoint
- [ ] Implement room availability checking endpoint (`GET /api/admin/room_availability`)
- [ ] Add room conflict validation to event session creation (single and recurring)
- [ ] Integrate availability check into event creation form (real-time conflict warnings)
- [ ] Enhance event session detail page with attendees, status management, revenue
- [ ] Add booking status management (attended/no-show)
- [ ] Add event admin routes
- [ ] Tests: staffing table helpers, facility staff lookup, EventListComponent, EventCreateComponent, EventDetailComponent, recurring creation, room availability specs

### Phase 5 — Client-Side CRUD Extensions
- [ ] Define `CrudFormAssist` and `ComputedDateRule` interfaces in shared types
- [ ] Read `formAssist` from route state in CRUD form component, apply `defaults` in create mode
- [ ] Wire up `computedDates` rules with auto-compute logic and toggle UI
- [ ] Handle microsecond timestamp arithmetic for computed dates
- [ ] Update product detail's `onCreateEventSession()` to pass `formAssist` via route state
- [ ] Tests: default application, computed field auto-update, manual override toggle, edit mode behavior

### Phase 6 — Subscription Management Pages
- [ ] Create `SubscriptionListComponent` with status dashboard
- [ ] Create `SubscriptionDetailComponent` with billing history and entitlement
- [ ] Integrate admin subscription actions (cancel, retry, change product)
- [ ] Create admin subscription creation form with person/product/card selectors
- [ ] Implement admin card setup flow (save card for customer via Square Web Payments SDK)
- [ ] Create `POST /api/admin/save_card` endpoint (or enhance existing)
- [ ] Implement "rest of month free" flow (entitlement now, billing starts next month)
- [ ] Add subscription admin routes
- [ ] Tests: SubscriptionListComponent, SubscriptionDetailComponent, admin creation, card setup specs

### Phase 7 — Pricing Overview Page
- [ ] Create `PricingOverviewComponent` with schedule and product views
- [ ] Implement price schedule CRUD within the page
- [ ] Implement "copy prices from schedule" feature
- [ ] Tests: PricingOverviewComponent spec

### Phase 8 — Entitlement Management Page
- [ ] Create `EntitlementSearchComponent` with person/product search
- [ ] Create or reuse admin entitlement endpoint with filtering
- [ ] Display seat assignments inline
- [ ] Tests: EntitlementSearchComponent spec

### Phase 9 — Waitlist Management
- [ ] Add waitlist card to event session detail page (sorted by booking creation time)
- [ ] Implement "promote to confirmed" action per waitlisted booking
- [ ] Backend logic for waitlist promotion with capacity validation
- [ ] Auto-suggest promotion on confirmed booking cancellation
- [ ] Notification to promoted person (email)
- [ ] Tests: waitlist display, promotion, capacity validation, ordering

### Phase 10 — Facilities & Infrastructure
- (No work needed — facilities are already CRUD-able through existing admin portal)
- [ ] Facility schedule view (deferred until demand)

---

## Resolved Decisions (from Open Questions)

1. **Recurring event creation (R2.5)**: **In scope for Phase 4.** Important use case — creating sessions one by one is too tedious. Added recurring creation UI with pattern selection, preview, and batch creation to Phase 4.3.

2. **Room availability checking (R2.20)**: **In scope for Phase 4.** Very important — double-booking prevention is critical. Added room availability endpoint and conflict validation to Phase 4.6, integrated into both single and recurring session creation.

3. **Event calendar view (R2.10)**: **Deferred to separate document.** Will be needed eventually as a user-facing feature, not part of this admin portal work. Admin event list (sorted/filtered) is sufficient for admin workflow.

4. **Admin subscription creation (R2.11)**: **In scope for Phase 6.** Key workflow: admin sets up subscription during new member intro workshop, charges for next month but gives rest of current month free. Includes saving a card for the customer. Added as Phase 6.3.

5. **Product duplication (R2.2)**: **In scope for Phase 3.** Important for tiered memberships — Silver → Gold → Platinum are additive, so duplicating and modifying is the natural workflow. Added as Phase 3.4 with backend duplication endpoint.

6. **Manage Data visibility**: **Resolved — manage_products users do NOT see Manage Data.** That is for actual admins only. `manage_products` users only see the custom admin pages (Phases 3-8). Updated in Phase 1.4.

7. **Smart defaults and computed fields for CRUD forms (Phase 5)**: **Resolved — client-side only, via route state.** See Alternatives Considered below for the full decision process.

---

## Alternatives Considered

### Phase 5: How to handle smart defaults and computed fields in CRUD forms

**Context**: When creating an event session from a product page, the product already knows the default capacity and duration. The admin shouldn't have to re-enter these. The question was: where should this intelligence live, and how general should the solution be?

#### Approach 1: Bespoke Event Session Creation Page

Build a purpose-built Angular form specifically for creating event sessions, replacing the generic admin table CRUD for this use case.

- New `EventSessionCreateComponent` with full control over the form
- Knows the parent product, reads `default_capacity` and `duration_minutes`
- Auto-computes end time, pre-fills capacity, has "custom end time" toggle

**Pros**: Full UX control, no risk of breaking generic CRUD, simple to implement.
**Cons**: Only benefits event sessions. Duplicates form rendering logic (FK pickers, date inputs, validation) that generic CRUD already handles. Another component to maintain. Every future table needing similar behavior requires another bespoke page.

**Verdict**: Rejected. Too narrow — the underlying need (defaults and computed fields) is general enough to warrant a reusable solution.

#### Approach 2: Server-Side Metadata for Defaults and Derived Fields

Teach the admin table CRUD system to understand field defaults and computed fields through server-side metadata tables.

- New metadata table `admin_field_default_mappings`: maps `parent_table.column → child_table.column` for default values
- Derived fields expressed as metadata formulas (e.g., `end_time_us = start_time_us + duration_minutes * 60 * 1000000`)
- CRUD form fetches parent row on load, applies mapped defaults, registers derived field computations

**Pros**: Every table benefits automatically. Metadata-driven — configurable without code changes. Single system to maintain.
**Cons**: Significantly more complex. Expressing time arithmetic in database metadata is fragile and hard to debug. The CRUD form needs "auto-compute" state per field. Over-engineering — how many tables actually need this? The server gains complexity for what is purely a client UI concern.

**Verdict**: Rejected. We don't want to express formulas in database metadata. This is a client UI friendliness concern that shouldn't complicate the server.

#### Approach 3: Hybrid — Lightweight CRUD Extensions + Bespoke Where Needed

Extend the generic CRUD with simple default value passing via query params, but keep complex derived-field logic (duration→end time, recurring creation) in a bespoke event session form.

- Simple defaults via query param: `defaults=capacity:20,status:scheduled`
- Bespoke `EventSessionCreateComponent` for duration computation and recurring mode

**Pros**: Simple defaults work across all tables. Complex event logic gets proper UX. Lower risk.
**Cons**: Still requires a bespoke page. Two systems working together. The query param approach was the initial direction before the delivery mechanism was refined.

**Verdict**: Partially accepted — the idea of extending the generic CRUD was right, but splitting between CRUD defaults and a bespoke form was unnecessary. The computed date logic is general enough to live in the CRUD form itself.

#### Approach 4: JSON Query Parameters for Both Defaults and Computed Fields

Pass both `defaults` and `computed` as separate JSON query params to the CRUD form. No server changes, no bespoke pages — the generic CRUD handles everything.

- `defaults={"capacity":"20","status":"scheduled"}`
- `computed={"dates":[{"source":"start_time_us","dest":"end_time_us","offsetMinutes":60,"autoByDefault":true}]}`

**Pros**: Fully general, no server changes, no bespoke pages. Any caller can pass both types of assistance to any CRUD form.
**Cons**: Raw JSON in URLs is not URL-safe — braces, quotes, and colons must be percent-encoded, producing ugly URLs like `%7B%22capacity%22%3A%2220%22%7D`. Two separate query params rather than a unified object. Angular handles the encoding automatically, but the URLs are unreadable and can get long with complex computed rules.

**Verdict**: Close, but the delivery mechanism was wrong. The JSON-in-URL approach works but is inelegant.

#### Approach 5 (Chosen): Unified `CrudFormAssist` Object via Angular Route State

Combine defaults and computed field rules into a single `CrudFormAssist` interface, passed via Angular's route state rather than query params.

```typescript
interface CrudFormAssist {
  defaults?: Record<string, string>;
  computedDates?: ComputedDateRule[];
}
```

Passed via `router.navigate(..., { state: { formAssist } })`. The CRUD form reads `history.state?.formAssist` on init.

**Pros**:
- Clean URLs — no JSON encoding in the address bar
- No encoding/parsing concerns — typed JavaScript object, not serialized strings
- Single unified object for all form assistance — no splitting across multiple params
- Fully general — any page navigating to any CRUD form can pass assistance
- No server changes — purely client-side
- Extensible — add `computedNumbers`, `computedStrings`, etc. as needed without restructuring

**Cons**:
- Route state is lost on page refresh — but this is acceptable since nobody bookmarks or shares CRUD create form URLs. The user always navigates there from a parent page.

**Verdict**: Accepted. This is the chosen approach for Phase 5.