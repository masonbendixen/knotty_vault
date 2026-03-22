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

## Dependencies

| Library | Conan Package | Purpose |
|---------|--------------|---------|
| **FTXUI** | `ftxui/6.1.9` | Terminal UI framework — dashboard menus, tables, forms, split views. Pure ANSI sequences, works over SSH. **Zero external dependencies.** |
| **replxx** | `replxx/0.0.4` | Interactive line editor — history, tab completion, syntax highlighting for the command-line mode. **Zero external dependencies** (pthreads on Linux only). |
| **Abseil** | `abseil/20220623.1` (already in conanfile) | Command-line flag parsing for the non-interactive `--command=X` mode. |

Both FTXUI and replxx are available on Conan Center. They're lightweight, cross-platform (Windows + Linux), and header/static-link friendly.

## Dual-Mode Architecture

The tool operates in three modes:

### Mode 1: One-Shot Command (`--command=X`)
Run a single command and exit. No TUI, no REPL. Output goes to stdout. Used for scripting and CI.
```bash
knottyyoga_test_helper --command=list_subscriptions --person_id=3
```

### Mode 2: Dashboard (FTXUI) — Default Interactive Mode
Launch with no `--command` flag. FTXUI takes over the terminal with a full TUI: menus, tables, forms, and a status bar. Navigate with arrow keys, Enter to select, Escape to go back.

Press `:` to drop into Command Mode (replxx). Press `q` or `Ctrl+C` to quit.

### Mode 3: Command Mode (replxx)
Entered from the dashboard via `:`, or launched directly with `--repl`. Full line editing with history (saved to `~/.knottyyoga_test_history`), tab completion on command names and flags, and multi-line support.

Type `dashboard` or `back` to return to FTXUI. Type `quit` or `exit` to exit. Press Enter on an empty prompt to return to the dashboard.

### Terminal Ownership

FTXUI and replxx cannot own the terminal simultaneously. The modal switch works:

1. **FTXUI active**: FTXUI owns the alternate screen buffer and raw input mode.
2. **User presses `:`**: FTXUI calls `screen.Exit()`, restoring the normal terminal.
3. **replxx active**: replxx takes over stdin with line editing, history, completion.
4. **User types `back`/empty Enter**: replxx returns, FTXUI re-enters its event loop.

Both modes share:
- The same `DatabaseHelper` and `TransactionProvider` (kept alive between commands)
- The same command dispatch table (a `std::map<std::string, CommandFn>`)
- The same service instances (secrets, mail, square)

## FTXUI Dashboard Layout

### Main Menu Screen
```
┌─────────────────────────────────────────────────────┐
│  Knotty Yoga Test Helper                    [q]uit  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ▶ Subscriptions                                    │
│    Events & Bookings                                │
│    Users & Permissions                              │
│    Products & Pricing                               │
│    Entitlements                                     │
│    Configuration                                    │
│    Email Testing                                    │
│                                                     │
├─────────────────────────────────────────────────────┤
│  DB: knottyyoga (connected)     [:] command mode    │
└─────────────────────────────────────────────────────┘
```

### Subscriptions Screen (after selecting "Subscriptions")
```
┌─────────────────────────────────────────────────────┐
│  Subscriptions                    [Esc] back [q]uit │
├─────────────────────────────────────────────────────┤
│ ID  Person           Product              Status    │
│ ──  ──────           ───────              ──────    │
│ 12  Mason Bendixen   Monthly Membership   active    │
│ 13  Jane Doe         Gold Membership      past_due  │
│ 14  Bob Smith        Monthly Membership   cancelled │
│                                                     │
├─────────────────────────────────────────────────────┤
│ Actions:                                            │
│ [S] Simulate billing failure  [B] Run billing       │
│ [E] Expire grace periods      [A] Advance billing   │
│ [R] Reset to active           [C] Change product    │
├─────────────────────────────────────────────────────┤
│  ↑↓ navigate  Enter: details  [:] command mode      │
└─────────────────────────────────────────────────────┘
```

