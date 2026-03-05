---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 1/28/2026
Version: 0.1
tags: 
---
# Overview

In the document in this directory called Payment base SQL Tables.md, we created the following tables:

products, price_schedules, product_prices, product_entitlement_rules, purchases, purchase_items, payments, purchase_payments, entitlements, entitlement_assignments.

Inside sql_util/table_helpers, there are CRUD C++ class wrappers for many important tables. I'd like to create wrappers for all of these new payment tables. Please look at how I named each of the wrappers for the h/cpp/test.cpp and do the same for these classes with the same formatting, namespaces, etc. Please note the reasons that I added more than one of the various C R U D methods for any given helper and look for reasons to do the same here as well. Please follow similar patterns. Also, for things where things can be null, please add either separate methods and default parameters. Please note the order in which I create dependent tables and stored procedures for test code. You already noted the dependency order for creating the various table definitions so please do the same ordering in creating tables for testing.

I would like you to put together a plan for this first and not jump to making code changes until I tell you. Please stay in plan mode but use this document after the overview section to place your design. Please put an open questions / issues section at the end for me to comment on and iterate. I'd like to see full class signatures with all the methods. Where you do more than a simple CRUD operator, please call that out and note why. Please give me open questions if you feel like there is an option to make something easier to use and call.

For tests, I want the positive functionality and separate tests for edge and error cases. All explicit error and exception paths should be handled.

# Patterns Observed

From existing table helpers in `sql_util/table_helpers/`:

| Aspect | Pattern |
|--------|---------|
| **File naming** | `{table_name}.h`, `{table_name}.cpp`, `{table_name}_test.cpp` |
| **Class naming** | PascalCase (e.g., `People`, `Roles`, `DeviceTokens`) |
| **Namespace** | `TableHelpers` |
| **Constructor** | `explicit ClassName(DatabaseHelper databaseHelper);` |
| **Create** | Returns `int64_t` PK via `DbCrud::AddRowToTableFetchInt64PrimaryKey` |
| **Read single** | Returns `KeyValueTable` (empty if not found for simple lookups) |
| **Read array** | Returns `KeyValueTableArray` (empty array if no matches) |
| **Update** | Separate methods per field (e.g., `SetName`, `SetStatus`) |
| **Delete** | By primary key, no-op if not found |
| **Nullable fields** | Check if value is empty string, no separate methods |
| **Updated timestamp** | Set to `DbSchema::kDatabaseInfoDefaultNow` on updates |

**When multiple read methods are added:**
- Lookup by unique column (e.g., `GetProductByCode`)
- Lookup by FK returning array (e.g., `GetPurchaseItemsByPurchaseId`)
- Business queries (e.g., `GetActiveProducts`, `GetPriceSchedulesValidAt`)

---

# Files to Create

## Table Helpers
Location: `server/knottyyoga_server/src/sql_util/table_helpers/`

| Table | Header | Implementation | Test |
|-------|--------|----------------|------|
| products | `products.h` | `products.cpp` | `products_test.cpp` |
| price_schedules | `price_schedules.h` | `price_schedules.cpp` | `price_schedules_test.cpp` |
| product_prices | `product_prices.h` | `product_prices.cpp` | `product_prices_test.cpp` |
| product_entitlement_rules | `product_entitlement_rules.h` | `product_entitlement_rules.cpp` | `product_entitlement_rules_test.cpp` |
| purchases | `purchases.h` | `purchases.cpp` | `purchases_test.cpp` |
| purchase_items | `purchase_items.h` | `purchase_items.cpp` | `purchase_items_test.cpp` |
| payments | `payments.h` | `payments.cpp` | `payments_test.cpp` |
| purchase_payments | `purchase_payments.h` | `purchase_payments.cpp` | `purchase_payments_test.cpp` |
| entitlements | `entitlements.h` | `entitlements.cpp` | `entitlements_test.cpp` |
| entitlement_assignments | `entitlement_assignments.h` | `entitlement_assignments.cpp` | `entitlement_assignments_test.cpp` |

## Shared Test Utilities
Location: `server/knottyyoga_server/test/src/util/`

| Purpose | Header | Implementation |
|---------|--------|----------------|
| Payment table test setup | `payment_table_test_helper.h` | `payment_table_test_helper.cpp` |

**Total: 32 files** (20 table helper files + 10 test files + 2 shared test helper files)

---

# Class Signatures

## 1. Products

