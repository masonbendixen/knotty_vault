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
- **Scheduled jobs**: `knottyyoga_helper` runs under systemd on the same EC2 when it lands (see `Scheduled Jobs.md`). Not blocking for initial deploy.
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
4. **No health endpoint** (needed for the `knottyyoga_helper` watchdog mode, CloudFront health checks, and manual smoke tests).
5. **No CloudFront origin-secret middleware** — Phase 1.7 adds it.
6. **No migration mechanism** — `database_helper` destructively rebuilds the DB, which is fine for dev but will wipe customer data in prod. Must add a forward-only, versioned migration path before the second deploy.
7. **No production build pipeline** — we'll ship native x86-64 Linux binaries from GitLab CI.
8. **No `.gitlab-ci.yml`** — CI with postgres service is feasible in GitLab and we'll wire that up.

---

# Phase 1 — Code & Config Prerequisites (Lowest Layer First)

Goal: make the application configurable per environment and observable enough to run unattended on an EC2 instance fronted by CloudFront. These changes should land before any AWS work.

## 1.1 Parameterize database connection via environment variables

Touches the lowest layer (database access). Everything above depends on the DB, so this is first.

- [ ] Update `server/knottyyoga_server/src/sql_util/database_access/database_helper_init.cpp` to read from env vars with sensible fallbacks to current dev defaults:
  - `KNOTTYYOGA_DB_HOST` (fallback: current platform-dependent value)
  - `KNOTTYYOGA_DB_PORT` (fallback: `5432`)
  - `KNOTTYYOGA_DB_USER` (fallback: `docker`)
  - `KNOTTYYOGA_DB_PASSWORD` (fallback: `docker`)
  - `KNOTTYYOGA_DB_NAME` (fallback: `kDatabaseName`)
  - `KNOTTYYOGA_DB_SSLMODE` (fallback: `prefer`; set to `require` in prod)
- [ ] Update the connection string builder to include `sslmode=<mode>` when set.
- [ ] Add a unit test `database_helper_init_test.cpp` that:
  - Sets env vars via `setenv` / `_putenv_s` and asserts the connection string reflects them.
  - Clears env vars and asserts the defaults.
- [ ] Log (at `LogInfo`) the host/port/db name (NOT the password) at startup so misconfig is obvious in logs.

