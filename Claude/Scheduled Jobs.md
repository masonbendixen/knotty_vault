---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/30/2026
Version: 0.2
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

# Scheduled Jobs Helper — Design Plan

## Executive Summary

A single new executable (`knottyyoga_helper`) that serves as a **Scheduled Job Runner** — calling the web server's admin endpoints on configurable intervals to process billing, notifications, cleanup, and reminders.

**Watchdog responsibilities are handled by AWS infrastructure** (see `Deploying to AWS.md`):
- **Process crashes**: systemd `Restart=on-failure` restarts the Docker container within seconds
- **Application hangs**: CloudWatch Synthetics canary hits `/api/health` every 5 minutes, alerts via SNS on failure
- **VM-level failures**: EC2 instance status checks + CloudWatch alarms

The helper runs as a separate Docker container from the same image, with `--entrypoint knottyyoga_helper`. It does NOT manage the web server process — that's systemd's job.

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
| 5 | `/api/admin/process_voucher_expiry` | POST | Daily | 2:30 AM | Sends expiry warning emails for vouchers expiring within configurable window (default 7 days, via `voucher_expiry_reminder_days` secret). Also deactivates already-expired vouchers. |

All five require `manage_subscriptions` permission. All are idempotent — running multiple times in a day causes no harm (though notification endpoints may send duplicate emails).

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

# 2. Health & Monitoring (AWS-native — no custom watchdog)

The original plan called for a custom watchdog-of-watchdog architecture. **This has been replaced by AWS-native primitives** (decided in `Deploying to AWS.md`, Phase 5.3):

| Failure | Handled by | Detection time |
|---|---|---|
| Crow process crash | systemd `Restart=on-failure` on the Docker container | <5s |
| Crow process hung | CloudWatch Synthetics canary → `/api/health` | <10 min |
| Helper process crash | systemd `Restart=on-failure` on its container | <5s |
| EC2 VM failure | EC2 instance status check + CloudWatch alarm | <2 min |

**Health endpoint** (`GET /api/health`): **Already implemented** in Phase 1.2 of the AWS deployment plan. Returns `{"status":"ok|fail","db":"ok|fail","version":"<git-sha>"}` with a `SELECT 1` DB probe. Used by CloudWatch Synthetics, not by the helper.

The helper does NOT monitor the web server. It only runs scheduled jobs.

---

# 3. Executable Design

## 3.1 Command-Line Interface

Using `absl::flags` (same pattern as `knottyyoga_database_helper`):

```
knottyyoga_helper [flags]

Scheduler flags:
  --server_url                Web server base URL for API calls
                              Default: "http://localhost:80"
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
  --waitlist_refunds_interval Default: 3600 (hourly)
```

No watchdog flags — process health is handled by systemd + CloudWatch (see Section 2).

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

The helper runs a single main loop using **Boost.Asio** timers:

```
┌──────────────────────────────────────────────────────────────┐
│  boost::asio::io_context                                      │
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
│  Timer: voucher_expiry (every 24h, aligned to 2:30 AM)        │
│    → POST /api/admin/process_voucher_expiry                   │
│                                                               │
│  Timer: event_reminders (every 1h)                            │
│    → POST /api/admin/send_event_reminders                     │
│                                                               │
│  Timer: waitlist_refunds (every 1h)                           │
│    → POST /api/admin/process_waitlist_refunds                 │
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
└──────────────────────────────────────────────────────────────┘
```

All timers are async. The `io_context::run()` drives the entire event loop. No raw threads needed — everything is single-threaded with async I/O, which avoids all the threading complexity that was the reason for not putting timers in the Crow server.

No process management, no watchdog, no TCP listener. The helper's only job is to call admin endpoints on schedule. If the helper crashes, systemd restarts its container.

## 3.4 HTTP Client for API Calls

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
│   │   ├── scheduler.h                     # Main loop: timer-based job execution
│   │   ├── scheduler.cpp
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

The `knotty_yoga_scheduler` library contains all the helper-specific code (job scheduler, API client). It links against `knotty_yoga_core` for access to `HttpClient` and other utilities.

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

## Phase 0: Health Check Endpoint on Web Server ✅
**Already implemented** in Phase 1.2 of `Deploying to AWS.md`.

