---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 2/6/2026
Version: 0.1
tags: 
---
# Overview

In the planning directory, there is Payment Design Document.md. I am currently working through Section 12 the implementation plan and am in phase 1. In particular, I would like to tackle "Purchase creation with server-side pricing." I previously worked with you to create an implement Product browsing and quoting endpoints.md that was focused on scenario 8.1.1 User purchases a one time item for themself like an intro workshop or massage. I suspect a lot of the work done there also went towards completing this item as well. Using this document, the code base, and the product browsing document, let's start putting together a plan to accomplish this work item (which might already be mostly completed). Let's enumerate the tables that will be needed, the features that will be used (like secrets, email, square client, etc).  Please go into plan mode and use this document to generate the in progress plan. Don't touch this overview but feel free to use everything below Overview as a scratch pad. I plan on doing several iterations building this document in plan mode before moving to implementing code. Please start with a high level description of what we are doing and what tables and components will be involved.

# High-Level Description

## Status: COMPLETE

**Backend: 100% Complete | Frontend: 100% Complete | All Phases A-G: Complete**

The thin slice for Phase 1 requires end-to-end functionality where a user can browse products, purchase an intro workshop, pay with a card, and view their purchase history in the portal. The server-side work is complete, but the Angular client needs significant work.

Phase C (Generic Table CRUD) planning is complete with all major decisions finalized. Ready to begin implementation.

## Goal: End-to-End Thin Slice

A logged-in user can:
1. Browse the product catalog → see products with resolved prices
2. Select a product (e.g., Intro Workshop) → add to cart
3. Proceed to checkout → enter card details via Square
4. Submit payment → receive confirmation
5. View purchase history → see completed purchases in user portal

---

# Current Status

## Backend (Complete ✅)

| Endpoint | Status | Purpose |
|----------|--------|---------|
| `GET /api/catalog_products` | ✅ | Product catalog with user-specific pricing |
| `POST /api/catalog_quote` | ✅ | Preview pricing (read-only) |
| `POST /api/purchase_create` | ✅ | Create purchase with server-resolved pricing |
| `POST /api/purchase_pay_card/{id}` | ✅ | Process card payment via Square |
| `GET /api/payments` | ✅ | Payment history for current user |

Supporting infrastructure:
- ✅ PurchaseHelper, CatalogHelper, PaymentHelper, EntitlementHelper
- ✅ SquareClient with CreatePayment
- ✅ IdempotencyHelper for safe retries
- ✅ Payment confirmation email
- ✅ All table helpers and db_schema

## Frontend (In Progress 🔄)

### What Exists
| Component | Location | Status |
|-----------|----------|--------|
| Payment Types | `portal/types/payment.types.ts` | ✅ Ready |
| Payment Methods on ServerAccess | `portal/types/ServerAccess.ts` | ✅ Ready |
| ServerAccessNetwork (HTTP) | `portal/services/network_abstraction/ServerAccess.ts` | ✅ Ready |
| ServerAccessMock (local mode) | `portal/services/network_abstraction/ServerAccess.mock.ts` | ✅ Ready |
| Square Payment Service | `portal/services/square-payment.service.ts` | ✅ Ready |
| Square TypeScript Types | `types/square.d.ts` | ✅ Ready |
| Error Handling (RFC 7807) | `shared/services/error/error.service.ts` | ✅ Ready |
| UI CLAUDE.md | `ui/CLAUDE.md` | ✅ Ready |

### What's Missing
| Component | Location | Priority | Purpose | Status |
|-----------|----------|----------|---------|--------|
| Directory structure plan | - | HIGH | Define separation of admin/user/public areas | ✅ Phase B Complete |
| DataResultsWithCount type | Backend + Client | HIGH | Server-side pagination with total count | 🔄 Phase C.0 |
| Generic table CRUD | `controls/` | HIGH | Reusable paginated table view with CRUD | 🔄 Phase C Ready |
| Admin product list | Admin Portal | MEDIUM | Manage products (view/edit) | Depends on Phase C |
| Shop/catalog page | Public pages | HIGH | Browse products with prices | Phase D |
| Checkout component | Public pages | HIGH | Card entry + purchase flow | Phase E |
| Purchase history | User area | HIGH | View past purchases | Phase F |
| Routes & navigation | - | MEDIUM | Shop, checkout, user area links | Phase G |
| Permission-based content | Various | MEDIUM | Different views by auth/membership | Future |

---

# Tables Involved

| Table | Purpose |
|-------|---------|
| `products` | Product catalog |
| `price_schedules` | Active price schedule |
| `product_prices` | Prices by product + schedule + permission |
| `purchases` | Purchase records |
| `purchase_items` | Line items with snapshotted prices |
| `payments` | Payment transaction records |
| `purchase_payments` | Links payments to purchases |
| `entitlements` | Access grants from purchases |
| `entitlement_assignments` | Who has access |

---

# Components Involved

## Backend (Complete)
| Component | Location |
|-----------|----------|
| `PurchaseHelper` | `payment/purchase_helper.h/cpp` |
| `CatalogHelper` | `payment/catalog_helper.h/cpp` |
| `PaymentHelper` | `payment/payment_helper.h/cpp` |
| `EntitlementHelper` | `payment/entitlement_helper.h/cpp` |
| `SquareClient` | `square/square_client.h/cpp` |

## Frontend (To Build)
| Component | Location | Phase |
|-----------|----------|-------|
| `DataResultsWithCount` type | `shared/types/DataResults.ts` | C.0 |
| `getRowsByColumnWithCount` | `shared/services/network/` | C.0 |
| `table-view-control` | `controls/table-view-control/` | C.1 |
| `composite-row-control` (add returnUrl) | `controls/composite-row-control/` | C.2 |
| Admin table routes | `pages/admin/` | C.3 |
| `CatalogComponent` | `pages/shop/catalog/` | D |
| `CheckoutComponent` | `pages/shop/checkout/` | E |
| `PurchaseHistoryComponent` | `pages/account/purchases/` | F |

---

# Implementation Plan

## Phase A: Foundation (Types & Service) ✅
- [x] Create payment types (`portal/types/payment.types.ts`)
- [x] Add payment methods to ServerAccess interface (`portal/types/ServerAccess.ts`)
- [x] Implement in ServerAccessNetwork (`portal/services/network_abstraction/ServerAccess.ts`)
- [x] Implement in ServerAccessMock (`portal/services/network_abstraction/ServerAccess.mock.ts`)
- [x] Update ServerAccessProxy (`portal/services/ServerAccess.ts`)
- [x] Add tests (`ServerAccess.mock.spec.ts`)
- [x] Create CLAUDE.md for UI project
- [x] Idempotency key generation via `crypto.randomUUID()` (browser native)

## Phase B: Client Directory Structure Planning

### Current State Analysis

| Directory | Current Purpose | Routes |
|-----------|-----------------|--------|
| `home/` | Public pages | `/`, `/about`, `/classes`, `/staff` |
| `auth/` | Mixed: login, user profile, AND admin | `/p/login`, `/p/profile`, `/p/admin` |
| `portal/` | Reusable form controls (misleading name) | (none) |
| `calendar/` | Scheduling | `/calendar` |
| `shared/` | Shared services/components | - |