```cpp
namespace TableHelpers {

class Products {
public:
    explicit Products(DatabaseHelper databaseHelper);
    Products(const Products&) = default;
    Products& operator=(const Products&) = default;
    ~Products() = default;

    // CREATE - isActive defaults to true
    int64_t AddProduct(
        Transaction& transaction,
        std::string_view code,
        std::string_view name,
        std::string_view description,
        std::string_view kind,
        bool isActive = true);

    // READ - Sorted by: name ASC, id ASC
    KeyValueTable GetProduct(Transaction& transaction, int64_t id) const;
    KeyValueTable GetProductByCode(Transaction& transaction, std::string_view code) const;
    KeyValueTableArray GetProducts(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetActiveProducts(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetProductsByKind(
        Transaction& transaction,
        std::string_view kind,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // UPDATE
    void SetName(Transaction& transaction, int64_t id, std::string_view name);
    void SetDescription(Transaction& transaction, int64_t id, std::string_view description);
    void SetKind(Transaction& transaction, int64_t id, std::string_view kind);
    void SetIsActive(Transaction& transaction, int64_t id, bool isActive);

    // DELETE
    void DeleteProduct(Transaction& transaction, int64_t id);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetProductByCode`: Products referenced by code in URLs/APIs rather than numeric IDs
- `GetActiveProducts`: Checkout flow shows only purchasable items
- `GetProductsByKind`: Filter products by type ("membership", "class_pack", etc.)
- `GetProducts` with pagination: Supports admin UI with large product catalogs

---

## 2. PriceSchedules

```cpp
namespace TableHelpers {

class PriceSchedules {
public:
    explicit PriceSchedules(DatabaseHelper databaseHelper);
    PriceSchedules(const PriceSchedules&) = default;
    PriceSchedules& operator=(const PriceSchedules&) = default;
    ~PriceSchedules() = default;

    // CREATE - validToUs is optional (std::nullopt = no end date), isActive defaults to true
    int64_t AddPriceSchedule(
        Transaction& transaction,
        std::string_view name,
        int64_t validFromUs,
        std::optional<int64_t> validToUs = std::nullopt,
        bool isActive = true);

    // READ - Sorted by: is_active DESC, valid_to_us ASC NULLS FIRST, id ASC
    KeyValueTable GetPriceSchedule(Transaction& transaction, int64_t id) const;
    KeyValueTableArray GetPriceSchedules(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetActivePriceSchedules(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPriceSchedulesValidAt(
        Transaction& transaction,
        int64_t timestampUs,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // UPDATE
    void SetName(Transaction& transaction, int64_t id, std::string_view name);
    void SetValidFromUs(Transaction& transaction, int64_t id, int64_t validFromUs);
    void SetValidToUs(Transaction& transaction, int64_t id, std::optional<int64_t> validToUs);
    void SetIsActive(Transaction& transaction, int64_t id, bool isActive);

    // DELETE
    void DeletePriceSchedule(Transaction& transaction, int64_t id);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetActivePriceSchedules`: Admin UI dropdown for selecting schedules
- `GetPriceSchedulesValidAt`: Checkout - determine applicable pricing periods at purchase time
- Sorting order: Active schedules first, then by validToUs (NULL = no end = first, then most recent end dates)

---

## 3. ProductPrices

```cpp
namespace TableHelpers {

class ProductPrices {
public:
    explicit ProductPrices(DatabaseHelper databaseHelper);
    ProductPrices(const ProductPrices&) = default;
    ProductPrices& operator=(const ProductPrices&) = default;
    ~ProductPrices() = default;

    // CREATE - permissionId is optional (std::nullopt = standard price for all users)
    int64_t AddProductPrice(
        Transaction& transaction,
        int64_t productId,
        int64_t priceScheduleId,
        std::optional<int64_t> permissionId,
        std::string_view currency,
        int64_t amountCents);

    // READ - All paginated methods sorted by: updated_us DESC, created_us DESC, id ASC
    // Pass non-null totalCount to get total matching rows (for pagination UI)
    KeyValueTable GetProductPrice(Transaction& transaction, int64_t id) const;
    KeyValueTableArray GetProductPrices(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetProductPricesByProductId(
        Transaction& transaction,
        int64_t productId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetProductPricesByPriceScheduleId(
        Transaction& transaction,
        int64_t priceScheduleId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // Returns exact match for permissionId, or falls back to NULL permission if no match
    // Priority: specific permission match > NULL permission (fallback)
    KeyValueTable GetProductPriceByProductSchedulePermission(
        Transaction& transaction,
        int64_t productId,
        int64_t priceScheduleId,
        std::optional<int64_t> permissionId) const;

    // Returns lowest price among matching permissions, or NULL permission if none match
    // Use case: User has multiple permissions, find best available price
    KeyValueTable GetBestProductPriceByProductSchedulePermissions(
        Transaction& transaction,
        int64_t productId,
        int64_t priceScheduleId,
        const std::vector<int64_t>& permissionIds) const;

    // UPDATE
    void SetCurrency(Transaction& transaction, int64_t id, std::string_view currency);
    void SetAmountCents(Transaction& transaction, int64_t id, int64_t amountCents);

    // DELETE
    void DeleteProductPrice(Transaction& transaction, int64_t id);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetProductPricesByProductId`: Admin view of all prices for a product