- [x] `GET /api/health` — returns `{"status":"ok|fail","db":"ok|fail","version":"<git-sha>"}`
- [x] DB probe via `SELECT 1`, 503 on failure, 200 otherwise
- [x] Tests: green path, DB-failure path, env-var handling, JSON shape, full HTTP integration
- [x] Wired into `web_app.cpp`

## Phase 1: Executable Skeleton
**Get the basic executable building.**

- [ ] Create `src/scheduler/` directory with `CMakeLists.txt`
- [ ] Add `knotty_yoga_scheduler` library and `knottyyoga_helper` executable to top-level CMake
- [ ] Implement `main.cpp` with `absl::flags` for all command-line options
- [ ] Verify the executable builds, links against `knotty_yoga_core`, and runs with `--help`

## Phase 2: API Client with Authentication
**Authenticated HTTP client for calling admin endpoints.**

- [ ] Implement `ApiClient` class:
  - `Login(email, password)` → calls `POST /api/login`, stores session cookie
  - `CallEndpoint(method, path)` → makes authenticated HTTP call
  - Auto-re-authenticates on `401` response
  - Configurable base URL
- [ ] Verify `HttpClient` supports cookie handling; extend if needed
- [ ] Tests for `ApiClient` (mock HTTP responses, verify auth flow, verify retry on 401)

## Phase 3: Job Scheduler
**Timer-based execution of scheduled jobs.**

- [ ] Define `ScheduledJob` struct: name, endpoint path, HTTP method, interval, last-run timestamp, enabled flag
- [ ] Implement `JobScheduler` class:
  - Owns a set of `ScheduledJob` definitions
  - Uses Boost.Asio timers to fire jobs on their intervals
  - Calls `ApiClient` for each job
  - Logs results (success/failure, response summary)
  - Handles alignment to wall-clock times (e.g., "run at 1:00 AM daily" not "run every 24h from startup")
- [ ] Configure all jobs from Section 1 (existing + new endpoints)
- [ ] Tests for `JobScheduler` (mock `ApiClient`, verify timer firing, verify job execution)

## Phase 4: Main Loop Assembly
**Wire everything together.**

- [ ] Implement `Scheduler` class (main orchestrator):
  - Creates `boost::asio::io_context`
  - Initializes `ApiClient`, `JobScheduler`
  - Runs `io_context.run()` as the main event loop
  - Handles SIGTERM/SIGINT for graceful shutdown
- [ ] Wire up in `main.cpp`
- [ ] Integration test: start helper, verify it calls endpoints on schedule

## Phase 8: New Admin Endpoints on Web Server
**Implement the 4 new admin endpoints that the scheduler will call.**

### 8a: Event Reminder Emails
- [ ] Create `EventReminderNotification` in `business_logic/scheduling/`:
  - Query bookings for events starting within `event_reminder_hours` that haven't had a reminder sent
  - Need a `reminder_sent_us` column on `bookings` table (or a separate `booking_notifications` table)
  - Send reminder email with event details (name, date/time, location)
  - Return count of reminders sent
- [x] Create email template `event_reminder_mail.h/cpp` — Blue-themed reminder email with event details
- [x] Create `POST /api/admin/send_event_reminders` endpoint — Finds confirmed bookings within reminder window, sends emails, marks `reminder_sent_us`
- [x] Tests for notification logic (5 tests in `event_reminder_helper_test.cpp`) and email template (2 tests)
- [x] Added `reminder_sent_us` column to `bookings` table
- [x] Added `send_event_reminders` command (alias `ser`) to test helper executable

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

## Phase 8: Logging & Operational Concerns
**Production-readiness.**

- [ ] Structured logging for all operations:
  - Job execution: name, start time, duration, success/failure, response summary
  - Authentication: login success/failure, re-auth events
- [ ] Use stdout logging (same `InitializeLogging()` as the server, Phase 1.3 of AWS deploy). Docker/systemd captures stdout to the journal, CloudWatch Logs agent tails it.
- [ ] Graceful shutdown: handle SIGTERM/SIGINT to stop timers cleanly (Docker sends SIGTERM on `docker stop`)

---

# 6. Library Dependencies

