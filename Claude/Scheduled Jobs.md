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

The **service account** is a regular `people` entry with email `scheduler@knottyyoga.local`, the `manage_subscriptions` permission, and `email_verified` set true. It is created automatically by `knottyyoga_database_helper` on first run.

**Password sourcing** — both processes read the same environment variable:
- `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` is set once in `/etc/knottyyoga/server.env` (the existing shared `--env-file`).
- `knottyyoga_database_helper` reads it, hashes it with the same scheme used by normal user signup, and stores the hash in the `people` row. Idempotent: on subsequent runs, if the row already exists, leave it alone (operators can rotate the password by deleting the row and re-running, or via a separate rotation flow).
- `knottyyoga_helper` reads the same env var at startup and uses it to call `POST /api/login`.
- If the env var is missing when `knottyyoga_database_helper` runs, fail loudly with a message instructing the operator to generate a strong password and add it to the env file. Do not auto-generate — the operator wouldn't know the value to give the helper.

**Hiding the service account from admin UIs** — service accounts are identified by their `@knottyyoga.local` email domain rather than a new column on `people`. Admin user-list endpoints filter out rows whose email ends in `@knottyyoga.local`. Login UI does not need a special block since the service account email is internal-only.

This approach:
- Uses the existing auth infrastructure (no new auth mechanism)
- Works over HTTP (the helper doesn't need direct database access for scheduled jobs)
- Service account permissions are managed through the existing RBAC system
- Single env var = single source of truth for the password

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

**Wall-clock alignment uses the host's local timezone.** Daily and monthly timers compute their next firing instant using `std::chrono::system_clock::now()` interpreted in the host's local time (via `std::localtime` / `std::mktime`). On AWS, the EC2 host should be configured to the studio's timezone (set in cloud-init / systemd-timesyncd). No timezone secret is needed — operators control alignment by setting the host TZ. On Windows dev machines, this naturally maps to the developer's local time, which is fine for testing.

No process management, no watchdog, no TCP listener. The helper's only job is to call admin endpoints on schedule. If the helper crashes, systemd restarts its container.

## 3.4 HTTP Client for API Calls

The codebase already has `Http::HttpClient` (`util/http/http_client.h`) wrapping libcurl with a stateless `Execute(HttpRequest) → HttpResponse` interface. The helper executable links against `knotty_yoga_core`, giving it access to this HTTP client.

**Cookie handling lives in `ApiClient`, not in `HttpClient`.** `HttpClient` stays stateless — every request/response carries its own headers map, and `ApiClient` is the only consumer that needs session semantics. The cookie logic in `ApiClient`:

1. After `POST /api/login` succeeds, parse `Set-Cookie` headers from `HttpResponse.headers` and store the name/value pairs in an internal map.
2. On every subsequent `Execute` call, attach a `Cookie:` header built from the stored map to `HttpRequest.headers`.
3. On `401`, clear the cookie map, call `Login` again, and retry the original request once.

The implementation is small (~30 lines, single origin, no full RFC 6265 — no domain matching, expiry, or path semantics needed). This keeps `HttpClient` and its test doubles simple and confines session state to the one component that needs it.

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
│   └── sql_util/table_helpers/
│       ├── scaled_photos.h                      # Existing — add DeleteOlderThan() method
│       └── scaled_photos.cpp
```

(Scaled photo cleanup is a new method on the existing `ScaledPhotosTable` rather than a new business-logic helper, since the work is a single bounded DELETE — no email, no Square, no multi-step orchestration.)

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

Phases are listed in recommended implementation order. Phases 1–4 are new admin endpoints on the existing web server — they can be developed, tested, and merged independently of the helper executable, since the test helper executable can invoke them directly. Phases 5–8 build the helper executable. Phase 9 unblocks end-to-end integration testing of the helper. Phases 10–11 are follow-ups (Phase 10 is blocked on the waitlist feature).

## Phase 0: Health Check Endpoint on Web Server ✅
**Already implemented** in Phase 1.2 of `Deploying to AWS.md`.

- [x] `GET /api/health` — returns `{"status":"ok|fail","db":"ok|fail","version":"<git-sha>"}`
- [x] DB probe via `SELECT 1`, 503 on failure, 200 otherwise
- [x] Tests: green path, DB-failure path, env-var handling, JSON shape, full HTTP integration
- [x] Wired into `web_app.cpp`

## Phase 1: Expired Token Cleanup Endpoint ✅
**Cleanup endpoint — simple, low-risk, independently valuable.**

- [x] Added `DeleteExpired(Transaction&, int64_t asOfUs)` to `TableHelpers::DeviceTokens` and `TableHelpers::EmailVerifications`. Both use `DELETE ... WHERE expires_at < $1 RETURNING id` and return the count of deleted rows. Tests added to both `device_tokens_test.cpp` and `email_verifications_test.cpp` covering: expired vs. fresh row separation, empty database, and the strict-less-than boundary.
- [x] Created `Auth::TokenCleanupHelper` in `business_logic/auth/token_cleanup_helper.{h,cpp}`. `CleanupExpired(Transaction&, int64_t asOfUs = 0)` returns a `TokenCleanupResult { deviceTokensDeleted, emailVerificationsDeleted }`. When `asOfUs == 0`, looks up `now_us()` from the database. Tests in `token_cleanup_helper_test.cpp` cover both-tables-deleted, empty-database, and explicit `asOfUs` cases.
- [x] Created `POST /api/admin/cleanup_expired_tokens` endpoint in `endpoints/admin_cleanup_expired_tokens.{h,cpp}`. Auth: requires login + `manage_subscriptions` permission (consistent with the other scheduler-driven admin endpoints). Returns `{"device_tokens_deleted": N, "email_verifications_deleted": M}`. Endpoint tests cover 401 unauthenticated, 403 missing permission, 200 nothing-to-clean, and 200 deletes-expired-rows-and-leaves-fresh-ones.
- [x] Wired into `web_app.cpp`, `endpoints/CMakeLists.txt`, and `business_logic/auth/CMakeLists.txt`.

## Phase 2: Idempotency Key Cleanup Endpoint
**Cleanup endpoint — simple, low-risk, independently valuable.**

- [ ] Create `IdempotencyCleanup` in `business_logic/payment/`:
  - Delete idempotency keys where `expires_us < now`
  - Return count of deleted records
- [ ] Create `POST /api/admin/cleanup_idempotency_keys` endpoint
- [ ] Tests for cleanup logic and endpoint

## Phase 3: Event Reminder Emails
**Most-requested SHOULD HAVE feature; partially done.**

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

## Phase 4: Scaled Photo Cache Cleanup
**Cleanup endpoint — uses existing `kScaledPhotoMaxAgeUs` secret.**

- [ ] Add a `DeleteOlderThan(int64_t cutoffUs)` method to `ScaledPhotosTable` (`sql_util/table_helpers/scaled_photos.{h,cpp}`):
  - Deletes scaled photo rows whose age exceeds `kScaledPhotoMaxAgeUs`
  - Returns count of deleted records
  - Tests in `scaled_photos_test.cpp` (insert rows with varying ages, run cleanup, verify the right rows are deleted and counts returned)
- [ ] Create `POST /api/admin/cleanup_scaled_photos` endpoint that reads `kScaledPhotoMaxAgeUs` from secrets and calls the table helper. Endpoint test for auth + happy path.
- [ ] No on-disk blob cleanup needed — scaled photos are stored entirely in the `scaled_photos` table.

## Phase 5: Executable Skeleton
**Get the basic helper executable building.**

- [ ] Create `src/scheduler/` directory with `CMakeLists.txt`
- [ ] Add `knotty_yoga_scheduler` library and `knottyyoga_helper` executable to top-level CMake
- [ ] Implement `main.cpp` with `absl::flags` for all command-line options
- [ ] Verify the executable builds, links against `knotty_yoga_core`, and runs with `--help`

## Phase 6: API Client with Authentication
**Authenticated HTTP client for calling admin endpoints.**

- [ ] Implement `ApiClient` class wrapping `Http::HttpClient`:
  - `Login(email, password)` → calls `POST /api/login`, parses `Set-Cookie` headers from the response, stores cookie name/value pairs internally
  - `CallEndpoint(method, path)` → attaches `Cookie:` header from the stored map and makes an authenticated HTTP call
  - On `401`, clears the cookie map, re-calls `Login`, retries the original request once
  - Configurable base URL
- [ ] Tests for `ApiClient` (mock `HttpClient`, verify auth flow, verify cookie attach, verify retry on 401)

## Phase 7: Job Scheduler
**Timer-based execution of scheduled jobs.**

- [ ] Define `ScheduledJob` struct: name, endpoint path, HTTP method, interval, last-run timestamp, enabled flag
- [ ] Implement `JobScheduler` class:
  - Owns a set of `ScheduledJob` definitions
  - Uses Boost.Asio timers to fire jobs on their intervals
  - Calls `ApiClient` for each job
  - Logs results (success/failure, response summary)
  - Handles wall-clock alignment using the host's local timezone (e.g., "run at 1:00 AM daily" via `std::localtime`/`std::mktime`)
- [ ] Configure all jobs from Section 1 (existing + new endpoints)
- [ ] Tests for `JobScheduler` (mock `ApiClient`, verify timer firing, verify job execution)

## Phase 8: Main Loop Assembly
**Wire everything together.**

- [ ] Implement `Scheduler` class (main orchestrator):
  - Creates `boost::asio::io_context`
  - Initializes `ApiClient`, `JobScheduler`
  - Runs `io_context.run()` as the main event loop
  - Handles SIGTERM/SIGINT for graceful shutdown
- [ ] Wire up in `main.cpp`
- [ ] Integration test: start helper, verify it calls endpoints on schedule (depends on Phase 9 service account)

## Phase 9: Service Account Setup
**Create the service account used by the helper for authentication.**

- [ ] Add service account creation to `knottyyoga_database_helper`:
  - Read `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` env var. If missing, fail with a clear message instructing the operator to generate a strong password and add it to `/etc/knottyyoga/server.env`.
  - Create a `people` row for `scheduler@knottyyoga.local` with the hashed password, `email_verified = true`, and a placeholder display name like "Scheduler Service Account".
  - Assign `manage_subscriptions` permission via the existing role-assignment helpers.
  - Idempotent: if the row already exists, skip creation (don't update the password — operators rotate by deleting + re-running).
- [ ] Update admin user-list endpoints to filter out rows whose email ends in `@knottyyoga.local` so service accounts don't appear in the admin user list.
- [ ] `knottyyoga_helper` reads the same env var at startup. Pass via flag `--service_account_password=$SCHEDULER_SERVICE_ACCOUNT_PASSWORD` in the systemd unit, OR have the helper read the env var directly if the flag is unset.
- [ ] Tests:
  - `knottyyoga_database_helper` test: missing env var fails loudly; present env var creates the row idempotently.
  - Admin user-list endpoint test: service-account row is filtered out.

## Phase 10: Waitlist Refund Endpoint
**Hourly job to refund unfulfilled waitlist bookings after the event passes.**

Blocked on the waitlist feature (Phase 10 of `Product, Event, and Subscription Admin Portal.md`). Listed here so it isn't forgotten when waitlist lands.

- [ ] Create business-logic helper that:
  - Finds events where `end_time_us < now_us()` that have waitlisted bookings
  - For each remaining waitlisted booking, processes a full refund via `PaymentHelper`
  - Sets booking status to `cancelled` with reason "Event passed — waitlist refund"
  - Idempotent (re-running does nothing if all waitlisted bookings have already been refunded)
  - Returns count of refunds processed
- [ ] Create `POST /api/admin/process_waitlist_refunds` endpoint
- [ ] Add the job to the helper's scheduler config (hourly)
- [ ] Tests for helper and endpoint

## Phase 11: Logging & Operational Polish
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

## 7.3 New Configuration Needed (env var, not a database secret)

The service account password is **not** stored in the secrets table — it is supplied as an environment variable that both processes read:

| Env Var | Default | Purpose |
|---------|---------|---------|
| `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` | (must be configured) | Password for the `scheduler@knottyyoga.local` service account. `knottyyoga_database_helper` reads this on first run, hashes it, and stores the hash in the `people` row. `knottyyoga_helper` reads the same env var at startup and uses it to call `POST /api/login`. Both containers share `/etc/knottyyoga/server.env` via `--env-file`, so it's set in exactly one place. |

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

# 9. Resolved Questions

All design questions for this plan are now resolved. Decisions are folded into the relevant sections above; this list is a quick reference.

- ~~**Boost.Process version**~~ — No longer needed; the helper doesn't manage processes. AWS infrastructure handles restarts.
- ~~**Event reminder tracking**~~ — `reminder_sent_us` column added to `bookings` table. Endpoint implemented in Phase 3.
- ~~**Should the helper manage the web server?**~~ — No. systemd manages the Docker container. Helper is scheduler-only.
- ~~**Scaled photo cleanup mechanism**~~ — Add a `DeleteOlderThan(cutoffUs)` method to the existing `ScaledPhotosTable` helper, with tests in `scaled_photos_test.cpp`. No on-disk blob cleanup needed — scaled photos live entirely in the `scaled_photos` table. See Phase 4.
- ~~**Wall-clock alignment for daily jobs**~~ — Use the host's local timezone via `std::localtime` / `std::mktime`. On AWS the EC2 host TZ is set via cloud-init; on Windows dev it maps to the developer's local time. No timezone secret. See Section 3.3.
- ~~**Service account creation**~~ — Automatic creation by `knottyyoga_database_helper`. Password sourced from `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` env var (single source of truth, read by both processes via the shared `/etc/knottyyoga/server.env`). Fail loudly if missing — no auto-generation. Service accounts identified by `@knottyyoga.local` email domain (no new column on `people`); admin user-list endpoints filter that domain out. See Section 3.2 and Phase 9.
- ~~**HttpClient cookie support**~~ — Cookie jar logic lives inside the helper's `ApiClient` wrapper, not in `HttpClient`. `HttpClient` stays stateless. Implementation is ~30 lines: parse `Set-Cookie` after login, attach `Cookie:` header on subsequent requests, re-login on 401 and retry once. See Section 3.4.

---

# 10. Phase Dependency Notes

The phases in Section 5 are listed in recommended implementation order. A few cross-cutting notes:

- **Phases 1–4 (admin endpoints) are independent of phases 5–11 (helper executable).** They can be developed, reviewed, and merged in any order — the existing test helper executable can invoke them directly while the scheduler is still being built. This is also the lowest-risk place to start.
- **Phase 8 (Main Loop Assembly) depends on Phase 9 (Service Account Setup) for end-to-end integration testing.** Phases 5–7 can be unit-tested without a real service account by mocking `HttpClient` / `ApiClient`.
- **Phase 10 (Waitlist Refund) is blocked** on the waitlist feature in `Product, Event, and Subscription Admin Portal.md`. Listed last so it isn't forgotten when waitlist lands.
- **Phase 11 (Logging & Polish) layers on top** of phases 5–9 and can be deferred until the helper is otherwise functional.

This is a much simpler plan than the original — no watchdog, no TCP ping, no process management. The helper is just a timer loop that calls HTTP endpoints. AWS infrastructure handles all the process health monitoring.