- `GetProductPricesByPriceScheduleId`: Admin view of all prices in a schedule
- `GetProductPriceByProductSchedulePermission`: Finds exact permission match, falls back to NULL permission (standard price) if no match
- `GetBestProductPriceByProductSchedulePermissions`: Given a user's list of permissions, returns the lowest priced match (user may have multiple roles/permissions, we want to give them the best price)
- Count methods: Separate methods for total count to support UI pagination display ("showing 1-10 of 247")

---

## 4. ProductEntitlementRules

```cpp
namespace TableHelpers {

class ProductEntitlementRules {
public:
    explicit ProductEntitlementRules(DatabaseHelper databaseHelper);
    ProductEntitlementRules(const ProductEntitlementRules&) = default;
    ProductEntitlementRules& operator=(const ProductEntitlementRules&) = default;
    ~ProductEntitlementRules() = default;

    // CREATE - grantsPermissionId and validityDays are nullable
    int64_t AddProductEntitlementRule(
        Transaction& transaction,
        int64_t productId,
        std::optional<int64_t> grantsPermissionId,
        int64_t seatsDefault,
        std::string_view validityKind,
        std::optional<int64_t> validityDays = std::nullopt);

    // READ - Note: 1:1 mapping of productId to rule (unique constraint)
    // Sorted by: updated_us DESC, created_us DESC, id ASC
    KeyValueTable GetProductEntitlementRule(Transaction& transaction, int64_t id) const;
    KeyValueTable GetProductEntitlementRuleByProductId(Transaction& transaction, int64_t productId) const;
    KeyValueTableArray GetProductEntitlementRules(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // UPDATE
    void SetGrantsPermissionId(Transaction& transaction, int64_t id, std::optional<int64_t> grantsPermissionId);
    void SetSeatsDefault(Transaction& transaction, int64_t id, int64_t seatsDefault);
    void SetValidityKind(Transaction& transaction, int64_t id, std::string_view validityKind);
    void SetValidityDays(Transaction& transaction, int64_t id, std::optional<int64_t> validityDays);

    // DELETE
    void DeleteProductEntitlementRule(Transaction& transaction, int64_t id);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetProductEntitlementRuleByProductId`: Core fulfillment - after purchase, look up what entitlement to create. Returns `KeyValueTable` because there's a 1:1 mapping (unique constraint on product_id ensures one rule per product).

---

## 5. Purchases

```cpp
namespace TableHelpers {

class Purchases {
public:
    explicit Purchases(DatabaseHelper databaseHelper);
    Purchases(const Purchases&) = default;
    Purchases& operator=(const Purchases&) = default;
    ~Purchases() = default;

    // CREATE - portalNote is nullable
    int64_t AddPurchase(
        Transaction& transaction,
        int64_t payerPersonId,
        std::string_view status,
        std::string_view currency,
        int64_t subtotalCents,
        int64_t taxCents,
        int64_t totalCents,
        int64_t paidCents,
        std::string_view portalNote);  // empty = NULL

    // READ - All paginated methods sorted by: updated_us DESC, created_us DESC, id ASC
    KeyValueTable GetPurchase(Transaction& transaction, int64_t id) const;
    KeyValueTableArray GetPurchases(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPurchasesByPayerPersonId(
        Transaction& transaction,
        int64_t payerPersonId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPurchasesByStatus(
        Transaction& transaction,
        std::string_view status,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // UPDATE
    void SetStatus(Transaction& transaction, int64_t id, std::string_view status);
    void SetPaidCents(Transaction& transaction, int64_t id, int64_t paidCents);
    void SetPortalNote(Transaction& transaction, int64_t id, std::string_view portalNote);
    void SetTotals(
        Transaction& transaction,
        int64_t id,
        int64_t subtotalCents,
        int64_t taxCents,
        int64_t totalCents);

    // DELETE
    void DeletePurchase(Transaction& transaction, int64_t id);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetPurchasesByPayerPersonId`: User portal - "My Orders" view (with pagination and count)
- `GetPurchasesByStatus`: Admin - find pending/completed/cancelled orders (with pagination and count)
- `SetTotals`: Atomically update related amount fields when recalculating

**Note:** Removed `GetMostRecentPurchasesByPayerPersonId` as it's now redundant - `GetPurchasesByPayerPersonId` already sorts by `updated_us DESC` and supports pagination.

---

## 6. PurchaseItems

