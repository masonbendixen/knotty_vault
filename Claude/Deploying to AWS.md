---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/24/2026
Version: 0.2
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I'm getting ready to start deploying to AWS. I will initially deploy with the Square sandbox to let a few people try it out and get used to the flow. I'd like to figure out what will be involved to deploy to AWS. The C++ server really has no state itself. I also need to run the scheduled jobs process and have the test helper running so that I can log in through SSH and do various operations. I also need to deploy the database helper to set the initial state of the database. I also need a hosted postgres database.

I need to point DNS to the server, enable SSH. What other things do I need to be aware of? What are the costs going to be like? Which AWS hosting options are the best fit for me?

I also figure that once I have deployed, I need a plan for updating the server going forward. I figure when I deploy versions, I should probably save branches in GIT. I also might want to save snapshot copies of the db_schema folder for different versions and create update utilities to migrate / evolve the database schema. If I need to change a database table, is it better to give it a new table name? What are industry standards for this? I also use gitlab for version control. It supports creating a CI/CD pipeline but my tests on the server rely on a postgres database. Can I add that to a CI/CD pipeline on Gitlab?

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Architecture — Committed

**Decision (2026-04-24)**: EC2 + RDS + S3 + CloudFront, x86-64, no nginx. Moving to ARM and committing to Reserved Instances / Savings Plans happens after the soft launch stabilizes.

You have prior AWS experience (S3, RDS, Lambda, EC2 at Tableau), so the write-up below trims the hand-holding where you don't need it. Where something is project-specific (e.g., "CloudFront has to be in us-east-1 for the ACM cert"), it's spelled out; where it's generic AWS, it's terse.

```
             ┌──────────────────┐
 users ───►  │   CloudFront     │  (TLS via free ACM cert in us-east-1)
             │   distribution   │
             └──────┬───────┬───┘
                    │       │
      /* (default)  │       │  /api/*   (CachingDisabled, AllViewer)
                    ▼       ▼
           ┌──────────┐   ┌──────────────────────────────────┐
           │    S3    │   │  EC2 t3.small (x86, Ubuntu 24.04)│
           │ Angular  │   │  knottyyoga_the_server :80       │
           │  bundle  │   │  systemd, no TLS, no nginx       │
           │  (OAC)   │   │  CloudFrontOriginGuard middleware│
           └──────────┘   └──────────┬───────────────────────┘
                                     │ TLS (RDS CA)
                                     ▼
                           ┌──────────────────────┐
                           │  RDS db.t3.micro     │
                           │  Postgres 15         │
                           │  single-AZ, PITR on  │
                           └──────────────────────┘
```

## Committed choices

- **Compute**: EC2 `t3.small` (x86-64). Crow binds `0.0.0.0:80` directly. The server gets `cap_net_bind_service` so it can bind 80 without running as root. Migrate to `t4g.small` (ARM Graviton) later for ~20% cost savings — GitLab CI will need an ARM runner or a cross-build step at that point, so it's not happening on day one.
- **Origin protection**: `CloudFrontOriginGuard` middleware in Crow. CloudFront adds `X-Origin-Secret: <random>` on every forwarded request; the middleware rejects anything else with 403. No SG-by-IP-prefix bookkeeping. Detailed in Phase 1.7.
- **Database**: RDS `db.t3.micro` Postgres 15, single-AZ, 20 GB gp3, automated backups with 7-day PITR (free), deletion protection on. Connection uses `sslmode=verify-full` with the RDS CA bundle.
- **Frontend**: S3 bucket + CloudFront distribution. CloudFront terminates TLS via an ACM cert in us-east-1, serves the Angular bundle from S3 by OAC, and reverse-proxies `/api/*` to the EC2 Elastic IP.
- **DNS**: Route 53 hosted zone; apex + `www` alias records → CloudFront distribution.
- **Email**: Amazon SES.
- **Square**: sandbox initially; flip `kSquareEnvironment` secret to `production` later.
- **Scheduled jobs**: `knottyyoga_helper` is **complete** (see `Scheduled Jobs.md` — all 11 phases done) and ships with the initial deploy. Runs under its own systemd unit on the same EC2, in a separate container from the server, sharing `/etc/knottyyoga/server.env` via `--env-file`. Handles billing, reminders, voucher expiry, cleanup jobs, and waitlist refunds. Authenticates as the `scheduler@knottyyoga.local` service account which `knottyyoga_database_helper --migrate` provisions during initial deploy.
- **Admin / ops access**: SSH to the EC2 for running `knottyyoga_test_helper` ad-hoc.

## Why no nginx

Everything nginx would normally do is already handled by CloudFront or Crow:

| Classic nginx role | Replaced by |
|---|---|
| TLS termination | CloudFront + ACM |
| HTTP → HTTPS redirect | CloudFront viewer protocol policy |
| Static file serving | S3 via CloudFront OAC |
| Reverse proxy | CloudFront `/api/*` behavior → EC2 origin |
| gzip / compression | CloudFront auto-compression |
| Access logging | CloudFront logs to S3 + CloudWatch |
| Rate limiting | CloudFront request throttling + AWS WAF |
| Origin-secret check | Crow middleware (Phase 1.7) |

The only role nginx would retain is multiplexing if we ever served non-HTTP from the EC2 (WebSockets on a different port, a second process, etc.). We don't.

## Gotchas to remember during setup

These bite first-time CloudFront deployments — none are dealbreakers, but each is "oops, 90 minutes" if forgotten:

1. **ACM cert must be in us-east-1** (not your app region). CloudFront is a global service that only reads certs from that region.
2. **SPA routing**: CloudFront "Custom Error Response" must map 403 and 404 from the S3 origin to `/index.html` with status 200. Otherwise deep-linked Angular routes break on refresh.
3. **Cookies through CloudFront**: `/api/*` behavior needs Cache Policy `CachingDisabled` + Origin Request Policy `AllViewer`. Mis-configure once and sessions leak across users.
4. **Cache-bust `index.html` on every frontend deploy**. Angular's content-hashed chunks auto-bust, but `index.html` is not hashed. `aws cloudfront create-invalidation --paths /index.html` is the fix.
5. **Origin protection via custom header, not SG-IP-list**. AWS's CloudFront IP prefix list changes and requires periodic SG updates; the custom-header approach is stable.
6. **RDS `verify-full` requires the RDS CA bundle** at `/etc/knottyyoga/rds-ca.pem` on the EC2. Download during provisioning, not on first failed connection.

## Reserved Instances / Savings Plans — when to commit

AWS calls most long-term commitments "Reserved Instances" (per-service) or the newer "Compute Savings Plans" (across EC2/Fargate/Lambda). Note that AWS **"Dedicated Instances"** and **"Dedicated Hosts"** are something different — those are single-tenant hardware for compliance and *increase* your cost. What you want for cheaper billing is either a Reserved Instance or a Savings Plan.

- **Recommendation**: run on-demand for the first 2–4 weeks to confirm `t3.small` is right-sized. Then buy a 1-yr no-upfront Compute Savings Plan at whatever the average hourly burn has settled to. No-upfront preserves cash flow; 1-yr gives ~30% off; Savings Plans apply to any EC2 family, so you can migrate to ARM later without losing the discount.
- RDS has its own Reserved Instance mechanism (no Savings Plan equivalent yet). Same timing — wait until the instance type is confirmed before buying.

## Critical Code Gaps That Block Deploy (Summary)

These come first — they're the Phase 1 work. Each is detailed in its phase section below.

1. **DB connection is hardcoded** in `sql_util/database_access/database_helper_init.cpp` (user=docker, password=docker, host=postgresql). This **must** be driven by env vars before we can point at RDS.
2. **Secret bootstrap**: secrets live in the `config_secrets` table, but database credentials themselves can't live there (chicken-and-egg). DB credentials + a few startup-only flags are env vars; everything else stays DB-backed.
3. **Frontend `environment.prod.ts`** is a stub — missing Square Application ID and Location ID.
4. **No health endpoint** (needed for the CloudWatch Synthetics canary, CloudFront health checks, and manual smoke tests).
5. **No CloudFront origin-secret middleware** — Phase 1.7 adds it.
6. **No migration mechanism** — `database_helper` destructively rebuilds the DB, which is fine for dev but will wipe customer data in prod. Must add a forward-only, versioned migration path before the second deploy.
7. **No production build pipeline** — we'll ship native x86-64 Linux binaries from GitLab CI.
8. **No `.gitlab-ci.yml`** — CI with postgres service is feasible in GitLab and we'll wire that up.

---

# Phase 1 — Code & Config Prerequisites (Lowest Layer First)

Goal: make the application configurable per environment and observable enough to run unattended on an EC2 instance fronted by CloudFront. These changes should land before any AWS work.

## 1.1 Parameterize database connection via environment variables

Touches the lowest layer (database access). Everything above depends on the DB, so this is first.

- [x] Update `server/knottyyoga_server/src/sql_util/database_access/database_helper_init.cpp` to read from env vars with sensible fallbacks to current dev defaults:
  - `KNOTTYYOGA_DB_HOST` (fallback: current platform-dependent value)
  - `KNOTTYYOGA_DB_PORT` (fallback: `5432`)
  - `KNOTTYYOGA_DB_USER` (fallback: `docker`)
  - `KNOTTYYOGA_DB_PASSWORD` (fallback: `docker`)
  - `KNOTTYYOGA_DB_NAME` (fallback: `kDatabaseName`)
  - `KNOTTYYOGA_DB_SSLMODE` (fallback: `prefer`; set to `require` in prod)
  - `KNOTTYYOGA_DB_SSLROOTCERT` (fallback: empty; set to `/etc/knottyyoga/rds-ca.pem` for `verify-full` against RDS)
- [x] Update the connection string builder to include `sslmode=<mode>` and `sslrootcert=<path>` when those fields are non-empty.
- [x] Add a unit test `database_helper_init_test.cpp` that:
  - Sets env vars via `setenv` / `_putenv_s` and asserts both the parsed fields and the connection string reflect them.
  - Clears env vars and asserts the platform-specific defaults.
  - Verifies sslmode/sslrootcert are appended only when set.
- [x] Log (at `LogInfo`) the host/port/db name (NOT the password) at startup so misconfig is obvious in logs (`DatabaseHelperInit::LogStartupInfo()`, called from the no-arg `MakeProductionDatabaseHelper()`).

