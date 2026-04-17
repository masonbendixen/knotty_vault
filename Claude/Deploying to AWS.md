---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/16/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I'm getting ready to start deploying to AWS. I will initially deploy with the Square sandbox to let a few people try it out and get used to the flow. I'd like to figure out what will be involved to deploy to AWS. The C++ server really has no state itself. I also need to run the scheduled jobs process and have the test helper running so that I can log in through SSH and do various operations. I also need to deploy the database helper to set the initial state of the database. I also need a hosted postgres database.

I need to point DNS to the server, enable SSH. What other things do I need to be aware of? What are the costs going to be like? Which AWS hosting options are the best fit for me?

I also figure that once I have deployed, I need a plan for updating the server going forward. I figure when I deploy versions, I should probably save branches in GIT. I also might want to save snapshot copies of the db_schema folder for different versions and create update utilities to migrate / evolve the database schema. If I need to change a database table, is it better to give it a new table name? What are industry standards for this? I also use gitlab for version control. It supports creating a CI/CD pipeline but my tests on the server rely on a postgres database. Can I add that to a CI/CD pipeline on Gitlab?

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Executive Summary & Recommendations

## TL;DR — Recommended Shape of the First Deploy

For a "let a few people try it out" soft launch with Square sandbox, I recommend the **simplest viable AWS footprint** and then evolving it later:

- **Compute**: One AWS Lightsail VPS (Ubuntu 22.04 LTS, ARM, 2 vCPU / 2 GB, ~$12/mo) running:
  - nginx as TLS terminator + static file server + reverse proxy to C++ server
  - `knottyyoga_the_server` (C++ Crow) on `127.0.0.1:18080`, managed by systemd
  - `knottyyoga_helper` (scheduled jobs + watchdog, from `Scheduled Jobs.md`), managed by systemd — once it lands
  - `knottyyoga_test_helper` run on-demand via SSH (not a persistent service)
- **Database**: Lightsail managed PostgreSQL (~$15/mo for the smallest plan) OR self-host PostgreSQL on the same VPS for the first few weeks and migrate to managed later. Recommendation: **managed from day one** — backups + PITR are cheap insurance for real customer data.
- **DNS + TLS**: Route 53 for the domain, Let's Encrypt via certbot on the VPS (or AWS Certificate Manager if we later move to ALB).
- **Frontend**: Built Angular bundle served by nginx from the same VPS (same-origin with `/api/*` — no CORS needed, cookies simpler). *Alternative*: S3 + CloudFront, but that's more moving parts for v1.
- **Email**: Amazon SES (starts in sandbox mode — need to request production access). Square confirmation emails are already wired via `MailHelper`.
- **Square**: Sandbox for the initial rollout; flip `kSquareEnvironment` secret to `production` later.
- **Estimated total monthly**: **~$30–$50/mo** for the small-scale initial deploy.

Why Lightsail instead of EC2 + ALB + RDS for v1: it bundles bandwidth, has predictable flat pricing, and avoids the "death by a thousand line items" bill. We can graduate to full EC2/ECS/RDS when traffic or requirements demand it. The application architecture is so portable (stateless server, Postgres-only persistence) that moving later is straightforward.

## Hosting Option Comparison

| Option | Pros | Cons | Good for |
|---|---|---|---|
| **Lightsail VPS + Lightsail PG** (recommended) | Cheapest, flat pricing, includes bandwidth, simple mental model | Less flexible than full EC2, smaller instance sizes | Small-scale soft launch |
| EC2 + RDS + ALB + Route 53 | Full AWS power, horizontal scaling, managed cert via ACM | More pieces, more billing surface, more config | Long-term production |
| ECS Fargate + RDS | No EC2 hosts to patch, easy rolling deploys | More complex IaC, cold starts less of an issue here but still | Team with containerization discipline |
| AWS App Runner + RDS | Minimal ops, auto-scale | Fargate pricing on tiny workloads is expensive; some quirks with long-lived connections | Not a great fit |
| Elastic Beanstalk | Quick start | Legacy-feeling, opaque when things break | Skip |
| **EC2 + self-hosted Postgres** | Cheapest possible | You own backups/upgrades/replication | Only if budget is tight AND you accept operational risk |
Mason- Does it need to be ARM that I build for? It can't be x86? Can you give a cost breakdown of EC2 especially compared to lightsail. What is lightsail for?
## Critical Code Gaps That Block Deploy (Summary)