```cpp
namespace TableHelpers {

class PurchaseItems {
public:
    explicit PurchaseItems(DatabaseHelper databaseHelper);
    PurchaseItems(const PurchaseItems&) = default;
    PurchaseItems& operator=(const PurchaseItems&) = default;
    ~PurchaseItems() = default;

    // CREATE - pricingPermissionId is nullable
    // Note: lineTotalCents = quantity × unitPriceCents (caller computes)
    int64_t AddPurchaseItem(
        Transaction& transaction,
        int64_t purchaseId,
        int64_t productId,
        int64_t priceScheduleId,
        std::optional<int64_t> pricingPermissionId,
        int64_t quantity,
        int64_t unitPriceCents,
        int64_t lineTotalCents,
        std::string_view currency);

    // READ - Sorted by: created_us DESC, id ASC (no updated_us on this table)
    KeyValueTable GetPurchaseItem(Transaction& transaction, int64_t id) const;
    KeyValueTableArray GetPurchaseItems(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPurchaseItemsByPurchaseId(
        Transaction& transaction,
        int64_t purchaseId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPurchaseItemsByProductId(
        Transaction& transaction,
        int64_t productId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // UPDATE
    void SetQuantity(Transaction& transaction, int64_t id, int64_t quantity);
    void SetLineTotalCents(Transaction& transaction, int64_t id, int64_t lineTotalCents);

    // DELETE
    void DeletePurchaseItem(Transaction& transaction, int64_t id);
    void DeletePurchaseItemsByPurchaseId(Transaction& transaction, int64_t purchaseId);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetPurchaseItemsByPurchaseId`: Order receipt - show all line items
- `GetPurchaseItemsByProductId`: Reporting - which orders contain a specific product
- `DeletePurchaseItemsByPurchaseId`: Clear cart / cancel order in bulk

**Note:** `lineTotalCents` is indeed `quantity × unitPriceCents`. It's stored explicitly for audit/reporting purposes and to preserve the computed value at time of purchase.

---

## 7. Payments

```cpp
namespace TableHelpers {

class Payments {
public:
    explicit Payments(DatabaseHelper databaseHelper);
    Payments(const Payments&) = default;
    Payments& operator=(const Payments&) = default;
    ~Payments() = default;

    // CREATE - Many nullable fields
    int64_t AddPayment(
        Transaction& transaction,
        std::string_view provider,
        std::string_view providerPaymentId,
        std::string_view status,
        std::string_view currency,
        int64_t amountCents,
        int64_t coveredAmountCents,
        int64_t payerPersonId,
        std::string_view portalNote,               // empty = NULL
        std::optional<int64_t> refundForPaymentId, // std::nullopt for non-refunds
        std::string_view refundReason,             // empty = NULL
        std::string_view rawProviderJson);         // empty = NULL

    // READ - All paginated methods sorted by: updated_us DESC, created_us DESC, id ASC
    KeyValueTable GetPayment(Transaction& transaction, int64_t id) const;
    KeyValueTable GetPaymentByProviderAndProviderId(
        Transaction& transaction,
        std::string_view provider,
        std::string_view providerPaymentId) const;
    KeyValueTableArray GetPayments(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPaymentsByPayerPersonId(
        Transaction& transaction,
        int64_t payerPersonId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPaymentsByStatus(
        Transaction& transaction,
        std::string_view status,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetRefundsForPaymentId(
        Transaction& transaction,
        int64_t refundForPaymentId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // UPDATE
    void SetStatus(Transaction& transaction, int64_t id, std::string_view status);
    void SetCoveredAmountCents(Transaction& transaction, int64_t id, int64_t coveredAmountCents);
    void SetPortalNote(Transaction& transaction, int64_t id, std::string_view portalNote);
    void SetRawProviderJson(Transaction& transaction, int64_t id, std::string_view rawProviderJson);
    // Atomically update amount and covered amount together
    void SetAmounts(
        Transaction& transaction,
        int64_t id,
        int64_t amountCents,
        int64_t coveredAmountCents);

    // DELETE
    void DeletePayment(Transaction& transaction, int64_t id);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetPaymentByProviderAndProviderId`: Webhook idempotency - check if payment already recorded
- `GetPaymentsByPayerPersonId`: User portal - "My Payments" view
- `GetPaymentsByStatus`: Admin - find failed/pending payments
- `GetRefundsForPaymentId`: View all refunds issued against a parent payment
- `SetAmounts`: Atomically update amountCents and coveredAmountCents together (e.g., when payment amount changes)

---

## 8. PurchasePayments

```cpp
namespace TableHelpers {

class PurchasePayments {
public:
    explicit PurchasePayments(DatabaseHelper databaseHelper);
    PurchasePayments(const PurchasePayments&) = default;
    PurchasePayments& operator=(const PurchasePayments&) = default;
    ~PurchasePayments() = default;

    // CREATE
    int64_t AddPurchasePayment(
        Transaction& transaction,
        int64_t purchaseId,
        int64_t paymentId,
        int64_t appliedCents);

    // READ - Sorted by: created_us DESC, id ASC (no updated_us on this table)
    KeyValueTable GetPurchasePayment(Transaction& transaction, int64_t id) const;
    KeyValueTable GetPurchasePaymentByPurchaseAndPayment(
        Transaction& transaction,
        int64_t purchaseId,
        int64_t paymentId) const;
    KeyValueTableArray GetPurchasePayments(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPurchasePaymentsByPurchaseId(
        Transaction& transaction,
        int64_t purchaseId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetPurchasePaymentsByPaymentId(
        Transaction& transaction,
        int64_t paymentId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // UPDATE
    void SetAppliedCents(Transaction& transaction, int64_t id, int64_t appliedCents);

    // DELETE
    void DeletePurchasePayment(Transaction& transaction, int64_t id);
    void DeletePurchasePaymentsByPurchaseId(Transaction& transaction, int64_t purchaseId);
    void DeletePurchasePaymentsByPaymentId(Transaction& transaction, int64_t paymentId);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetPurchasePaymentByPurchaseAndPayment`: Check if payment already linked to purchase (avoid double-application)