**Note on RDS & `sslmode`**: RDS PostgreSQL requires either `require` or `verify-full` for production-grade TLS. `verify-full` needs the AWS RDS CA bundle installed in the image. Start with `require` (encrypt, don't verify CN). Good enough for v1.

## 1.2 Add a health-check endpoint

Used by: CloudWatch Synthetics canary (Phase 5.3), any future load balancer, manual smoke tests. **Not** consumed by `knottyyoga_helper` — the helper-as-watchdog idea was dropped in favor of AWS-native primitives (see `Scheduled Jobs.md` §2).

- [x] Add `endpoints/health.cpp` / `health.h` with a `GET /api/health` handler returning `{"status":"ok|fail","db":"ok|fail","version":"<git-sha>"}`.
  - Runs a trivial `SELECT 1` inside a transaction (`ProbeDatabase`) to validate DB connectivity.
  - Returns 503 if the DB probe throws or the provider is null; 200 otherwise.
- [x] Build version comes from env var `KNOTTYYOGA_VERSION` at request time (`GetBuildVersion()`); falls back to `"unknown"` when unset/empty.
- [x] Add `health_test.cpp` — green path, DB-failure path, env-var handling, JSON shape, full HTTP integration. Uses an in-test `ThrowingTransactionProvider` to drive the failure path without taking down a real DB.
- [x] Wire into `endpoints/CMakeLists.txt` (both header and cpp + test) and into `web_app.cpp` (include + `g_Health` reference) so MSVC keeps the routing translation unit alive.

## 1.3 Logging to stdout for systemd / CloudWatch

- [x] Existing `util/logging.cpp` was hardcoded to `std::cout`. Replaced with a `KNOTTYYOGA_LOG_DEST`-driven config: `stdout` (default), `stderr`, or a file path. `LogXxx()` now returns a stream pointed at the resolved destination; the file-path branch falls back to stdout (with a warning on stderr) if the file can't be opened.
- [x] Linux line-buffering confirmed: `InitializeLogging()` calls `setvbuf(file, _IOLBF, ...)` on the chosen stream, so each LogInfo() "...\n" lands in the systemd journal / CloudWatch Logs agent immediately rather than waiting for a 4 KB pipe buffer to flush. (Documented inline that MSVC treats `_IOLBF` as full-buffered, which is fine for Windows dev.)
- [x] `InitializeLogging()` wired into `main.cpp` and `database_helper/main.cpp` as the first call in each.
- [x] **Crow's built-in logger bridged into the same destination.** Crow ships with its own `CROW_LOG_INFO`/`ERROR`/etc. macros that route through `crow::ILogHandler`; the default `CerrLogHandler` writes to `std::cerr` regardless of our config. `InitializeLogging()` now installs a `KnottyyogaCrowLogHandler` (a process-lifetime static) that delegates to `*g_logStream` with the same `(timestamp) [LEVEL] message` format Crow's CerrLogHandler emits. Without this bridge, an operator picking `KNOTTYYOGA_LOG_DEST=/var/log/app.log` would see Knotty Yoga logs in the file but Crow's request-handling logs still hitting stderr — two streams to correlate. (`error_response.cpp` is the one current call site of `CROW_LOG_ERROR`; future Crow logging follows automatically.)
- [x] Added `logging_test.cpp` covering `ResolveLogDestination` (null / empty / "stdout" / "stderr" / absolute path / relative path / Windows-style path / case-sensitivity edge) plus `CrowLogLevelLabel` (every level returns the 8-char fixed-width prefix, out-of-range produces "UNKNOWN ").

**Advice**: systemd captures stdout/stderr automatically into the journal — no need for a custom log file path in the container/EC2 deploy. Simpler is better.

## 1.4 Frontend environment configuration

- [x] `environment.prod.ts` populated with Square **sandbox** Application ID (`sandbox-sq0idb-B1PoAtwzV7eEmN3u8FHLyQ`) and Location ID (`NWLEQ37Z06H6JEC`) from `Square credentials and Sandbox setup.md`. `production: true`, sandbox script URL. This is the soft-launch build's environment.
- [x] `environment.development.ts` updated — replaced the `LXXXX` Location ID placeholder with the real sandbox Location ID so `ng serve` actually tokenizes against Square sandbox.
- [x] Created `environment.prod-square-live.ts` for the eventual live flip — placeholder Application ID / Location ID, production script URL, with a top-of-file comment listing the four-step procedure to flip live (fill IDs → update angular.json → flip backend `kSquareEnvironment`/`kSquareAccessToken` → smoke test). NOT wired into angular.json so an accidental production build can't ship live-card creds.
- [x] `environment.ts` (the imported file) annotated with a comment explaining it's always file-replaced; placeholder values kept as a deliberately-broken fallback so an unconfigured `ng build` fails loud rather than ships placeholders.
- [x] `angular.json` `production` build configuration now file-replaces `environment.ts` → `environment.prod.ts`. `ng build --configuration=production` (the default) produces the soft-launch bundle.
- [x] `ServerAccessNetwork.ts` audited — every HTTP call uses a relative `/api/...` URL. Same-origin behind CloudFront works as-is; no `baseUrl` plumbing needed.

## 1.5 Cookies + CORS sanity pass for CloudFront same-origin deploy

Currently `ServerConfig::Initialize` reads `kWebsiteAddress` from DB secrets and configures CORS when `prodMode_` is on. With CloudFront serving both the Angular bundle (from S3) and `/api/*` (from EC2) under one distribution domain, the browser sees a single origin → CORS preflight never triggers → cookies flow with plain `SameSite=Lax`.

- [x] **Same-origin verified.** CloudFront fronts both `/*` (S3) and `/api/*` (EC2) under one host (`knottyyoga.com`). Browser sees same-origin → no CORS preflight → cookies flow with `SameSite=Lax`. The existing CORS middleware in `ServerConfig::Initialize` keys off `kWebsiteAddress`; in production it's effectively a no-op because preflights never fire from same-origin. (Direct hits to the EC2 IP would trigger CORS, but Phase 1.7's `CloudFrontOriginGuard` middleware will 403 those before they reach any handler.)
- [x] **Auth code audit**: existing cookie code in `business_logic/auth/session.cpp:213-237` already does the right thing for same-origin — `SameSite=Lax`, `httpOnly=true`, and in prod mode adds `Secure=true` + `Domain=<kWebsiteAddress>`. No code currently assumes a cross-origin frontend; no `SameSite=None` or hardcoded scheme appears outside test fixtures.
- [x] **Test added** — `SessionTest.InitializeFromLoginProdModeCookieHasSecureAndDomain` in `session_test.cpp` calls the full `ServerConfig::Initialize` path (via `EndpointTestHelper`'s WebApp) with `kServerProductionMode=true` + `kWebsiteAddress=knottyyoga.com`, then exercises `Session::InitializeFromLogin` and asserts the cookie carries `Secure`, `Domain=knottyyoga.com`, `SameSite=Lax`, `HttpOnly`. Locks the same-origin contract so a future cross-origin migration must be deliberate. (Phase 1.6 covers the proxy-trust side — making sure `X-Forwarded-Proto` is honored when running HTTP-only on EC2 behind CloudFront.)

### First-boot secrets to set on the EC2 (Phase 4.8 procedure references this)

These are the values that **must** be overridden before booting the server in production. Items marked "default OK" can ride the `secret_values.cpp` fallback. Items marked "must override" have wrong-for-prod defaults or empty defaults.

| Secret key | Value for soft launch | Why override |
|---|---|---|
| `production_mode_on` | `true` | Defaults to `false`; needed to enable Secure cookies + CORS |
| `website_address` | `knottyyoga.com` | Release default is `http://www.knottyyoga.com/`; we want the bare apex (cookies use this for the `Domain` attribute) |
| `square_access_token` | sandbox token from Square Developer Console | Release default is empty |
| `square_environment` | `sandbox` | Release default is `production`; we're on the sandbox during soft launch |
| `mail_server_name` | `email-smtp.us-west-2.amazonaws.com` | Default is `smtp.gmail.com` |
| `mail_server_port` | `465` (TLS wrapper) | Default is already `465`. **Corrected 9/18:** this row said `587` while the next said `login` — in `mail_helper.cpp` `login` is implicit TLS (port 465) and `tls` is STARTTLS (port 587); the pair 587/`login` fails the handshake. Either `465`+`login` (no change) or `587`+`tls` — never mixed. |
| `mail_server_method` | `login` | Default OK (already `login`) |
| `mail_smtp_username` | SES SMTP **username** (the `AKIA…` string) | **New key, 9/18.** Empty by default = "log in as the sender address" (Gmail). SES's username is not an address, and before this key the helper had no way to send one — see 4.7. |
| `mail_app_password` | SES SMTP password (created in IAM, NOT your console password) | **Default is now EMPTY** (honuware Phase 9.2 — see the rotation note below); in dev it is seeded from `HONUWARE_MAIL_APP_PASSWORD` |
| `Knotty Yoga and Spa` (sender name) | (use default) | Default OK |
| `knottyyogaandspa@gmail.com` (sender address) | `noreply@knottyyoga.com` (or whatever `kMailSenderAddress` is set to) | Defaults to the Gmail address; SES requires the From address match a verified domain identity |

The full list of secrets and their defaults lives in `src/util/secrets/secret_values.cpp`. Phase 4.8 (Secret bootstrap ordering) describes the operator workflow: provision DB → run `database_helper --migrate` to populate the `config_secrets` table from defaults → run `database_helper --seed-secrets-from-file secrets.json` (or `knottyyoga_test_helper`) to override the values above → start the server.

#### AT DEPLOY: rotate the Gmail app password and give knottyyoga its own

**Knotty Yoga and CommunityFinder currently share one Gmail mailbox and therefore one app password.** Both seed `knottyyogaandspa@gmail.com` as `kMailSenderAddress`, and as of 2026-09-10 both read the same `HONUWARE_MAIL_APP_PASSWORD` to seed `config_secrets.mail_app_password`. This was accepted deliberately as a temporary state so both apps can send mail during development — it is **not** the intended end state.

Background, because it explains why the shared credential exists at all: honuware's `secret_values.cpp` used to ship a real Gmail app password as a *framework default*, committed to the public `server_components` repo. Every app inherited it silently — knottyyoga had no mail-password seeding of its own. That literal was removed and the credential rotated in honuware Phase 9.2, and knottyyoga gained its own env-var seeding at the same time (`create_database.cpp` → `PopulateConfigSecrets`).

- [ ] **Mint a knottyyoga-specific app password** at https://myaccount.google.com/apppasswords, named identifiably (e.g. `knottyyoga-smtp-prod`). Separate credentials are independently revocable — a leak on one app does not force rotation on the other, which is the whole reason app passwords are cheap and disposable.
- [ ] **Store it in a password manager**, not a text file. A plaintext file under `Documents` is the pattern being retired here; note that `C:\Users\mason\Documents` and `C:\Users\mason\OneDrive\Documents` are *different* folders and only the second syncs to Microsoft's cloud — a distinction too subtle to be filing credentials against.
- [ ] **Rotate the currently shared credential** once both apps have their own, so the shared one stops being valid anywhere.
- [ ] **Decide the sender identity per app.** If knottyyoga moves to SES with `noreply@knottyyoga.com` (rows above), its Gmail app password becomes dev-only and the production credential is an SES SMTP password from IAM. CommunityFinder then keeps the Gmail mailbox and needs its own sender address regardless. Remember the coupling: **mailio authenticates using the sender address as the SMTP username**, so the password must belong to whatever `kMailSenderAddress` resolves to — a mismatch surfaces as "Mail sender rejection", which reads like an address problem rather than a credential one.
- [ ] Update the `mail_app_password` row above once the production value is an SES password rather than a Gmail one.

## 1.6 Reverse-proxy awareness in the C++ server

CloudFront forwards the viewer's scheme in `X-Forwarded-Proto: https`, but the TCP connection to Crow is plain HTTP on port 80. Code that infers scheme from the request itself would see `http` behind CloudFront — so any future caller that needs to know the *viewer*'s scheme/IP must consult the forwarded headers.

- [x] **Audit confirmed the cookie path is scheme-agnostic.** `session.cpp:213-237` (the only place that sets `Secure` on a cookie) keys off `ServerConfig::IsProdMode()`, not the request scheme. So `Secure=true` is emitted whenever the operator has set `production_mode_on=true`, regardless of whether Crow saw the request as HTTP. The viewer receives the response over HTTPS via CloudFront and accepts the `Secure` cookie correctly. **No cookie code change needed for the CloudFront deploy.**
- [x] Searched the codebase for any `req.is_secure()`, `req.scheme()`, `is_https`, etc. — none exist. No code path currently makes a wrong decision based on the EC2-leg's HTTP scheme.
- [x] Added `business_logic/auth/proxy_trust.{h,cpp}`:
  - `Auth::ProxyTrustEnabled()` — reads `KNOTTYYOGA_TRUST_PROXY` env var. True for `"1"` / `"true"` (case-insensitive); false for unset / empty / `"0"` / `"false"` / garbage.
  - `Auth::ResolveViewerScheme(req)` — when the proxy is trusted, returns the trimmed `X-Forwarded-Proto` value (e.g., `"https"`); otherwise empty string.
  - `Auth::ResolveViewerIp(req)` — when the proxy is trusted, returns the first IP from `X-Forwarded-For` (the original viewer; the rest of the comma-separated list is the proxy chain and is dropped); otherwise empty string.
  - The header itself documents *why* these helpers exist with no immediate consumer (cookie code already does the right thing) — they're available for future request-logging, abuse-detection by IP, HSTS preload checks, etc., and shipping the primitive now means the header-parsing logic + opt-in env var are tested before we need them.
- [x] **Defense-in-depth**: helpers default to "not trusted" so an operator who forgets to set `KNOTTYYOGA_TRUST_PROXY=1` on the EC2 just gets empty strings, not spoofed viewer IPs. Phase 1.7's `CloudFrontOriginGuard` middleware will additionally 403 any direct-EC2 request that bypasses CloudFront, so even when the env var IS set, attackers can't spoof headers because they can't reach the origin.
- [x] Tests in `proxy_trust_test.cpp` (16 cases, no fixtures, RAII `ProxyTrustEnvScope` to scrub env between tests):
  - `ProxyTrustEnabled` — unset / empty / `"1"` / `"true"` (lowercase) / `"True"` (mixed) / `"0"` / `"false"` / garbage.
  - `ResolveViewerScheme` — not-trusted-but-header-present returns empty / trusted-with-`https` / trusted-with-`http` / trusted-but-header-missing returns empty / whitespace trimming.
  - `ResolveViewerIp` — not-trusted returns empty / trusted single IP / trusted comma list returns first IP only / whitespace trimming / header missing / header empty.

## 1.7 Origin-secret middleware (replaces nginx)

Since we're dropping nginx, Crow needs to enforce the CloudFront-origin secret itself. This is what stops attackers from hitting the EC2 Elastic IP directly and bypassing the CDN/WAF/cache.

- [x] Added `endpoints/cloudfront_origin_guard.{h,cpp}`:
  - `Endpoints::CloudFrontOriginGuard` is a Crow middleware (`struct context`, `before_handle`, `after_handle`). Reads `KNOTTYYOGA_ORIGIN_SECRET` once in its constructor and caches the expected value.
  - `before_handle` flow: (1) guard disabled (env var unset/empty) → pass through; (2) URL starts with `/api/health` → pass through (allow-listed for Synthetics + watchdog probes); (3) `X-Origin-Secret` header matches expected → pass through; (4) otherwise: `res.code = 403` + `Content-Type: application/json` + body `{"error":"direct_origin_access_forbidden"}` + `res.end()` to short-circuit the handler.
  - `after_handle` is intentionally a no-op.
- [x] Wired into `endpoints/web_app.h` `AppType`: `crow::App<Endpoints::CloudFrontOriginGuard, crow::CookieParser, crow::CORSHandler>`. Existing endpoint tests work unchanged because the env var is unset in tests so the guard auto-disables.
- [x] Startup logging: on construction the guard emits one `LogInfo()` line — either "CloudFrontOriginGuard active: requests must carry X-Origin-Secret (allow-listed: /api/health*)" or "CloudFrontOriginGuard disabled: KNOTTYYOGA_ORIGIN_SECRET not set." Operators see immediately on first boot whether the guard armed.
- [x] Rejection logging is rate-limited to **once per minute per process** via a steady_clock-throttled `LogWarning()`, so a port scanner or misconfigured monitor can't drown the systemd journal in 403 messages. Throttled message identifies the missing-vs-mismatched case so operators have a useful first signal.
- [x] Tests in `cloudfront_origin_guard_test.cpp` (14 cases, no fixtures, RAII `OriginSecretEnvScope` for env hygiene):
  - Activation: unset / empty → inactive; non-empty → active.
  - Disabled guard passes every request through.
  - Active guard, secret-protected path: rejects no-header / wrong-header / empty-header (all → 403 with right body + `Content-Type: application/json` + `is_completed`); accepts correct-header (pass-through, `is_completed` false).
  - Health allow-list: `/api/health` and `/api/health/db` pass through without header; `/api/login` and `/` are rejected; `/api/healthz` is documented as currently allowed (canary test that pins the simple-prefix-match decision so a future tightening is deliberate).
  - `after_handle` is a no-op (preserves response body + code).
- [x] **Operator wiring** (Phase 4.6): set `KNOTTYYOGA_ORIGIN_SECRET=<random>` in `/etc/knottyyoga/server.env` and the matching `X-Origin-Secret` value as a CloudFront "Origin custom header" on the `/api/*` behavior. ✅ both halves done (4.4 env file, 4.6 origin header, 9/17). Rotation is written up in `RUNBOOK.md` §5 (CloudFront first, then env file + restart, ~30s of 403s; overlap window skipped for v1).

---

# Phase 2 — Build & Packaging

Goal: produce deployable artifacts repeatably via Docker containers, run under systemd on EC2.

## 2.1 Decision: Docker containers (decided 2026-04-30)

**Decision**: containerize. A single multi-stage Dockerfile produces one image containing all binaries. Reasons for switching from the original native-binary recommendation:

1. **System library headaches disappear at deploy time.** The GSSAPI/krb5 link-ordering battle during the Linux build proved the point: the runtime image has the exact libraries the binaries were built against. No `apt install` on the target EC2, no RPATH patching, no missing `.so` surprises.
2. **ECS migration later is near-zero work.** Push the image to ECR, create a task definition, done.
3. **The build container already exists** (`server/docker_project/`). The multi-stage Dockerfile extends it with a slim runtime stage.
4. **SSH + test_helper is barely harder.** `docker exec -it knottyyoga-server knottyyoga_test_helper` instead of running the binary directly.

### Container architecture

One image, multiple entrypoints. On the EC2, each process runs as a separate container from the same image:

```
knottyyoga:<version>
├── /opt/knottyyoga/bin/knottyyoga_the_server      (default entrypoint)
├── /opt/knottyyoga/bin/knottyyoga_database_helper
├── /opt/knottyyoga/bin/knottyyoga_test_helper
├── /opt/knottyyoga/bin/knottyyoga_helper
└── /opt/knottyyoga/certs/cacert.pem
```

- **Server container**: `docker run -d --name knottyyoga-server -p 80:80 --env-file /etc/knottyyoga/server.env knottyyoga:<version>`
- **Helper container** (scheduled jobs): same image, different entrypoint and `--network host` so it can hit the server on `localhost:80`: `docker run -d --name knottyyoga-helper --network host --env-file /etc/knottyyoga/server.env --entrypoint knottyyoga_helper knottyyoga:<version> --server_url=http://localhost:80 --service_account_email=scheduler@knottyyoga.local`
- **DB migration** (one-shot at deploy): `docker run --rm --env-file ... --entrypoint knottyyoga_database_helper knottyyoga:<version> --migrate` (`--entrypoint` is required — ENTRYPOINT is the server). Reads `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` from the env file to provision the scheduler service-account row (fails fast if unset).
- **Test helper** (ad-hoc via SSH): `docker exec -it knottyyoga-server knottyyoga_test_helper`

- [x] Wrote `server/knottyyoga_server/package/Dockerfile` — multi-stage build:
  - **Builder stage** (`gcc:14.2.0`): installs cmake, conan 2.x, patchelf, libkrb5-dev, then runs `build_linux_release.sh` to compile and stage all binaries.
  - **Runtime stage** (`ubuntu:22.04`): copies only `bin/`, `lib/`, `certs/`, `VERSION` from the builder. Installs minimal runtime deps (`libgssapi-krb5-2`, `libstdc++6`, `ca-certificates`). Default entrypoint is `knottyyoga_the_server`; override with `--entrypoint` for other binaries.
  - Build: `docker build -t knottyyoga:<ver> --build-arg KNOTTYYOGA_VERSION=<ver> -f package/Dockerfile .`
  - Image size: ~100-150 MB (vs ~2 GB builder stage).
- [x] Wrote `server/knottyyoga_server/package/build_linux_release.sh`. Runs `conan install`, `cmake -DCMAKE_BUILD_TYPE=Release`, `cmake --build`, then assembles a staging tree:
  - `bin/knottyyoga_the_server`, `bin/knottyyoga_database_helper`, `bin/knottyyoga_test_helper`, `bin/knottyyoga_helper` (all required; build fails fast if missing).
  - All bin files are stripped (`strip --strip-unneeded`) to keep the tarball small.
  - `lib/` populated by walking each binary's `ldd` output, filtering OS-provided libs (anything under `/lib`, `/usr/lib`, `/lib64`, `/usr/lib64`), and copying every other shared object. `patchelf --set-rpath '$ORIGIN/../lib'` rewrites each binary's RPATH so the bundled libs resolve without `LD_LIBRARY_PATH`. Bundled libs themselves get `$ORIGIN` so inter-lib deps stay inside `lib/`.
  - `certs/cacert.pem` copied from the source tree (libcurl trust store).
- [x] Tarball: `dist/knottyyoga-<version>.tar.gz` with the layout `bin/`, `lib/`, `certs/`, `systemd/` (placeholder for Phase 2.2), `migrations/` (placeholder for Phase 3), plus `VERSION` and `MANIFEST.txt` files at the root. Tar uses a top-level `knottyyoga-<version>/` prefix so untar'ing produces a single directory.
- [x] Version resolution: `KNOTTYYOGA_VERSION` env var if set; else git short-sha (with `-dirty` suffix when the worktree has uncommitted changes); else `dev-YYYYMMDDHHMMSS`. Same value goes into the tarball name and the `VERSION` file, and is what `KNOTTYYOGA_VERSION` should be set to on the EC2 so `/api/health` reports the matching build string.
- [x] Tool checks at the top of the script (`require_tool conan|cmake|patchelf|ldd|tar|g++`) — fail fast with a hint to `apt install` / `pip install` if anything's missing.
- [x] Configuration knobs via env vars: `BUILD_DIR`, `OUT_DIR`, `STAGE_DIR`, `JOBS` (defaults to `nproc`). Self-locating via `${BASH_SOURCE[0]}` so the script can be invoked from any cwd.
- [x] Sidesteps the recipe's `vs_layout` quirk on Linux by passing `--output-folder` to conan and an explicit `-DCMAKE_TOOLCHAIN_FILE` to cmake.
- [x] Companion `package/README.md` with quick-start instructions, env-var reference, troubleshooting tips, and a list of what's in the tarball.
- [x] Target OS/arch: **Ubuntu 22.04 LTS on x86-64**. Migrate to ARM64 (Graviton, ~20% cheaper) post-launch when the CI builder has an ARM runner or cross-build set up.

## 2.2 systemd units (Docker-based)

- [x] `knottyyoga-server.service` written at `server/knottyyoga_server/package/systemd/knottyyoga-server.service`. `Type=simple` foreground `docker run --rm`, `Restart=on-failure`, `RestartSec=5s`, `TimeoutStopSec=30s`. `ExecStartPre=-/usr/bin/docker rm -f knottyyoga-server` defends against zombie containers from a hard crash. Image tag pinned via `EnvironmentFile=/etc/knottyyoga/version.env` (`${KNOTTYYOGA_IMAGE_TAG}`).
- [x] `knottyyoga-helper.service` written at `server/knottyyoga_server/package/systemd/knottyyoga-helper.service`. Same `Type=simple` pattern. `--network host` so the helper hits the server on `localhost:80`. `--entrypoint knottyyoga_helper` plus `--server_url` and `--service_account_email` flags; `--service_account_password` intentionally omitted so the helper falls back to `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` from `server.env`. `After=knottyyoga-server.service` + `Wants=` (not `Requires=`) keeps the helper running across server restarts. `RestartSec=10s` gives the server breathing room after a restart so the helper's first login doesn't immediately fail. SIGTERM-clean per Phase 11 of `Scheduled Jobs.md`.
- [x] **Version pinning via `EnvironmentFile=/etc/knottyyoga/version.env`** (single-line `KNOTTYYOGA_IMAGE_TAG=vX.Y.Z`) instead of `sed`'ing the unit files in-place. Deploy script atomically rewrites that file and runs `systemctl restart` — no `daemon-reload` needed since the unit files themselves don't change. `version.env.example` ships in the tarball; install-time copy + edit.
- [x] **Do not** create a unit for `knottyyoga_test_helper` — it stays manual via SSH: `docker exec -it knottyyoga-server knottyyoga_test_helper`.
- [x] Unit files bundled into the tarball at `systemd/` (build script copies from `package/systemd/`; build fails fast if any of the four expected files — both `.service` files, `version.env.example`, `README.md` — is missing).
- [x] `package/systemd/README.md` documents the first-time install procedure, the update procedure, why each directive was chosen, and the common failure modes (most importantly: helper login failure when `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` changes after the initial `--migrate`).
- [x] Log lines validating env var wiring (matches 1.1 / 1.3). Docker captures stdout/stderr automatically; systemd journals it. The structured log format from Phase 11 of `Scheduled Jobs.md` (`[scheduler] event=…` / `[api_client] event=…`) is greppable in `journalctl`.

## 2.3 Frontend artifact

- [x] `ui/package/build_ui_release.sh` produces a self-contained tarball of the SPA. Mirrors the server's `build_linux_release.sh` conventions (same env-var names — `KNOTTYYOGA_VERSION`, `OUT_DIR`, etc. — same `[knottyyoga-ui-build] event=...` log-prefix shape, same git-sha-with-`-dirty`-suffix version fallback) so one CI pipeline can drive both halves of a deploy with one version string.
- [x] Build flow: `npm ci --no-audit --no-fund` (NOT `npm install` — `ci` refuses to start if the lockfile is out of sync, catching drift at CI time instead of papering over it); `npx ng build --configuration=production --output-path=<BUILD_DIR>`; auto-detect the servable directory (`dist/browser/` for Angular 17+ application-builder, `dist/` for the older browser-builder) so an Angular CLI upgrade can move the layout without breaking the script silently; stage everything at the tarball root (NOT under a `browser/` sub-prefix) so operators point Nginx's `root` at `/opt/knottyyoga/ui/` and `index.html` is right there.
- [x] Tarball layout: `knottyyoga-ui-<version>/{index.html, *.js, *.css, assets/, ..., VERSION, MANIFEST.txt}`. Single top-level prefix dir (matches the server tarball pattern) so untarring anywhere produces one named directory. The script's last step is a sanity check that `index.html` actually landed at the staged root — refuses to ship without it (catches `angular.json` drift before the live site 404s).
- [x] Tool requirements: bash, node ≥ 18, npm, tar. Script auto-installs `tar` when running as root on apt-based systems (the cheap one-liner case); refuses to auto-install Node because the distro packages are usually too old for Angular 19's engines field and a NodeSource install is the right call anyway. The Node-major-version check up front prints a clear error instead of letting the build die with a cryptic Webpack message 90 seconds in.
- [x] `SKIP_NPM_CI=1` escape hatch documented for local iteration; the README explicitly forbids CI from setting it.
- [x] `ui/package/README.md` mirrors `server/.../package/README.md`: quick-start, what's in the tarball, versioning rules, layout-detection rationale, and a sketch of the deploy-side extract-and-flip procedure (which is owned by Phase 5, not 2.3, but worth noting so a future reader knows where the producer hand-off ends).
- [x] Deploy-side extraction script (atomic `ln -sfn` flip into `/opt/knottyyoga/ui`) is intentionally NOT in this phase — it lives with the host setup in Phase 5. The producer (this script) and consumer (Phase 5's deploy script) are split so the producer can run in a CI image that has Node but no shell access to the EC2 host.

---

# Phase 3 — Database Migration Strategy

Goal: never lose customer data between versions. Stop using destructive rebuild in production.

## 3.1 Decision: to rename or alter?

You asked whether to give changed tables new names. **Industry standard answer**: no, not for most changes.

- **Compatible changes** (add column, add nullable column, add index, widen a type): plain `ALTER TABLE` is correct. No rename.
- **Breaking changes** (drop a column still read by the old code, change semantics of a column): use the **Expand / Migrate / Contract** pattern:
  1. *Expand*: add the new column/table alongside the old. Deploy code that writes to both and reads the old one.
  2. *Migrate*: backfill data from old to new.
  3. *Flip reads*: deploy code that reads the new column/table.
  4. *Contract*: drop the old column/table in a later release.
- **New table names** are only for genuinely new concepts or when two data models must coexist (e.g., a rewrite). Renaming tables to signal a schema change is an anti-pattern: breaks tooling, breaks queries in BI tools, forces client downtime.

What you already have that's unusual: the C++ code *is* the schema source of truth (`db_schema/`). That's fine, but we need the code to evolve additively and to have a record of what has already been applied to any given database.

## 3.2 Introduce a `schema_migrations` version table ✅

- [x] Added `schema_migrations` table at `db_schema/schema_migrations.{h,cpp}`. Columns:
  - `id` TEXT primary key (e.g. `"0001_baseline"`).
  - `applied_at_us` BIGINT NOT NULL DEFAULT `now_us()` — microseconds since epoch, matching every other timestamp column in the schema (the plan loosely said TIMESTAMPTZ; consistency with `admin_alerts.created_at`, `bookings.cancelled_us`, etc. won).
- [x] Registered the new table in `make_database_info.cpp` (created on every fresh DB build) and in `create_database.cpp` `CreateTables()` as the **first** table created, before anything else. Added to `db_schema/CMakeLists.txt`.
- [x] **Table helper** at `sql_util/table_helpers/schema_migrations.{h,cpp}` — single owner of all schema_migrations CRUD, per the layering rule:
  - `IsApplied(transaction, id)` → bool, via `DbCrud::LookupRowByValue`.
  - `ListAppliedIds(transaction)` → `StringArray`, custom SQL with multi-column ORDER BY (`applied_at_us` ASC, `id` ASC as deterministic tiebreak; one of the documented DbCrud-can't-express cases, so direct SQL stays inside the table helper).
  - `RecordApplied(transaction, id)` → inserts `(id, now_us())` via `DbCrud::AddRowToTable`. Duplicate ids throw on the underlying PK violation; gating is the caller's job.
  - **7 unit tests** in `schema_migrations_test.cpp`: empty-table reads, record-then-IsApplied, duplicate-throws, ListAppliedIds empty + ordered, multi-column ORDER BY tiebreak (forces `applied_at_us` equal across rows and asserts `id`-ascending order), IsApplied distinguishes recorded from not-recorded.
- [x] **Business-logic runner** at `business_logic/migration/migration_runner.{h,cpp}` — **no SQL in this layer**, pure orchestration that delegates every schema_migrations read/write to the table helper:
  - `IsApplied(transaction, id)` / `ListApplied(transaction)` — pass-through to the helper.
  - `ApplyOne(transaction, migration)` — if `helper.IsApplied(id)` skip; else `migration.apply(transaction)` then `helper.RecordApplied(id)`. Returns true/false.
  - `ApplyPending(transactionProvider, migrations)` — applies every unapplied migration **each in its own transaction via the supplied provider**, so a mid-list failure leaves earlier migrations committed and skips later ones. On failure throws `MigrationFailure { migrationId(), what() }`.
  - Structured logging: `[migration] event=applied|skipped|apply_failed id=…`.
  - **No bootstrap method.** The table is created by the normal `MakeDatabaseInfo` + `CreateTables` flow during initial database setup. Pre-deploy there is no prior production state to defend against, so a `CREATE TABLE IF NOT EXISTS` fallback would be dead code.
- [x] **11 unit tests** in `business_logic/migration/migration_runner_test.cpp`:
  - `IsApplied`: false for unknown id / empty id; true after the table helper records the id.
  - `ListApplied`: empty on fresh DB; returns ids in apply order.
  - `ApplyOne`: invokes callback + records id; **skips already-applied without calling the callback** (verified via invocation log); **does NOT record id when apply throws**; apply callback sees the same transaction (verified by creating a TEMP TABLE inside apply and reading from it afterward).
  - `ApplyPending`: empty list → empty result; applies all in order on fresh DB; skips already-applied and applies remainder (mixed result); stops at failing migration (invocation-log proof that the migration after the failing one was never attempted); wraps non-`std::exception` throws in `MigrationFailure`; returns all-skipped result when nothing is new.
- [x] Wired into `business_logic/CMakeLists.txt` (`add_subdirectory(migration)`), new `business_logic/migration/CMakeLists.txt`, and `sql_util/table_helpers/CMakeLists.txt` (new helper + test).
- [x] **Architecture fix during implementation.** Initial pass put CRUD SQL directly in `MigrationRunner` (raw `CREATE TABLE IF NOT EXISTS`, `SELECT COUNT(*)`, `SELECT id ORDER BY…`, plus a direct `DbCrud::AddRowToTable`) and added a `CREATE TABLE IF NOT EXISTS` bootstrap for "legacy hosts that predate this commit." Both violations of the project's layering rules — corrected on review by introducing the `TableHelpers::SchemaMigrations` helper and removing the speculative bootstrap. Lesson captured in `feedback_no_sql_in_business_logic.md` and `feedback_no_premature_defensive_code.md`.

## 3.3 Split `knottyyoga_database_helper` into two modes ✅

`knottyyoga_database_helper` is now split into two explicit, mutually-exclusive modes via flags. Both default to `false` so accidental invocation does nothing — the operator has to opt in.

- [x] **`--recreate_database`** preserved for dev/test, blocked in prod by `KNOTTYYOGA_ALLOW_DESTRUCTIVE`. Guard lives at `util/destructive_guard.{h,cpp}` (`IsDestructiveAllowed()` / `EnsureDestructiveAllowed()`):
  - Strict equality: only the literal string `"1"` authorizes. `"0"`, unset, `"true"`, `"yes"`, `"TRUE"`, `"01"`, `" 1"`, etc. all block — anything that looks like a typo fails closed.
  - Error message names the env var and the required value so operators know what to fix without grep'ing the source.
  - 9 unit tests in `destructive_guard_test.cpp` covering each case (unset / "0" / empty / non-one strings / exactly-"1") for both `IsDestructiveAllowed` and `EnsureDestructiveAllowed`, plus a test that asserts the error message mentions the env-var name and `"1"`. Uses an RAII `DestructiveEnvScope` guard so individual tests don't leak env state.
- [x] **`--migrate`** added. Calls `Migration::RunMigrateCommand` (the thin orchestration wrapper from below) with the project's migration list. Exit code is forwarded to the OS so `install.sh` can fail-fast on a bad migration.
- [x] **`business_logic/migration/migrate_command.{h,cpp}`** — `RunMigrateCommand(transactionProvider, databaseHelper, migrations) → int`. Pure orchestration on top of `MigrationRunner::ApplyPending`:
  - Returns 0 on success (zero or more migrations applied/skipped cleanly).
  - Returns 1 on `MigrationFailure` (a migration's apply() threw) — the per-migration failure was already logged by `ApplyPending`; this layer adds a single `[migrate] event=failure id=…` summary line for the operator.
  - Returns 1 on any other `std::exception` escape (defensive).
  - Takes the migration list as a parameter so tests can pass arbitrary fixtures without coupling to the project's current `BuildAllMigrations()`.
  - 6 unit tests in `migrate_command_test.cpp`: empty-list-returns-zero, applies pending in order, idempotent across runs, returns-one-on-migration-failure, stops-at-failing-migration (3-migration list where #2 throws and #3 must NOT run — verified via invocation log), mixed applied-and-skipped returns zero.
- [x] **`business_logic/migration/all_migrations.{h,cpp}`** — `BuildAllMigrations()` returns the project's canonical migration list. **Currently empty** per the no-premature-defensive-code rule: the fresh-install schema is built by `CreateAndPopulateDatabases` (the `--recreate_database` path), so until we have a real inter-version schema change to apply against a database with customer data, the list stays empty. The header documents the "when you add a migration" checklist.
  - 5 unit tests in `all_migrations_test.cpp`: empty-pre-first-deploy (the prompt to remove this assertion is the first time the list grows), all ids unique, all ids non-empty, all ids in lexicographic order (assumes the zero-padded numeric-prefix convention), all migrations have an `apply` callback.
- [x] **No baseline migration.** The spec originally called for a "0001_baseline" migration that re-runs the existing schema-creation code path. With `--recreate_database` as the canonical fresh-install path and no production state to defend against, a baseline migration would be speculative duplicate code. When we need it (e.g., to support `--migrate` directly against a truly empty DB in a future workflow), we'll add it then with knowledge of the actual schema-version-at-rest.
- [x] **`main.cpp`** rewritten as a flag dispatcher: validates exactly-one-of (`--recreate_database` xor `--migrate`), prints a help message and exits 1 if neither or both are set, otherwise delegates to `RunRecreate()` or `RunMigrate()`. Each path emits structured `[database_helper] event=…_starting/_done` log lines for the journal.
- [x] Wired into `util/CMakeLists.txt` and the existing `business_logic/migration/CMakeLists.txt`. No new CMake subdirs needed.

## 3.4 Snapshotting schema per release

You asked about saving copies of `db_schema/`. My take: **don't copy the directory**. Git tags per release (e.g., `v2026.04.16`) achieve the same goal without duplicated files and without drift.

- [ ] Adopt a release tag convention: `vYYYY.MM.DD` or `vMAJOR.MINOR.PATCH`. Recommendation: semver with prereleases (`v1.0.0-sandbox.1`).
- [ ] Tag every deployed build in git; the tag is the snapshot. Migrations that ship with that tag are the ones applied up to that point.
- [ ] The deployment script records the deployed tag in the DB (a `deployments` audit table — simple: id, version, deployed_at, notes). Useful for debugging "which build is broken?".

## 3.5 Rollback strategy

- [ ] Rolling back a code-only release: redeploy previous tarball, restart systemd unit. Near-zero downtime.
- [ ] Rolling back a code + schema release: redeploy previous binaries but **do not** roll back the migration. Old code must be forward-compatible with the new schema (which is why Expand/Migrate/Contract matters).
- [ ] Disaster recovery: restore from an RDS snapshot or point-in-time. Write this procedure down in a `RUNBOOK.md` in this repo once Phase 4 is complete. *Written 9/17 as `RUNBOOK.md` §6 (restore to a NEW instance, repoint `server.env`, verify, then retire the old one) — marked **unverified** until the Phase 5.1 PITR drill runs it for real; the box stays open until then.*

---

# Phase 4 — AWS Infrastructure

Goal: provision the accounts/services we'll actually deploy to.

## 4.1 Account bootstrap

- [x] Create AWS account (or use existing). ✅ 2026-05-13
	- Added knottyyoga account bound to knottyyogaandspa@gmail.com
- [x] Enable MFA on root. Never log in as root after bootstrap. ✅ 2026-05-13
	- Used Google Authenticator
- [ ] Create an IAM admin user for yourself; create `AWSCLI` access keys stored in a password manager.
	- Created an account masonbendixen with a password and created the group Administrators with the AdministratorAccess policy
	- The login URL is:
		- https://957014951609.signin.aws.amazon.com/console
	- Turned on MFA (note that the first one is root and the second is user)
	- IAM accounts don't have access to billing by default even as an admin
		- https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started-account-iam.html
	- **Still to do — create CLI access keys for `masonbendixen`:**
		1. Sign in to the AWS console as the `masonbendixen` IAM user (use the login URL above — select "IAM user", account ID `957014951609`).
		2. Top search bar → **IAM** → IAM console → left sidebar → **Users** → click `masonbendixen`.
		3. **Security credentials** tab → scroll to **Access keys** → **Create access key**.
		4. Use case: **Command Line Interface (CLI)**. Acknowledge the recommendation banner (best-practice is Identity Center, but for a single-operator account a long-lived access key is fine). **Next**.
		5. Optional description tag: `aws-cli local dev`. **Create access key**.
		6. Copy both the **Access key ID** and **Secret access key** (or click **Download .csv file**). The secret is shown **only once** — if you lose it you must delete the key and create a new one.
		7. Save both to your password manager, then add to `~/.aws/credentials`:
			```
			[knottyyoga]
			aws_access_key_id = AKIA...
			aws_secret_access_key = ...
			```
			And `~/.aws/config`:
			```
			[profile knottyyoga]
			region = us-west-2
			output = json
			```
		8. Verify: `aws --profile knottyyoga sts get-caller-identity` → should print your user ARN ending in `:user/masonbendixen`.
- [x] Set a **billing alarm** at $75/mo (sanity) so a misconfigured anything doesn't quietly run up a bill. ✅ 2026-05-13
	- As root: Billing → Preferences → enabled "Receive CloudWatch billing alerts". Billing metrics only live in `us-east-1` and can take hours to first appear.
	- As root: enabled "IAM User and Role Access to Billing Information" so the admin IAM user can see billing data.
	- Created SNS topic `billing-alerts` (Standard) with an email subscription; confirmed via the subscription confirmation email.
	- In `us-east-1` CloudWatch → Alarms → All alarms → Create alarm: metric **Billing → Total Estimated Charge (USD)**, static threshold **> 75**, notification action = SNS topic `billing-alerts`, alarm name `MonthlyBilling75`.
	- Optional follow-up: add lower early-warning alarms at $10 / $25 / $50.
	- Caveats: alarms can lag by several hours, and they only notify — they do **not** stop resources. Use the IAM admin (with MFA) for day-to-day; reserve root for billing, account recovery, and rare admin tasks.
- [x] Region: `us-west-2` (Oregon) for everything except the ACM cert. The ACM cert lives in `us-east-1` (CloudFront-global limitation) — you'll create that explicitly in Phase 4.5. ✅ 2026-05-13
- [x] In the AWS console region picker, default to `us-west-2`. When you switch over to ACM in Phase 4.5, remember to flip the region picker to `us-east-1` for that step only. ✅ 2026-05-13

## 4.2 Networking

The default VPC plus two security groups is all we need. The default VPC already has subnets in every us-west-2 AZ with route tables pointing at an internet gateway — no provisioning required, just verification.

- [x] **Confirm the default VPC and pick two subnets for the RDS subnet group.** ✅ 2026-05-14
	- AWS console region picker: **us-west-2 (Oregon)**.
	- Top search bar → type **VPC** → click **VPC**.
	- Left sidebar → **Your VPCs** → confirm one row with `Default VPC = Yes`. Note its VPC ID (e.g., `vpc-0abc…`).
		- vpc-0059b262559e0779a
	- Left sidebar → **Subnets** → filter by that VPC ID (top filter box). You should see four subnets — one per AZ (`us-west-2a/b/c/d`). Pick any two AZs (e.g., `us-west-2a` and `us-west-2b`); you'll point the RDS subnet group at these in Phase 4.4.
		- subnet-072002670dde5d5f0
		- subnet-08c9d7ce4caad5c78
		- subnet-0a4544e5444e7fdcf
		- subnet-0c50cfd5c793c5f1b
	- Sanity check: click each chosen subnet → **Route table** tab → there should be a route `0.0.0.0/0 → igw-…` (this is what makes it a *public* subnet).
- [x] **Create security group `knottyyoga-web` (for EC2).** ✅ 2026-05-14
	- VPC console → left sidebar → **Security groups** → **Create security group**.
	- **Name:** `knottyyoga-web` (AWS rejects names that begin with `sg-` — that prefix is reserved for the auto-generated SG ID)
	- **Description:** `Knotty Yoga web tier (EC2)`
	- **VPC:** the default VPC
	- **Inbound rules → Add rule** twice:
		1. Type: `SSH` (port 22); Source: **My IP** (the dropdown auto-fills your current public IP as `/32`). If you're on a dynamic ISP IP this will need updating later — Phase 5.2 covers that.
		2. Type: `HTTP` (port 80); Source: `Anywhere-IPv4` (`0.0.0.0/0`). Origin protection is enforced in the Crow middleware via `X-Origin-Secret`, not in the SG.
	- **Outbound rules:** leave the default `All traffic → 0.0.0.0/0`.
	- **Create security group**. Note the new SG ID.
		- sg-0accf95c33945db08
- [x] **Create security group `knottyyoga-db` (for RDS).** ✅ 2026-05-14
	- VPC console → **Security groups** → **Create security group**.
	- **Name:** `knottyyoga-db`
	- **Description:** `Knotty Yoga DB tier (RDS)`
	- **VPC:** the default VPC
	- **Inbound rules → Add rule** once:
		- Type: `PostgreSQL` (port 5432); Source: **Custom** → start typing `knottyyoga` and pick `knottyyoga-web` from the autocomplete. This is the key bit — only the web tier can talk to the DB.
	- **Outbound rules:** leave default.
	- **Create security group**.
- [x] **Verify.** Security Groups list should show both new SGs bound to the default VPC. Note both IDs — you'll select `knottyyoga-web` in the EC2 wizard (Phase 4.3) and `knottyyoga-db` in the RDS wizard (Phase 4.4). ✅ 2026-05-14

## 4.3 Compute: EC2

- [x] **Create the SSH key pair you'll use to log in.** ✅ 2026-05-14
	- Region: **us-west-2**.
	- Top search → **EC2** → EC2 console → left sidebar → **Network & Security → Key Pairs** → **Create key pair**.
	- **Name:** `knottyyoga-ec2`
	- **Key pair type:** `ED25519` (smaller, modern)
	- **Private key file format:** `.pem` (Linux/macOS/OpenSSH on Windows) or `.ppk` (Windows + PuTTY)
	- **Create key pair** — the browser downloads the private key. Move it somewhere safe (e.g., `~/.ssh/knottyyoga-ec2.pem`); AWS does **not** keep a copy.
		- C:\Users\mason\.ssh
		- Google drive / Knotty Yoga / Website / ssh
	- Lock the file: `chmod 400 ~/.ssh/knottyyoga-ec2.pem` (Linux/macOS); on Windows, right-click the file → Properties → Security → Advanced → Disable inheritance → grant only your user Read access.
- [x] **Launch the EC2 instance.** ✅ 2026-05-14
	- EC2 console → left sidebar → **Instances** → **Launch instances**.
	- **Name:** `knottyyoga-server`
	- **Application and OS Images (AMI):** click **Ubuntu** in the quick-start grid → confirm `Ubuntu Server 24.04 LTS (HVM), SSD Volume Type` → architecture **64-bit (x86)** (not ARM). (22.04 is no longer offered as a plain image in the us-west-2 quick-start grid; the surviving 22.04 AMIs are SQL Server bundles. 24.04 LTS is supported through April 2029 and Docker abstracts the host kernel from the `ubuntu:22.04` runtime container, so this swap is safe.)
	- **Instance type:** `t3.small`
	- **Key pair (login):** `knottyyoga-ec2`
	- **Network settings → Edit:**
		- VPC: default
		- Subnet: one of the two AZs you picked in 4.2 (e.g., `us-west-2a`)
		- Auto-assign public IP: **Enable** (we'll attach an Elastic IP next, but first-boot needs network either way)
		- Firewall (security groups): **Select existing security group** → check `knottyyoga-web`. Uncheck any launch-wizard default SG.
	- **Configure storage:** 1× `20 GiB`, volume type **gp3**. Leave Encryption ON (default).
	- **Launch instance**.
	- Wait ~30 seconds; refresh Instances → state `Running`, status checks `2/2 checks passed`.
- [x] **Allocate an Elastic IP and associate it.** ✅ 2026-05-14
	- EC2 console → left sidebar → **Network & Security → Elastic IPs** → **Allocate Elastic IP address** → **Allocate**.
	- Select the new EIP → **Actions → Associate Elastic IP address**.
	- **Resource type:** Instance; **Instance:** `knottyyoga-server` → **Associate**.
	- Note the Elastic IP — that's your origin endpoint for CloudFront in 4.6 and your SSH target.
		- 34.215.204.200
	- Cost note: an EIP is **free while attached** to a running instance; ~$3/mo only if unattached or attached to a stopped instance.
- [x] **First-boot system setup.** SSH from your laptop: ✅ 2026-05-14
	```bash
	ssh -i ~/.ssh/knottyyoga-ec2.pem ubuntu@<elastic-ip>
	```
	Then on the EC2:
	```bash
	sudo apt update && sudo apt upgrade -y
	sudo apt install -y docker.io postgresql-client ufw
	sudo systemctl enable --now docker
	sudo usermod -aG docker ubuntu
	exit                                         # log out and back in so docker-group membership applies
	ssh -i ~/.ssh/knottyyoga-ec2.pem ubuntu@<elastic-ip>
	docker ps                                    # should succeed without sudo
	```
- [ ] **No `cap_net_bind_service` setup needed.** Docker `-p 80:<internal>` maps the privileged host port regardless of the in-container user. The Crow process inside the container runs as root by default for single-process containers — fine, it's isolated by the container boundary.
- [ ] **Generate the env-file secrets and stash them.** On your laptop or on the EC2:
	```bash
	openssl rand -base64 32   # use as KNOTTYYOGA_ORIGIN_SECRET
	openssl rand -base64 32   # use as SCHEDULER_SERVICE_ACCOUNT_PASSWORD
	```
	Record both values in your password manager — you'll paste them into `/etc/knottyyoga/server.env` at the **end of Phase 4.4**, once the RDS endpoint and DB password are also known. (Writing the file in one shot after 4.4 is cleaner than the two-pass approach where you create it here with placeholders and fill in DB fields later.)
	- KNOTTYYOGA_ORIGIN_SECRET
		- See AWS Secrets
	- SCHEDULER_SERVICE_ACCOUNT_PASSWORD
		- See AWS Secrets
- [ ] **Enable `ufw` (host firewall, defense in depth with the SG).** On the EC2:
	```bash
	sudo ufw default deny incoming
	sudo ufw default allow outgoing
	sudo ufw allow 22/tcp
	sudo ufw allow 80/tcp
	sudo ufw --force enable
	sudo ufw status verbose
	```
- [ ] **CloudWatch Agent — skip for v1.** The systemd journal tailed to CloudWatch Logs (Phase 5.3) is enough. Install the agent later only when you actually want per-instance metrics beyond the EC2 defaults (memory, disk usage).

## 4.4 Database: RDS Postgres

- [x] **Create the RDS subnet group.** ✅ 2026-05-15
	- Region: **us-west-2**.
	- Top search → **RDS** → RDS console → left sidebar → **Subnet groups** → **Create DB subnet group**.
	- **Name:** `knottyyoga-db-subnet-group`
	- **Description:** same
	- **VPC:** default VPC
	- **Availability Zones:** select the same two AZs you used in 4.2 (e.g., `us-west-2a`, `us-west-2b`). RDS requires ≥2 AZs in a subnet group even for single-AZ instances.
	- **Subnets:** pick one subnet in each chosen AZ (the default-VPC public subnets you confirmed in 4.2)
	- **Create**.
- [x] **Provision the RDS instance.** ✅ 2026-05-15
	- RDS console → left sidebar → **Databases** → **Create database**.
	- **Engine options:** the picker is now a combined **"Aurora and RDS"** screen. Choose **Amazon RDS** (NOT Amazon Aurora — Aurora is a separate, pricier engine that starts at ~2 instances' worth of cost and is overkill here), then engine **PostgreSQL**.
	- **Choose a database creation method:** **Full configuration** (this is the renamed "Standard create"). Do **NOT** use **Easy create** — it applies production defaults and hides the knobs this plan needs (db.t3.micro, single-AZ, blank initial DB name, backup window, deletion protection).
	- **Engine version:** any current major is fine — latest 15.x, 16.x, or 17.x. (Note: PG 16+ tightens `CREATE DATABASE ... OWNER` — see the `GRANT` line in the "Create the application role and database" step below.)
	- **Templates:** if a Templates selector appears, pick **Dev/Test** or **Free tier** (the Production template forces Multi-AZ, which we're not paying for yet). The **Availability and durability** choice below is what actually controls cost, so that's the one that matters.
	- **Availability and durability:** **Single-AZ instance deployment (1 instance)**. The other options — *Multi-AZ instance deployment (2 instances)* and *Multi-AZ cluster deployment (3 instances)* — add a synchronous standby / reader fleet at ~2× and ~3× the instance cost. Multi-AZ is a modify-in-place change later if HA is ever needed, so there's no lock-in from starting single-AZ.
	- **Settings:**
		- DB instance identifier: `knottyyoga`
		- Master username: `postgres`
		- Master password: generate with `openssl rand -base64 24`, save to password manager
			- AWS Secrets
	- **Instance configuration → Instance type** (the console renamed "DB instance class" → "Instance type"):
		- The list defaults to a class-family filter. If you only see `db.m*`/`db.r*` classes at `.large` and up, the filter is on **Standard** or **Memory optimized** — switch the family selector to **Burstable classes (includes t classes)** to expose the `t` family.
		- Pick **`db.t3.micro`**. If it doesn't appear even under the Burstable filter, use **`db.t4g.micro`** (Graviton/ARM burstable — ~10% cheaper, and the managed DB host's architecture is independent of our x86 app, so this is a no-downside swap). `db.t3.small` is the fallback if more RAM is wanted.
		- **Caveat tied to the Availability choice above:** *Multi-AZ cluster deployment (3 instances)* does not support burstable classes at all — if you picked that, no `t`-class will ever show. Burstable requires **Single-AZ instance deployment (1 instance)** (or Multi-AZ *instance* deployment), which is what this plan uses.
		- The `d`-suffixed variants (`db.m7gd.*` etc.) add local NVMe SSD and are irrelevant here — RDS data lives on the separate gp3 EBS volume configured under **Storage** below.
	- **Storage:**
		- Storage type: gp3
		- Allocated storage: `20` GiB
		- Storage autoscaling: enabled, Maximum storage threshold `100` GiB
	- **Connectivity:**
		- Compute resource: **Don't connect to an EC2 compute resource** (we'll wire it manually via SG)
		- VPC: default VPC
		- DB subnet group: `knottyyoga-db-subnet-group`
		- Public access: **No**
		- VPC security group (firewall): **Choose existing** → select `knottyyoga-db`. **Remove** the auto-selected `default` SG if it's there.
		- Availability Zone: pick either of your two AZs
		- Database port: 5432
		- **Create an RDS Proxy: leave UNCHECKED.** RDS Proxy is a managed connection pooler for serverless/Lambda apps that storm the DB with short-lived connections. This is a single long-running Crow process with a small stable libpqxx pool — no pooler needed. It also bills ~$0.015/vCPU-hr (~$22/mo on a 2-vCPU `db.t3.micro`, more than the instance itself). Can be added later without an instance rebuild if the API ever moves to Lambda or hits connection limits.
	- **Database authentication:** Password authentication
	- **Monitoring:**
		- Enhanced Monitoring: **off** for now (saves a few dollars; toggle on later if you need it)
		- **Performance Insights: ENABLE** with the default **7-day retention** (the 7-day tier is free and is the single most useful post-launch "why is it slow" tool — top SQL, waits, DB load). Do not raise retention (paid tier). If the wizard greys it out on `db.t3.micro` for the chosen engine version/region, skip it — no loss.
		- **Log exports (publish to CloudWatch Logs): leave ALL unchecked for v1.** *IAM DB auth error log* is pointless — we use password auth, not IAM DB auth. *PostgreSQL log* / *Upgrade log* are viewable in the RDS console (Logs & events tab) without paying CloudWatch ingestion; exporting is a no-reboot modify you can enable later if you ever want centralized retention. (App-log → CloudWatch in Phase 5.3 is the Crow journal, a separate concern from DB logs.)
		- **DevOps Guru: NO.** Paid per-resource ML anomaly detection — real monthly cost and overkill for one low-traffic `db.t3.micro`; Performance Insights covers what you'd actually inspect.
	- **Additional configuration:**
		- Initial database name: **leave blank** — we'll create the app DB manually so it's owned by a non-superuser role
		- Backup retention period: `7` days
		- Backup window: **Choose a window** (NOT "No preference" — that lets AWS pick a random slot that may hit booking hours). Set **09:00–10:00 UTC** (= 02:00–03:00 PDT, dead of night Pacific).
		- Backup tags: leave **Copy tags to snapshots** checked (default) **and** also check **Copy tags to automated backup**. Hygiene only, zero cost — keeps orphaned backups/snapshots identifiable later.
		- Backup replication: leave **Enable replication in another AWS Region UNCHECKED**. Cross-region backup copies add data-transfer + duplicate-storage cost; cross-region DR is explicitly out of scope for v1 (7-day in-region PITR + the manual PITR verification step cover us). The nested "Enable encryption" checkbox is moot while replication is off — ignore it.
		- Encryption: **enabled** (default; can't be changed later)
			- **AWS KMS key: keep the default `aws/rds`** (AWS-managed key). Free, zero-maintenance, AES-256 at rest. A customer-managed key (CMK) is only worth its $1/mo + API cost for customer-controlled key policy, per-key CloudTrail audit, or **cross-account encrypted snapshot sharing** (snapshots under `aws/rds` can't be shared/copied to another account) — none planned here (single account, single instance). Key choice is fixed at creation like encryption itself. Unrelated to the app-level `config_secrets` encryption (`KNOTTYYOGA_SECRET_KEY` / Phase 8) — this only protects the RDS storage volume.
		- Maintenance window: **Choose a window** (NOT "No preference"). Must NOT overlap the backup window — AWS defers maintenance if it collides. Set **Sunday 11:00–12:00 UTC** (= 04:00–05:00 PDT Sunday): weekend, early-morning Pacific, cleanly after the 09:00–10:00 UTC backup window.
		- **Deletion protection: ENABLE** ← important
		- Enable auto minor version upgrade: **keep CHECKED** (default). Same-major security/bug patches (e.g., 15.5 → 15.6) applied in the maintenance window — want these automatic on an unattended host. Major-version upgrades are never automatic; you still control those.
	- **Create database**.
	- Wait ~10 minutes for status to flip from `Creating` to `Available`.
- [x] **Record the endpoint.** RDS console → Databases → `knottyyoga` → **Connectivity & security** tab → copy the **Endpoint** (e.g., `knottyyoga.xxxxxx.us-west-2.rds.amazonaws.com`). Save it — you'll use it as `KNOTTYYOGA_DB_HOST` in the consolidated `server.env` step at the end of this phase. ✅ 2026-05-15
	- knottyyoga.cjise0agyhh6.us-west-2.rds.amazonaws.com
- [x] **Download the RDS CA bundle to the EC2.** On the EC2: ✅ 2026-05-15
	```bash
	sudo curl -fsSL -o /etc/knottyyoga/rds-ca.pem \
	  https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
	sudo chmod 644 /etc/knottyyoga/rds-ca.pem
	```
	Pairs with `KNOTTYYOGA_DB_SSLMODE=verify-full` from the env file. (Phase 1.1 plans the sslmode support in the server's DB connection layer.) The **Certificate authority** dropdown in the create-database wizard (defaults to `rds-ca-rsa2048-g1`) needs no change — `global-bundle.pem` contains the roots for every RDS CA (RSA-2048, RSA-4096, ECC), so `verify-full` validates regardless of which one is selected. RSA-2048 default is the broadly-compatible choice and the `g1` CAs are valid into the 2060s (no `rds-ca-2019`-style forced rotation).
- [x] **Generate the application DB password.** This is NOT supplied by AWS — you create it yourself, now. It's a fresh, separate password for the app's `knottyyoga` database role (distinct from the RDS *master* password for `postgres`). On the EC2 or your laptop: ✅ 2026-05-15
	```bash
	openssl rand -base64 24
	```
	Save it to your password manager. It plugs into the `CREATE ROLE` statement below **and** becomes `KNOTTYYOGA_DB_PASSWORD` in `server.env`.
- [x] **Create the application role and database.** From the EC2 (the only host that can reach RDS, thanks to the SG rule): ✅ 2026-05-15
	```bash
	PGPASSWORD='My84dSDdpIBwXgIKb4yi1doef2JoJA+T' psql \
	  "host=knottyyoga.cjise0agyhh6.us-west-2.rds.amazonaws.com port=5432 user=postgres dbname=postgres sslmode=verify-full sslrootcert=/etc/knottyyoga/rds-ca.pem"
	```
	Then in the psql shell (paste the password from the previous step in place of `<app password>`):
	```sql
	CREATE ROLE knottyyoga LOGIN PASSWORD 'LynKL2JHmSpo+u1QJ8q0SX0LhCmVvExb';
	GRANT knottyyoga TO postgres;   -- REQUIRED on PG 16+: CREATE DATABASE ... OWNER needs the
	                                -- creating role to be a member of the owner role. Harmless on PG 15.
	                                -- (`postgres` is the RDS master username you're connected as.)
	CREATE DATABASE knottyyoga OWNER knottyyoga;
	\q
	```
	Save the app password — you'll use it as `KNOTTYYOGA_DB_PASSWORD` in the consolidated `server.env` step below. The `postgres` master password is only used for occasional maintenance — keep it in the password manager but **not** in the env file.
- [x] **Create `/etc/knottyyoga/server.env`** (deferred from Phase 4.3 — all values are now known). On the EC2: ✅ 2026-05-15
	```bash
	sudo mkdir -p /etc/knottyyoga
	sudo nano /etc/knottyyoga/server.env
	```
	Paste, substituting the values you've collected so far (origin secret + scheduler password from Phase 4.3; RDS endpoint + app password from earlier in this phase):
	```
	PORT=80
	KNOTTYYOGA_ORIGIN_SECRET=Rpxpk23whEtmToEMmEZpuFk0+KwK/ukpTZD3AQauoDQ=
	KNOTTYYOGA_TRUST_PROXY=1
	KNOTTYYOGA_DB_HOST=knottyyoga.cjise0agyhh6.us-west-2.rds.amazonaws.com
	KNOTTYYOGA_DB_NAME=knottyyoga
	KNOTTYYOGA_DB_USER=knottyyoga
	KNOTTYYOGA_DB_PASSWORD=LynKL2JHmSpo+u1QJ8q0SX0LhCmVvExb
	KNOTTYYOGA_DB_SSLMODE=verify-full
	KNOTTYYOGA_DB_SSLROOTCERT=/etc/knottyyoga/rds-ca.pem
	SCHEDULER_SERVICE_ACCOUNT_PASSWORD=d5jLtv36Ng8mi/O7nKLW/JztPZR3St9/1HUkBH9x2Nw=
	```
	Lock it:
	```bash
	sudo chmod 600 /etc/knottyyoga/server.env
	sudo chown root:root /etc/knottyyoga/server.env
	```
	- [ ] ⚠️ **Found 9/17 while writing the runbook: the block above has no `HONUWARE_SECRET_KEY`.** It is the at-rest encryption key for `config_secrets` (honuware Phase 8.1). `MakeSecretsAtRest` uses the env var whenever it is set, falls back to a hard-coded dev key when it is not in non-prod mode, and **throws in prod mode** — `ValidateProdEnvironment` lists it among the required vars, so flipping `production_mode_on` (§1.5) with the file as written refuses to boot. **It must go in before the first `--migrate` (5.1):** rows encrypted under the dev key are unreadable under a real key added afterwards, which would mean redoing every `set_secret`. Generate with `openssl rand -base64 32 | tr '+/' '-_' | tr -d '='` — **URL-safe and unpadded, not plain `openssl rand -base64 32`**; the decoder is libsodium's URLSAFE_NO_PADDING variant and refuses a standard key with a misleading "not valid base64" (verified 9/22). Password manager, then append `HONUWARE_SECRET_KEY=<value>` to the file. `RUNBOOK.md` §4 carries the same warning in the bootstrap order.
- [ ] **Verify PITR (Point-in-Time Recovery) once — DEFER TO PHASE 5.1. Do NOT run during 4.4.** At this point in 4.4 the `knottyyoga` database is empty (no schema, no data), so a restore proves nothing. This is a Phase 5.1 smoke-test task: run it only *after* the app is deployed and has real data. RDS gives 7-day PITR automatically; this just proves the restore mechanism works and the data is actually in the backups before you ever need it for real.

	When you do it (Phase 5.1), step by step:
	1. RDS console → Databases → select `knottyyoga` → **Actions → Restore to point in time**.
	2. **Restore time:** choose **Latest restorable time** (you're proving the mechanism + data presence, not recovering a specific moment — no need for Custom).
	3. The wizard opens the full create-DB form, pre-filled from the source instance. **Leave everything at the pre-filled values EXCEPT these overrides:**
		- **DB instance identifier:** `knottyyoga-pitr-test` ← this is the "name" field you were looking for.
		- **DB instance class:** confirm `db.t3.micro` (it lives only minutes — don't let it inherit anything bigger).
		- **Availability & durability:** Single-AZ (don't let it flip to Multi-AZ — wasteful even briefly).
		- **VPC:** default VPC; **DB subnet group:** `knottyyoga-db-subnet-group`; **Public access:** No.
		- **VPC security group:** **`knottyyoga-db`** — *the critical override.* If the wizard defaults to the `default` SG, the EC2 can't reach the test instance on 5432 and the test will falsely look like a failure. Remove `default` if present.
		- **Deletion protection:** **OFF.** The source has it ON and the restore may inherit it; turning it off here makes cleanup a single Delete with no extra modify step.
	4. **Restore DB instance** → wait ~10 min for status `Available`.
	5. Copy the test instance's endpoint (Connectivity & security tab). From the EC2, connect and prove data is present:
		```bash
		PGPASSWORD='<knottyyoga app password>' psql \
		  "host=<knottyyoga-pitr-test endpoint> port=5432 user=knottyyoga dbname=knottyyoga sslmode=verify-full sslrootcert=/etc/knottyyoga/rds-ca.pem" \
		  -c "SELECT count(*) FROM people;"
		```
		A non-zero count (or any table with data you expect post-smoke-test) confirms the backup contains real data.
	6. **Clean up:** RDS console → select `knottyyoga-pitr-test` → **Actions → Delete** → **uncheck** "Create final snapshot", **uncheck** "Retain automated backups", type the confirmation phrase → **Delete**. (Deletion protection was set OFF in step 3, so this is one action.)

## 4.5 DNS + TLS

`knottyyoga.com` is registered at a non-AWS provider. We're keeping the registrar there but moving DNS *hosting* to Route 53 so CloudFront alias records work cleanly. The registrar just needs its NS records updated.

- [x] **Create the Route 53 hosted zone.** ✅ 2026-05-15
	- Top search → **Route 53** → Route 53 console (region-agnostic — no picker needed).
	- Left sidebar → **Hosted zones** → **Create hosted zone**.
	- **Domain name:** `knottyyoga.com`
	- **Type:** Public hosted zone
	- **Create hosted zone**. Cost: $0.50/mo per zone.
	- On the new zone's page, note the four values in the `NS` record (e.g., `ns-123.awsdns-12.com`, `ns-456.awsdns-34.net`, ...). You'll paste these at your registrar in the next step.
		- ns-1258.awsdns-29.org
		- ns-1637.awsdns-12.co.uk
		- ns-786.awsdns-34.net 
		- ns-148.awsdns-18.com
- [x] **Repoint your current registrar's nameservers at Route 53.** ✅ 2026-05-15
	- Log in to your existing DNS provider (where you registered the domain).
	- Find **Nameservers** or **DNS Management → Custom Nameservers**.
	- Replace the existing nameservers with the four Route 53 values from the previous step.
	- **Do not** delete the domain registration itself — you're only changing who hosts the DNS records.
	- Propagation is usually <1 hour but can take up to 48 hours. Monitor with `dig +short NS knottyyoga.com` (or https://www.whatsmydns.net/#NS/knottyyoga.com) — when both show the four `*.awsdns-*` values, propagation is done.
- [ ] **(Optional, later)** Migrate the registrar itself to Route 53 (Route 53 → **Registered domains → Transfer in**). Costs roughly the same per year; consolidates billing. Non-urgent — can be done any time without disturbing anything.
- [x] **Request the ACM certificate in `us-east-1`.** ✅ 2026-05-15
	- AWS console region picker → **flip to us-east-1 (N. Virginia)**. CloudFront only reads certs from `us-east-1`, regardless of where your app runs. This is the #1 ACM gotcha.
	- Top search → **Certificate Manager** → ACM console.
	- **Request certificate** → **Request a public certificate** → **Next**.
	- **Fully qualified domain names:**
		- `knottyyoga.com`
		- click **Add another name to this certificate** → `www.knottyyoga.com`
	- **Validation method:** DNS validation (recommended)
	- **Key algorithm:** RSA 2048
	- **Certificate export:** leave the default **Disable export**. The cert is consumed only by CloudFront (an ACM-integrated service that reads it directly from ACM), so the private key never needs to leave AWS. Non-exportable is free (exportable carries a per-cert charge), more secure (no downloadable key material), and auto-renews with no action. Enable export only if some non-AWS host ever needs the raw key — not the case here (TLS terminates at CloudFront; EC2 runs plain HTTP).
	- **Request**.
	- This is a **single certificate with two names** (`knottyyoga.com` primary + `www.knottyyoga.com` as a SAN) → **one ARN** covering both. There is not a separate cert/ARN per domain.
	- On the new certificate's page (status `Pending validation`), expand **each** domain row and click **Create records in Route 53** → confirm. ACM writes the validation `CNAME`s into your hosted zone for you. Both names must validate before the cert issues, so don't skip either row.
	- The ARN is assigned at request time and never changes — you can copy it now (during `Pending validation`); it's stable through to `Issued`. **But** CloudFront (4.6) only accepts the cert once status shows `Issued`, so the real gate for 4.6 is the status flip, not having the ARN.
	- Wait 5–30 minutes for status `Pending validation` → `Issued`. Save the ARN — CloudFront needs it in 4.6.
		- AWS Secrets
	- **Remember to flip the region picker back to `us-west-2` before any later step.**
- [ ] **Do NOT create production `A` records yet.** Soft-launch / friends-and-family testers can hit the site via the `dXXXXXX.cloudfront.net` URL CloudFront gives you. Skipping the `A` records means there's no live production DNS to break while you're shaking things out.
	- **9/17 — the "nothing to break" part is now literally true, which changes the gate.** The old site came down during covid; `knottyyoga.com` has resolved to nothing since the NS switch in May (confirmed: the Route 53 zone has no `A`/`www` records). So DNS is not what holds this back — the **certificate** is: the alias records need the distribution to carry `knottyyoga.com` + `www` as alternate domain names, which needs an *Issued* `us-east-1` cert, and the saved ARN is `us-west-2` (see 4.6's notes). The work item is therefore: re-request/validate the cert in `us-east-1` → attach it + both CNAMEs to `E23TY4IAUHGM6H` → create the two alias records below. Do it whenever convenient rather than as a go-live ceremony — and **before the prod-mode smoke test in 5.1**, because prod mode pins CORS and the cookie `Domain` to `website_address = knottyyoga.com`, which the `cloudfront.net` URL cannot satisfy.
- [ ] **At go-live: create the alias records.** **DEPENDS ON PHASE 4.6 — do not attempt during 4.5.** The "Choose distribution" dropdown is a fixed auto-populated picker (you cannot type or search a name into it). It stays **empty until** (a) the CloudFront distribution exists (created in 4.6) **and** (b) that distribution has `knottyyoga.com` + `www.knottyyoga.com` set as **Alternate domain names (CNAMEs)** with the ACM cert attached. Until both are true the dropdown shows nothing — that is expected, not a bug. Come back here only at actual go-live, after 4.6 is fully done and tested via the `dXXXXXX.cloudfront.net` URL.
	- Route 53 → Hosted zones → `knottyyoga.com` → **Create record**.
	- Record 1 (apex):
		- Record name: (leave blank — apex)
		- Record type: `A`
		- **Alias:** ON
		- Route traffic to: **Alias to CloudFront distribution** → pick yours
		- **Create records**.
	- Repeat for `www`:
		- Record name: `www`
		- Record type: `A`
		- **Alias:** ON
		- Route traffic to: **Alias to CloudFront distribution** → pick yours
	- Alias records have no per-query charge (a `CNAME` would).
- [ ] **At go-live: flip Square from sandbox to production.** Phase 4.8's secret bootstrap covers updating `kSquareEnvironment` and the access token in `config_secrets`.

## 4.6 S3 + CloudFront

### S3 bucket for the frontend

- [x] **Create the bucket.** ✅ 2026-05-18
	- Region: **us-west-2** (same as EC2 — keeps the API ↔ bucket latency low if anything ever needs cross-talk).
	- Top search → **S3** → S3 console → **Create bucket**.
	- **Bucket type: General purpose** (NOT Directory). Directory = S3 Express One Zone: single-AZ, ultra-low-latency, different API — reduced durability and zero benefit for a CloudFront-cached SPA origin.
	- **Bucket namespace: Global namespace** (auto-pairs with General purpose; "Account regional namespace" goes with Directory buckets).
	- **Bucket name:** `knottyyoga-ui-prod` (General-purpose names are globally unique across all AWS accounts — if taken, append a random suffix like `knottyyoga-ui-prod-7k2j`).
	- **Object Ownership:** ACLs disabled (recommended).
	- **Block Public Access settings:** leave **all four** blocks **ON**. CloudFront will reach the bucket via Origin Access Control (OAC) — far more secure than making the bucket public.
	- **Bucket Versioning:** **Enable** (cheap insurance if a bad deploy overwrites files).
	- **Default encryption:** SSE-S3 (default, free).
	- **Create bucket**.
- [x] **Confirm static website hosting is OFF.** Bucket → **Properties** tab → "Static website hosting" should say **Disabled**. CloudFront serves the content, not S3's website endpoint. ✅ 2026-05-18
- [x] **Create the `ci-deploy` IAM user (for GitLab CI to push builds).** ✅ 2026-09-17
	- Top search → **IAM** → IAM console → left sidebar → **Users → Create user**.
	- **User name:** `ci-deploy`
	- **Provide user access to the AWS Management Console:** **No** (programmatic-only).
	- **Next → Permissions options:** **Attach policies directly**.
	- Open a new browser tab → IAM → **Policies → Create policy** → **JSON** tab → paste:
		```json
		{
		  "Version": "2012-10-17",
		  "Statement": [
		    {
		      "Sid": "S3Deploy",
		      "Effect": "Allow",
		      "Action": ["s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
		      "Resource": [
		        "arn:aws:s3:::knottyyoga-ui-prod",
		        "arn:aws:s3:::knottyyoga-ui-prod/*"
		      ]
		    },
		    {
		      "Sid": "CloudFrontInvalidate",
		      "Effect": "Allow",
		      "Action": ["cloudfront:CreateInvalidation", "cloudfront:GetInvalidation"],
		      "Resource": "*"
		    }
		  ]
		}
		```
		(`GetInvalidation` is what `deploy_ui.sh`'s `WAIT=1` polls; without it the upload and the invalidation still succeed but the wait fails with AccessDenied.)
		Name the policy `knottyyoga-ci-deploy` → **Create policy**.
	- Back in the user-creation tab → refresh the policy list → search `knottyyoga-ci-deploy` → check it → **Next → Create user**.
	- Open the new user → **Security credentials** tab → **Create access key** → use case **Application running outside AWS** → **Next → Create access key**.
	- Copy the **Access key ID** and **Secret access key** — secret is shown **only once**. Save both to your password manager *and* to GitLab CI (Project → Settings → CI/CD → Variables) as:
		- `AWS_ACCESS_KEY_ID` — **Masked**, **Protected**
			- AWS Secrets
		- `AWS_SECRET_ACCESS_KEY` — **Masked**, **Protected**
			- AWS Secrets

### CloudFront distribution

- [x] **Create the distribution with the S3 origin and default behavior.** ✅ 2026-05-18
	- CloudFront is a global service; the region picker doesn't matter for the distribution itself, but the **ACM cert dropdown only shows certs from `us-east-1`** — that's the constraint, not the picker setting.
	- Top search → **CloudFront** → CloudFront console → **Create distribution**.
	- **Plan selector (Free / Pro $15 / Business $200): choose Free.** The Free plan's usage allowance is **1,000,000 requests/month + 100 GB egress/month** (it is *not* a per-object size cap — CloudFront serves multi-MB/GB objects on any plan, so the app's large studio images in `business_logic/images/` are fine, cached at edge after first fetch). Soft-launch studio traffic is nowhere near either ceiling: even ~5k visits/mo with image browsing is ~250–400k requests and well under 100 GB. Exceeding the allowance does not throttle or break anything — CloudFront just bills the overage at standard rates (see the CloudFront cost section below, which already concluded "effectively free at this scale"). Upgrading Free → Pro later is non-destructive (no distribution rebuild), so there is no reason to pre-pay for a soft launch.
	- **"Get started" page (redesigned wizard) field choices:**
		- **Distribution name:** `knottyyoga-prod` (just a tag, changeable later — keep consistent with `knottyyoga-server`/`knottyyoga-ui-prod`/`knottyyoga-db`).
		- **Description:** optional — e.g., `Knotty Yoga production CDN — S3 frontend + /api proxy to EC2`, or leave blank.
		- **Distribution type: Single website or app.** NOT "Multi-tenant architecture" (that's CloudFront's SaaS feature for serving many customer domains from one shared template — wrong model for a single studio site).
		- **Domain (Route 53 managed domain - optional): LEAVE BLANK / skip.** Deliberate: the plan does not point `knottyyoga.com` at CloudFront until go-live — soft-launch testers use the `dXXXXXX.cloudfront.net` URL so there's no live production DNS to break (see "Do NOT create production A records yet" in 4.5). Using this field now would create the production alias records prematurely. It also avoids the unresolved **us-east-1 cert** requirement (the saved cert ARN at line ~709 is `us-west-2`, which CloudFront cannot use) — a custom domain here would demand a valid us-east-1 Issued cert. Build with the default `*.cloudfront.net` domain + default cert, soft-launch-test against that URL, then add alternate domain names + cert + Route 53 records at go-live (non-destructive edit; Phase 4.5 alias step).
	- **Origin (redesigned wizard — this is the S3 frontend origin; the EC2 `/api/*` origin is a separate one added later):**
		- **Origin type: Amazon S3.** (The later `/api/*` origin targets the EC2 — none of the offered types, S3 / ELB / API Gateway, is a plain custom HTTP server; handle the EC2 origin when adding the second origin/behavior.)
		- **S3 origin:** pick `knottyyoga-ui-prod` from the bucket picker.
		- **Origin path:** leave blank (Angular bundle is at bucket root).
		- **"Allow private S3 bucket access to CloudFront": CHECK it.** This is the new wizard's one-click replacement for manually creating an OAC + pasting a bucket policy: it creates the OAC and wires the bucket policy so the bucket stays fully private (Block Public Access stays ON) and only CloudFront can read it. **After the distribution is created, verify** S3 → bucket → Permissions → Bucket policy actually received the OAC grant. If the wizard shows a "copy this policy" banner instead of auto-applying, paste it into the S3 bucket policy — otherwise CloudFront 403s every object.
		- **Origin settings: Use recommended origin settings** (default timeouts/attempts are fine for a static S3 origin).
		- **Cache settings: Use recommended cache settings tailored to serving S3 content** (≈ `Managed-CachingOptimized`: long TTLs + compression — correct for the static bundle). This is the **default behavior only**. The `/api/*` path needs the opposite (caching disabled, all headers/query/cookies forwarded) — a separate cache behavior added with the EC2 origin later. Do not let the S3-recommended caching apply to `/api/*`.
	- **Default cache behavior — NOT a separate wizard page in the redesigned flow.** The streamlined wizard collapses it into the "Use recommended cache settings for S3" choice you made under Origin. The detailed knobs below are applied/verified **post-creation** via **Distribution → Behaviors → (default) → Edit** (see the post-creation checklist after "Create distribution"). Target state for the default behavior:
		- Viewer protocol policy: **Redirect HTTP to HTTPS**
		- Allowed HTTP methods: GET, HEAD
		- Cache policy: `Managed-CachingOptimized`; Origin request policy: (none); Response headers policy: `Managed-SecurityHeadersPolicy`
		- Compress objects automatically: **Yes**
	- **Web Application Firewall (WAF) / Enable security:** Do **not** enable security protections for v1 — it has a per-month base cost (~$5/mo Web ACL + $1/mo per rule + $0.60 per M requests). The redesigned wizard offers a **"Use monitor mode"** checkbox (unchecked by default) — leave it unchecked and skip the whole security section. Monitor mode is NOT a free WAF: it still creates a billed Web ACL, it only makes rules *count* instead of *block*. WAF is a non-destructive post-launch attach if real abuse/bot traffic appears; when that day comes, enabling monitor mode *first* (count rules, watch CloudWatch for false positives, then flip to blocking) is the right rollout — but it's not a v1 expense.
	- **Settings — also NOT inline in the redesigned wizard.** The streamlined "Get started" flow does not ask for price class / domain / cert / root object / HTTP versions / logging. It creates the distribution with defaults; you apply these **after creation** (see post-creation checklist below). At creation there is nothing to enter for these.
	- **Create distribution.**
	- Wait ~5–10 minutes for status `Deployed`. Note the distribution's domain name (`dXXXXXX.cloudfront.net`) and its **Distribution ID** (e.g., `E1234567890ABC`) — you'll need the ID for cache invalidations.
		- dv1tgxa9ok30f.cloudfront.net
		- Distribution ID: `E23TY4IAUHGM6H` (recorded 9/17 — the `DISTRIBUTION_ID` for `deploy_ui.sh`)
- [x] **Post-creation settings (the redesigned wizard defers all of these — apply them now via the distribution's tabs).** ✅ 2026-05-21
	- **Distribution → Settings → Edit:**
		- **Default root object: `index.html`** ← **CRITICAL and easy to miss.** The streamlined wizard does NOT set this; without it, the distribution root returns S3 XML/error instead of the Angular app.
		- **Price class: Use only North America and Europe** (cheaper than worldwide; fine for this audience).
		- Supported HTTP versions: ensure **HTTP/2 + HTTP/3**.
		- **IPv6: ENABLED** (the default — leave on). Viewer-edge setting only; lets IPv6-only/dual-stack viewers reach the nearest edge over IPv6. Does NOT touch the origin side — CloudFront → S3 and CloudFront → EC2 remain IPv4, so the `knottyyoga-web` SG rules (IPv4-only) are unaffected. One thing to know, not a reason to disable: viewers connecting via IPv6 land an IPv6 address in `X-Forwarded-For`, which the EC2's `ResolveClientIp` (`business_logic/auth/proxy_trust.cpp`) will surface in audit/login-attempt logs — expected, not a bug. If WAF is ever added with IP-based rules, those rules would need to cover both v4 and v6.
		- Standard logging: **off** for v1.
		- **Custom SSL certificate: leave as "Default CloudFront certificate" — nothing to attach now.** A custom cert is only meaningful once you add Alternate domain names; both are deferred to go-live (Phase 4.5 alias step). Also still gated on resolving the **us-east-1 cert** issue (saved ARN at line ~709 is `us-west-2`, which CloudFront cannot use — re-request in us-east-1 before go-live). Until then, the default cert handles TLS for the `*.cloudfront.net` URL the soft launch runs on.
		- **Alternate domain names (CNAME): leave blank.** Same go-live deferral. Adding a CNAME without a matching us-east-1 Issued cert is blocked anyway.
	- **Distribution → Behaviors → (default) → Edit.** This is the S3 static frontend behavior — **not** the webserver. Webserver methods/forwarding go on the separate `/api/*` behavior added further below; do not conflate the two.
		- **Viewer protocol policy:** Redirect HTTP to HTTPS.
		- **Allowed HTTP methods: GET, HEAD** (keep default). Do NOT allow POST/PUT/PATCH/DELETE here — those belong on `/api/*`. S3 wouldn't accept them for static hosting anyway; widening this field at the default behavior buys nothing and just widens attack surface against the bucket.
		- **Restrict viewer access: No.** That option enforces CloudFront signed URLs/cookies (paywall use case). The Angular bundle is public; everyone needs it. Auth happens in-band via session cookies handled by the app, not at the CloudFront edge.
		- **Cache policy:** `Managed-CachingOptimized`.
		- **Origin request policy: None.** S3 serves a static object by URL — viewer headers/cookies/queries are irrelevant to S3 and only pollute the cache key. Best practice for an S3 static origin.
		- **Response headers policy:** `Managed-SecurityHeadersPolicy` (HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, etc.).
		- **Compress objects automatically:** Yes.
		- The "recommended S3 cache settings" picked at creation should already match most of this — verify, don't assume.
- [x] **Verify (or paste) the OAC bucket policy in S3.** ✅ 2026-09-17
	- **Where "the JSON CloudFront gave you" comes from — this step was written for the OLD wizard.** The old create-distribution flow ended with a yellow banner, *"The S3 bucket policy needs to be updated"*, with a **Copy policy** button; that banner is the JSON this step meant. The redesigned wizard you used ("Allow private S3 bucket access to CloudFront", above) applies the policy itself, so there may be nothing to paste. Check first, then fall back:
	1. **Check whether it is already there.** S3 console → `knottyyoga-ui-prod` → **Permissions** tab → **Bucket policy**. If there is a statement with `"Principal": { "Service": "cloudfront.amazonaws.com" }` and an `AWS:SourceArn` condition naming your distribution, the wizard did it — tick this step and move on.
	2. **If the policy box is empty, the Copy button lives on the ORIGIN, not the distribution.** CloudFront → `knottyyoga-prod` → **Origins** tab → select the S3 origin → **Edit** → scroll to **Origin access** (Origin access control settings) → **Copy policy**. Then S3 → bucket → Permissions → **Bucket policy → Edit** → paste → **Save changes**.
	3. **Or write it by hand** — it is a fixed shape with two blanks, and the account ID is the one from the console login URL (line ~409). The Distribution ID is the `E…` value in the **ID** column of the CloudFront distribution list (the doc recorded the domain `dv1tgxa9ok30f.cloudfront.net` but not the ID — note it beside the domain once you have it):
		```json
		{
		  "Version": "2008-10-17",
		  "Id": "PolicyForCloudFrontPrivateContent",
		  "Statement": [
		    {
		      "Sid": "AllowCloudFrontServicePrincipal",
		      "Effect": "Allow",
		      "Principal": { "Service": "cloudfront.amazonaws.com" },
		      "Action": "s3:GetObject",
		      "Resource": "arn:aws:s3:::knottyyoga-ui-prod/*",
		      "Condition": {
		        "StringEquals": {
		          "AWS:SourceArn": "arn:aws:cloudfront::957014951609:distribution/<DISTRIBUTION_ID>"
		        }
		      }
		    }
		  ]
		}
		```
		`s3:GetObject` only — CloudFront reads objects, it never lists or writes. The `SourceArn` condition is what makes the grant safe with Block Public Access still ON: the service principal is shared by every CloudFront distribution in the world, and the condition narrows it to yours.
	- Without this policy, CloudFront gets `403 Forbidden` from S3 on every request — which, once the SPA fallback (below) is in place, shows up as `index.html` for every URL *including the bundle's own JS*, i.e. a blank page rather than an obvious error. Test with a direct object URL (`https://dv1tgxa9ok30f.cloudfront.net/index.html`) before adding the fallback so a 403 is still visible as a 403.
- [x] **Add the API origin for `/api/*`.** ✅ 2026-09-17
	- CloudFront → your distribution → **Origins** tab → **Create origin**.
	- **Origin domain: `ec2-34-215-204-200.us-west-2.compute.amazonaws.com`** — the EC2's **Public IPv4 DNS** (EC2 → Instances → `knottyyoga-server` → Details). **CloudFront refuses a bare IP** ("Origin domain cannot be an IP address" — an earlier draft of this step said to type the IP; it was wrong). Every Elastic IP gets this AWS-assigned name — the IP with dashes plus the region — and it resolves to `34.215.204.200` for as long as the EIP stays associated, which is the same lifetime the IP itself has. (The alternative is a Route 53 record such as `origin.knottyyoga.com → A 34.215.204.200`; not needed, and it would put an A record in the production zone before go-live.)
	- **Protocol:** **HTTP only**
	- **HTTP port:** 80
	- **Add custom header:**
		- Header name: `X-Origin-Secret`
		- Value: the origin secret you generated in Phase 4.3 (`openssl rand -base64 32`, saved to the password manager as "AWS Secrets") and wrote into `/etc/knottyyoga/server.env` at the end of Phase 4.4 — it is the `KNOTTYYOGA_ORIGIN_SECRET=…` line in that block (line ~647 of this document; the file on the EC2 uses that legacy spelling, which the server honours as a fallback for `HONUWARE_ORIGIN_SECRET`). Paste everything after the `=`, trailing `=` of the base64 included. To confirm against the live file: `sudo grep ORIGIN_SECRET /etc/knottyyoga/server.env` on the EC2. **Must match exactly** — the Crow middleware (Phase 1.7) compares this header on every API request and 403s without it.
	- **Create origin**.
	- Why a header instead of SG-by-IP-prefix? CloudFront's egress IP ranges churn; chasing them in security groups is operational pain. The shared-secret header is the pragmatic answer — no nginx needed.
- [x] **Add the `/api/*` behavior.** ✅ 2026-09-17
	- CloudFront → your distribution → **Behaviors** tab → **Create behavior**.
	- **Path pattern:** `api/*`
	- **Origin and origin groups:** the API origin you just created
	- **Viewer protocol policy:** Redirect HTTP to HTTPS.
	- **Allowed HTTP methods:** GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE (the "all methods" option). The API uses all of them — login is POST, updates PUT/PATCH, deletes DELETE, CORS preflights OPTIONS.
	- **Restrict viewer access: No.** Auth is enforced in-band by the Crow app via session cookies; CloudFront-edge signed URLs/cookies aren't part of the auth model.
	- **Cache policy:** `Managed-CachingDisabled` (every API response is dynamic; never cache).
	- **Origin request policy:** `Managed-AllViewerExceptHostHeader`. Forwards all viewer cookies, query strings, and headers — except `Host`, which is stripped so the EC2 sees its own Host (avoids collisions with the origin guard / Crow routing).
	- **Response headers policy:** `Managed-SecurityHeadersPolicy` (consistent with the default behavior; adds HSTS/`X-Content-Type-Options`/etc. to API JSON responses too).
	- **Compress objects automatically: Yes** (the default) — but it is inert here: CloudFront only compresses when the cache policy has Gzip/Brotli support on, and `Managed-CachingDisabled` has both off. Leaving it Yes is harmless, matches the default behavior, and starts working if a custom TTL-0 policy with encodings ever replaces CachingDisabled. If API compression ever matters, do it at the origin (Crow), not here.
	- **Create behavior**.
- [x] **Add SPA fallback error responses.** Without this, refreshing on `/calendar` or any deep link returns 403/404 from S3 (the bucket doesn't actually contain `/calendar/index.html`). ✅ 2026-09-17
	- CloudFront → your distribution → **Error pages** tab → **Create custom error response**.
	- Response 1:
		- HTTP error code: **403: Forbidden**
		- Customize error response: **Yes**
		- Response page path: `/index.html`
		- HTTP response code: **200: OK**
	- Repeat for **404: Not Found**.
- [x] **Cache invalidation hygiene.** *Not a step — the rationale for what the deploy script below invalidates.* Angular content-hashes the root bundles (`main-XXXX.js`, `styles-XXXX.css`, `media/*`), so those never need invalidating: a new build has new names and the old ones stay valid for anyone still holding the old `index.html`. What is **not** hashed is everything copied verbatim from `src/assets` — the D-DIN fonts — plus `favicon.ico`, `index.html`, `VERSION` and `MANIFEST.txt`. Those keep their names across builds, so they get a short `Cache-Control` and a fixed invalidation list (`/index.html /favicon.ico /VERSION /MANIFEST.txt /assets/*` — five paths against the 1,000-path/month free tier; a wildcard counts as one). *(An earlier version of this bullet said "only `index.html`" — true for the bundles, wrong for `assets/`, and the template it pointed at would have marked the fonts immutable for a year.)*

### Frontend deploy script (for GitLab CI and for operators)

- [x] **Written: `ui/package/deploy_ui.sh`** ✅ 2026-09-17 — beside its producer `ui/package/build_ui_release.sh` (Phase 2.3) rather than at a new `deploy/` root, because it consumes exactly what that script stages (`ui/release/stage/`, `index.html` at the root) and the two are one pipeline. **Not yet run against AWS** — the first real run needs the AWS CLI installed with the `knottyyoga-ci-deploy` key configured, and the Distribution ID (still not recorded in this doc — the `E…` value; note it beside `dv1tgxa9ok30f.cloudfront.net` above).
	```bash
	./ui/package/build_ui_release.sh                                   # producer: ng build → ui/release/stage
	DISTRIBUTION_ID=E1234567890ABC ./ui/package/deploy_ui.sh           # consumer: upload + invalidate
	DISTRIBUTION_ID=E1234567890ABC DRY_RUN=1 ./ui/package/deploy_ui.sh # prints every aws command, runs none
	```
	Env: `DISTRIBUTION_ID` (required, validated as `E…` so the cloudfront.net domain is refused with a message), `BUCKET` (default `knottyyoga-ui-prod`), `SOURCE_DIR` or first argument (default the staged tree), `PRUNE=1`, `WAIT=1`, `DRY_RUN=1`. From Git Bash on Windows set `MSYS_NO_PATHCONV=1` or `/index.html` gets rewritten into a Windows path before the aws CLI sees it.
	- **Three passes, in an order that makes the deploy atomic for the browser:** (1) hashed `*.js`/`*.css`/`media/*` as `immutable, max-age=1y`; (2) everything unhashed except `index.html` as `max-age=86400`; (3) `index.html` last, `max-age=0, must-revalidate`. Until (3) lands every request still resolves against the previous build, whose files are all still present.
	- **No `--delete`.** The template above had it, first — which deletes the previous build's chunks while browsers holding the previous `index.html` can still lazy-load them, turning a routine deploy into mid-session 404s. `PRUNE=1` deletes objects absent from the current build as an explicit list diff (not `sync --delete`, which would also re-upload without the cache headers). Run it as part of a *later* deploy, once the previous build has been live long enough that nobody has its `index.html` open; with deploys days apart, "the next deploy" is fine.
	- **Content types are stated, not guessed,** for `.js`/`.css`/`index.html`. The aws CLI infers them from Python's `mimetypes`, which on Windows reads the registry, and a stray editor install can map `.js` to `text/plain`. `Managed-SecurityHeadersPolicy` sends `X-Content-Type-Options: nosniff`, under which a script served as `text/plain` is **refused** — the symptom is a blank page with console errors, on a deploy that reported success. Fonts and images are not subject to nosniff, so their guessed types are fine.
	- Verified by dry run against a fake staged tree (pass order, command shapes, and the three precondition failures: missing ID, domain-instead-of-ID, raw `dist/` instead of the staged tree).
- [ ] **First real run — step by step.** Two scripts, both bash, both run from **Git Bash** (Start menu → *Git Bash*; it is what ships with Git for Windows and it inherits Node and the aws CLI from the Windows PATH). Not PowerShell — the scripts will not run there. "Producer" and "consumer" just mean: the first script *produces* the built site on disk, the second *consumes* that directory and pushes it to AWS. You run one, then the other.
	1. **Install the AWS CLI v2** (the `aws` command; it is not installed on this machine yet). Either, in PowerShell: `winget install --id Amazon.AWSCLI`, or download and run the MSI from https://awscli.amazonaws.com/AWSCLIV2.msi. **Close and reopen every terminal afterwards** — the PATH change is not picked up by open windows. Verify in a new Git Bash: `aws --version` → `aws-cli/2.x.x …`.
	2. **Create the deploy key.** It does not exist yet — it is the still-unchecked **"Create the `ci-deploy` IAM user"** step at the top of this section (4.6, just after the bucket). Do that step now, in full: IAM user `ci-deploy` (no console access) → policy `knottyyoga-ci-deploy` from the JSON above (now includes `GetInvalidation`) → attach → **Security credentials → Create access key → "Application running outside AWS"**. You get an **Access key ID** (`AKIA…`, not secret) and a **Secret access key** (shown once). Save both to the password manager under *AWS Secrets*. GitLab CI variables can wait until Phase 6 — the key is what matters now.
	3. **Give the CLI that key, as a named profile.** In Git Bash:
		```bash
		aws configure --profile knottyyoga-deploy
		```
		It asks four questions: *AWS Access Key ID* (paste), *AWS Secret Access Key* (paste), *Default region name* → `us-west-2`, *Default output format* → `json`. This writes `C:\Users\mason\.aws\credentials` and `.aws\config`. A named profile rather than the default so it cannot collide with any other AWS identity on the machine, and so a script run without the profile fails with "no credentials" rather than silently deploying as someone else. Then, in the same Git Bash window:
		```bash
		export AWS_PROFILE=knottyyoga-deploy
		aws sts get-caller-identity        # → "Arn": "arn:aws:iam::957014951609:user/ci-deploy"
		aws s3 ls s3://knottyyoga-ui-prod/ # empty output (bucket is empty) and NO AccessDenied — proves the policy attached
		```
		`export` lasts for that window only; re-run it in each new Git Bash, or the script stops with the "no credentials" error.
	4. **Find and record the Distribution ID.** CloudFront console → *Distributions* → the row whose *Domain name* is `dv1tgxa9ok30f.cloudfront.net` → its **ID** column, `E` followed by 13 letters/digits. Write it here, beside the domain at line ~820, and keep it handy — it is the one value the deploy script cannot look up for you.
	5. **Run the producer** (build the site). In Git Bash, from the repo root:
		```bash
		cd /c/Users/mason/source/repos/knottyyoga
		./ui/package/build_ui_release.sh
		```
		It runs `npm ci` (reinstalls `node_modules` from the lockfile — a minute or two) and `ng build --configuration=production` (another minute or two), then stages the result at `ui/release/stage/` with `index.html` at its root and writes `ui/release/knottyyoga-ui-<git-sha>.tar.gz`. The last lines say `wrote …tar.gz` and its size. `ui/release/` is gitignored (added 9/17). The Angular budget warnings it prints are pre-existing and not a failure. **Verified on this machine in Git Bash, 9/17** — it needed one fix (the cleanup trap failed on Windows because Git Bash's `ln -s` copies a directory instead of linking it, and the script reported exit 1 after doing all its work). A `-dirty` suffix on the version means uncommitted changes in the tree. **You can skip this script if you prefer** — the consumer accepts any directory with `index.html` at its root, so `cd ui && npx ng build --configuration=production` followed by pointing step 6 at `ui/dist/ui/browser` also works; you just lose the `VERSION` file and the tarball to keep for rollback.
	6. **Run the consumer, dry first** (upload + invalidate). Same window (so `AWS_PROFILE` is still set):
		```bash
		export DISTRIBUTION_ID=E23TY4IAUHGM6H
		DRY_RUN=1 ./ui/package/deploy_ui.sh
		```
		(An earlier draft of this step said to `export MSYS_NO_PATHCONV=1` first. **Do not** — that was wrong, and it produced `The user-provided path /c/Users/… does not exist` on the first real run: it stops Git Bash mangling `/index.html`, but it equally stops Git Bash translating `/c/Users/…` into the `C:\Users\…` that `aws.exe` needs. The script now handles both halves itself — conversion off, local paths handed over via `cygpath -m` — verified 9/17 against the real CLI with the `ci-deploy` key.)
		It prints every `aws` command it *would* run and runs none. Read them: three `s3 sync`/`cp` passes into `s3://knottyyoga-ui-prod/`, then one `create-invalidation` on your distribution. If that looks right:
		```bash
		WAIT=1 ./ui/package/deploy_ui.sh
		```
		`WAIT=1` blocks until CloudFront reports the invalidation complete (1–3 minutes) so the check in step 7 is meaningful the moment the script returns. Expect `event=deploy_done version=<git-sha>` as the last line but one.
	7. **Check.** In a browser: `https://dv1tgxa9ok30f.cloudfront.net/VERSION` shows the git sha the producer printed; `https://dv1tgxa9ok30f.cloudfront.net/` shows the site. If `/VERSION` is right but `/` is an S3 XML error, the **Default root object** (post-creation settings above) is not set. If everything 403s, the OAC bucket policy step above did not take.
	8. **Every later deploy** is steps 5–6 again (three commands: `export`s, producer, consumer) — no AWS console work. Add `PRUNE=1` to the consumer every few deploys to clear the previous builds' chunks from the bucket.
- [x] **First run done — the frontend is live at `https://dv1tgxa9ok30f.cloudfront.net/`** ✅ 2026-09-17, version `7831bb51-dirty`. Verified from the outside: `/VERSION` reads the sha; `main-*.js` and `styles-*.css` arrive as `application/javascript` / `text/css` with `immutable, max-age=1y`; `index.html` as `text/html, max-age=0, must-revalidate`; fonts one day; `/calendar` returns 200 + `index.html` (the SPA fallback works); `nosniff` and HSTS present on everything (the security-headers policy is attached). Two findings from that run:
	- ⚠️ **The `knottyyoga-ci-deploy` policy is missing `cloudfront:GetInvalidation`** — it was created from the JSON before that action was added above, so `WAIT=1` failed with AccessDenied *after* the invalidation had been created (the deploy itself was complete). Fix once: IAM → **Policies** → `knottyyoga-ci-deploy` → **Edit** → JSON tab → replace `"Action": "cloudfront:CreateInvalidation"` with `"Action": ["cloudfront:CreateInvalidation", "cloudfront:GetInvalidation"]` → **Next** → **Save changes**. Takes effect within seconds; no new access key needed.
	- **`/api/health` returns 504 — expected.** CloudFront's API origin is wired (a wrong `X-Origin-Secret` would be a 403, not a 504); the 504 is an origin timeout because **nothing is listening on the EC2's port 80 yet**. That is Phase 5.1, the server's first deploy. Nothing in 4.6 is wrong.
	- *(Script polish from the run: `VERSION` and `MANIFEST.txt` are now uploaded as `text/plain` — extensionless, so the CLI guessed `binary/octet-stream` and a browser downloaded `/VERSION` instead of showing it. Takes effect on the next deploy.)*
	- **The upload log showed `assets/styles/*.scss` going to the bucket** — the app's SCSS sources, because `angular.json` listed `src/assets` wholesale and the shared stylesheets live under it (they are the Sass `includePaths` root, so they cannot move). Not harmful, but source on a public URL for no reason. Fixed 9/17: the `assets` entry in both the `build` and `test` targets is now the glob form with `ignore: ["styles/**", ".gitkeep"]`; a production build confirms `assets/` is just `fonts/` + `svg/`, and Karma is unaffected. The 21 files already in the bucket disappear on the next `PRUNE=1` deploy — safe to prune immediately in this one case, since no browser ever referenced them.
- [x] **Document in `RUNBOOK.md`:** frontend-only deploys (`deploy_ui.sh`) run independently of backend deploys — no EC2 work needed. ✅ 9/17 — **`RUNBOOK.md` now exists at the repo root** (the plan said "in this repo" at 3.5, and an operator document belongs with the code and must carry no secrets — unlike this vault, which holds the `server.env` block verbatim). §2 is the frontend deploy, prune and rollback. It also carries the other five things this plan promised it: the origin-secret rotation (1.7), the DR outline (3.5, marked unverified until the PITR drill), the secret bootstrap order (4.8, corrected — `--seed-secrets-from-file` does not exist, `set_secret` is the mechanism), the test-helper safe/unsafe command split (5.2), and Session Manager on/offboarding (5.2, marked not yet set up). Sections whose system half is undeployed are marked ⏳ rather than written as if live.

## 4.7 Email via SES

SES has two trip wires: **(1) regional** — you verify the domain and request prod access *per region*, and **(2) sandbox mode** — until you request production access, SES will only deliver to addresses you've explicitly verified. Don't skip the production-access request.

- [ ] **Verify the sending domain.**
	- Region: **us-west-2** (pick one region and stick with it — the `config_secrets` SMTP host is region-specific).
	- Top search → **Amazon Simple Email Service** → SES console.
	- Left sidebar → **Configuration → Identities** → **Create identity**. *(The console renamed this in 2026; it used to be a top-level "Verified identities" entry. The other new sidebar groups — "Pricing plan", "Email validation", "Mail Manager", "Virtual Deliverability Manager" — are separate products or plans; nothing in this plan needs them. New accounts sit on the Essentials pricing plan by default, which is fine for the soft launch.)*
	- The **Create identity** page, field by field in the order it shows them (2026 console):
	- **Identity details**
		- **Identity type:** **Domain**. Not "Email address" — a verified domain covers every address under it (`noreply@`, `info@`, whatever `kMailSenderAddress` ends up being); a verified address covers one.
		- **Domain:** `knottyyoga.com` — the bare apex. No `www.`, no `mail.`; subdomains inherit from the apex.
		- **Assign a default configuration set:** **leave unchecked.** Configuration sets are the hook for publishing send/bounce/complaint *events* to SNS or CloudWatch. Bounce handling for v1 is SES's account-level suppression list, which is on by default and needs no set. A set can be attached to the identity later without re-verifying.
		- **Assign to a tenant:** **leave unchecked.** SES "tenants" are its own multi-sender reputation-isolation feature — nothing to do with this app's tenancy model. One studio, one identity.
		- **Use a custom MAIL FROM domain:** **leave unchecked** (as before). Without it the envelope sender is `amazonses.com`, so SPF alignment fails DMARC but DKIM alignment passes, and one aligned method is all DMARC needs. Add `mail.knottyyoga.com` later (one MX + one TXT record) if a receiver ever complains.
	- **Verifying your domain**
		- The blue box says it: the hosted zone is in Route 53 in this account, so SES will write the DNS records itself. Nothing to do here except expand **Advanced DKIM settings** and confirm the three choices below.
		- **Identity type (under Advanced DKIM settings):** **Easy DKIM**. The radio is NOT preselected on this page — pick it, or the form will not submit. Not "Deterministic Easy DKIM" (for cloning an identity into a second region using a parent region's keys — one region here) and not BYODKIM (your own key pair; nothing gained).
		- **DKIM signatures:** **Enabled** (the default — leave it; the page itself says disabling is not recommended).
		- **DKIM signing key length:** `RSA_2048_BIT`.
		- **Publish DNS records to Route53:** **checked** (default when the zone is in this account — verify it is; this is what auto-creates the three `CNAME`s).
	- **Tags:** skip.
	- **Create identity**.
	- Wait ~5 minutes; the identity's **Verification status** flips to **Verified** and **DKIM status** to **Successful**. If it stays pending >10 min, double-check the Route 53 CNAMEs were actually created (Route 53 → Hosted zones → `knottyyoga.com` → look for three `*._domainkey.knottyyoga.com` records).
- [x] **Request production access (sandbox → production).** Until you do this, SES will only deliver to addresses you've added to **Identities** — useless for real users. ✅ 2026-09-17
	- SES console → left sidebar → **Get set up** (the top item; the Account dashboard's sandbox banner also links there via *View Get set up page* — the dashboard itself no longer carries the button). It is a three-step checklist:
		1. **Add an email address** — verify `masonbendixen@gmail.com` as an *Email address* identity (Identities → Create identity → Email address → click the link SES emails you). Needed even with the domain verified: it is the address SES sends its confirmations to and the one address the sandbox can deliver to, and step 3 stays greyed out without it.
		2. **Add a sending domain** — already satisfied by the `knottyyoga.com` identity above.
		3. **Request production access** — the button lives here.
	- Fill in the form:
		- Mail type: **Transactional** (account verifications, payment receipts, password resets — *not* marketing)
		- Website URL: `https://knottyyoga.com`
		- Use case description — be specific. Example: *"Transactional email for a yoga studio web app: account-verification emails on signup, payment receipts after class purchases, and password reset emails. Estimated volume under 100/day initially. We will monitor bounces and complaints via SNS topics on the verified identity and immediately suppress problem addresses."*
		- How you'll handle bounces/complaints: mention SNS notifications + automated suppression
		- Additional contacts: leave default
	- **Submit**. AWS typically responds within 24 hours; on approval, your daily sending quota jumps from 200 → 50,000.
- [x] **Create SMTP credentials.** ✅ 2026-09-18
	- SES console → left sidebar → **SMTP settings**.
	- **Choose credential method: `IAM SMTP credentials`** — the right-hand option, NOT the "Recommended" one. *Mail Manager SMTP* routes sends through a Mail Manager **ingress endpoint** (that is what the `ingressendpoint-2026…` name field is for) with traffic policies and rule sets — it is the paid add-on that has been appearing all over the SES sidebar, and its own fine print says "Mail Manager processing charges apply". IAM SMTP credentials is the classic path: one IAM user with a send-only policy and an SMTP password derived from its secret key. Nothing beyond the per-send cost.
	- **This page no longer shows the endpoint or ports at all** (with IAM SMTP credentials selected it is just two buttons, *Manage existing* and *Create IAM credentials*). They are fixed per region, so nothing needs looking up: endpoint **`email-smtp.us-west-2.amazonaws.com`**; **TLS Wrapper ports 465 / 2465** (what `mail_server_method = login` needs); STARTTLS ports 25 / 587 / 2587 (what `tls` would need). The port and the method are a matched pair — table below.
	- **Create IAM credentials** (orange, ↗ — opens the IAM console in a new tab, titled *Create user for SMTP*). Pre-filled:
		- **User name:** a generated `ses-smtp-user.2026…` → **rename to `ses-smtp-knottyyoga`**.
		- **Permissions:** an inline policy `AmazonSesSendingAccess` granting `ses:SendRawEmail` — leave it; that is the send-only grant.
		- **Create user**.
	- The confirmation page shows the **SMTP user name** (an IAM-derived `AKIA…` string) and, behind *Show*, the **SMTP password** — **the only time it is displayed.** Click **Download .csv file** and save both to the password manager. The password is derived from the IAM secret key by SES's signing algorithm; it is not the secret key itself, and the page hands you the derived value, so never try to substitute a regular IAM secret. Back on SMTP settings, *Manage existing* now lists the user.
	- Nothing goes into `config_secrets` today — that is the "load" step below, run once the server is up in 5.1. Keep the `.csv`.
- **Reference — the values to load into `config_secrets`. Not a Phase 4 step: it is executed as 5.1 step 6**, once `--migrate` has created the table. Kept here because it is the SES knowledge; the checkbox lives in 5.1. The **real key names** — an earlier draft used names (`kMailHost`, `kMailUser`…) that do not exist. (Table sits outside the list on purpose: Obsidian does not render a table nested in a list item.)

| `config_secrets` key | Value for SES | Note |
|---|---|---|
| `mail_server_name` | `email-smtp.us-west-2.amazonaws.com` | |
| `mail_server_port` | `465` | **Pair with `mail_server_method = login`.** In `mail_helper.cpp`, `login` means implicit TLS from the first byte (`mailio::smtps` + `LOGIN`), which is SES's *TLS Wrapper* port 465; `tls` means STARTTLS, which is 587. `587` + `login` — what this doc used to say — fails the TLS handshake. 465/`login` needs no method change from the Gmail default; 587/`tls` is the equivalent alternative if you prefer AWS's documented port. |
| `mail_server_method` | `login` | (default — leave) |
| `mail_smtp_username` | the SES SMTP username (`AKIA…`) | **New key, 9/18 (honuware).** The helper used to log in with the *sender address* as the SMTP username, which is what Gmail wants and what SES cannot accept — so SES could not have worked at all. Empty (the default) keeps the old behaviour; set it and SES authenticates the IAM user while the From stays the studio's address. |
| `mail_app_password` | the SES SMTP password | the same row the Gmail app password occupied in dev |
| `mail_sender_address` | `noreply@knottyyoga.com` | **Must be an address under the verified domain identity** — SES refuses `554 Message rejected: Email address is not verified` for a From it has not verified, and the seeded default is the gmail address. Any local part works once the domain is verified; nothing needs to exist at that mailbox for sending. |
| `mail_sender_name` | (keep) | |

- **Smoke test → 5.1 step 9.** Once the rows are loaded and the server is running: `knottyyoga_test_helper --command=send_test_email` (real mail on) to an address you own, then the registration path. While still in the SES sandbox the recipient must be a verified **Identity** — the gmail address from the *Get set up* step qualifies.

## 4.8 Secret bootstrap ordering

> **Design note only — there is nothing to do here. The steps live in 5.1 (9/18).** This section used to hold the ordered list of first-deploy operations; that made it look like Phase 4 work, and it never was — none of it can run until the Docker image is on the EC2, which is 5.1's first two boxes. The list moved there verbatim (as 5.1 steps 3–8) and this section keeps only the *why*.

**Why the order is fixed.** `MailHelper`, `SquareClient` and `ServerConfig` all read `config_secrets` — but that table does not exist until the schema is installed, that step cannot run without `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` in `server.env`, the rows it writes are encrypted under `HONUWARE_SECRET_KEY` (so that key must be in the file *before* it runs — rows encrypted under the dev fallback key are unreadable under a real key added later), and the scheduler helper cannot log in until the server is up and the `people` row from that step exists. Hence: env file complete → install schema → `set_secret` rows → server → helper.

⚠️ **The schema step is `--install_schema`, not `--migrate`** (corrected 9/22, and `--install_schema` added the same day). Migrations evolve an existing schema and fail on an empty database; `--recreate_database` cannot run against RDS at all. The full reasoning, and the end-to-end verification, are in 5.1 step 4. `--migrate` is correct for every deploy *after* the first.

**What "the database" and "where" mean here, since the questions came up.** The database is the **production PostgreSQL on RDS** (`knottyyoga.cjise0agyhh6.us-west-2.rds.amazonaws.com`, created in 4.4 on 5/15 and empty — zero tables — ever since); never the local Docker Postgres, which is dev only. Every command runs **on the EC2 over SSH** (`ssh -i ~/.ssh/knottyyoga-ec2.pem ubuntu@34.215.204.200`, a terminal, not the AWS web console), because the `knottyyoga-db` security group admits only the EC2, and each command is a binary inside the Docker image: `sudo docker run --rm -v /etc/knottyyoga:/etc/knottyyoga:ro --env-file /etc/knottyyoga/server.env --entrypoint <binary> knottyyoga:<tag> <args…>` — both the mount and `--entrypoint` are required; see 5.1 step 4.

**"Provision DB; create app user", spelled out** (both done in 4.4): *provision* = the RDS instance, a managed PostgreSQL server with one superuser login (`postgres`); *create app user* = from the EC2, `psql` as `postgres` and `CREATE ROLE knottyyoga LOGIN PASSWORD '…'`, `GRANT knottyyoga TO postgres`, `CREATE DATABASE knottyyoga OWNER knottyyoga` — the app's own role and its own empty database, so the server never runs as the superuser. To see the state it is in today, from the EC2: `PGPASSWORD='<app password>' psql "host=knottyyoga.cjise0agyhh6.us-west-2.rds.amazonaws.com user=knottyyoga dbname=knottyyoga sslmode=verify-full sslrootcert=/etc/knottyyoga/rds-ca.pem" -c '\dt'` → `Did not find any relations.` — a connection that works *and* an empty database is exactly right before 5.1 step 4.

- [x] Document this sequence in `RUNBOOK.md` ✅ 9/17, §4 (corrected there and in 5.1: `--seed-secrets-from-file` was never built, `set_secret` is the mechanism).
- ~~Add the `--seed-secrets-from-file` subcommand~~ — dropped 9/18. `set_secret` is one call per row and there are about eight rows, once; a file-ingest subcommand plus a test is more code than the problem.

---

# Phase 5 — Initial Deploy

## 5.1 Manual first deploy

Purposely manual — gets you comfortable with the pieces before automating.

> **This is the one executable sequence, in order.** It absorbs two things Phase 4 described but could not run: the SES "load into `config_secrets`" step from 4.7 (step 6 here) and the whole of 4.8 (steps 3–8). Those sections are now references; the checkboxes are here. Restructured 9/18 — the earlier version referenced a `deploy/install.sh` that does not exist (the deploy script is Phase 7.4; `package/systemd/README.md` is the manual procedure this list follows).

**On your machine** (Docker Desktop; the build git-clones the pinned honuware, so it needs network):
- [x] **1. Build the image.** From the repo root in Git Bash or PowerShell: ✅ 2026-09-22
	```
	docker build -t knottyyoga:v1.0.2 --build-arg KNOTTYYOGA_VERSION=v1.0.2 -f server/knottyyoga_server/package/Dockerfile server/knottyyoga_server
	```
	First build compiles every dependency and takes ~20 min; later builds reuse the layer cache unless the source changed. The tag is what `version.env` will name.
	> ⚠️ **The release image had never been RUN before 9/22, and the first attempt found three bugs in it.** All three are fixed; the tag moved v1.0.0 → v1.0.2 as each was corrected. Worth recording, because each failed in a way that pointed somewhere other than its cause:
	> 1. **Runtime base too old.** The builder `gcc:14.2.0` is Debian bookworm (**glibc 2.36**); the runtime stage was `ubuntu:22.04` (**glibc 2.35**). *Every* binary died at startup — `version 'GLIBC_2.36' not found`, naming both the executable and the bundled `libstdc++`. Fixed: the runtime is now `debian:bookworm-slim`, the exact release the builder derives from. Re-check this if the builder image is ever bumped.
	> 2. **Helpers need `--entrypoint`.** `ENTRYPOINT` is `knottyyoga_the_server`, so `… knottyyoga:<tag> knottyyoga_database_helper --install_schema` runs **the server** with the helper's name as an argument. The tell is an error naming `knottyyoga_the_server` when you asked for a helper. The Dockerfile documented the broken form in its header and the correct one 100 lines below; both now agree.
	> 3. **No seed artwork shipped.** `build_linux_release.sh` staged `bin/`, `lib/`, `certs/`, `VERSION` and `systemd/` — never `src/database_helper/img/`. Dev builds get it from a CMake POST_BUILD copy, so nobody noticed. A first deploy would have seeded a database with **no images at all** — no hero, no class photos, no tier icons, no instructor portraits, neither Figma SVG — and `AttachSeedPhoto` only *warns*, so the deploy would have reported success and the site would simply look bare. Fixed: the script stages `img/` next to the binaries (where `SeedImageDirectory()` looks, which also means the Dockerfile needed no change) and now fails the build if it is missing or empty.
- [x] **2. Get it onto the EC2.** No ECR yet, so the file route. ⚠️ **These are bash commands — run them in Git Bash, not PowerShell.** PowerShell has no `gzip`, and worse, its `>` is `Out-File`, which applies *text* encoding to binary and silently corrupts the archive — corruption that only surfaces later as a confusing `docker load` failure on the EC2. (Docker's "cowardly refusing to save to a terminal" is what stops the naive PowerShell version from producing a broken file at all.) ✅ 2026-09-22
	```bash
	cd /c/Users/mason/source/repos/knottyyoga
	docker save knottyyoga:v1.0.2 | gzip > knottyyoga-v1.0.2.tar.gz     # 176 MB -> 63 MB
	scp -i ~/.ssh/knottyyoga-ec2.pem knottyyoga-v1.0.2.tar.gz ubuntu@34.215.204.200:~
	```
	**In PowerShell**, use `-o` so docker writes the file itself and no shell redirection is involved (uncompressed, 176 MB — fine to send as-is):
	```powershell
	docker save knottyyoga:v1.0.2 -o knottyyoga-v1.0.2.tar
	scp -i $HOME\.ssh\knottyyoga-ec2.pem knottyyoga-v1.0.2.tar ubuntu@34.215.204.200:~
	```
	`docker load` accepts either form. Both sizes verified 9/22; the tarball lands in the repo root, which is gitignored for it.

**On the EC2** (`ssh -i ~/.ssh/knottyyoga-ec2.pem ubuntu@34.215.204.200`):
- [x] **3. Load the image and finish `server.env`.** ✅ 2026-09-22
	```bash
	sudo docker load < ~/knottyyoga-v1.0.2.tar.gz
	sudo docker images knottyyoga            # v1.0.2 listed
	```
	Then the one gap in the file written in 4.4 — `HONUWARE_SECRET_KEY`. ⚠️ **It must be URL-safe, unpadded base64. `openssl rand -base64 32` alone does NOT work**: the decoder is libsodium's `sodium_base64_VARIANT_URLSAFE_NO_PADDING`, which wants `-`/`_` instead of `+`/`/` and rejects the trailing `=`. A standard key is refused with *"HONUWARE_SECRET_KEY is set but not valid base64"* — a message that reads like your key is malformed when it is perfectly good base64, just the wrong flavour. Verified against the real binary on 9/22. Generate it this way:
	```bash
	openssl rand -base64 32 | tr '+/' '-_' | tr -d '='      # -> 43 chars, no '='
	```
	Save it to the password manager, then append `HONUWARE_SECRET_KEY=<value>` to `/etc/knottyyoga/server.env`. **Before step 4, not after** — rows written under the dev fallback key cannot be read under a real key added later. (The origin secret and scheduler password a few steps above are opaque strings, never base64-decoded, so plain `openssl rand -base64 32` is right for those.)
- [x] **4. Create the schema** — `--install_schema`, **not** `--migrate`: ✅ 2026-09-23
	```bash
	sudo docker run --rm \
	    -v /etc/knottyyoga:/etc/knottyyoga:ro \
	    --env-file /etc/knottyyoga/server.env \
	    --entrypoint knottyyoga_database_helper \
	    knottyyoga:v1.0.2 --install_schema
	```
	⚠️ **`-v /etc/knottyyoga:/etc/knottyyoga:ro` is required too.** `--env-file` is read by the docker *client on the host*, so the variables arrive without any mount — but a variable whose **value is a path** is resolved **inside** the container. `server.env` sets `HONUWARE_DB_SSLROOTCERT=/etc/knottyyoga/rds-ca.pem`, so without the mount the connection dies with `root certificate file "/etc/knottyyoga/rds-ca.pem" does not exist`. (Hit on the first real run, 9/22.) Read-only, and the directory rather than the single file, so rotating the CA bundle stays a host-side `curl`.
	⚠️ **`--entrypoint` is required.** The image's `ENTRYPOINT` is `knottyyoga_the_server`, so naming a helper *after* the image passes its name as an argument to the server and runs the wrong binary. The tell is an error mentioning `knottyyoga_the_server` when you asked for a helper. (This doc had the wrong form until 9/22.)
	Creates every table in the empty `knottyyoga` database from 4.4, seeds `config_secrets` with the non-secret defaults, and provisions `scheduler@knottyyoga.local` from `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` (fails fast if that is unset).
	> ⚠️ **This step said `--migrate` until 9/22, and that does not work on a first deploy.** Verified by running it against a brand-new empty database: it fails immediately with `ERROR: relation "schema_migrations" does not exist`, exit 1. Migrations *evolve* a schema — every one of the ten is a guarded ALTER/INSERT against tables that must already exist — so there has to be a schema first. `--migrate` is the right command for **every deploy after this one**.
	>
	> **`--recreate_database` is not the answer either**, for three independent reasons, each sufficient on its own: it issues `DROP`/`CREATE DATABASE`, and the app's `knottyyoga` role has no `CREATEDB` privilege; it opens that connection with no dbname, which libpq resolves to a database named after the *user* — and since the RDS role and database share the name, the session is already inside the database it then tries to drop (`cannot drop the currently open database`); and a database recreated by the `postgres` master would be owned by the master, leaving the app's role unable to read its own tables. It is the local-dev command, where the role is a superuser and the connection lands in a different database.
	>
	> **`--install_schema` was added 9/22** to fill that gap — the managed-database first-deploy path (RDS, Cloud SQL). It never issues `DROP`/`CREATE DATABASE`, so it needs no `CREATEDB` and preserves the ownership set up in 4.4, and because it runs as the app's own role every object ends up owned by the role that will use it. It refuses a database that already has tables (pointing you at `--migrate`) unless `--force`, which additionally requires `HONUWARE_ALLOW_DESTRUCTIVE=1`.
	>
	> Verified end to end against a real empty PostgreSQL (9/22): installs **118 tables** — the same count as the dev database — with 5 people, 6 classes, 83 `config_secrets` rows and the scheduler account; a subsequent **`--migrate` then applies all 10 migrations cleanly** (`applied=10 skipped=0`), which is what makes step 4 and every later deploy compose; re-running `--install_schema` is refused; `--force` without the destructive guard is refused; `--force` with it succeeds.
- [x] **5. Install the systemd units.** ⚠️ **The unit files are not on the EC2 yet** — they are not in the Docker image (which carries only `VERSION bin certs lib`), and the `knottyyoga-v1.0.2.tar.gz` you shipped in step 2 is the *image*, whose contents are layer blobs. They live in the repo and only ever reach a host via the release tarball (`build_linux_release.sh`, which this path does not build) or a direct copy. **From your workstation**, repo root: ✅ 2026-09-23
	```bash
	scp -i ~/.ssh/knottyyoga-ec2.pem \
	    server/knottyyoga_server/package/systemd/knottyyoga-server.service \
	    server/knottyyoga_server/package/systemd/knottyyoga-helper.service \
	    server/knottyyoga_server/package/systemd/version.env.example \
	    ubuntu@34.215.204.200:~
	```
	Then on the EC2, **from `~`** (the `cp` commands in the README take bare file names, so the directory matters), follow first-time install steps **1 through 4** in `server/knottyyoga_server/package/systemd/README.md`: copy the two `.service` files to `/etc/systemd/system/`, write `/etc/knottyyoga/version.env` containing `KNOTTYYOGA_IMAGE_TAG=v1.0.2` (the shipped `.example` says `v1.0.0-sandbox.1` — a placeholder; leave it and both units look for an image that does not exist), **`chmod 600` both files (step 3 — do not skip it: `version.env` is world-readable until you do, and `server.env` holds the database password and the at-rest key)**, then `sudo systemctl daemon-reload`. Skip only its steps 5 and 5b — the schema and the secrets are this plan's steps 4 and 6.
	> **If you copied the units before 9/23, re-copy them now.** Both gained `-v /etc/knottyyoga:/etc/knottyyoga:ro` that day; without it the server and helper fail at startup on the RDS CA path. `daemon-reload` after.
- [ ] **6. Set the secrets that ship empty.** Seven rows. **Fill in the three values only you have, then paste the whole block once** — no re-typing the `docker run` per row. On the EC2:
	```bash
	# --- the three from your password manager ------------------------------
	SES_USER='AKIA…'                     # SES SMTP username, from the ses-smtp-knottyyoga .csv (4.7)
	SES_PASS='…'                         # SES SMTP password, same .csv
	SQUARE_TOKEN='…'                     # Square SANDBOX access token
	# -----------------------------------------------------------------------
	IMG=knottyyoga:v1.0.2
	set_secret() {
	  sudo docker run --rm \
	      -v /etc/knottyyoga:/etc/knottyyoga:ro \
	      --env-file /etc/knottyyoga/server.env \
	      --entrypoint knottyyoga_test_helper "$IMG" \
	      --nosend_real_email --command=set_secret --key="$1" --value="$2"
	}

	set_secret mail_server_name    email-smtp.us-west-2.amazonaws.com
	set_secret mail_server_port    465
	set_secret mail_smtp_username  "$SES_USER"
	set_secret mail_app_password   "$SES_PASS"
	set_secret mail_sender_address noreply@knottyyoga.com
	set_secret square_access_token "$SQUARE_TOKEN"
	set_secret square_environment  sandbox

	unset SES_USER SES_PASS SQUARE_TOKEN   # keep them out of the rest of the session
	```
	Each call prints `Secret '<name>' set to '<value>'.`; seven successes and you are done. Tested end to end on 9/22, including a password containing `+`, `/`, `=` and a space — the quoting above holds. The five mail values come from 4.7's table, the two Square ones from §1.5.
	**History note:** those three assignments land in `~/.bash_history`. Either prefix each with a space (bash's default `HISTCONTROL=ignorespace` then skips them) or `history -d` afterwards.
	**Leave `production_mode_on` and `website_address` for step 9** — prod mode pins CORS and cookies to `knottyyoga.com`, which the `cloudfront.net` URL cannot satisfy, so flipping it now makes the site impossible to test.
- [x] **7. Start the server.** `sudo systemctl enable --now knottyyoga-server`, then `curl -sS http://localhost/api/health` → `{"db":"ok","status":"ok","version":"…"}`. From your machine, `https://dv1tgxa9ok30f.cloudfront.net/api/health` → the 504 from 4.6 becomes a 200. If it is a **403**, the `X-Origin-Secret` on the CloudFront origin does not match `server.env`. ✅ 2026-09-24
	> ⚠️ **Step 6 is a hard prerequisite, not an ordering preference.** Verified 9/22 by booting the real image: with `mail_app_password` empty the server **aborts at startup** — `terminate called after throwing an instance of 'std::runtime_error'`, because `MakeMailHelper` is composed during boot and throws on the empty password. The message names the secret, so it is diagnosable, but the process is gone. Set the secrets first.
	> **`PORT=80` in `server.env` is load-bearing too.** The server defaults to **18080** inside the container; both units publish `-p 80:80`. With `PORT` absent the server listens on 18080, docker forwards 80, and every request times out with nothing in the log to explain it. The 4.4 block sets it — leave it there.
	> **`version` in the health response:** images up to and including v1.0.3 answer `"version":"unknown"`, because the build tag reached `/opt/knottyyoga/VERSION` but was never turned into the `HONUWARE_VERSION` env var the endpoint reads. Fixed in the Dockerfile on 9/22 (takes effect on the next image). Until then, add `HONUWARE_VERSION=v1.0.3` to `server.env` if you want the endpoint to identify the build — it is the whole reason that field exists.
- [x] **8. Start the helper.** `sudo systemctl enable --now knottyyoga-helper`, then `sudo journalctl -u knottyyoga-helper -n 50 --no-pager` — expect `[api_client] event=login_success email=scheduler@knottyyoga.local status=200 cookies=1` then `[scheduler] event=event_loop_starting`. `event=login_failure` means the env-var password does not match the hash in the `people` row — most likely the env var changed after step 4; `RUNBOOK.md` §5 has the reset. ✅ 2026-09-24
- [x] **9. Smoke test through the real front door.** On `https://dv1tgxa9ok30f.cloudfront.net/`: register a user (the verification email proves SES end to end — while still in the SES sandbox the recipient must be a verified identity, so use the gmail address), log in, process a sandbox Square payment. Then, once the `us-east-1` cert and the alias records are in place (4.5's go-live step), `set_secret` `website_address` = `knottyyoga.com` and `production_mode_on` = `true`, restart the server, and repeat the smoke test on `https://knottyyoga.com`. ✅ 2026-09-24
- [x] **10. PITR drill** — `RUNBOOK.md` **§6a**, run once now that there is data worth restoring. Deferred from 4.4. ✅ 2026-09-25
	> **§6 was one procedure until 9/24 and it was the wrong one to point at.** It described a *failover*: restore to a new instance, then repoint `server.env` at it and restart. Run as a drill that takes production off its real database. Split into **§6a, the drill** — restore to a separate instance, read it directly with `psql` from the EC2, compare the counts against the live database, delete the copy; nothing live is touched — and **§6b, the real failover**, unchanged. Use 6a here.
	- Run 9/25/26 

## 5.2 SSH access hardening

Two access paths: raw SSH for you (simpler local tooling) and AWS Systems Manager Session Manager for additional operators (no key juggling, IAM-controlled, full audit trail).

### Your own SSH (primary)

- [x] Disable password auth in `/etc/ssh/sshd_config` (`PasswordAuthentication no`). ✅ 2026-09-25
- [x] Use key-based auth only; your public key in `ubuntu`'s `~/.ssh/authorized_keys`. Lock the SG inbound 22 rule to your home IP. ✅ **both were already true — clarified 9/25.** *The key:* selecting a key pair at launch made AWS inject its **public** half into `/home/ubuntu/.ssh/authorized_keys` at first boot; the `.pem` is the **private** half, which is why `ssh -i` works at all. Verify: `ssh-keygen -lf ~/.ssh/authorized_keys`. *The SG:* Phase 4.2 created `knottyyoga-web` (`sg-0accf95c33945db08`) with SSH 22 → **My IP** on 5/14 — a single `/32`, still matching, since SSH works today. Confirm it reads `x.x.x.x/32` and not `0.0.0.0/0`.
	- [x] **Add a second key**, so a lost or corrupted `.pem` is not a permanent lockout. Generate it on the **Windows machine in Git Bash** (which has `ssh-keygen`, `ssh` and `scp`); only the **`.pub`** ever leaves the laptop. ✅ 2026-09-25
		1. **Generate.** Creates `knottyyoga-backup` (private) and `knottyyoga-backup.pub` (public):
			```bash
			ssh-keygen -t ed25519 -C "mason-backup" -f ~/.ssh/knottyyoga-backup
			```
			Set a passphrase at the prompt. It is a break-glass key you will rarely type, so the cost is near zero and the file alone stops being enough to get in.
		2. **Append the public half**, authenticating with the key you already have. ⚠️ **One line — paste it whole.** The trailing `'…'` is what makes `ssh` *run a command* rather than open a shell; drop it (easy to do if a `\`-continued version is pasted a line at a time) and ssh feeds your public key to the remote bash as a command, which answers `-bash: line 1: ssh-ed25519: command not found`. Nothing is appended and nothing is harmed, but it looks alarming.
			```bash
			cat ~/.ssh/knottyyoga-backup.pub | ssh -i ~/.ssh/knottyyoga-ec2.pem ubuntu@34.215.204.200 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
			```
			The `scp` equivalent is the same thing in two steps — `scp` copies, it cannot append — and leaves a stray copy in `~` unless you clean it up:
			```bash
			scp -i ~/.ssh/knottyyoga-ec2.pem ~/.ssh/knottyyoga-backup.pub ubuntu@34.215.204.200:~
			ssh -i ~/.ssh/knottyyoga-ec2.pem ubuntu@34.215.204.200 \
			  'cat ~/knottyyoga-backup.pub >> ~/.ssh/authorized_keys && rm ~/knottyyoga-backup.pub'
			```
		3. **Verify in a NEW Git Bash window, leaving the current session open** — that open session is what lets you fix a mistake instead of being locked out:
			```bash
			ssh -i ~/.ssh/knottyyoga-backup ubuntu@34.215.204.200 'echo backup key works'
			```
			Then confirm both keys are trusted — expect two lines, the AWS key and `ED25519 … mason-backup`:
			```bash
			ssh -i ~/.ssh/knottyyoga-ec2.pem ubuntu@34.215.204.200 'ssh-keygen -lf ~/.ssh/authorized_keys'
			```
		4. **Store BOTH private keys off this laptop** — password managers take file attachments. The two files, and note neither has a `.pub`: `~/.ssh/knottyyoga-backup` (the one just generated — no extension) and `~/.ssh/knottyyoga-ec2.pem`. The `.pub` needs no backup: it is not secret and `ssh-keygen -y -f <private>` regenerates it.
			⚠️ **This is the step that decides whether the exercise was worth anything.** Both keys sitting on one disk protects against a corrupted file but not against losing the machine — the likelier failure, and the one the second key exists for. And `knottyyoga-ec2.pem` is the more urgent of the two: it is the key that works today, and per the Phase 4.3 step that created it, **AWS does not keep a copy** — lose it with no second key trusted and SSH access is gone for good, leaving only console-based recovery.
	- ⚠️ **When your ISP changes your IP, SSH stops working with no error that explains it** — the packets are just dropped, so it looks like the host is down. Fix, and note that **`knottyyoga-web` is a SECURITY GROUP, not an instance** — it will not appear under *Instances*, which lists only `knottyyoga-server` (the instance it is attached to):
		- *From the instance:* EC2 → **Instances** → `knottyyoga-server` → **Security** tab → click the `sg-0accf95c33945db08 (knottyyoga-web)` link.
		- *Or directly:* EC2 → left sidebar → **Network & Security → Security groups** → `knottyyoga-web`.
		- Then **Inbound rules** tab → **Edit inbound rules** → the `SSH` / port `22` row → **Source** → **My IP** (AWS fills in your current address as a `/32`) → **Save rules**. Immediate; nothing restarts. The two rules should read SSH 22 from `x.x.x.x/32` and HTTP 80 from `0.0.0.0/0`.
		- The console is always reachable, so this is never a true lockout — but it is exactly the annoyance Session Manager below removes.
- [x] Add a `RUNBOOK.md` section describing how to run `knottyyoga_test_helper` via SSH — which commands are safe in prod, which ones aren't. ✅ 9/17, §7: every registered command sorted into read-only / deliberate-write / never (fabricates state or runs a scheduler job by hand), plus the two defaults that bite — it auto-logs-in as Mason, and **`--send_real_email` is ON by default**, so prod runs pass `--nosend_real_email`.

### Session Manager (for additional operators, e.g., your retired friend)

- [x] **Give the instance an AWS identity, then check the agent.** Two things, expanded 9/25 because the one-line version assumed the vocabulary. ✅ 2026-09-25
	- *Why an "instance profile":* Session Manager works by the **instance** calling the Systems Manager service, so the instance needs AWS permissions of its own — it has none today. An IAM **role** holds permissions; an **instance profile** is the wrapper that lets a role attach to an EC2 instance rather than to a person. The console creates the profile implicitly, so you only ever pick the role. `AmazonSSMManagedInstanceCore` is AWS-maintained and contains exactly what SSM needs; you do not author it.
	- **Create the role:** IAM → **Roles** → *Create role* → trusted entity **AWS service** → use case **EC2** → Next → tick **`AmazonSSMManagedInstanceCore`** → Next → name `knottyyoga-ec2-ssm` → *Create role*.
	- **Attach it:** EC2 → **Instances** → `knottyyoga-server` → **Actions → Security → Modify IAM role** → `knottyyoga-ec2-ssm` → *Update IAM role*. No restart; live within a minute or two.
	- **The agent:** preinstalled on Ubuntu 24.04 AMIs, but **as a snap, not a deb** — so `systemctl status amazon-ssm-agent` finds nothing and it looks absent. On the EC2:
		```bash
		snap list amazon-ssm-agent
		sudo snap start --enable amazon-ssm-agent
		systemctl status snap.amazon-ssm-agent.amazon-ssm-agent --no-pager | head -5
		```
		Usually already running, in which case this just confirms it. Restart it (`sudo snap restart amazon-ssm-agent`) **after** attaching the role — it caches credential failures.
	- **Verify in the console, no CLI needed:** Systems Manager → **Fleet Manager** (or Session Manager → *Start session*). The instance appears as a managed node within a couple of minutes; *Start session* gives a browser shell with no key and no port 22. If it never appears, the role is not attached or the agent is not running.
	- ⚠️ **The `aws ssm start-session` check in the next bullet needs different credentials than your deploy profile.** `knottyyoga-deploy` is the `ci-deploy` user — S3 and CloudFront only, no `ssm:StartSession`. That verification needs a key for your own `masonbendixen` user; the console route above avoids the question.
	- **Worth doing even though you are solo:** this is the way out of the port-22 lockout described in *Your own SSH* above. Session Manager is outbound-only from the instance, so a changed home IP cannot shut it off.
- [x] **Verify in the console, not the CLI:** Systems Manager → **Session Manager** → *Start session* → `knottyyoga-server` → **Start session**. A browser shell on the EC2 with no SSH key and no port 22 — which is the whole claim being tested. ✅ 2026-09-25
	> ⚠️ **`aws ssm start-session --target i-03dcc463764ac0d19` fails three ways in a row** on this machine, and fixing each only reveals the next (all three confirmed 9/25):
	> 1. `NoRegion` — there is exactly one local profile, `knottyyoga-deploy`, and no `default`. Without `--profile`/`AWS_PROFILE` the CLI falls back to `default`, which does not exist.
	> 2. `AccessDeniedException` on `ssm:StartSession` — `knottyyoga-deploy` is the `ci-deploy` user: S3 and CloudFront only.
	> 3. `SessionManagerPlugin is not found` — the command needs a **separate** Session Manager plugin, not part of the AWS CLI and not installed here.
	>
	> To make the CLI route work later: `winget install Amazon.SessionManagerPlugin` (reopen the terminal), create an access key for the `masonbendixen` user, `aws configure --profile masonbendixen` with region `us-west-2`, then `aws ssm start-session --target i-03dcc463764ac0d19 --profile masonbendixen`. It verifies nothing the console does not.
- [ ] Create an IAM user for each additional operator (e.g., `friend-of-mason`). Attach a policy that grants `ssm:StartSession` on this specific instance ARN, plus `ssm:TerminateSession` and `ssm:DescribeSessions` for their own sessions. Then either give them console access (Systems Manager → Session Manager → *Start session*, nothing to install) or, for the CLI, they install the **Session Manager plugin** locally, create their own access key, and run `aws ssm start-session --target i-03dcc463764ac0d19 --profile <theirs>`. The plugin is a separate download from the AWS CLI — worth saying up front, since its absence surfaces only at the moment they try to connect.
- [x] Document the onboarding/offboarding procedure in `RUNBOOK.md`: granting a new operator is "create IAM user + attach policy", revoking is "delete the IAM user". No rebooting, no editing files on the EC2. ✅ **9/25 — `RUNBOOK.md` §8 written in full.** Onboard: IAM user (with console access, so they need nothing installed) + a `knottyyoga-operator-ssm` policy whose JSON is in the runbook — scoped to **this instance's ARN only**, with `${aws:username}-*` on the session ARN so an operator can end their own sessions and nobody else's, and the three describe calls at `"*"` because they take no resource-level permissions. Offboard: delete the user, or detach the policy to suspend reversibly; open sessions die with the credentials. Audit: CloudTrail logs `StartSession`/`TerminateSession` per principal with no host-side setup, and full keystroke transcripts to S3/CloudWatch are a preference worth enabling once more than one person has access. Also recorded: a Session Manager shell is `ssm-user` **with sudo**, so it is as powerful as SSH and §7's safe/unsafe command list applies.
	- ⚠️ The section is marked **unverified** — written from the AWS docs, not from an onboarding actually performed, and the policy is the part most likely to need a tweak on first use. The prerequisite (the `knottyyoga-ec2-ssm` role on the instance + the snap agent) is the step above and is what makes any of it work.
- [x] **Audit trail — the automatic half needs nothing.** ✅ 9/25. CloudTrail already records every session against its IAM principal; to read it, **CloudTrail → Event history → filter Event name = `StartSession`** (also `TerminateSession`) — principal, time, source IP and target instance are all in the row. While you are the only operator that is the entire audit question, since every session is yours. Written up as `RUNBOOK.md` §8 *Audit*.
- [ ] **Keystroke transcripts — deliberately deferred.** Worth it when the second operator actually exists, not before: a transcript answers *what they ran*, which CloudTrail cannot, and that only matters once somebody else is on the box. Ten minutes then, no harder than now. Full procedure in `RUNBOOK.md` §8, including the trap that decides whether it works: **`AmazonSSMManagedInstanceCore` does not grant the instance role permission to write to the log destination**, so without an extra inline policy sessions start normally and silently never log — a failure indistinguishable from success. Also noted there: leave *Encrypt log data* off unless you are creating a KMS key, because ticked it is mandatory and sessions fail to start without one.

### Why no shared SSH keys

Adding more public keys to `authorized_keys` works but has bad ergonomics: rotating one user's key means editing files on every EC2 you ever build, no audit trail, you have to remember who has what. Session Manager + per-user IAM scales without that mess.

## 5.3 Observability + watchdog replacement

This is the section that replaces the custom watchdog-of-watchdogs from `Scheduled Jobs.md`. AWS-native primitives cover the same job with less code.

### Logs

- [ ] **Ship the two units' journals to CloudWatch Logs.** ⚠️ **Not with "the CloudWatch Logs agent" — that tool cannot do this** (established 9/25). The legacy `awslogs` agent is deprecated, and its replacement, the **unified CloudWatch agent**, has inputs for *files* and *Windows events* only — **no journald input**, so it cannot be pointed at `journalctl` at all. Two things that do work:
	- **fluent-bit — the recommended path.** Native `systemd` input that filters by unit, `cloudwatch_logs` output, single binary, and if it dies it simply stops shipping rather than affecting the app.
		1. **IAM first** — same trap as the SSM session transcripts: `AmazonSSMManagedInstanceCore` does **not** grant log writes. **This is the policy document** — the JSON editor opens holding a skeleton template, so select all, delete, and paste this over it:
			```json
			{
			  "Version": "2012-10-17",
			  "Statement": [
			    {
			      "Effect": "Allow",
			      "Action": [
			        "logs:CreateLogStream",
			        "logs:PutLogEvents",
			        "logs:DescribeLogStreams"
			      ],
			      "Resource": "arn:aws:logs:us-west-2:957014951609:log-group:/knottyyoga/ec2:*"
			    }
			  ]
			}
			```
			It lets the instance create streams and write events **only inside `/knottyyoga/ec2`**, not to any other log group in the account. Account id and region are already filled in.
			**Getting to that editor — "Create inline policy" is buried in a dropdown:** IAM → **Roles** (not Policies — "inline" exists only on a principal, so the Policies section never offers it) → `knottyyoga-ec2-ssm` → **Permissions** tab → **Add permissions** button → **Create inline policy** → switch **Visual** to **JSON** → paste the above → Next → name it `knottyyoga-app-logs` → Create policy.
			*Equivalent and easier to find:* IAM → Policies → Create policy → JSON → paste the same document → name it, then Roles → `knottyyoga-ec2-ssm` → Add permissions → **Attach policies** → select it. A managed policy is reusable and listed under Policies; an inline one dies with the role. Either works here.
		2. **Create the log group.** A *log group* is CloudWatch's container for logs — a folder; the *streams* inside it are the files, and fluent-bit creates one per systemd unit. CloudWatch console → left sidebar **Logs → Log Management** → **Create log group** → **Log group name** `/knottyyoga/ec2` (must match `log_group_name` in the fluent-bit config exactly) → **Retention setting: 1 month** → Create.
			- ⚠️ **The sidebar was reorganised — there is no "Log groups" entry any more.** Under *Logs* you now get **Log Management**, *Log Analytics* and *Log Anomalies*; the log-groups list lives inside **Log Management**. Or skip the navigation entirely with this deep link, which is stable however the sidebar is arranged: `https://us-west-2.console.aws.amazon.com/cloudwatch/home?region=us-west-2#logsV2:log-groups`
			- *Retention* = how long CloudWatch keeps the data before deleting it. The default is **Never expire**, i.e. paying storage on every line forever.
			- **Create it by hand rather than letting fluent-bit do it** — that is why the config sets `auto_create_group false`. An auto-created group inherits *Never expire*, and nothing tells you until the bill drifts.
			- Cost: nothing. The free tier includes 5 GB/month of ingestion, and two units on a low-traffic studio site produce single-digit megabytes — three orders of magnitude under it.
		3. **Install fluent-bit — on the EC2**, over SSH as `ubuntu`. It is the agent that reads the journal and ships it, so it runs on the machine producing the logs; nothing here touches the Windows box. (Steps 1–2 above are console work from the laptop; steps 3–5 are all on the EC2.)
			```bash
			ssh -i ~/.ssh/knottyyoga-ec2.pem ubuntu@34.215.204.200
			curl -fsSL https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh
			```
			That script adds fluent-bit's official apt repository and installs from it. To avoid piping a remote script to a shell, do the same by hand — add their GPG key and repo per fluent-bit's Ubuntu install docs, then `sudo apt-get install fluent-bit`.
		4. **Write the config** — on the EC2. Back up the default, then write the file in one command rather than editing it (the `<<'EOF'` … `EOF` pair means "everything between is the file content"; the closing `EOF` must sit alone at the start of its line):
			```bash
			sudo cp /etc/fluent-bit/fluent-bit.conf /etc/fluent-bit/fluent-bit.conf.orig

			sudo tee /etc/fluent-bit/fluent-bit.conf > /dev/null <<'EOF'
			[SERVICE]
			    Flush        5
			    Daemon       Off
			    Log_Level    info

			[INPUT]
			    Name            systemd
			    Tag             ky.*
			    Systemd_Filter  _SYSTEMD_UNIT=knottyyoga-server.service
			    Systemd_Filter  _SYSTEMD_UNIT=knottyyoga-helper.service
			    Read_From_Tail  On

			[OUTPUT]
			    Name               cloudwatch_logs
			    Match              ky.*
			    region             us-west-2
			    log_group_name     /knottyyoga/ec2
			    log_stream_prefix  journal-
			    auto_create_group  false
			EOF

			cat /etc/fluent-bit/fluent-bit.conf
			```
			- **`[SERVICE]`** — global settings; `Flush 5` batches every five seconds.
			- **`[INPUT]`** — `Name systemd` reads the journal. The two `Systemd_Filter` lines restrict it to these units, so ssh logins and cron noise are not shipped. `Read_From_Tail On` starts from now instead of replaying the entire journal history on first start.
			- **`[OUTPUT]`** — `Match ky.*` takes everything the input tagged; the rest names the log group from step 2.
			- **The `*` in `Tag ky.*` is load-bearing.** fluent-bit substitutes the unit name for it, so records arrive tagged `ky.knottyyoga-server.service` / `ky.knottyyoga-helper.service`; with `log_stream_prefix` that yields **one CloudWatch stream per unit**. Write `Tag ky` without the `*` and both services interleave into a single stream — functional, but much harder to read.
		5. `sudo systemctl enable --now fluent-bit`, restart a unit to generate lines, confirm two streams appear. Nothing showing → `sudo journalctl -u fluent-bit -n 30`; an IAM failure surfaces there as AccessDenied from the output plugin.
	- **Docker's `awslogs` log driver** on both units is the fewer-moving-parts alternative, but **a container refuses to start if the driver cannot reach CloudWatch** — that puts logging in the critical path of the server booting, which is a bad trade for a production API. Noted and rejected.
	- **Why bother, with one instance:** not convenience — `journalctl` over SSH is fine day to day — but that **journal logs die with the instance**. Replace or lose the EC2 and every log explaining why goes with it. Reasonable to defer until after the soft launch settles; the reason it exists is the day you need it most.
- [ ] Set CloudWatch Logs retention to **30 days** (1 month) on `/knottyyoga/ec2` — and on `/knottyyoga/ssm-sessions` if the §8 session transcripts get enabled. The default is *Never expire*, which quietly accrues storage charges forever.
- [ ] Cap journald to **500 MB** total disk via `/etc/systemd/journald.conf` (`SystemMaxUse=500M`) so a chatty service can't fill `/var/log`.
- [ ] (Optional) Enable CloudFront access logs → a dedicated S3 bucket. Free aside from S3 storage; skip until you actually want HTTP-level visibility.

### Health-check + alarming

- [ ] Create an SNS topic `knottyyoga-alerts` and subscribe your email to it.
- [ ] CloudWatch alarm on **EC2 instance status check** — alarms when AWS itself thinks the VM is unhealthy. Action: notify SNS topic.
- [ ] CloudWatch alarm on **EC2 system status check** — alarms on underlying-host issues (rare). Action: notify SNS topic.
- [ ] CloudWatch alarm on **disk-free percentage < 20%** (requires CloudWatch Agent reporting disk metrics). Action: notify SNS topic.
- [ ] **CloudWatch Synthetics canary** hitting `https://<your CloudFront domain>/api/health` every 5 minutes. Alarms after 2 consecutive failures. ~$0.0012/run = ~$10/mo for 5-minute interval. (Or skip Synthetics and use UptimeRobot's free tier — 5-minute interval, free for up to 50 monitors. Same coverage.)

### Process resiliency

- [ ] systemd unit's `Restart=on-failure` covers process-level crashes (planned in Phase 2.2).
- [ ] **No custom watchdog process needed**. The custom `knottyyoga_helper` watchdog mode from `Scheduled Jobs.md` is dropped from scope. `knottyyoga_helper` retains only the scheduled-jobs runner (subscription billing, reminders).

### What this stack catches vs. misses

| Failure | Detected by | Time to detect |
|---|---|---|
| Crow process crash | systemd `Restart=on-failure` | <5s |
| Crow process hung but not crashed | Synthetics canary | <10 min |
| EC2 VM hung / kernel panic | EC2 instance status check | <2 min |
| EC2 host-hardware failure | EC2 system status check + auto-recovery | <2 min |
| Disk full | CloudWatch alarm | <2 min |
| RDS down | App's own DB exception → 503 → Synthetics fails | <10 min |
| AZ outage | Synthetics fails; manual rebuild needed (single-AZ design) | minutes; resolution = hours |

For a soft launch, that coverage is plenty. Multi-AZ EC2 / RDS is a Phase 8 upgrade if real users start depending on uptime.

---

# Phase 6 — GitLab CI/CD

You asked whether you can run backend tests that need Postgres in GitLab CI. **Yes** — GitLab "services" let you spin up a Postgres sidecar per job. Works well.

## 6.1 Pipeline skeleton

- [ ] Commit `.gitlab-ci.yml` at repo root with stages: `build`, `test`, `package`, `deploy-manual`.
- [ ] Use a pinned custom builder image that has GCC 12.4, Conan 2, CMake 3.24+, libpqxx-dev, and Postgres client. Publish this image to GitLab Container Registry so builds are fast and reproducible.

## 6.2 Backend test job with Postgres sidecar

- [ ] Job `test:backend` uses `services: [postgres:13.1-alpine]` with env vars `POSTGRES_USER=docker POSTGRES_PASSWORD=docker POSTGRES_DB=knottyyoga`.
- [ ] Script: `conan install`, `cmake`, `make`, then `bin/knottyyoga_tests` with env vars pointing at `postgres` as the hostname.
- [ ] The test support already supports running in a transaction that gets rolled back, so no cleanup is needed between tests.
- [ ] Cache `~/.conan2/p` to speed up Conan.

## 6.3 Frontend test + build job

- [ ] Job `test:frontend` runs `npm ci && ng test --watch=false --browsers=ChromeHeadlessCI` and `ng lint`.
- [ ] Job `build:frontend` runs `ng build --configuration=production` and publishes `ui/dist/ui/` as a GitLab artifact.

## 6.4 Package job

- [ ] Job `package` runs on `main` tags, builds release binaries, and uploads the server + UI tarballs as GitLab release artifacts (or S3).

## 6.5 Deploy job

- [ ] Job `deploy-manual` is a manual-trigger job (click Play in GitLab UI) that:
  - SSHs to the EC2 using a deploy key stored in GitLab CI variables.
  - Runs `/opt/knottyyoga/deploy/install.sh <artifact-url>`.
- [ ] Start with **manual** deploys; go auto once you're confident. Auto-deploys on push-to-main for a payments-processing app are risky until CI coverage is strong.

---

# Phase 7 — Versioning & Ongoing Update Workflow

## 7.1 Release convention

- [ ] Decide on semver with prerelease tags: `v1.0.0-sandbox.1`, `v1.0.0-sandbox.2`, ..., then `v1.0.0` when flipping to Square live.
- [ ] One git tag per deployed build. Do not deploy untagged commits.
- [ ] Keep `CHANGELOG.md` updated with one section per tag — at minimum, the list of applied migrations (important!) and any secret/env changes.

## 7.2 Per-release schema changes

- [ ] Every PR that changes `db_schema/` must also add a migration to `migrations/` with the next numeric prefix. Enforce this via a CI check script (`check_migrations.sh`) that fails if `db_schema/` changed and no new `migrations/*.sql` was added.
- [ ] Migration review checklist (adds to this doc): is it additive? is it backfilled? does it run in a transaction? does the code that ships in the same tag work with *both* pre- and post-migration schema?

## 7.3 Branch strategy

You mentioned saving branches per version — I'd do this via tags instead of branches. Branches signal active development; a release snapshot is best expressed as an immutable tag. Use branches only for long-lived back-porting if you need hotfixes on an older release line. For a solo/small-team soft launch, tags are plenty.

## 7.4 Update procedure

- [ ] `git tag -a vX.Y.Z -m "..."` → push tag → CI builds artifacts → Release created in GitLab.
- [ ] Operator clicks `deploy-manual` in GitLab → artifact deploys to EC2.
- [ ] EC2 `install.sh`:
  1. Pulls image: `docker pull <ecr-repo>/knottyyoga:vX.Y.Z`.
  2. Runs migrations: `docker run --rm --env-file /etc/knottyyoga/server.env <image> knottyyoga_database_helper --migrate`. (Idempotent for the scheduler service-account row — second-and-later runs are a no-op.)
  3. Stops the helper first: `systemctl stop knottyyoga-helper`. SIGTERM-clean per Phase 11 of `Scheduled Jobs.md` — graceful shutdown takes <1s.
  4. Stops the server: `docker stop knottyyoga-server`.
  5. Starts the new server: `docker run -d --name knottyyoga-server -p 80:80 --env-file /etc/knottyyoga/server.env <image>`.
  6. Health-check poll on `/api/health`; abort + rollback to previous image tag (both containers) if health fails within 30s.
  7. Starts the new helper: `systemctl start knottyyoga-helper`. Verify in journalctl that it re-authenticates successfully.
  8. Prune old images: `docker image prune -f`.

---

# Phase 8 — Nice-to-haves (post-soft-launch)

Not required to ship; listed so we don't forget.

- [ ] Migrate EC2 to `t4g.small` (ARM Graviton) for ~20% compute savings. Needs an ARM-capable CI builder or cross-build.
- [ ] RDS multi-AZ (doubles RDS cost; buy when a real outage hurts).
- [ ] AWS WAF rules attached to the CloudFront distribution for basic abuse protection (rate limits, common-attack managed rule set, geo-blocking if desired). $5/mo base + $1 per rule + $0.60 per million requests.
- [ ] Separate staging environment (second tiny EC2 + RDS, used for final pre-prod validation).
- [ ] Structured JSON logging — easier to grep CloudWatch.
- [ ] Encrypted secrets-at-rest in the `config_secrets` table (column-level encryption with a key from env var) instead of plaintext. Plaintext is ok for a tiny soft launch but you'll want this before real revenue flows.
- [ ] CloudFront access logs → S3 for HTTP-level visibility (free aside from S3 storage of the log files).
- [ ] Buy the 1-yr Compute Savings Plan once the instance type is confirmed.
- [ ] **Helper liveness alarm**: CloudWatch Logs metric filter on the `knottyyoga-helper` log group looking for `[scheduler] event=job_success` lines, with an alarm if no match in the last 25 hours (longest interval is daily billing). Catches the case where the helper is "running" per systemd but its login keeps failing, so no jobs ever execute. Cheap insurance once we have customer data depending on the billing cycle.

---

# Monthly Cost Estimate (soft launch)

## Per-service cost detail

Prices in us-west-2 (Oregon), April 2026. These are the AWS public list prices — verify against the AWS Pricing Calculator before committing.

### EC2

| Component | Rate | Monthly (soft launch) |
|---|---|---:|
| `t3.small` on-demand | $0.0208/hr | $15.18 (730 hr) |
| `t3.small` 1-yr reserved, no upfront | — | ~$9.50 |
| `t3.small` 3-yr reserved, no upfront | — | ~$6.50 |
| EBS gp3 root, 20 GB | $0.08/GB-mo | $1.60 |
| EBS snapshots (1 weekly) | $0.05/GB-mo | ~$1 |
| Data out to internet (non-CloudFront) | $0.09/GB (first 100 GB free) | ~$0 |
| Data out to CloudFront (same region) | **free** | $0 |
| Elastic IP (attached to running instance) | free | $0 |
| **EC2 subtotal (on-demand)** | | **~$18/mo** |
| **EC2 subtotal (1-yr reserved)** | | **~$12/mo** |

*Bandwidth note*: because `/api/*` traffic flows EC2 → CloudFront → user, AWS bills the EC2 → CloudFront hop at zero. Your EC2 data-out costs are effectively free at soft-launch volume.

### RDS

| Component | Rate | Monthly (soft launch) |
|---|---|---:|
| `db.t3.micro` (1 vCPU / 1 GB) on-demand, single-AZ | $0.018/hr | $13.14 |
| `db.t3.micro` 1-yr reserved, no upfront | — | ~$9.00 |
| `db.t4g.micro` (ARM) on-demand | $0.016/hr | $11.68 |
| Storage, gp3 20 GB | $0.115/GB-mo | $2.30 |
| Automated backups | **free up to DB size** | $0 |
| PITR (point-in-time recovery) | included | $0 |
| Extra manual snapshots | $0.095/GB-mo above DB size | ~$0–$1 |
| Data transfer in | free | $0 |
| Data transfer out (to EC2 in same AZ) | free | $0 |
| Multi-AZ (optional, doubles compute) | — | skip for v1 |
| **RDS subtotal (on-demand, x86)** | | **~$16/mo** |
| **RDS subtotal (1-yr reserved, x86)** | | **~$11/mo** |

### CloudFront

The free tier (first 12 months) is generous enough that CloudFront is effectively free at soft-launch scale.

| Component | Rate (North America) | Monthly (soft launch) |
|---|---|---:|
| Data out to internet | $0.085/GB (first 1 TB/mo free for 12 months) | $0 free-tier, then ~$1–5 |
| HTTPS requests | $0.01 per 10,000 (first 10M/mo free for 12 months) | $0 free-tier, then ~$0.50 |
| Invalidation requests | first 1,000 paths/mo free | $0 |
| Origin Shield (optional caching layer) | $0.0075/10k requests | skip for v1 |
| **CloudFront subtotal (first 12 months)** | | **~$0/mo** |
| **CloudFront subtotal (after free tier)** | | **~$1–5/mo** |

Assumption: soft launch traffic ≈ 5–20 GB/mo and 100k–1M requests/mo. Even scaled to 100 GB and 10M requests you're under $15/mo.

### S3

| Component | Rate | Monthly (soft launch) |
|---|---|---:|
| Storage (Standard class), Angular bundle ≈ 5–10 MB | $0.023/GB-mo | ~$0 |
| PUT/COPY/POST (deploys only) | $0.005 per 1,000 | ~$0 |
| GET (CloudFront reads from S3, mostly cached) | $0.0004 per 1,000 | ~$0 |
| Data out to CloudFront | free | $0 |
| **S3 subtotal** | | **~$0/mo** (literally under $0.10) |

### Total

| Mode | EC2 | RDS | CF | S3 | Other* | **Total** |
|---|---:|---:|---:|---:|---:|---:|
| On-demand, first 12 months | $18 | $16 | $0 | $0 | $2 | **~$36/mo** |
| On-demand, after free tier | $18 | $16 | $3 | $0 | $2 | **~$39/mo** |
| 1-yr reserved, first 12 months | $12 | $11 | $0 | $0 | $2 | **~$25/mo** |
| 1-yr reserved, after free tier | $12 | $11 | $3 | $0 | $2 | **~$28/mo** |

*"Other" = Route 53 hosted zone + queries (~$1), SES (~$0 on AWS egress), CloudWatch Logs (~$0 in free tier), domain registration amortized (~$1).*

Pricing caveat: AWS adjusts prices occasionally; verify current rates in the AWS Pricing Calculator before committing.

---

# Resolved Questions (decisions log)

All previously open questions are answered. Decisions are recorded here so we can trace why the plan looks the way it does, and so future-Mason has the rationale.

- ✅ **Architecture** — EC2 + RDS + S3 + CloudFront, no nginx.
- ✅ **Build target** — x86-64 for v1; migrate to ARM (Graviton) post-launch.
- ✅ **TLS** — ACM + CloudFront, no certbot.
- ✅ **Origin protection** — Crow `CloudFrontOriginGuard` middleware checks `X-Origin-Secret`.
- ✅ **Domain** — `KnottyYoga.com`, currently registered at another DNS provider. Plan: keep the registrar, but stand up a Route 53 hosted zone for DNS so we get apex-alias records to CloudFront. Update the registrar's NS records to point at Route 53. Migrating the registrar to AWS later is optional and trivial. (Phase 4.5 details.)
- ✅ **Region** — `us-west-2` (Oregon) for EC2/RDS/S3. ACM cert for CloudFront is in `us-east-1` regardless (CloudFront-global limitation).
- ✅ **Staging environment** — **No separate staging.** The recommendation: launch in `us-west-2` directly into what will become production, run on the Square *sandbox* with no DNS pointing at it (use the CloudFront distribution's auto-generated `dXXXXXX.cloudfront.net` URL, share that with friend-testers). When you're ready, point `knottyyoga.com` at it via Route 53 and flip `kSquareEnvironment` to `production`. Reasons: doubling the cost and config surface for a one-person project rarely pays back; a friends-and-family sandbox period is its own staging.
  - The day you'd actually want a separate staging environment: when (a) you have paying customers and need to test schema migrations against prod-like data without risk, or (b) more than one developer is shipping in parallel. Neither is true today.
- ✅ **Watchdog / heartbeat — let AWS do most of it.** The custom watchdog-of-watchdogs from `Scheduled Jobs.md` was designed for self-hosted environments. On AWS, simpler primitives cover most of it:
  1. systemd `Restart=on-failure` restarts a crashed process within seconds. (Phase 2.2.)
  2. CloudWatch alarm on the EC2 instance-status check + SNS email tells you if the VM itself is wedged.
  3. CloudWatch Synthetics canary (or a free external uptime probe like UptimeRobot) hits `/api/health` every ~5 min and pages on failure.
  4. Auto-scaling-group-of-one with an instance-replacement policy is overkill for a soft launch but worth knowing exists.
  → **Decision** (already implemented in `Scheduled Jobs.md`): `knottyyoga_helper` is the **scheduled-jobs runner only** — subscription renewals, reminders, voucher expiry, cleanup jobs, waitlist refunds. No watchdog mode. Phase 5.3 covers the CloudWatch alarms + Synthetics canary.
- ✅ **Square credentials** — values come from `secret_values.cpp` (the `production`/`debug` ifdef'd block). Phase 1.4 will pull the sandbox values for `environment.prod.ts` and the production values when you flip live.
- ✅ **Backup testing** — exercise RDS restore once during initial deploy, then quarterly. (Tracked in Phase 5.1 + Phase 8.)
- ✅ **Savings Plan timing** — run on-demand for 2–4 weeks, then buy a **1-yr Compute Savings Plan**. Switching is easy: Compute Savings Plans commit to a $/hr spend, not a specific instance, so changing instance type/family/size/region (e.g., later migrating to ARM `t4g.small`) keeps the discount as long as you stay within the committed hourly burn. The lock-in cost is "you owe AWS this $/hr for 12 months even if you scale down." For RDS the equivalent is a Reserved Instance, which *is* tied to instance family — so RDS RI commitment should wait until you're confident on `db.t3.micro`, OR be skipped (the RDS RI savings on a single small instance are only ~$50/yr; not worth the inflexibility).
- ✅ **Log retention** — journald capped at 500 MB on the EC2; CloudWatch Logs retention 30 days. (Phase 5.3.)
- ✅ **Admin access** — Mason only on day one, but design for granting access to others. **Use AWS Systems Manager Session Manager**, not raw SSH key juggling, for the secondary operator. SSM gives you: no public key on the EC2, AWS-IAM-controlled access (grant/revoke instantly via IAM policy), full audit trail in CloudTrail, no inbound port 22 needed. The retired-friend gets an IAM user + Session Manager permission, runs `aws ssm start-session --target <instance-id>` from their machine, and they're in. Phase 5.2 details.
- ✅ **`db_schema/` snapshots** — git tags only; no directory copies.
- ✅ **Destructive migration safety** — `--recreate_database` blocked in prod unless `KNOTTYYOGA_ALLOW_DESTRUCTIVE=1` env var is set. (Phase 3.3.)
- ✅ **Scheduler service-account password** — single env var `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` in `/etc/knottyyoga/server.env`, read by both `knottyyoga_database_helper` (hashes it into the `people` row) and `knottyyoga_helper` (uses it to log in). The database helper fails fast if the env var isn't set, so production can't accidentally provision the row without a password. Rotation: delete the row in `people`, update the env var, re-run `--migrate`. See `Scheduled Jobs.md` §3.2.

---

# Phase 0 — Decisions checklist (fill before Phase 1 starts)

- [x] Architecture committed — EC2 + RDS + S3 + CloudFront, x86-64, no nginx
- [x] Domain chosen — `KnottyYoga.com` (keep at current registrar; Route 53 hosted zone for DNS only)
- [x] AWS region chosen — `us-west-2` (app); `us-east-1` (ACM cert for CloudFront)
- [x] Square sandbox values confirmed — pull from `secret_values.cpp` ifdef'd `production`/`debug` block
- [ ] SES sender identity agreed (likely `noreply@knottyyoga.com`; needs your call on the local-part)
- [x] Staging env — **no**, soft-launch environment doubles as staging (no DNS, sandbox Square, friends-only)
- [x] `knottyyoga_helper` in-scope for soft launch — scheduled-jobs runner only; **all 11 phases of `Scheduled Jobs.md` complete**; AWS Synthetics + CloudWatch alarms replace the custom watchdog
- [x] Resolved Questions log filled in