**Issues with current structure:**
- `portal/` doesn't contain the admin portal - it has form components
- `auth/` has admin routes mixed with user profile routes
- No clear separation between public, user, and admin areas

### Proposed Restructure Options

#### Option A: Minimal Changes + New Modules
Keep existing structure mostly intact, add new modules for missing areas.

```
app/
├── auth/                    # Keep: login, register, password flows
├── home/                    # Keep: public pages (about, classes, staff)
├── shop/                    # NEW: catalog, product browsing, checkout
├── account/                 # NEW: user's area (profile, purchases, entitlements)
├── admin/                   # MOVE from auth: admin portal
├── calendar/                # Keep: scheduling
├── shared/                  # Keep: shared components/services
└── form-controls/           # RENAME from portal: reusable form widgets
```

**Routes:**
- `/` - Home (public)
- `/shop`, `/shop/:productId`, `/checkout/:purchaseId` - Shopping (public/auth)
- `/p/login`, `/p/register` - Auth
- `/account`, `/account/purchases`, `/account/entitlements` - User area (auth required)
- `/admin`, `/admin/products`, `/admin/users` - Admin (admin role required)

#### Option B: Feature-Based Modules ✅ SELECTED

See **Decision Points - Summary** and **Final Directory Structure** sections below for the finalized version with all naming decisions applied.

#### Option C: Content-Tier Based
Organize around the different content tiers.

```
app/
├── public/                  # Visible to everyone
│   ├── home/                # Landing, about
│   ├── classes/             # Class info
│   └── shop/                # Product catalog (prices may vary by tier)
├── member/                  # Requires authentication
│   ├── account/             # Profile, settings
│   ├── purchases/           # Purchase history
│   └── my-classes/          # Classes user has access to
├── premium/                 # Silver/Gold/Platinum exclusive content
│   └── ...
├── staff/                   # Admin/staff only
│   └── admin/               # Product/user management
├── auth/                    # Login, register
└── shared/
```

### Permission-Based Content Strategy

How content varies by user state:

| Page | Unauthenticated | Authenticated | Silver | Gold | Platinum |
|------|-----------------|---------------|--------|------|----------|
| Home | Public info | + Welcome | + Silver deals | + Gold deals | + Platinum deals |
| Shop | Public prices | Member prices | Silver prices | Gold prices | Platinum prices |
| Classes | Class list | + "My Classes" | + Silver classes | + Gold classes | + All classes |
| Account | Redirect to login | Profile/purchases | + Silver benefits | + Gold benefits | + Platinum benefits |

### Decision Points - Summary

| Decision | Choice |
|----------|--------|
| Structure | **Option B** - Feature-based modules |
| User area name | **`account`** |
| Form controls directory | **`controls`** (rename from `portal`) |
| URL prefix for user area | **`/my/`** → `/my/account`, `/my/purchases` |
| Feature modules directory | **`pages`** (rename from `features`) |
| Price visibility | **Public prices visible**, show membership discount info |
| AuthService location | **`core/services/`** |

### Final Directory Structure

```
app/
├── core/                         # App-wide singletons
│   ├── guards/                   # Route guards (auth, admin, permission)
│   ├── interceptors/             # HTTP interceptors
│   └── services/                 # AuthService, ErrorService, etc.
│
├── pages/                        # Feature modules (renamed from features)
│   ├── public/                   # Public-facing pages (no auth required)
│   │   ├── home/
│   │   ├── about/
│   │   ├── classes/
│   │   └── staff/
│   │
│   ├── auth/                     # Authentication flows
│   │   ├── login/
│   │   ├── register/
│   │   └── forgot-password/
│   │
│   ├── shop/                     # Shopping (public browse, auth for checkout)
│   │   ├── catalog/
│   │   ├── product-detail/
│   │   └── checkout/
│   │
│   ├── account/                  # User's private area (auth required)
│   │   ├── profile/
│   │   ├── purchases/
│   │   └── entitlements/
│   │
│   ├── admin/                    # Admin portal (admin role required)
│   │   ├── dashboard/
│   │   ├── products/
│   │   ├── users/
│   │   └── purchases/
│   │
│   └── calendar/                 # Scheduling (auth required)
│
├── shared/                       # Reusable, stateless components
│   ├── components/
│   ├── pipes/
│   ├── directives/
│   └── types/
│
└── controls/                     # Form controls (renamed from portal)
    ├── simple-text/
    ├── simple-bool/
    ├── simple-date/
    ├── composite-control/
    └── composite-row-control/
```

### Final Route Structure

| Route | Page Module | Auth | Role |
|-------|-------------|------|------|
| `/` | public/home | No | - |
| `/about` | public/about | No | - |
| `/classes` | public/classes | No | - |
| `/staff` | public/staff | No | - |
| `/login` | auth/login | No | - |
| `/register` | auth/register | No | - |
| `/shop` | shop/catalog | No | - |
| `/shop/:id` | shop/product-detail | No | - |
| `/checkout/:purchaseId` | shop/checkout | Yes | - |
| `/my/account` | account/profile | Yes | - |
| `/my/purchases` | account/purchases | Yes | - |
| `/my/entitlements` | account/entitlements | Yes | - |
| `/admin` | admin/dashboard | Yes | admin |
| `/admin/products` | admin/products | Yes | admin |
| `/admin/users` | admin/users | Yes | admin |
| `/calendar` | calendar | Yes | - |

### Path Aliases

```typescript
// tsconfig.json
"paths": {
  "@core/*": ["src/app/core/*"],
  "@pages/*": ["src/app/pages/*"],
  "@shared/*": ["src/app/shared/*"],
  "@controls/*": ["src/app/controls/*"]
}
```

### Implementation Tasks

**Phase B.1: Directory Restructure**
- [x] Create `core/` directory ✅ 2026-02-11
  - [x] Create `core/guards/` ✅ 2026-02-11
  - [x] Create `core/interceptors/` ✅ 2026-02-11
  - [x] Create `core/services/` ✅ 2026-02-11
  - [x] Move `AuthService` from `shared/services/auth/` to `core/services/` ✅ 2026-02-11
  - [x] Move guards from `shared/services/auth/` to `core/guards/` ✅ 2026-02-11
  - [x] Move interceptors from `shared/interceptors/` to `core/interceptors/` ✅ 2026-02-11

- [x] Create `pages/` directory structure ✅ 2026-02-11
  - [x] Create `pages/public/` and move content from `home/` ✅ 2026-02-11
  - [x] Create `pages/auth/` and move login/register from `auth/` ✅ 2026-02-11
  - [x] Create `pages/account/` and move profile components from `auth/` ✅ 2026-02-11
  - [x] Create `pages/admin/` and move admin components from `auth/` ✅ 2026-02-11
  - [x] Create `pages/shop/` (new - catalog, product-detail, checkout) ✅ 2026-02-11
  - [x] Move `calendar/` to `pages/calendar/` ✅ 2026-02-11

- [x] Rename `portal/` to `controls/` ✅ 2026-02-11