- `GetPurchasePaymentsByPurchaseId`: Order detail - show all payments applied
- `GetPurchasePaymentsByPaymentId`: Payment detail - show how payment was split across orders
- `DeletePurchasePaymentsByPurchaseId`: Bulk delete when canceling a purchase
- `DeletePurchasePaymentsByPaymentId`: Bulk delete when voiding a payment

---

## 9. Entitlements

```cpp
namespace TableHelpers {

class Entitlements {
public:
    explicit Entitlements(DatabaseHelper databaseHelper);
    Entitlements(const Entitlements&) = default;
    Entitlements& operator=(const Entitlements&) = default;
    ~Entitlements() = default;

    // CREATE - purchaseItemId, notes are nullable
    // When purchaseItemId is std::nullopt, this is a "manual entitlement" (see rationale)
    int64_t AddEntitlement(
        Transaction& transaction,
        int64_t purchaseId,
        std::optional<int64_t> purchaseItemId,
        int64_t productId,
        int64_t validFromUs,
        int64_t validToUs,
        int64_t seatsTotal,
        std::string_view status,
        std::string_view notes);           // empty = NULL

    // READ - All paginated methods sorted by: updated_us DESC, created_us DESC, id ASC
    KeyValueTable GetEntitlement(Transaction& transaction, int64_t id) const;
    KeyValueTableArray GetEntitlements(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetEntitlementsByPurchaseId(
        Transaction& transaction,
        int64_t purchaseId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetEntitlementsByProductId(
        Transaction& transaction,
        int64_t productId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetEntitlementsByStatus(
        Transaction& transaction,
        std::string_view status,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetActiveEntitlementsValidAt(
        Transaction& transaction,
        int64_t timestampUs,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    // For notification/cleanup jobs - find entitlements expiring before a timestamp
    // Sorted by valid_to_us ASC (soonest expiring first)
    KeyValueTableArray GetEntitlementsExpiringBefore(
        Transaction& transaction,
        int64_t timestampUs,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // UPDATE
    void SetStatus(Transaction& transaction, int64_t id, std::string_view status);
    void SetSeatsTotal(Transaction& transaction, int64_t id, int64_t seatsTotal);
    void SetValidToUs(Transaction& transaction, int64_t id, int64_t validToUs);
    void SetNotes(Transaction& transaction, int64_t id, std::string_view notes);
    void RevokeEntitlement(
        Transaction& transaction,
        int64_t id,
        int64_t revokedUs,
        std::string_view revokedReason);

    // DELETE
    void DeleteEntitlement(Transaction& transaction, int64_t id);
    void DeleteEntitlementsByPurchaseId(Transaction& transaction, int64_t purchaseId);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetEntitlementsByPurchaseId`: Order fulfillment - show what was granted
- `GetEntitlementsByProductId`: Reporting - all users with a specific entitlement type
- `GetEntitlementsByStatus`: Admin - find active vs expired entitlements
- `GetActiveEntitlementsValidAt`: Core access control - what entitlements are active now (filters by status='active' AND time range)
- `GetEntitlementsExpiringBefore`: For scheduled jobs that send expiration reminders or clean up expired entitlements (sorted by soonest expiring first)
- `RevokeEntitlement`: Business operation - sets status, revoked_us, and revoked_reason atomically
- `DeleteEntitlementsByPurchaseId`: Bulk delete when canceling a purchase

**Manual Entitlements:** When `purchaseItemId` is `std::nullopt`, the entitlement is "manual" - granted by an admin rather than through the normal purchase flow. Use cases:
- Complimentary access (gift, comp, promo code)
- Migrating legacy members from old system
- Resolving customer service issues ("here's a free month")
- Testing/demo accounts

The `purchaseId` is still required to maintain audit trail, but the entitlement isn't tied to a specific line item.

---

## 10. EntitlementAssignments

