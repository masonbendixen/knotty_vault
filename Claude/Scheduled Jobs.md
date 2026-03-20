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

Look at the document: Subscriptions- Recurring billing and card management.md and the Scheduled Jobs section at the end of the document. I want to build a plan for a helper executable that does scheduled jobs and acts as a server watchdog. Please move the content from that document to here. Look at these documents:

- [[Payment Design Document]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]

And the code base for other ideas for things that need to be called from the helper process. For the watchdog process, I feel like there should be some kind of ping endpoint that this watchdog process calls every so often. If that doesn't return a response within a configurable interval, it should kill the webserver process and restart it. The name of the executable for the web process should be a configurable setting. The watchdog process should also spawn a separate instance that pings the first instance using some mechanism. If that doesn't return a response within a configurable interval, it should kill the other process, take over the watchdog responsibility, and kick off a new watchdog process to watch the watchdog. This should ideally be portable code that runs on windows and linux. Ideally we use the standard library and boost (or come up with other libraries on conan) to do this process orchestration support.

# Scheduled Jobs & Watchdog Helper — Design Plan

## Executive Summary

A single new executable (`knottyyoga_helper`) that serves two purposes:
1. **Scheduled Job Runner** — Calls the web server's admin endpoints on configurable intervals to process billing, notifications, cleanup, and reminders
2. **Server Watchdog** — Monitors the web server process health and restarts it on failure, with a self-healing watchdog-of-watchdog architecture

The executable runs in two modes (via command-line flag): **primary mode** (scheduler + web server watcher) and **watchdog mode** (watches the primary process). The primary spawns a watchdog; the watchdog monitors the primary. If either dies, the survivor takes corrective action.

---

# 1. Scheduled Jobs Inventory

## 1.1 Existing Admin Endpoints (from Subscriptions document)

These endpoints are fully implemented in the web server and just need an external caller.

| # | Endpoint | Method | Frequency | Recommended Time | Purpose |
|---|----------|--------|-----------|-----------------|---------|
| 1 | `/api/admin/run_billing` | POST | Daily | 1:00 AM | Process all active subscriptions whose `next_billing_us` is past due. Creates purchases, charges saved cards, advances subscriptions or sets `past_due` with grace period. |
| 2 | `/api/admin/expire_grace_periods` | POST | Daily | 1:30 AM (after billing) | Finds `past_due` subscriptions past their grace period, sets to `expired`, revokes entitlements. |
| 3 | `/api/admin/check_expiring_entitlements` | POST | Daily | 2:00 AM | Sends reminder emails for entitlements expiring within configurable window (default 7 days, via `entitlement_expiry_reminder_days` secret). |
| 4 | `/api/admin/check_expiring_cards` | POST | Monthly | 1st of month, 3:00 AM | Notifies users whose saved cards expire this month or next. |

All four require `manage_subscriptions` permission. All are idempotent — running multiple times in a day causes no harm (though notification endpoints may send duplicate emails).

**Source**: `endpoints/admin_run_billing.cpp`, `endpoints/admin_expire_grace_periods.cpp`, `endpoints/admin_check_expiring_entitlements.cpp`, `endpoints/admin_check_expiring_cards.cpp`

## 1.2 New Endpoints Needed

These scheduled tasks don't have admin endpoints yet. Each needs a new endpoint on the web server that the helper can call.