- [x] Update `tsconfig.json` path aliases ✅ 2026-02-11
  - [x] Add `@core/*`, `@pages/*`, `@controls/*` ✅ 2026-02-11
  - [x] Remove old aliases (`@auth/*`, `@home/*`, `@portal/*`, `@calendar/*`) ✅ 2026-02-11

- [x] Update all imports throughout codebase ✅ 2026-02-11

**Phase B.2: Route Updates**
- [x] Update `app.routes.ts` with new structure ✅ 2026-02-11
- [x] Change `/p/` prefix to `/my/` for user routes ✅ 2026-02-11
- [x] Flatten auth routes (`/login`, `/register` instead of `/p/login`) ✅ 2026-02-11
- [x] Add shop routes (`/shop`, `/shop/:id`, `/checkout/:purchaseId`) ✅ 2026-02-11
- [x] Update route guards ✅ 2026-02-11

**Phase B.3: Documentation**
- [x] Update `ui/CLAUDE.md` with new structure ✅ 2026-02-11
- [x] Update this planning document ✅ 2026-02-11

## Phase C: Generic Table CRUD Controls

### Goal

Create a reusable, generic paginated table view control that can provide basic CRUD support for any database table. This builds on the existing `composite-control` (single field editor) and `composite-row-control` (full row editor) to create a complete table management experience.

### Existing Controls

| Control | Purpose | Location |
|---------|---------|----------|
| `simple-text` | Edit VARCHAR fields | `controls/simple-text/` |
| `simple-bool` | Edit BOOLEAN fields | `controls/simple-bool/` |
| `simple-date` | Edit DATE/TIMESTAMP fields | `controls/simple-date/` |
| `long-text` | Edit TEXT fields | `controls/long-text/` |
| `composite-control` | Aggregates simple controls, edits single field based on type | `controls/composite-control/` |
| `composite-row-control` | Edits a full row of data using multiple composite-controls | `controls/composite-row-control/` |

### New Control: `table-view-control`

A paginated, read-only view of table data with CRUD action buttons.

**Inputs:**
- `tableName: string` - The database table to display
- `pageSize: number` - Number of rows per page (default: 10)
- `pageOffset: number` - Current page offset (default: 0)
- `sortColumn?: string` - Column to sort by (default: primary key)
- `sortAscending?: boolean` - Sort direction (default: true)

**Features:**
- Display paginated rows from the table
- Pagination controls: First | Prev | [1] [2] [3] ... | Next | Last
- Show current page info: "Showing 1-10 of 47"
- Per-row actions: Edit button, Delete button
- Table-level actions: "New Item" button
- Column headers from schema (friendly names when available)
- Loading state while fetching data

**Actions:**
- **Edit**: Navigate to row editor (composite-row-control) in edit mode
- **Delete**: Material confirmation dialog → call delete API → refresh table
- **New Item**: Navigate to row editor (composite-row-control) in create mode

### Enhancement: `composite-row-control`

The existing `composite-row-control` already has most inputs needed:
- `tableName: string` - The table being edited ✅ exists
- `isCreateMode: boolean` - Operation mode ✅ exists
- `primaryKeyName: string` - Primary key column name ✅ exists
- `primaryKeyValue: string` - Primary key value (for edit mode) ✅ exists

**New Input:**
- `returnUrl?: string` - Where to navigate after save or cancel

**Behavior:**
- **Create mode** (`isCreateMode=true`): Empty form, save calls `addItem` API
- **Edit mode** (`isCreateMode=false`): Pre-populated form, save calls `updateItem` API
- On success (save) or cancel → navigate to `returnUrl` (or fallback to previous page)

---

### Decisions

#### Q1: Routing Strategy ✅ DECIDED

**Decision: Nested Routes with Pagination in URL**

```
/admin/tables/:tableName/view                        → Table view (default page)
/admin/tables/:tableName/view/:pageSize/:pageOffset  → Table view w/ pagination
/admin/tables/:tableName/edit/:id                    → Edit row
/admin/tables/:tableName/new                         → Create new row
```

The `returnUrl` pattern allows row editor to navigate back to the exact page the user was viewing.

#### Q2: Table Selection UI 🔄 DEFERRED

**Status: Needs further discussion**

Options considered:
- **Dropdown in header** - Simplest, may not scale to many tables
- **Sidebar navigation** - Good UX, but horizontal space is limited on mobile
- **Container control** - Dropdown that hosts `table-view-control` with navigation via output events

For Phase C, we'll start with a simple dropdown approach using `root_tables` from schema. The container control approach may be revisited when we have more experience with the flow.

#### Q3: Delete Confirmation ✅ DECIDED

**Decision: Material Dialog (Option B)**

Use Angular Material dialog with styled confirmation that can show row details. Provides consistent, polished UX.

#### Q4: Pagination Implementation ✅ DECIDED

**Decision: Server-Side Pagination with Total Count**

Use the existing `getRowsByColumn` API with server-side pagination. Requires backend modification:
- Create new `DataResultsWithCount` type that includes `DataResults` plus `totalCount: number`
- Modify `getRowsByColumn` endpoint to return total count for pagination display

#### Q5: Which Tables Should Be Editable ✅ DECIDED

**Decision: Use `root_tables` (Option A)**

The `root_tables` field in `DatabaseSchema` was specifically designed for this purpose - it lists all top-level, admin-editable tables.

#### Q6: Form Validation ✅ DECIDED

**Decision: Schema-Driven Using All ColumnDataInfo Fields**

Use all available validation fields from `ColumnDataInfo`:
- `nullable` - Whether field can be empty
- `required` - Whether field must have a value
- `max_length` - Maximum string length (for text fields)
- `regex` - Pattern validation

Future enhancement: Add custom validators per table/column via metadata.

#### Q7: Foreign Key Handling ⏳ DEFERRED

**Decision: Defer to Later Phase**

Foreign key handling with autocomplete search is the desired UX, but:
- `root_tables` won't have foreign keys by definition
- Viewing/editing nested tables (which have FKs) is a separate, larger work item

Will implement autocomplete for FK fields in a future phase when nested table editing is tackled.

**Exposing more top level tables**
- Root tables
	- admin_column_data_info
	- admin_column_friendly_names
	- classes
	- config_secrets
	- permissions
	- price_schedules
	- products
	- roles
	- people
- Allowed tables
	- classes
	- products
- Admin tables
	- admin_column_data_info
	- admin_column_friendly_names
	- classes
	- config_secrets
	- permissions
	- price_schedules
	- products
	- roles
	- people
- Things to do
	- Inside database_helper/create_database.cpp in the PopulateAdminTopLevelTables, look up each name under Admin tables above in the db_schema directory and add the associated k{pascal_case_table_name}Table as an AddRow entry to this function. In PopulateAllowedTables, add products and remove people. In db_schema/roles.h, add constants for kRoleAdmin, kRoleUser, kRoleTeachers and then modify PopulateRoles to use these constants. In auth/session.h, add a method bool IsAdmin() const that checks for the role kRoleAdmin. Add tests for this as well. In endpoints/endpoint_auth_helper.cpp, inside GetAllowedTables, add a check on the session for IsAdmin() and then create a TableHelpers::AdminTopLevelTables and return GetAdminTopLevelTables() for the admin case and the existing GetAllowedTables() otherwise. Inside EndpointAuthHelper::IsTableAllowed, do the same check for IsAdmin and use the GetAdminTopLevelTables() for the admin case or go with the existing functionality otherwise. Please add tests for all of these changes which will involve creating users with admin permissions and creating a session for the endpoints.

