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

 I'd like to have command line switches to be able to run various test case scenarios and have a -? -help that lists these options. There are a number of manual test scenarios that can't be done through the webpage but need to be forced by running SQL commands against the database or CURL commands against the server. It would be nice to add those cases to this binary.

In particular, look at:

[[Subscriptions- Recurring billing and card management]]

Look at the various stages that can't be tested through the website and add sections in this document to enable performing those various manual tests.

# Plan

## Purpose

A standalone command-line executable (`knottyyoga_test_helper`) that performs manual test scenarios which can't be exercised through the website alone. These scenarios require direct database manipulation, calling admin-only business logic, or both in sequence.

The tool connects directly to the database (like `knottyyoga_database_helper`) and calls business logic code directly — no running server required. This makes it self-contained: one command sets up state, runs the operation, and prints results.

## Architecture

### Approach: Direct database + business logic (no HTTP)

Rather than making HTTP calls to a running server, this tool:
1. Connects to PostgreSQL directly via `MakeProductionDatabaseHelper()`
2. Creates a `ProductionTransactionProvider` for transaction support
3. Instantiates business logic helpers (`SubscriptionHelper`, `EntitlementHelper`, etc.) directly
4. Creates real or test implementations of services (secrets, mail, square) as needed

**Why not HTTP/curl?** The tool needs to both manipulate database state (SQL) and trigger business logic. Doing this through HTTP would require a running server, session authentication, admin permissions, and two separate tools. Direct access is simpler and more reliable for testing.

### Pattern: follows `knottyyoga_database_helper`

- New directory: `src/test_helper/`
- New library target: `knotty_yoga_test_helper`
- New executable: `knottyyoga_test_helper`
- Uses Abseil flags for command-line parsing
- Links against `knotty_yoga_core`

### Mail behavior

For test scenarios that trigger emails (billing, notifications), the tool should use `MakeTestMailHelper()` by default so emails aren't actually sent during testing. A `--send_real_email` flag can opt-in to real email sending via `MakeMailHelper()`.

## Command-Line Interface

```
knottyyoga_test_helper --command=<command_name> [options]

Commands:
  help                          List all commands and descriptions
  list_subscriptions            Show all subscriptions (optionally for a person)
  list_entitlements             Show all entitlements (optionally for a person)
  simulate_billing_failure      Set an active subscription to past_due with grace period
  run_billing                   Process all due subscription billing cycles
  expire_grace_periods          Expire past_due subscriptions whose grace period ended
  check_expiring_cards          Find and notify about cards expiring soon
  check_expiring_entitlements   Find and notify about entitlements expiring soon
  set_card_expiration           Change a saved card's expiration date
  change_subscription_product   Upgrade or downgrade a subscription's product
  set_secret                    Set a configuration secret value
  list_products                 Show all subscription products

Common flags:
  --command=<name>              Command to run (required)
  --person_id=<id>              Person ID (for commands that need it)
  --subscription_id=<id>        Subscription ID (for commands that need it)
  --send_real_email             Actually send emails (default: capture only)
  --help / -?                   Show help
```

## Test Scenarios (from Subscription Planning Document)

### 1. `simulate_billing_failure` — Grace Period Flow Setup

**What it does**: Takes an active subscription and sets it to `past_due` with a grace period, simulating what happens when automatic billing fails. This is the setup step for testing the Retry Payment button in the UI.

**Flags**:
- `--subscription_id=<id>` (required)
- `--grace_days=<n>` (optional, default: read from `subscription_grace_period_days` secret, fallback 7)

**Operations**:
1. Validate the subscription exists and is active
2. Look up grace period days from secrets (or use `--grace_days` override)
3. Update subscription: `status = 'past_due'`, `grace_period_ends_us = now + grace_days`
4. Print the updated subscription details

**From planning doc** (Grace Period Flow manual testing): This replaces the manual SQL `UPDATE subscriptions SET status = 'past_due', grace_period_ends_us = ...`

### 2. `run_billing` — Process Due Subscriptions

**What it does**: Calls `SubscriptionHelper::RunBilling()` directly. Processes all active subscriptions whose `next_billing_us` is in the past.

**Flags**:
- `--send_real_email` (optional — send billing confirmation/failure emails)

**Operations**:
1. Create SubscriptionHelper with test Square client (queued success responses) and test/real mail helper
2. Call `RunBilling(transaction)`
3. Print results: total due, total charged, total failed, per-subscription details

