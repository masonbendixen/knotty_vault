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
           │    S3    │   │  EC2 t3.small (x86, Ubuntu 22.04)│
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
- [ ] Set a **billing alarm** at $75/mo (sanity) so a misconfigured anything doesn't quietly run up a bill.
- [ ] Region: `us-west-2` (Oregon) for everything except the ACM cert. The ACM cert lives in `us-east-1` (CloudFront-global limitation) — you'll create that explicitly in Phase 4.5.
- [ ] In the AWS console region picker, default to `us-west-2`. When you switch over to ACM in Phase 4.5, remember to flip the region picker to `us-east-1` for that step only.

## 4.2 Networking

- [ ] Use the default VPC. Two public subnets (in different AZs) already exist — need both because RDS subnet groups require a minimum of two AZs even for single-AZ instances.
- [ ] Security groups:
  - `sg-knottyyoga-web` (EC2): allows 22 (SSH from your home IP only) and 80 (from 0.0.0.0/0 — origin protection is enforced in the Crow middleware, not the SG).
  - `sg-knottyyoga-db` (RDS): allows 5432 from `sg-knottyyoga-web` only.

## 4.3 Compute: EC2

- [ ] Create key pair in EC2 console (or import your existing public key).
- [ ] Launch `t3.small` (x86) instance, Ubuntu 22.04 LTS AMI, 20 GB gp3 root volume, in default VPC public subnet.
- [ ] Attach `sg-knottyyoga-web`.
- [ ] Allocate an Elastic IP and attach it. Free while attached. Needed so the CloudFront origin target doesn't change on stop/start.
- [ ] First boot: `apt update && apt upgrade`; install `docker.io` and `postgresql-client`.
- [ ] Docker handles port binding — no `cap_net_bind_service` needed. The container runs as root internally (standard for single-process containers); the EC2 host user doesn't matter.
- [ ] Create `/etc/knottyyoga/server.env` (chmod 600) containing `PORT=80`, `KNOTTYYOGA_ORIGIN_SECRET=<random>`, `KNOTTYYOGA_TRUST_PROXY=1`, the `KNOTTYYOGA_DB_*` vars, `KNOTTYYOGA_DB_SSLROOTCERT=/etc/knottyyoga/rds-ca.pem`, and `SCHEDULER_SERVICE_ACCOUNT_PASSWORD=<random>`. Generate the scheduler password with `openssl rand -base64 32` (or any source ≥24 chars); both `knottyyoga_database_helper` and `knottyyoga_helper` read this var from the same env file, so set it once.
- [ ] Enable `ufw`: deny incoming default, allow 22 + 80.
- [ ] Install CloudWatch Agent if you want metrics beyond basic EC2 ones. Optional for v1 — the systemd journal tailed to CloudWatch Logs is enough.

## 4.4 Database: RDS Postgres

- [ ] Create RDS subnet group spanning the two default-VPC public subnets.
- [ ] Provision `db.t3.micro`, engine Postgres 15, single-AZ, 20 GB gp3 storage, auto-minor-version upgrades on.
- [ ] Enable automated backups with 7-day retention (default). Turn on **deletion protection**.
- [ ] Attach `sg-knottyyoga-db`.
- [ ] Record the RDS endpoint; note it's a DNS name (e.g., `knottyyoga.xxxxxx.us-west-2.rds.amazonaws.com`). Put it in `/etc/knottyyoga/server.env` as `KNOTTYYOGA_DB_HOST`.
- [ ] Create the application database and a non-superuser role:
  ```sql
  CREATE ROLE knottyyoga LOGIN PASSWORD '...';
  CREATE DATABASE knottyyoga OWNER knottyyoga;
  ```
- [ ] Download the AWS RDS global CA bundle to `/etc/knottyyoga/rds-ca.pem`; set `KNOTTYYOGA_DB_SSLMODE=verify-full`. (Phase 1.1 plans the sslmode support.)
- [ ] Verify PITR by running a toy restore as part of Phase 5.1 smoke tests.

## 4.5 DNS + TLS

`knottyyoga.com` is registered at a non-AWS provider. We're keeping the registrar there but moving DNS hosting to Route 53 so CloudFront alias records work cleanly. The registrar just needs its NS records updated.