---

### Implementation Tasks

**Phase C.0: Backend Enhancement** ✅ Complete
- [x] Create `DataResultsWithCount` type (contains `DataResults` + `totalCount`)
- [x] Modify `getRowsByColumn` endpoint to return `DataResultsWithCount`
- [x] Update client types and service methods

**Phase C.1: Table View Control**
- [x] Create `table-view-control` component in `controls/` ✅ 2026-02-11
- [x] Implement server-side pagination logic with total count ✅ 2026-02-11
- [x] Display table data with column headers from schema ✅ 2026-02-11
- [x] Add row action buttons (Edit, Delete) ✅ 2026-02-11
- [x] Add "New Item" button ✅ 2026-02-11
- [x] Implement delete with Material confirmation dialog ✅ 2026-02-11
- [x] Add loading and empty states ✅ 2026-02-11

**Phase C.2: Enhance Composite Row Control**
- [x] Add `returnUrl` input for navigation after save/cancel ✅ 2026-02-11
- [x] Implement navigation on success (addItem/updateItem) → `returnUrl` ✅ 2026-02-11
- [x] Implement navigation on cancel → `returnUrl` ✅ 2026-02-11

**Phase C.3: Admin Table Routes**
- [x] Add routes: ✅ 2026-02-11
  - `/admin/tables/:tableName/view`
  - `/admin/tables/:tableName/view/:pageSize/:pageOffset`
  - `/admin/tables/:tableName/edit/:id`
  - `/admin/tables/:tableName/new`
- [x] Wire up navigation between components ✅ 2026-02-11
- [x] Add table selector dropdown using `root_tables` ✅ 2026-02-11
- [x] Update admin dashboard to use new table view ✅ 2026-02-11

**Phase C.4: Testing**
- [x] Unit tests for table-view-control ✅ 2026-02-12
- [x] Unit tests for composite-row-control with returnUrl ✅ 2026-02-12
- [x] Integration test: create, edit, delete flow ✅ 2026-02-12

**Phase C.5: Exposing More Top-Level Tables**

Goal: Expose additional database tables in the admin dashboard with role-based access control. Admin users see all admin tables; regular users see only a limited set of allowed tables.

**Table tiers:**
| Tier | Tables | Who sees them |
|------|--------|---------------|
| Allowed (regular users) | `classes`, `products` | Any authenticated user |
| Admin (admin role) | `classes`, `config_secrets`, `permissions`, `price_schedules`, `products`, `roles`, `people` | Admin users only |

**Architecture note — Root table filtering:**
The `get_db_schema` endpoint calls `GenerateRootTablesArrayMetadata()` which returns only tables that are both (a) in the allowed tables list AND (b) are "root tables" (tables with no outgoing FK columns in the schema). `admin_column_data_info` and `admin_column_friendly_names` have FK references to `admin_top_level_tables`, making them non-root tables. **Decision:** Exclude these two tables from the admin list. The remaining 7 tables are all root tables.

**C.5.1: Add role string constants** (`db_schema/roles.h`) ✅
- [x] Add `kRoleNameAdmin = "admin"`, `kRoleNameUser = "user"`, `kRoleNameTeachers = "teachers"` constants
- [x] Update `PopulateRoles` in `create_database.cpp` to use these constants instead of string literals

**C.5.2: Add `Session::IsAdmin()`** (`auth/session.h`, `auth/session.cpp`) ✅
- [x] Add `bool IsAdmin(Transaction& transaction)` that delegates to `ActiveUserHasRole(transaction, DbSchema::kRoleNameAdmin)`
- [x] Add unit tests for `IsAdmin()` with admin and non-admin users
  - Create roles table, role_assignments table, and people table
  - Assign admin role to test user, verify `IsAdmin()` returns true
  - Verify `IsAdmin()` returns false for user without admin role

**C.5.3: Update endpoint auth for role-based table access** ✅
- [x] Update `EndpointAuthHelper::GetAllowedTables(Transaction&)`:
  - Check `session_.IsLoggedIn() && session_.IsAdmin(transaction)`
  - Admin → merge `AdminTopLevelTables` into base `AllowedTables`
  - Non-admin → return existing `app_.GetAllowedTables(transaction)`
  - Wrapped in try/catch for graceful degradation
- [x] Update `EndpointAuthHelper::IsTableAllowed(Transaction&, tableName)` to use updated `GetAllowedTables`
- [x] Removed `const` from `GetAllowedTables`/`IsTableAllowed` since they now access `session_` (non-const)

**C.5.4: Populate database tables** (`database_helper/create_database.cpp`) ✅
- [x] Update `PopulateAdminTopLevelTables` — add all 7 admin tables using their `k*Table` constants from `db_schema/`:
  - `kClassesTable`, `kConfigSecretsTable`, `kPermissionsTable`, `kPriceSchedulesTable`, `kProductsTable`, `kRolesTable`, `kPeopleTable`
- [x] Update `PopulateAllowedTables` — change to: `kClassesTable`, `kProductsTable` (remove `kPeopleTable`)

**C.5.5: Add UI metadata for new tables** (`database_helper/create_database.cpp`) ✅
- [x] Add `PopulateAdminTableFriendlyNames` entries for new tables (friendly display names)
- [x] Add `PopulateAdminColumnFriendlyNames` entries for columns of new tables
- [x] Add `PopulateAdminColumnDataInfo` entries for columns of new tables (html_input_type, labels, hints, validation)
- [x] Tables needing metadata: `config_secrets`, `permissions`, `price_schedules`, `products`, `roles`
  - `classes` and `people` already have metadata

**C.5.6: Resolve root table filtering** ✅ Decided
- [x] Decision: Exclude `admin_column_data_info` and `admin_column_friendly_names` from the admin tables list. They have FK references to `admin_top_level_tables` making them non-root tables, so the schema endpoint won't return them. No code change needed to `GenerateRootTablesArrayMetadata`.

**C.5.7: Backend tests** ✅
- [x] Test `Session::IsAdmin()` (see C.5.2)
- [x] Test `EndpointAuthHelper::GetAllowedTables` returns admin tables when session has admin role
- [x] Test `EndpointAuthHelper::GetAllowedTables` returns allowed tables when session does NOT have admin role
- [x] Test `EndpointAuthHelper::IsTableAllowed` grants access to admin-only table for admin user
- [x] Test `EndpointAuthHelper::IsTableAllowed` denies access to admin-only table for non-admin user
- [x] Tests will need: people table, roles table, role_assignments table, sessions table, admin_top_level_tables, allowed_tables

**C.5.8: Frontend mock updates** ✅
- [x] Update `ServerAccess.mock.ts` database schema to include all admin tables with their columns
- [x] Update mock `root_tables` to match the new admin table list (7 tables)
- [x] Add mock table data for all 5 new tables (config_secrets, permissions, price_schedules, products, roles)
- [x] Add/update integration tests for admin dashboard with expanded table list (5 new tests)
- [x] All 218 Angular tests pass

