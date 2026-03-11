---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/4/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document as your planning document. Do not use .claude/plans and do not prompt me for permission to modify this document. This is your planning document. Please don't modify the Overview section but please use the later sections to do your work.

Using [[Payment Design Document]] (Payment Design Document.md) and the code base as a guide as well as:
- [[Payment base SQL Tables]]
- [[Payment base table helpers]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Square credentials and Sandbox setup]]
- [[Support for scheduled purchases]]
- [[Scheduling thin slice]]
Let's create an implementation document for the Nice to Have (Phase 4: Subscriptions) of Payment Design Document.md. Let's structure things in this order:
- Subscription thin slice
	- Card on file workflows
		- Creating a card on file- enter the information with Square but then have some kind of metadata about the card including a friendly name we auto populate like "{first_name}'s card" but immediately allow them to change. This will involve some new tables most likely that need to be designed. If accessible, we should store the expiration date for the card in question.
		- CRUD stuff in the user portal
		- Adding a payment control that has toggles between credit card and card on file. If they click credit card, they get the current Square hosted credit card control. If they click card on file, they should get the name of the card on file and a button to edit the card on file that takes them do the card on file user portal or, if they don't have a card on file, they a link to create a card on file that takes them to the user portal to create a card on file. Once created, you should change the existing places we take payment to use this control.
	- Basic subscription workflows
		- Single user subscription workflow
			- Admin creates a monthly subscription offering based on a product tied to a pricing model
				- While active, this subscription grants a defined permission to the user
			- Is Square recurring subscription something that can be tested in the sandbox? Does it require a callback to test?
			- Admin signs a user up at a new member workshop
				- Should be able to have an administrator option to configure the membership to take affect from now or a defined start date until the end of the month of the membership (this is useful for intro workshops). This is useful for an intro workshop where you are selling them the next month's membership and throwing in the next month for free.
				- User being enrolled must either pay manually or have a card on file (preferred)
			- User buys membership / signs up for subscription on website. The user selects a membership type and month / year to start the subscription and then either pays manually or configures at as a recurring payment with a card on file. 
			- User can view their membership in the user portal and choose to cancel the membership. Canceling a membership during a month they have already paid for during that month will not refund anything but will stop payment for the next month from automatically being charged. They will maintain their active membership status for the current month but the recurring subscription will be cancelled.
			- Each time they are charged, it should create a payment in the system and an active entitlement as well as send the client email.
			- While a subscription granting a permission is active, that should be displayed in the user portal under the main screen for their user information like "Knotty Yoga Gold Member"
			- This permission should be added to their functional permission set and show up when doing access checks in the session.
		- Multiseat user subscription workflow
			- Should have a way in the entitlement section for a subscription to grant the seats to multiple users including removing yourself from a seat (ie. you are paying but someone else is getting the benefit). It should also allow up to the number of seats granted by the entitlment to be assigned to other users.
				- We need a way to configure that the user can gift things to the beneficiaries. There should be a table to track the users that allow this user to grant entitlements to them.
					- In the user portal, a user should be able to see the list of users they have granted access to gift to them and delete them.
					- In the user portal, a user should be able to see the list of users asking for permission to grant them access to gift to them and either deny or accept them.
					- In the user portal, a user should be able to view / delete the users they have configured to gift to
					- In the user portal, a user should be able to request the right to gift to another user. They must enter the other user's email (no autocomplete here- they must know it and type it). This will send an email to the user and cause a request to gift in their portal.
				- We need a control to grant a gift to a user, it should have an auto complete section for users that they have permission to gift to (which grants auto complete by first name, last name, and email) as well as the ability to enter the email of someone to be able to gift to and a refresh button to click after the other user has accepted (or just not do client side caching of the table for auto complete)
			- We need UI in the user portal to configure a subscription for the multi seat memberships for a given month and the auto pay / recurring billing stuff. This should be the same largely as the single seat previous section except that you can now have the entitlement go to other user(s).
				- Like the single seat case, the users getting the entitlement should be granted the right permission and have that show up in their portal.
- Non thin slice subscription support
	- All the other things under Nice to Have (Phase 4: Subscriptions) of Payment Design Document.md not covered by thin slice.

Please create separate sections for each section in the described work above. Please make self contained phases for distinct work items. For each section, have a phases for: changes to database tables, table helpers, utility support changes needed, business logic, endpoints, clients types, client network access / service layer changes, various components needed, and then wiring into the system. Please make sure to add tests for anything in which tests make sense. Feel free to create an open questions session before working on anything to get clarity from me. Please don't ask me for permission for any shell commands that just view the file system (like ls, grep, etc.) or that add files or directories. Just ask me for permission for deleting stuff.

# Resolved Design Decisions