**Note on RDS & `sslmode`**: RDS PostgreSQL requires either `require` or `verify-full` for production-grade TLS. `verify-full` needs the AWS RDS CA bundle installed in the image. Start with `require` (encrypt, don't verify CN). Good enough for v1.

## 1.2 Add a health-check endpoint

Used by: the `knottyyoga_helper` watchdog (see `Scheduled Jobs.md`), any future load balancer, monitoring.

- [ ] Add `endpoints/health.cpp` / `health.h` with a `GET /api/health` handler returning `{"status":"ok","db":"ok"|"fail","version":"<git-sha>"}`.
  - Runs a trivial `SELECT 1` inside a transaction to validate DB connectivity.
  - Returns 503 if the DB probe throws.
- [ ] Compile-time constant `kBuildVersion` (or read from env var `KNOTTYYOGA_VERSION`) so ops can confirm which build is live.
- [ ] Add `health_test.cpp` — green path and DB-failure path. Follow the `EndpointTestHelper` pattern used by other endpoint tests.
- [ ] Wire into `endpoints/CMakeLists.txt` (both header and cpp).

## 1.3 Logging to stdout for systemd / CloudWatch

- [ ] Inspect `util/logging.h`/`.cpp`. If logs currently go to a file path, make the destination controllable via `KNOTTYYOGA_LOG_DEST` (values: `stdout`, `stderr`, `<file path>`). Default stays as current for dev.
- [ ] Confirm that on Linux the server flushes stdout on each line (systemd journal and CloudWatch Logs agent tail line-by-line).
- [ ] Add tests where practical (e.g., helper that resolves destination from env var).

**Advice**: systemd captures stdout/stderr automatically into the journal — no need for a custom log file path in the container/EC2 deploy. Simpler is better.

## 1.4 Frontend environment configuration

- [ ] Populate `ui/src/environments/environment.prod.ts` with Square **sandbox** Application ID and Location ID for the initial rollout (pulled from the existing `Square credentials and Sandbox setup.md`). These are client-side public identifiers — they're supposed to be in the bundle.
- [ ] Decide: do we want a separate `environment.prod-square-live.ts` configuration for when we flip to Square production? **Recommendation**: yes — create the config but leave commented until we're ready, so the "soft launch" build isn't accidentally using live Square credentials.
- [ ] Add a production build configuration in `ui/angular.json` if one doesn't already exist that maps to `environment.prod.ts`.
- [ ] Ensure the frontend uses relative URLs (`/api/...`) so it works same-origin behind CloudFront. Scan `ServerAccessNetwork.ts` for any hardcoded absolute URLs — if present, make them use a `baseUrl` from environment config.

## 1.5 Cookies + CORS sanity pass for CloudFront same-origin deploy

Currently `ServerConfig::Initialize` reads `kWebsiteAddress` from DB secrets and configures CORS when `prodMode_` is on. With CloudFront serving both the Angular bundle (from S3) and `/api/*` (from EC2) under one distribution domain, the browser sees a single origin → CORS preflight never triggers → cookies flow with plain `SameSite=Lax`.

- [ ] Verify: with CloudFront fronting both behaviors, the browser sees `Origin: https://knottyyoga.example` for both static assets and API. Same-origin → CORS preflight not triggered → cookies flow without `SameSite=None; Secure` gymnastics.
- [ ] Document in `Deploying to AWS.md` (this doc) the secret values that must be set before first boot: `kWebsiteAddress`, `kServerProductionMode=true`, `kSquareAccessToken`, `kSquareEnvironment=sandbox`, plus any email/SES secrets.
- [ ] If any auth code currently assumes the frontend lives at a *different* origin, add a test fixture exercising the same-origin case and the CloudFront-forwarded header handling (`X-Forwarded-Proto`, `X-Forwarded-For`, `CloudFront-Viewer-Address`).

## 1.6 Reverse-proxy awareness in the C++ server

CloudFront forwards the viewer's scheme in `X-Forwarded-Proto: https`, but the TCP connection to Crow is plain HTTP on port 80. Without trusting the forwarded scheme, the `Secure` cookie flag won't be emitted and sessions will silently break on HTTPS.

- [ ] Confirm the server trusts `X-Forwarded-Proto: https` when setting the `Secure` flag on cookies. If today it infers scheme from the request itself (which will be `http` behind CloudFront), cookies set as `Secure` will be dropped by the browser.
- [ ] Add a `KNOTTYYOGA_TRUST_PROXY` flag that, when true, tells the cookie/session code to treat the forwarded scheme as authoritative.
- [ ] Tests for both the trust-proxy-on and trust-proxy-off paths in `cookie_manager_test.cpp` or a new `proxy_trust_test.cpp`.

## 1.7 Origin-secret middleware (replaces nginx)

Since we're dropping nginx, Crow needs to enforce the CloudFront-origin secret itself. This is what stops attackers from hitting the EC2 Elastic IP directly and bypassing the CDN/WAF/cache.

- [ ] Add a `CloudFrontOriginGuard` middleware to `endpoints/middleware/` (or the existing middleware folder if Crow's `App` type params it). On each incoming request:
  1. If the path starts with `/api/health` (or whatever unauthenticated path we pick), pass through — so AWS target groups can probe.
  2. Otherwise require header `X-Origin-Secret: <expected>` where `<expected>` is read from env var `KNOTTYYOGA_ORIGIN_SECRET` at startup.
  3. Missing or mismatched → respond 403 with body `{"error":"direct_origin_access_forbidden"}` and log once per minute (to avoid log-flood on scanners).
- [ ] Log at startup whether the guard is active (`KNOTTYYOGA_ORIGIN_SECRET` set) or disabled (not set — for local dev).
- [ ] Tests:
  - `origin_guard_test.cpp` — verify request with correct header passes, missing header 403s, wrong header 403s, health-check passes regardless, empty env var disables the guard.
- [ ] Wire the secret into `/etc/knottyyoga/server.env` on the EC2 and into CloudFront's "Origin custom headers" config. Document rotation procedure in `RUNBOOK.md` (generate new random, update CloudFront first, update env file + restart systemd unit — short overlap where both values work would require two headers, skip for v1, accept a ~30s outage during rotation).

---

# Phase 2 — Build & Packaging

Goal: produce deployable artifacts repeatably. We have two real options; I'm recommending native binaries + systemd over Docker for v1.

## 2.1 Decide: native binaries vs. Docker

**My recommendation**: build static-ish native Linux binaries and ship them as `.tar.gz` artifacts, run under systemd. Reasons:

1. No existing prod Dockerfile — writing a good multi-stage one for a C++/Conan/libpqxx/Crow/mailio stack is real work.
2. The app is stateless C++ — Docker's main selling points (isolation, fast process restart) matter less here.
3. Simpler CI pipeline.
4. Easy to SSH in, inspect, and run `knottyyoga_test_helper` ad hoc.

The trade-off is slightly less reproducibility across build machines; GitLab CI with a pinned builder image neutralizes that.

**If you'd rather containerize anyway** (reasonable if you want to later go ECS), write a single multi-stage Dockerfile that produces three thin runtime images from one `builder` stage: `knottyyoga-server`, `knottyyoga-db-helper`, `knottyyoga-helper`.

- [ ] Write `server/knottyyoga_server/package/build_linux_release.sh` that runs `conan install`, `cmake -DCMAKE_BUILD_TYPE=Release`, `cmake --build`, and collects:
  - `bin/knottyyoga_the_server`
  - `bin/knottyyoga_database_helper`
  - `bin/knottyyoga_test_helper`
  - `bin/knottyyoga_helper` (once it exists from the Scheduled Jobs plan)
  - Any runtime `.so` dependencies not in base OS (via `ldd` + copy)
  - Certificates / static resources used at runtime, if any
- [ ] Produce a single tarball `knottyyoga-<version>.tar.gz` with a flat layout: `bin/`, `lib/`, `systemd/` (units), `migrations/` (see Phase 3). No `nginx/` — CloudFront replaces it.
- [ ] Target OS/arch: **Ubuntu 22.04 LTS on x86-64** for the initial deploy. Pin this in the build image. Migrate to ARM64 (Graviton, ~20% cheaper) post-launch when the CI builder has an ARM runner or cross-build set up.

## 2.2 systemd units

- [ ] `knottyyoga-server.service` — `ExecStart=/opt/knottyyoga/bin/knottyyoga_the_server`, `EnvironmentFile=/etc/knottyyoga/server.env`, `Restart=on-failure`, `User=knottyyoga`.
- [ ] `knottyyoga-helper.service` — same pattern for the scheduled jobs/watchdog helper (when it exists).
- [ ] **Do not** create a unit for `knottyyoga_test_helper` — it stays manual via SSH.
- [ ] Log lines validating env var wiring (matches 1.1 / 1.3).

## 2.3 Frontend artifact

- [ ] `ng build --configuration=production` in CI produces `ui/dist/ui/`.
- [ ] Zip that up as `knottyyoga-ui-<version>.tar.gz`.
- [ ] Deploy script extracts it to `/opt/knottyyoga/ui/` atomically (extract to a new dir then `mv` the symlink).

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

## 3.2 Introduce a `schema_migrations` version table

- [ ] Add `schema_migrations` table (id TEXT PK, applied_at TIMESTAMPTZ) to `db_schema/`.
- [ ] Add a `MigrationRunner` helper in `business_logic/migration/` that:
  - Takes a list of `{ id, sql_or_cpp_functor }` migrations.
  - In a transaction per migration: check if `id` is in `schema_migrations`; if not, apply, then insert the row.
  - Never goes backward. (Industry standard: migrations are forward-only; roll back with a new forward migration.)
- [ ] Tests: `migration_runner_test.cpp` — applies in order, skips already-applied, aborts cleanly on SQL error, etc.

## 3.3 Split `knottyyoga_database_helper` into two modes

Today `--recreate_database` is destructive. We want two modes:

- [ ] Preserve `--recreate_database` for **dev/test only**. In prod, a safety env var (`KNOTTYYOGA_ALLOW_DESTRUCTIVE=0` by default) blocks it from running. The binary exits with a clear error.
- [ ] Add `--migrate` which:
  1. Connects using the same env vars as the server.
  2. Ensures `schema_migrations` exists (bootstraps it if new DB).
  3. Runs all pending migrations via `MigrationRunner`.
  4. Exits 0 on success.
- [ ] First "baseline" migration (`id = "0001_baseline"`) executes the *existing* schema-creation code path to produce the full current schema on an empty DB.
- [ ] From then on, every schema change is a new migration file with a monotonic id (`0002_add_subscription_tier.sql`, `0003_soft_delete_bookings.cpp`, etc.).
- [ ] Tests: `database_helper_test.cpp` — apply baseline twice (second is a no-op), apply sequential migrations, refuse destructive in prod.

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

- [ ] Create AWS account (or use existing).
- [ ] Enable MFA on root. Never log in as root after bootstrap.
- [ ] Create an IAM admin user for yourself; create `AWSCLI` access keys stored in a password manager.
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
- [ ] First boot: `apt update && apt upgrade`; install `postgresql-client` only.
- [ ] Allow `knottyyoga_the_server` to bind port 80 as non-root: `sudo setcap 'cap_net_bind_service=+ep' /opt/knottyyoga/bin/knottyyoga_the_server` (the install script runs this after each deploy). Cleaner than running as root.
- [ ] Create `knottyyoga` system user; `/opt/knottyyoga/{bin,migrations}`; `/etc/knottyyoga/server.env` (chmod 600) containing `PORT=80`, `KNOTTYYOGA_ORIGIN_SECRET=<random>`, `KNOTTYYOGA_TRUST_PROXY=1`, the `KNOTTYYOGA_DB_*` vars, and `KNOTTYYOGA_DB_SSLROOTCERT=/etc/knottyyoga/rds-ca.pem`.
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
  2. Write `/etc/knottyyoga/server.env` with `KNOTTYYOGA_DB_*` vars.
  3. Run `knottyyoga_database_helper --migrate` (creates schema + `config_secrets` table empty).
  4. Run `knottyyoga_test_helper` to insert initial secret rows (or write a dedicated `knottyyoga_database_helper --seed-secrets-from-file secrets.json` subcommand — small scope, worth doing).
  5. `systemctl start knottyyoga-server`. Server now boots, loads secrets, configures Square + Mail + CORS.
- [ ] Add the `--seed-secrets-from-file` subcommand to `database_helper` + a test that validates ingestion.

---

# Phase 5 — Initial Deploy

## 5.1 Manual first deploy

Purposely manual — gets you comfortable with the pieces before automating.

- [ ] Build artifacts locally (or via temporary GitLab CI one-shot).
- [ ] SCP tarballs to EC2: `scp knottyyoga-v1.0.0.tar.gz ubuntu@<ip>:/tmp/`.
- [ ] Extract to `/opt/knottyyoga/` via a small shell script (`deploy/install.sh`) that also:
  - Installs systemd units.
  - Grants the server the `cap_net_bind_service` capability so it can bind port 80 as the `knottyyoga` user.
  - Runs `knottyyoga_database_helper --migrate`.
  - `systemctl daemon-reload && systemctl enable --now knottyyoga-server`.
- [ ] Smoke test: `curl https://knottyyoga.example/api/health`.
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
  1. Downloads artifact.
  2. Extracts to versioned dir (`/opt/knottyyoga/releases/vX.Y.Z/`).
  3. Runs migrations.
  4. Atomically swaps `/opt/knottyyoga/current` symlink.
  5. `systemctl restart knottyyoga-server`.
  6. Health-check poll; abort + rollback symlink if health fails within 30s.

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
  → **Decision**: in `knottyyoga_helper`, keep only the **scheduled-jobs runner** (subscription renewals, reminders, etc.). Drop the watchdog/watchdog-of-watchdog pieces. Phase 5.3 will add the CloudWatch alarms + Synthetics canary. Update `Scheduled Jobs.md` to reflect this when you implement that helper.
- ✅ **Square credentials** — values come from `secret_values.cpp` (the `production`/`debug` ifdef'd block). Phase 1.4 will pull the sandbox values for `environment.prod.ts` and the production values when you flip live.
- ✅ **Backup testing** — exercise RDS restore once during initial deploy, then quarterly. (Tracked in Phase 5.1 + Phase 8.)
- ✅ **Savings Plan timing** — run on-demand for 2–4 weeks, then buy a **1-yr Compute Savings Plan**. Switching is easy: Compute Savings Plans commit to a $/hr spend, not a specific instance, so changing instance type/family/size/region (e.g., later migrating to ARM `t4g.small`) keeps the discount as long as you stay within the committed hourly burn. The lock-in cost is "you owe AWS this $/hr for 12 months even if you scale down." For RDS the equivalent is a Reserved Instance, which *is* tied to instance family — so RDS RI commitment should wait until you're confident on `db.t3.micro`, OR be skipped (the RDS RI savings on a single small instance are only ~$50/yr; not worth the inflexibility).
- ✅ **Log retention** — journald capped at 500 MB on the EC2; CloudWatch Logs retention 30 days. (Phase 5.3.)
- ✅ **Admin access** — Mason only on day one, but design for granting access to others. **Use AWS Systems Manager Session Manager**, not raw SSH key juggling, for the secondary operator. SSM gives you: no public key on the EC2, AWS-IAM-controlled access (grant/revoke instantly via IAM policy), full audit trail in CloudTrail, no inbound port 22 needed. The retired-friend gets an IAM user + Session Manager permission, runs `aws ssm start-session --target <instance-id>` from their machine, and they're in. Phase 5.2 details.
- ✅ **`db_schema/` snapshots** — git tags only; no directory copies.
- ✅ **Destructive migration safety** — `--recreate_database` blocked in prod unless `KNOTTYYOGA_ALLOW_DESTRUCTIVE=1` env var is set. (Phase 3.3.)

---

# Phase 0 — Decisions checklist (fill before Phase 1 starts)

- [x] Architecture committed — EC2 + RDS + S3 + CloudFront, x86-64, no nginx
- [x] Domain chosen — `KnottyYoga.com` (keep at current registrar; Route 53 hosted zone for DNS only)
- [x] AWS region chosen — `us-west-2` (app); `us-east-1` (ACM cert for CloudFront)
- [x] Square sandbox values confirmed — pull from `secret_values.cpp` ifdef'd `production`/`debug` block
- [ ] SES sender identity agreed (likely `noreply@knottyyoga.com`; needs your call on the local-part)
- [x] Staging env — **no**, soft-launch environment doubles as staging (no DNS, sandbox Square, friends-only)
- [x] `knottyyoga_helper` in-scope for soft launch — scheduled-jobs runner only; AWS Synthetics + CloudWatch alarms replace the custom watchdog
- [x] Resolved Questions log filled in