**C.5.9: Database reset** ✅
- [x] Rebuild and run the `create_database` executable to repopulate the database with new table entries
- [x] Verify all new tables appear in admin dashboard for admin user
- [x] Fix: `main.cpp` now merges `allowed_tables` + `admin_top_level_tables` when building `DatabaseInfo` so all 7 tables have schema metadata loaded at startup
- [x] Fix: `DatabaseSchemaService.ts` refreshes schema on auth state changes
- [ ] Verify non-admin user sees only allowed tables

## Phase D: Shop/Catalog Pages (Public)

**Goal**: Create customer-facing shop pages — a card-based product catalog and a product detail page. Uses the existing `getCatalogProducts()` API which returns active products with resolved prices from the active price schedule. No backend changes needed. Permission-based pricing is deferred to a later iteration.

**Existing infrastructure (all ready)**:
- `CatalogProduct` type → `shared/types/payment.types.ts`
- `getCatalogProducts()` → `shared/types/ServerAccess.ts` (interface), `ServerAccessNetwork.ts` (HTTP), `ServerAccess.mock.ts` (mock with 3 products)
- `formatCents()` utility → `shared/types/payment.types.ts`
- Shop route `/shop` → `app.routes.ts` (lazy-loaded, no auth required)
- Shop routes file → `pages/shop/shop.routes.ts` (placeholder TODOs only)

**D.1: Create CatalogComponent** (`pages/shop/catalog/`) ✅
- [x] `catalog.component.ts` / `.html` / `.scss` / `.spec.ts`
- [x] On init, call `serverAccess.getCatalogProducts()` to fetch products
- [x] Display products as responsive card grid (TailwindCSS)
- [x] Each card shows: product name, description (truncated), formatted price
- [x] Each card links to `/shop/:id` (product detail page)
- [x] Loading state while fetching, empty state if no products
- [x] Standalone component, inject `ServerAccess` via `SERVER_ACCESS_TOKEN`

**D.2: Create ProductDetailComponent** (`pages/shop/product-detail/`) ✅
- [x] `product-detail.component.ts` / `.html` / `.scss` / `.spec.ts`
- [x] Read `:id` from route params
- [x] Fetch catalog products and find matching product by ID (client-side filter — fine for small catalog)
- [x] Display full product info: name, full description, kind, formatted price
- [x] "Buy" / "Purchase" button (placeholder for Phase E checkout)
- [x] "Back to catalog" link
- [x] 404 handling if product ID not found

**D.3: Wire up routes** (`pages/shop/shop.routes.ts`) ✅
- [x] `/shop` → `CatalogComponent`
- [x] `/shop/:id` → `ProductDetailComponent`

**D.4: Add Shop to navigation** (`shared/services/header/mockHeaderResponse.ts`) ✅
- [x] Add "Shop" `InternalLink` to `/shop` in header menu
- [x] Visible to all users (no auth required)
- [x] Position: after "Services" → `Get Started | About | Our Classes | Services | Shop | ...`

**D.5: Tests** ✅
- [x] Catalog: display product cards, loading state, empty state, price formatting, links to detail (8 tests)
- [x] Product detail: display details for valid ID, not-found for invalid ID, price formatting, back link (7 tests)
- [x] All 233 tests pass

**Files to create/modify**:
| File | Action |
|------|--------|
| `pages/shop/catalog/catalog.component.ts` | Create |
| `pages/shop/catalog/catalog.component.html` | Create |
| `pages/shop/catalog/catalog.component.scss` | Create |
| `pages/shop/catalog/catalog.component.spec.ts` | Create |
| `pages/shop/product-detail/product-detail.component.ts` | Create |
| `pages/shop/product-detail/product-detail.component.html` | Create |
| `pages/shop/product-detail/product-detail.component.scss` | Create |
| `pages/shop/product-detail/product-detail.component.spec.ts` | Create |
| `pages/shop/shop.routes.ts` | Edit |
| `shared/services/header/mockHeaderResponse.ts` | Edit |

**Verification**:
- `ng test` — all existing + new tests pass
- `ng serve -c local` — navigate to `/shop`, see 3 product cards with prices
- Click a card → `/shop/:id` shows full product details
- Header shows "Shop" link for all users
- Back button on detail page returns to catalog

## Phase E: Checkout Flow

**Goal**: Enable actual purchases by creating a checkout component that integrates with the Square Web Payments SDK for card tokenization and the existing backend payment APIs. The backend is 100% complete. This phase is purely frontend Angular work.

### Key Design Decisions

1. **Route**: `/shop/checkout/:productId` (uses product ID, not purchase ID)
   - Page works on refresh (product info fetched from catalog)
   - Purchase is created when user clicks "Pay", not on page load
   - Avoids orphaned pending purchases from browsing

2. **Auth**: `AuthGuard` on the checkout child route only
   - Catalog and product detail remain public
   - If user clicks "Purchase" while logged out, guard redirects to `/login`

3. **Purchase creation timing**: On "Pay" click, not page load
   - Checkout page displays catalog price (already server-resolved)
   - First click: create purchase → tokenize card → pay
   - Retry after error: reuse existing purchase → tokenize new card → pay

4. **No cart**: Direct single-product checkout (thin-slice)

### Existing Infrastructure (all ready)

| What | File | Notes |
|------|------|-------|
| `CatalogProduct`, `Purchase`, `PayCardRequest`, `PayCardResponse`, `formatCents()` | `shared/types/payment.types.ts` | All types defined |
| `ProblemDetails`, `ErrorTypes`, `isProblemDetails()` | `shared/types/ApiError.ts` | RFC 7807 error handling |
| `getCatalogProducts()`, `createPurchase()`, `purchasePayCard()` | `shared/types/ServerAccess.ts` | Interface + HTTP + Mock all ready |
| `SquarePaymentService` | `shared/services/square-payment.service.ts` | `attachCard()`, `tokenizeCard()`, `destroy()` |
| `AuthGuard` | `core/guards/auth-guards.ts` | Redirects to `/login` |
| `AuthService` | `core/services/auth.service.ts` | `authData$`, `authData.isAuth` |

### Implementation Steps

**E.1: Fix `ServerAccessNetwork.purchasePayCard()` response mapping** ✅
- [x] **File**: `shared/services/network/ServerAccessNetwork.ts`
- [x] Backend returns `{ payment, purchase, entitlements: [...] }` (array) but frontend `PayCardResponse` has `entitlements_created: number` (count)
- [x] Add a `map()` pipe (same pattern as existing `getCatalogProducts()` transform): define local `RawPayCardResponse` interface, map `entitlements.length` → `entitlements_created`

**E.2: Enable Purchase button on product detail page** ✅
- [x] `product-detail.component.ts` — Add `Router` injection and `buyProduct()` method that navigates to `/shop/checkout/:productId`
- [x] `product-detail.component.html` — Replace disabled "Purchase (Coming Soon)" button with active "Purchase" button wired to `(click)="buyProduct()"`
- [x] `product-detail.component.spec.ts` — Add test for buy button navigation