### Events & Bookings Screen
```
┌─────────────────────────────────────────────────────┐
│  Events & Bookings                [Esc] back [q]uit │
├─────────────────────────────────────────────────────┤
│ Sessions:                                           │
│ ID  Event              Date          Cap   Booked WL│
│ ──  ─────              ────          ───   ────── ──│
│  1  Intro Workshop     Mar 25 10AM   20    15     2 │
│  2  Yoga Flow          Mar 26 6PM     1     1     3 │
│  3  Partner Yoga       Mar 28 2PM    10     0     0 │
│                                                     │
├─────────────────────────────────────────────────────┤
│ Actions:                                            │
│ [F] Fill to capacity   [T] Change session time      │
│ [W] Process waitlist refunds                        │
│                                                     │
│ Selected session actions (Enter on a row):          │
│ [L] List bookings  [P] Promote waitlist entry       │
│ [X] Cancel booking                                  │
├─────────────────────────────────────────────────────┤
│  ↑↓ navigate  Enter: bookings  [:] command mode     │
└─────────────────────────────────────────────────────┘
```

### Session Bookings Detail (after Enter on a session)
```
┌─────────────────────────────────────────────────────┐
│  Intro Workshop — Mar 25 10:00 AM   [Esc] back      │
│  Capacity: 20  Booked: 15  Waitlisted: 2            │
├─────────────────────────────────────────────────────┤
│ Confirmed:                                          │
│ ID  Person           Booked At                      │
│  5  Alice Smith      Mar 20, 2026                   │
│  6  Bob Jones        Mar 21, 2026                   │
│  ...                                                │
│                                                     │
│ Waitlisted (FIFO):                                  │
│ #  ID  Person           Joined                      │
│ 1  22  Carol White      Mar 22, 2026                │
│ 2  23  Dave Brown       Mar 22, 2026                │
│                                                     │
├─────────────────────────────────────────────────────┤
│ [P] Promote selected  [X] Cancel selected           │
│ [:] command mode                                    │
└─────────────────────────────────────────────────────┘
```

### Users & Permissions Screen
```
┌─────────────────────────────────────────────────────┐
│  Users & Permissions              [Esc] back [q]uit │
├─────────────────────────────────────────────────────┤
│ ID  Name              Email                  Roles  │
│ ──  ────              ─────                  ─────  │
│  1  Mason Bendixen    mason@example.com      admin  │
│  2  Jane Doe          jane@example.com       user   │
│  3  Bob Smith         bob@example.com        user   │
│                                                     │
├─────────────────────────────────────────────────────┤
│ Actions:                                            │
│ [N] Create test user   [R] Assign role              │
│ [P] Grant permission   [D] Save default card        │
├─────────────────────────────────────────────────────┤
│  ↑↓ navigate  Enter: details  [:] command mode      │
└─────────────────────────────────────────────────────┘
```

### Form Example: Create Test User
```
┌─────────────────────────────────────────────────────┐
│  Create Test User                                   │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Email:      [test@example.com                   ]  │
│  First Name: [Test                               ]  │
│  Last Name:  [User                               ]  │
│  Password:   [Password123!                       ]  │
│                                                     │
│  ☐ Assign admin role                                │
│  ☐ Grant manage_products permission                 │
│                                                     │
│         [Create]     [Cancel]                       │
│                                                     │
├─────────────────────────────────────────────────────┤
│  Tab: next field  Enter: submit  Esc: cancel        │
└─────────────────────────────────────────────────────┘
```

## replxx Tab Completion

When in command mode, replxx provides intelligent completion:

| Context | Completion |
|---------|-----------|
| Empty prompt | All command names |
| `list_` | `list_subscriptions`, `list_entitlements`, `list_bookings`, `list_event_sessions`, `list_products` |
| `cancel_booking --` | `--booking_id=`, `--send_real_email` |
| `create_test_user --email=` | No completion (free text) |
| `set_secret --key=` | Secret key names from `secret_keys.h` |

### History

replxx saves command history to `~/.knottyyoga_test_history`. History persists across sessions. Ctrl+R searches history. Up/down arrows navigate history.

### Context-Aware Pre-Population

When dropping from FTXUI into command mode via `:`, the prompt can be pre-populated based on the current dashboard context:

| Dashboard Context | Pre-populated Command |
|---|---|
| Subscription row selected (ID 12) | `simulate_billing_failure --subscription_id=12` |
| Session row selected (ID 1) | `list_bookings --session_id=1` |
| Waitlisted booking selected (ID 22) | `promote_waitlist_entry --booking_id=22` |
| No specific selection | Empty prompt |