```cpp
namespace TableHelpers {

class EntitlementAssignments {
public:
    explicit EntitlementAssignments(DatabaseHelper databaseHelper);
    EntitlementAssignments(const EntitlementAssignments&) = default;
    EntitlementAssignments& operator=(const EntitlementAssignments&) = default;
    ~EntitlementAssignments() = default;

    // CREATE - Validates seat count doesn't exceed seatsTotal
    // Throws std::invalid_argument if no seats available
    int64_t AddEntitlementAssignment(
        Transaction& transaction,
        int64_t entitlementId,
        int64_t personId);

    // READ - Sorted by: created_us DESC, id ASC (no updated_us on this table)
    KeyValueTable GetEntitlementAssignment(Transaction& transaction, int64_t id) const;
    KeyValueTable GetEntitlementAssignmentByEntitlementAndPerson(
        Transaction& transaction,
        int64_t entitlementId,
        int64_t personId) const;
    KeyValueTableArray GetEntitlementAssignments(
        Transaction& transaction,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetEntitlementAssignmentsByEntitlementId(
        Transaction& transaction,
        int64_t entitlementId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetEntitlementAssignmentsByPersonId(
        Transaction& transaction,
        int64_t personId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;
    KeyValueTableArray GetActiveEntitlementAssignmentsByPersonId(
        Transaction& transaction,
        int64_t personId,
        int64_t pageSize = 0,
        int64_t pageOffset = 0,
        int64_t* totalCount = nullptr) const;

    // Helper to check seat availability
    int64_t GetActiveAssignmentCount(Transaction& transaction, int64_t entitlementId) const;

    // UPDATE - Soft delete via removed_us/removed_reason
    // Uses now_us() for removed_us timestamp
    void RemoveEntitlementAssignment(
        Transaction& transaction,
        int64_t id,
        std::string_view removedReason);

    // DELETE - Hard delete
    void DeleteEntitlementAssignment(Transaction& transaction, int64_t id);
    void DeleteEntitlementAssignmentsByEntitlementId(Transaction& transaction, int64_t entitlementId);

private:
    DatabaseHelper databaseHelper_;
};

} // namespace TableHelpers
```

**Additional Methods Rationale:**
- `GetEntitlementAssignmentByEntitlementAndPerson`: Check if person already has this entitlement assigned
- `GetEntitlementAssignmentsByEntitlementId`: View who is using an entitlement's seats
- `GetEntitlementAssignmentsByPersonId`: User portal - "My Memberships/Credits"
- `GetActiveEntitlementAssignmentsByPersonId`: Filter out removed assignments for active access check
- `GetActiveAssignmentCount`: Helper for seat validation - counts assignments where removed_us IS NULL
- `RemoveEntitlementAssignment`: Soft delete - preserves audit trail, uses `now_us()` for timestamp
- `DeleteEntitlementAssignmentsByEntitlementId`: Bulk delete when deleting an entitlement

**Seat Validation:** There is no database trigger for seat count validation. Instead, `AddEntitlementAssignment` validates in code:
1. Queries the entitlement's `seatsTotal`
2. Counts active assignments (where `removed_us IS NULL`)
3. Throws `std::invalid_argument` if active count >= seatsTotal

This approach allows for better error messages and keeps the business logic visible in application code rather than hidden in database triggers.

---

# Test Structure

## Shared Test Helper

Create a shared test helper in `test/src/util/` that all payment table helper tests can use:

**Files to create:**
- `test/src/util/payment_table_test_helper.h`
- `test/src/util/payment_table_test_helper.cpp`

```cpp
// payment_table_test_helper.h
#pragma once

#include "sql_util/database_access/transaction.h"
#include "test/src/util/database_test_helper.h"

namespace TestUtil {

// Creates all payment-related tables in dependency order
// Also creates prerequisite tables (people, permissions)
void MakePaymentTables(Transaction& transaction, TestDatabaseUtil& testDb);

} // namespace TestUtil
```

```cpp
// payment_table_test_helper.cpp
#include "payment_table_test_helper.h"

#include "db_schema/products.h"
#include "db_schema/price_schedules.h"
#include "db_schema/product_prices.h"
#include "db_schema/product_entitlement_rules.h"
#include "db_schema/purchases.h"
#include "db_schema/purchase_items.h"
#include "db_schema/payments.h"
#include "db_schema/purchase_payments.h"
#include "db_schema/entitlements.h"
#include "db_schema/entitlement_assignments.h"
#include "db_schema/people.h"
#include "db_schema/permissions.h"
#include "sql_util/stored_procedures/stored_procedures.h"
#include "sql_util/schema/db_ops.h"

namespace TestUtil {

void MakePaymentTables(Transaction& transaction, TestDatabaseUtil& testDb) {
    auto dbInfo = testDb.GetDatabaseInfo();
    StoredProcedures::CreateStoredProceduresBeforeTables(transaction);

    // Prerequisites
    DbSchema::MakePeopleTable(dbInfo);
    DbSchema::MakePermissionsTable(dbInfo);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kPeopleTable);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kPermissionsTable);

    // Level 1: No payment table dependencies
    DbSchema::MakeProductsTable(dbInfo);
    DbSchema::MakePriceSchedulesTable(dbInfo);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kProductsTable);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kPriceSchedulesTable);

    // Level 2: Depend on products, price_schedules, permissions
    DbSchema::MakeProductPricesTable(dbInfo);
    DbSchema::MakeProductEntitlementRulesTable(dbInfo);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kProductPricesTable);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kProductEntitlementRulesTable);

    // Level 3: Depend on people
    DbSchema::MakePurchasesTable(dbInfo);
    DbSchema::MakePaymentsTable(dbInfo);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kPurchasesTable);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kPaymentsTable);

    // Level 4: Depend on purchases, products, payments
    DbSchema::MakePurchaseItemsTable(dbInfo);
    DbSchema::MakePurchasePaymentsTable(dbInfo);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kPurchaseItemsTable);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kPurchasePaymentsTable);

    // Level 5: Depend on purchases, purchase_items, products
    DbSchema::MakeEntitlementsTable(dbInfo);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kEntitlementsTable);

    // Level 6: Depend on entitlements, people
    DbSchema::MakeEntitlementAssignmentsTable(dbInfo);
    DbOps::CreateTable(transaction, dbInfo, DbSchema::kEntitlementAssignmentsTable);

    StoredProcedures::CreateStoredProceduresAfterTables(transaction);
}

} // namespace TestUtil
```