**E.3: Create CheckoutComponent** (`pages/shop/checkout/`) ✅
- [x] `checkout.component.ts`
- [x] `checkout.component.html`
- [x] `checkout.component.scss`
- [x] `checkout.component.spec.ts`

**Component state machine**:
```
loading → ready → processing → success
                → error (retry → processing → ...)
loading → product_not_found
```

**State fields**:
- `state: CheckoutState` — 'loading' | 'ready' | 'processing' | 'success' | 'error' | 'product_not_found'
- `product: CatalogProduct | null` — loaded from catalog
- `errorMessage: string` — user-facing error text
- `purchase: Purchase | null` (private) — created on first pay attempt, reused on retry

**Lifecycle**:
- `ngOnInit`: Read `:productId` from route, call `getCatalogProducts()`, find product, set state to `ready`
- After product loads: Call `squarePayment.attachCard('#card-container')`
- `ngOnDestroy`: Call `squarePayment.destroy()`

**`onPay()` flow** (async):
1. Set state to `processing`, clear errors
2. Create purchase if not already created: `serverAccess.createPurchase({ items: [{ product_id, quantity: 1 }] })`
3. Tokenize card: `await squarePayment.tokenizeCard()` → nonce
4. Generate idempotency key: `crypto.randomUUID()`
5. Pay: `serverAccess.purchasePayCard(purchase.id, { source_id: nonce, idempotency_key })`
6. On success: state = `success`
7. On error: state = `error`, set `errorMessage` based on error type

**Error handling**:
- `Error` (from Square SDK tokenization) → show `error.message`
- `HttpErrorResponse` with `ProblemDetails` → map `ErrorTypes.PAYMENT_DECLINED` to "Your card was declined", etc.
- Fallback → "An unexpected error occurred. Please try again."

**Template structure**:
- Back to shop link
- Loading spinner
- Product-not-found state
- Order summary card (product name, kind, price, total)
- Payment details card with `<div id="card-container">` for Square form
- Error message banner
- "Pay $XX.XX" button (disabled during processing, shows spinner)
- Success state (check icon, "Payment Successful!", Continue Shopping button)

**Styles**: `max-width: 600px` centered layout (narrower than catalog for focused checkout form)

**E.4: Register checkout route** ✅
- [x] **File**: `pages/shop/shop.routes.ts`
- [x] Add `{ path: 'checkout/:productId', component: CheckoutComponent, canActivate: [AuthGuard] }`
- [x] **Critical**: Must come BEFORE `:id` wildcard — Angular matches routes in order

```typescript
const routes: Routes = [
  { path: '', component: CatalogComponent },
  { path: 'checkout/:productId', component: CheckoutComponent, canActivate: [AuthGuard] },
  { path: ':id', component: ProductDetailComponent },
];
```

**E.5: Checkout component tests** (10 tests) ✅
- [x] Displays order summary for valid product ID
- [x] Shows product-not-found for invalid ID (999)
- [x] Shows product-not-found for non-numeric ID (abc)
- [x] Calls `squarePayment.attachCard('#card-container')` after product loads
- [x] `onPay()` creates purchase, tokenizes card, and pays successfully
- [x] `onPay()` reuses existing purchase on retry (createPurchase called only once)
- [x] Shows error message when tokenization fails
- [x] Shows error message when payment is declined (ProblemDetails with PAYMENT_DECLINED)
- [x] Calls `squarePayment.destroy()` on component destroy
- [x] Pay button is disabled during processing state

Mock setup: Mock `SquarePaymentService` with spies for `attachCard`, `tokenizeCard`, `destroy`. Follow existing pattern from `product-detail.component.spec.ts` for the `createComponent` helper.

### Files Modified/Created

| Step | File | Action |
|------|------|--------|
| E.1 | `shared/services/network/ServerAccessNetwork.ts` | Modify — add `map()` to `purchasePayCard()` |
| E.2 | `pages/shop/product-detail/product-detail.component.ts` | Modify — add Router, `buyProduct()` |
| E.2 | `pages/shop/product-detail/product-detail.component.html` | Modify — enable Purchase button |
| E.2 | `pages/shop/product-detail/product-detail.component.spec.ts` | Modify — add buy navigation test |
| E.3 | `pages/shop/checkout/checkout.component.ts` | **Create** |
| E.3 | `pages/shop/checkout/checkout.component.html` | **Create** |
| E.3 | `pages/shop/checkout/checkout.component.scss` | **Create** |
| E.5 | `pages/shop/checkout/checkout.component.spec.ts` | **Create** |
| E.4 | `pages/shop/shop.routes.ts` | Modify — add checkout route with AuthGuard |

**Total**: 5 files modified, 4 files created.

### Verification

1. **`ng test`** — all existing + new tests pass
2. **`ng serve -c local`**:
   - `/shop` — catalog displays products
   - `/shop/:id` — product detail shows "Purchase" button (not disabled)
   - Click "Purchase" while logged out — redirected to `/login`
   - Click "Purchase" while logged in — navigates to `/shop/checkout/:productId`
   - Checkout page shows order summary with product name and price
   - Square card form renders (may not fully initialize with placeholder location ID)
3. **Route ordering**: `/shop/checkout/1` loads checkout, `/shop/1` loads product detail
4. **Error states**: Invalid product ID shows not-found, card errors display messages

## Phase F: User Purchase History

Logged-in users can view their purchase history at `/my/purchases`. Each purchase shows product names, prices, timestamps, status, and any entitlements granted.

### F.1: Add `createdUs` to `PurchaseInfo` and product details to `PurchaseItemInfo`

**Modify `server/.../payment/purchase_helper.h`**:
- Add `int64_t createdUs = 0;` to `PurchaseInfo` struct
- Add `std::string productName;` and `std::string productCode;` to `PurchaseItemInfo` struct

**Modify `server/.../payment/purchase_helper.cpp`**:
- In `PurchaseInfoFromKeyValueTable()`: read `kPurchasesCreatedUs` (constant already exists in `db_schema/purchases.h`) into `info.createdUs`
- In `PurchaseItemInfoFromKeyValueTable()`: look up product via `Products::GetProduct(transaction, productId)` to populate `productName` and `productCode`
  - Requires changing `PurchaseItemInfoFromKeyValueTable` signature to accept `Transaction&` (currently `const`-only)
  - Add `TableHelpers::Products` member to `PurchaseHelper` (or use the one on `CatalogHelper`)

**Modify `server/.../payment/payment_key_value_table.cpp`**:
- `PurchaseInfoToKeyValueTable()`: add `table["created_us"] = StringFromInt(info.createdUs);`
- `PurchaseItemInfoToKeyValueTable()`: add `table["product_name"] = info.productName;` and `table["product_code"] = info.productCode;`

**Tests for F.1 — Modify existing test files**:

**Modify `server/.../payment/purchase_helper_test.cpp`** — add tests:
1. `GetPurchaseHasCreatedUs` — Create a purchase, retrieve it via `GetPurchase()`, verify `createdUs > 0`
2. `GetPurchaseItemsHaveProductName` — Create a purchase with a known product ("Workshop"), retrieve it, verify `items[0].productName == "Workshop"` and `items[0].productCode == "WORKSHOP_1"`
3. `GetPurchasesForPersonHasCreatedUs` — Create purchases, retrieve via `GetPurchasesForPerson()`, verify each has `createdUs > 0`