| # | Proposed Endpoint | Method | Frequency | Purpose | Source Reference |
|---|-------------------|--------|-----------|---------|-----------------|
| 5 | `/api/admin/send_event_reminders` | POST | Hourly | Find bookings for events starting within the `event_reminder_hours` window (default 24h, via secret) that haven't had a reminder sent yet. Send reminder emails. Mark bookings as reminded. | Scenario 13 in Support for scheduled purchases. Secret key `kEventReminderHours` exists in `secret_keys.h`. Reminder emails are the #1 SHOULD HAVE item. |
| 6 | `/api/admin/cleanup_expired_tokens` | POST | Daily | Delete expired device tokens (older than `kAuthDeviceTokenMaxDurationInMicros`) and expired email verifications (older than `kEmailVerificationExpirationWindowInMicros`). Currently these are only checked at use time, not cleaned up. | `device_tokens` table has `created_us`/`last_seen_us`; `email_verifications` has expiration. Neither has cleanup. |
| 7 | `/api/admin/cleanup_idempotency_keys` | POST | Daily | Delete expired idempotency keys (where `expires_us` is past). Prevents unbounded table growth. | `idempotency_keys` table has `expires_us` column. `IdempotencyHelper` checks expiry but doesn't reap old records. |
| 8 | `/api/admin/cleanup_scaled_photos` | POST | Daily | Delete scaled photo cache entries older than `kScaledPhotoMaxAgeUs`. Secret keys for interval (`kScaledPhotoReaperIntervalUs`, default 24h) and max age (`kScaledPhotoMaxAgeUs`) already exist. | `secret_keys.h` defines both configuration keys. No reaper implementation exists yet. |
| 12 | `/api/admin/process_waitlist_refunds` | POST | Hourly | Find all events where `end_time_us < now_us()` that have waitlisted bookings. For each remaining waitlisted booking, process a full refund and set status to cancelled with reason "Event passed — waitlist refund". Idempotent. | Phase 10 Waitlist Management in Product, Event, and Subscription Admin Portal.md. |

## 1.3 Future Scheduled Jobs (Not Yet Implementable)

These will become relevant as more features are built. Listed here for completeness — no endpoints or business logic exist yet.

| # | Job | Frequency | Depends On | Purpose |
|---|-----|-----------|------------|---------|
| 9 | Generate provider availability from schedule templates | Weekly or on-demand | Schedule templates feature (STRETCH tier scenarios 49-54) | Batch-generate concrete `provider_availability` entries from `schedule_templates` for a configurable scheduling window (e.g., 12 weeks out). |
| 10 | **Waitlist refund processing** | **Hourly** | **Waitlist feature (Phase 10)** | **For events that have passed, auto-refund any remaining waitlisted bookings that weren't promoted. Endpoint: `POST /api/admin/process_waitlist_refunds`. Finds events where `end_time_us < now_us()` with waitlisted bookings, processes full refund for each, sets booking status to cancelled with reason "Event passed — waitlist refund". Idempotent.** |
| 11 | No-show marking | Daily (morning after) | Check-in feature (STRETCH scenario 61) | Mark bookings where the event/service has passed and the attendee was never checked in as "no_show". |

---

# 2. Watchdog Architecture

## 2.1 Web Server Health Check

The web server needs a lightweight health check endpoint.