These come first — they're the Phase 1 work. Each is detailed in its phase section below.

1. **DB connection is hardcoded** in `sql_util/database_access/database_helper_init.cpp` (user=docker, password=docker, host=postgresql). This **must** be driven by env vars before we can point at RDS/Lightsail PG.
2. **Secret bootstrap**: secrets live in the `config_secrets` table, but database credentials themselves can't live there (chicken-and-egg). DB credentials need env vars; everything else stays DB-backed.
3. **Frontend `environment.prod.ts`** is a stub — missing Square Application ID and Location ID.
4. **No health endpoint** (needed for LB/watchdog probes and for the `knottyyoga_helper` watchdog mode).
5. **No migration mechanism** — `database_helper` destructively rebuilds the DB, which is fine for dev but will wipe customer data in prod. Must add a forward-only, versioned migration path before the second deploy.
6. **No production Dockerfile** (or native build recipe) — `server/docker_project/Dockerfile` is only a build-env stub.
7. **No `.gitlab-ci.yml`** — CI with postgres service is totally feasible in GitLab and we'll wire that up.

---

# Phase 1 — Code & Config Prerequisites (Lowest Layer First)

Goal: make the application configurable per environment and observable enough to run unattended on a VPS. These changes should land before any AWS work.

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

**Advice**: systemd captures stdout/stderr automatically into the journal — no need for a custom log file path in the container/VPS deploy. Simpler is better.

## 1.4 Frontend environment configuration

- [ ] Populate `ui/src/environments/environment.prod.ts` with Square **sandbox** Application ID and Location ID for the initial rollout (pulled from the existing `Square credentials and Sandbox setup.md`). These are client-side public identifiers — they're supposed to be in the bundle.
- [ ] Decide: do we want a separate `environment.prod-square-live.ts` configuration for when we flip to Square production? **Recommendation**: yes — create the config but leave commented until we're ready, so the "soft launch" build isn't accidentally using live Square credentials.
- [ ] Add a production build configuration in `ui/angular.json` if one doesn't already exist that maps to `environment.prod.ts`.
- [ ] Ensure the frontend uses relative URLs (`/api/...`) so it works same-origin behind nginx. Scan `ServerAccessNetwork.ts` for any hardcoded absolute URLs — if present, make them use a `baseUrl` from environment config.

## 1.5 Cookies + CORS sanity pass for same-origin deploy

Currently `ServerConfig::Initialize` reads `kWebsiteAddress` from DB secrets and configures CORS when `prodMode_` is on. If the frontend and backend ship from the same origin via nginx (recommended), CORS isn't actually exercised — but the config still needs to be correct for the health of cookies.

- [ ] Verify: with nginx terminating TLS and proxying `/api/*` to the C++ server, the browser sees `Origin: https://knottyyoga.example` for both static assets and API. Same-origin → CORS preflight not triggered → cookies flow without `SameSite=None; Secure` gymnastics.
- [ ] Document in `Deploying to AWS.md` (this doc) the secret values that must be set before first boot: `kWebsiteAddress`, `kServerProductionMode=true`, `kSquareAccessToken`, `kSquareEnvironment=sandbox`, plus any email/SES secrets.
- [ ] If any auth code currently assumes the frontend lives at a *different* origin, add a test fixture exercising the same-origin case and reverse-proxy header handling (`X-Forwarded-Proto`, `X-Forwarded-For`).

## 1.6 Reverse-proxy awareness in the C++ server

- [ ] Confirm the server trusts `X-Forwarded-Proto: https` when setting the `Secure` flag on cookies. If today it infers scheme from the request itself (which will be `http` behind nginx), cookies set as `Secure` will be dropped by the browser.
- [ ] Add a `KNOTTYYOGA_TRUST_PROXY` flag that, when true, tells the cookie/session code to treat the forwarded scheme as authoritative.
- [ ] Tests for both the trust-proxy-on and trust-proxy-off paths in `cookie_manager_test.cpp` or a new `proxy_trust_test.cpp`.

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
- [ ] Produce a single tarball `knottyyoga-<version>.tar.gz` with a flat layout: `bin/`, `lib/`, `systemd/` (units), `nginx/` (conf snippet), `migrations/` (see Phase 3).
- [ ] Decide on the target OS/arch. **Recommendation**: Ubuntu 22.04 LTS on ARM64 (Lightsail/EC2 Graviton is ~20% cheaper and plenty fast for Crow). Pin this in the build image.

