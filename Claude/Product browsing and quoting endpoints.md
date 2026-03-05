---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 2/3/2026
Version: 0.1
tags: 
---
# Overview

In the planning directory, there is Product Design Document.md. I am trying to do the scenario 8.1.1 User purchases a one time item for themself like an intro workshop or massage. Using this document and the code base, let's start putting together a plan to accomplish this work item. Let's enumerate the tables that will be needed, the features that will be used (like secrets, email, square client, etc). I like how I layered the authentication type work flows and have the db_schema with the tables at the bottom, table helpers that are CRUD wrappers for the tables a layer up, core services like email, secrets, etc at a similar level to the table helpers, a business layer like PersonHelper, CookieManager, Session, and auth helper as a business logic layer, and then the endpoints. I would like something similar for payments but feel like we should have a sister directory to auth (maybe called payment?) that wraps the table helpers, email, secret, and square API calls in something parallel to PersonHelper (maybe PaymentHelper?). Please go into plan mode and use this document to generate the in progress plan. Don't touch this overview but feel free to use everything below Overview as a scratch pad. I plan on doing several iterations building this document in plan mode before moving to implementing code. Please start with a high level description of what we are doing and what tables and components will be involved.

# High-Level Description

## What We Are Building

**Scenario 8.1.1**: A logged-in user can browse the product catalog, select a one-time item (e.g., intro workshop, massage), pay with a credit card, and receive an entitlement granting them access to the purchased service.

This is the "thin slice" that establishes the core payment flow. The user journey:
1. Browse catalog → see products with resolved prices
2. Select product and quantity → create purchase with server-side pricing
3. Enter card details → client-side tokenization via Square Web Payments SDK
4. Submit payment → server charges via Square API, creates entitlement
5. View confirmation → receipt shown in portal

## Architecture Overview

Following the existing auth layer pattern, we will create a parallel `payment/` directory with similar layering:

```
Layer 4: Endpoints
├── catalog_products.cpp      (GET /api/catalog_products)
├── catalog_quote.cpp         (POST /api/catalog_quote)
├── purchase_create.cpp       (POST /api/purchase_create)
├── purchase_pay_card.cpp     (POST /api/purchase_pay_card/{id})
└── payments.cpp              (GET /api/payments)

Layer 3: Business Logic (payment/)
├── payment_helper.h/cpp      - Orchestrates payment flow (like PersonHelper)
├── purchase_helper.h/cpp     - Purchase creation and pricing resolution
├── entitlement_helper.h/cpp  - Entitlement creation and assignment
└── catalog_helper.h/cpp      - Product catalog and price resolution

Layer 2: Core Services
├── square/square_client.h    - Square API wrapper (already exists)
├── secrets/                  - Configuration (already exists)
├── util/mail/                - Email sending (already exists)
└── idempotency_helper.h/cpp  - Idempotency key management

Layer 1: Table Helpers (sql_util/table_helpers/, namespace TableHelpers::)
├── Products                  (already exists)
├── PriceSchedules            (already exists)
├── ProductPrices             (already exists)
├── Purchases                 (already exists)
├── PurchaseItems             (already exists)
├── Payments                  (already exists)
├── PurchasePayments          (already exists)
├── ProductEntitlementRules   (already exists)
├── Entitlements              (already exists)
├── EntitlementAssignments    (already exists)
└── IdempotencyKeys           (needs to be created)

Layer 0: Database Schema (db_schema/)
├── products.h/cpp                  (already exists)
├── price_schedules.h/cpp           (already exists)
├── product_prices.h/cpp            (already exists)
├── purchases.h/cpp                 (already exists)
├── purchase_items.h/cpp            (already exists)
├── payments.h/cpp                  (already exists)
├── purchase_payments.h/cpp         (already exists)
├── product_entitlement_rules.h/cpp (already exists)
├── entitlements.h/cpp              (already exists)
├── entitlement_assignments.h/cpp   (already exists)
└── idempotency_keys.h/cpp          (needs to be created)
```

---

# Tables Involved

## Already Exist (schema + table helpers implemented)

### Products and Pricing
| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `products` | Catalog of purchasable items | id, code, name, description, kind, is_active |
| `price_schedules` | Time-bounded price sets | id, name, valid_from_us, valid_to_us, is_active |
| `product_prices` | Price per product per schedule | product_id, price_schedule_id, permission_id, amount_cents |
| `product_entitlement_rules` | What access a product grants | product_id, grants_permission_id, seats_default, validity_kind |