## File Structure

```
src/test_helper/
    main.cpp                    Entry point, mode selection, flag parsing
    command_registry.h          Command dispatch table + metadata (name, description, flags)
    command_registry.cpp
    command_runner.h            Executes commands with shared DB/service context
    command_runner.cpp
    commands/
        subscription_commands.h/cpp    Subscription-related commands
        booking_commands.h/cpp         Event/booking/waitlist commands
        user_commands.h/cpp            User creation, role/permission assignment
        product_commands.h/cpp         Product and variant commands
        utility_commands.h/cpp         Secrets, email, general utilities
    dashboard/
        dashboard.h/cpp                FTXUI main loop, screen management
        main_menu.h/cpp                Top-level menu component
        subscription_screen.h/cpp      Subscription list + actions
        booking_screen.h/cpp           Events, sessions, bookings views
        user_screen.h/cpp              User management screen
        form_helpers.h/cpp             Reusable FTXUI form components (text input, checkbox, etc.)
    repl/
        repl.h/cpp                     replxx setup, history, completion, prompt loop
        completer.h/cpp                Tab completion logic (commands, flags, DB values)
    CMakeLists.txt
```

## CMake Changes

### New Conan dependencies (`conanfile.py`)

```python
Library("ftxui", "6.1.9", CMakeInfo("ftxui", "ftxui::ftxui")),
Library("replxx", "0.0.4", CMakeInfo("replxx", "replxx::replxx")),
```

### `src/test_helper/CMakeLists.txt` (new)

```cmake
target_sources(knotty_yoga_test_helper PRIVATE
    command_registry.h
    command_registry.cpp
    command_runner.h
    command_runner.cpp
    commands/subscription_commands.h
    commands/subscription_commands.cpp
    commands/booking_commands.h
    commands/booking_commands.cpp
    commands/user_commands.h
    commands/user_commands.cpp
    commands/product_commands.h
    commands/product_commands.cpp
    commands/utility_commands.h
    commands/utility_commands.cpp
    dashboard/dashboard.h
    dashboard/dashboard.cpp
    dashboard/main_menu.h
    dashboard/main_menu.cpp
    dashboard/subscription_screen.h
    dashboard/subscription_screen.cpp
    dashboard/booking_screen.h
    dashboard/booking_screen.cpp
    dashboard/user_screen.h
    dashboard/user_screen.cpp
    dashboard/form_helpers.h
    dashboard/form_helpers.cpp
    repl/repl.h
    repl/repl.cpp
    repl/completer.h
    repl/completer.cpp
)

add_executable(knottyyoga_test_helper "main.cpp")
target_link_libraries(knottyyoga_test_helper
    knotty_yoga_core
    knotty_yoga_test_helper
    ${ABSL_LIB}
    ${PQXX_LIB}
    ftxui::ftxui
    replxx::replxx
)
```

### Top-level `CMakeLists.txt` changes

1. Add `add_library(knotty_yoga_test_helper "")` alongside the other library targets
2. Add PDB property for Windows: `set_target_properties(knottyyoga_test_helper PROPERTIES LINK_FLAGS "/PDB:knottyyoga_test_helper.pdb")`

### `src/CMakeLists.txt` change

Add `add_subdirectory(test_helper)` alongside existing subdirectories.

## Implementation Approach

### `main.cpp` — Mode Selection

```cpp
#include <absl/flags/flag.h>
#include <absl/flags/parse.h>
#include "command_registry.h"
#include "command_runner.h"
#include "dashboard/dashboard.h"
#include "repl/repl.h"

ABSL_FLAG(std::string, command, "", "Run a single command and exit");
ABSL_FLAG(bool, repl, false, "Start in command-line mode (skip dashboard)");
// ... other flags ...

int main(int argc, char** argv) {
    absl::ParseCommandLine(argc, argv);

    // Shared context: DB connection, services, command table
    auto context = CreateSharedContext();

    std::string command = absl::GetFlag(FLAGS_command);

    if (!command.empty()) {
        // Mode 1: One-shot command
        return context.runner.Execute(command, /* args from flags */);
    }

    if (absl::GetFlag(FLAGS_repl)) {
        // Mode 3: REPL only (no dashboard)
        return RunRepl(context);
    }

    // Mode 2: Dashboard (default) with modal switch to REPL
    return RunDashboard(context);
}
```