## 2.2 systemd units

- [ ] `knottyyoga-server.service` — `ExecStart=/opt/knottyyoga/bin/knottyyoga_the_server`, `EnvironmentFile=/etc/knottyyoga/server.env`, `Restart=on-failure`, `User=knottyyoga`.
- [ ] `knottyyoga-helper.service` — same pattern for the scheduled jobs/watchdog helper (when it exists).
- [ ] **Do not** create a unit for `knottyyoga_test_helper` — it stays manual via SSH.
- [ ] Log lines validating env var wiring (matches 1.1 / 1.3).

## 2.3 nginx reverse-proxy configuration

- [ ] `nginx/knottyyoga.conf` snippet:
  - `server_name knottyyoga.example;`
  - Listen 443 SSL with Let's Encrypt cert paths.
  - Redirect 80 → 443.
  - Serve Angular bundle from `/opt/knottyyoga/ui/` (`try_files $uri $uri/ /index.html;`).
  - `location /api/` → `proxy_pass http://127.0.0.1:18080;` with `proxy_set_header X-Forwarded-Proto https; X-Forwarded-For $remote_addr; Host $host;`.
  - Long-poll/WebSocket headers if the app uses them (Crow WebSocket support is there; check if any endpoints use it today — I didn't find any, so skip until needed).
- [ ] Document certbot setup steps: `sudo certbot --nginx -d knottyyoga.example`.

## 2.4 Frontend artifact

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
- [ ] Disaster recovery: restore RDS/Lightsail PG snapshot. Write this procedure down in a `RUNBOOK.md` in this repo once Phase 4 is complete.

---

# Phase 4 — AWS Infrastructure

Goal: provision the accounts/services we'll actually deploy to.

## 4.1 Account bootstrap

- [ ] Create AWS account (or use existing).
- [ ] Enable MFA on root. Never log in as root after bootstrap.
- [ ] Create an IAM admin user for yourself; create `AWSCLI` access keys stored in a password manager.
- [ ] Set a **billing alarm** at $75/mo (sanity) so a misconfigured anything doesn't quietly run up a bill.
- [ ] Pick a region. **Recommendation**: `us-west-2` (Oregon) — cheap, reliable; or `us-east-1` if you prefer proximity. Stick with one.

## 4.2 Networking

- [ ] If Lightsail: no VPC work needed. Lightsail manages its own networking; peer it to default VPC only if you need to talk to EC2/RDS later.
- [ ] If EC2+RDS route: use the default VPC for v1. Two subnets (one public for EC2, one private for RDS), which already exist in every default VPC.
- [ ] Security groups:
  - `sg-knottyyoga-web`: allows 22 (SSH from your home IP only), 80, 443 from 0.0.0.0/0.
  - `sg-knottyyoga-db`: allows 5432 from `sg-knottyyoga-web` only.

## 4.3 Compute: Lightsail VPS

- [ ] Create Lightsail instance: Ubuntu 22.04 LTS, ARM 2 vCPU / 2 GB plan ($12/mo).
- [ ] Attach a static IP (free while attached to an instance).
- [ ] Upload your SSH public key during creation (do not use Lightsail's default key).
- [ ] After first boot: `apt update && apt upgrade`, install `nginx`, `postgresql-client`, `certbot python3-certbot-nginx`.
- [ ] Create `knottyyoga` system user (`useradd -r -s /bin/false knottyyoga`).
- [ ] Create `/opt/knottyyoga/{bin,ui,migrations}` owned by that user, `/etc/knottyyoga/server.env` (chmod 600, root:knottyyoga).
- [ ] Enable `ufw` with rules: deny incoming default, allow 22/80/443.

## 4.4 Database: Lightsail managed PostgreSQL

- [ ] Provision Lightsail PG instance (smallest plan, same region as VPS, same AZ if possible).
- [ ] Enable automated snapshots (Lightsail has a daily snapshot option — turn on).
- [ ] Record connection endpoint, port, master username, master password. Store in your password manager.
- [ ] Create the application database and a non-superuser role for the app (`CREATE ROLE knottyyoga LOGIN PASSWORD '...'; GRANT ALL ON DATABASE knottyyoga TO knottyyoga;`).
- [ ] From the VPS: `psql` a test connection over the private VPC endpoint.

**Alternative**: RDS `db.t4g.micro` (same price ballpark, more flexible but more config). Pick Lightsail for simplicity; migrate to RDS later via logical replication if needed.
Mason- what is the cost difference between lightsail and RDS in cost? RDS does backups too, right?

## 4.5 DNS + TLS

- [ ] Buy (or transfer) the domain. **Recommendation**: use Route 53 as registrar too — consolidates billing and DNS control.
- [ ] Create a Route 53 hosted zone. $0.50/mo flat.
- [ ] Create an `A` record pointing the apex (or `www`) to the Lightsail static IP.
- [ ] After DNS propagates, run `sudo certbot --nginx -d knottyyoga.example -d www.knottyyoga.example` to get an LE cert. Certbot sets up auto-renewal via a systemd timer.
- [ ] Confirm HTTPS reachable, HTTP auto-redirects, certificate chain is valid (`ssl-labs` test — aim for A).

## 4.6 Email via SES

- [ ] Verify the sending domain in SES (add a CNAME/TXT in Route 53).
- [ ] Request production access (SES starts in sandbox mode limiting to verified recipients only). This can take a day.
- [ ] Create an SMTP credential pair in SES. Put the SMTP username/password in `config_secrets` via a one-time `knottyyoga_test_helper` run after first deploy.
- [ ] Verify: trigger a test email path (e.g., `person_verify_mail`) and confirm delivery.

## 4.7 Secret bootstrap ordering

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
- [ ] SCP tarballs to VPS: `scp knottyyoga-v1.0.0.tar.gz ubuntu@<ip>:/tmp/`.
- [ ] Extract to `/opt/knottyyoga/` via a small shell script (`deploy/install.sh`) that also:
  - Installs systemd units.
  - Installs nginx snippet, reloads nginx.
  - Runs `knottyyoga_database_helper --migrate`.
  - `systemctl daemon-reload && systemctl enable --now knottyyoga-server`.
- [ ] Smoke test: `curl https://knottyyoga.example/api/health`.
- [ ] Log in via the frontend, register a user, process a sandbox Square payment end-to-end.

## 5.2 SSH access hardening

- [ ] Disable password auth in `/etc/ssh/sshd_config` (`PasswordAuthentication no`).
- [ ] Use key-based auth only; record public keys of any authorized operator in `~/.ssh/authorized_keys` for both `ubuntu` and `knottyyoga` (knottyyoga for emergency access if needed).
- [ ] Add a `RUNBOOK.md` section describing how to run `knottyyoga_test_helper` via SSH — which commands are safe in prod, which ones aren't.
- [ ] Optional: enable AWS Systems Manager Session Manager as a backup access path so you don't depend on your home IP / SSH key forever. Only meaningful if we move off Lightsail.

## 5.3 Observability (low-cost baseline)

- [ ] Set up CloudWatch Logs agent on the VPS (free tier: 5 GB/mo), tailing `/var/log/nginx/access.log`, the journal for `knottyyoga-server.service`, and `knottyyoga-helper.service`.
- [ ] Set up an uptime check — CloudWatch Synthetics, or something free like UptimeRobot — pointed at `/api/health`. Alert via email.
- [ ] Configure `journalctl` retention to a sensible cap (e.g., 500 MB) so disk doesn't fill.

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
  - SSHs to the VPS using a deploy key stored in GitLab CI variables.
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
- [ ] Operator clicks `deploy-manual` in GitLab → artifact deploys to VPS.
- [ ] VPS `install.sh`:
  1. Downloads artifact.
  2. Extracts to versioned dir (`/opt/knottyyoga/releases/vX.Y.Z/`).
  3. Runs migrations.
  4. Atomically swaps `/opt/knottyyoga/current` symlink.
  5. `systemctl restart knottyyoga-server`.
  6. Health-check poll; abort + rollback symlink if health fails within 30s.

---

# Phase 8 — Nice-to-haves (post-soft-launch)

Not required to ship; listed so we don't forget.

- [ ] CloudFront in front of the Angular bundle for static-asset caching (meaningful only when we see real users).
- [ ] RDS multi-AZ (if we move off Lightsail).
- [ ] AWS WAF rules on nginx for basic abuse protection.
- [ ] Separate staging environment (second tiny Lightsail VPS + DB, used for final pre-prod validation).
- [ ] Move Angular bundle to S3 + CloudFront, leaving the VPS to do API only. Reduces VPS load; enables edge caching.
- [ ] Structured JSON logging — easier to grep CloudWatch.
- [ ] Encrypted secrets-at-rest in the `config_secrets` table (column-level encryption with a key from env var) instead of plaintext. Plaintext is ok for a tiny soft launch but you'll want this before real revenue flows.

---

# Monthly Cost Estimate (soft launch)

| Line item | Approx. $/mo |
|---|---:|
| Lightsail VPS (2 vCPU / 2 GB ARM) | $12 |
| Lightsail managed PostgreSQL (smallest) | $15 |
| Route 53 hosted zone | $0.50 |
| Route 53 queries (light) | ~$0.50 |
| SES (first 62k emails/mo free from EC2/Lightsail; otherwise $0.10/1000) | ~$0 |
| CloudWatch Logs (< 5 GB free tier) | ~$0 |
| Lightsail snapshots (~$0.05/GB) | ~$1 |
| Domain registration (amortized) | ~$1 |
| **Total** | **~$30/mo** |

Costs scale up with traffic mainly on Lightsail bandwidth overage ($0.09/GB after the included 2–3 TB) and RDS/Lightsail DB plan size. For the first dozen users, you'll be nowhere near the ceilings.

Pricing caveat: AWS adjusts prices occasionally; verify current rates before committing.

---

# Open Questions

These are things I want your answer on before or during implementation. Adding here instead of prompting at the terminal.

1. **Domain**: do you already own a domain for Knotty Yoga, or will you buy one during this project? Does it need to live under a subdomain (e.g., `app.knottyyoga.com`)?
2. **Region**: any preference for `us-west-2` vs `us-east-1` vs something closer to your users? (User latency for a studio in WA/OR/CA strongly favors `us-west-2`.)
3. **Lightsail vs. EC2+RDS**: my recommendation is Lightsail for v1. Any reason to jump straight to EC2/RDS (e.g., you already have AWS SSO/IAM strategy, you expect rapid scale, you want Infrastructure-as-Code via Terraform from day one)?
4. **Staging environment**: do you want a separate staging VPS+DB from the start (~$27/mo extra), or will the soft-launch environment *be* the staging environment for a while?
5. **`knottyyoga_helper` availability**: the Scheduled Jobs plan isn't implemented yet. Do we soft-launch without it (meaning: no automated subscription renewals, no scheduled reminders) and add it in a subsequent release? I think yes — minimizes initial scope.
6. **Square Application ID / Location ID**: are the sandbox values in `Square credentials and Sandbox setup.md` current and correct? I'll pull from there for `environment.prod.ts` unless told otherwise.
7. **Backup/restore testing**: how often do you want to exercise restore from snapshot? My suggestion: once during the initial deploy (prove it works), then quarterly thereafter.
8. **TLS**: are you comfortable with Let's Encrypt via certbot (free, auto-renews, industry standard) or do you want AWS ACM? ACM only matters if we add an ALB/CloudFront.
9. **Log retention**: journald default is "until disk fills". Want me to set a fixed cap (e.g., 500 MB) and a CloudWatch retention of 30 days? That's my default recommendation.
10. **Admin access**: who besides you needs SSH access to the VPS? Any second operator's public key we need to include from day one?
11. **"Save snapshot copies of `db_schema/` per version"**: I argued against this above (git tags suffice). Are you persuaded, or do you have a specific reason you want directory copies? There's a scenario where it helps — e.g., generating a schema diff report between two versions — but a script that diffs across git tags solves that too.
12. **Destructive migration safety**: I'm proposing that `--recreate_database` becomes unavailable in prod by default (needs an explicit env var to re-enable). Agreed?

---

# Phase 0 — Decisions checklist (fill before Phase 1 starts)

- [ ] Domain chosen
- [ ] AWS region chosen
- [ ] Lightsail vs. EC2+RDS decision
- [ ] Square sandbox values confirmed
- [ ] SES sender identity agreed
- [ ] Staging env: yes / no / later
- [ ] `knottyyoga_helper` in-scope for soft launch: yes / no
- [ ] Open Questions 1–12 answered