**Note**: Uses a test Square client with queued success results by default. A `--use_real_square` flag could be added later if needed, but for manual testing purposes the test client is safer.

### 3. `expire_grace_periods` — Expire Past-Due Subscriptions

**What it does**: Calls `SubscriptionHelper::ExpireGracePeriods()`. Finds all `past_due` subscriptions whose grace period has ended, sets them to `expired`, and revokes their entitlements.

**Flags**: (none required)

**Operations**:
1. Create SubscriptionHelper
2. Call `ExpireGracePeriods(transaction)`
3. Print count of expired subscriptions

### 4. `check_expiring_cards` — Card Expiration Notifications

**What it does**: Calls the card expiring notification business logic. Finds saved cards expiring in the current or next month and sends (or captures) notification emails.

**Flags**:
- `--send_real_email` (optional)

**Operations**:
1. Create `CardExpiringNotification` helper
2. Call `CheckAndNotify(transaction)`
3. Print results: cards found, notifications sent

### 5. `check_expiring_entitlements` — Entitlement Expiry Reminders

**What it does**: Calls the entitlement expiring notification business logic. Finds active entitlements expiring within the configured window and sends (or captures) reminder emails.

**Flags**:
- `--send_real_email` (optional)

**Operations**:
1. Create `EntitlementExpiringNotification` helper
2. Call `CheckAndNotify(transaction)`
3. Print results: entitlements found, notifications sent

### 6. `set_card_expiration` — Modify Card Expiry for Testing

**What it does**: Changes a saved card's expiration month/year. Useful for testing expiring card notifications without waiting for a real card to expire.

**Flags**:
- `--card_id=<id>` (required — or `--person_id` to find their cards)
- `--exp_month=<1-12>` (required)
- `--exp_year=<yyyy>` (required)

**Operations**:
1. Validate the card exists
2. Update `exp_month` and `exp_year` on the saved card
3. Print the updated card details

**From planning doc**: Replaces `UPDATE saved_cards SET exp_month = 3, exp_year = 2026 ...`

### 7. `change_subscription_product` — Upgrade/Downgrade

**What it does**: Calls `SubscriptionHelper::ChangeSubscriptionProduct()`. Changes a subscription from one product to another.

**Flags**:
- `--subscription_id=<id>` (required)
- `--new_product_id=<id>` (required)
- `--immediate` (flag — true = upgrade now, false/absent = downgrade deferred)
- `--send_real_email` (optional)

**Operations**:
1. Create SubscriptionHelper
2. Look up subscription to get the person_id
3. Call `ChangeSubscriptionProduct(transaction, subscriptionId, personId, newProductId, immediate)`
4. Print old subscription, new subscription, and new entitlement (if any)

### 8. `set_secret` — Configure Test Parameters

**What it does**: Sets a secret/configuration value in the database. Useful for tuning grace period duration, entitlement reminder window, etc.

**Flags**:
- `--key=<secret_name>` (required)
- `--value=<secret_value>` (required)

**Operations**:
1. Insert or update the secret in the `secrets` table
2. Print confirmation

**Common secrets for testing**:
- `subscription_grace_period_days` — grace period duration (default 7)
- `entitlement_expiry_reminder_days` — reminder window (default 7)

### 9. `list_subscriptions` — View Subscription State

**What it does**: Lists all subscriptions, or subscriptions for a specific person.

**Flags**:
- `--person_id=<id>` (optional — filter to one person)

**Operations**:
1. Query subscriptions table (all or filtered)
2. Print each subscription: id, person_id, product name, status, period dates, next billing date, cancel reason

### 10. `list_entitlements` — View Entitlement State

**What it does**: Lists all entitlements, or entitlements for a specific person (via assignments).

**Flags**:
- `--person_id=<id>` (optional — filter to one person)

**Operations**:
1. Query entitlements table (all or filtered via entitlement_assignments join)
2. Print each entitlement: id, product name, status, valid from/to, seats, assigned people

### 11. `list_products` — View Available Products

**What it does**: Lists all subscription-type products with their prices.

**Operations**:
1. Query products where type = 'subscription'
2. Join with product_prices for current pricing
3. Print each product: id, code, name, price

## File Structure

```
src/test_helper/
    main.cpp                    Entry point, flag definitions, command dispatch
    test_helper_commands.h      Command function declarations
    test_helper_commands.cpp    Command implementations
    CMakeLists.txt              Build config
```

## CMake Changes

### `src/test_helper/CMakeLists.txt` (new)