### Shared Context

All three modes share a `TestHelperContext`:

```cpp
struct TestHelperContext {
    DatabaseHelper databaseHelper;
    TransactionProviderPtr transactionProvider;
    Secrets::SecretsHelperPtr secretsHelper;
    Mail::MailHelperPtr mailHelper;           // test or real based on flag
    Square::SquareClientPtr squareClient;     // test client
    CommandRegistry registry;                 // command name → function map
    CommandRunner runner;                     // executes commands in transactions
};
```

The context is created once at startup and kept alive for the lifetime of the tool. In interactive mode, this means the DB connection persists across commands — no reconnect overhead.

### Command Registry

```cpp
struct CommandInfo {
    std::string name;
    std::string category;       // "Subscriptions", "Bookings", etc.
    std::string description;
    std::vector<FlagInfo> flags; // for tab completion and help
    CommandFn execute;           // std::function<int(Transaction&, const Args&)>
};

class CommandRegistry {
public:
    void Register(CommandInfo info);
    const CommandInfo* Find(const std::string& name) const;
    std::vector<std::string> GetCommandNames() const;
    std::vector<std::string> GetFlagNames(const std::string& command) const;
    std::vector<CommandInfo> GetByCategory(const std::string& category) const;
    void PrintHelp() const;
    void PrintCommandHelp(const std::string& command) const;
};
```

### Modal Switch: Dashboard ↔ REPL

```cpp
// In dashboard.cpp
void RunDashboard(TestHelperContext& context) {
    while (true) {
        auto action = ShowDashboard(context);  // FTXUI event loop

        if (action == DashboardAction::Quit) break;

        if (action == DashboardAction::CommandMode) {
            // FTXUI has exited, terminal is normal
            auto replAction = RunReplSession(context, action.prePopulated);
            // replxx session done, loop back to FTXUI
            if (replAction == ReplAction::Quit) break;
            continue;  // re-enter FTXUI
        }
    }
}
```

### Output Format

**One-shot mode**: Plain text tables to stdout (pipeable, greppable).

**REPL mode**: Same plain text but with ANSI colors for readability:
- Green: success messages, active status
- Red: errors, failed status, past_due
- Yellow: warnings, waitlisted status
- Cyan: headers, column names
- Gray: cancelled, expired

**Dashboard mode**: FTXUI renders everything — no direct stdout. Tables are FTXUI components with scrolling and selection.

## Implementation Checklist

### Phase 1: Foundation
- [x] Add FTXUI (6.1.9) and replxx (0.0.4) to `conanfile.py`
- [x] Create `src/test_helper/` directory structure (commands/, dashboard/, repl/)
- [x] `main.cpp` — three-mode selection (one-shot, dashboard, repl), absl flags, auto-login
- [x] `command_registry.h/cpp` — command dispatch table with metadata, name/alias lookup, help, parsing
- [x] `command_runner.h/cpp` — shared context with real DB/secrets/Square, test mail, login system
- [x] `repl/repl.cpp` — replxx with history, tab completion, user-aware prompt, modal return
- [x] `dashboard/dashboard.cpp` — FTXUI main menu with modal switch to REPL via `:`
- [x] `CMakeLists.txt` — build configuration linking core, tests, ftxui, replxx
- [x] Update top-level `CMakeLists.txt` (library target, PDB) and `src/CMakeLists.txt` (subdirectory)
- [ ] Verify it builds on Windows (requires `conan install` to fetch new packages)

### Phase 2: Commands
- [x] `commands/subscription_commands.cpp` — list_subscriptions (ls), simulate_billing_failure (sb), run_billing (rb), expire_grace_periods (eg), advance_subscription_billing (ab), reset_subscription
- [x] `commands/booking_commands.cpp` — list_event_sessions (les), list_bookings (lb), simulate_sold_out_event, cancel_booking (cb), promote_waitlist_entry (pw), process_waitlist_refunds, set_event_session_time
- [x] `commands/user_commands.cpp` — create_test_user (cu) with --admin flag, list_users (lu)
- [x] `commands/product_commands.cpp` — list_products (lp), list_entitlements (le)
- [x] `commands/utility_commands.cpp` — set_secret (ss), set_card_expiration, check_expiring_cards, check_expiring_entitlements, send_test_email
- [x] All commands registered in command_runner.cpp via per-category Register functions
- [ ] One-shot mode verified (`--command=X`)