### Purchases and Payments
| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `purchases` | Shopping cart / order record | id, payer_person_id, status, total_cents, paid_cents |
| `purchase_items` | Line items in a purchase | purchase_id, product_id, quantity, unit_price_cents |
| `payments` | Individual payment transactions | id, provider, status, amount_cents, payer_person_id |
| `purchase_payments` | Links payments to purchases (many-to-many) | purchase_id, payment_id, applied_cents |

### Entitlements
| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `entitlements` | Access grants from funded purchases | purchase_id, product_id, valid_from_us, valid_to_us, seats_total, status |
| `entitlement_assignments` | Who is assigned to each entitlement seat | entitlement_id, person_id, removed_us |

## Need to Create

### Infrastructure
| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `idempotency_keys` | Prevents duplicate operations | scope, key, endpoint, status, response_json, expires_us |

---

# Components Involved

## Already Implemented

### Core Infrastructure
| Component | Location | Purpose |
|-----------|----------|---------|
| `SquareClient` | `square/square_client.h` | Square Payments API wrapper |
| `HttpClient` | `util/http/http_client.h` | libcurl HTTP wrapper |
| `SecretsHelper` | `secrets/secrets_helper.h` | Configuration/credentials |
| `MailHelper` | `util/mail/` | Email sending |
| `TransactionProvider` | `sql_util/database_access/` | Database transactions |
| `EndpointAuthHelper` | `auth/auth_helper.h` | Session/auth context for endpoints |
| `ErrorResponse` | `util/error_response.h` | RFC 7807 JSON error responses |
| Square Web Payments SDK | `ui/` (Angular) | Client-side card tokenization |

### Table Helpers (all in `TableHelpers::` namespace)
| Class | File | Purpose |
|-------|------|---------|
| `Products` | `sql_util/table_helpers/products.h` | CRUD for products |
| `PriceSchedules` | `sql_util/table_helpers/price_schedules.h` | CRUD for price_schedules |
| `ProductPrices` | `sql_util/table_helpers/product_prices.h` | CRUD for product_prices |
| `Purchases` | `sql_util/table_helpers/purchases.h` | CRUD for purchases |
| `PurchaseItems` | `sql_util/table_helpers/purchase_items.h` | CRUD for purchase_items |
| `Payments` | `sql_util/table_helpers/payments.h` | CRUD for payments |
| `PurchasePayments` | `sql_util/table_helpers/purchase_payments.h` | CRUD for purchase_payments |
| `ProductEntitlementRules` | `sql_util/table_helpers/product_entitlement_rules.h` | CRUD for entitlement rules |
| `Entitlements` | `sql_util/table_helpers/entitlements.h` | CRUD for entitlements |
| `EntitlementAssignments` | `sql_util/table_helpers/entitlement_assignments.h` | CRUD for assignments |

## Need to Create

### Business Logic Layer (`payment/`)
| Component | Purpose |
|-----------|---------|
| `PaymentHelper` | Orchestrates the full payment flow: create payment → link to purchase → create entitlement |
| `PurchaseHelper` | Creates purchases with server-resolved pricing, manages purchase lifecycle |
| `EntitlementHelper` | Creates entitlements from funded purchases, manages seat assignments |
| `CatalogHelper` | Resolves product catalog with permission-based pricing |
| `IdempotencyHelper` | Manages idempotency keys, prevents duplicate charges |

### Table Helper (needs to be created)
| Class | Purpose |
|-------|---------|
| `TableHelpers::IdempotencyKeys` | CRUD for idempotency_keys table |