**Modify `server/.../payment/payment_key_value_table_test.cpp`** — add tests:
4. `PurchaseInfoToKeyValueTableIncludesCreatedUs` — Set `info.createdUs = 1700000000000000`, convert, verify `table["created_us"] == "1700000000000000"`
5. `PurchaseItemInfoToKeyValueTableIncludesProductName` — Set `info.productName = "Workshop"` and `info.productCode = "WORKSHOP_1"`, convert, verify `table["product_name"] == "Workshop"` and `table["product_code"] == "WORKSHOP_1"`

### F.2: Create `GET /api/purchases` endpoint

**Create `server/.../endpoints/purchases.h`** — header with `GetPurchases()` declaration

**Create `server/.../endpoints/purchases.cpp`**:
- Route: `GET /api/purchases`
- Auth: require `session.IsLoggedIn()`
- Logic:
  1. `PurchaseHelper::GetPurchasesForPerson(transaction, session.GetPersonId())`
  2. For each purchase, `EntitlementHelper::GetEntitlementsForPurchase(transaction, purchase.id)`
  3. Build nested JSON (reuse `PurchaseToJson()` / `PurchaseItemToJson()` helpers from `purchase_create.cpp` — adapt to include new fields and entitlements)
- Response shape:
```json
{
  "purchases": [
    {
      "id": 1, "status": "funded", "currency": "USD",
      "total_cents": 5000, "paid_cents": 5000, "created_us": 1234567890000000,
      "items": [
        { "id": 1, "product_id": 1, "product_name": "Intro Workshop", "product_code": "INTRO_WORKSHOP",
          "quantity": 1, "unit_price_cents": 5000, "line_total_cents": 5000, "currency": "USD" }
      ],
      "entitlements": [
        { "id": 1, "product_id": 1, "status": "active",
          "valid_from_us": ..., "valid_to_us": ..., "seats_total": 1, "seats_used": 0 }
      ]
    }
  ]
}
```

**Modify `server/.../endpoints/web_app.cpp`** — add `#include "purchases.h"` and `auto g_Purchases = &Endpoints::GetPurchases;`
**Modify `server/.../endpoints/CMakeLists.txt`** — add `purchases.h` and `purchases.cpp` to sources

### F.3: Backend endpoint test

**Create `server/.../endpoints/purchases_test.cpp`**:
1. Returns 401 when not logged in
2. Returns empty array for user with no purchases
3. Returns purchases with items including product names
4. Returns entitlements nested under each funded purchase
5. Only returns purchases for the logged-in user

Uses `TestUtil::MakePaymentTables` + `MakeSessionsTable` for setup.

**Modify `server/.../endpoints/CMakeLists.txt`** — add `purchases_test.cpp` to test sources

### F.4: Frontend types and ServerAccess method