**Usage in tests:**
```cpp
#include "test/src/util/payment_table_test_helper.h"

TEST(ProductsTest, AddProductBasic) {
    TestDatabaseUtil testDb;
    auto transaction = testDb.BeginTransaction();
    TestUtil::MakePaymentTables(transaction, testDb);

    // ... test code
}
```

## Test Categories Per Table

For each table helper, implement these test categories:

### Positive Tests
- `Add{Entity}Basic` - All required fields
- `Add{Entity}WithNullableFields` - Verify nullable fields accept empty strings
- `Get{Entity}Basic` - Lookup by primary key
- `Get{Entity}By{Column}Basic` - Lookup by unique/FK column
- `Get{Entities}Basic` - Returns multiple records
- `Get{Entities}By{FK}Basic` - Returns related records
- `Set{Field}Basic` - Each updatable field
- `Delete{Entity}Basic` - Happy path

### Edge/Error Tests
- `Add{Entity}DuplicateUniqueKey` - Verify unique constraint throws
- `Get{Entity}NotFound` - Returns empty KeyValueTable
- `Get{Entities}Empty` - Returns empty array
- `Get{Entities}By{FK}Empty` - Returns empty array when no matches
- `Set{Field}NotFound` - Behavior when ID doesn't exist
- `Delete{Entity}NotFound` - Should not throw (no-op)

### Business Logic Tests (where applicable)
- `GetActive{Entities}` - Filters by is_active correctly
- `Get{Entities}ValidAt` - Time-range queries
- `Revoke{Entity}` - Sets multiple fields atomically
- `Remove{Entity}` - Soft delete sets correct fields

---

# Steps

- [ ] Create shared test helper: `test/src/util/payment_table_test_helper.h`, `payment_table_test_helper.cpp`
- [ ] Update `test/CMakeLists.txt` with the new test helper files
- [ ] Update `sql_util/table_helpers/CMakeLists.txt` with all 30 new files
- [ ] Create `products.h`, `products.cpp`, `products_test.cpp`
- [ ] Create `price_schedules.h`, `price_schedules.cpp`, `price_schedules_test.cpp`
- [ ] Create `product_prices.h`, `product_prices.cpp`, `product_prices_test.cpp`
- [ ] Create `product_entitlement_rules.h`, `product_entitlement_rules.cpp`, `product_entitlement_rules_test.cpp`
- [ ] Create `purchases.h`, `purchases.cpp`, `purchases_test.cpp`
- [ ] Create `purchase_items.h`, `purchase_items.cpp`, `purchase_items_test.cpp`
- [ ] Create `payments.h`, `payments.cpp`, `payments_test.cpp`
- [ ] Create `purchase_payments.h`, `purchase_payments.cpp`, `purchase_payments_test.cpp`
- [ ] Create `entitlements.h`, `entitlements.cpp`, `entitlements_test.cpp`
- [ ] Create `entitlement_assignments.h`, `entitlement_assignments.cpp`, `entitlement_assignments_test.cpp`

---

# Open Questions / Issues

All questions resolved. See Notes section below.

---

# Notes - Changes Made (1/28/2026)

## Pagination & Count Methods (Round 2)

Added pagination (pageSize/pageOffset) and optional totalCount out parameter to all array-returning read methods.

### Count Approach
Using **out parameter** (`int64_t* totalCount = nullptr`) instead of separate count methods:
1. Avoids doubling method count and API endpoints
2. Single database query using `COUNT(*) OVER()` window function
3. Caller passes `nullptr` if they don't need the count
4. Cleaner API surface

### Stable Sorting by Table