**New endpoint**: `GET /api/health`
- **No authentication required** (the helper doesn't need to log in for health checks)
- Returns `200 OK` with a minimal JSON body: `{"status": "ok", "uptime_seconds": 12345}`
- Should be as simple as possible — no database calls, no business logic
- Just confirms the Crow server is alive and accepting HTTP requests

The helper calls this endpoint on a configurable interval (default: every 30 seconds). If the endpoint doesn't respond within a configurable timeout (default: 10 seconds), the helper considers the server unhealthy. After a configurable number of consecutive failures (default: 3), the helper kills the web server process and restarts it.

## 2.2 Self-Healing Process Pair

The architecture uses two instances of the same executable running simultaneously:

```
┌─────────────────────────────────────────────────┐
│  PRIMARY PROCESS (knottyyoga_helper)             │
│                                                  │
│  Responsibilities:                               │
│  1. Run scheduled jobs on their intervals        │
│  2. Ping web server health endpoint              │
│  3. Kill/restart web server if unhealthy         │
│  4. Listen on a TCP port for watchdog pings      │
│  5. Respond to watchdog pings                    │
│  6. Monitor watchdog child — respawn if it dies  │
│                                                  │
│  Spawns ──► WATCHDOG PROCESS                     │
└─────────────────────────────────────────────────┘
          ▲                           │
          │ TCP ping                  │ TCP ping
          │ (health check)            ▼
┌─────────────────────────────────────────────────┐
│  WATCHDOG PROCESS (knottyyoga_helper --watchdog) │
│                                                  │
│  Responsibilities:                               │
│  1. Ping primary process on its TCP port         │
│  2. If primary doesn't respond:                  │
│     a. Kill primary process                      │
│     b. Become the new primary (take over all     │
│        primary responsibilities)                 │
│     c. Spawn a new watchdog process              │
│                                                  │
│  Does NOT run scheduled jobs while in            │
│  watchdog mode — only monitors.                  │
└─────────────────────────────────────────────────┘
```

### Failure Scenarios

**Web server crashes**:
1. Primary detects health check failure (consecutive misses exceed threshold)
2. Primary kills web server process (if still running)
3. Primary restarts web server process
4. Primary resumes health checks

**Primary process crashes**:
1. Watchdog detects ping failure on TCP port
2. Watchdog kills primary process (if still running)
3. Watchdog promotes itself to primary mode:
   - Starts running scheduled jobs
   - Starts monitoring web server
   - Opens TCP listen port for watchdog pings
4. Watchdog spawns a new watchdog child

**Watchdog process crashes**:
1. Primary detects its watchdog child process has exited
2. Primary spawns a new watchdog child
3. (Primary never stops running — this is just resilience)

**Both crash simultaneously** (e.g., machine reboot):
- An OS-level service manager (systemd on Linux, Windows Service, or Task Scheduler) should be configured to start the primary process on boot
- The primary will then spawn its watchdog as usual

## 2.3 Inter-Process Communication: TCP Ping

The primary process runs a lightweight TCP listener on a configurable port (default: 18090). The protocol is minimal:

1. Watchdog connects to `localhost:{port}`
2. Watchdog sends: `PING\n`
3. Primary responds: `PONG\n`
4. Connection closed

This uses **Boost.Asio** (already available — `boost::asio` is used by `ThreadPool`). Boost.Asio provides cross-platform async TCP, which works identically on Windows and Linux.

If the primary doesn't respond within the timeout, the watchdog counts it as a failure. After consecutive failures exceed the threshold, the watchdog takes over.

---

# 3. Executable Design

## 3.1 Command-Line Interface

Using `absl::flags` (same pattern as `knottyyoga_database_helper`):

```
knottyyoga_helper [flags]

Mode flags:
  --watchdog                  Run in watchdog mode (monitor primary process)
                              Default: false (run as primary)

Web server watchdog flags:
  --server_executable         Path to web server executable
                              Default: "knottyyoga_the_server"
  --server_url                Web server base URL for health checks and API calls
                              Default: "http://localhost:18080"
  --server_health_interval    Seconds between web server health checks
                              Default: 30
  --server_health_timeout     Seconds to wait for health check response
                              Default: 10
  --server_health_failures    Consecutive failures before restart
                              Default: 3
  --manage_server             Whether to manage (start/restart) the web server process
                              Default: true

Inter-process watchdog flags:
  --watchdog_port             TCP port for primary/watchdog communication
                              Default: 18090
  --watchdog_interval         Seconds between watchdog pings
                              Default: 15
  --watchdog_timeout          Seconds to wait for ping response
                              Default: 5
  --watchdog_failures         Consecutive failures before takeover
                              Default: 3

Scheduler flags:
  --service_account_email     Email for authenticating to admin endpoints
                              Default: "scheduler@knottyyoga.local"
  --service_account_password  Password for the service account
                              (should be set via environment variable in production)

Scheduling interval overrides (seconds, 0 = disabled):
  --billing_interval          Default: 86400 (daily)
  --grace_period_interval     Default: 86400 (daily)
  --expiring_entitlements_interval  Default: 86400 (daily)
  --expiring_cards_interval   Default: 2592000 (monthly)
  --event_reminders_interval  Default: 3600 (hourly)
  --token_cleanup_interval    Default: 86400 (daily)
  --idempotency_cleanup_interval  Default: 86400 (daily)
  --photo_cleanup_interval    Default: 86400 (daily)
```

## 3.2 Authentication Strategy

The helper authenticates to the web server via HTTP, using the existing auth system:

1. On startup, the helper calls `POST /api/login` with the service account credentials
2. Stores the session cookie
3. Uses the cookie for all subsequent admin endpoint calls
4. If a call returns `401`, re-authenticates and retries

The **service account** is a regular `people` entry with the `manage_subscriptions` permission (and any other permissions needed for future admin endpoints). It should be created as part of database initialization.

This approach:
- Uses the existing auth infrastructure (no new auth mechanism)
- Works over HTTP (the helper doesn't need direct database access for scheduled jobs)
- Service account permissions are managed through the existing RBAC system

## 3.3 Scheduler Loop

The primary process runs a single main loop using **Boost.Asio** timers:

```
┌──────────────────────────────────────────────────────────────┐
│  boost::asio::io_context                                      │
│                                                               │
│  Timer: server_health_check (every 30s)                       │
│    → GET /api/health                                          │
│    → On failure: increment counter, kill/restart if threshold  │
│                                                               │
│  Timer: watchdog_tcp_listener (always-on)                      │
│    → Accept connections, respond to PING with PONG             │
│                                                               │
│  Timer: billing (every 24h, aligned to 1:00 AM)               │
│    → POST /api/admin/run_billing                              │
│                                                               │
│  Timer: grace_periods (every 24h, aligned to 1:30 AM)         │
│    → POST /api/admin/expire_grace_periods                     │
│                                                               │
│  Timer: expiring_entitlements (every 24h, aligned to 2:00 AM) │
│    → POST /api/admin/check_expiring_entitlements              │
│                                                               │
│  Timer: expiring_cards (monthly, aligned to 1st, 3:00 AM)     │
│    → POST /api/admin/check_expiring_cards                     │
│                                                               │
│  Timer: event_reminders (every 1h)                            │
│    → POST /api/admin/send_event_reminders                     │
│                                                               │
│  Timer: token_cleanup (every 24h)                             │
│    → POST /api/admin/cleanup_expired_tokens                   │
│                                                               │
│  Timer: idempotency_cleanup (every 24h)                       │
│    → POST /api/admin/cleanup_idempotency_keys                 │
│                                                               │
│  Timer: photo_cleanup (every 24h)                             │
│    → POST /api/admin/cleanup_scaled_photos                    │
│                                                               │
│  Child process monitor: watchdog process                       │
│    → Check if child is alive, respawn if dead                  │
└──────────────────────────────────────────────────────────────┘
```

All timers are async. The `io_context::run()` drives the entire event loop. No raw threads needed — everything is single-threaded with async I/O, which avoids all the threading complexity that was the reason for not putting timers in the Crow server.

## 3.4 Process Management

**Library**: `Boost.Process` (part of Boost 1.86, already in `conanfile.py`)

Boost.Process provides cross-platform process spawning, monitoring, and termination:

```cpp
#include <boost/process.hpp>
namespace bp = boost::process;

// Spawn web server
bp::child server(server_executable_path, bp::std_out > stdout, bp::std_err > stderr);

// Check if alive
if (server.running()) { ... }

// Kill
server.terminate();

// Wait for exit
server.wait();
```

This works identically on Windows (uses `CreateProcess`/`TerminateProcess`) and Linux (uses `fork`/`exec`/`kill`).

**Note**: Boost.Process v2 (in Boost 1.86) is the current version. It uses `boost::process::v2` namespace and integrates with Boost.Asio for async child monitoring. We should verify which API surface Boost 1.86 exposes and use v2 if available.

## 3.5 HTTP Client for API Calls

The codebase already has `Http::HttpClient` (`util/http/http_client.h`) wrapping libcurl. The helper executable links against `knotty_yoga_core`, giving it access to this HTTP client.

However, since the helper needs cookie-based auth, we need to handle:
- Storing the session cookie from `/api/login` response
- Sending it with subsequent requests

The existing `HttpClient` may or may not support cookie management. If not, we can either:
1. Extend `HttpClient` to support cookies (preferred — benefits tests too)
2. Use libcurl's cookie jar directly in the helper
3. Create a thin `SchedulerHttpClient` wrapper that manages cookies

---

# 4. Project Structure

## 4.1 New Files

```
server/knottyyoga_server/
├── src/
│   ├── scheduler/                          # New directory for helper executable
│   │   ├── CMakeLists.txt                  # Library + executable targets
│   │   ├── main.cpp                        # Entry point, flag parsing
│   │   ├── scheduler.h                     # Primary mode: scheduler + web server watcher
│   │   ├── scheduler.cpp
│   │   ├── watchdog.h                      # Watchdog mode: monitor primary process
│   │   ├── watchdog.cpp
│   │   ├── health_checker.h                # HTTP health check logic
│   │   ├── health_checker.cpp
│   │   ├── process_manager.h               # Cross-platform process spawn/kill/monitor
│   │   ├── process_manager.cpp
│   │   ├── tcp_ping_server.h               # Boost.Asio TCP listener for PONG responses
│   │   ├── tcp_ping_server.cpp
│   │   ├── tcp_ping_client.h               # Boost.Asio TCP client for PING requests
│   │   ├── tcp_ping_client.cpp
│   │   ├── scheduled_job.h                 # Job definition: endpoint, interval, name
│   │   ├── scheduled_job.cpp
│   │   ├── job_scheduler.h                 # Timer-based job execution engine
│   │   ├── job_scheduler.cpp
│   │   ├── api_client.h                    # Authenticated HTTP client for admin endpoints
│   │   └── api_client.cpp
│   │
│   ├── endpoints/
│   │   ├── health.h                        # New: GET /api/health (no auth)
│   │   ├── health.cpp
│   │   ├── admin_send_event_reminders.h    # New: POST /api/admin/send_event_reminders
│   │   ├── admin_send_event_reminders.cpp
│   │   ├── admin_cleanup_tokens.h          # New: POST /api/admin/cleanup_expired_tokens
│   │   ├── admin_cleanup_tokens.cpp
│   │   ├── admin_cleanup_idempotency.h     # New: POST /api/admin/cleanup_idempotency_keys
│   │   ├── admin_cleanup_idempotency.cpp
│   │   ├── admin_cleanup_photos.h          # New: POST /api/admin/cleanup_scaled_photos
│   │   └── admin_cleanup_photos.cpp
│   │
│   ├── business_logic/
│   │   ├── scheduling/
│   │   │   ├── event_reminder_notification.h    # New: send reminder emails
│   │   │   └── event_reminder_notification.cpp
│   │   ├── auth/
│   │   │   ├── token_cleanup.h                  # New: expired token/verification cleanup
│   │   │   └── token_cleanup.cpp
│   │   └── payment/
│   │       ├── idempotency_cleanup.h            # New: expired key cleanup
│   │       └── idempotency_cleanup.cpp
│   │
│   └── business_logic/images/
│       ├── scaled_photo_cleanup.h               # New: scaled photo cache cleanup
│       └── scaled_photo_cleanup.cpp
```

## 4.2 CMake Changes

In the top-level `CMakeLists.txt`, add a new library and executable:

```cmake
add_library(knotty_yoga_scheduler "")
add_subdirectory(src/scheduler)

add_executable(knottyyoga_helper "src/scheduler/main.cpp")
target_link_libraries(knottyyoga_helper
    knotty_yoga_core
    knotty_yoga_scheduler
    ${ABSL_LIB}
    ${Boost_LIB}
    ${CURL_LIB}
)
```

The `knotty_yoga_scheduler` library contains all the helper-specific code (watchdog, ping server/client, process manager, job scheduler, API client). It links against `knotty_yoga_core` for access to `HttpClient` and other utilities.

## 4.3 Build Outputs

After building:
```
build/
├── knottyyoga_the_server       # Web server (existing)
├── knottyyoga_database_helper  # Database setup (existing)
├── knottyyoga_helper           # Scheduler + watchdog (new)
└── certs/
    └── cacert.pem              # SSL certs (existing, also needed by helper)
```

---

# 5. Implementation Phases

## Phase 0: Health Check Endpoint on Web Server
**Backend work first — gives the watchdog something to ping.**

- [ ] Create `GET /api/health` endpoint in web server
  - No authentication required
  - Returns `{"status": "ok", "uptime_seconds": N}`
  - Track server start time in `WebApp` or a global
  - Register in `web_app.cpp` routing
- [ ] Tests for health endpoint

## Phase 1: Executable Skeleton & Process Manager
**Get the basic executable building and able to spawn/monitor processes.**

- [ ] Create `src/scheduler/` directory with `CMakeLists.txt`
- [ ] Add `knotty_yoga_scheduler` library and `knottyyoga_helper` executable to top-level CMake
- [ ] Implement `main.cpp` with `absl::flags` for all command-line options
- [ ] Implement `ProcessManager` class using Boost.Process:
  - `SpawnProcess(executable, args) → ProcessHandle`
  - `IsRunning(ProcessHandle) → bool`
  - `Terminate(ProcessHandle)`
  - `WaitForExit(ProcessHandle) → exit_code`
  - Cross-platform (Windows + Linux)
- [ ] Tests for `ProcessManager` (spawn a trivial process, check alive, terminate)

## Phase 2: TCP Ping Server & Client
**Inter-process health check mechanism.**

- [ ] Implement `TcpPingServer` using Boost.Asio:
  - Listens on configurable port
  - Accepts connections async
  - Reads `PING\n`, responds `PONG\n`, closes
  - Integrates with shared `io_context`
- [ ] Implement `TcpPingClient` using Boost.Asio:
  - Connects to `localhost:{port}` with configurable timeout
  - Sends `PING\n`, waits for `PONG\n`
  - Returns success/failure
- [ ] Tests for both (connect server to client, verify round-trip)

## Phase 3: Web Server Health Checker
**HTTP-based health monitoring of the web server.**

- [ ] Implement `HealthChecker` class:
  - Uses `HttpClient` to call `GET /api/health`
  - Tracks consecutive failures
  - Fires callback when failure threshold exceeded
  - Configurable interval, timeout, failure threshold
- [ ] Integrate with `ProcessManager` for kill/restart on failure
- [ ] Tests for `HealthChecker` (mock `HttpClient`, verify failure counting, verify restart trigger)

## Phase 4: API Client with Authentication
**Authenticated HTTP client for calling admin endpoints.**

- [ ] Implement `ApiClient` class:
  - `Login(email, password)` → calls `POST /api/login`, stores session cookie
  - `CallEndpoint(method, path)` → makes authenticated HTTP call
  - Auto-re-authenticates on `401` response
  - Configurable base URL
- [ ] Verify `HttpClient` supports cookie handling; extend if needed
- [ ] Tests for `ApiClient` (mock HTTP responses, verify auth flow, verify retry on 401)

## Phase 5: Job Scheduler
**Timer-based execution of scheduled jobs.**

- [ ] Define `ScheduledJob` struct: name, endpoint path, HTTP method, interval, last-run timestamp, enabled flag
- [ ] Implement `JobScheduler` class:
  - Owns a set of `ScheduledJob` definitions
  - Uses Boost.Asio timers to fire jobs on their intervals
  - Calls `ApiClient` for each job
  - Logs results (success/failure, response summary)
  - Handles alignment to wall-clock times (e.g., "run at 1:00 AM daily" not "run every 24h from startup")
- [ ] Configure all 8 jobs from Section 1 (4 existing + 4 new endpoints)
- [ ] Tests for `JobScheduler` (mock `ApiClient`, verify timer firing, verify job execution)

## Phase 6: Primary Mode Assembly
**Wire everything together for the primary process.**

- [ ] Implement `Scheduler` class (primary mode orchestrator):
  - Creates `boost::asio::io_context`
  - Initializes `HealthChecker`, `ProcessManager`, `TcpPingServer`, `ApiClient`, `JobScheduler`
  - Spawns web server process (if `--manage_server` is true)
  - Spawns watchdog child process
  - Monitors watchdog child — respawn if it exits
  - Runs `io_context.run()` as the main event loop
- [ ] Wire up in `main.cpp`: if `--watchdog` is false → run `Scheduler`
- [ ] Integration test: start primary, verify it spawns watchdog, verify health checks run

## Phase 7: Watchdog Mode Assembly
**The watchdog process that monitors the primary.**

- [ ] Implement `Watchdog` class:
  - Creates `boost::asio::io_context`
  - Initializes `TcpPingClient` targeting primary's TCP port
  - Periodically pings primary process
  - On failure threshold:
    - Kills primary process (via PID passed as command-line arg or discovered)
    - Promotes self to primary mode (instantiates `Scheduler`)
    - Spawns new watchdog child
- [ ] Wire up in `main.cpp`: if `--watchdog` is true → run `Watchdog`
- [ ] Handle PID passing: primary passes its own PID to watchdog child via `--primary_pid` flag
- [ ] Tests for `Watchdog` (mock TCP client, verify failure detection, verify promotion)

## Phase 8: New Admin Endpoints on Web Server
**Implement the 4 new admin endpoints that the scheduler will call.**

### 8a: Event Reminder Emails
- [ ] Create `EventReminderNotification` in `business_logic/scheduling/`:
  - Query bookings for events starting within `event_reminder_hours` that haven't had a reminder sent
  - Need a `reminder_sent_us` column on `bookings` table (or a separate `booking_notifications` table)
  - Send reminder email with event details (name, date/time, location)
  - Return count of reminders sent
- [ ] Create email template `event_reminder_mail.h/cpp`
- [ ] Create `POST /api/admin/send_event_reminders` endpoint
- [ ] Tests for notification logic and endpoint

### 8b: Expired Token Cleanup
- [ ] Create `TokenCleanup` in `business_logic/auth/`:
  - Delete device tokens where `created_us + max_duration < now`
  - Delete email verifications past expiration
  - Return counts of deleted records
- [ ] Create `POST /api/admin/cleanup_expired_tokens` endpoint
- [ ] Tests for cleanup logic and endpoint

### 8c: Idempotency Key Cleanup
- [ ] Create `IdempotencyCleanup` in `business_logic/payment/`:
  - Delete idempotency keys where `expires_us < now`
  - Return count of deleted records
- [ ] Create `POST /api/admin/cleanup_idempotency_keys` endpoint
- [ ] Tests for cleanup logic and endpoint

### 8d: Scaled Photo Cache Cleanup
- [ ] Create `ScaledPhotoCleanup` in `business_logic/images/`:
  - Delete scaled photo cache entries older than `kScaledPhotoMaxAgeUs`
  - Return count of deleted records
- [ ] Create `POST /api/admin/cleanup_scaled_photos` endpoint
- [ ] Tests for cleanup logic and endpoint

## Phase 9: Service Account Setup
**Create the service account used by the helper for authentication.**

- [ ] Add service account creation to `knottyyoga_database_helper` (or as a migration step):
  - Create a `people` entry for the service account (e.g., `scheduler@knottyyoga.local`)
  - Assign `manage_subscriptions` permission (and any other needed permissions)
  - Mark as verified (skip email verification)
- [ ] Document that the service account password should be set via environment variable, not hardcoded
- [ ] Add a secret key for the service account password so it can be configured in the secrets table

## Phase 10: Logging & Operational Concerns
**Production-readiness.**

- [ ] Structured logging for all operations:
  - Job execution: name, start time, duration, success/failure, response summary
  - Health checks: status, consecutive failures
  - Process events: spawn, terminate, restart, watchdog promotion
- [ ] Log rotation / log file configuration (or use stdout and let the OS service manager handle it)
- [ ] Graceful shutdown: handle SIGTERM/SIGINT (Linux) and service stop events (Windows)
  - Stop timers, close TCP listener, terminate child processes cleanly
- [ ] Consider writing a PID file for the primary process

---

# 6. Library Dependencies

| Library | Already Available? | Used For |
|---------|-------------------|----------|
| Boost.Asio | Yes (via `boost/1.86.0` in conanfile) | Async timers, TCP server/client, io_context event loop |
| Boost.Process | Yes (via `boost/1.86.0` in conanfile) | Cross-platform process spawn, monitor, terminate |
| Boost.Filesystem | Yes (already `find_package(Boost COMPONENTS filesystem)`) | Path handling for executable locations |
| abseil (absl::flags) | Yes (via `abseil/20220623.1` in conanfile) | Command-line flag parsing |
| libcurl (HttpClient) | Yes (via `libcurl/7.86.0` in conanfile) | HTTP calls to web server endpoints |
| Standard library | Yes | `<chrono>`, `<string>`, `<functional>`, `<memory>` |

No new Conan dependencies needed. Everything is available in the existing dependency set.

---

# 7. Configuration Summary

## 7.1 Command-Line Flags (helper executable)

All operational configuration for the helper process. See Section 3.1 for full list.

## 7.2 Secrets (web server database)

These already exist and are used by the admin endpoints. The helper doesn't read these directly — the web server uses them when processing the API calls.

| Secret Key | Default | Used By |
|------------|---------|---------|
| `subscription_grace_period_days` | 7 | `expire_grace_periods` |
| `entitlement_expiry_reminder_days` | 7 | `check_expiring_entitlements` |
| `event_reminder_hours` | 24 | `send_event_reminders` (new) |
| `scaled_photo_reaper_interval_us` | 86400000000 (24h) | Reference for cleanup frequency |
| `scaled_photo_max_age_us` | (not yet set) | `cleanup_scaled_photos` (new) |

## 7.3 New Secret Needed

| Secret Key | Default | Purpose |
|------------|---------|---------|
| `scheduler_service_account_password` | (must be configured) | Password for the service account. The helper reads this from an environment variable or flag; it's also stored hashed in the `people` table. |

---

# 8. Deployment Considerations

## 8.1 Linux (Production — Docker)

The helper runs as a systemd service (or supervisor process) alongside the Docker container:

```ini
[Unit]
Description=Knotty Yoga Scheduler & Watchdog
After=postgresql.service

[Service]
ExecStart=/opt/knottyyoga/knottyyoga_helper \
    --server_executable=/opt/knottyyoga/knottyyoga_the_server \
    --server_url=http://localhost:8080 \
    --service_account_email=scheduler@knottyyoga.local
Environment=SCHEDULER_PASSWORD=<from-secrets-manager>
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

If both the helper AND its watchdog somehow die, systemd's `Restart=always` brings the helper back, which then restarts the web server and spawns a new watchdog.

## 8.2 Windows (Development)

Run from command line or as a Windows Service:

```batch
knottyyoga_helper.exe ^
    --server_executable=knottyyoga_the_server.exe ^
    --server_url=http://localhost:18080 ^
    --service_account_email=scheduler@knottyyoga.local ^
    --service_account_password=%SCHEDULER_PASSWORD%
```

For development, `--manage_server=false` can be used to disable web server management (just run scheduled jobs while you run the web server separately in Visual Studio).

---

# 9. Open Questions

| # | Question | Notes |
|---|----------|-------|
| 1 | **Boost.Process version**: Boost 1.86 ships both v1 (`boost::process`) and v2 (`boost::process::v2`). Which API surface should we target? v2 integrates more cleanly with Boost.Asio for async child monitoring. Need to verify what's available with our Conan package. | Recommend: try v2 first, fall back to v1 if build issues arise. |
| 2 | **Event reminder tracking**: Should we add a `reminder_sent_us` column to the `bookings` table, or create a separate `booking_notifications` table? A column is simpler but only supports one reminder. A table supports multiple reminder types later. | Recommend: `reminder_sent_us` column for now (YAGNI). Add a table later if multiple reminder types are needed. |
| 3 | **Scaled photo cleanup**: What table/mechanism stores scaled photos? Need to verify the actual storage mechanism before implementing the cleanup endpoint. | Need to examine `business_logic/images/` for how scaled photos are stored. |
| 4 | **Wall-clock alignment**: Should daily jobs run at specific wall-clock times (e.g., 1:00 AM) or simply at fixed intervals from startup? Wall-clock alignment is more predictable but more complex to implement (need timezone awareness). | Recommend: wall-clock alignment using the studio's configured timezone (`kDefaultStudioTimezone`). |
| 5 | **Should the helper manage the web server by default?** In some deployments (Docker, Kubernetes), an orchestrator already manages the web server process. The helper should be able to run in "scheduler-only" mode without process management. | Already addressed: `--manage_server=false` flag. |
| 6 | **Service account creation**: Should this be automatic (part of `knottyyoga_database_helper`) or manual (admin creates via portal)? | Recommend: automatic creation in database helper with a well-known email. Password set via secret/environment variable. |
| 7 | **HttpClient cookie support**: Does the existing `HttpClient` wrapper support storing and sending cookies across requests? If not, what's the best extension approach? | Need to examine `util/http/http_client.h/cpp` to determine current cookie handling capability. |

---

# 10. Recommended Implementation Order

The phases above are ordered by dependency, but for practical development, the recommended sequence is:

1. **Phase 0** — Health endpoint (quick win, immediately useful for any monitoring)
2. **Phase 8b, 8c** — Cleanup endpoints (simple, low risk, independently valuable)
3. **Phase 8a** — Event reminders (most user-facing value of the new endpoints)
4. **Phase 8d** — Photo cleanup (needs investigation of photo storage)
5. **Phase 1** — Executable skeleton + process manager
6. **Phase 2** — TCP ping server/client
7. **Phase 4** — API client with authentication
8. **Phase 5** — Job scheduler
9. **Phase 3** — Web server health checker
10. **Phase 6** — Primary mode assembly
11. **Phase 7** — Watchdog mode assembly
12. **Phase 9** — Service account setup
13. **Phase 10** — Logging and operational polish

This order delivers independently useful server endpoints first (which can be tested manually or via curl), then builds the helper executable infrastructure, and finishes with the watchdog self-healing system.