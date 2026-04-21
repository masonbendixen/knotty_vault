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

> **Note**: this TL;DR has been superseded by "Updated recommendation (TL;DR v2)" further down. Leaving the original for history — see v2 for the current recommendation (EC2 + RDS + S3 + CloudFront, no nginx).

For a "let a few people try it out" soft launch with Square sandbox, the simplest viable footprint is a single Lightsail VPS. After walking through the tradeoffs (and Mason's follow-up questions), we've shifted to the real-AWS architecture instead, which is documented below. This section is retained only so the Lightsail fallback stays legible as an alternative.

- **Compute**: One AWS Lightsail VPS (Ubuntu 22.04 LTS, 2 vCPU / 2 GB, ~$12/mo) running:
  - nginx as TLS terminator + static file server + reverse proxy to C++ server
  - `knottyyoga_the_server` (C++ Crow) on `127.0.0.1:18080`, managed by systemd
  - `knottyyoga_helper` (scheduled jobs + watchdog, from `Scheduled Jobs.md`), managed by systemd — once it lands
  - `knottyyoga_test_helper` run on-demand via SSH (not a persistent service)
- **Database**: Lightsail managed PostgreSQL (~$15/mo for the smallest plan).
- **DNS + TLS**: Route 53 for the domain, Let's Encrypt via certbot on the VPS.
- **Frontend**: Built Angular bundle served by nginx from the same VPS (same-origin with `/api/*`).
- **Email**: Amazon SES.
- **Square**: Sandbox for the initial rollout; flip `kSquareEnvironment` secret to `production` later.
- **Estimated total monthly**: **~$30/mo** flat.

## Hosting Option Comparison

| Option | Pros | Cons | Good for |
|---|---|---|---|
| Lightsail VPS + Lightsail PG | Cheapest flat pricing, bundled bandwidth, simple mental model | Doesn't compose with S3/CloudFront/ACM/IAM cleanly | Tiny soft launch, DigitalOcean-style |
| **EC2 + RDS + S3 + CloudFront** (recommended, see below) | Proper separation of static vs API; CDN for free-tier frontend; grows with traffic | More pieces to configure once; slightly higher billing surface | Real-world production — what I'd deploy |
| EC2 + RDS + ALB | Full AWS power + managed LB for blue/green | ALB is $16–22/mo you don't need yet | When you need multi-instance HA |
| ECS Fargate + RDS | No hosts to patch | More complex IaC, higher $/hr | Team with container discipline |
| AWS App Runner + RDS | Minimal ops, auto-scale | Pricey on tiny workloads; quirky with long-lived conns | Not a great fit |
| Elastic Beanstalk | Quick start | Legacy-feeling, opaque when things break | Skip |
| EC2 + self-hosted Postgres | Cheapest possible | You own backups/upgrades/replication | Skip for a payments app |

## Answers to inline questions

### Does it need to be ARM? Can I build for x86?

**x86 is totally fine** — no reason you must build for ARM. Crow, libpqxx, Boost, Conan 2, and the rest of the stack all support both. ARM (Graviton) is ~10–20% cheaper for the same performance, but it adds friction:

- Your Windows dev box is x86-64. Building an x86-64 Linux binary from CI (GitLab shared runners are x86) is trivial. Building an ARM64 Linux binary either requires ARM runners (self-hosted) or cross-compilation, which is annoying.
- If you containerize later, Docker BuildKit can cross-build, but Conan's dep cache and some sharp C++ libs don't always play nice.
- **Recommendation**: ship x86-64 for the soft launch. Revisit ARM when you're optimizing cost at scale. The ~$2/mo saved isn't worth the build-system churn right now.

### What is Lightsail for?

Lightsail is AWS's "simplified VPS" — think DigitalOcean or Linode, but inside the AWS account. One monthly price (no line items) gets you a VM, bundled bandwidth, optional managed DB, optional load balancer, and a DNS/domain. It's designed for MVPs and people intimidated by full AWS. The catch: **it doesn't compose cleanly with the rest of AWS**. You can't use S3+CloudFront as a front door to a Lightsail instance in any first-class way — Lightsail lives in its own VPC with restricted peering, and integrations with RDS/IAM/ACM are weak. The moment you want the full AWS toolbox, Lightsail is the wrong home.

### EC2 vs Lightsail cost breakdown (similar spec, us-west-2, April 2026 rates)

| Item | Lightsail 2GB ARM | Lightsail 2GB x86 | EC2 t4g.small (ARM) | EC2 t3.small (x86) |
|---|---:|---:|---:|---:|
| Compute (on-demand) | $12.00/mo | $12.00/mo | $12.26/mo | $15.18/mo |
| Compute (1-yr reserved, no upfront) | — | — | ~$7.50/mo | ~$9.50/mo |
| Storage (60 GB SSD included vs 20 GB EBS gp3) | included | included | $1.60/mo | $1.60/mo |
| Bandwidth out | 3 TB included | 3 TB included | $0.09/GB (no free tier on Lightsail migration) | $0.09/GB |
| Static IP | free | free | $3.60/mo if not attached to running instance | same |

**Takeaway**: Lightsail's headline price beats EC2 *only because of the bundled 3 TB of outbound data*. A small API at light traffic moves maybe 20–100 GB/mo → $1.80–$9/mo on EC2, so actual delta is ~$5–10/mo. When you front the server with CloudFront (see the next section), almost all outbound goes through CloudFront instead — and CloudFront's data-out price is cheaper than EC2's, plus its first 1 TB is free for the first 12 months. So with CloudFront in front, EC2 effectively ties or beats Lightsail on cost and gives you everything AWS offers.

### Is S3 + CloudFront + RDS + EC2/ECS a viable architecture?

**Yes — and it's what I'd actually recommend for Knotty Yoga.** The stateless C++ server is a near-perfect fit for the split.

Architecture sketch:

```
              ┌──────────────────┐
 users ───►   │   CloudFront     │  (TLS via free ACM cert)
              │   distribution   │
              └──────┬───────┬───┘
                     │       │
        /* (default)│       │ /api/*  (no caching, forward cookies + auth)
                     ▼       ▼
              ┌──────────┐  ┌──────────────────────┐
              │    S3    │  │  EC2 t3.small (x86)  │
              │ Angular  │  │  Crow server :8080   │
              │  bundle  │  │  systemd, no TLS     │
              └──────────┘  └──────────┬───────────┘
                                       │ TLS (RDS cert)
                                       ▼
                                ┌──────────────┐
                                │ RDS db.t4g.  │
                                │    micro     │
                                │  Postgres 15 │
                                └──────────────┘
```

**Why this is better than Lightsail for a real app**:

1. **Faster frontend**: the Angular bundle is served from S3 via CloudFront edge POPs — your users in Seattle hit a POP in Seattle, not Oregon. Dramatic perceived-speed win over a single VPS.
2. **API isn't competing with static serving**: CloudFront eats all the "give me a .js file" requests. Your EC2 only handles `/api/*` — genuine work.
3. **Free TLS via ACM**: provisioned and renewed automatically, attached to the CloudFront distribution. No certbot to babysit.
4. **Native composability**: RDS ↔ EC2 via VPC security groups, S3 ↔ CloudFront via OAC, IAM for deploy creds, CloudWatch for all logs. Everything integrates.
5. **Deploy the frontend independently**: `aws s3 sync ui/dist/ui/ s3://…` + CloudFront invalidation is simpler than SCP'ing a tarball for a frontend-only change. Backend deploys stay on the EC2.

**Gotchas to be aware of**:

1. **SPA routing**: CloudFront needs a "Custom Error Response" rule that rewrites 403/404 to `/index.html` with HTTP 200 — otherwise deep-linked Angular routes break on refresh. ~5 lines of config but you *must* remember it.
2. **Cookies through CloudFront**: for the `/api/*` behavior you must set Cache Policy to `CachingDisabled` AND Origin Request Policy to `AllViewer` (forwards all cookies, query strings, headers). Miss this and either caching breaks auth (old session leaks) or cookies don't reach the origin.
3. **Invalidation on deploy**: CloudFront caches static assets by content hash (Angular's hashed filenames) so the bundle auto-busts. But `index.html` is NOT hashed — you need to invalidate `/index.html` on every frontend deploy. Trivial (`aws cloudfront create-invalidation --paths /index.html`) but easy to forget.
4. **Origin protection**: with CloudFront in front, attackers could still hit the EC2 IP directly and bypass the CDN. Mitigate with one of: (a) restrict EC2's security group to CloudFront IP ranges only (AWS publishes them — but they change, needs periodic update), (b) require a custom header in CloudFront → origin and have nginx/Crow reject requests without it, (c) put the EC2 in a private subnet behind an internal ALB that CloudFront talks to (more $). (b) is the pragmatic answer.
5. **WebSockets / SSE**: CloudFront supports them with extra config, but we don't use either today.
6. **Slightly longer first-time setup**: you'll spend a couple of hours wiring CloudFront's behaviors, cache policies, origin access, ACM cert. After that it's static config.

**What about ECS vs EC2?** ECS Fargate removes the "patch the VM" chore but costs more (~$15/mo for an always-on 0.25 vCPU / 0.5 GB task plus $0.04/GB-hr ephemeral storage; realistically $20–25/mo for a 1 GB task) and the deployment story is more container-centric. If you already have a containerized build, Fargate is tidy. Given we don't yet have a production Dockerfile, **EC2 is simpler for v1**. We can graduate later.

## Updated recommendation (TL;DR v2)

Build for the real-AWS architecture from the start, and **skip nginx entirely**. You're right that nginx is just a middleman once CloudFront is doing TLS + reverse-proxy + static serving — every one of its classic roles is already covered. We'll have Crow listen on port 80 directly, with a tiny middleware that enforces the CloudFront-origin secret.

- **Compute**: EC2 `t3.small` (x86, ~$15/mo on-demand, ~$9.50/mo reserved) running `knottyyoga_the_server` + `knottyyoga_helper` under systemd. Crow binds 0.0.0.0:80. No TLS, no nginx.
- **Database**: RDS `db.t3.micro` Postgres (x86, ~$13/mo + $2.30/mo storage), single-AZ, automated backups on (RDS does daily snapshots + 7-day point-in-time recovery *out of the box, free*).
- **Frontend**: S3 bucket + CloudFront distribution. Angular bundle deployed via `aws s3 sync`.
- **CDN/TLS/Reverse proxy**: single CloudFront distribution with two behaviors — default → S3, `/api/*` → EC2 origin. Free ACM cert.
- **Origin protection**: CloudFront adds a custom header `X-Origin-Secret: <random>` on every forwarded request. Crow middleware drops any request missing that header. Attackers hitting the EC2 IP directly get a 403. Simpler and more robust than SG-by-CloudFront-IP-prefix, and less code than standing up nginx.
- **DNS**: Route 53 hosted zone + apex alias to CloudFront.
- **Email**: SES (same as before).
- **Monthly total**: ~$32–40/mo on-demand, ~$27–35/mo with 1-yr reserved instance.

This reads more expensive than Lightsail on paper (~$5/mo more) but gives you: edge caching, free TLS, proper backups, real AWS IAM, clean scaling runway. For a payments-processing app, worth it.

### Why nginx adds no value here (justification)

The classic reasons to put nginx in front of an app server — and what replaces them in this architecture:

| nginx role | Replaced by |
|---|---|
| TLS termination | CloudFront + ACM (free, auto-renewed) |
| HTTP → HTTPS redirect | CloudFront "Redirect HTTP to HTTPS" viewer protocol policy |
| Static file serving | S3 via CloudFront OAC |
| Reverse proxy to app server | CloudFront origin behavior `/api/*` → EC2 |
| gzip / compression | CloudFront auto-compression + Crow's `compress` middleware |
| Request logging | CloudFront access logs to S3 + CloudWatch |
| Rate limiting (basic) | CloudFront request-limit behaviors; AWS WAF for anything serious |
| Graceful reload on deploy | systemd restart + CloudFront never-down |
| Origin-secret check | Crow middleware (see Phase 1.7) |

The one genuine miss: if you needed to serve something non-HTTP directly from the EC2 (WebSockets, SSE, a second process on a different port) you'd want nginx to multiplex. The C++ server exposes only `/api/*` over HTTP → no multiplexing needed.

**What to drop from the old Lightsail plan**:

- Certbot on the VPS (ACM handles TLS at CloudFront).
- nginx entirely (everything it did is covered by CloudFront or Crow middleware).
- Lightsail managed DB (RDS replaces).
- Static files on the VPS (S3 replaces).

**What carries over unchanged**:

- Phases 1 (code prereqs) and 3 (DB migrations) don't care which compute we pick.
- Phase 6 (GitLab CI) mostly unchanged; deploy step now invokes `aws s3 sync` + CloudFront invalidation + SSH-to-EC2.
- Phase 7 (versioning + rollback) unchanged.

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
- [ ] Produce a single tarball `knottyyoga-<version>.tar.gz` with a flat layout: `bin/`, `lib/`, `systemd/` (units), `migrations/` (see Phase 3). No `nginx/` — CloudFront replaces it on the recommended path. (Add an `nginx/` folder only if you pick the Lightsail fallback.)
- [ ] Decide on the target OS/arch. **Recommendation**: Ubuntu 22.04 LTS on ARM64 (Lightsail/EC2 Graviton is ~20% cheaper and plenty fast for Crow). Pin this in the build image.

## 2.2 systemd units

- [ ] `knottyyoga-server.service` — `ExecStart=/opt/knottyyoga/bin/knottyyoga_the_server`, `EnvironmentFile=/etc/knottyyoga/server.env`, `Restart=on-failure`, `User=knottyyoga`.
- [ ] `knottyyoga-helper.service` — same pattern for the scheduled jobs/watchdog helper (when it exists).
- [ ] **Do not** create a unit for `knottyyoga_test_helper` — it stays manual via SSH.
- [ ] Log lines validating env var wiring (matches 1.1 / 1.3).

## 2.3 nginx reverse-proxy configuration (Lightsail fallback only — skip on recommended path)

Only needed if you're going with Option B (Lightsail). On the recommended path (Option A), CloudFront handles TLS, HTTP→HTTPS redirect, and static serving — nothing for nginx to do.

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

## 4.3 Compute: Lightsail VPS (legacy path — skip if going with EC2 + CloudFront)

- [ ] Create Lightsail instance: Ubuntu 22.04 LTS, 2 vCPU / 2 GB plan ($12/mo).
- [ ] Attach a static IP (free while attached to an instance).
- [ ] Upload your SSH public key during creation (do not use Lightsail's default key).
- [ ] After first boot: `apt update && apt upgrade`, install `nginx`, `postgresql-client`, `certbot python3-certbot-nginx`.
- [ ] Create `knottyyoga` system user (`useradd -r -s /bin/false knottyyoga`).
- [ ] Create `/opt/knottyyoga/{bin,ui,migrations}` owned by that user, `/etc/knottyyoga/server.env` (chmod 600, root:knottyyoga).
- [ ] Enable `ufw` with rules: deny incoming default, allow 22/80/443.

## 4.3-alt Compute: EC2 (recommended path — no nginx)

- [ ] Create key pair in EC2 console (or import your existing public key).
- [ ] Launch `t3.small` (x86) instance, Ubuntu 22.04 LTS AMI, 20 GB gp3 root volume, in default VPC public subnet.
- [ ] Security group `sg-knottyyoga-web`:
  - Inbound 22 from your home IP only.
  - Inbound 80 from `0.0.0.0/0`. Origin protection is enforced in the Crow middleware (`X-Origin-Secret` check — see Phase 1.7), not at the SG level. This avoids the ongoing chore of keeping up with CloudFront's IP prefix list.
  - No 443 (CloudFront handles TLS; Crow listens HTTP on 80).
- [ ] Allocate an Elastic IP, attach it to the instance. Free while attached. Needed so the DNS / CloudFront origin target doesn't change on stop/start.
- [ ] First boot: `apt update && apt upgrade`; install `postgresql-client` only. No nginx, no certbot.
- [ ] Allow `knottyyoga_the_server` to bind port 80 as non-root: `sudo setcap 'cap_net_bind_service=+ep' /opt/knottyyoga/bin/knottyyoga_the_server` after each deploy (the install script handles this). Cleaner than running as root.
- [ ] Create `knottyyoga` system user; `/opt/knottyyoga/{bin,migrations}`; `/etc/knottyyoga/server.env` (chmod 600) containing `PORT=80`, `KNOTTYYOGA_ORIGIN_SECRET=<random>`, `KNOTTYYOGA_TRUST_PROXY=1`, and the `KNOTTYYOGA_DB_*` vars.
- [ ] Enable `ufw` with rules: deny incoming default, allow 22 + 80.
- [ ] Install CloudWatch Agent if you want metrics beyond basic EC2 ones. Optional for v1 — systemd journal tailed to CloudWatch Logs is enough.
- [ ] Consider 1-yr reserved instance once you're confident the instance type is right (locks in ~40% savings).

## 4.4 Database: Lightsail managed PostgreSQL

- [ ] Provision Lightsail PG instance (smallest plan, same region as VPS, same AZ if possible).
- [ ] Enable automated snapshots (Lightsail has a daily snapshot option — turn on).
- [ ] Record connection endpoint, port, master username, master password. Store in your password manager.
- [ ] Create the application database and a non-superuser role for the app (`CREATE ROLE knottyyoga LOGIN PASSWORD '...'; GRANT ALL ON DATABASE knottyyoga TO knottyyoga;`).
- [ ] From the VPS: `psql` a test connection over the private VPC endpoint.

**Lightsail managed PG vs RDS — cost and backup comparison** (answering Mason's question):

| Aspect | Lightsail managed PG (smallest) | RDS db.t3.micro (x86) | RDS db.t4g.micro (ARM) |
|---|---|---|---|
| Compute + 40 GB storage | $15.00/mo flat | $13.14/mo + $2.30/mo (20 GB) = **$15.44/mo** | $11.68/mo + $2.30/mo = **$13.98/mo** |
| 1-yr reserved (no upfront) | not available | ~$9/mo + storage | ~$8/mo + storage |
| Automated daily backups | yes, 7-day retention | yes, 7-day retention (free ≤ DB size) | same |
| Point-in-time recovery (PITR) | **no** — snapshots only | **yes, to any second in retention window** | same |
| Snapshots on demand | yes, $0.05/GB/mo | yes, $0.095/GB/mo (free up to DB size) | same |
| Multi-AZ failover | no | optional (~2× the cost) | same |
| Read replicas | no | yes | yes |
| Parameter tuning / extensions | limited | full | full |
| VPC peering with EC2 in standard AWS | painful | native | native |

**Short answer**: costs are within ~$1–2/mo of each other at this scale. **RDS wins** on backups because of point-in-time recovery (you can restore to 13:47:03 on Tuesday, not just "yesterday's snapshot"). For a payments app you genuinely want PITR. Pick RDS.

If going with the **EC2 + RDS + S3 + CloudFront** recommendation above, this whole section's "Lightsail managed PG" is replaced by RDS. See Phase 4.4-alt below.

## 4.4-alt Database: RDS Postgres (recommended path)

- [ ] Create RDS subnet group spanning at least two AZs in your default VPC (needed even for single-AZ instances).
- [ ] Provision `db.t3.micro` (or `db.t4g.micro` if going ARM), engine Postgres 15, single-AZ, 20 GB gp3 storage, auto-minor-version upgrades on.
- [ ] Enable automated backups with 7-day retention (default). Turn on "deletion protection" so a rogue script can't `aws rds delete-db-instance` you into the ground.
- [ ] Security group `sg-knottyyoga-db`: inbound 5432 from `sg-knottyyoga-web` only.
- [ ] Record the RDS endpoint; note it's a DNS name (e.g., `knottyyoga.xxxxxx.us-west-2.rds.amazonaws.com`), not a static IP — put in `/etc/knottyyoga/server.env` as `KNOTTYYOGA_DB_HOST`.
- [ ] Create the application database and non-superuser role (same SQL as the Lightsail path).
- [ ] Download the AWS RDS CA bundle to `/etc/knottyyoga/rds-ca.pem`; set `KNOTTYYOGA_DB_SSLMODE=verify-full` and add a `KNOTTYYOGA_DB_SSLROOTCERT` env var that the connection-string builder can pick up. (Phase 1.1 already plans for sslmode.)
- [ ] Verify PITR by running a toy restore (Phase 5.1 smoke test).

## 4.5 DNS + TLS (Lightsail path)

- [ ] Buy (or transfer) the domain. **Recommendation**: use Route 53 as registrar too — consolidates billing and DNS control.
- [ ] Create a Route 53 hosted zone. $0.50/mo flat.
- [ ] Create an `A` record pointing the apex (or `www`) to the Lightsail static IP.
- [ ] After DNS propagates, run `sudo certbot --nginx -d knottyyoga.example -d www.knottyyoga.example` to get an LE cert. Certbot sets up auto-renewal via a systemd timer.
- [ ] Confirm HTTPS reachable, HTTP auto-redirects, certificate chain is valid (`ssl-labs` test — aim for A).

## 4.5-alt DNS + TLS (CloudFront path, recommended)

- [ ] Buy domain via Route 53; hosted zone $0.50/mo.
- [ ] In ACM (in `us-east-1` — CloudFront *requires* certs from us-east-1, even if your origin is elsewhere!), request a public cert for `knottyyoga.example` and `www.knottyyoga.example` with DNS validation. Route 53 can auto-create the validation CNAMEs — one click.
- [ ] Do **not** create `A` records for the domain yet — they'll point at the CloudFront distribution once it's created (Phase 4.x).
- [ ] When the CloudFront distribution is live, create Route 53 `A` alias records (apex + `www`) pointing to the CloudFront distribution. Alias records are free (no per-query cost).

## 4.x S3 + CloudFront (recommended path)

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
  - Grants the server the `cap_net_bind_service` capability so it can bind port 80 as the `knottyyoga` user.
  - Runs `knottyyoga_database_helper --migrate`.
  - `systemctl daemon-reload && systemctl enable --now knottyyoga-server`.
  - (Lightsail fallback only: installs nginx snippet and reloads nginx.)
- [ ] Smoke test: `curl https://knottyyoga.example/api/health`.
- [ ] Log in via the frontend, register a user, process a sandbox Square payment end-to-end.

## 5.2 SSH access hardening

- [ ] Disable password auth in `/etc/ssh/sshd_config` (`PasswordAuthentication no`).
- [ ] Use key-based auth only; record public keys of any authorized operator in `~/.ssh/authorized_keys` for both `ubuntu` and `knottyyoga` (knottyyoga for emergency access if needed).
- [ ] Add a `RUNBOOK.md` section describing how to run `knottyyoga_test_helper` via SSH — which commands are safe in prod, which ones aren't.
- [ ] Optional: enable AWS Systems Manager Session Manager as a backup access path so you don't depend on your home IP / SSH key forever. Only meaningful if we move off Lightsail.

## 5.3 Observability (low-cost baseline)

- [ ] Set up CloudWatch Logs agent on the EC2 (free tier: 5 GB/mo), tailing the systemd journals for `knottyyoga-server.service` and `knottyyoga-helper.service`. CloudFront's own access logs go directly to a separate S3 bucket (configured on the distribution) if you want HTTP-level visibility — optional, ~$0 at low traffic.
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
- [ ] AWS WAF rules attached to the CloudFront distribution for basic abuse protection (rate limits, common-attack managed rule set, geo-blocking if desired). $5/mo base + $1 per rule + $0.60 per million requests.
- [ ] Separate staging environment (second tiny Lightsail VPS + DB, used for final pre-prod validation).
- [ ] Move Angular bundle to S3 + CloudFront, leaving the VPS to do API only. Reduces VPS load; enables edge caching.
- [ ] Structured JSON logging — easier to grep CloudWatch.
- [ ] Encrypted secrets-at-rest in the `config_secrets` table (column-level encryption with a key from env var) instead of plaintext. Plaintext is ok for a tiny soft launch but you'll want this before real revenue flows.

---

# Monthly Cost Estimate (soft launch)

## Per-service cost detail (Option A breakdown)

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

## Option A — recommended: EC2 + RDS + S3 + CloudFront

| Line item | On-demand $/mo | 1-yr reserved $/mo |
|---|---:|---:|
| EC2 t3.small (x86, 2 vCPU / 2 GB) | $15.18 | $9.50 |
| EBS gp3 20 GB root | $1.60 | $1.60 |
| EC2 data out (tiny, most traffic via CloudFront) | ~$0.50 | ~$0.50 |
| RDS db.t3.micro + 20 GB gp3 | $15.44 | $11.30 |
| RDS snapshots (up to DB size free; beyond that $0.095/GB) | ~$0 | ~$0 |
| S3 storage (Angular bundle ≈ 5 MB) | ~$0 | ~$0 |
| S3 requests + data out to CloudFront (AWS-internal) | ~$0 | ~$0 |
| CloudFront (first 1 TB out free 12 months, then $0.085/GB) | ~$0 | ~$0 |
| CloudFront requests (free tier 10M/mo for 12 months) | ~$0 | ~$0 |
| ACM certificate | free | free |
| Route 53 hosted zone | $0.50 | $0.50 |
| Route 53 queries (light) | ~$0.50 | ~$0.50 |
| SES (first 62k emails/mo free from AWS egress) | ~$0 | ~$0 |
| CloudWatch Logs (< 5 GB free tier) | ~$0 | ~$0 |
| Domain registration (amortized) | ~$1.00 | ~$1.00 |
| **Total** | **~$35/mo** | **~$25/mo** |

After the first 12 months of AWS "new customer" free tier, add ~$5–10/mo for CloudFront data+requests at the soft-launch scale. Still sub-$50/mo.

## Option B — alternative: Lightsail VPS + Lightsail managed PG

| Line item | $/mo |
|---|---:|
| Lightsail VPS (2 vCPU / 2 GB) | $12.00 |
| Lightsail managed PostgreSQL (smallest) | $15.00 |
| Lightsail snapshots | ~$1 |
| Route 53 hosted zone + queries | ~$1 |
| SES | ~$0 |
| CloudWatch Logs | ~$0 |
| Domain registration (amortized) | ~$1 |
| **Total** | **~$30/mo** |

Option A costs ~$5/mo more on-demand (cheaper if reserved) and gives you a real CDN, native AWS integrations, point-in-time DB recovery, and room to scale without migration. **Pick A** unless setup time is the hard constraint.

Pricing caveat: AWS adjusts prices occasionally; verify current rates in the AWS Pricing Calculator before committing.

---

# Open Questions

These are things I want your answer on before or during implementation. Adding here instead of prompting at the terminal.

1. **Domain**: do you already own a domain for Knotty Yoga, or will you buy one during this project? Does it need to live under a subdomain (e.g., `app.knottyyoga.com`)?
2. **Region**: any preference for `us-west-2` vs `us-east-1` vs something closer to your users? (User latency for a studio in WA/OR/CA strongly favors `us-west-2`.)
3. **Architecture — Option A (EC2 + RDS + S3 + CloudFront) vs Option B (Lightsail)**: my updated recommendation is Option A for the reasons above — better perf, proper backups, room to scale, only ~$5/mo more. Any objection? The tradeoff is ~2–3 extra hours of one-time setup for CloudFront + S3 + IAM.
4. **Staging environment**: do you want a separate staging VPS+DB from the start (~$27/mo extra), or will the soft-launch environment *be* the staging environment for a while?
5. **`knottyyoga_helper` availability**: the Scheduled Jobs plan isn't implemented yet. Do we soft-launch without it (meaning: no automated subscription renewals, no scheduled reminders) and add it in a subsequent release? I think yes — minimizes initial scope.
6. **Square Application ID / Location ID**: are the sandbox values in `Square credentials and Sandbox setup.md` current and correct? I'll pull from there for `environment.prod.ts` unless told otherwise.
7. **Backup/restore testing**: how often do you want to exercise restore from snapshot? My suggestion: once during the initial deploy (prove it works), then quarterly thereafter.
8. **TLS**: with the CloudFront architecture recommendation, TLS is free via ACM and handled by CloudFront. No certbot needed. Agreed?
13. **ARM vs x86 build target**: I'm suggesting x86-64 for the first deploy (simpler CI, same Windows dev box, ~$2/mo more). Any reason to go ARM from day one?
14. **CloudFront origin protection method**: the check lives in the C++ server as a Crow middleware (Phase 1.7), not in nginx. CloudFront injects `X-Origin-Secret`; Crow rejects any request missing/mismatched. No nginx involved. Agreed?
15. **Reserved instance commitment**: 1-yr reserved (no upfront) saves ~40% on EC2/RDS. I'd commit after ~2 weeks of running on-demand to confirm the instance type is right. Agreed?
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