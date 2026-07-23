---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/21/2026
Version: 0.2
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Deploying to AWS (see Deploying to AWS.md), I have realized that there is an opportunity given that Cloudfront serves the client and reverse proxies to the webserver for /api. I have always wanted to support multiple clients with my server code but I have realized I can support multiple clients with the same server instance and probably the same RDS database.

I can have either the same Angular client with a client personalization layer OR have fully separate client bundles that share a lot of components and client infrastructure with their own top level app for maximum flexibility. Regardless, I can have all of them share the same server. Cloudfront can pass a unique identifier for each front end site. We can modify the server to take this site identifier. to multiplex the same server for multiple sites.

We need to figure out how to differentiate different clients in the database. One way would be to add a site identifier to every table. Another would be separate schemas. A more costly but fully isolated option would be multiple databases. Can you help me explore the various alternatives here to make a decision?

Can you start with brainstorming the high level work here before moving on to the actual implementation plan. I have a lot of places where we add an item and fetch the primary key. It's not clear to me now if the primary key for each table stays the same and we just add a uniqueness constraint for the primary key and the site identifier (which should already be unique based on the primary key alone).

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Part 1 — High-Level Brainstorming & Architecture Decision

> This part is the "explore the alternatives and decide" work you asked for. Part 2 is the phased implementation plan. Part 3 is Open Questions. Read Part 1 first — the whole implementation plan hinges on the isolation-model decision in §1.3.

## 1.1 The opportunity, restated against what the code actually is

You already deploy behind CloudFront (see `Deploying to AWS.md` — committed architecture: EC2 + RDS + S3 + CloudFront, one Crow binary, one Postgres database). CloudFront already reverse-proxies `/api/*` to the EC2 origin and already injects a per-distribution custom header (`X-Origin-Secret`, read by `endpoints/cloudfront_origin_guard.cpp`). The multi-tenant opportunity is real precisely because **CloudFront can inject a second per-distribution header that names the tenant**, and the server can multiplex on it.

Three facts about the current code shape the entire decision — all confirmed by reading the source:

1. **The database name is already a parameter.** `MakeProductionDatabaseHelper(std::string_view databaseName)` exists today (`sql_util/database_access/database_helper.h:34`). The default no-arg overload bakes in `kDatabaseName = "knottyyoga"` (`sql_util/database_common.h`). Nothing else in the connection path assumes a single database — the name flows straight into the libpqxx connection string in `database_helper_init.cpp`.

2. **There is one libpqxx connection per process, guarded by one process-wide mutex.** `ProductionTransactionProvider::RunInTransaction` takes `std::lock_guard<std::mutex>(connectionMutex_)` for the entire transaction lifetime (`production_transaction_provider.cpp:11-24`), because libpqxx connections are not thread-safe. Every request in the multithreaded Crow app already serializes on this one mutex. **This is the single most important architectural constraint** — it dictates how each isolation model must acquire its connection.

> **Q (Mason): connections aren't thread-safe and can't be shared? Should we use a pool of connections to scale across threads? I'm assuming establishing a connection is expensive and we want to reuse them.**
>
> Correct on every point. A single `pqxx::connection` allows only one open `pqxx::work` at a time — two threads issuing statements on it throws *"Started new transaction … while transaction … was still active."* That's why `RunInTransaction` holds `connectionMutex_` for the whole transaction; today **all DB work in the process serializes on one mutex** (non-DB work — cookie parsing, gzip, post-commit image scaling — still parallelizes). Establishing a connection IS expensive (TCP + TLS + Postgres auth + a server-side backend process), so reuse is right; the code already keeps one long-lived connection rather than creating per-request, and the header flags the pool as the known long-term fix.
>
> **Pooling is the eventual answer, but keep it separate from this tenancy work**, for two reasons:
> 1. **Multi-tenancy already buys a concurrency win for free.** Under Model C each tenant gets its own connection and its own mutex, so tenant A no longer blocks tenant B — strictly better than today. Within a single tenant you still serialize, which is fine for a low-traffic studio app.
> 2. **Pooling and tenancy multiply against the same scarce resource** (`max_connections`). One connection per tenant is `N`; a pool of size `P` per tenant is `N × P`. On a `t3.micro` (~80–100 `max_connections`) you can't have both "dozens of tenants" and a fat pool each without PgBouncer.
>
> So: single connection **per tenant** now (already a concurrency improvement, and the default in Open Question 4); per-tenant pooling is a clean follow-up gated by connection-count, not tenancy. The plan makes the swap cheap — `TenantResources` owns the provider behind `MakeTenantTransactionProvider()` (Phase 2.1/2.2), so a pooled provider can replace the single-connection one **without touching any caller**.

3. **Per-tenant configuration already lives in a per-database table.** Square credentials, the Square environment (sandbox vs prod), SMTP/SES credentials, the sender identity, and the website address are all rows in `config_secrets` (read via `SecretsHelper`, never via the generic CRUD layer). If each tenant has its **own** `config_secrets` table, all of this per-tenant config "just works" with **zero** changes to `SecretsHelper`, `SquareClient`, or `MailHelper` construction logic — only *when* and *with which database* they're built changes.

These three facts strongly bias the decision, as we'll see.

## 1.2 The three isolation models

### Model A — Row-level (`tenant_id` column on every table, one shared schema)
Add a `site_id` / `tenant_id` column to every business table; every query filters `WHERE tenant_id = ?`.