### Phase 3: REPL
- [x] `repl/repl.cpp` — replxx setup, history file, prompt loop (implemented in Phase 1)
- [x] Tab completion for commands, aliases, and `--flag=` names (inline in repl.cpp via CommandRegistry)
- [x] `ParseCommandLine` in `command_registry.cpp` — splits input into command + args
- [x] History persistence — `%USERPROFILE%\.knottyyoga_test_history` / `~/.knottyyoga_test_history`
- [x] `--repl` mode wired in `main.cpp`
- [x] Modal return: empty Enter / `back` → Dashboard, `quit` / Ctrl+D → Quit
- [x] User-aware prompt: `knotty [Mason]> `

### Phase 4: Dashboard
- [ ] `dashboard/dashboard.cpp` — FTXUI main loop, modal switch to REPL
- [ ] `dashboard/main_menu.cpp` — top-level category menu
- [ ] `dashboard/subscription_screen.cpp` — subscription list + action keys
- [ ] `dashboard/booking_screen.cpp` — sessions list → bookings detail drill-down
- [ ] `dashboard/user_screen.cpp` — user list + creation form
- [ ] `dashboard/form_helpers.cpp` — reusable form components
- [ ] Context-aware REPL pre-population (`:` on selected row)
- [ ] Status bar with DB connection info