```cmake
target_sources(knotty_yoga_test_helper
    PRIVATE
    test_helper_commands.h
    test_helper_commands.cpp
)

add_executable(knottyyoga_test_helper "main.cpp")
target_link_libraries(knotty_yoga_test_helper ${ABSL_LIB} ${PQXX_LIB} knotty_yoga_core)
target_link_libraries(knottyyoga_test_helper ${ABSL_LIB} knotty_yoga_core knotty_yoga_test_helper)
```

### Top-level `CMakeLists.txt` changes

1. Add `add_library(knotty_yoga_test_helper "")` alongside the other library targets
2. Add PDB property for Windows: `set_target_properties(knottyyoga_test_helper PROPERTIES LINK_FLAGS "/PDB:knottyyoga_test_helper.pdb")`

### `src/CMakeLists.txt` change

Add `add_subdirectory(test_helper)` alongside existing subdirectories.

## Implementation Approach

### `main.cpp` — Flag definitions and dispatch

```cpp
#include <iostream>
#include <absl/flags/flag.h>
#include <absl/flags/parse.h>
#include "test_helper_commands.h"

ABSL_FLAG(std::string, command, "", "Command to run (use 'help' to list)");
ABSL_FLAG(int64_t, subscription_id, 0, "Subscription ID");
ABSL_FLAG(int64_t, person_id, 0, "Person ID");
ABSL_FLAG(int64_t, card_id, 0, "Card ID");
ABSL_FLAG(int64_t, new_product_id, 0, "New product ID");
ABSL_FLAG(int32_t, exp_month, 0, "Expiration month (1-12)");
ABSL_FLAG(int32_t, exp_year, 0, "Expiration year");
ABSL_FLAG(int32_t, grace_days, 0, "Grace period days override");
ABSL_FLAG(bool, immediate, false, "Immediate change (upgrade)");
ABSL_FLAG(bool, send_real_email, false, "Send real emails instead of capturing");
ABSL_FLAG(std::string, key, "", "Secret key name");
ABSL_FLAG(std::string, value, "", "Secret value");

int main(int argc, char** argv) {
    absl::ParseCommandLine(argc, argv);
    std::string command = absl::GetFlag(FLAGS_command);
    // dispatch to command functions...
}
```

### `test_helper_commands.cpp` — Command implementations

Each command is a standalone function that:
1. Creates `DatabaseHelper` via `MakeProductionDatabaseHelper()`
2. Creates `TransactionProvider` via `MakeProductionTransactionProvider()`
3. Runs in a transaction
4. Creates the necessary helpers with test or real service implementations
5. Calls the business logic
6. Prints results to stdout

### Output format

Plain text, human-readable. Each command prints a header and then key-value or tabular output. Example:

```
=== Subscriptions for Person 3 ===
ID    Product              Status     Period Start          Period End            Next Billing
---   -------              ------     ------------          ----------            ------------
12    Monthly Membership   active     2026-03-01            2026-04-01            2026-04-01
13    Gold Membership      cancelled  2026-02-01            2026-03-01            --
```

## Implementation Checklist

- [ ] Create `src/test_helper/` directory
- [ ] `main.cpp` — flag definitions, command dispatch, help text
- [ ] `test_helper_commands.h` — command function declarations
- [ ] `test_helper_commands.cpp` — all command implementations
- [ ] `CMakeLists.txt` — build configuration
- [ ] Update top-level `CMakeLists.txt` — add library target and PDB property
- [ ] Update `src/CMakeLists.txt` — add_subdirectory
- [ ] Verify it builds on Windows

## Questions / Decisions Needed

1. **Square client for `run_billing`**: The test Square client with queued success responses is safe but doesn't exercise real payment. Is there a scenario where you'd want `--use_real_square` to actually charge sandbox cards? (Can add later if needed.)

2. **Output format**: Plain text tables as shown above, or would JSON output be useful for scripting? (Can start with plain text and add `--json` later.)

3. **Should this link against `knotty_yoga_tests` too?** The test mail helper, test secrets helper, and test square client all live in that library. Currently `knotty_yoga_tests` is only used by the test executable, but we'd need it here for the test service implementations. Alternatively, we could move the test util factories into `knotty_yoga_core` (less clean) or create a separate `knotty_yoga_test_utils` library.

4. **Any additional scenarios** beyond what's in the subscription planning doc? For example:
   - Create a test subscription from scratch (person + product + card + subscription)
   - Advance time on a subscription (set next_billing_us to the past to make it billable)
   - Reset a subscription back to active from expired/cancelled