**Modify `ui/.../shared/types/payment.types.ts`**:
- Add `created_us?: number;` to `Purchase` interface (optional — `purchase_create` response doesn't include it)
- Add `product_name?: string;` and `product_code?: string;` to `PurchaseItem` interface
- Add `entitlements?: Entitlement[];` to `Purchase` interface

**Modify `ui/.../shared/types/ServerAccess.ts`** — add `getPurchases(): Observable<Purchase[]>;`

**Modify `ui/.../shared/services/network/ServerAccessNetwork.ts`** — implement:
```typescript
getPurchases(): Observable<Purchase[]> {
  return this.http.get<{ purchases: Purchase[] }>('/api/purchases', { withCredentials: true })
    .pipe(map(response => response.purchases));
}
```

**Modify `ui/.../shared/services/network/ServerAccess.mock.ts`** — mock implementation filtering by session user

**Modify `ui/.../shared/services/network/ServerAccess.ts`** (proxy) — add delegation

### F.5: Create PurchaseHistoryComponent

**Create** `ui/.../pages/account/purchase-history/purchase-history.component.ts`:
- Standalone component with SharedModule imports
- Loads purchases on init via `serverAccess.getPurchases()`
- States: loading, empty, loaded, error
- Uses `formatCents()` and `formatTimestamp()` from `payment.types.ts`

**Create** `ui/.../pages/account/purchase-history/purchase-history.component.html`:
- List of mat-expansion-panel per purchase
- Each shows: date, status badge, total, items with product names/prices
- Expandable entitlements section (status, validity dates, seats)
- Empty state with link to `/shop`

**Create** `ui/.../pages/account/purchase-history/purchase-history.component.scss`

### F.6: Add route

**Modify `ui/.../pages/account/account.routes.ts`** — add:
```typescript
{ path: 'purchases', component: PurchaseHistoryComponent }
```
Accessible at `/my/purchases` (account module loads at `/my`).

### F.7: Frontend tests

**Create** `ui/.../pages/account/purchase-history/purchase-history.component.spec.ts`:
- Loading state, empty state, displays purchases with items, displays entitlements, error handling

### Files Summary

| Step | File | Action |
|------|------|--------|
| F.1 | `payment/purchase_helper.h` | Modify |
| F.1 | `payment/purchase_helper.cpp` | Modify |
| F.1 | `payment/payment_key_value_table.cpp` | Modify |
| F.1 | `payment/purchase_helper_test.cpp` | Modify — add 3 tests |
| F.1 | `payment/payment_key_value_table_test.cpp` | Modify — add 2 tests |
| F.2 | `endpoints/purchases.h` | **Create** |
| F.2 | `endpoints/purchases.cpp` | **Create** |
| F.2 | `endpoints/web_app.cpp` | Modify |
| F.2 | `endpoints/CMakeLists.txt` | Modify |
| F.3 | `endpoints/purchases_test.cpp` | **Create** |
| F.4 | `shared/types/payment.types.ts` | Modify |
| F.4 | `shared/types/ServerAccess.ts` | Modify |
| F.4 | `shared/services/network/ServerAccessNetwork.ts` | Modify |
| F.4 | `shared/services/network/ServerAccess.mock.ts` | Modify |
| F.4 | `shared/services/network/ServerAccess.ts` | Modify |
| F.5 | `pages/account/purchase-history/purchase-history.component.ts` | **Create** |
| F.5 | `pages/account/purchase-history/purchase-history.component.html` | **Create** |
| F.5 | `pages/account/purchase-history/purchase-history.component.scss` | **Create** |
| F.6 | `pages/account/account.routes.ts` | Modify |
| F.7 | `pages/account/purchase-history/purchase-history.component.spec.ts` | **Create** |

**Total**: 13 files modified, 7 files created

### Verification

1. **Backend tests**: Build and run all payment tests + `purchases_test.cpp` — all pass
2. **Frontend tests**: `ng test` — all existing + new tests pass
3. **Integration** (`ng serve -c development`):
   - Log in, make a purchase via checkout
   - Navigate to `/my/purchases` — purchase appears with product name, price, date, status, entitlements
   - Log out — `/my/purchases` redirects to login
4. **Edge cases**: User with no purchases sees empty state

## Phase G: Routes & Navigation ✅

With Phases A-F complete, the shopping and purchase history features are functional but disconnected from navigation. Users have no way to discover purchase history from the header menu, the checkout success page only offers "Continue Shopping" with no path to view the receipt, and the account landing page (`/my/account`) is a stub showing raw JSON. Phase G connects all the dots.

### G.1: Add "My Purchases" to header navigation

**Modify: `ui/src/app/shared/services/header/mockHeaderResponse.ts`**
- Insert a "My Purchases" item in the `userDropdown.menu` array, directly after "Profile"
- Uses `HeaderButtonKind.InternalLink` with `goTo: '/my/purchases'`

### G.2: Add "View Purchases" link to checkout success

**Modify: `ui/src/app/pages/shop/checkout/checkout.component.html`**
- In the success state (`state === 'success'`, lines 22-29), add a second link below "Continue Shopping":
  ```html
  <a mat-stroked-button routerLink="/my/purchases" class="din-bold ml-3">
    View My Purchases
  </a>
  ```
- Both buttons sit side by side: "Continue Shopping" (primary/raised) and "View My Purchases" (stroked/outlined)

### G.3: Replace account profile stub with dashboard

The current `UserComponent` at `/my/account` is a stub that shows raw JSON. Replace it with a card-based account dashboard.

**Modify: `ui/src/app/pages/account/profile/user.component.ts`**
- Remove `UserService` dependency (commented-out code), `JSON` property, `userInfo` property
- Add `AuthService` and `Router` injections (same pattern as `UserInformationComponent`)
- Subscribe to `authData$` with `takeUntil(this.destroy$)`
- Add navigation methods: `onUserInfo()`, `onPurchases()`, `onChangePassword()`
- Import `CommonModule`, `MatCardModule`, `MatButtonModule`, `MatIconModule` (drop `SharedModule`)

**Modify: `ui/src/app/pages/account/profile/user.component.html`**
- Welcome heading: "Welcome, {firstName}" (or "My Account" if not auth)
- Three `mat-card` navigation cards in a grid:

| Card | Icon | Title | Description | Route |
|------|------|-------|-------------|-------|
| 1 | `person` | User Information | View and edit your profile | `/my/user-information` |
| 2 | `receipt_long` | Purchase History | View your orders and entitlements | `/my/purchases` |
| 3 | `lock` | Change Password | Update your account password | `/my/update-user-password` |

Each card is clickable (navigates via `router.navigate()`). Test IDs: `#card-user-info`, `#card-purchases`, `#card-password`.

**Modify: `ui/src/app/pages/account/profile/user.component.scss`**
- Dashboard container: `max-width: 800px; margin: 0 auto; padding: 24px 16px`
- Card grid: CSS grid, 1 column on mobile, 3 columns on desktop (`grid-template-columns: repeat(auto-fit, minmax(220px, 1fr))`)
- Card hover effect for clickability feedback

### G.4: Update tests

**Modify: `ui/src/app/pages/account/profile/user.component.spec.ts`**
- Set up with `AuthService`, `ServerAccessMock`, `RouterTestingModule` (same pattern as `user_information.component.spec.ts`)
- Tests:
  1. Displays welcome message with user's first name after login
  2. Clicking user info card navigates to `/my/user-information`
  3. Clicking purchases card navigates to `/my/purchases`
  4. Clicking change password card navigates to `/my/update-user-password`

### Files Summary

| Step | File | Action |
|------|------|--------|
| G.1 | `shared/services/header/mockHeaderResponse.ts` | Modify — add "My Purchases" to user dropdown |
| G.2 | `pages/shop/checkout/checkout.component.html` | Modify — add "View My Purchases" link in success state |
| G.3 | `pages/account/profile/user.component.ts` | Modify — rewrite as dashboard |
| G.3 | `pages/account/profile/user.component.html` | Modify — card-based dashboard layout |
| G.3 | `pages/account/profile/user.component.scss` | Modify — dashboard grid styling |
| G.4 | `pages/account/profile/user.component.spec.ts` | Modify — rewrite tests for dashboard |

**Total**: 6 files modified, 0 files created

### Verification

1. **Frontend tests**: `ng test` — all 254+ tests pass (including new dashboard tests)
2. **Manual test** (`ng serve -c local`):
   - Click user dropdown in header → "My Purchases" link visible, navigates to `/my/purchases`
   - Navigate to `/my/account` → see dashboard with 3 cards and welcome message
   - Click each card → navigates to correct page
   - Complete a checkout → success page shows "View My Purchases" link alongside "Continue Shopping"
   - Click "View My Purchases" → navigates to `/my/purchases`

---

# API Reference

## Request/Response Formats

### GET /api/catalog_products
**Response (200):**
```json
{
  "items": [
    {
      "product": {
        "id": 1, "code": "INTRO_WORKSHOP", "name": "Intro Workshop",
        "description": "A beginner workshop", "kind": "one_time"
      },
      "price": { "amount_cents": 5000, "currency": "USD", "permission_id": null }
    }
  ]
}
```
Note: `ServerAccessNetwork` transforms this nested format into flat `CatalogProduct[]`.

### POST /api/purchase_create
**Request:**
```json
{
  "items": [{"product_id": 1, "quantity": 1}],
  "portal_note": "optional"
}
```
**Response (201):**
```json
{
  "id": 1,
  "payer_person_id": 5,
  "status": "pending",
  "currency": "USD",
  "subtotal_cents": 5000,
  "tax_cents": 0,
  "total_cents": 5000,
  "paid_cents": 0,
  "items": [
    {
      "id": 1, "product_id": 1, "price_schedule_id": 1,
      "quantity": 1, "unit_price_cents": 5000, "line_total_cents": 5000, "currency": "USD"
    }
  ]
}
```

### POST /api/purchase_pay_card/{purchaseId}
**Request:**
```json
{
  "source_id": "cnon:card-nonce-from-square-sdk",
  "idempotency_key": "uuid-here"
}
```
**Response (200):**
```json
{
  "payment": {
    "id": 1, "provider": "square", "provider_payment_id": "PAYMENT_xyz",
    "status": "COMPLETED", "amount_cents": 5000, "currency": "USD",
    "payer_person_id": 5
  },
  "purchase": {
    "id": 1, "status": "funded", "total_cents": 5000, "paid_cents": 5000,
    "items": [...]
  },
  "entitlements": [
    { "id": 1, "purchase_id": 1, "product_id": 1, "status": "active" }
  ]
}
```
Note: `ServerAccessNetwork` transforms `entitlements[]` (array) into `entitlements_created` (count) to match the frontend `PayCardResponse` type.
```

### GET /api/payments
```json
[
  {
    "id": 1,
    "status": "captured",
    "amount_cents": 5000,
    "created_us": 1234567890000000
  }
]
```

---

# Verification

When complete, verify the thin slice works end-to-end:
1. Log in as test user
2. Navigate to catalog/shop page
3. Select Intro Workshop
4. Enter test card (4111 1111 1111 1111)
5. Submit payment
6. See confirmation
7. Navigate to purchase history
8. See completed purchase listed