- **Schema work:** add a column to ~70 tables (the per-tenant ones), fold `tenant_id` into every natural-unique constraint, and re-key. (`db_schema/` declares all tables; DDL is generated from `DatabaseInfo`/`TableInfo`/`ColumnInfo` metadata, so the column add is mechanical — but it's ~70 tables.)
- **Query work:** **enormous.** Every table helper, every `DbCrud` call, every business-logic path, and the *entire generic CRUD surface* (`add_item`, `update_item`, `delete_item`, `get_row`, `get_table_rows`, `get_filtered_table_rows`, `get_rows_by_column`, `get_row_by_values`, `get_fk_options`, `resolve_fk_display`) must inject and enforce the tenant filter. The generic CRUD endpoints route through `DatabaseRESTHelper` → `DbCrud`; tenant scoping would have to be pushed into `DbCrud` itself to be safe, and even then FK pickers / display-template joins can leak across tenants if any join forgets the predicate.
- **Isolation:** **weakest.** Correctness depends on every single query getting the predicate right, forever. One forgotten `WHERE` = silent cross-tenant data leak. Postgres Row-Level Security (RLS) policies could backstop this, but the code connects as one role and would need `SET app.tenant_id` per transaction + policies on 70 tables — significant work and still a shared blast radius.
- **Connection model:** unchanged (single connection, single schema). This is its *only* real advantage here.
- **Ops:** one schema to migrate (migrations run once). No per-tenant backup/restore granularity — a bad restore touches everyone.

### Model B — Schema-per-tenant (one Postgres schema per tenant, shared database)
Each tenant gets a Postgres schema (`tenant_acme`, `tenant_knotty`, …) holding the full set of tables. Resolve the tenant per request and issue `SET search_path = <tenant_schema>` at transaction start.

- **Schema work:** **none to the table definitions.** The existing DDL builds the same tables inside each schema. `CreateTables()` would target a schema (set `search_path` before creating, or qualify names).
- **Query work:** **none to table helpers / business logic / generic CRUD.** They keep issuing unqualified table names; `search_path` resolves them to the right schema. This is the big win shared with Model C.
- **Isolation:** **strong** (schema boundary), but depends on `search_path` being set on every transaction. A transaction that forgets it falls back to `public` (or whatever default) — fragile in the same family as Model A but with a *much* smaller surface (one place sets `search_path`, not 70 queries).
- **Connection model:** **fits the single-connection model best.** One connection serves all tenants; you just switch `search_path` per transaction. No connection-count growth. But all tenants still share the one process-wide mutex → no concurrency improvement.
- **Ops:** one database, many schemas. Migrations loop over schemas. Per-tenant restore is awkward (schema-level `pg_dump` is possible but you're restoring into a live shared DB). `config_secrets` is per-schema → per-tenant config works for free.

### Model C — Database-per-tenant (one Postgres database per tenant, same RDS instance)
Each tenant gets its own database (`knottyyoga_acme`, `knottyyoga_knotty`, …) on the same RDS instance. Resolve the tenant per request and route to that tenant's connection.

- **Schema work:** **none.** Each database is built by the existing `CreateAndPopulateDatabases()` path against a different db name. The `MakeProductionDatabaseHelper(databaseName)` overload already exists.
- **Query work:** **none to table helpers / business logic / generic CRUD.** They run against whichever connection they're handed. The change is isolated to *which* `TransactionProvider` an endpoint uses.
- **Isolation:** **maximal.** Cross-tenant data access is physically impossible — a query against tenant A's connection cannot see tenant B's tables. No predicate discipline required, ever. This is the safest possible answer and removes a whole class of "did I scope that query?" bugs from the generic CRUD surface.
- **Connection model:** **the main new work.** The single baked-at-startup connection becomes a small registry of per-tenant `DatabaseHelper` + `TransactionProvider`, created lazily and cached. Each tenant gets its own connection and its own mutex → tenants no longer block each other (a *concurrency improvement* over today). The cost is Postgres `max_connections` (~100 on a t3.micro): one persistent connection per tenant is fine for dozens of tenants; hundreds would need PgBouncer or a bounded pool (a documented future guardrail, not a day-one concern).
- **Ops:** many databases, one RDS instance. Migrations loop over databases. **Per-tenant PITR / snapshot / restore is clean** (you can `pg_dump`/restore one tenant without touching others, and later lift a big tenant onto its own RDS instance with no code change). `config_secrets` is per-database → per-tenant config works for free.
	- > **Q (Mason): is there additional AWS cost for multiple databases?**
	  >
	  > No direct per-database fee. RDS bills for the **instance**, not for logical databases — `CREATE DATABASE knottyyoga_acme` on your existing instance costs $0 extra by itself. You pay for instance compute (hourly, by instance class), allocated storage (GB-month, = total data across all DBs), backup storage (snapshot size beyond the free allotment), and I/O — all of which scale with **aggregate data and load**, not with the number of databases. The expensive alternative is a *separate RDS instance per tenant* (each carries its own hourly compute charge); database-per-tenant on one shared instance avoids that while still giving clean per-tenant `pg_dump`/restore and a trivial path to lift a heavy tenant onto its own instance later. The real ceiling is `max_connections` (memory-bound, ~80–100 on `t3.micro`), which ties back to §1.2's connection question and Open Question 4 — a scaling limit managed with PgBouncer, not a billing line item.

## 1.3 Decision matrix and recommendation

| Criterion | A: row-level `tenant_id` | B: schema-per-tenant | C: database-per-tenant |
|---|---|---|---|
| Table-definition changes | ~70 tables + constraints | none | none |
| Query/CRUD changes | **massive** (every query + generic CRUD) | none | none |
| Cross-tenant leak risk | **high** (per-query discipline forever) | low (one `search_path` per txn) | **none** (physical) |
| Per-tenant config (Square/mail/site) | rework `SecretsHelper` to be tenant-keyed | free (own `config_secrets`) | free (own `config_secrets`) |
| Connection model change | none | small (`search_path` hook) | **medium** (per-tenant registry) |
| Concurrency vs today | same (one mutex) | same (one mutex) | **better** (mutex per tenant) |
| Connection-count pressure | none | none | grows with tenant count |
| Per-tenant backup / restore / PITR | none | awkward | **clean** |
| Lift a tenant to dedicated infra later | hard | medium | **trivial** |
| Onboard a new tenant | INSERT rows everywhere | create schema + populate | create DB + populate |

**Recommendation: Model C (database-per-tenant) on the shared RDS instance, with Model B (schema-per-tenant) kept as a drop-in fallback behind the same abstraction.**

Why C over B: identical code impact on the lower stack, but maximal isolation, clean per-tenant backup/restore/PITR, a path to lifting a heavy tenant onto its own RDS instance with no code change, and *better* concurrency than today (per-tenant mutex). The only price is connection-count management, which is a non-issue at soft-launch scale and has a well-known fix (PgBouncer) if you ever reach hundreds of tenants.

Why **reject A**: the query/CRUD blast radius and the permanent cross-tenant-leak risk are not worth the one advantage (no connection change), especially given the large generic-CRUD surface that would each need bullet-proof tenant scoping. The Website Makeover doc already reached the same conclusion (Phase 1.4: *"recommend (a)* [separate database per tenant] *for now, defer (b)* [row-level] *until there's a real second customer"* — note their lettering is inverted from this doc's).

**The unifying design insight that makes this safe and small:** isolate the entire change at the *edge*. Introduce two new concepts —

- a **TenantContext** resolved once per request from the CloudFront header, and
- a **TenantResources registry** that maps a tenant to its `{TransactionProvider, SecretsHelper, SquareClient, MailHelper, website/CORS config}`, built lazily and cached —

and have endpoints pull their `TransactionProvider` (and helpers) from the request's resolved tenant instead of from global singletons. The **table-helper layer, business-logic layer, db_schema layer, migration framework, and SQL are untouched.** Models B and C differ only inside one method (the tenant `TransactionProvider`'s connection acquisition: a different `dbname` for C, a `SET search_path` for B). That makes the isolation mechanism a pluggable detail and lets us build/test everything else once.

## 1.4 Answering your primary-key question directly

> *"It's not clear to me whether the primary key for each table stays the same and we just add a uniqueness constraint for the primary key and the site identifier… I have a lot of places where I add an item and fetch the primary key."*

**Under the recommended Model C (and under Model B), this concern disappears entirely.** Every table keeps its existing `BIGSERIAL`/`BIGINT GENERATED` surrogate primary key (every sampled table uses `AddColumnPrimaryKey(..., DB_TYPE_BIGSERIAL)` — `classes`, `products`, `people`, `sessions`, `purchases`, etc.). Each tenant has its **own** copy of the table with its **own** sequence, so:

- The PK definition does **not** change.
- You do **not** need a composite `(tenant_id, id)` key.
- `AddRowToTableFetch{Int,Int64}PrimaryKey` (which uses `INSERT … RETURNING <pk>`, `database_crud_helpers.cpp`) is **unchanged** — it returns the surrogate id from that tenant's sequence. All "add an item and fetch the primary key" call sites keep working with no edits. This is one of the strongest reasons to prefer C/B over A.

For completeness, **had we chosen Model A** (we did not), the answer would be: keep the same `BIGSERIAL` PK (the global sequence already makes `id` unique on its own, so no composite PK is needed), add a `tenant_id` column, and fold `tenant_id` into each table's **natural** uniqueness constraints — e.g. `people.email UNIQUE` → `UNIQUE (tenant_id, email)`, `products.code` → `(tenant_id, code)`, `sessions.uuid` → `(tenant_id, uuid)` — so two tenants can each have `alice@example.com`. FKs would still reference the surrogate `id` alone. `RETURNING` still returns `id`; you'd additionally have to inject `tenant_id` into every INSERT. This is exactly the "massive query/CRUD changes" cost that makes A unattractive.

## 1.5 What stays global vs. becomes per-tenant

From the schema audit, classifying every category:

**Stays GLOBAL (one logical definition, but physically duplicated per tenant DB under Model C):**
- All 12 `admin_*` metadata tables (`admin_column_data_info`, `admin_column_friendly_names`, `admin_table_friendly_names`, `admin_table_display_template`, `admin_top_level_tables`, `admin_nested_tables`, `admin_table_permissions`, `admin_column_redactions`, `admin_enums`, `admin_enum_values`, `admin_column_enums`, `admin_alerts`) — they describe the app's table structure / UI and are seeded identically by `Populate*`.
- The permission/role **catalog**: `permissions`, `roles`, `role_permissions`, `permission_implications` (the fixed set of permission/role types the app defines), plus `allowed_tables` and the table↔permission map.

Under Model C these are simply re-seeded in each tenant DB by the existing `CreateAndPopulateDatabases()` path — no new mechanism. When the *catalog itself* changes (a new permission, new admin column metadata), a migration applies it to every tenant DB via the per-tenant migration loop (Phase 5.2).

**Becomes PER-TENANT (already per-DB under Model C, no schema change):**
- All business data: `people`, `classes`, `products`, `bookings`, `purchases`, `payments`, `entitlements`, `event_sessions`, `sessions`, `role_assignments`, and the rest.
- `config_secrets` — and therefore **every per-tenant setting it holds**: `square_access_token`, `square_environment`, `mail_*` (SMTP/SES creds + sender name/address), `website_address`, `website_address_login`, `production_mode_on`, and the email-subject strings.

**New GLOBAL control-plane state (the one genuinely new concept):**
- A small **control database** (`honuware_control`, per Open Question 3) with a `tenants` table mapping the CloudFront site key → database name + display metadata + status. This is the registry the server consults to resolve and route tenants. (Details in Phase 1.)

## 1.6 What is global today and must change (the singleton inventory)

Confirmed process-wide singletons that embed a single-tenant assumption and must move behind the per-tenant registry (or be split global/per-tenant):

1. **`DatabaseHelper` + `TransactionProvider`** — one connection baked at startup (`main.cpp`). → per-tenant registry (Phase 2).
2. **`SecretsHelper`** — built once from the single DB (`main.cpp`). → per-tenant, built from the tenant DB (Phase 4.1).
3. **`SquareClient`** — built once from the single token (`main.cpp`). → per-tenant from tenant secrets (Phase 4.2).
4. **`MailHelper`** — built once (`main.cpp`). → per-tenant from tenant secrets (Phase 4.3).
5. **`ServerConfig` singleton** — holds prod-mode (genuinely global) **and** CORS origin / website address (per-tenant). → split: global `DeploymentConfig` vs per-tenant site config (Phase 4.4).
6. **Email templates** — ~15 mail builders hardcode `"Knotty Yoga"` / `"Knotty Yoga and Spa"` in HTML (`business_logic/payment/*_mail.cpp`, `business_logic/scheduling/*_mail.cpp`, `business_logic/auth/person_verify_mail.cpp`). → parameterize by tenant branding (Phase 4.5).
7. **Scheduler** (`knottyyoga_helper`) — logs in once as `scheduler@knottyyoga.local` and calls admin endpoints with no tenant context. → loop over tenants, send the site header per tenant (Phase 6).
8. **Frontend branding** — logo, footer address/email/tagline, About text are hardcoded (`header.component.ts`, `footer.component.html`, `shared/constants/about.ts`). → fetched at boot from `/api/site_info` (Phase 7).

**Stays global, no change:** `PORT`, `KNOTTYYOGA_LOG_DEST`, `KNOTTYYOGA_TRUST_PROXY`, `KNOTTYYOGA_ORIGIN_SECRET` (origin protection is deployment-wide, not tenant identity), the at-rest `KNOTTYYOGA_SECRET_KEY` master key, the HTTP client library, and the thread pool.

## 1.7 Request flow, end to end (target design)

```
CloudFront distribution for tenant "acme"
  origin custom headers on /api/*:
    X-Origin-Secret: <deployment-wide secret>     (existing, unchanged)
    X-Honuware-Site: acme                          (NEW: per-distribution tenant key)
        │
        ▼
EC2 Crow server (one process, all tenants)
  middleware: CloudFrontOriginGuard (unchanged) → CookieParser → CORS → CsrfGuard → SecurityHeaders
        │
        ▼
  Endpoint handler:
    EndpointAuthHelper::Initialize()
      1. read X-Honuware-Site header                            ── NEW
      2. TenantResolver: site key → TenantContext{tenantId, dbName, ...}   ── NEW (control DB, cached)
      3. TenantResources registry: get-or-build {provider, secrets, square, mail, siteConfig}  ── NEW
      4. session.InitializeFromCookie(tenantProvider)           ── now runs against the TENANT db
        │
        ▼
  Business logic / table helpers / generic CRUD
    run against the tenant's TransactionProvider  ── UNCHANGED CODE, tenant-correct by construction
```

Cookies are naturally tenant-scoped because each tenant has its own domain (its own CloudFront distribution); the `session_token` only resolves inside that tenant's `sessions` table. Health (`/api/health`) stays tenant-agnostic and is already allow-listed in the origin guard.

## 1.8 Componentization impact — how [[Splitting the server up into components]] reshapes this plan

The componentization plan is now fully decided (all 12 of its questions resolved) and sequenced **ahead** of this project: componentize in-place (its Phases 1–3) → extract to the public `honuware` repo on GitHub (its Phases 4–5) → **this multi-tenant conversion** → first deploy. That ordering changes where this plan's code lives, what several things are named, and a few real pieces of the design. This section is the delta; the phase text in Part 2 has been updated to match.

### 1.8.0 As-built status (7/21/2026): the componentization + extraction are COMPLETE — this plan is next in the sequence

Everything §1.8 predicted in future tense now **exists**. The ground truth this plan executes against:

- **The honuware server components live in their own public repo**: `github.com/honuware/server_components` (local checkout `C:\Users\mason\source\repos\server_components`), seven CMake targets (`honuware_foundation`, `honuware_data`, `honuware_services`, `honuware_square`, `honuware_platform`, `honuware_testing`, `honuware_tests`) laid out under `components/<layer>/` with per-component include roots and a build-enforced layer DAG (`cmake/honuware_layering.cmake` + `tools/check_include_layering.py`). **Standalone GitHub Actions CI is green** (Linux, `gcc:14.2.0` container + Postgres 13.1 service, full component suite with a test-count floor), and a local Linux test client (`server_components/docker/`) reproduces CI from Windows against the existing `knotty-net` Postgres.
- **knottyyoga consumes honuware via CMake FetchContent pinned to a SHA** (top-level `server/knottyyoga_server/CMakeLists.txt`); the local `components/` tree is gone. **Cross-repo co-dev workflow** (the loop this whole plan runs in): the app's Linux docker client (`server/docker_project/`) mounts the local `server_components` checkout and builds with `-DFETCHCONTENT_SOURCE_DIR_HONUWARE=/honuware` **by default**, so editing both repos and building as one requires zero setup; when a slice stabilizes → [Mason] push honuware (its CI must go green) → re-pin the app's `GIT_TAG` SHA → [Claude] verify the pinned SHA (`HONUWARE_SRC=none` docker build+test) → [Mason] commit both. Division of labor (amended 7/21/2026): **Claude runs all Linux-docker builds/tests** (both repos' clients); Mason does Windows/VS verification (rare) and every git write; Claude uses git read-only for inspection.
- **Every §1.8.3 hand-off item landed** during componentization, so the seams this plan plugs into are real code today:
    1. `Mail::TenantBranding { studioName; senderName; senderAddress; websiteUrl; }` in `components/services/util/mail/tenant_branding.{h,cpp}`, with `::Mail::LoadSenderAddress(secrets, txn)` / `::Mail::LoadTenantBranding(...)` — the **framework** mail builders (`person_verify_mail`, `quick_account_welcome_mail`) are already branding-parameterized, and all ~37 production From-address sites go through `LoadSenderAddress`. Phase 4.5 has only the app-side (payment/scheduling) builders left.
    2. `ScheduledJob` is fully self-describing (`serverUrl`, `serviceAccountEmail`, `serviceAccountPassword`, `headers` vector — its header comment even names the tenant plan as the intended user) and the engine maintains one authenticated `ApiClient` per distinct `(serverUrl,email,password)` target, with extra headers surviving the 401 re-login retry. **Note: the engine was never physically extracted to honuware** — it lives app-side in `src/scheduler/` (target `knotty_yoga_scheduler`, written app-agnostic in namespace `Scheduler`); that's fine for tenancy, which only needs the data-driven job list. The knottyyoga catalog (`src/scheduler/knottyyoga_job_catalog.{h,cpp}`, `BuildKnottyYogaJobs(target, intervals)`) already copies `target.headers` onto every job — `main.cpp` just never populates them yet. Phase 6 is now mostly "build catalog × tenants."
    3. The migration framework is namespaced: `MigrationRunner` + `migration_namespace` (`kFrameworkMigrationNamespace="honuware"`, `kAppMigrationNamespace="app"`) + `framework_migrations` live in `components/platform/business_logic/migration/`; the app composes `BuildAllMigrations() = framework ++ app` in `src/business_logic/migration/`.
    4. `GlobalDatabaseTestSupport::Initialize(const DbSchema::DatabaseInfo&)` takes the composed schema, and **`EnsureNamedDatabase(name, DatabaseInfo)`** (create-once, cached) already exists — the exact seam Phase 3.4's physical-isolation suite uses.
    5. `WebApp` carries a typed service registry (`SetService<T>` / `GetService<T>`); Square is registered app-side from `main.cpp` and surfaced via the app-derived `AppEndpointAuthHelper::GetSquareClient()` (`src/endpoints/app_endpoint_auth_helper.*`). The framework `EndpointAuthHelper` (`components/platform/endpoints/`) exposes session/cookies/secrets/mail/db only — precisely the façade `TenantResources` extends.
    6. Schema assembly is split: `MakeFrameworkTables(dbInfo)` (platform `db_schema`) + `MakeAppTables(dbInfo)` (app), composed by the app's `MakeDatabaseInfo(name)`; secrets defaults are registered (`Secrets::Values::FillInSecretsStringView` framework + `App::FillInAppSecretDefaults` app, incl. brand values); the service account is parameterized (`EnsureSchedulerServiceAccount(..., schedulerEmail, password)` + app-side `app_service_account_config.h`); framework endpoints self-anchor via `Endpoints::RegisterFrameworkEndpoints()` (`components/platform/endpoints/register_framework_endpoints.*`).
    7. Env vars: the six brandable vars are renamed `HONUWARE_*` (legacy `KNOTTYYOGA_*` fallback via `Util::GetEnvWithFallback`). **Known leftover:** the DB connection vars are still `KNOTTYYOGA_DB_HOST/_PORT/_USER/_PASSWORD/_NAME/_SSLMODE/_SSLROOTCERT` (logged follow-up from the extraction CI work) — folded into this plan's Phase 0.
- **Not done there, absorbed here:** the splitting plan's Phase 5 example server was demoted (CommunityFinder is the proving consumer, §1.10); its Phase 4.4 Conan packaging stays deferred; and the `create_database.cpp` framework/app split (CommunityFinder hand-off item 2) is folded into this plan's Phase 5.1 where tenant provisioning needs the same mechanism.
- **Confirmed absent (this plan builds it):** no `TenantContext` / `TenantResolver` / `TenantResources` / `honuware_control` / `site_info` code exists in either repo yet (the only `X-Honuware-Site` mentions are forward-looking *examples* in three scheduler test files exercising the `ScheduledJob.headers` mechanism).
- **One fact that shrinks Phase 3 dramatically (verified 7/21/2026):** `EndpointAuthHelper` already exposes `GetTransactionProvider()` / `GetSecretsHelper()` / `GetMailHelper()` / `GetDatabaseHelper()` / `GetCookieManager()` + a generic `GetService<T>()`, and **180 of 182 app endpoint `.cpp` already acquire their provider through `endpointAuthHelper.GetTransactionProvider()`** (only ~3 sites use `webApp->GetTransactionProvider()`, e.g. the tenant-agnostic `health.cpp`). Re-pointing the helper's internals at the resolved tenant's resources therefore converts essentially the whole endpoint surface at once — Phase 3.4 is an accessor re-point + a small audit, not a 260-endpoint migration.

### 1.8.1 Tenancy becomes a honuware framework feature

The component architecture already reserves a slot for `business_logic/tenancy/` inside `honuware_platform`. Making that explicit:

- The `tenants` control schema, `TableHelpers::Tenants`, `TenantContext`, `TenantResolver`, `TenantResourceRegistry`, the edge resolution in `EndpointAuthHelper`, the provisioning/migration loops, and `/api/site_info` are all **framework code** — zero yoga specifics, and every honuware consumer (the friends' sites included) gets multi-site support for free.
- Their tests ship in the honuware repo and run under its GitHub Actions CI (Linux + Postgres service container).
- **Single-tenant consumers must pay ~zero ceremony.** Make `TenantResolver` an interface with two implementations: the control-DB resolver (multi-tenant deployments) and a `FixedTenantResolver` that serves one configured `TenantContext` with **no control DB and no CloudFront header at all**. The friends' sites and local dev both run the fixed resolver; the knottyyoga deployment runs the control-DB one. This subsumes (and upgrades) the dev escape hatch in Phase 3.1.
- The control DB is **per deployment** (one per RDS instance/consumer), not a shared global service — `honuware_control` is just its default name.

### 1.8.2 Repo map — where each phase's work lands

| Phase | Lands in |
|---|---|
| 0 — co-dev setup + `HONUWARE_DB_*` rename | honuware `data` (+ CI/docs) |
| 1 — control plane (schema, helper, resolver) | honuware `platform` (`components/platform/{db_schema, sql_util/table_helpers, business_logic/tenancy}`) |
| 2 — pooled provider + registry | honuware `data` (pool) + `platform` (tenancy) |
| 3 — request-edge resolution | honuware `platform` (web core) + `testing` (EndpointTestHelper); mode switch in app `main.cpp` |
| 4.1/4.3/4.4 — secrets, mail, config | honuware `services`/`platform`; **4.2 Square wiring → app** (§1.8.3 #1) |
| 4.5 — email branding | app only (framework builders already done — §1.8.0 #1) |
| 5 — create_database split, provisioning & migrations | framework halves + `ProvisionTenant`/`MigrateAllTenants` in honuware `platform`; CLI + composed `DatabaseInfo` stay in the app's `database_helper` |
| 6 — scheduler | engine already done (self-describing jobs); tenant iteration + job catalog app-side `scheduler/main.cpp` |
| 7 — frontend branding | `/api/site_info` endpoint → honuware `platform`; all Angular work → app `ui/` (`@honuware/ui` untouched — §1.9) |
| 8 — infra | ops (unchanged) |

Practical consequence: most of this project is **cross-repo work** — honuware and knottyyoga edited together with `FETCHCONTENT_SOURCE_DIR_HONUWARE` pointed at the local checkout, and a honuware version bump when each slice stabilizes. The component plan's Q6 discussion priced this in ("a few weeks of editing both repos together"); the phases here are already sliced so each is a clean bump point.

### 1.8.3 Design changes forced (or gifted) by the component boundaries

1. **`TenantResources` gains an app extension point (Square).** Component Phase 1.6 removes `SquareClient` from the framework `EndpointAuthHelper` façade: the framework exposes session/cookies/db/secrets/mail, and the app supplies a derived accessor for Square. Tenancy follows the same shape — the framework `TenantResources` holds `{DatabaseHelper, TransactionProvider, SecretsHelper, MailHelper, site config/branding}`, and `TenantResourceRegistry` is constructed with an **app-supplied factory** (`std::function<std::shared_ptr<TenantResources>(const TenantContext&)>`, wired in `main.cpp` — the composition root the component plan's Phase 3.2 establishes). knottyyoga's factory returns a derived `KnottyTenantResources` adding the per-tenant `SquareClient` (built from tenant secrets using the `honuware_square` client component); the app-derived endpoint helper preserves the `helper.GetSquareClient()` call-site ergonomics. The framework never references Square. (Phases 2.1, 3.2, 4.2 updated.)
2. **The scheduler engine stays tenancy-agnostic.** After component Phase 1.7 the engine consumes a data-driven `std::vector<ScheduledJob>` and links foundation only — it cannot read the control DB, and shouldn't. Multi-tenancy is expressed in the *job list*: `ScheduledJob` grows per-job request headers and login credentials; app-side `scheduler/main.cpp` reads the active-tenant list from the control DB (the app-side main keeps app-layer deps anyway) and builds catalog × tenants, each job carrying `X-Honuware-Site: <site_key>`. (Phase 6 updated.)
3. **Provisioning and migrations take composed inputs.** Component Phase 2.3 splits `MakeDatabaseInfo()` into framework-table + app-table assembly composed by the app. So the framework provisioning function is shaped `ProvisionTenant(controlProvider, TenantSpec, const DatabaseInfo&)`, and the per-tenant migration loop takes a composed migration list — **framework migrations and app migrations are two streams, applied framework-first, tracked in `schema_migrations` under separate id namespaces** so honuware upgrades and app changes can't collide. The CLI surface stays in `knottyyoga_database_helper`. (Phases 5.1/5.2 updated.)
4. **Framework mail-builder branding moves earlier.** `person_verify_mail.cpp` (and any other framework mail body) hardcodes "Knotty Yoga" today but is slated to move into the *public* honuware repo — it can't ship with the brand baked in. The `TenantBranding` parameterization of the **framework** builders therefore becomes a prerequisite of the extraction and should be executed during the componentization work (its Phase 1.3 secrets-defaults registration is the natural mechanism; a single-tenant consumer registers one branding). Phase 4.5 here then covers only the app-side builders (payment/scheduling) plus sourcing the branding struct per-tenant. **Action: fold this item into the component plan's Phase 1 when executing it.**
5. **Naming.** Everything new in this plan is honuware-branded (Q2/Q3 below): header `X-Honuware-Site`, control DB `honuware_control`, env vars `HONUWARE_CONTROL_DB_NAME` and `HONUWARE_FIXED_SITE_KEY`. Existing env vars this plan references (`KNOTTYYOGA_ORIGIN_SECRET`, `KNOTTYYOGA_TRUST_PROXY`, `KNOTTYYOGA_SECRET_KEY`, `KNOTTYYOGA_DEV_CORS_ORIGIN`, `KNOTTYYOGA_ALLOW_DESTRUCTIVE`) belong to code that moves into honuware and will be renamed `HONUWARE_*` (new name read first, old name as fallback) per the component plan's convention — this plan uses the new names.
6. **What componentization hands tenancy for free.** Parameterized database name (component 1.2), secrets defaults via registration (1.3), the de-Square'd `EndpointAuthHelper` (1.6), the data-driven scheduler (1.7), the framework/app `DatabaseInfo` split (2.3), `GlobalDatabaseTestSupport` taking composed inputs (2.6), and the "everything app-specific enters at the composition root" audit (3.2). These were previously implicit prep inside this plan's phases; by the time this plan executes they already exist, which shrinks Phases 2–4 noticeably.

## 1.9 Frontend componentization impact — how [[Componentizing the frontend]] reshapes Phase 7

The frontend componentization completed through its Phase 4 on 7/21/2026: **`@honuware/ui` is a published npm package** (`@honuware/ui@0.1.1`, public registry with provenance; repo `github.com/honuware/web_components`, local checkout `C:\Users\mason\source\repos\honuware-web-components`) with eight secondary entry points (`foundation`, `access`, `controls`, `photos`, `auth`, `crud`, `square`, `testing`). The knottyyoga app consumes the **exact-pinned published package** — the library source no longer lives in the app repo. Cross-repo co-dev = sibling checkout + the commented `@honuware/ui/*` paths override in `ui/tsconfig.json` (the frontend's `FETCHCONTENT_SOURCE_DIR` analog); releases = bump `projects/honuware-ui/package.json` → tag `vX.Y.Z` → CI publishes → `npm install @honuware/ui@X.Y.Z --save-exact` in the app.

What this changes for Phase 7 (and confirms):

1. **The auth pages are library pages and provably brand-free.** `hw-login`/`hw-register`/`hw-verify` ship in `@honuware/ui/auth` with spec-pinned no-brand assertions (`not.toContain('knotty')`). Phase 7's de-branding scope is therefore **only the app shell**: `ui/src/app/shared/components/header/header.component.ts` (logo asset `assets/svg/KnottyYoga_logo_white.svg` + `KNOTTY_YOGA_LOGO_ALT_TEXT`), `footer.component.html` (address block, `info@knottyyoga.com` mailto, `© Copyright KnottyYoga`), and `shared/constants/about.ts`.
2. **`HONUWARE_BRANDING` exists as a decision, not code — keep it that way until needed.** Frontend Q12 decided a static `HONUWARE_BRANDING` token (display name, logo URL, website URL) whose provider later becomes Phase 7's `SiteConfigService`; it was deliberately **not** created because no library surface consumes branding today (no-premature-code). Phase 7 builds the app-side `SiteConfigService` first; the token is introduced only if/when a library surface needs it.
3. **`getSiteInfo()` is an app-side `ServerAccess` addition, not a library-access change.** Under Q10's shared-bundle strategy, `/api/site_info` is consumed by the app shell — so the method lands on the app's `ServerAccess` interface + `ServerAccessNetwork` + `ServerAccess.mock.ts` + proxy (+ mock spec, per the standing frontend testing rule). The library's narrow seams (`CrudAccess`/`AuthAccess`/`PhotoAccess`) are untouched; a `@honuware/ui` `SiteAccess` seam can be added later if CommunityFinder or the showcase wants runtime branding.
4. **The SPA never sets `X-Honuware-Site`** — CloudFront injects it on `/api/*` — so the published library needs **zero tenant awareness** (already true; stays true).

## 1.10 CommunityFinder factoring — the second consumer this plan must not break (and quietly serves)

CommunityFinder (plan: `C:\Users\mason\Documents\Obsidian\CommunityFinder\Claude\Setting up the project.md`) is **a separate honuware consumer app** — its own repo, own deployment, own database, own CloudFront — **not a tenant row on the knottyyoga deployment**. Multi-tenancy (this plan) multiplexes *one app's* deployment across sites; CommunityFinder is a *different app* on the same framework. Its extraction gate (its Phase 0 = splitting-plan 4.1/4.2) is now **satisfied**, so it can bootstrap any time — possibly **concurrently with this plan**. Its own doc already encodes the sequencing: *"CommunityFinder starts on pre-tenancy honuware as a plain single-tenant server; when the tenancy work lands in honuware, single-tenant consumers adopt `FixedTenantResolver` (no control DB, no CloudFront site header) via a version bump — the tenancy plan explicitly designs for this."* That yields four hard constraints on this plan:

1. **`FixedTenantResolver` is a first-class production mode, not a dev shim.** A single-tenant consumer's `main.cpp` must need only a few lines (build one `TenantContext` from its own app config) and **no new required env vars, no control DB, no site header**. Acceptance test for Phase 3: knottyyoga's own dev mode runs exactly this path.
2. **Every slice keeps honuware's standalone CI green.** Tenancy tests compile into `honuware_tests`, so they run in the component repo's own GitHub Actions (Linux + Postgres) — that green, plus knottyyoga's suite, is the per-slice gate. This is what protects a concurrent CommunityFinder bootstrap from breakage; the honuware README's new-consumer checklist gains the resolver wiring lines when Phase 3 lands (a mechanical upgrade note for any consumer mid-bootstrap on an older SHA).
3. **The `create_database.cpp` framework/app split serves both masters.** CommunityFinder's hand-off item 2 asks platform for `CreateFrameworkTables(...)` / `PopulateFrameworkTables(...)` so a new consumer composes instead of copying ~3,000 seed lines; tenant provisioning (Phase 5.1) needs the same callable create+populate path per tenant database. One mechanism, built once, in Phase 5.1.
4. **`HONUWARE_DB_*` rename unblocks both.** CommunityFinder's hand-off item 4 and the extraction CI's logged follow-up are the same fix (the DB connection env vars are still `KNOTTYYOGA_DB_*`); it lands here in Phase 0 since clean env naming also matters for the control-DB configuration.

CommunityFinder's frontend consumes `@honuware/ui` with static branding — nothing in this plan's Phase 7 is on its critical path.

---

# Part 2 — Implementation Plan

Phases are ordered lowest-layer-first. Within each phase, subsections are numbered and also ordered lowest-layer-first. Every code change lists its test work as a checkbox (per the "always add tests" rule). The isolation mechanism (Model C dbname vs Model B search_path) is confined to Phase 2.2 so the rest of the plan is mechanism-agnostic.

**Where code lands (per §1.8.0 as-built reality):** framework code goes in the honuware checkout `C:\Users\mason\source\repos\server_components` under `components/<layer>/…` (compiled into `honuware_data`/`honuware_platform`/`honuware_testing`; tests into `honuware_tests` so they run in honuware's own CI *and* the app suite); app code goes in `knottyyoga/server/knottyyoga_server/src/…` (`knotty_yoga_core`); frontend app code in `knottyyoga/ui/src/…`. Each work item below is tagged **[hw]** (honuware repo), **[app]** (knottyyoga server), or **[ui]** (Angular app).

> **The cross-repo loop (every phase runs this way):**
> 1. Claude edits both trees as one. The **app's Linux docker client already runs in co-dev mode by default**: `server/docker_project/load_container.cmd` mounts the local `server_components` checkout at `/honuware` and passes `-DFETCHCONTENT_SOURCE_DIR_HONUWARE=/honuware` — no config needed. (For Mason's occasional Windows/VS work, the same override goes in `CMakeSettings.json`, per the root CLAUDE.md.)
> 2. **[Claude] gate per slice — build + test in the Linux docker containers:** the app suite via `server/docker_project/build_and_test.sh` (builds `knottyyoga_tests`, which runs **both** bags — app `knotty_yoga_tests` + component `honuware_tests` against the app's composed schema), **and** honuware's standalone suite via `server_components/docker/build_and_test.sh`. Both must be green before a slice is "done".
> 3. At each phase boundary (marked "**⇑ bump point**"): [Mason] pushes honuware (GitHub Actions must go green) → Claude re-pins knottyyoga's `GIT_TAG` to the new SHA → **[Claude] verifies the pinned SHA** with a `HONUWARE_SRC=none` docker build+test (exercises the FetchContent clone, not the local override) → [Mason] commits both repos.
>
> **Build/test/git (amended 7/21/2026):** Claude **builds and runs all C++ tests in the Linux docker containers** (app: `server/docker_project/`; honuware: `server_components/docker/`) — proactively, as the gate for every slice. Claude does **not** build/test on Windows (that disturbs Visual Studio; Linux-green makes Windows-only breakage rare — Mason spot-verifies Windows occasionally). Git: Claude may **inspect** state read-only (`git status`/`log`/`diff`/`show`/`rev-parse`) but performs **no write operations** — add/commit/push/tag/reset are Mason's.

## Phase 0 — Cross-repo setup + prerequisite cleanups (NEW: the on-ramp)

Goal: the two-repo workflow proven live, and the two logged leftovers that tenancy config depends on cleared. Small by design — one gate.

### 0.1 Prove the co-dev loop in the Linux docker clients **[Claude]**
- [x] Run the **honuware** client green: `server_components/docker/build_and_test.sh` — the per-slice framework gate. **DONE 7/21/2026: `EXIT=0`, 1312 tests / 127 suites passed (floor 1000), `[honuware] OK`; test run 32.5s.**
- [x] Run the **app** client green in co-dev mode: `./docker_project/build_and_test.sh` — builds against the local `server_components` checkout via the `/honuware` mount + `FETCHCONTENT_SOURCE_DIR_HONUWARE` override, runs **both** test bags. This is the loop every later phase edits inside; proven before any tenancy code exists so workflow problems can't masquerade as tenancy bugs. **DONE 7/21/2026: `EXIT=0`, log confirms `[knottyyoga] honuware : /honuware (local override)`, 4459 tests / 493 suites passed (floor 3500), `[knottyyoga] OK`; test run ~320s.**
- [x] **For Windows/VS sessions:** `FETCHCONTENT_SOURCE_DIR_HONUWARE=C:/Users/mason/source/repos/server_components` added to `server/knottyyoga_server/CMakeSettings.json`'s `x64-Debug` cache variables (type `PATH`). **DONE 7/21/2026** — so a VS configure/build also uses the local honuware checkout while both repos are mid-change. **To build against the pinned SHA in VS instead, delete this variable and re-configure** (the docker clients toggle this via `HONUWARE_SRC=none` — the ⇑ bump-point check). *(Note: this is a machine-specific absolute path in a VS-local file; if the repo ever moves, update or switch to `${projectDir}/../../../server_components`.)*

> **Reusable non-interactive docker invocation (how Claude drives the clients every slice — `load_container.cmd` opens an interactive shell, so replicate its `docker run` sans `-it`, pointing the command at the script). Run from Git Bash with `MSYS_NO_PATHCONV=1` so `/src` etc. aren't path-mangled.**
> ```bash
> # honuware standalone (framework gate)
> MSYS_NO_PATHCONV=1 docker run --rm --network knotty-net \
>   -v "C:/Users/mason/source/repos/server_components:/src" \
>   -v honuware-conan2:/root/.conan2 -v honuware-linux-build:/build \
>   -e HONUWARE_DB_SSLMODE=disable -w /src \
>   honuware_build:latest bash docker/build_and_test.sh
>
> # app in co-dev mode (local honuware override)
> MSYS_NO_PATHCONV=1 docker run --rm --network knotty-net \
>   -v "C:/Users/mason/source/repos/knottyyoga/server:/src" \
>   -v "C:/Users/mason/source/repos/server_components:/honuware" \
>   -v honuware-conan2:/root/.conan2 -v knottyyoga-linux-build:/build \
>   -e HONUWARE_SRC_DIR=/honuware -e HONUWARE_DB_SSLMODE=disable -w /src \
>   knottyyoga_build:latest bash docker_project/build_and_test.sh
> ```
> For the ⇑ bump-point pinned-SHA check, drop the `/honuware` mount + `HONUWARE_SRC_DIR` (forces the FetchContent clone of the pinned SHA — needs git+network). Redirect to a scratch log + `tail` the end (build output is large; only the test-count + `OK` lines matter). Never run the *same* suite from Windows and Linux at once (each DROP/CREATEs its DB). A filtered run (`--gtest_filter=…`) skips the count floor — use it to iterate fast on one suite.

### 0.2 `KNOTTYYOGA_DB_*` → `HONUWARE_DB_*` rename **[hw]** (the logged extraction follow-up + CommunityFinder hand-off item 4)
- [x] In `components/data/sql_util/database_access/database_helper_init.{h,cpp}`: switched the seven DB connection env vars to a `DbEnvOr(canonical, legacy, default)` helper wrapping `Util::GetEnvWithFallback("HONUWARE_DB_X", "KNOTTYYOGA_DB_X")` (then the string default when neither is set or the value is empty — the exact convention `logging.cpp` uses for `HONUWARE_LOG_DEST`). Header comment block rewritten. **DONE 7/21/2026.**
- [x] Updated the setters/docs: honuware `.github/workflows/ci.yml` env block (`HONUWARE_DB_*`, note rewritten), honuware `docker/load_container.cmd` + app `server/docker_project/load_container.cmd` (`-e HONUWARE_DB_SSLMODE`), honuware `README.md` + `docker/README.md` env docs, and the knottyyoga root `CLAUDE.md` env table (new `HONUWARE_DB_*` + `HONUWARE_VERSION` rows with legacy-fallback column). *(The DB vars are set in `load_container.cmd`/`ci.yml`, not `build_and_test.sh`.)* **Deploy configs deferred to Phase 8.4** (`server.env` / systemd / ECS docs still name `KNOTTYYOGA_DB_*` — legacy fallback keeps them working; SERVER.md + `package/**/README.md` switch there). **DONE 7/21/2026.**
- [x] Tests in `database_helper_init_test.cpp`: `kAllEnvVars` + `EnvScope` now scrub/restore **both** name families; the per-field tests use the canonical `HONUWARE_DB_*` names; added `LegacyNameHonoredWhenCanonicalUnset`, `CanonicalWinsWhenBothSet`, `EmptyCanonicalUsesDefaultIgnoringLegacy`, and a legacy-name `EnvScope` restore test. **DONE.**
- [x] `components/platform/endpoints/health.cpp` `GetBuildVersion()` → `Util::GetEnvWithFallback("HONUWARE_VERSION", "KNOTTYYOGA_VERSION")`; header comment updated; `health_test.cpp` `VersionEnvScope` generalized to both names + added `GetBuildVersionLegacyFallback` / `GetBuildVersionCanonicalWinsOverLegacy`. **DONE.**
- [ ] Optional, fold-in while touching honuware: the logged cosmetic scrub of residual `knotty_yoga_*` mentions in component CMake *comments*. *(Not done — deferred, purely cosmetic.)*
    - **Gate (co-dev, canonical `HONUWARE_DB_SSLMODE`, 7/21/2026): honuware full 1316 tests `[honuware] OK`; app full 4463 (Linux, `EXIT=0`).** **WINDOWS-CAUGHT FIX:** Mason's Windows run flagged one test, `EmptyCanonicalUsesDefaultIgnoringLegacy` — it set an *empty-string* canonical var alongside a real legacy var and asserted the default. That edge isn't portable: POSIX `setenv(name,"")` keeps a present-but-empty var (→ default, on Linux, where it passed), but Windows `_putenv_s(name,"")` **removes** the var (→ it reads as absent → the legacy fallback correctly wins). The **code is correct on both**; the test asserted a platform-dependent quirk. Removed that test (portable precedence stays covered by `LegacyNameHonoredWhenCanonicalUnset` + `CanonicalWinsWhenBothSet` + `EmptyEnvFallsBackToDefault`) and corrected `DbEnvOr`'s comment to say the empty-string edge is deliberately not relied on. Linux re-verified green (1316); **[Mason] re-run Windows to confirm** (the remaining tests are all platform-portable).

**⇑ bump point — DONE except [Mason]'s commit (7/21–22/2026).** [Mason] pushed honuware (Windows + CI green) → HEAD `d6cefe617455975b2b3b994e85100007eebdd8f5` (working tree clean, `origin/master` matches). [Claude] re-pinned the app's `CMakeLists.txt` `GIT_TAG` `3f17b07…` → `d6cefe6…`, then **verified the pinned SHA**: fresh build volume `knottyyoga-linux-build-pinned`, **no** `/honuware` override → FetchContent cloned `d6cefe6` from GitHub and built it → **app 4463 tests `[knottyyoga] OK`, exit 0** (`[knottyyoga] honuware : pinned SHA`). **Remaining: [Mason] commit both repos** — knottyyoga working tree has the `GIT_TAG` re-pin + the Phase 0/0.2 app-side files (`CMakeSettings.json`, `docker_project/load_container.cmd`, root `CLAUDE.md`). **→ Phase 0 COMPLETE; next is Phase 1.** *(Kept `knottyyoga-linux-build-pinned` as the dedicated pinned-mode build tree for future bump-point checks — it must never get the `FETCHCONTENT_SOURCE_DIR_HONUWARE` override, or it stops being a valid pinned check.)*

## Phase 1 — Control plane & tenant model (lowest layer: new control DB + registry)

Goal: a queryable source of truth that maps a CloudFront site key to a tenant + its database name, plus an in-memory resolver. No request path touches this yet. **Lands in:** `honuware_platform` — new directory `components/platform/business_logic/tenancy/` plus one table pair in platform's `db_schema`/`table_helpers`.

**DONE (7/22/2026) — gated green: honuware full 1334 tests (+18), app co-dev full 4481 (+18), both `[…] OK`, exit 0.** All new code in `honuware_platform` (`db_schema/tenants.*` + `make_control_database_info.*`; `sql_util/table_helpers/tenants.*`; `business_logic/tenancy/{control_database,tenant_context,tenant_resolver}.*`); `tenants` composed into both test mains. **Three refinements surfaced during implementation:**
- **(a) `TenantContext` gained a `status` field** (beyond the plan's field list) — 1.4 requires the resolver convey suspended-vs-unknown, which needs status on the context. `FixedTenantResolver` sets it `active`.
- **(b) 1.3 test strategy corrected — the planned `EnsureNamedDatabase(control-only schema)` path is INFEASIBLE.** The harness's `SetupAllTables` always runs the *after-tables* stored procs (`get_admin_alerts_in_window()` → `SETOF admin_alerts`), which reference framework tables the minimal control schema deliberately lacks → it would fail to stand up. So `EnsureControlDatabase()`/`MakeControlDatabaseHelper()` (which create/connect a *real* database) are validated **live in Phase 5.1** (`--create-tenant`); 1.3's unit test is **env-name resolution only**. The control schema's validity is proven instead by `make_control_database_info_test` (structure/columns/defaults) + `tenants_test` (the table working in the composed primary test DB). Production `EnsureControlDatabase` is correct-by-design (it only creates `now_us()` + the two tables, never the after-tables procs).
- **(c) `HONUWARE_DB_NAME` override footgun noted in code** — `MakeControlDatabaseHelper` wraps `MakeProductionDatabaseHelper`, which honors the `HONUWARE_DB_NAME` override; a control-DB deployment doesn't set it, and a dedicated always-force-the-name variant is a Phase 5.1 concern (multi-DB process).

**⇑ bump point DONE except [Mason]'s commit (7/22/2026):** honuware Phase 1 committed + pushed → SHA `280551efe9ba5054ecd870f04099af5000bf9ce4` (CI green). [Claude] re-pinned the app's `GIT_TAG` `d6cefe6…` → `280551e…` and **verified the pinned SHA**: fresh-clone build on the dedicated `knottyyoga-linux-build-pinned` volume (no `/honuware` override → FetchContent cloned `280551e` from GitHub) → **app 4481 tests `[knottyyoga] OK`, exit 0**. **Remaining: [Mason] commit the knottyyoga side** — `CMakeLists.txt` re-pin + the Phase 1 `test/src/main.cpp` edit + the still-uncommitted Phase 0/0.2 app-side files (`CMakeSettings.json`, `docker_project/load_container.cmd`, root `CLAUDE.md`).

### 1.1 Control-database schema (`db_schema` layer) **[hw]**
- [x] Add `components/platform/db_schema/tenants.{h,cpp}` defining the `tenants` table with `BIGSERIAL id` PK and columns: `site_key` (UNIQUE — the value CloudFront sends in `X-Honuware-Site`), `database_name` (UNIQUE), `display_name`, `status` (`active`/`suspended`, default `active`), `max_connections` (BIGINT default 1 — the per-tenant pool knob, Q4), `created_us`, `updated_us`. Follow the column-constant + `MakeTenantsTable(DatabaseInfo&)` pattern of the neighboring framework tables.
- [x] Add `components/platform/db_schema/make_control_database_info.{h,cpp}` — `DbSchema::MakeControlDatabaseInfo(std::string_view controlDbName)` assembling the control DB's schema: **`tenants` + `schema_migrations`** (for the control DB's own evolution) and nothing else. **Deliberately NOT part of `MakeFrameworkTables`** — tenant databases must not carry a `tenants` table; the control DB is a separate, minimal schema.
- [x] Register both pairs in `components/platform/db_schema/CMakeLists.txt` (`target_sources(honuware_platform …)`; tests into `${HONUWARE_TESTS_TARGET}`).
- [x] `make_control_database_info_test.cpp`: control info contains exactly {`tenants`, `schema_migrations`}; `MakeFrameworkTables` does **not** contain `tenants`; column/constraint assertions for the `tenants` table (mirroring `make_database_info_test.cpp`'s style).
- [x] **Test-DB availability:** both test mains compose the control tables into their primary test schema so the table-helper/resolver tests below run against the ordinary harness — `server_components/test/main.cpp` (framework `DatabaseInfo` + control tables) and knottyyoga `test/src/main.cpp` (`MakeDatabaseInfo(kTestDatabaseName)` + control tables). Explicit composition per the house style; no harness change needed.

### 1.2 Tenants table helper (`table_helpers` layer) **[hw]**
- [x] Add `components/platform/sql_util/table_helpers/tenants.{h,cpp}` — `TableHelpers::Tenants` with a `TenantRow { id; siteKey; databaseName; displayName; status; maxConnections; }` and `LookupBySiteKey(transaction, siteKey) → std::optional<TenantRow>`, `ListActive(transaction) → std::vector<TenantRow>`, `Insert(transaction, row) → int64_t`, `SetStatus(transaction, siteKey, status)`. Prefer `DbCrud` helpers (`GetRow`/`GetRowsByValues`/`AddRowToTableFetchInt64PrimaryKey`); custom SQL only where `DbCrud` can't express it.
- [x] `tenants_test.cpp`: insert/lookup round-trip, unknown site key → empty, duplicate `site_key`/`database_name` throws (UNIQUE), `ListActive` filters `suspended`, `SetStatus` transitions. Register helper + test in `components/platform/sql_util/table_helpers/CMakeLists.txt`.

### 1.3 Control-database access + bootstrap **[hw]**
- [x] Add `components/platform/business_logic/tenancy/control_database.{h,cpp}` — `Tenancy::ControlDatabaseName()` (env `HONUWARE_CONTROL_DB_NAME`, default `honuware_control`), `MakeControlDatabaseHelper()` (wraps `MakeProductionDatabaseHelper(ControlDatabaseName())`), and `EnsureControlDatabase()` — create-the-database-if-absent + create/refresh its tables from `MakeControlDatabaseInfo`, reusing the same no-database-helper + DDL machinery `CreateAndPopulateDatabases()` uses today (framework-callable; the app CLI invokes it in Phase 5.1, and `--create-tenant` auto-ensures it).
- [x] New dir wiring: `components/platform/business_logic/tenancy/CMakeLists.txt` + `add_subdirectory(tenancy)` in `components/platform/business_logic/CMakeLists.txt` (pattern: the `migration/` sibling).
- [x] Tests: env-name resolution (default + override, `EnvScope` pattern); `EnsureControlDatabase` exercised via the harness's `EnsureNamedDatabase("test_honuware_control", MakeControlDatabaseInfo(...))` create-once pattern (asserts the physical control schema stands up + is idempotent).

### 1.4 Tenant resolver (business-logic layer, no request coupling yet) **[hw]**
- [x] Add `components/platform/business_logic/tenancy/tenant_context.h` — immutable `Tenancy::TenantContext { int64_t tenantId; std::string siteKey; std::string databaseName; std::string displayName; int64_t maxConnections; }` (carrying `maxConnections` so the Phase 2 provider factory needs no second lookup).
- [x] Add `components/platform/business_logic/tenancy/tenant_resolver.{h,cpp}` — `TenantResolver` interface with `Resolve(siteKey) → std::optional<TenantContext>` + `Invalidate(siteKey)` / `InvalidateAll()`, and the two implementations (§1.8.1): **`ControlDbTenantResolver`** (holds the control `TransactionProviderPtr`; cache-on-read map + mutex; suspended tenants resolve with status so the edge can distinguish unknown vs suspended) and **`FixedTenantResolver`** (returns one configured `TenantContext`; `Resolve("")` and `Resolve(itsOwnKey)` succeed, any *other* key returns empty so the edge can reject contradictions — §1.10 #1: this is CommunityFinder's and dev's production path). No TTL — invalidation is wired into provisioning (Phase 5.1).
- [x] `tenant_resolver_test.cpp`: resolves a seeded control row; caches (second lookup doesn't re-query — count via a wrapped/counting provider); unknown key → empty; `Invalidate` forces re-query; suspended → resolved-with-status; `FixedTenantResolver` empty-key + own-key + foreign-key behaviors.[[Consulting and Extra Revenue]]

## Phase 2 — Per-tenant connection & transaction routing (database-access layer)

Goal: turn "one connection baked at startup" into "a lazily-built, cached per-tenant `TransactionProvider`," behind an interface that hides Model C vs B. **Lands in:** `honuware_data` (the pooled provider) + `honuware_platform` (registry, factory).

### 2.1 Pooled transaction provider (data layer — lowest first) **[hw]**
- [x] Add `components/data/sql_util/database_access/pooled_transaction_provider.{h,cpp}` — a `TransactionProvider` implementation that owns a **bounded lazy pool of connections** for ONE database (created from a `DatabaseHelper`), default max **1** (Q4). Shape: free-list + condition variable; `RunInTransaction` acquires a connection (creating lazily up to the bound), runs the transaction, releases. **A pool of 1 is byte-for-byte today's mutex-plus-connection semantics** — launch behavior unchanged. Record an **acquire-wait metric** (count + total wait µs, queryable) so raising a tenant's bound is data-driven; log a process-wide warning when aggregate created connections cross a configurable soft ceiling (guardrail for the ~80–100 `max_connections` instance limit).
- [x] `pooled_transaction_provider_test.cpp` (data-layer tests, real test DB): pool of 1 serializes exactly like `ProductionTransactionProvider` (two threads, second waits); pool of 2 lets two transactions overlap (barrier inside the first transaction); lazy creation (0 connections until first use, never exceeds bound); acquire-wait metric records contention; release-on-exception returns the connection.

### 2.2 Tenant resource registry + pluggable isolation factory (the ONLY place Model C vs B differs) **[hw]**
- [x] Add `components/platform/business_logic/tenancy/tenant_resources.{h,cpp}` — `Tenancy::TenantResources` owns the per-tenant `{TenantContext, DatabaseHelper, TransactionProviderPtr}` (the Phase 4 helpers — secrets/mail/site config — are added to this struct later, lazily built). Plus `TenantResourceRegistry` with `GetOrCreate(const TenantContext&) → std::shared_ptr<TenantResources>` (map keyed by `tenantId`, mutex-guarded, lazy) constructed with an **app-supplied factory** (`std::function<std::shared_ptr<TenantResources>(const TenantContext&)>`, defaulting to the framework type) so the app returns a derived type carrying app services — knottyyoga's `KnottyTenantResources` adds the per-tenant `SquareClient` in Phase 4.2 (§1.8.3 #1); the framework never references Square.
- [x] The isolation mechanism, confined to one factory: `MakeTenantTransactionProvider(const TenantContext&)` in `tenant_resources.cpp` —
  - **Model C (decided, Q1):** `PooledTransactionProvider(MakeProductionDatabaseHelper(ctx.databaseName), /*bound=*/ctx.maxConnections)` — a distinct pool per tenant database.
  - **Model B (documented fallback, not built):** a decorator issuing `SET LOCAL search_path` on a shared provider — kept as a header comment describing the swap, so switching models stays a one-file change.
- [x] `tenant_resources_test.cpp`: `GetOrCreate` builds once + caches (same pointer twice); distinct tenants → distinct providers; custom factory → derived type surfaces; the factory yields a provider bound to the context's database name + bound (assert via the provider's introspection / the test seam — no real second DB needed here; physical isolation is Phase 3.4's suite).

### 2.3 Wire the registry into `WebApp` **[hw + app]**
- [x] **[hw]** `WebApp` (`components/platform/endpoints/web_app.h` + `web_app_framework.cpp`) gains optional `TenantResolver` + `TenantResourceRegistry` members with setters/getters — **in addition to** the existing global provider for now (don't rip out the single-tenant path yet; the legacy global provider remains the fallback used only by tenant-agnostic endpoints like health, and by not-yet-migrated call sites during Phase 3.4).
- [x] **[app]** `src/main.cpp` composition root: build the resolver per the mode switch (Phase 3.1) + the registry (with the app factory — identity/default until Phase 4.2) and hand both to `WebApp`. In control mode, log the active-tenant count at startup. *(Transitional: for now installs a single `FixedTenantResolver` (tenantId=1, siteKey/databaseName=`App::kDatabaseName`) + a default `TenantResourceRegistry`; the `HONUWARE_TENANT_MODE` switch + active-tenant log land in Phase 3.1.)*
- [x] Tests: `web_app`-level construction coverage rides the existing endpoint-helper tests once Phase 3.4 teaches `EndpointTestHelper` to install a default fixed resolver + registry; add a focused test that `WebApp` returns the installed resolver/registry.

**⇑ bump point** — control plane + registry exist, nothing routed yet; both suites green.

> **✅ Phase 2 complete — 2026-07-22.** Both Linux docker clients green with the co-dev local override: **honuware full 1346** (+12), **app co-dev full 4493** (+12). 12 new tests: `PooledTransactionProviderTest` (5), `TenantResourcesTest` (6), `WebAppTenancyTest` (1).
> **Two findings worth carrying forward:**
> 1. **`TenantResources::databaseHelper` is `std::optional<DatabaseHelper>`, not a bare `DatabaseHelper`.** `DatabaseHelper` has no default constructor, so `make_shared<TenantResources>()` (which the registry's custom-factory tests do) wouldn't compile with a bare member; a custom factory also may legitimately leave it unbuilt. Phase 4 (secrets/mail) must `->` through the optional.
> 2. **`MakeTenantTransactionProvider` / the pool take the tenant's database *name* (a `std::string`), not a prebuilt `DatabaseHelper`** — the pool calls `MakeProductionDatabaseHelper(name)` internally to mint each connection (so pool-of-1 == today's single-connection shape exactly). Consequence — **`HONUWARE_DB_NAME` footgun:** `MakeProductionDatabaseHelper` honors that override, so a control-mode process MUST NOT set `HONUWARE_DB_NAME` or every tenant misroutes to one DB. Documented as a header comment on the factory; a name-forcing connection variant is deferred to Phase 5.1.
> **⇑ bump point still pending [Mason]:** push honuware Phase 2 → CI green → [Claude] re-pin app `GIT_TAG` + verify pinned SHA on a fresh volume → [Mason] commit app.

## Phase 3 — Request-edge tenant resolution (endpoints / auth layer)

Goal: every request resolves its tenant before touching the DB, and endpoints transparently use the tenant's provider. This is the layer where the multiplexing actually happens. **Lands in:** `honuware_platform` (`components/platform/endpoints/`) + `honuware_testing` (`components/testing/endpoints/endpoint_test_helper.*`), with the mode selection in the app composition root.

### 3.1 The site header + the mode switch **[hw + app]**
- [x] **[hw]** Header name (decided, Q2): **`X-Honuware-Site`** — one documented constant (`Tenancy::kSiteHeaderName`, in `tenant_header.h` under the tenancy dir). Read via `req.get_header_value(...)` — same mechanism as `cloudfront_origin_guard.cpp`.
- [x] **Mode selection (composition root, satisfies §1.10 #1 — zero ceremony single-tenant):** default mode is **fixed** — the app constructs `FixedTenantResolver` with a `TenantContext` from its own config (site key from `HONUWARE_FIXED_SITE_KEY` if set, else the app default `knotty`; database name = the app's existing configured name). **Control mode is opt-in** via `HONUWARE_TENANT_MODE=control` (reads the control DB per Phase 1.3). So: local dev + single-tenant consumers (CommunityFinder) run with **zero new env vars**; the knottyyoga prod deployment sets `HONUWARE_TENANT_MODE=control`. **[app]** implement the switch in `src/main.cpp`; **[hw]** document both modes in the honuware README's new-consumer checklist.
- [x] Fixed-mode contradiction rule: a request carrying a site header that isn't the fixed tenant's key is rejected (400), never silently served (the `FixedTenantResolver::Resolve` semantics from Phase 1.4 make this natural).

### 3.2 Resolve tenant in `EndpointAuthHelper` before session init **[hw]**
- [x] In `EndpointAuthHelper::Initialize()` (`components/platform/endpoints/endpoint_auth_helper.cpp`): (1) read `X-Honuware-Site`; (2) `TenantResolver::Resolve` (from `WebApp`); (3) on miss/suspended → record + surface the 3.3 error; (4) `TenantResourceRegistry::GetOrCreate`; (5) store the resolved `TenantContext` + `shared_ptr<TenantResources>` on the helper; (6) initialize the session against the **tenant** provider. *(As-built — see completion note: (2)/(3) moved to a `TenantResolutionGuard` middleware; Initialize re-resolves for success only, decoupling failure surfacing from routing.)*
- [x] Accessor work on the framework façade — **all the accessors already exist** (`GetTransactionProvider`/`GetSecretsHelper`/`GetMailHelper`/`GetDatabaseHelper`/`GetCookieManager`, today delegating to `WebApp`'s globals); the change is their *internals*: after `Initialize()` resolves a tenant, `GetTransactionProvider()`/`GetDatabaseHelper()` return the **tenant's**; add new `GetTenantContext()` + `GetTenantResources()`. The secrets/mail accessors re-point to tenant resources **in Phase 4** (until then they keep delegating to the globals so every intermediate build is green). `GetSquareClient()` stays on the app-derived `AppEndpointAuthHelper` (`src/endpoints/app_endpoint_auth_helper.*`) and re-points to `KnottyTenantResources` in Phase 4.2 — matching the componentization Phase 1.6 façade split.

### 3.3 Tenant-resolution failure handling **[hw]**
- [x] Behavior for missing/unknown/suspended tenant on a normal `/api/*` request in control mode: `421 Misdirected Request` (or `400`) with a JSON error (`{"error":"unknown_site"}` / `"site_suspended"`), rate-limited logging like the origin guard's. **Never fall through to a default tenant in control mode** (that would cross-serve data). *(As-built: control mode → 421 with `missing_site_header` / `unknown_site` / `site_suspended`; fixed-mode contradiction → 400 `site_mismatch`.)*
- [x] `/api/health` (and any other allow-listed, tenant-agnostic routes) keep working with **no** tenant — they must not touch the tenant accessors. Audit the origin-guard allow-list and the health handler. *(Verified: `health.cpp` reads `webApp->GetTransactionProvider()` directly — never constructs `EndpointAuthHelper` — and the guard allow-lists the `/api/health` prefix in every mode.)*

### 3.4 Cut endpoints over to the tenant provider + test harness **[hw + app]**
- [x] **The cutover is the accessor re-point, not a migration** (§1.8.0 verified: 180/182 app endpoints already call `endpointAuthHelper.GetTransactionProvider()`) — flipping the helper's internals converts the endpoint surface at once. The remaining work is an **audit**: (a) the ~3 direct `webApp->GetTransactionProvider()` sites (`health.cpp` stays global by design — tenant-agnostic; classify the others); (b) any business-logic helpers constructed with a provider captured at startup rather than per-request (grep construction sites); (c) the startup-time uses in `main.cpp`/`ServerConfig::Initialize` (legitimately the deployment's primary DB — leave). Keep the Phase 2.3 legacy global as the documented fallback for exactly the tenant-agnostic set. *(Confirmed by 4506 app tests passing with the re-point in place — every endpoint that uses `GetTransactionProvider()`/`GetDatabaseHelper()` now routes to the resolved tenant with zero endpoint edits.)*
- [x] **[hw testing]** `components/testing/endpoints/endpoint_test_helper.{h,cpp}`: install a **default fixed tenant** over the existing test database under a known test site key, so **existing endpoint tests pass with zero per-test edits**; add a helper to install a control-mode resolver + two control rows for routing tests. Add the focused test proving two site keys route to two distinct providers (two control rows over the same physical test DB — Q5's cheap default). *(`kDefaultTestSiteKey = "test-site"`; the test registry factory returns resources backed by the **test transaction provider** so resolved tenants stay inside the test's aborted transaction. `InstallControlModeTenants` + `EndpointAuthHelperTenancyTest.ControlModeRoutesDistinctTenants`.)*
- [x] Physical-isolation smoke suite (Q5): use the existing `GlobalDatabaseTestSupport::EnsureNamedDatabase("test_honuware_tenant_b", <the composed schema>)` (built for exactly this in componentization 2.6) — a handful of tests proving what two-rows-same-DB structurally can't: a row written via tenant A's provider is invisible via tenant B's **through the generic CRUD path**; PK sequences advance independently (§1.4's answer, now pinned by a test); per-DB `config_secrets` values are independent. *(`tenant_physical_isolation_test.cpp`: `current_database()` differs; `people` + `config_secrets` rows on A invisible on B; B's BIGSERIAL is independent. All transactions abort — no cross-run pollution. Note: because `DatabaseHelper::RunInTransaction` aborts, the cross-DB check uses a nested live A-txn + separate B-txn rather than a committed row, plus the `current_database()` assertion for unambiguous physical separation.)*
- [x] Edge tests **[hw]**: missing header in control mode → error; unknown site → error; suspended → error; health works headerless in both modes; fixed mode rejects a contradicting header; fixed mode serves headerless requests. *(`tenant_resolution_guard_test.cpp`, 10 tests.)*

**⇑ bump point** — the server is now multiplexing-capable end to end (with global-service Phase 4 helpers still shared); both suites green. **This is the riskiest phase — gate it carefully.**

> **✅ Phase 3 complete — 2026-07-22.** Both Linux docker clients green with the co-dev local override: **honuware full 1359** (+13), **app co-dev full 4506** (+13). All existing endpoint tests passed with the default fixed tenant installed — the risky cutover did not require a single per-endpoint edit. New tests: `TenantResolutionGuardTest` (10), `EndpointAuthHelperTenancyTest` (3), `TenantPhysicalIsolationTest` (2, filtered-gated separately).
> **Key as-built decision — resolution moved to a middleware.** The plan put resolution + failure-surfacing in `EndpointAuthHelper::Initialize()`, but endpoints unconditionally overwrite `resp.code` after `Initialize()`, so an error set there can't survive. Instead a new Crow middleware **`Endpoints::TenantResolutionGuard`** (added to `WebApp::AppType`, right after `CloudFrontOriginGuard`) does the request-edge resolution and short-circuits bad requests uniformly (zero endpoint edits — same mechanism as the origin guard). `EndpointAuthHelper::Initialize()` then **re-resolves the same site key for the success path only** (the resolver caches, so it's a cache hit) and re-points its provider/database + rebinds the session. This decouples "reject bad requests" (middleware) from "route good requests" (auth helper) and keeps `WebApp::SetTenantResolver(resolver, mode)` as the single install point (it feeds both the stored resolver and the guard).
> **Other findings:** (1) `EndpointAuthHelper::session_` became `std::optional<Auth::Session>` so it can be rebound to the tenant database in `Initialize()` (Session is non-movable + has no default ctor). (2) The `tenants` table's `UNIQUE(database_name)` forbids two control rows sharing one physical DB name, so the cheap same-DB routing harness gives each seeded tenant a distinct placeholder `database_name` (the test factory ignores it and routes both to the test DB anyway). (3) `main.cpp` control mode calls `Tenancy::EnsureControlDatabase()` at startup so the active-tenant-count log doesn't crash on a fresh control deploy (idempotent; real provisioning is Phase 5).
> **⇑ bump point pending [Mason]:** push honuware Phase 3 → CI green → [Claude] re-pin app `GIT_TAG` 2d94967→new + verify pinned SHA on the fresh volume → [Mason] commit app (`CMakeLists` re-pin + `src/main.cpp`).

## Phase 4 — Per-tenant services (secrets, payments, mail, site config)

Goal: the helpers that currently bake a single tenant's identity become per-tenant, sourced from the tenant DB and cached in `TenantResources`. Each subsection is lower-layer (util) before higher-layer (business-logic email bodies). **Lands in:** `honuware_services`/`honuware_platform`, except Square wiring (**[app]**, 4.2) and the app-side mail builders (**[app]**, 4.5). The heavy lifting here shrank dramatically (§1.8.0 #1): `TenantBranding` + `::Mail::LoadSenderAddress`/`LoadTenantBranding` exist, the framework mail builders are already parameterized, and all ~37 From-address sites already read through tenant-ready lookups — what's left is *which* secrets helper those lookups run against.

### 4.1 Per-tenant `SecretsHelper` **[hw]**
- [ ] `TenantResources` lazily builds the tenant's `SecretsHelper` from its own `DatabaseHelper` (`components/platform/business_logic/tenancy/tenant_resources.cpp`); `EndpointAuthHelper::GetSecretsHelper()` re-points from the global to the tenant resources. Keep the **global** at-rest master key `HONUWARE_SECRET_KEY` (it decrypts every tenant's `config_secrets.value` — §1.6 "stays global"). `SecretsHelper` needs no interface change — only a different `DatabaseHelper` (verified by construction in componentization 1.3/2.2).
- [ ] Tests: a `SecretsHelper` built against tenant-A resources reads tenant-A values; the cross-tenant-bleed case runs on the Phase 3.4 physical-isolation pair (two helpers → two databases → distinct `config_secrets` values).

### 4.2 Per-tenant `SquareClient` **[app]**
- [ ] New `src/business_logic/app_tenant_resources.{h,cpp}` — `KnottyTenantResources : Tenancy::TenantResources` adding a lazy `GetSquareClient()`: reads `Secrets::kSquareAccessToken` + `kSquareEnvironment` (`business_logic/app_secret_keys.h`) from the **tenant** secrets (4.1) and builds via `honuware_square`'s `MakeSquareClient(httpClient, token, isSandbox)` — sandbox/prod **per tenant**. Wire the registry factory in `src/main.cpp` to return it (the Phase 2.2 extension point).
- [ ] `AppEndpointAuthHelper::GetSquareClient()` (`src/endpoints/app_endpoint_auth_helper.cpp`) re-points from the process-wide `WebApp` service registry to the request's `KnottyTenantResources` (downcast of `GetTenantResources()`); the 21 Square-using endpoints keep their `helper.GetSquareClient()` call sites unchanged. Retire the startup `webApp.SetService<Square::SquareClient>(...)` registration once all call sites go through the tenant path.
- [ ] Tests: tenant on sandbox vs tenant on prod yield correctly-configured clients (per-tenant secrets on the isolation pair); `EndpointTestHelper` registers its `MakeTestSquareClient` double into the default fixed tenant's resources so the ~15 existing Square endpoint tests pass unchanged; `app_tenant_resources_test.cpp` for the factory/laziness.

### 4.3 Per-tenant `MailHelper` **[hw]**
- [ ] `TenantResources` lazily builds `MailHelper` per tenant (`MakeMailHelper(Transaction&, tenantSecrets)` — the services-layer constructor already reads SMTP config through the given secrets helper); the framework mail accessor re-points to it. Sender identity flows automatically: every From-address site already calls `::Mail::LoadSenderAddress(secrets, txn)`, so handing it the **tenant** secrets helper completes per-tenant identity with zero call-site edits.
- [ ] Tests: SMTP config + sender identity reflect the tenant (isolation pair, distinct `mail_sender_*` values); existing mail tests keep passing via the test double registered in the default fixed tenant.

### 4.4 `ServerConfig` global vs per-tenant **[hw]**
- [ ] Smaller than originally planned (verified 7/21/2026): the `Auth::ServerConfig` singleton (`components/platform/business_logic/auth/server_config.{h,cpp}`) **already stores only the global bits** (`prodMode_`/`testMode_`) — website address and CORS are read from secrets at `Initialize` and applied to the middleware, not stored. So: keep the singleton as the deployment-global config (its startup read runs against the deployment's **primary** DB — knottyyoga's own, tenant #1 — unchanged sequence in `main.cpp`); no rename needed unless clarity wants it. The real work is the two per-tenant behaviors below.
- [ ] Rework CORS: each tenant is same-origin behind its own CloudFront, so prod CORS is effectively a no-op; replace the single pinned-origin registration with a dynamic check validating the request `Origin` against the **resolved tenant's** origin, keeping the dev `HONUWARE_DEV_CORS_ORIGIN` path. Remove the `ServerConfig` global-singleton assumption.
- [ ] Tests: cookie `Domain`/`Secure` derive from the tenant's website address (extend the existing `SessionTest.InitializeFromLoginProdMode…` coverage to assert per-tenant domain on the isolation pair); CORS accepts the tenant origin and rejects others.

### 4.5 App-side email templates onto `TenantBranding` **[app]**
- [ ] The framework builders are done (§1.8.0 #1). This phase covers the **app-side** builders — **30 `*_mail.cpp` files** under `src/business_logic/{payment,scheduling}/` still hardcode the brand (verified count, 7/21/2026; `tenant_branding.h`'s own comment marks them "parameterized in the tenancy project's Phase 4.5"): replace `"Knotty Yoga"` / `"Knotty Yoga and Spa"` literals with `FormatString` `{studio_name}`-style placeholders (keep `NormalizeCrLf`), each builder taking `const Mail::TenantBranding&` filled via `::Mail::LoadTenantBranding(tenantSecrets, txn)`. **Enumerate all 30 as a checklist when executing** so none is missed; watch the two known traps (memory: qualify `::Mail::`; no `= nullptr` secrets default params that silently drop mail).
- [ ] Tests: each builder's `*_test.cpp` gains a substitution case with a non-brand studio name + `Not(HasSubstr("Knotty Yoga"))` (the framework builders' pattern).

**⇑ bump point** — every per-request service is tenant-scoped; both suites green.

## Phase 5 — Provisioning & migrations (database_helper / ops)

Goal: stand up and evolve tenant databases repeatably. **Lands in:** framework provisioning/migration functions in `honuware_platform`; the CLI + composed framework-and-app `DatabaseInfo`/migration lists stay in the app's `src/database_helper/` (§1.8.3 #3).

### 5.1 Split `create_database.cpp` into framework/app halves **[hw + app]** *(pulled in per §1.10 #3 — serves tenant provisioning AND CommunityFinder hand-off item 2)*
- [ ] **[hw]** Platform gains `CreateFrameworkTables(...)` (framework DDL + indexes in FK order) and `PopulateFrameworkTables(...)` (the framework tables' admin metadata, base roles/permissions, `allowed_tables` entries for framework tables, framework + registered-app secret defaults, scheduler service account via the parameterized `EnsureSchedulerServiceAccount`) — extracted from the app's `src/database_helper/create_database.cpp`, mirroring the `MakeFrameworkTables`/`MakeAppTables` split. Home: `components/platform/business_logic/` (next to `migration/`).
- [ ] **[app]** `create_database.cpp`'s `CreateAndPopulateDatabases()` becomes composition: framework halves + its app half (app DDL, app admin metadata `PopulateAdmin*`, app seeds). **Gate: `--recreate_database` produces an identical database before/after the refactor** (the existing full suite against a recreated DB is the check).
- [ ] Tests **[hw]**: the framework halves stand up a framework-only database (this is exactly what honuware's own test main + CI exercise); composition test that framework+app yields today's full table set.

### 5.2 `--create-tenant` command **[hw + app]**
- [ ] **[hw]** Framework function `Tenancy::ProvisionTenant(controlProvider, const TenantSpec&, const DbSchema::DatabaseInfo&, <create+populate callable>)` in `components/platform/business_logic/tenancy/provisioning.{h,cpp}`: `EnsureControlDatabase()` → create the tenant database → run the composed create/populate path against it → insert the `tenants` control row → `Invalidate(siteKey)` on the resolver. Idempotency: an existing `site_key` is an explicit error (not a duplicate row); a `--force` recreate path guards behind the existing destructive gate (`HONUWARE_ALLOW_DESTRUCTIVE`).
- [ ] **[app]** `knottyyoga_database_helper --create-tenant --site-key=<k> --db-name=<n> --display-name=<…>` mode (alongside the existing `--recreate_database`/`--migrate` modes in `src/database_helper/main.cpp`), passing the composed `MakeDatabaseInfo(dbName)` + the app's composed create/populate callable down.
- [ ] **[app]** A companion `--seed-tenant-secrets --site-key=<k> --file=<secrets.json>` step (or reuse the existing secret-seeding workflow) to set that tenant's Square/mail/website secrets. Documented order: create-tenant → seed-secrets → tenant live.
- [ ] Tests **[hw]**: control-row insert + duplicate-site-key error + resolver invalidation (control rows over the test DB); the physical create path is covered by the `EnsureNamedDatabase`-based tests + a live `--create-tenant` run of `knottyyoga_database_helper` (Claude can run this inside the app docker container against the dev Postgres — it creates a *new* scratch tenant DB, nothing destructive).

### 5.3 Per-tenant migration loop **[hw + app]**
- [ ] **[hw]** `Tenancy::MigrateAllTenants(controlProvider, const std::vector<Migration>& composed)` — iterate `Tenants::ListActive`, run `MigrationRunner::ApplyPending(...)` against **each** tenant database, plus the control DB's own (framework-namespace) migration list. The composed list is the existing namespaced framework++app stream (§1.8.0 #3 — already built and tested); per-migration transaction semantics already hold. Log per-tenant progress; **abort on first failure** (decided, Q6) — re-runs skip already-migrated tenants via `schema_migrations`.
- [ ] **[app]** Extend `--migrate` in `src/database_helper/main.cpp`: fixed mode → migrate the one configured DB (today's behavior); control mode present → the tenant loop.
- [ ] Tests **[hw]**: a fixture migration list applies to two tenant DBs (isolation pair) and is idempotent on re-run; a failure in tenant A aborts before tenant B with a clear report; control-DB migrations tracked under the framework namespace.

### 5.4 Onboarding runbook **[docs]**
- [ ] Document the end-to-end onboard: `--create-tenant` → `--seed-tenant-secrets` → create CloudFront distribution + S3 bundle + ACM cert + DNS → set the distribution's `X-Honuware-Site` (+ shared `X-Origin-Secret`) origin headers → smoke-test `/api/health` then a tenant request. (Cross-references Phase 8 and `Deploying to AWS.md`.) Add the single-tenant consumer note (no control DB — this runbook is knottyyoga-deployment-only).

**⇑ bump point** — tenants can be stood up + evolved repeatably; both suites green.

## Phase 6 — Scheduler multi-tenant

Goal: background jobs run for every tenant, not just one. **The engine is already tenancy-agnostic and self-describing** (`knotty_yoga_scheduler`, foundation-only; `ScheduledJob` carries `serverUrl`/credentials/`headers`, one authenticated `ApiClient` per distinct target — §1.8.0 #2). What remains is app-side catalog × tenants.

### 6.1 Tenant-aware job list (app-side) **[app]**
- [ ] `src/scheduler/main.cpp` gains the mode switch: fixed mode (default) → today's single `JobTarget` (unchanged behavior); control mode → read `Tenants::ListActive` via `MakeControlDatabaseHelper()` at startup **and on a refresh interval**, and build `BuildKnottyYogaJobs(targetForTenant, intervals)` per active tenant, concatenated. Each tenant's `JobTarget.headers` carries `X-Honuware-Site: <site_key>` (+ the `X-Origin-Secret` header when `HONUWARE_ORIGIN_SECRET` is set on the deployment — the scheduler talks to the origin directly, bypassing CloudFront, so it must supply both itself).
- [ ] Refresh semantics: a newly-provisioned tenant is picked up on the next refresh without a restart; a suspended tenant's jobs stop. (The engine consumes a job list — simplest correct implementation: rebuild the scheduler's job set on refresh; document the chosen mechanism.)

### 6.2 Per-tenant authenticated calls **[app]**
- [ ] Login per tenant as that tenant's scheduler service account (each tenant DB has its own service-account row from Phase 5's provisioning; `serviceAccountEmail` from `App::kSchedulerServiceAccountEmail`). Password scope: **shared** `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` across tenants (decided, Q7). The engine already logs in once per distinct `(serverUrl,email,password)` target — with per-tenant site headers on the jobs, per-tenant sessions fall out of the existing `AuthenticateTargets` machinery; verify the 401-retry keeps the site header (it does — extra headers survive the retry, per the engine's tests).
- [ ] Tests: catalog-×-tenants construction (extend `knottyyoga_job_catalog_test.cpp`'s existing two-tenant concatenation case to assert the site header per tenant); control-list refresh logic; engine header/target behavior is already covered in the honuware repo (`job_scheduler_test`/`api_client_test`).

**⇑ bump point** — background jobs are tenant-complete; both suites green.

## Phase 7 — Frontend per-tenant branding

Goal: one Angular bundle serves all tenants; branding arrives at boot from the API. (Full theming is Website Makeover Phase 5; this is the minimal hook.) Scope re-grounded per §1.9: the `@honuware/ui` library pages are already brand-free and published — **only the app shell de-brands here**; the library is untouched (no `@honuware/ui` release needed for this phase).

### 7.1 `/api/site_info` endpoint (backend first, per layering) **[hw]**
- [ ] Add `components/platform/endpoints/site_info.{h,cpp}` — `GET /api/site_info` returning the resolved tenant's public branding as JSON: `display_name` (tenants row / branding), `website_url`, `logo_url` (nullable for now), sourced via the request's `TenantContext` + tenant secrets (`::Mail::LoadTenantBranding` supplies studio name + website URL; extend with a `site_logo_url` secret key, default empty). Unauthenticated; no session init needed beyond tenant resolution; mark cacheable (`Cache-Control: public, max-age=300`). Anchor it in `register_framework_endpoints.cpp`.
- [ ] `components/platform/endpoints/site_info_test.cpp`: returns the resolved tenant's branding (drive two site keys → two brandings via per-DB secrets); unknown site → the Phase 3.3 error; works with no auth cookie; served headerless under the fixed resolver (single-tenant consumers get their static branding).

### 7.2 ServerAccess method **[ui]**
- [ ] Add `getSiteInfo(): Observable<SiteInfo>` to the app's `ServerAccess` interface (`shared/types/ServerAccess.ts` + a `SiteInfo` type), `ServerAccessNetwork` (`GET /api/site_info`), `ServerAccess.mock.ts` (returns the dev branding), and `ServerAccessProxy`; add `ServerAccess.mock.spec.ts` coverage (per the frontend testing rule). The `@honuware/ui` access seams are **not** extended (§1.9 #3).

### 7.3 Replace hardcoded branding **[ui]**
- [ ] `SiteConfigService` (`core/services/site-config.service.ts`): fetches `/api/site_info` via a `provideAppInitializer` before render, exposes `siteInfo` with the current hardcoded values as synchronous fallback (never block render on a failed fetch — fall back and log). Spec: initializer resolves, fallback path on error.
- [ ] Refactor the three §1.9-enumerated shells to read from `SiteConfigService`: `shared/components/header/header.component.ts` (logo URL + alt text — keep `assets/svg/KnottyYoga_logo_white.svg` as the fallback asset), `shared/components/footer/footer.component.html` (+`.ts`: address lines, contact email `info@knottyyoga.com`, tagline, `© Copyright KnottyYoga` name — social links can stay app-static for now), `shared/constants/about.ts` consumers. Component spec updates for each (per the "test every component change" rule).
- [ ] `HONUWARE_BRANDING` token: **only if** a `@honuware/ui` surface turns out to need branding during this phase does the token get created (in `@honuware/ui/foundation`, static default, `SiteConfigService` as the app-side provider — frontend Q12); otherwise explicitly defer. Today no library surface reads a brand (§1.9 #2).

### 7.4 Theming hand-off
- [ ] Note the dependency on Website Makeover Phase 5 (DB-driven theme tokens + `/api/site_theme`); align `site_info` so it can carry or coexist with theme tokens rather than duplicating them. Also note for CommunityFinder: it runs static branding and does not consume `/api/site_info` (§1.10).

## Phase 8 — Infrastructure (per-tenant CloudFront / DNS / headers)

Goal: the AWS wiring that makes one origin serve many branded sites. (**All [Mason] ops**; code is ready after Phases 1–7. Nothing here blocks Phases 0–7 — the whole plan is testable locally with the fixed resolver + two control rows.)

### 8.1 Per-tenant distribution
- [ ] Per tenant: S3 bucket with the **same** built `ui` artifact (Q10: one shared bundle — branding arrives at runtime from `/api/site_info`), CloudFront distribution, ACM cert (us-east-1), Route 53 records — following `Deploying to AWS.md` Phase 4/5 conventions.

### 8.2 Header injection
- [ ] On each distribution's `/api/*` behavior, set origin custom headers: the shared `X-Origin-Secret` (existing) **and** that tenant's `X-Honuware-Site=<site_key>` (must match the tenant's `tenants.site_key` control row from Phase 5.1's `--create-tenant`).

### 8.3 Origin-secret strategy
- [ ] Keep `X-Origin-Secret` deployment-wide for now (origin protection, not identity — decided, Q8). Record the option to make it per-tenant later for defense-in-depth.

### 8.4 Server environment
- [ ] Extend `/etc/knottyyoga/server.env` (+ any ECS task definitions per `Deploying to AWS.md`): the prod server runs the **control-DB resolver** (no `HONUWARE_FIXED_SITE_KEY`), with `HONUWARE_CONTROL_DB_NAME` optional (default `honuware_control`); the new `HONUWARE_DB_*` names from Phase 0 replace `KNOTTYYOGA_DB_*` in deploy configs (legacy fallback keeps old configs working during the transition).

---

# Part 3 — Open Questions

**Status (7/21/2026): ALL 10 questions decided** and folded into the plan above (Q4 stands adopted-by-default with no objection raised; Q5 confirmed). Summary: Q1 = Model C; Q2 = `X-Honuware-Site`; Q3 = `honuware_control` control DB; Q4 = bounded pool defaulting to 1 connection/tenant; Q5 = two-control-rows default + one real second test DB for isolation; Q6 = abort on first migration failure; Q7 = shared scheduler password; Q8 = deployment-wide origin secret; Q9 = identical catalog baseline; Q10 = shared frontend bundle. **The plan below was re-grounded 7/21/2026 against the completed componentization efforts (server §1.8.0, frontend §1.9, CommunityFinder §1.10) — Part 2 is now the concrete, execution-ready version.**

Mason- Please look at [[Splitting the server up into components]] and update this document based on the componentization effort.
	- Claude- Done — the componentization impact is analyzed in the new **§1.8** and threaded through every affected phase in Part 2. The headline changes: the tenancy machinery becomes honuware **framework** code with a per-phase repo map (§1.8.1–1.8.2); `TenantResolver` gains a `FixedTenantResolver` so single-tenant consumers (the friends' sites, local dev) need no control DB or CloudFront header; `TenantResources` gains an app-factory extension point so Square stays app-side, matching the component plan's `EndpointAuthHelper` façade split; the scheduler engine stays foundation-only with tenancy expressed in the app-built job list; provisioning/migrations take the composed framework+app `DatabaseInfo`/migration lists (framework migrations in their own id namespace); framework mail-builder branding is **pulled forward into the componentization work** as an extraction prerequisite — that one item should be fed back into the component plan's Phase 1 when executing it; and all new names are honuware-branded per Q2/Q3. Sequencing is now: componentize → extract → **this plan** → first deploy, mostly as cross-repo work via `FETCHCONTENT_SOURCE_DIR_HONUWARE`.

1. **Isolation model — confirm Model C (database-per-tenant).** Part 1 recommends C with B as a swappable fallback and rejects A. *Default: proceed with C.* Confirm, or pick B/A.
	- Mason- I'll go with your recommendation.
2. **Header name & semantics.** I propose `X-Knotty-Site: <site_key>` where `site_key` is a short stable slug (e.g. `knotty`, `acme`). *Default: `X-Knotty-Site`.* OK, or different name/value (e.g. send the database name directly, or a UUID)?
	- Mason- Per [[Splitting the server up into components]], let's go with X-Honuware-Site
	- Claude- Decided — renamed throughout (§1.7 diagram, Phases 3.1, 6.1, 8.2).
3. **Control-plane location.** A dedicated `knottyyoga_control` database on the same RDS instance holding `tenants`. *Default: dedicated control DB.* Alternative: a JSON/env tenant manifest for the very first soft-launch (simpler, not dynamic) — acceptable, or go straight to the control DB?
	- Mason- Let's go with honuware_control and do the control db.
	- Claude- Decided — `honuware_control` + the `HONUWARE_CONTROL_DB_NAME` override threaded through §1.5 and Phases 1.3/5. Two notes from the componentization pass: the control DB is **per deployment** (each honuware consumer runs its own — §1.8.1), and single-tenant consumers skip it entirely via `FixedTenantResolver`, so the control DB only exists where multi-tenancy is actually in play.
4. **Connection-count ceiling.** One persistent libpqxx connection per active tenant. Fine to dozens; beyond ~50 we'd add PgBouncer or a bounded/evicting pool. *Default: one-per-tenant now, document the PgBouncer trigger.* Any near-term tenant-count target that would change this?
	- Mason- Do you think one connection for tenant will be too limited? Would it be worth it to have a pool of connections per tenant?
	- Claude- One connection per tenant will **not** be a *throughput* problem at studio scale: transactions are milliseconds, so a single connection sustains on the order of a hundred-plus transactions/second per tenant — far above real load. The genuine risk is **head-of-line blocking**: a transaction that holds the tenant's connection across a slow external call (a Square request can take seconds) queues every other request for that tenant behind it. And the budget cuts against fat pools: pool size multiplies against tenant count (`N × P`) into the ~80–100 `max_connections` ceiling on a t3.micro — 20 tenants × pool-of-4 is already the wall. So my recommendation is the middle path: **build the provider as a bounded lazy pool whose size defaults to 1**. Today's design (one connection + one mutex) *is* a pool of 1, so this generalizes the existing shape rather than adding machinery — and "give this busy tenant 2–3 connections" becomes a per-tenant config knob (a `max_connections` column on the control `tenants` row) instead of a refactor. Pair it with an acquire-wait metric and an aggregate-connection warning so raising a limit is data-driven, keep Square/external HTTP out of the DB critical section where feasible (the cheaper fix for head-of-line blocking), and reach for PgBouncer only if aggregate demand ever approaches the instance ceiling. **Adopted as the default and folded into Phase 2.2** — flag if you'd rather ship the plain single-connection provider and defer any pool shape entirely.
5. **Test strategy for multi-tenant routing.** The suite uses one shared test DB (`test_knottyyoga`). To test routing without standing up many DBs, I'll register **two control rows pointing at the same physical test DB under two site keys** and assert routing/caching/independence. *Default: that approach.* Or do you want a second real test database provisioned in the harness?
	- Mason- What do you recommend?
	- Claude- Both, weighted heavily toward the cheap one. **(a) Default for the bulk of tests: two control rows → the same physical test DB.** This proves resolution, caching, invalidation, suspension, header edge cases, and that two site keys yield two distinct providers/resources — zero new infrastructure, fast, and these tests move to the honuware repo's CI after extraction. **(b) Plus a tiny physical-isolation suite on one real second database.** Two-rows-same-DB structurally *cannot* prove the thing Model C exists for — that tenant A's data is invisible to tenant B. Have `GlobalDatabaseTestSupport` create `test_honuware_tenant_b` once per run (the same create-once pattern the test-speed work established; one extra `CREATE DATABASE` per run is cheap) and keep the suite to a handful of tests: a row written via tenant A's provider is invisible via tenant B's through the generic CRUD path, PK sequences advance independently, per-DB `config_secrets` values are independent. **Adopted as the default and folded into Phases 3.4/4.1.**
	- Mason- I'll go with your recommendation.
6. **Migration failure policy across tenants.** When `--migrate` hits a failure on tenant K of N, do we **abort** (stop, surface, fix, re-run — re-runs skip already-migrated tenants) or **continue** and report all failures at the end? *Default: abort on first failure* (safest; matches the existing single-DB runner's stop-on-failure semantics).
	- Mason- Let's go with the safer option. Fail fast is generally good.
7. **Scheduler service-account password scope.** Shared `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` across all tenants, or per-tenant passwords? *Default: shared* (simplest; each tenant still has its own service-account row, just the same secret).
	- Mason- I'm fine with your recommendation.
8. **Origin-secret scope.** Keep one deployment-wide `X-Origin-Secret`, or make it per-tenant for defense-in-depth? *Default: deployment-wide now, per-tenant later.*
	- Mason- I'll go with your recommendation.
9. **Global-catalog drift.** Catalog tables (`admin_*`, `permissions`/`roles`) are re-seeded per tenant DB. If you ever want a tenant to customize, say, its admin column labels, Model C supports it for free (it's just their row), but the *baseline* is identical everywhere. *Default: identical baseline, per-tenant customization allowed but not built.* Confirm you don't need per-tenant permission catalogs on day one.
	- Mason- Confirmed. That sounds reasonable.
10. **Frontend strategy.** One shared bundle + runtime branding from `/api/site_info` (recommended; minimal), vs. separate per-client app bundles sharing components (max flexibility, more build/deploy machinery). *Default: shared bundle + runtime branding*, deferring full theming to Website Makeover Phase 5. Confirm.
	- Mason- Let's do shared bundle for now. Will probably componentize later but I want to get this done server side for now.
	- Claude- Decided — this also matches the component plan's Q7 (frontend sharing deferred to its own future job). Phase 7 stays the minimal `/api/site_info` hook: the endpoint is framework, the Angular consumption is app-side.

---

# Implementation Log

(Checkboxes above are unchecked — this plan is written but not yet implemented. As we implement, each item gets checked off per the "mark completed" workflow. Backend layers land before frontend within each phase, per the established convention.)

- **7/21/2026 — Plan re-grounded to as-built reality (v0.2).** All 10 open questions closed; added §1.8.0 (server componentization/extraction COMPLETE — the seams this plan plugs into now exist as real code, with paths), §1.9 (frontend: `@honuware/ui@0.1.1` published; Phase 7 scope shrinks to the app shell), §1.10 (CommunityFinder = separate single-tenant consumer; its constraints + two shared work items folded in). Part 2 rewritten execution-ready: new Phase 0 (co-dev loop + `HONUWARE_DB_*` rename), concrete `components/…` + `src/…` file paths and **[hw]/[app]/[ui]** tags on every item, the fixed-vs-control **mode switch** (`HONUWARE_TENANT_MODE`, default fixed = zero-ceremony single-tenant), the pooled provider as its own data-layer item, `create_database.cpp` framework/app split pulled into Phase 5.1, and per-phase **⇑ bump points** for the honuware push → re-pin → app-green cadence. Ready to start at Phase 0.