### Phase 5: Polish
- [ ] ANSI colors in REPL output
- [ ] Error handling for all DB operations (print error, don't crash)
- [ ] `--send_real_email` flag respected across all modes
- [ ] Help system (`help`, `help <command>`, `?` in dashboard)

## Resolved Questions

1. **Square client**: Use the **real Square sandbox client** (not test mocks). The tool should create real sandbox cards, process real sandbox payments, etc. This means reading `square_access_token` and `square_environment` from the production secrets table and constructing a real `SquareClient` + `HttpClient`, same as `main.cpp` does for the web server.

2. **Service implementations**: Use **real secrets** (from the database) and **real Square** (sandbox). Only mail gets the test helper by default (with `--send_real_email` to opt-in to real mail). This means the tool connects to the same database as the web server and reads the same configuration.

3. **FTXUI dependencies**: **None.** FTXUI has zero external dependencies — no Boost, no ncurses, nothing. Pure C++ standard library. Latest version on Conan is 6.1.9 (not 5.0.0 as originally noted). No conflicts with our project.

4. **replxx dependencies**: **None** (just pthreads on Linux, which is a system library). No Boost, no readline, no ncurses. Zero-config, BSD-3-Clause licensed. No conflicts.

## Updated Architecture Based on Resolved Questions

### Service Initialization

Since we're using real services (not test mocks), initialization mirrors `main.cpp`:

```cpp
// Real database connection
DatabaseHelper databaseHelper = MakeProductionDatabaseHelper();
auto transactionProvider = MakeProductionTransactionProvider(databaseHelper);

// Real secrets from the database
Secrets::SecretsHelperPtr secretsHelper;
transactionProvider->RunInTransaction([&](Transaction& transaction) {
    secretsHelper = Secrets::MakeSecretsHelper(databaseHelper);
});

// Real Square client (sandbox)
Square::SquareClientPtr squareClient;
transactionProvider->RunInTransaction([&](Transaction& transaction) {
    std::string token = secretsHelper->LookupSecret(transaction, Secrets::kSquareAccessToken);
    std::string env = secretsHelper->LookupSecret(transaction, Secrets::kSquareEnvironment);
    if (!token.empty()) {
        auto httpClient = Http::MakeHttpClient();
        squareClient = Square::MakeSquareClient(httpClient, token, env == "sandbox");
    }
});

// Test mail by default, real mail with --send_real_email
Mail::MailHelperPtr mailHelper;
if (sendRealEmail) {
    transactionProvider->RunInTransaction([&](Transaction& transaction) {
        mailHelper = Mail::MakeMailHelper(transaction, secretsHelper);
    });
} else {
    mailHelper = Mail::Test::MakeTestMailHelper();
}
```

This means the tool can:
- Create real sandbox cards via Square API
- Process real sandbox payments
- Read/write the same secrets the server uses
- Send real emails when explicitly requested

## Resolved Questions (Round 2)

1. **FTXUI version**: Use **6.1.9** (latest stable).

2. **Card creation**: Use the Square sandbox test card `4111 1111 1111 1111` (any CVC, any future expiration, any zip). The `create_test_user` command should offer an option to save this card for the user via the real Square sandbox API + `CardHelper::CreateCard`. This gives the user a saved card on file for subscription and payment testing without needing the web UI.

3. **Dashboard refresh**: **Auto-refresh** after every action. The dashboard re-queries the DB and updates the visible table/screen immediately after a command completes. No manual refresh needed.

4. **Persistent session / current user**: The tool maintains a "logged-in user" context. On startup, it auto-logs in as `masonbendixen@gmail.com` (looked up by email in the people table). The status bar shows the current user. Commands like `list_subscriptions` default to the current user's person_id. Switch users with `login --email=jane@example.com` or `login --person_id=3`. The `create_test_user` command offers to switch to the newly created user.

5. **Command aliases**: Yes — short aliases for common commands. Aliases work in both REPL and dashboard command bar. Tab completion shows both the alias and the full name.

| Alias | Command |
|-------|---------|
| `ls` | `list_subscriptions` |
| `le` | `list_entitlements` |
| `lb` | `list_bookings` |
| `les` | `list_event_sessions` |
| `lp` | `list_products` |
| `lu` | `list_users` (new — shows people table) |
| `sb` | `simulate_billing_failure` |
| `rb` | `run_billing` |
| `eg` | `expire_grace_periods` |
| `pw` | `promote_waitlist_entry` |
| `cb` | `cancel_booking` |
| `cu` | `create_test_user` |
| `ss` | `set_secret` |
| `ab` | `advance_subscription_billing` |
| `who` | `current_user` (show who's logged in) |

## Default Login Behavior

On startup, the tool:
1. Connects to the database
2. Looks up `masonbendixen@gmail.com` in the people table
3. If found, sets that person as the "current user" and displays it in the status bar
4. If not found (fresh database), prompts: "Default user not found. Use `login --email=X` or `cu` to create a test user."

The current user is shown in:
- The FTXUI status bar: `User: Mason Bendixen (masonbendixen@gmail.com) ID:1`
- The REPL prompt: `knotty [Mason]> `
- One-shot mode header: `Logged in as Mason Bendixen (ID 1)`

Commands that accept `--person_id` use the current user's ID when the flag is omitted:
- `list_subscriptions` → shows subscriptions for current user
- `list_entitlements` → shows entitlements for current user
- `list_bookings` → shows bookings for current user
- `simulate_billing_failure --subscription_id=12` → validates the subscription belongs to current user (or allows `--any_user` flag to override)

---

## Additional Test Scenarios (from Phases 8-10 and codebase analysis)

### 12. `process_waitlist_refunds` — Post-Event Waitlist Cleanup

**What it does**: Calls the waitlist refund business logic that would normally run as a scheduled job. Finds all waitlisted bookings for events that have already passed and cancels them with purchase cancellation.

**Flags**:
- `--send_real_email` (optional)

**Operations**:
1. Query bookings with `status = 'waitlisted'` joined to event_sessions where `end_time_us < now_us()`
2. For each: cancel the booking, cancel the purchase
3. Print count of refunds processed

**Why manual testing needs this**: Events happen at specific times. You can't easily wait for an event to pass to verify the refund job works. This command lets you: (a) create an event session with a past time, (b) add a waitlisted booking to it, (c) run this command to verify the refund fires.

### 13. `simulate_sold_out_event` — Set Up Waitlist Testing

**What it does**: Creates an event session at full capacity with one confirmed booking, so the next booking attempt will be waitlisted. Optionally creates the waitlisted booking too.

**Flags**:
- `--session_id=<id>` (required — existing event session to fill)
- `--create_waitlist_entry` (optional — also create a waitlisted booking for `--person_id`)
- `--person_id=<id>` (required if `--create_waitlist_entry`)

**Operations**:
1. Set `booked_count = capacity` on the event session
2. If `--create_waitlist_entry`: create a purchase + booking with `status = 'waitlisted'` for the person
3. Print session state and any created booking

**Why manual testing needs this**: To test the waitlist flow (joining, promotion, cancel), you need a sold-out event. Through the UI you'd need to create enough accounts to fill the event. This shortcut fills the capacity directly.

### 14. `promote_waitlist_entry` — Promote a Waitlisted Booking

**What it does**: Calls `BookingHelper::AdminPromoteWaitlistEntry()` directly for a specific booking. Promotes a waitlisted person to confirmed with optional capacity increase.

**Flags**:
- `--booking_id=<id>` (required)
- `--increase_capacity` (optional — also increase session capacity by 1)
- `--send_real_email` (optional — send promotion notification)

**Operations**:
1. Call `AdminPromoteWaitlistEntry(transaction, bookingId, increaseCapacity)`
2. Print updated booking status and session capacity

### 15. `cancel_booking` — Cancel a Booking

**What it does**: Calls `BookingHelper::CancelBooking()` directly. For confirmed bookings, auto-promotes the earliest waitlisted person.

**Flags**:
- `--booking_id=<id>` (required)
- `--send_real_email` (optional — send promotion email if auto-promotion occurs)

**Operations**:
1. Call `CancelBooking(transaction, bookingId)`
2. Print: cancelled booking info, whether promotion occurred, promoted booking info
3. Print updated session booked_count

### 16. `set_event_session_time` — Move an Event to Past/Future

**What it does**: Changes an event session's start_time_us and end_time_us. Useful for testing time-dependent scenarios like waitlist refunds (which only fire for past events) and booking validation (which prevents booking past events).

**Flags**:
- `--session_id=<id>` (required)
- `--hours_offset=<n>` (required — positive = future, negative = past, relative to now)

**Operations**:
1. Calculate new start/end times: `now + hours_offset * 3600000000` (preserving the original duration)
2. Update event_sessions `start_time_us` and `end_time_us`
3. Print the updated session times

**Why manual testing needs this**: You can't easily create a past event session through the UI (the create form validates future dates). This lets you move an event to the past to test waitlist refunds and other time-dependent flows.

### 17. `list_bookings` — View Booking State

**What it does**: Lists bookings, optionally filtered by event session or person.

**Flags**:
- `--session_id=<id>` (optional — filter by session)
- `--person_id=<id>` (optional — filter by person)

**Operations**:
1. Query bookings table with optional filters
2. Join with people for names, event_sessions + products for event names
3. Print each booking: id, person name, event name, status, created_us, waitlist position

### 18. `create_test_user` — Create a User for Testing

**What it does**: Creates a fully validated user (bypassing email verification) with a specified email, name, and password.

**Flags**:
- `--email=<email>` (required)
- `--first_name=<name>` (optional, default "Test")
- `--last_name=<name>` (optional, default "User")
- `--password=<password>` (optional, default "Password123!")

**Operations**:
1. Call `PersonHelper::CreateFullyValidatedUser()`
2. Print the created person's ID and email

**Why manual testing needs this**: Creating users through the registration flow requires email verification. For rapid testing of multi-user scenarios (like waitlist with multiple people), this shortcut is essential.

### 19. `advance_subscription_billing` — Make a Subscription Billable

**What it does**: Sets a subscription's `next_billing_us` to a time in the past so the next `run_billing` invocation will process it.

**Flags**:
- `--subscription_id=<id>` (required)

**Operations**:
1. Validate the subscription exists and is active with a `next_billing_us` set
2. Set `next_billing_us = now - 1 hour` and advance `current_period_start_us`/`current_period_end_us` to the next month
3. Print the updated subscription dates

**Why manual testing needs this**: Subscription billing only processes subscriptions whose `next_billing_us` is in the past. You'd have to wait a full billing cycle to test it naturally.

### 20. `reset_subscription` — Reset a Subscription to Active

**What it does**: Resets a cancelled, expired, or past_due subscription back to active status. Useful for re-testing flows without recreating the subscription.

**Flags**:
- `--subscription_id=<id>` (required)

**Operations**:
1. Set `status = 'active'`, clear `cancelled_us`, `cancel_reason`, `grace_period_ends_us`
2. Set `current_period_start_us = now`, `current_period_end_us = start of next month`
3. Print the updated subscription

### 21. `list_event_sessions` — View Event Session State

**What it does**: Lists event sessions, showing capacity, booked count, status, and waitlist info.

**Flags**:
- `--product_id=<id>` (optional — filter by product)
- `--include_past` (optional — also show past sessions, default only future)

**Operations**:
1. Query event_sessions joined with products
2. For each session, count waitlisted bookings
3. Print: id, product name, start time, capacity, booked/capacity, waitlisted count, status

### 22. `create_variant_prices` — Bulk Set Variant Prices

**What it does**: Sets prices for all active variants of a product in the current active price schedule.

**Flags**:
- `--product_id=<id>` (required)
- `--prices=<amount1,amount2,...>` (required — one per variant in sort_order, in dollars e.g. "160.00,220.00,300.00")

**Operations**:
1. Look up the active price schedule
2. Look up active variants for the product sorted by sort_order
3. For each variant: create or update the product_price entry
4. Print the variant names and their new prices

**Why manual testing needs this**: Setting up variant pricing through the admin UI's pricing matrix is tedious for multiple variants across schedules. This bulk command speeds up test setup.

### 23. `simulate_payment_failure` — Test Failed Payment Flows

**What it does**: Creates a subscription charge record with `status = 'failed'` and sets the subscription to `past_due` with a grace period. This simulates what happens when the Square payment API returns a failure during automatic billing.

**Flags**:
- `--subscription_id=<id>` (required)
- `--failure_reason=<text>` (optional, default "Card declined — test simulation")

**Operations**:
1. Create a subscription_charges record with `status = 'failed'`, `failure_reason`
2. Set subscription `status = 'past_due'`, set `grace_period_ends_us`
3. Print the subscription and charge details

**Why manual testing needs this**: Without this, you'd need to actually set up a card that declines (difficult in sandbox) to test the past_due → retry → grace period → expire flow.

### 24. `send_test_email` — Verify Email Configuration

**What it does**: Sends a test email to verify the mail server configuration is working.

**Flags**:
- `--to=<email>` (required)
- `--template=<name>` (optional — "payment_confirmation", "booking_confirmation", "waitlist_confirmation", "waitlist_promotion", "subscription_created". Default: simple test message)

**Operations**:
1. Create a mail helper using production settings
2. Generate the email body (either a simple test message or the specified template with dummy data)
3. Send the email
4. Print success/failure

## Updated Command List

```
knottyyoga_test_helper --command=<command_name> [options]

Commands:
  help                          List all commands and descriptions

  --- Subscription Testing ---
  list_subscriptions            Show all subscriptions (optionally for a person)
  simulate_billing_failure      Set an active subscription to past_due with grace period
  simulate_payment_failure      Create a failed charge record and set past_due
  run_billing                   Process all due subscription billing cycles
  expire_grace_periods          Expire past_due subscriptions whose grace period ended
  advance_subscription_billing  Make a subscription billable by setting next_billing to past
  reset_subscription            Reset a cancelled/expired subscription to active
  change_subscription_product   Upgrade or downgrade a subscription's product

  --- Card Testing ---
  set_card_expiration           Change a saved card's expiration date
  check_expiring_cards          Find and notify about cards expiring soon

  --- Entitlement Testing ---
  list_entitlements             Show all entitlements (optionally for a person)
  check_expiring_entitlements   Find and notify about entitlements expiring soon

  --- Event / Booking / Waitlist Testing ---
  list_event_sessions           Show event sessions with capacity and waitlist info
  list_bookings                 Show bookings filtered by session or person
  simulate_sold_out_event       Fill an event to capacity for waitlist testing
  cancel_booking                Cancel a booking (with auto-promotion if waitlisted)
  promote_waitlist_entry        Promote a waitlisted booking to confirmed
  process_waitlist_refunds      Refund remaining waitlisted bookings for past events
  set_event_session_time        Move an event session to past or future

  --- Product / Variant Testing ---
  list_products                 Show all products (optionally filter by kind)
  create_variant_prices         Bulk set variant prices for a product

  --- Utility ---
  create_test_user              Create a fully validated test user
  set_secret                    Set a configuration secret value
  send_test_email               Send a test email to verify mail configuration

Common flags:
  --command=<name>              Command to run (required)
  --person_id=<id>              Person ID (for commands that need it)
  --subscription_id=<id>        Subscription ID
  --session_id=<id>             Event session ID
  --booking_id=<id>             Booking ID
  --product_id=<id>             Product ID
  --send_real_email             Actually send emails (default: capture only)
  --help / -?                   Show help
```