- [ ] Create a Route 53 hosted zone for `knottyyoga.com` ($0.50/mo). Note the four NS records Route 53 assigns.
- [ ] At your current DNS provider, change the nameservers for `knottyyoga.com` to those four Route 53 NS values. **Do not** delete the registration there; you only swap the NS pointers. Propagation is typically <1 hour but can take up to 48 hours.
- [ ] (Optional, later) Migrate the registrar itself to Route 53. Costs roughly the same as your current registrar, consolidates billing. Can be done at any time without disturbing anything.
- [ ] In ACM **in `us-east-1`** (CloudFront *only* reads certs from `us-east-1` regardless of where your app runs), request a public cert for `knottyyoga.com` and `www.knottyyoga.com` with DNS validation. Route 53 can auto-create the validation CNAMEs — one click in the ACM console.
- [ ] **Do not** create `A` records for the domain yet — keep CloudFront accessible only via its `dXXXXXX.cloudfront.net` URL during the soft-launch / friends-and-family phase. That's how we avoid running production DNS while still letting testers reach the site (paste them the CloudFront URL directly).
- [ ] When you're ready to flip live: in the Route 53 hosted zone, create two `A`-type *alias* records (apex `knottyyoga.com` and `www.knottyyoga.com`) pointing at the CloudFront distribution. Alias records are free of per-query charges.
- [ ] At go-live, also flip the `kSquareEnvironment` secret from `sandbox` to `production` (Phase 4.8 secret bootstrap covers this).

## 4.6 S3 + CloudFront

### S3 bucket for the frontend

- [ ] Create bucket `knottyyoga-ui-prod` in the same region as EC2. Block all public access (CloudFront will reach it via Origin Access Control — more secure than "make bucket public").
- [ ] Enable versioning (cheap insurance if a bad deploy overwrites files).
- [ ] Disable static website hosting on the bucket itself — we don't need it; CloudFront will serve the content.
- [ ] Create IAM user `ci-deploy` with policy allowing `s3:PutObject` + `s3:DeleteObject` + `s3:ListBucket` on this bucket only + `cloudfront:CreateInvalidation` on the distribution. Store its access key in GitLab CI variables.

### CloudFront distribution

- [ ] Create a CloudFront distribution with two behaviors:
  - **Default behavior** (`*`): origin = S3 bucket via **Origin Access Control** (OAC, the modern replacement for OAI). Viewer protocol policy = redirect HTTP→HTTPS. Cache policy = `Managed-CachingOptimized`. Response headers policy = `Managed-SecurityHeadersPolicy`. Compress objects automatically = yes.
  - **API behavior** (`api/*`): origin = EC2 Elastic IP (HTTP, port 80). Viewer protocol = redirect HTTP→HTTPS. Cache policy = `Managed-CachingDisabled`. Origin request policy = `Managed-AllViewerExceptHostHeader` (forwards all cookies, headers, query strings to origin).
- [ ] Alternate domain names (CNAMEs): `knottyyoga.example`, `www.knottyyoga.example`.
- [ ] SSL certificate = the ACM cert created above (must be in us-east-1).
- [ ] Custom error responses: map HTTP 403 and 404 from the S3 origin to `/index.html` with response code 200 — this is what makes Angular's deep-linked routes work on refresh.
- [ ] **Origin protection**: generate a long random string, add a CloudFront "Origin custom header" `X-Origin-Secret: <value>` on the `/api/*` behavior's origin. The Crow middleware from Phase 1.7 rejects anything missing the header with 403. This is the "pragmatic answer" — no nginx needed, no SG-by-IP-prefix churn.
- [ ] After deploy, invalidate `/index.html` (Angular's hashed asset filenames auto-bust cache; only `index.html` needs manual invalidation).

### Frontend deploy script (for GitLab CI and for operators)

- [ ] Script `deploy/deploy-ui.sh`:
  1. `aws s3 sync ui/dist/ui/ s3://knottyyoga-ui-prod/ --delete --cache-control 'public, max-age=31536000, immutable'` for hashed assets.
  2. Override `--cache-control 'public, max-age=0, must-revalidate'` for `index.html` (so the browser always checks for a new one).
  3. `aws cloudfront create-invalidation --distribution-id <ID> --paths /index.html`.
- [ ] Document in `RUNBOOK.md` that frontend-only deploys can happen independently of backend.

## 4.7 Email via SES

- [ ] Verify the sending domain in SES (add a CNAME/TXT in Route 53).
- [ ] Request production access (SES starts in sandbox mode limiting to verified recipients only). This can take a day.
- [ ] Create an SMTP credential pair in SES. Put the SMTP username/password in `config_secrets` via a one-time `knottyyoga_test_helper` run after first deploy.
- [ ] Verify: trigger a test email path (e.g., `person_verify_mail`) and confirm delivery.

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

- [ ] Attach the AWS-managed `AmazonSSMManagedInstanceCore` IAM policy to the EC2's instance profile. Install the `amazon-ssm-agent` package (already preinstalled on Ubuntu 22.04 AMIs, just needs to be `enabled` and `started`).
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