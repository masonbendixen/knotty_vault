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
| `mail_server_port` | `587` (STARTTLS) | Default is `465` (SSL) — SES supports both, 587 is the AWS-recommended path |
| `mail_server_method` | `login` | Default OK (already `login`) |
| `mail_app_password` | SES SMTP password (created in IAM, NOT your console password) | Default is the Gmail app password |
| `Knotty Yoga and Spa` (sender name) | (use default) | Default OK |
| `knottyyogaandspa@gmail.com` (sender address) | `noreply@knottyyoga.com` (or whatever `kMailSenderAddress` is set to) | Defaults to the Gmail address; SES requires the From address match a verified domain identity |

The full list of secrets and their defaults lives in `src/util/secrets/secret_values.cpp`. Phase 4.8 (Secret bootstrap ordering) describes the operator workflow: provision DB → run `database_helper --migrate` to populate the `config_secrets` table from defaults → run `database_helper --seed-secrets-from-file secrets.json` (or `knottyyoga_test_helper`) to override the values above → start the server.

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
- [ ] **Operator wiring** (Phase 4.6): set `KNOTTYYOGA_ORIGIN_SECRET=<random>` in `/etc/knottyyoga/server.env` and the matching `X-Origin-Secret` value as a CloudFront "Origin custom header" on the `/api/*` behavior. Document rotation in `RUNBOOK.md`: generate new random → update CloudFront first → update env file + `systemctl restart knottyyoga-server` → expect ~30s outage during the cut-over (overlap window with two valid headers skipped for v1).

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
- **DB migration** (one-shot at deploy): `docker run --rm --env-file ... knottyyoga:<version> knottyyoga_database_helper --migrate`. Reads `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` from the env file to provision the scheduler service-account row (fails fast if unset).
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
- [ ] Disaster recovery: restore from an RDS snapshot or point-in-time. Write this procedure down in a `RUNBOOK.md` in this repo once Phase 4 is complete.

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
	- LynKL2JHmSpo+u1QJ8q0SX0LhCmVvExb
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
- [ ] **Create the `ci-deploy` IAM user (for GitLab CI to push builds).**
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
		      "Action": "cloudfront:CreateInvalidation",
		      "Resource": "*"
		    }
		  ]
		}
		```
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
- [ ] **Post-creation settings (the redesigned wizard defers all of these — apply them now via the distribution's tabs).**
	- **Distribution → Settings → Edit:**
		- **Default root object: `index.html`** ← **CRITICAL and easy to miss.** The streamlined wizard does NOT set this; without it, the distribution root returns S3 XML/error instead of the Angular app.
		- **Price class: Use only North America and Europe** (cheaper than worldwide; fine for this audience).
		- Supported HTTP versions: ensure **HTTP/2 + HTTP/3**.
		- Standard logging: **off** for v1.
		- Alternate domain names + Custom SSL certificate: **leave at defaults (no CNAME, default CloudFront cert).** Deferred to go-live (Phase 4.5 alias step), and still gated on resolving the **us-east-1 cert** issue (saved ARN at line ~709 is `us-west-2`, which CloudFront cannot use). Attaching a CNAME without a matching us-east-1 cert is blocked anyway.
	- **Distribution → Behaviors → (default) → Edit:** confirm Viewer protocol policy = **Redirect HTTP to HTTPS**, Compress objects automatically = **Yes**, Cache policy = `Managed-CachingOptimized`, Response headers policy = `Managed-SecurityHeadersPolicy`. (The "recommended S3 cache settings" should already match most of this — verify, don't assume.)
- [ ] **Paste the OAC bucket policy into S3.**
	- S3 console → `knottyyoga-ui-prod` → **Permissions** tab → **Bucket policy → Edit** → paste the JSON CloudFront gave you → **Save changes**.
	- Without this step, CloudFront gets `403 Forbidden` from S3 on every request.
- [ ] **Add the API origin for `/api/*`.**
	- CloudFront → your distribution → **Origins** tab → **Create origin**.
	- **Origin domain:** the EC2 Elastic IP (just the bare IP, no `http://`, e.g., `54.123.45.67`)
	- **Protocol:** **HTTP only**
	- **HTTP port:** 80
	- **Add custom header:**
		- Header name: `X-Origin-Secret`
		- Value: the `KNOTTYYOGA_ORIGIN_SECRET` value you put in `/etc/knottyyoga/server.env`. **Must match exactly** — the Crow middleware (Phase 1.7) compares this header on every API request and 403s without it.
	- **Create origin**.
	- Why a header instead of SG-by-IP-prefix? CloudFront's egress IP ranges churn; chasing them in security groups is operational pain. The shared-secret header is the pragmatic answer — no nginx needed.
- [ ] **Add the `/api/*` behavior.**
	- CloudFront → your distribution → **Behaviors** tab → **Create behavior**.
	- **Path pattern:** `api/*`
	- **Origin and origin groups:** the API origin you just created
	- **Viewer protocol policy:** Redirect HTTP to HTTPS
	- **Allowed HTTP methods:** GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE (the "all methods" option)
	- **Cache key and origin requests:**
		- Cache policy: `Managed-CachingDisabled`
		- Origin request policy: `Managed-AllViewerExceptHostHeader` (forwards all cookies, query strings, and most headers — the Host header is stripped so the EC2 sees the right one)
	- **Create behavior**.
- [ ] **Add SPA fallback error responses.** Without this, refreshing on `/calendar` or any deep link returns 403/404 from S3 (the bucket doesn't actually contain `/calendar/index.html`).
	- CloudFront → your distribution → **Error pages** tab → **Create custom error response**.
	- Response 1:
		- HTTP error code: **403: Forbidden**
		- Customize error response: **Yes**
		- Response page path: `/index.html`
		- HTTP response code: **200: OK**
	- Repeat for **404: Not Found**.
- [ ] **Cache invalidation hygiene.** Angular's hashed asset filenames mean only `index.html` needs to be invalidated on each deploy — the rest auto-busts via the URL change. The deploy script below handles this with `cloudfront:CreateInvalidation`.

### Frontend deploy script (for GitLab CI and for operators)

- [ ] **Write `deploy/deploy-ui.sh`** (template — adjust paths to match the Angular 19 build output):
	```bash
	#!/usr/bin/env bash
	set -euo pipefail
	DIST_DIR="ui/dist/ui/browser"           # confirm with `ng build` output
	BUCKET="knottyyoga-ui-prod"
	DISTRIBUTION_ID="EXXXXXXXXXXXXX"        # paste your CF distribution ID

	# Sync hashed assets — cacheable forever
	aws s3 sync "$DIST_DIR/" "s3://$BUCKET/" \
	  --delete \
	  --cache-control 'public, max-age=31536000, immutable' \
	  --exclude 'index.html'

	# Upload index.html with no-cache headers (browser must always recheck)
	aws s3 cp "$DIST_DIR/index.html" "s3://$BUCKET/index.html" \
	  --cache-control 'public, max-age=0, must-revalidate'

	# Invalidate only index.html
	aws cloudfront create-invalidation \
	  --distribution-id "$DISTRIBUTION_ID" \
	  --paths /index.html
	```
- [ ] **Document in `RUNBOOK.md`:** frontend-only deploys (`deploy-ui.sh`) run independently of backend deploys — no EC2 work needed.

## 4.7 Email via SES

SES has two trip wires: **(1) regional** — you verify the domain and request prod access *per region*, and **(2) sandbox mode** — until you request production access, SES will only deliver to addresses you've explicitly verified. Don't skip the production-access request.

- [ ] **Verify the sending domain.**
	- Region: **us-west-2** (pick one region and stick with it — the `config_secrets` SMTP host is region-specific).
	- Top search → **Amazon Simple Email Service** → SES console.
	- Left sidebar → **Verified identities** → **Create identity**.
	- **Identity type:** Domain
	- **Domain:** `knottyyoga.com`
	- **Use a custom MAIL FROM domain:** leave off for now (can add later)
	- **Advanced DKIM settings:** Easy DKIM (default; recommended)
		- DKIM signing key length: `RSA_2048_BIT`
	- **Publish DNS records to Route 53:** **Yes** (auto-creates three `CNAME` records in your hosted zone)
	- **Create identity**.
	- Wait ~5 minutes; the identity's **Verification status** flips to **Verified** and **DKIM status** to **Successful**. If it stays pending >10 min, double-check the Route 53 CNAMEs were actually created (Route 53 → Hosted zones → `knottyyoga.com` → look for three `*._domainkey.knottyyoga.com` records).
- [ ] **Request production access (sandbox → production).** Until you do this, SES will only deliver to addresses you've added to **Verified identities** — useless for real users.
	- SES console → left sidebar → **Account dashboard** → there'll be a banner or **Request production access** button (also under "Account details").
	- Fill in the form:
		- Mail type: **Transactional** (account verifications, payment receipts, password resets — *not* marketing)
		- Website URL: `https://knottyyoga.com`
		- Use case description — be specific. Example: *"Transactional email for a yoga studio web app: account-verification emails on signup, payment receipts after class purchases, and password reset emails. Estimated volume under 100/day initially. We will monitor bounces and complaints via SNS topics on the verified identity and immediately suppress problem addresses."*
		- How you'll handle bounces/complaints: mention SNS notifications + automated suppression
		- Additional contacts: leave default
	- **Submit**. AWS typically responds within 24 hours; on approval, your daily sending quota jumps from 200 → 50,000.
- [ ] **Create SMTP credentials.**
	- SES console → left sidebar → **SMTP settings**.
	- Note the **SMTP endpoint** (e.g., `email-smtp.us-west-2.amazonaws.com`) and ports (587 for STARTTLS, 465 for TLS-wrapped).
	- **Create SMTP credentials** → IAM user name: `ses-smtp-knottyyoga` → **Create user**.
	- This produces an **SMTP username** and **SMTP password** — these look different from regular IAM access keys (the password is derived from the IAM secret key via SES's signing algorithm — don't try to reuse a regular IAM secret here). Save both to your password manager; they aren't shown again.
- [ ] **Load the SMTP credentials into `config_secrets` after first deploy.** Done via a one-time `knottyyoga_test_helper` run in Phase 4.8/5.1 against the running RDS. Required keys:
	- `kMailHost` = `email-smtp.us-west-2.amazonaws.com`
	- `kMailPort` = `587`
	- `kMailUser` = the SES SMTP username
	- `kMailPassword` = the SES SMTP password
	- `kMailFromAddress` = `noreply@knottyyoga.com` (or similar from-address on the verified domain)
- [ ] **Smoke test.** Once secrets are loaded and the server's running, trigger a verification email path (e.g., register a test user) and confirm delivery to a real inbox. If you're still in SES sandbox at this point, the test recipient address has to be added to **Verified identities** first.

## 4.8 Secret bootstrap ordering

Secrets chicken-and-egg: `MailHelper`, `SquareClient`, `ServerConfig` all pull from `config_secrets` — but the DB connection needs to work first.

- [ ] Document this sequence in `RUNBOOK.md`:
  1. Provision DB; create app user.
  2. Write `/etc/knottyyoga/server.env` with `KNOTTYYOGA_DB_*` vars **and** `SCHEDULER_SERVICE_ACCOUNT_PASSWORD`. The migrate step below fails fast if the scheduler password isn't set, so it must be present before step 3.
  3. Run `knottyyoga_database_helper --migrate` (creates schema + `config_secrets` table empty + **provisions the `scheduler@knottyyoga.local` row in `people` with the env-var password hashed in**). The provision step is idempotent — a second run with the same password is a no-op; rotating the password means deleting the row and re-running.
  4. Run `knottyyoga_test_helper` to insert initial secret rows (or write a dedicated `knottyyoga_database_helper --seed-secrets-from-file secrets.json` subcommand — small scope, worth doing).
  5. `systemctl start knottyyoga-server`. Server now boots, loads secrets, configures Square + Mail + CORS.
  6. `systemctl start knottyyoga-helper`. Helper authenticates as the scheduler service account (env-var password matches the hash from step 3), kicks off its timer loop.
- [ ] Add the `--seed-secrets-from-file` subcommand to `database_helper` + a test that validates ingestion.

---

# Phase 5 — Initial Deploy

## 5.1 Manual first deploy

Purposely manual — gets you comfortable with the pieces before automating.

- [ ] Build the Docker image locally: `docker build -t knottyyoga:v1.0.0 -f server/knottyyoga_server/package/Dockerfile server/knottyyoga_server`.
- [ ] Push to ECR (or `docker save | scp | docker load` for the first deploy before ECR is set up).
- [ ] On the EC2, run `deploy/install.sh` which:
  - Runs `docker run --rm --env-file /etc/knottyyoga/server.env knottyyoga:<version> knottyyoga_database_helper --migrate` (creates schema, provisions the scheduler service account from `SCHEDULER_SERVICE_ACCOUNT_PASSWORD`).
  - Updates the version tag in both systemd units and restarts in order: `systemctl restart knottyyoga-server` then `systemctl restart knottyyoga-helper`.
- [ ] Smoke test: `curl https://knottyyoga.example/api/health`.
- [ ] Smoke test the helper: `journalctl -u knottyyoga-helper -n 50` — expect to see `[api_client] event=login_success email=scheduler@knottyyoga.local status=200 cookies=1` shortly after start, then `[scheduler] event=event_loop_starting`. If `event=login_failure` appears instead, the env-var password doesn't match the hash in the `people` row (most likely: env-var was added after the initial `--migrate`, so re-run migrate to update the hash or delete the row first).
- [ ] Log in via the frontend, register a user, process a sandbox Square payment end-to-end.

## 5.2 SSH access hardening

Two access paths: raw SSH for you (simpler local tooling) and AWS Systems Manager Session Manager for additional operators (no key juggling, IAM-controlled, full audit trail).

### Your own SSH (primary)

- [ ] Disable password auth in `/etc/ssh/sshd_config` (`PasswordAuthentication no`).
- [ ] Use key-based auth only; your public key in `ubuntu`'s `~/.ssh/authorized_keys`. Lock the SG inbound 22 rule to your home IP.
- [ ] Add a `RUNBOOK.md` section describing how to run `knottyyoga_test_helper` via SSH — which commands are safe in prod, which ones aren't.

### Session Manager (for additional operators, e.g., your retired friend)

- [ ] Attach the AWS-managed `AmazonSSMManagedInstanceCore` IAM policy to the EC2's instance profile. Install the `amazon-ssm-agent` package (already preinstalled on Ubuntu 24.04 AMIs, just needs to be `enabled` and `started`).
- [ ] Verify by running `aws ssm start-session --target i-xxxxxxxx` from your own machine — you should land in a shell on the EC2 without any SSH key involved.
- [ ] Create an IAM user for each additional operator (e.g., `friend-of-mason`). Attach a policy that grants `ssm:StartSession` on this specific instance ARN, plus `ssm:TerminateSession` and `ssm:DescribeSessions` for their own sessions. They generate their own access keys and `aws ssm start-session --target i-xxxxxxxx`.
- [ ] Document the onboarding/offboarding procedure in `RUNBOOK.md`: granting a new operator is "create IAM user + attach policy", revoking is "delete the IAM user". No rebooting, no editing files on the EC2.
- [ ] Audit trail: SSM session activity is logged in CloudTrail automatically. Optionally, enable session logging to S3 or CloudWatch Logs to capture every keystroke (worth it for prod with multiple operators).

### Why no shared SSH keys

Adding more public keys to `authorized_keys` works but has bad ergonomics: rotating one user's key means editing files on every EC2 you ever build, no audit trail, you have to remember who has what. Session Manager + per-user IAM scales without that mess.

## 5.3 Observability + watchdog replacement

This is the section that replaces the custom watchdog-of-watchdogs from `Scheduled Jobs.md`. AWS-native primitives cover the same job with less code.

### Logs

- [ ] Install the CloudWatch Logs agent on the EC2 (free tier covers 5 GB/mo of ingest). Configure it to tail the systemd journals for `knottyyoga-server.service` and `knottyyoga-helper.service`.
- [ ] Set CloudWatch Logs retention to **30 days** for both log groups.
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