| Library | Already Available? | Used For |
|---------|-------------------|----------|
| Boost.Asio | Yes (via `boost/1.86.0` in conanfile) | Async timers, io_context event loop |
| abseil (absl::flags) | Yes (via `abseil/20220623.1` in conanfile) | Command-line flag parsing |
| libcurl (HttpClient) | Yes (via `libcurl/7.86.0` in conanfile) | HTTP calls to web server endpoints |
| Standard library | Yes | `<chrono>`, `<string>`, `<functional>`, `<memory>`, `<csignal>` |

No new Conan dependencies needed. Boost.Process and Boost.Filesystem are no longer required since the helper doesn't manage processes.

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

## 8.1 Linux (Production — Docker on EC2)

The helper runs as a **separate Docker container** from the same image as the web server, managed by its own systemd unit (see `Deploying to AWS.md` Phase 2.2):

```bash
# The helper container talks to the server container over the Docker network
docker run -d --name knottyyoga-helper \
    --network host \
    --env-file /etc/knottyyoga/server.env \
    --entrypoint knottyyoga_helper \
    knottyyoga:<version> \
    --server_url=http://localhost:80 \
    --service_account_email=scheduler@knottyyoga.local
```

systemd's `Restart=on-failure` handles helper crashes. No watchdog needed — the helper doesn't manage the web server process.

The helper uses `--network host` so it can reach the server container on `localhost:80`. Both containers read their config from `/etc/knottyyoga/server.env`.

## 8.2 Windows (Development)

Run from command line alongside the web server (started separately in Visual Studio):

```batch
knottyyoga_helper.exe ^
    --server_url=http://localhost:18080 ^
    --service_account_email=scheduler@knottyyoga.local ^
    --service_account_password=%SCHEDULER_PASSWORD%
```

---

# 9. Open Questions

## 9.1 Already Resolved

- ~~**Boost.Process version**~~ — No longer needed; the helper doesn't manage processes. AWS infrastructure handles restarts.
- ~~**Event reminder tracking**~~ — `reminder_sent_us` column added to `bookings` table. Endpoint implemented in Phase 8a.
- ~~**Should the helper manage the web server?**~~ — No. systemd manages the Docker container. Helper is scheduler-only.

## 9.2 Open — Please Answer

1. **Scaled photo cleanup table/mechanism.** Scaled photos are stored in the `scaled_photos` table (helpers at `sql_util/table_helpers/scaled_photos.{h,cpp}`, schema in `db_schema/photos.{h,cpp}`). Cleanup would mean deleting rows older than `kScaledPhotoMaxAgeUs`. Should the cleanup business logic live in `business_logic/images/scaled_photo_cleanup.{h,cpp}` and delegate row deletion to a new method on the existing `ScaledPhotosTable` helper, or should it issue the delete query directly via `DbCrud`? Any other state to clean up beyond the table rows (e.g., on-disk blobs)?

   **Answer:** Mason- Let's add it to ScaledPhotosTable helper and make sure to add tests.

2. **Wall-clock alignment for daily jobs.** Should daily jobs run at specific wall-clock times (e.g., billing at 1:00 AM, grace at 1:30 AM, etc., as listed in Section 1.1) using the studio's configured timezone, or simply at fixed intervals from startup? If wall-clock, where does the timezone come from — a new secret, an existing one, or the host's local time?

   **Answer:** Mason- Let's go with host's local time zone.

3. **Service account creation — automatic vs. manual.** Should the `scheduler@knottyyoga.local` service account be created automatically by `knottyyoga_database_helper` (with `manage_subscriptions` permission, marked verified, password sourced from a secret/env var), or created manually as a one-time provisioning step? If automatic, should the password default to a generated value on first run, or require operator-supplied input?

   **Answer:** Mason- What do you suggest?

   **Claude's recommendation:** Automatic creation in `knottyyoga_database_helper`, password sourced from a single environment variable (e.g., `SCHEDULER_SERVICE_ACCOUNT_PASSWORD`) that is read by **both** processes:
   - `knottyyoga_database_helper` reads the env var, hashes it, and stores the hash in the `people` row for `scheduler@knottyyoga.local` (creating the row + permission grant on first run; idempotent on subsequent runs).
   - `knottyyoga_helper` reads the same env var at startup and uses it to call `POST /api/login`.

   Both containers already share `/etc/knottyyoga/server.env` via `--env-file` (see Section 8.1), so there's exactly one place to set the password. If the env var is missing when `knottyyoga_database_helper` runs, fail loudly with an instruction to generate a strong password and add it to the env file — don't auto-generate, because then the operator wouldn't know the value to give the helper.

   Follow-up question: should the service account be flagged in the `people` table somehow (e.g., a `is_service_account bool` column) so it's hidden from the admin user list and can't be logged into via the normal UI? Or is `scheduler@knottyyoga.local` distinguishable enough by its email domain?

   **Answer:**