### Endpoints (`endpoints/`)
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/catalog_products` | GET | Return products visible to current user with resolved prices |
| `/api/catalog_quote` | POST | Preview pricing for requested items (read-only) |
| `/api/purchase_create` | POST | Create purchase with server-resolved pricing |
| `/api/purchase_pay_card/{id}` | POST | Charge card via Square, create entitlement |
| `/api/payments` | GET | Payment history for current user |

---

# Key Design Decisions

## 1. Server-Side Price Resolution
Prices are **always** resolved on the server at purchase creation time. The client never sends prices. This:
- Prevents price manipulation
- Enables permission-based pricing
- Creates an audit trail via `purchase_items.pricing_permission_id`

## 2. Idempotency for Payment Safety
All payment operations use idempotency keys to prevent double-charging:
- Client generates unique key per payment attempt
- Server checks `idempotency_keys` table before calling Square
- On retry, returns cached response instead of re-charging

## 3. Immutable Records
Payments and entitlements are never mutated:
- Refunds create new payment records with negative amounts
- Entitlement changes use `revoked_us` or create new records
- Complete audit trail preserved

## 4. PaymentHelper as Orchestrator
`PaymentHelper` is the single entry point for payment operations, similar to how `PersonHelper` orchestrates auth flows:
```cpp
// PaymentHelper composes:
// - SquareClient (external API)
// - IdempotencyHelper (safety)
// - PurchasesTableHelper, PaymentsTableHelper (persistence)
// - EntitlementHelper (fulfillment)
// - MailHelper (notifications)
```

---

# Implementation Order

## Phase 1: Idempotency Schema and Table Helper
- [x] Create `idempotency_keys` db_schema (`db_schema/idempotency_keys.h/cpp`)
- [x] Create `TableHelpers::IdempotencyKeys` table helper
- [x] Unit tests for `TableHelpers::IdempotencyKeys`

## Phase 2: Business Logic Helpers
- [x] Create `IdempotencyHelper` (`payment/idempotency_helper.h/cpp`)
- [x] Unit tests for `IdempotencyHelper`
- [x] Create `CatalogHelper` with price resolution logic (`payment/catalog_helper.h/cpp`)
- [x] Unit tests for `CatalogHelper`
- [x] Create `PurchaseHelper` (`payment/purchase_helper.h/cpp`)
- [x] Unit tests for `PurchaseHelper`
- [x] Create `EntitlementHelper` (`payment/entitlement_helper.h/cpp`)
- [x] Unit tests for `EntitlementHelper`
- [x] Create `PaymentHelper` orchestrator (`payment/payment_helper.h/cpp`)
- [x] Unit tests for `PaymentHelper`

## Phase 3: Catalog Endpoints
- [x] Create `/api/catalog_products` endpoint
- [x] Create `/api/catalog_quote` endpoint
- [x] Unit tests for catalog endpoints

## Phase 4: Purchase Endpoint
- [x] Create `/api/purchase_create` endpoint
- [x] Unit tests for purchase endpoint

## Phase 5: Payment Endpoint
- [x] Create `/api/purchase_pay_card/{id}` endpoint
- [x] Unit tests for payment endpoint (integration tests require Square sandbox setup)

## Phase 6: History Endpoint
- [x] Create `/api/payments` endpoint ✅ 2026-02-06
- [x] Unit tests for payments endpoint ✅ 2026-02-06

## Phase 7: Polish
- [x] Payment confirmation email ✅ 2026-02-06
- [x] End-to-end testing ✅ 2026-02-06

**Note**: Most schema and table helper work is already complete. The primary effort is in the business logic layer (`payment/` directory) and endpoints.

---

# Open Questions

1. **Tax handling**: Should we support tax calculation in Phase 1, or always use `tax_cents = 0`?
   - Recommendation: Start with `tax_cents = 0`, add later if needed
   - I will go with your recommendation.

2. **Price schedule selection**: What if multiple schedules are active? Use most recently created?
   - Recommendation: Use `valid_from_us DESC LIMIT 1` for active schedules
   - I plan on setting things up so that I set the valid to to be the same as the valid_from_us so this doesn't happen but let's say that only one (the most recent one or the one with the later valid_from_us) is active at a time.

3. **Entitlement validity**: For one-time purchases, what is `validity_kind`?
   - Options: "instant" (valid immediately, no expiry), or "days_from_activation" with configurable days
   - Recommendation: Start with "instant" for simplicity
   - I'll go with your recommendation for now.

4. **Partial payments**: Should Phase 1 support partial payments?
   - Recommendation: No. Phase 1 requires full payment in single transaction.
   - Let's go with full payment for now.

---

# Next Steps

Once this high-level plan is approved, we will:
1. Detail the database schema (column types, constraints, indexes)
2. Define the table helper interfaces
3. Design the business layer APIs
4. Specify the endpoint request/response formats