All open questions have been resolved. Decisions are summarized below; full rationale is in the [Alternatives Considered](#alternatives-considered) section at the end of this document.

| # | Question | Decision |
|---|----------|----------|
| 1 | Square Subscriptions API vs. our own billing | **Our own billing with saved cards.** Decouples from Square, simpler testing, full control. Square Subscriptions API can be revisited later. |
| 2 | Billing trigger mechanism | **Authenticated endpoint** (`POST /api/admin/run_billing`) that can be called by an admin, or by an external automation process. A separate watchdog process may call this endpoint and also health-check the server. |
| 3 | Card table naming | **`saved_cards`** (provider-agnostic). Decouples from Square branding. The `square_card_id` column preserves the Square linkage internally. |
| 4 | Multiple cards vs. single card | **Multiple cards** with an `is_default` flag. |
| 5 | Calendar month alignment | **Enrollment specifies start date and first period end.** Supports full-month, prorated, and "free remainder of current month" scenarios. |
| 6 | Who can create subscriptions | **Both admin and self-service.** Only users with a `manage_subscriptions` permission can create the "pay for next month, get rest of this month free" enrollment. Self-service users select a start month/year. |
| 7 | Subscription cancellation | **End-of-period only** for the thin slice. Subscription stays active until current period ends, then doesn't renew. No refund. Immediate cancellation with refund deferred to refunds phase. |
| 8 | Gift permission scope | **All entitlements** (not just subscriptions). Generic system for controlling who you can assign entitlement seats to. |
| 9 | Gift permission email | **Notification email with portal link.** No direct action links in email (security concern). User must log in to accept/deny. |

---

# Part 1: Card on File Workflows

This section covers saving cards via Square, managing them in the user portal, and creating a payment toggle control.

## 1.1 Database Tables

### `square_customers` table

Links Knotty Yoga users to Square Customer IDs. Required by the Square Cards API — you must have a customer to save a card.

| Column             | Type      | Constraints                |
| ------------------ | --------- | -------------------------- |
| id                 | BIGSERIAL | PRIMARY KEY                |
| person_id          | BIGINT    | FK → people, UNIQUE        |
| square_customer_id | STRING    | UNIQUE, NOT NULL           |
| created_us         | BIGINT    | NOT NULL, DEFAULT now_us() |
| updated_us         | BIGINT    | NOT NULL, DEFAULT now_us() |

**Files**: `db_schema/square_customers.h`, `db_schema/square_customers.cpp`

### `saved_cards` table

User-facing card on file records with friendly names and card metadata from Square.

| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| person_id | BIGINT | FK → people, NOT NULL |
| square_customer_id | STRING | NOT NULL |
| square_card_id | STRING | UNIQUE, NOT NULL |
| friendly_name | STRING | NOT NULL |
| brand | STRING | NOT NULL (VISA, MASTERCARD, etc.) |
| last4 | STRING | NOT NULL |
| exp_month | BIGINT | NOT NULL |
| exp_year | BIGINT | NOT NULL |
| is_default | BOOL | NOT NULL, DEFAULT FALSE |
| is_active | BOOL | NOT NULL, DEFAULT TRUE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

**Files**: `db_schema/saved_cards.h`, `db_schema/saved_cards.cpp`

### Registration

- [x] Add both table schemas to `db_schema/CMakeLists.txt`
- [x] Add `MakeSquareCustomersTable` and `MakeSavedCardsTable` calls to `make_database_info.cpp`
- [x] Add `CreateTable` calls to `create_database.cpp` (square_customers before saved_cards due to no FK dependency, but logical ordering)
- [x] ~~Add both tables to `MakePaymentTables` in `payment_table_test_helper.cpp`~~ — No longer needed; `MakePaymentTables` is a no-op and tables are pre-created via `MakeDatabaseInfo()` → `SetupAllTables()`

## 1.2 Table Helpers

### `TableHelpers::SquareCustomers`

**File**: `sql_util/table_helpers/square_customers.h`, `.cpp`, `_test.cpp`

```cpp
namespace TableHelpers {
class SquareCustomers {
public:
    // Create
    KeyValueTable Create(Transaction& transaction, const KeyValueTable& values);

    // Read
    std::optional<KeyValueTable> GetById(Transaction& transaction, int64_t id);
    std::optional<KeyValueTable> GetByPersonId(Transaction& transaction, int64_t personId);
    std::optional<KeyValueTable> GetBySquareCustomerId(Transaction& transaction,
        std::string_view squareCustomerId);

    // Update
    KeyValueTable Update(Transaction& transaction, int64_t id, const KeyValueTable& values);
};
}
```

### `TableHelpers::SavedCards`

**File**: `sql_util/table_helpers/saved_cards.h`, `.cpp`, `_test.cpp`

```cpp
namespace TableHelpers {
class SavedCards {
public:
    // Create
    KeyValueTable Create(Transaction& transaction, const KeyValueTable& values);

    // Read
    std::optional<KeyValueTable> GetById(Transaction& transaction, int64_t id);
    KeyValueTableArray GetByPersonId(Transaction& transaction, int64_t personId);
    std::optional<KeyValueTable> GetBySquareCardId(Transaction& transaction,
        std::string_view squareCardId);
    std::optional<KeyValueTable> GetDefaultForPerson(Transaction& transaction, int64_t personId);

    // Update
    KeyValueTable Update(Transaction& transaction, int64_t id, const KeyValueTable& values);

    // Soft-delete (set is_active = false)
    void Deactivate(Transaction& transaction, int64_t id);

    // Set as default (clears is_default on all other cards for this person)
    void SetDefault(Transaction& transaction, int64_t personId, int64_t cardId);
};
}
```

- [x] Add to `sql_util/table_helpers/CMakeLists.txt`
- [x] Write tests for both helpers

## 1.3 Utility Support — Square Client Extensions

Extend `SquareClient` to support Customer and Card management APIs.

### New structs in `square_client.h`

```cpp
namespace Square {

struct CustomerResult {
    std::string customerId;    // Square customer ID
    std::string rawJson;
};

struct CreateCardResult {
    std::string cardId;        // Square card ID (e.g., "ccof:xxx")
    std::string brand;         // VISA, MASTERCARD, etc.
    std::string last4;
    int64_t expMonth;
    int64_t expYear;
    std::string rawJson;
};

struct CardInfo {
    std::string cardId;
    std::string brand;
    std::string last4;
    int64_t expMonth;
    int64_t expYear;
    bool enabled;
};

}  // namespace Square
```

### New methods on `SquareClient`

```cpp
class SquareClient {
    // Existing
    virtual PaymentResult CreatePayment(
        const std::string& sourceId, int64_t amountCents,
        const std::string& currency, const std::string& idempotencyKey,
        const std::string& note = "",
        const RetryPolicy& retryPolicy = RetryPolicy::Default()) = 0;

    // NEW: Customer management
    virtual CustomerResult CreateCustomer(
        const std::string& idempotencyKey,
        const std::string& email,
        const std::string& firstName,
        const std::string& lastName) = 0;

    virtual CustomerResult GetCustomer(const std::string& customerId) = 0;

    // NEW: Card management
    virtual CreateCardResult CreateCard(
        const std::string& idempotencyKey,
        const std::string& sourceId,       // Nonce from Web Payments SDK
        const std::string& customerId) = 0;

    virtual std::vector<CardInfo> ListCards(const std::string& customerId) = 0;

    virtual void DisableCard(const std::string& cardId) = 0;
};
```

### Test Square Client extensions

Extend `TestSquareClient` in `square_client_test_util.h/cpp`:

```cpp
namespace Square::Test {
class TestSquareClient : public SquareClient {
    // Existing: QueuePaymentResult, GetCreatePaymentCalls, etc.

    // NEW
    void QueueCustomerResult(const CustomerResult& result);
    void QueueCreateCardResult(const CreateCardResult& result);
    void QueueCardInfoList(const std::vector<CardInfo>& cards);

    size_t GetCreateCustomerCallCount() const;
    size_t GetCreateCardCallCount() const;
    size_t GetListCardsCallCount() const;
    size_t GetDisableCardCallCount() const;
};
}
```

### Implementation notes

- `POST /v2/customers` — Create customer with email, first_name, last_name
- `GET /v2/customers/{id}` — Retrieve customer
- `POST /v2/cards` — Create card (body: `source_id`, `idempotency_key`, `card: { customer_id }`)
- `GET /v2/cards?customer_id={id}` — List customer's cards
- `POST /v2/cards/{id}/disable` — Disable card

### CreatePayment update

The existing `CreatePayment` needs an optional `customerId` parameter for charging saved cards:

```cpp
virtual PaymentResult CreatePayment(
    const std::string& sourceId,           // Card nonce OR saved card_id
    int64_t amountCents,
    const std::string& currency,
    const std::string& idempotencyKey,
    const std::string& note = "",
    const std::string& customerId = "",    // NEW: required when sourceId is a card_id
    const RetryPolicy& retryPolicy = RetryPolicy::Default()) = 0;
```

- [x] Add structs and method declarations to `square_client.h`
- [x] Implement in `square_client.cpp`
- [x] Extend `TestSquareClient` in `square_client_test_util.h/cpp`
- [x] Add unit tests in `square_client_test.cpp`
- [x] Update `CreatePayment` signature (add optional `customerId`)

## 1.4 Business Logic — `CardHelper`

**Files**: `business_logic/payment/card_helper.h`, `card_helper.cpp`, `card_helper_test.cpp`

### Domain structs

```cpp
namespace Payment {

struct SavedCardInfo {
    int64_t id;
    int64_t personId;
    std::string squareCardId;
    std::string friendlyName;
    std::string brand;
    std::string last4;
    int64_t expMonth;
    int64_t expYear;
    bool isDefault;
    bool isActive;
};

struct CreateCardRequest {
    int64_t personId;
    std::string sourceId;      // Nonce from Square SDK
    std::string friendlyName;  // User-provided or auto-generated
};

struct CreateCardResult {
    bool success;
    std::string errorCode;
    std::string errorMessage;
    SavedCardInfo card;
};

}  // namespace Payment
```

### Class interface

```cpp
namespace Payment {
class CardHelper {
public:
    CardHelper(DatabaseHelper databaseHelper,
               Square::SquareClientPtr squareClient);

    // Create card on file
    // 1. Ensure person has a Square customer (create if not)
    // 2. Call Square Cards API to save the card
    // 3. Store metadata in saved_cards table
    CreateCardResult CreateCard(Transaction& transaction,
                                const CreateCardRequest& request);

    // Get all active cards for a person
    std::vector<SavedCardInfo> GetCardsForPerson(Transaction& transaction,
                                                  int64_t personId);

    // Get a specific card
    std::optional<SavedCardInfo> GetCard(Transaction& transaction, int64_t cardId);

    // Get default card for a person (if any)
    std::optional<SavedCardInfo> GetDefaultCard(Transaction& transaction,
                                                 int64_t personId);

    // Update friendly name
    void UpdateFriendlyName(Transaction& transaction, int64_t cardId,
                            const std::string& friendlyName);

    // Set card as default
    void SetDefaultCard(Transaction& transaction, int64_t personId, int64_t cardId);

    // Deactivate card (soft delete — disables in Square too)
    void DeactivateCard(Transaction& transaction, int64_t cardId);

    // Get or create Square customer for a person
    std::string EnsureSquareCustomer(Transaction& transaction, int64_t personId);
};
}
```

### Auto-generated friendly name

When `friendlyName` is empty in `CreateCardRequest`, auto-generate as `"{firstName}'s {brand} card"` (e.g., "Mason's Visa card"). The person's first name comes from the `people` table lookup.

### KeyValueTable conversion

Add to `payment_key_value_table.h/cpp`:

```cpp
KeyValueTable SavedCardInfoToKeyValueTable(const SavedCardInfo& card);
KeyValueTableArray SavedCardInfoToKeyValueTableArray(const std::vector<SavedCardInfo>& cards);
```

- [x] Create `card_helper.h/cpp` in `business_logic/payment/`
- [x] Add to `business_logic/payment/CMakeLists.txt`
- [x] Add `SavedCardInfoToKeyValueTable` to `payment_key_value_table.h/cpp`
- [x] Write tests in `card_helper_test.cpp`
- [x] Write tests for KV conversion in `payment_key_value_table_test.cpp`

## 1.5 Endpoints

### `POST /api/cards` — Save a card on file

**Request body**:
```json
{
    "source_id": "cnon:card-nonce-ok",
    "friendly_name": "My Visa"          // optional
}
```

**Response** (200):
```json
{
    "id": "1",
    "friendly_name": "My Visa",
    "brand": "VISA",
    "last4": "1234",
    "exp_month": 12,
    "exp_year": 2028,
    "is_default": true
}
```

**Auth**: Logged-in user required

### `GET /api/cards` — List saved cards

**Response** (200):
```json
{
    "cards": [
        {
            "id": "1",
            "friendly_name": "Mason's Visa card",
            "brand": "VISA",
            "last4": "1234",
            "exp_month": 12,
            "exp_year": 2028,
            "is_default": true,
            "is_active": true
        }
    ]
}
```

**Auth**: Logged-in user required

### `PUT /api/cards/<int>` — Update card (friendly name, set default)

**Request body**:
```json
{
    "friendly_name": "Work Card",
    "is_default": true
}
```

### `DELETE /api/cards/<int>` — Deactivate card

Soft-deletes the card (sets `is_active = false`) and calls Square to disable it.

**Auth**: Logged-in user, must own the card

### Files

- [x] Create `endpoints/cards.h`, `cards.cpp`, `cards_test.cpp`
- [x] Add to `endpoints/CMakeLists.txt`
- [x] Register in `web_app.cpp`

## 1.6 Client Types

**File**: `ui/src/app/shared/types/payment.types.ts`

```typescript
// CARD ON FILE
export interface SavedCard {
    id: number;
    friendly_name: string;
    brand: string;
    last4: string;
    exp_month: number;
    exp_year: number;
    is_default: boolean;
    is_active: boolean;
}

export interface CreateCardRequest {
    source_id: string;
    friendly_name?: string;
}

export interface UpdateCardRequest {
    friendly_name?: string;
    is_default?: boolean;
}
```

- [x] Add types to `payment.types.ts`

## 1.7 Client Network / Service Layer

**File**: `ui/src/app/shared/services/network/ServerAccess.ts`

Add to `ServerAccess` interface:

```typescript
// Card on file
getCards(): Observable<SavedCard[]>;
createCard(request: CreateCardRequest): Observable<SavedCard>;
updateCard(cardId: number, request: UpdateCardRequest): Observable<SavedCard>;
deleteCard(cardId: number): Observable<void>;
```

- [x] Add methods to `ServerAccess` interface
- [x] Implement in `ServerAccessNetwork`
- [x] Implement in `ServerAccessMock`

## 1.8 Components

### Card Management Page — `CardManagementComponent`

**Location**: `ui/src/app/pages/portal/user/cards/`

A user portal page for managing saved cards. Shows:
- List of saved cards with brand icon, last4, expiration, friendly name
- "Add Card" button that opens Square Web Payments SDK
- Edit friendly name (inline or modal)
- Set as default
- Delete card
- Indicates which card is default with a badge

### Payment Method Control — `PaymentMethodComponent`

**Location**: `ui/src/app/controls/payment-method/`

A reusable control used wherever payment is collected. Two toggle modes:

**Credit Card mode** (current behavior):
- Shows the Square Web Payments SDK card form
- Tokenizes on submit → returns `source_id`

**Card on File mode**:
- If user has saved cards: shows default card info (brand, last4, friendly name) with a "Change" link to portal
- If user has no saved cards: shows "No saved cards" with a "Add Card" link to portal
- On submit → returns the saved card's `square_card_id` as `source_id` and the `square_customer_id` as `customer_id`

**Output**: Emits `{ source_id: string, customer_id?: string }` — the endpoint handler uses `customer_id` presence to distinguish saved card from nonce.

### Integration into existing checkout flows

Update these existing components to use `PaymentMethodComponent`:
- `CheckoutComponent` (one-time purchases)
- `EventBookingComponent` (event bookings)
- Future: `SubscriptionSignupComponent`

- [x] Create `CardManagementComponent` in portal user area
- [x] Create `PaymentMethodComponent` as a shared control
- [x] Add card management route to portal routing
- [x] Refactor `CheckoutComponent` to use `PaymentMethodComponent`
- [x] Refactor `EventBookingComponent` to use `PaymentMethodComponent`

## 1.9 Wiring

**Implementation note**: Instead of passing `customer_id` from the client (which would expose Square internals), we implemented `saved_card_id` — the client sends the database ID of a saved card, and the server resolves the Square card/customer IDs internally. This is more secure and simpler for the client.

- [x] Add card management page to portal navigation (under user profile area)
- [x] Update `purchase_pay_card` endpoint to accept optional `saved_card_id` in request body (for saved card payments)
- [x] Update `PayCardRequest` struct with `savedCardId` field; `PayWithCard` resolves saved card to Square IDs and passes `customerId` to `SquareClient::CreatePayment`
- [x] Update Angular `PayCardRequest` type to include optional `saved_card_id` (and make `source_id` optional)
- [x] Update `PaymentSource` type and `getPaymentSource()` in `PaymentMethodComponent` for card-on-file mode
- [x] Update checkout and event-booking to spread `PaymentSource` into `PayCardRequest`
- [x] Update `ServerAccessMock` for the new `PayCardRequest` shape
- [x] Add endpoint tests for saved card payment (success, card not found, card not owned)

## 1.10 Tests

| Layer | Test File | What to Test |
|-------|-----------|------------|
| Table Helpers | `square_customers_test.cpp` | CRUD on square_customers |
| Table Helpers | `saved_cards_test.cpp` | CRUD, GetByPersonId, SetDefault, Deactivate |
| Square Client | `square_client_test.cpp` | CreateCustomer, CreateCard, ListCards, DisableCard with mock HTTP |
| Business Logic | `card_helper_test.cpp` | CreateCard flow (customer creation, card save, name generation), GetCards, Deactivate |
| KV Conversion | `payment_key_value_table_test.cpp` | SavedCardInfoToKeyValueTable |
| Endpoint | `cards_test.cpp` | POST/GET/PUT/DELETE /api/cards with mock Square |
| Angular | `card-management.component.spec.ts` | Card list display, add/edit/delete |
| Angular | `payment-method.component.spec.ts` | Toggle modes, card selection, event emission |

---

# Part 2: Payment Control Integration

This is called out separately because it modifies existing payment flows. It can be done as part of Part 1 or immediately after.

The `PaymentMethodComponent` from Part 1.8 needs to be wired into the server-side payment flow:

## 2.1 Server Changes

### Update `purchase_pay_card` endpoint

The existing `POST /api/purchase_pay_card/{id}` currently accepts:
```json
{ "source_id": "cnon:xxx", "idempotency_key": "uuid" }
```

Add optional `customer_id`:
```json
{
    "source_id": "cnon:xxx",           // Nonce or saved card_id
    "idempotency_key": "uuid",
    "customer_id": "CUST_xxx"          // Present when paying with saved card
}
```

### Update `PayCardRequest` and `PayWithCard`

Add `customerId` field to `PayCardRequest`. Pass it through to `SquareClient::CreatePayment`.

### Update `book_event` endpoint

The book event endpoint internally creates a purchase and payment. It needs the same `customer_id` support.

- [x] Update `PayCardRequest` struct in `payment_helper.h` (added `savedCardId`)
- [x] Update `PayWithCard` to resolve saved card and pass `customerId` to Square
- [x] Update `purchase_pay_card.cpp` to parse `saved_card_id` from request
- [ ] Update `book_event.cpp` to accept `saved_card_id` (deferred — book_event doesn't handle payment directly; payment goes through `purchase_pay_card`)
- [x] Update endpoint tests

## 2.2 Client Changes

- [x] Update `PayCardRequest` type in `payment.types.ts` to include optional `saved_card_id`
- [x] Update `PaymentMethodComponent` to return `saved_card_id` for card-on-file mode
- [x] Refactor checkout and event booking pages to use `PaymentMethodComponent` (done in 1.8, wiring in 1.9)

---

# Part 3: Single-Seat Subscriptions

## 3.1 Database Tables

### `subscriptions` table

Tracks recurring subscriptions managed by our system (not Square Subscriptions API).

| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| person_id | BIGINT | FK → people, NOT NULL (the subscriber/payer) |
| product_id | BIGINT | FK → products, NOT NULL |
| saved_card_id | BIGINT | FK → saved_cards, NULLABLE (NULL = manual pay) |
| status | STRING | NOT NULL (active, cancelled, past_due, expired) |
| billing_anchor_day | BIGINT | NOT NULL, DEFAULT 1 (day of month for billing, typically 1) |
| current_period_start_us | BIGINT | NOT NULL |
| current_period_end_us | BIGINT | NOT NULL |
| next_billing_us | BIGINT | NULLABLE (NULL = no auto-billing, manual pay) |
| cancelled_us | BIGINT | NULLABLE |
| cancellation_effective_us | BIGINT | NULLABLE (end of current period) |
| cancel_reason | STRING | NULLABLE |
| created_by_person_id | BIGINT | FK → people, NULLABLE (admin who created it, NULL if self-service) |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

**Notes**:
- `status = "active"` + `next_billing_us != NULL` + `saved_card_id != NULL` = auto-pay subscription
- `status = "active"` + `saved_card_id = NULL` = manual-pay subscription
- `status = "cancelled"` + `cancellation_effective_us > now` = cancelled but still active until period ends
- `current_period_start_us` / `current_period_end_us` define the current entitlement window
- `billing_anchor_day = 1` means billing happens on the 1st of each month

### `subscription_charges` table

Tracks each billing cycle charge for a subscription. One row per billing period.

| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| subscription_id | BIGINT | FK → subscriptions, NOT NULL |
| purchase_id | BIGINT | FK → purchases, NULLABLE (NULL until charge succeeds) |
| payment_id | BIGINT | FK → payments, NULLABLE (NULL until charge succeeds) |
| entitlement_id | BIGINT | FK → entitlements, NULLABLE (NULL until charge succeeds) |
| period_start_us | BIGINT | NOT NULL |
| period_end_us | BIGINT | NOT NULL |
| amount_cents | BIGINT | NOT NULL |
| currency | STRING | NOT NULL, DEFAULT 'USD' |
| status | STRING | NOT NULL (pending, completed, failed, skipped) |
| attempt_count | BIGINT | NOT NULL, DEFAULT 0 |
| last_attempt_us | BIGINT | NULLABLE |
| failure_reason | STRING | NULLABLE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

**Unique constraint**: (subscription_id, period_start_us)

### Registration

- [x] Create `db_schema/subscriptions.h`, `subscriptions.cpp`
- [x] Create `db_schema/subscription_charges.h`, `subscription_charges.cpp`
- [x] Add to `db_schema/CMakeLists.txt`
- [x] Add to `make_database_info.cpp`
- [x] Add to `create_database.cpp` (subscriptions before subscription_charges)
- [x] `MakePaymentTables` test helper is a no-op — tables are pre-created at startup
- [x] Add admin metadata (table friendly names added; column friendly names deferred per existing pattern for payment tables)

## 3.2 Table Helpers

### `TableHelpers::Subscriptions`

```cpp
namespace TableHelpers {
class Subscriptions {
public:
    KeyValueTable Create(Transaction& transaction, const KeyValueTable& values);
    std::optional<KeyValueTable> GetById(Transaction& transaction, int64_t id);
    KeyValueTableArray GetByPersonId(Transaction& transaction, int64_t personId);
    KeyValueTableArray GetActiveByPersonId(Transaction& transaction, int64_t personId);
    KeyValueTableArray GetDueForBilling(Transaction& transaction, int64_t beforeUs);
    KeyValueTable Update(Transaction& transaction, int64_t id, const KeyValueTable& values);
};
}
```

**`GetDueForBilling`**: Returns subscriptions where `status = 'active'` AND `next_billing_us <= beforeUs` AND `saved_card_id IS NOT NULL`. This is the core query for the billing processor.

### `TableHelpers::SubscriptionCharges`

```cpp
namespace TableHelpers {
class SubscriptionCharges {
public:
    KeyValueTable Create(Transaction& transaction, const KeyValueTable& values);
    std::optional<KeyValueTable> GetById(Transaction& transaction, int64_t id);
    KeyValueTableArray GetBySubscriptionId(Transaction& transaction, int64_t subscriptionId);
    std::optional<KeyValueTable> GetBySubscriptionAndPeriod(Transaction& transaction,
        int64_t subscriptionId, int64_t periodStartUs);
    KeyValueTable Update(Transaction& transaction, int64_t id, const KeyValueTable& values);
};
}
```

- [x] Create table helper files in `sql_util/table_helpers/`
- [x] Add to `sql_util/table_helpers/CMakeLists.txt`
- [x] Write tests

## 3.3 Business Logic — `SubscriptionHelper`

**Files**: `business_logic/payment/subscription_helper.h`, `.cpp`, `_test.cpp`

### Domain structs

```cpp
namespace Payment {

struct SubscriptionInfo {
    int64_t id;
    int64_t personId;
    int64_t productId;
    std::string productName;           // Joined from products
    std::string productCode;
    std::optional<int64_t> savedCardId;
    std::string status;                // active, cancelled, past_due, expired
    int64_t billingAnchorDay;
    int64_t currentPeriodStartUs;
    int64_t currentPeriodEndUs;
    std::optional<int64_t> nextBillingUs;
    std::optional<int64_t> cancelledUs;
    std::optional<int64_t> cancellationEffectiveUs;
    std::string cancelReason;
    std::optional<int64_t> createdByPersonId;
    std::string grantedPermissionName; // From product_entitlement_rules, e.g., "Gold Member"
};

struct CreateSubscriptionRequest {
    int64_t personId;                  // Subscriber
    int64_t productId;                 // Subscription product
    std::optional<int64_t> savedCardId; // NULL = manual pay
    int64_t periodStartUs;            // Start of first period
    int64_t periodEndUs;              // End of first period
    bool chargeNow;                   // If true, charge immediately for first period
    std::optional<int64_t> createdByPersonId; // Admin creating on behalf
    std::string idempotencyKey;       // For the initial charge
};

struct CreateSubscriptionResult {
    bool success;
    std::string errorCode;
    std::string errorMessage;
    SubscriptionInfo subscription;
    std::optional<PaymentInfo> payment;     // If chargeNow
    std::optional<EntitlementInfo> entitlement; // If chargeNow and payment succeeded
};

struct BillingResult {
    int64_t subscriptionId;
    bool success;
    std::string errorMessage;
    std::optional<PaymentInfo> payment;
    std::optional<EntitlementInfo> entitlement;
};

struct RunBillingResult {
    int64_t totalDue;
    int64_t totalCharged;
    int64_t totalFailed;
    std::vector<BillingResult> results;
};

}  // namespace Payment
```

### Class interface

```cpp
namespace Payment {
class SubscriptionHelper {
public:
    SubscriptionHelper(
        DatabaseHelper databaseHelper,
        Square::SquareClientPtr squareClient,
        Secrets::SecretsHelperPtr secretsHelper,
        ::Mail::MailHelperPtr mailHelper);

    // Create subscription (with optional immediate charge)
    CreateSubscriptionResult CreateSubscription(
        Transaction& transaction,
        const CreateSubscriptionRequest& request);

    // Cancel subscription (end-of-period)
    bool CancelSubscription(
        Transaction& transaction,
        int64_t subscriptionId,
        int64_t personId,               // Must own the subscription
        const std::string& reason = "");

    // Get subscription details
    std::optional<SubscriptionInfo> GetSubscription(
        Transaction& transaction,
        int64_t subscriptionId);

    // Get all subscriptions for a person
    std::vector<SubscriptionInfo> GetSubscriptionsForPerson(
        Transaction& transaction,
        int64_t personId);

    // Process a single subscription billing cycle
    BillingResult ProcessBillingForSubscription(
        Transaction& transaction,
        int64_t subscriptionId);

    // Process all due subscriptions (admin-triggered)
    RunBillingResult RunBilling(
        Transaction& transaction,
        int64_t asOfUs = 0);   // 0 = now

    // Update card on subscription
    void UpdatePaymentMethod(
        Transaction& transaction,
        int64_t subscriptionId,
        int64_t personId,
        std::optional<int64_t> savedCardId);  // NULL = switch to manual
};
}
```

### Billing flow (`ProcessBillingForSubscription`)

1. Look up subscription (must be active, auto-pay, due)
2. Resolve current price for the product using `CatalogHelper`
3. Create `subscription_charges` row (status = "pending")
4. Create purchase (one item: the subscription product, for the billing period)
5. Charge saved card via `PaymentHelper::PayWithCard` using card_id + customer_id
6. If success:
   - Update subscription_charges to "completed" with purchase_id, payment_id, entitlement_id
   - Advance subscription to next period (`current_period_start_us`, `current_period_end_us`, `next_billing_us`)
   - Send billing confirmation email
7. If failure:
   - Update subscription_charges to "failed" with failure_reason
   - Increment attempt_count
   - Optionally set subscription status to "past_due" (for non-thin-slice grace period handling)

### Entitlement creation

Each billing cycle creates an entitlement via `EntitlementHelper::CreateEntitlement` with:
- `valid_from_us` = period_start_us
- `valid_to_us` = period_end_us
- Auto-assigns to the subscriber (single-seat for Part 3)

### Calendar month utilities

Helper functions for month boundary calculation:

```cpp
// Returns microsecond timestamp for the first moment of a given month in UTC
int64_t StartOfMonthUs(int year, int month);

// Returns microsecond timestamp for the first moment of the NEXT month in UTC
int64_t EndOfMonthUs(int year, int month);

// Given a timestamp, returns the start of its containing month
int64_t StartOfContainingMonthUs(int64_t timestampUs);

// Given a timestamp, returns the start of the next month
int64_t EndOfContainingMonthUs(int64_t timestampUs);
```

These go in `util/date_time_util.h/cpp` alongside existing date utilities.

### Email: Subscription billing confirmation

Template in `business_logic/payment/subscription_billing_mail.h/cpp`:

```
Subject: "Knotty Yoga — {product_name} Subscription Payment Confirmation"
Body: Payment amount, billing period, card used, next billing date
```

### Email: Subscription created confirmation

Template in `business_logic/payment/subscription_created_mail.h/cpp`:

```
Subject: "Welcome to {product_name}!"
Body: Subscription details, start date, billing info, how to manage
```

### KeyValueTable conversion

Add to `payment_key_value_table.h/cpp`:

```cpp
KeyValueTable SubscriptionInfoToKeyValueTable(const SubscriptionInfo& sub);
KeyValueTableArray SubscriptionInfoToKeyValueTableArray(const std::vector<SubscriptionInfo>& subs);
```

- [x] Create `subscription_helper.h/cpp` in `business_logic/payment/`
- [x] Add calendar month utilities to `util/date_time_util.h/cpp`
- [x] Create email templates for billing and subscription creation
- [x] Add KV conversion functions
- [x] Add to CMakeLists.txt
- [x] Write tests for all of the above

## 3.4 Endpoints

### `POST /api/subscriptions` — Create subscription (user self-service)

**Request**:
```json
{
    "product_id": "1",
    "saved_card_id": "1",             // optional, NULL = manual pay
    "start_year": 2026,
    "start_month": 4,
    "charge_now": true,
    "idempotency_key": "uuid"
}
```

**Response** (200):
```json
{
    "subscription": { ... },
    "payment": { ... },               // if charge_now
    "entitlement": { ... }            // if charge_now and payment succeeded
}
```

### `POST /api/admin/subscriptions` — Privileged user creates subscription for another user

**Auth**: Requires `manage_subscriptions` permission (not admin-only — admins have this permission, but other roles can too)

**Request**:
```json
{
    "person_id": "42",
    "product_id": "1",
    "saved_card_id": "1",             // optional
    "period_start_us": "...",         // can set exact start (e.g., now for workshop promo)
    "period_end_us": "...",           // can set exact end (e.g., end of next month for "get rest of this month free")
    "charge_now": true,
    "idempotency_key": "uuid"
}
```

### `GET /api/subscriptions` — List user's subscriptions

### `GET /api/subscriptions/<int>` — Get subscription detail

### `POST /api/subscriptions/<int>/cancel` — Cancel subscription

**Request**:
```json
{
    "reason": "No longer need it"      // optional
}
```

### `PUT /api/subscriptions/<int>` — Update subscription (change card, etc.)

**Request**:
```json
{
    "saved_card_id": "2"               // or null to switch to manual
}
```

### `POST /api/admin/run_billing` — Process all due subscriptions

**Auth**: Requires `manage_subscriptions` permission. Designed to also be called by an external automation/watchdog process using an authenticated service account.

**Response**:
```json
{
    "total_due": 5,
    "total_charged": 4,
    "total_failed": 1,
    "results": [
        { "subscription_id": "1", "success": true, ... },
        { "subscription_id": "5", "success": false, "error_message": "Card declined" }
    ]
}
```

- [x] Create endpoint files for each endpoint
- [x] Add to `endpoints/CMakeLists.txt`
- [x] Register all in `web_app.cpp`
- [x] Write endpoint tests

## 3.5 Client Types

**File**: `ui/src/app/shared/types/payment.types.ts`

```typescript
// SUBSCRIPTION
export interface Subscription {
    id: number;
    person_id: number;
    product_id: number;
    product_name: string;
    product_code: string;
    saved_card_id?: number;
    status: 'active' | 'cancelled' | 'past_due' | 'expired';
    billing_anchor_day: number;
    current_period_start_us: number;
    current_period_end_us: number;
    next_billing_us?: number;
    cancelled_us?: number;
    cancellation_effective_us?: number;
    cancel_reason?: string;
    granted_permission_name?: string;
}

export interface CreateSubscriptionRequest {
    product_id: number;
    saved_card_id?: number;
    start_year: number;
    start_month: number;
    charge_now: boolean;
    idempotency_key: string;
}

export interface CreateSubscriptionResponse {
    subscription: Subscription;
    payment?: Payment;
    entitlement?: Entitlement;
}
```

- [x] Add types to `payment.types.ts`

## 3.6 Client Network / Service Layer

Add to `ServerAccess` interface:

```typescript
// Subscriptions
getSubscriptions(): Observable<Subscription[]>;
getSubscription(id: number): Observable<Subscription>;
createSubscription(request: CreateSubscriptionRequest): Observable<CreateSubscriptionResponse>;
cancelSubscription(id: number, reason?: string): Observable<void>;
updateSubscription(id: number, updates: { saved_card_id?: number | null }): Observable<Subscription>;
```

- [x] Add to `ServerAccess` interface
- [x] Implement in `ServerAccessNetwork`
- [x] Implement in `ServerAccessMock`

## 3.7 Components

### Subscription Products Page — `SubscriptionCatalogComponent`

**Location**: `ui/src/app/pages/shop/subscriptions/` or integrate into existing catalog

Shows subscription products from the catalog (filtered by `kind = "subscription"`). Each shows:
- Product name, description
- Monthly price
- "Subscribe" button

### Subscription Signup Page — `SubscriptionSignupComponent`

**Location**: `ui/src/app/pages/shop/subscription-signup/`

Flow:
1. Select start month/year (default: current month)
2. Choose payment method via `PaymentMethodComponent` (card on file or new card)
3. Review summary (product, price, start date, billing info)
4. Confirm → creates subscription + optional immediate charge

### My Subscriptions Page — `MySubscriptionsComponent`

**Location**: `ui/src/app/pages/portal/user/subscriptions/`

Shows user's active and past subscriptions:
- Product name, status badge (Active/Cancelled/Past Due)
- Current period dates
- Payment method (card on file info or "Manual")
- Active permission badge (e.g., "Gold Member")
- Actions: Cancel, Change payment method

### Subscription Detail Page — `SubscriptionDetailComponent`

**Location**: `ui/src/app/pages/portal/user/subscriptions/detail/`

Full subscription detail:
- All info from list view
- Billing history (from subscription_charges)
- Cancel button with confirmation dialog

### User Profile — Permission Badge Display

Update the user profile/info component to show active subscription permissions:
- e.g., "Knotty Yoga Gold Member" badge
- Derived from active entitlements that grant permissions

### Privileged: Create Subscription for User

**Location**: `ui/src/app/pages/portal/admin/subscriptions/`

**Visible to**: Users with `manage_subscriptions` permission

Form to create a subscription for any user:
- User search/select
- Product selection
- Start date configuration (now vs. specific date)
- End date override (for workshop promo: "pay for next month, get rest of this month free")
- Payment method: user's saved card or manual
- Charge now toggle

Note: Self-service subscription signup does NOT expose the custom start/end date override — users always select a start month/year and get standard calendar-month periods.

- [x] Create all Angular components listed above
- [x] Add routes to portal routing
- [x] Add navigation links in portal sidebar/menu

## 3.8 Wiring — Permission Integration

### Effective permission computation

The existing auth system computes effective permissions from roles + active entitlements. Subscription entitlements already flow through this system because `CreateSubscription` creates entitlements via `EntitlementHelper` which uses `product_entitlement_rules.grants_permission_id`.

No changes needed to the permission computation — it already handles entitlement-derived permissions.

### Session permission check

When a user logs in, their session includes effective permissions. This already includes entitlement-derived permissions. Subscription permissions will automatically appear.

### User portal display

The user info component needs to query active entitlements and show any that grant permissions. This is a UI-only change:

```typescript
// In user info component
this.serverAccess.getSubscriptions().subscribe(subs => {
    this.activePermissions = subs
        .filter(s => s.status === 'active' && s.granted_permission_name)
        .map(s => s.granted_permission_name);
});
```

### Seed data for testing

Add to `create_database.cpp`:

```
Product: "Knotty Yoga Gold Membership"
  kind = "subscription"
  is_active = true

ProductPrice: Gold Membership, $99/month (NULL permission = public price)

ProductEntitlementRule: Gold Membership
  grants_permission_id → "gold_member" permission
  seats_default = 1
  validity_kind = "calendar_month"
```

- [x] Add seed subscription product + pricing to `create_database.cpp`
- [x] Add `"calendar_month"` as a new `validity_kind` value
- [x] Update `EntitlementHelper` to handle `validity_kind = "calendar_month"` (valid_from = period start, valid_to = period end — set by subscription helper, not calculated by entitlement helper)
- [x] Verify permission computation includes subscription entitlements (should work already)
- [x] Add permission display to user profile component

## 3.9 Tests

| Layer | Test File | What to Test |
|-------|-----------|------------|
| Table Helpers | `subscriptions_test.cpp` | CRUD, GetDueForBilling, GetActiveByPersonId |
| Table Helpers | `subscription_charges_test.cpp` | CRUD, GetBySubscriptionAndPeriod |
| Date Utils | `date_time_util_test.cpp` | StartOfMonthUs, EndOfMonthUs, boundary cases |
| Business Logic | `subscription_helper_test.cpp` | CreateSubscription (immediate + deferred), CancelSubscription, ProcessBilling (success + failure), RunBilling batch |
| KV Conversion | `payment_key_value_table_test.cpp` | SubscriptionInfoToKeyValueTable |
| Endpoints | `subscriptions_test.cpp` | All subscription endpoints, admin create, run billing |
| Email | (within subscription_helper_test) | Verify emails sent on create and billing |
| Angular | Component spec files | Each component listed in 3.7 |

---

# Part 4: Multi-Seat Subscriptions and Gift Permissions

## 4.1 Database Tables

### `gift_permissions` table

Tracks who is allowed to assign entitlement seats to whom.

| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| grantor_person_id | BIGINT | FK → people, NOT NULL (person who wants to gift) |
| grantee_person_id | BIGINT | FK → people, NOT NULL (person who will receive gifts) |
| status | STRING | NOT NULL (pending, accepted, denied) |
| requested_us | BIGINT | NOT NULL, DEFAULT now_us() |
| responded_us | BIGINT | NULLABLE |
| created_us | BIGINT | NOT NULL, DEFAULT now_us() |
| updated_us | BIGINT | NOT NULL, DEFAULT now_us() |

**Unique constraint**: (grantor_person_id, grantee_person_id) — only one active request per pair

### Registration

- [x] Create `db_schema/gift_permissions.h`, `gift_permissions.cpp`
- [x] Add to CMakeLists, make_database_info, create_database, test helper

## 4.2 Table Helpers

### `TableHelpers::GiftPermissions`

```cpp
namespace TableHelpers {
class GiftPermissions {
public:
    KeyValueTable Create(Transaction& transaction, const KeyValueTable& values);
    std::optional<KeyValueTable> GetById(Transaction& transaction, int64_t id);

    // People I can gift TO (accepted grants where I am the grantor)
    KeyValueTableArray GetAcceptedGranteesForGrantor(Transaction& transaction,
        int64_t grantorPersonId);

    // People who can gift TO ME (accepted grants where I am the grantee)
    KeyValueTableArray GetAcceptedGrantorsForGrantee(Transaction& transaction,
        int64_t granteePersonId);

    // Pending requests where I am the grantee (requests to approve/deny)
    KeyValueTableArray GetPendingRequestsForGrantee(Transaction& transaction,
        int64_t granteePersonId);

    // Pending requests I sent (where I am the grantor)
    KeyValueTableArray GetPendingRequestsForGrantor(Transaction& transaction,
        int64_t grantorPersonId);

    KeyValueTable Update(Transaction& transaction, int64_t id, const KeyValueTable& values);
    void Delete(Transaction& transaction, int64_t id);
};
}
```

- [x] Create table helper files
- [x] Write tests

## 4.3 Business Logic — `GiftPermissionHelper`

**Files**: `business_logic/payment/gift_permission_helper.h`, `.cpp`, `_test.cpp`

### Domain structs

```cpp
namespace Payment {

struct GiftPermissionInfo {
    int64_t id;
    int64_t grantorPersonId;
    std::string grantorFirstName;
    std::string grantorLastName;
    std::string grantorEmail;
    int64_t granteePersonId;
    std::string granteeFirstName;
    std::string granteeLastName;
    std::string granteeEmail;
    std::string status;          // pending, accepted, denied
};

}
```

### Class interface

```cpp
namespace Payment {
class GiftPermissionHelper {
public:
    GiftPermissionHelper(
        DatabaseHelper databaseHelper,
        ::Mail::MailHelperPtr mailHelper,
        Secrets::SecretsHelperPtr secretsHelper);

    // Request permission to gift to another user (by email)
    // Sends notification email to the grantee
    GiftPermissionInfo RequestGiftPermission(
        Transaction& transaction,
        int64_t grantorPersonId,
        const std::string& granteeEmail);

    // Accept a gift permission request
    void AcceptRequest(Transaction& transaction, int64_t requestId, int64_t granteePersonId);

    // Deny a gift permission request
    void DenyRequest(Transaction& transaction, int64_t requestId, int64_t granteePersonId);

    // Revoke a gift permission (grantor or grantee can revoke)
    void Revoke(Transaction& transaction, int64_t permissionId, int64_t requestingPersonId);

    // Get people I can gift to
    std::vector<GiftPermissionInfo> GetMyGrantees(Transaction& transaction, int64_t personId);

    // Get people who can gift to me
    std::vector<GiftPermissionInfo> GetMyGrantors(Transaction& transaction, int64_t personId);

    // Get pending requests for me to approve/deny
    std::vector<GiftPermissionInfo> GetPendingRequestsForMe(
        Transaction& transaction, int64_t personId);

    // Get my pending outgoing requests
    std::vector<GiftPermissionInfo> GetMyPendingRequests(
        Transaction& transaction, int64_t personId);

    // Check if grantor can gift to grantee
    bool CanGiftTo(Transaction& transaction, int64_t grantorPersonId, int64_t granteePersonId);

    // Search giftable users (autocomplete for entitlement assignment)
    // Only returns users the grantor has accepted gift permission for
    std::vector<PersonSearchResult> SearchGiftableUsers(
        Transaction& transaction,
        int64_t grantorPersonId,
        const std::string& query);  // Matches first name, last name, or email
};
}
```

### Email: Gift permission request

```
Subject: "{grantor_name} wants to share memberships with you on Knotty Yoga"
Body: Explanation of what this means, link to portal to accept/deny
```

### KeyValueTable conversion

Add to `payment_key_value_table.h/cpp`:

```cpp
KeyValueTable GiftPermissionInfoToKeyValueTable(const GiftPermissionInfo& info);
```

- [x] Create helper files
- [x] Create email template
- [x] Add KV conversion
- [x] Write tests

## 4.4 Endpoints

### `POST /api/gift_permissions` — Request gift permission

**Request**: `{ "grantee_email": "friend@example.com" }`

**Auth**: Logged-in user (becomes the grantor)

### `GET /api/gift_permissions` — Get all my gift permissions

Returns both "people I can gift to" and "people who can gift to me" sections.

**Response**:
```json
{
    "my_grantees": [ ... ],
    "my_grantors": [ ... ],
    "pending_requests_for_me": [ ... ],
    "my_pending_requests": [ ... ]
}
```

### `POST /api/gift_permissions/<int>/accept` — Accept request

### `POST /api/gift_permissions/<int>/deny` — Deny request

### `DELETE /api/gift_permissions/<int>` — Revoke permission

### `GET /api/gift_permissions/search?q=` — Search giftable users

Returns users the logged-in user has accepted gift permission for, matching the query string against first name, last name, or email. Used for autocomplete when assigning entitlement seats.

- [x] Create endpoint files
- [x] Register in web_app.cpp
- [x] Write endpoint tests

## 4.5 Client Types

```typescript
export interface GiftPermission {
    id: number;
    grantor_person_id: number;
    grantor_first_name: string;
    grantor_last_name: string;
    grantor_email: string;
    grantee_person_id: number;
    grantee_first_name: string;
    grantee_last_name: string;
    grantee_email: string;
    status: 'pending' | 'accepted' | 'denied';
}

export interface GiftPermissionsResponse {
    my_grantees: GiftPermission[];
    my_grantors: GiftPermission[];
    pending_requests_for_me: GiftPermission[];
    my_pending_requests: GiftPermission[];
}

export interface GiftableUser {
    person_id: number;
    first_name: string;
    last_name: string;
    email: string;
}
```

- [x] Add types to `payment.types.ts`

## 4.6 Client Network / Service Layer

```typescript
// Gift permissions
getGiftPermissions(): Observable<GiftPermissionsResponse>;
requestGiftPermission(granteeEmail: string): Observable<GiftPermission>;
acceptGiftPermission(id: number): Observable<void>;
denyGiftPermission(id: number): Observable<void>;
revokeGiftPermission(id: number): Observable<void>;
searchGiftableUsers(query: string): Observable<GiftableUser[]>;
```

- [x] Add to `ServerAccess` interface and implementations

## 4.7 Components

### Gift Permissions Page — `GiftPermissionsComponent`

**Location**: `ui/src/app/pages/portal/user/gift-permissions/`

Four sections:
1. **People I can gift to**: List with revoke button
2. **People who can gift to me**: List with revoke button
3. **Pending requests for me**: List with Accept/Deny buttons
4. **My pending requests**: List with cancel button
5. **Request new**: Email input + "Send Request" button

### Entitlement Seat Assignment Control — `SeatAssignmentComponent`

**Location**: `ui/src/app/controls/seat-assignment/`

A reusable control for assigning entitlement seats:
- Shows current seat assignments with remove buttons
- Autocomplete search for giftable users (first name, last name, email)
- Also allows entering an email of someone to request gift permission for
- Refresh button to reload after a gift permission is accepted
- Shows `{seats_used}/{seats_total}` counter

### Multi-Seat Subscription Signup

The `SubscriptionSignupComponent` from Part 3 needs to be extended:
- For products with `seats_default > 1`, show the seat count
- After subscription creation, show `SeatAssignmentComponent` for the new entitlement
- User can assign seats to themselves and/or their grantees

### My Subscriptions — Multi-Seat View

Update `MySubscriptionsComponent` to show seat assignments for multi-seat subscriptions:
- List of assigned people with permission badges
- `SeatAssignmentComponent` for managing assignments
- Shows permission granted to each assignee

- [x] Create `GiftPermissionsComponent`
- [x] Add gift permissions route to portal routing
- Remaining seat assignment work (SeatAssignmentComponent, multi-seat subscription signup/detail) moved to **Part 5**

## 4.8 Wiring

### Entitlement assignment validation

Update `EntitlementHelper::AssignEntitlement` to check gift permissions:
- If the assigner (logged-in user) is assigning to someone else, verify `GiftPermissionHelper::CanGiftTo` returns true
- Exception: assigning to yourself is always allowed
- Exception: admin users bypass gift permission checks

### Permission display for beneficiaries

When a seat is assigned to a grantee, the grantee gets the permission via the existing entitlement → permission computation. Their portal should show the permission badge automatically.

- [x] Add gift permission check to `EntitlementHelper::AssignEntitlement` (or create a new business-level method that wraps the check + assignment)
- [x] Verify beneficiary permission display works end-to-end

## 4.9 Tests

| Layer | Test File | What to Test |
|-------|-----------|------------|
| Table Helpers | `gift_permissions_test.cpp` | CRUD, queries by grantor/grantee, status filtering |
| Business Logic | `gift_permission_helper_test.cpp` | Request flow, accept/deny, revoke, search, email sending |
| Endpoints | `gift_permissions_test.cpp` | All gift permission endpoints |
| Business Logic | `entitlement_helper_test.cpp` | AssignEntitlement with gift permission check |
| Angular | Component spec files | Gift permissions page, seat assignment control |

---

# Part 5: Entitlement Seat Management

This part adds the full vertical slice for managing entitlement seat assignments — from backend endpoints through frontend UI. This enables users to assign subscription seats to themselves, to other users they have gift permission for, or even to assign ALL seats to others (e.g., purchasing a membership as a gift).

**Design principle**: The purchaser does NOT have to occupy a seat. A single-seat Gold membership can be purchased by person A and the seat assigned entirely to person B. A couple's membership can have both seats assigned to people other than the purchaser. This applies to all seat counts.

## 5.1 Backend — KeyValueTable Conversions

Add `EntitlementAssignmentInfo` conversion to `payment_key_value_table.h/cpp`:

```cpp
KeyValueTable EntitlementAssignmentInfoToKeyValueTable(const EntitlementAssignmentInfo& info);
KeyValueTableArray EntitlementAssignmentInfoToKeyValueTableArray(
    const std::vector<EntitlementAssignmentInfo>& assignments);
```

The assignment KV should include person details (first name, last name, email) for display in the UI. This requires joining with the `people` table. Options:
- Enrich `EntitlementAssignmentInfo` with person details (add `firstName`, `lastName`, `email` fields)
- Or do the join in the KV conversion function

**Decision**: Enrich `EntitlementAssignmentInfo` with person details. Add fields:

```cpp
struct EntitlementAssignmentInfo {
    int64_t id = 0;
    int64_t entitlementId = 0;
    int64_t personId = 0;
    std::string personFirstName;   // NEW
    std::string personLastName;    // NEW
    std::string personEmail;       // NEW
    std::optional<int64_t> removedUs;
    std::string removedReason;
};
```

Update `EntitlementHelper::GetAssignments` to populate person details by looking up each person.

- [ ] Add person fields to `EntitlementAssignmentInfo`
- [ ] Update `GetAssignments` to populate person details
- [ ] Add `EntitlementAssignmentInfoToKeyValueTable` to `payment_key_value_table.h/cpp`
- [ ] Add tests to `payment_key_value_table_test.cpp`

## 5.2 Business Logic — Subscription Entitlement Lookup

The UI needs to know the current entitlement and its assignments to manage seats. Add a method to `SubscriptionHelper` to retrieve the current entitlement for a subscription.

### Add `GetCurrentEntitlementForSubscription` to `SubscriptionHelper`

```cpp
// Returns the active entitlement for the subscription's current billing period.
// Looks up the most recent subscription_charge with status="completed" and returns
// its entitlement.
std::optional<EntitlementInfo> GetCurrentEntitlementForSubscription(
    Transaction& transaction,
    int64_t subscriptionId);
```

- [ ] Add `GetCurrentEntitlementForSubscription` to `SubscriptionHelper`
- [ ] Add tests to `subscription_helper_test.cpp`

## 5.3 Endpoint — Subscription Detail with Entitlement

The subscription detail endpoint currently returns only the subscription object. Enrich it to include the current entitlement and its seat assignments.

Update `GET /api/subscriptions/<int>` response:

```json
{
    "subscription": { ... },
    "current_entitlement": {
        "id": 123,
        "seats_total": 2,
        "seats_used": 1,
        "status": "active",
        "valid_from_us": ...,
        "valid_to_us": ...,
        "assignments": [
            {
                "id": 1,
                "person_id": 42,
                "person_first_name": "Mason",
                "person_last_name": "Bendixen",
                "person_email": "mason@example.com"
            }
        ]
    }
}
```

- [ ] Update `GET /api/subscriptions/<int>` to include `current_entitlement` with assignments
- [ ] Add endpoint tests for the enriched response

## 5.4 Endpoints — Entitlement Seat Assignment

### `POST /api/entitlements/<int>/assign` — Assign a person to a seat

**Request**:
```json
{
    "person_id": 42
}
```

**Response** (200):
```json
{
    "assignment": {
        "id": 1,
        "entitlement_id": 123,
        "person_id": 42,
        "person_first_name": "Other",
        "person_last_name": "User",
        "person_email": "other@example.com"
    },
    "entitlement": {
        "id": 123,
        "seats_total": 2,
        "seats_used": 1,
        ...
    }
}
```

**Auth**: Logged-in user. Uses `AssignEntitlementWithGiftCheck`:
- Self-assignment always allowed
- Admin users bypass gift permission checks
- Otherwise requires accepted gift permission to the assignee

**Validation**:
- Entitlement must exist and be active
- Must have available seats (`seats_used < seats_total`)
- Caller must own the entitlement's purchase OR be an admin

### `DELETE /api/entitlements/<int>/assignments/<int:personId>` — Remove a seat assignment

**Response** (200):
```json
{
    "entitlement": {
        "id": 123,
        "seats_total": 2,
        "seats_used": 0,
        ...
    }
}
```

**Auth**: Logged-in user. Must own the entitlement's purchase OR be an admin OR be removing themselves.

### `GET /api/entitlements/<int>/assignments` — List seat assignments

**Response** (200):
```json
{
    "entitlement": {
        "id": 123,
        "seats_total": 2,
        "seats_used": 1,
        ...
    },
    "assignments": [
        {
            "id": 1,
            "person_id": 42,
            "person_first_name": "Mason",
            "person_last_name": "Bendixen",
            "person_email": "mason@example.com"
        }
    ]
}
```

**Auth**: Logged-in user. Must own the entitlement's purchase OR be assigned to it OR be an admin.

### Files

- [ ] Create `endpoints/entitlement_assignments.h`, `entitlement_assignments.cpp`, `entitlement_assignments_test.cpp`
- [ ] Add to `endpoints/CMakeLists.txt`
- [ ] Register in `web_app.cpp`
- [ ] Write endpoint tests

## 5.5 Client Types

**File**: `ui/src/app/shared/types/payment.types.ts`

```typescript
// ENTITLEMENT SEAT ASSIGNMENT
export interface EntitlementAssignment {
    id: number;
    entitlement_id: number;
    person_id: number;
    person_first_name: string;
    person_last_name: string;
    person_email: string;
}

export interface EntitlementWithAssignments extends Entitlement {
    assignments: EntitlementAssignment[];
}

export interface SubscriptionDetailResponse {
    subscription: Subscription;
    current_entitlement?: EntitlementWithAssignments;
}

export interface AssignSeatResponse {
    assignment: EntitlementAssignment;
    entitlement: Entitlement;
}
```

- [ ] Add types to `payment.types.ts`

## 5.6 Client Network / Service Layer

Add to `ServerAccess` interface:

```typescript
// Entitlement seat management
getSubscriptionDetail(id: number): Observable<SubscriptionDetailResponse>;
getEntitlementAssignments(entitlementId: number): Observable<{
    entitlement: Entitlement;
    assignments: EntitlementAssignment[];
}>;
assignEntitlementSeat(entitlementId: number, personId: number): Observable<AssignSeatResponse>;
removeEntitlementAssignment(entitlementId: number, personId: number): Observable<{
    entitlement: Entitlement;
}>;
```

**Note**: `getSubscriptionDetail` replaces the existing `getSubscription` method (or could be a new method that returns the richer response). The existing `getSubscription` remains for list views where entitlement details aren't needed.

- [ ] Add methods to `ServerAccess` interface
- [ ] Implement in `ServerAccessNetwork`
- [ ] Implement in `ServerAccessMock`
- [ ] Add tests to `ServerAccess.mock.spec.ts`

## 5.7 Components

### Entitlement Seat Assignment Control — `SeatAssignmentComponent`

**Location**: `ui/src/app/shared/components/seat-assignment/`

A reusable control for assigning and managing entitlement seats. Inputs: entitlement ID, seats_total, seats_used, current assignments.

**Display**:
- Seat counter: `{seats_used}/{seats_total} seats assigned`
- Progress bar or visual indicator of seat usage
- List of current assignments:
  - Each shows person name and email
  - Remove button (X) next to each assignment
- If seats available (`seats_used < seats_total`):
  - Autocomplete search field for giftable users (calls `searchGiftableUsers`)
  - "Assign to myself" quick button
  - The autocomplete results show first name, last name, email
  - Selecting a user calls `assignEntitlementSeat`
- If no giftable users exist and user wants to assign to someone else:
  - Link to gift permissions page ("Set up sharing first")
- Refresh button to reload assignments (useful after accepting a gift permission)

**Events emitted**:
- `seatAssigned`: when a seat is assigned
- `seatRemoved`: when a seat is removed

```typescript
@Component({
    selector: 'app-seat-assignment',
    templateUrl: './seat-assignment.component.html',
    styleUrls: ['./seat-assignment.component.scss']
})
export class SeatAssignmentComponent {
    @Input() entitlementId!: number;
    @Input() seatsTotal!: number;
    @Input() seatsUsed!: number;
    @Input() assignments: EntitlementAssignment[] = [];

    @Output() seatAssigned = new EventEmitter<AssignSeatResponse>();
    @Output() seatRemoved = new EventEmitter<void>();
}
```

### Subscription Detail — Multi-Seat View

Update `SubscriptionDetailComponent` to:
- Fetch full subscription detail (with entitlement + assignments) using `getSubscriptionDetail`
- If the subscription has a current entitlement, show the `SeatAssignmentComponent`
- For single-seat subscriptions: show seat assignment (user can assign to themselves or gift to another)
- For multi-seat subscriptions: show seat assignment with full counter and multi-user list

### Subscription Signup — Post-Creation Seat Assignment

Update `SubscriptionSignupComponent` to:
- After successful subscription creation with `charge_now = true`, show `SeatAssignmentComponent` for the new entitlement
- For products with `seats_default > 1`, display the seat count in the product info before purchase
- Show a note like "This membership includes {seats_total} seats that you can assign to yourself and others"

### My Subscriptions — Seat Summary

Update `MySubscriptionsComponent` list view to show a brief seat summary for each subscription:
- "{seats_used}/{seats_total} seats assigned" badge on each subscription card
- This requires the list endpoint to include entitlement info, OR a separate call per subscription (batched)

**Pragmatic approach**: For the list view, just show the subscription status and a "Manage" button. The detail view (which already gets enriched with entitlement info) handles all seat management. This avoids N+1 API calls in the list.

- [ ] Create `SeatAssignmentComponent`
- [ ] Update `SubscriptionDetailComponent` to use `getSubscriptionDetail` and show `SeatAssignmentComponent`
- [ ] Update `SubscriptionSignupComponent` for post-creation seat assignment
- [ ] Add routes/navigation as needed

## 5.8 Tests

| Layer | Test File | What to Test |
|-------|-----------|------------|
| KV Conversion | `payment_key_value_table_test.cpp` | `EntitlementAssignmentInfoToKeyValueTable` |
| Business Logic | `subscription_helper_test.cpp` | `GetCurrentEntitlementForSubscription` |
| Endpoints | `entitlement_assignments_test.cpp` | Assign seat, remove assignment, list assignments, permission checks |
| Endpoints | `subscriptions_test.cpp` | Updated detail response includes entitlement + assignments |
| Angular | `seat-assignment.component.spec.ts` | Display, assign, remove, autocomplete |
| Angular | `ServerAccess.mock.spec.ts` | New entitlement assignment mock methods |

---

# Part 6: Non-Thin-Slice Subscription Support

These are the remaining Phase 4 scenarios from Payment Design Document.md not covered by the thin slice (Parts 1-5).

## 6.1 Grace Period Handling (Scenarios 14, 15)

When a subscription auto-pay charge fails:

### Database Changes

Add columns to `subscriptions`:
| Column | Type | Notes |
|--------|------|-------|
| grace_period_ends_us | BIGINT | NULLABLE, set when payment fails |
| grace_period_days | BIGINT | NOT NULL, DEFAULT 7 |

### Business Logic Changes

Update `SubscriptionHelper::ProcessBillingForSubscription`:
1. On payment failure, set `status = "past_due"` and `grace_period_ends_us = now + grace_period_days`
2. Send "payment failed" email with link to update payment method
3. During grace period, entitlement remains valid (permission computation checks grace_period_ends_us)
4. If user updates card and re-billing succeeds, clear grace period
5. If grace period expires, set `status = "expired"`, revoke entitlement

### New Endpoint

`POST /api/subscriptions/<int>/retry_billing` — User triggers a retry after updating their card.

### Email Templates

- "Payment failed" email (with retry instructions)
- "Grace period expiring" email (2 days before expiry)

- [ ] Add grace period columns
- [ ] Update billing logic for failure handling
- [ ] Add retry endpoint
- [ ] Create email templates
- [ ] Update permission computation for grace period
- [ ] Tests

## 6.2 Card Management Enhancements (Scenarios 17, 18, 19, 21)

Mostly covered by Part 1. Remaining items:

### Default payment method (Scenario 19)

Already included in Part 1 (`is_default` on `saved_cards`, `SetDefault` method).

### Expiring card notification (Scenario 21)

Scheduled job (or admin-triggered endpoint) that queries `saved_cards` for cards expiring within the next month and sends notification emails.

### New Endpoint

`POST /api/admin/check_expiring_cards` — Finds cards expiring soon and sends notification emails.

### Email Template

```
Subject: "Your card ending in {last4} is expiring soon"
Body: Card details, link to update in portal
```

- [ ] Create expiring card check endpoint
- [ ] Create email template
- [ ] Tests

## 6.3 Subscription Lifecycle (Scenarios 16, 20)

### Buy next month, get current month free (Scenario 16)

Already supported by the admin subscription creation flow in Part 3. Admin sets:
- `period_start_us` = now
- `period_end_us` = end of next month
- This gives the user the remainder of the current month + all of next month for one payment

No additional code needed — this is a configuration decision at enrollment time.

### Expiring entitlement reminder (Scenario 20)

Scheduled job that queries `entitlements` expiring within 7 days (or configurable) and sends reminder emails. Not subscription-specific — applies to all entitlements.

### New Endpoint

`POST /api/admin/check_expiring_entitlements` — Finds entitlements expiring soon and sends emails.

### Email Template

```
Subject: "Your {product_name} access expires in {days} days"
Body: Entitlement details, link to renew/subscribe
```

- [ ] Create expiring entitlement check endpoint
- [ ] Create email template
- [ ] Tests

## 6.4 Subscription Upgrades/Downgrades (Scenarios 31, 32, 33)

These are Post-MVP / Low Priority per the Payment Design Document. Deferring from this planning document.

**High-level approach when implemented**:
- **Upgrade**: Cancel current subscription, create new one with upgraded product. If mid-period, prorate the difference.
- **Downgrade**: Cancel current subscription effective at period end, create new one starting next period.
- **Reactivate**: Create a new subscription for the same product. Old subscription records remain for audit trail.

---

# Implementation Order Summary

The recommended implementation order across all parts:

| Order | Part | Section | Description | Dependencies |
|-------|------|---------|-------------|-------------|
| 1 | 1.1 | DB Tables | square_customers + saved_cards tables | None |
| 2 | 1.2 | Table Helpers | SquareCustomers + SavedCards helpers + tests | 1 |
| 3 | 1.3 | Utility | Square Client extensions (Customer, Card APIs) + tests | None (parallel with 1-2) |
| 4 | 1.4 | Business Logic | CardHelper + tests | 2, 3 |
| 5 | 1.5 | Endpoints | Card endpoints + tests | 4 |
| 6 | 3.1 | DB Tables | subscriptions + subscription_charges tables | 1 |
| 7 | 3.2 | Table Helpers | Subscriptions + SubscriptionCharges helpers + tests | 6 |
| 8 | 3.3 | Business Logic | SubscriptionHelper + date utils + emails + tests | 4, 7 |
| 9 | 3.4 | Endpoints | Subscription endpoints + tests | 8 |
| 10 | 2.1-2.2 | Integration | Payment control updates (customer_id support) | 4 |
| 11 | 1.6-1.8 | Client | Card types + network + components | 5 |
| 12 | 3.5-3.7 | Client | Subscription types + network + components | 9 |
| 13 | 1.9, 2.2 | Client | Payment method component + refactor existing flows | 11 |
| 14 | 3.8 | Wiring | Permission integration + seed data | 9, 12 |
| 15 | 4.1-4.2 | DB + Helpers | gift_permissions table + helper | 6 |
| 16 | 4.3-4.4 | Business + Endpoints | GiftPermissionHelper + endpoints | 15 |
| 17 | 4.5-4.7 | Client | Gift permission types + network + components | 16 |
| 18 | 4.8 | Wiring | Entitlement assignment validation | 16, 17 |
| 19 | 5.1-5.2 | Backend | Entitlement assignment KV + subscription entitlement lookup | 18 |
| 20 | 5.3 | Endpoint | Subscription detail with entitlement + assignments | 19 |
| 21 | 5.4 | Endpoints | Entitlement seat assignment endpoints + tests | 19 |
| 22 | 5.5-5.6 | Client | Seat assignment types + network + mock | 20, 21 |
| 23 | 5.7 | Components | SeatAssignmentComponent + subscription detail/signup wiring | 22 |
| 24 | 6.1 | Enhancement | Grace period handling | 9 |
| 25 | 6.2-6.3 | Enhancement | Expiring card/entitlement notifications | 5, 9 |

---

# Alternatives Considered

## Q1: Square Subscriptions API vs. Our Own Billing

**Decision**: Our own billing with saved cards.

| Approach | Pros | Cons |
|----------|------|------|
| **Square Subscriptions API** | Square handles billing, retries, invoicing automatically; less server-side logic; users could manage subscriptions in Square's UI | Requires Square Catalog API sync (plans/variations must exist in Square); limited sandbox testing (can't view/edit subscriptions in Sandbox Dashboard); can't simulate card failures in Sandbox; tight coupling to Square; requires webhook callbacks |
| **Our own billing** ✓ | Full control; simpler testing; no Catalog API dependency; matches existing architecture; works with existing CreatePayment; easier to test end-to-end; decoupled from Square (easier to switch providers later) | We handle billing scheduling, retries, and failure handling ourselves |

**Rationale**: The decoupling from Square was a primary driver — if we switch payment providers, our subscription logic doesn't change. Square's Subscriptions API would have been the only feature requiring webhook callbacks, so avoiding it simplifies the server architecture. We already have `CreatePayment` working with retry logic, and Square's Payments API accepts saved card IDs as the `source_id`, so we can charge saved cards through our existing infrastructure. Square Subscriptions API could be revisited later if we need Square's invoicing or automatic retry features — the `subscriptions` table can gain a `square_subscription_id` column and billing logic can delegate to Square.

## Q2: Billing Trigger — Admin UI vs. Server Timer vs. External Cron

**Decision**: Authenticated endpoint callable by admin or external automation process.

| Approach | Complexity | Notes |
|----------|------------|-------|
| **Admin-triggered endpoint** ✓ | Low | `POST /api/admin/run_billing` with `manage_subscriptions` permission |
| **Server-side timer** | Medium | Background thread adds threading complexity to Crow server |
| **External cron** | Low-Medium | Separate process calls the billing endpoint on schedule |

**Rationale**: Starting with an authenticated endpoint that can be called manually by privileged users or by an external automation/watchdog process. A separate watchdog process is planned that would call this endpoint periodically and also monitor server health. Server-side timers were rejected because they add threading complexity to the Crow server architecture.

## Q3: Card Table Naming — `square_cards` vs. `saved_cards`

**Decision**: `saved_cards` (provider-agnostic naming).

**Rationale**: User-facing features like `friendly_name` are provider-agnostic. The portal shouldn't expose "Square" branding. The `square_card_id` column preserves the Square linkage internally. This supports a potential future switch to a different payment provider without renaming the table.

## Q4: Single Card vs. Multiple Cards

**Decision**: Multiple cards with `is_default` flag.

**Rationale**: Minimal extra work over single card. Avoids the awkward UX of having to delete a card before adding a new one.

## Q5: Calendar Month Alignment — Fixed vs. Flexible

**Decision**: Enrollment specifies start date and first period end (Option C).

| Approach | Description |
|----------|-------------|
| **Prorated first month** | Pay for partial month, then full months |
| **Full price for partial month** | Pay full price even if starting mid-month |
| **Enrollment-configured** ✓ | Start date and first period end specified at enrollment time |

**Rationale**: Supports all scenarios — full-month self-service signups, prorated admin enrollments, and the "pay for next month, get rest of this month free" workshop promotion pattern. Self-service users always get standard calendar-month periods; only users with `manage_subscriptions` permission can configure custom start/end dates.

## Q6: Subscription Creation — Admin-Only vs. Both Admin and Self-Service

**Decision**: Both, with permission-gated features.

**Rationale**: Both admin and self-service share most backend logic. The key distinction: only users with `manage_subscriptions` permission can create the "pay for next month, get rest of this month free" enrollment pattern (custom period start/end). Self-service users select a start month/year and get standard calendar-month periods. This permission is not admin-specific — other roles can have it too.

## Q7: Cancellation — Immediate vs. End-of-Period

**Decision**: End-of-period only for the thin slice.

**Rationale**: Simplest approach. Subscription stays active until the current paid period ends, then doesn't renew. No refund is issued. Immediate cancellation with prorated refund is deferred to the refunds phase.

## Q8-Q9: Gift Permission System

**Decisions**:
- Gift permissions apply to **all entitlements** (not just subscriptions) — generic seat assignment control
- Gift permission request sends a **notification email with portal link** — no direct action links in email for security reasons

## Single `cards` table vs. `square_customers` + `saved_cards`

Considered merging into a single table, but the Square customer relationship is 1:1 with person (one customer per person) while cards are 1:many. Keeping them separate matches the Square API structure and avoids denormalization.