4. **HttpClient cookie support.** The existing `HttpClient` interface (`util/http/http_client.h`) is a single `Execute(HttpRequest) → HttpResponse` call — `HttpRequest` has a `headers` map and `HttpResponse` exposes `headers`, but there is no built-in cookie jar. Three options for how the helper handles auth cookies:
   - **(a)** Extend `HttpClient` (or add a `CookieJar` member) so cookies are persisted across `Execute` calls automatically. Benefits other consumers and tests.
   - **(b)** Add cookie-jar logic only inside the helper's `ApiClient` wrapper — parse `Set-Cookie` headers from the login response, attach `Cookie:` header on subsequent requests. Keeps `HttpClient` minimal.
   - **(c)** Skip session cookies entirely — give admin endpoints an alternative auth mechanism (e.g., a static service token header) that the helper can use without login.

   Which option do you prefer?

   **Answer:** What do you think? I'm leaning towards "a" since it seems to be the easiest to use but I could be convinced otherwise.

   **Claude's recommendation:** I'd push back gently and recommend **(b) — cookie logic inside `ApiClient`**. Reasoning:

   - **`HttpClient` is intentionally stateless today.** The interface is one virtual call: `Execute(HttpRequest) → HttpResponse`. Adding a cookie jar makes it stateful, which means: every test double has to model jar behavior, every consumer has to think about whether two calls share state, and the libcurl impl has to either keep a persistent `CURL*` or replay cookies on every call. That's a meaningful API change for one consumer.
   - **The cookie logic is tiny.** All the helper needs is: parse `Set-Cookie` headers from the login response, store name/value pairs, attach a `Cookie:` header on subsequent requests. ~30 lines inside `ApiClient`. You don't need full RFC 6265 (domain matching, expiry, paths) because the helper only ever talks to one origin with a session cookie.
   - **YAGNI.** The helper is the only client that needs cookies right now. If a second tool ever needs the same behavior, it can either share an `ApiClient` or pull the cookie logic out into a small `CookieJar` helper at that point — and we'll know more about real usage.
   - **Tests stay simple.** The existing `TestHttpClient` doesn't need a cookie story. `ApiClient` tests can mock the underlying `HttpClient` and verify Cookie header behavior at one level up.

   The "ease of use" win for (a) is real, but it's outweighed by giving up the simple stateless `HttpClient` contract. Option (c) I'd avoid — adding a parallel non-cookie auth path to admin endpoints expands the auth surface area for one caller's convenience.

   Want me to go with (b), or did I miss something that tips it toward (a)?

   **Answer:**

---

# 10. Recommended Implementation Order

The phases above are ordered by dependency, but for practical development, the recommended sequence is:

1. **Phase 0** — ✅ Health endpoint (done — Phase 1.2 of AWS deploy)
2. **Phase 5b, 5c** — Cleanup endpoints (simple, low risk, independently valuable)
3. **Phase 5a** — Event reminders (partially done — endpoint exists, needs integration)
4. **Phase 5d** — Photo cleanup (needs investigation of photo storage)
5. **Phase 1** — Executable skeleton (CMake, main.cpp, flag parsing)
6. **Phase 2** — API client with authentication
7. **Phase 3** — Job scheduler (timer-based execution engine)
8. **Phase 4** — Main loop assembly (wire it all together)
9. **Phase 6** — Service account setup
10. **Phase 7** — Waitlist refund endpoint
11. **Phase 8** — Logging and operational polish

This is a much simpler plan than the original — no watchdog, no TCP ping, no process management. The helper is just a timer loop that calls HTTP endpoints. AWS infrastructure handles all the process health monitoring.