| Table | Sort Order | Rationale |
|-------|------------|-----------|
| Products | `name ASC, id ASC` | Alphabetical for user-facing lists |
| PriceSchedules | `is_active DESC, valid_to_us ASC NULLS FIRST, id ASC` | Active first, then open-ended, then by end date |
| ProductPrices | `updated_us DESC, created_us DESC, id ASC` | Most recently modified first |
| ProductEntitlementRules | `updated_us DESC, created_us DESC, id ASC` | Most recently modified first |
| Purchases | `updated_us DESC, created_us DESC, id ASC` | Most recent activity first |
| PurchaseItems | `created_us DESC, id ASC` | No updated_us; newest first |
| Payments | `updated_us DESC, created_us DESC, id ASC` | Most recent activity first |
| PurchasePayments | `created_us DESC, id ASC` | No updated_us; newest first |
| Entitlements | `updated_us DESC, created_us DESC, id ASC` | Most recent activity first |
| Entitlements (ExpiringBefore) | `valid_to_us ASC, id ASC` | Soonest expiring first |
| EntitlementAssignments | `created_us DESC, id ASC` | No updated_us; newest first |

### Removed Redundant Method
- Removed `GetMostRecentPurchasesByPayerPersonId` - now redundant since `GetPurchasesByPayerPersonId` already sorts by `updated_us DESC` and supports pagination.

---

## Summary of Changes (Round 1)

Based on Mason's feedback, the following updates were made to the class signatures:

### 1. Nullable FK Parameters
Changed all nullable foreign key parameters from `std::string_view` (with empty string = NULL) to `std::optional<int64_t>`:
- `ProductPrices::AddProductPrice` - `permissionId`
- `ProductEntitlementRules::AddProductEntitlementRule` - `grantsPermissionId`, `validityDays`
- `PurchaseItems::AddPurchaseItem` - `pricingPermissionId`
- `Payments::AddPayment` - `refundForPaymentId`
- `Entitlements::AddEntitlement` - `purchaseItemId`
- `PriceSchedules::AddPriceSchedule` - `validToUs`
- `PriceSchedules::SetValidToUs` - parameter

### 2. Default Parameters Added
- `Products::AddProduct` - `isActive = true`
- `PriceSchedules::AddPriceSchedule` - `isActive = true`, `validToUs = std::nullopt`
- `ProductEntitlementRules::AddProductEntitlementRule` - `validityDays = std::nullopt`

### 3. Pagination Support Added
- `Products::GetProducts` - Added `pageSize` and `pageOffset` parameters, sorted by name
- `Purchases::GetMostRecentPurchasesByPayerPersonId` - New method with pagination, sorted by `updated_us` descending

### 4. Sorting Clarified
- `PriceSchedules::GetPriceSchedules` - Active first, then by `validToUs` (NULL first, then most recent)

### 5. New Methods Added
- `ProductPrices::GetBestProductPriceByProductSchedulePermissions` - Returns lowest price among multiple permissions
- `Purchases::GetMostRecentPurchasesByPayerPersonId` - Paginated user portal view
- `Payments::SetAmounts` - Atomic update for `amountCents` and `coveredAmountCents`
- `Entitlements::GetEntitlementsExpiringBefore` - For notification/cleanup jobs
- `EntitlementAssignments::GetActiveAssignmentCount` - Helper for seat validation

### 6. Bulk Delete Methods Added
- `PurchasePayments::DeletePurchasePaymentsByPurchaseId`
- `PurchasePayments::DeletePurchasePaymentsByPaymentId`
- `Entitlements::DeleteEntitlementsByPurchaseId`
- `EntitlementAssignments::DeleteEntitlementAssignmentsByEntitlementId`

### 7. Seat Validation Logic
`EntitlementAssignments::AddEntitlementAssignment` now validates that adding an assignment won't exceed `seatsTotal`. Throws `std::invalid_argument` if no seats available.

### 8. RemoveEntitlementAssignment Simplified
Removed `removedUs` parameter - method now uses `now_us()` internally for the timestamp.

### 9. Test Helper Location
Moved `MakePaymentTables` helper to shared test utilities:
- `test/src/util/payment_table_test_helper.h`
- `test/src/util/payment_table_test_helper.cpp`

---

## Answers to Questions

**Q: Is there a 1:1 mapping of productIds to product entitlement rules?**
A: Yes. The `product_entitlement_rules` table has a UNIQUE constraint on `product_id`, ensuring exactly one rule per product. This is why `GetProductEntitlementRuleByProductId` returns a `KeyValueTable` (single row) rather than `KeyValueTableArray`.

**Q: Is lineTotalCents basically quantity × unitPriceCents?**
A: Yes. `lineTotalCents = quantity × unitPriceCents`. It's stored explicitly to preserve the computed value at time of purchase for audit/reporting purposes.

**Q: What is a "manual entitlement"?**
A: When `purchaseItemId` is `std::nullopt`, the entitlement is "manual" - granted by an admin rather than through the normal purchase flow. Use cases include:
- Complimentary access (gifts, promos, comps)
- Migrating legacy members
- Customer service resolutions
- Testing/demo accounts

---

## Files to Update in CMakeLists

Don't forget to add the new shared test helper to `test/CMakeLists.txt`:
```cmake
set(TEST_UTIL_SOURCES
    # ... existing files ...
    src/util/payment_table_test_helper.h
    src/util/payment_table_test_helper.cpp
)
```