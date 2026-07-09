---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/9/2026
Version: 0.1
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

### 1.8.1 Tenancy becomes a honuware framework feature

The component architecture already reserves a slot for `business_logic/tenancy/` inside `honuware_platform`. Making that explicit:

- The `tenants` control schema, `TableHelpers::Tenants`, `TenantContext`, `TenantResolver`, `TenantResourceRegistry`, the edge resolution in `EndpointAuthHelper`, the provisioning/migration loops, and `/api/site_info` are all **framework code** — zero yoga specifics, and every honuware consumer (the friends' sites included) gets multi-site support for free.
- Their tests ship in the honuware repo and run under its GitHub Actions CI (Linux + Postgres service container).
- **Single-tenant consumers must pay ~zero ceremony.** Make `TenantResolver` an interface with two implementations: the control-DB resolver (multi-tenant deployments) and a `FixedTenantResolver` that serves one configured `TenantContext` with **no control DB and no CloudFront header at all**. The friends' sites and local dev both run the fixed resolver; the knottyyoga deployment runs the control-DB one. This subsumes (and upgrades) the dev escape hatch in Phase 3.1.
- The control DB is **per deployment** (one per RDS instance/consumer), not a shared global service — `honuware_control` is just its default name.

### 1.8.2 Repo map — where each phase's work lands

| Phase | Lands in |
|---|---|
| 1 — control plane (schema, helper, resolver) | honuware `platform` |
| 2 — per-tenant connections & registry | honuware `data` + `platform` |
| 3 — request-edge resolution | honuware `platform` (web core) + `testing` (EndpointTestHelper) |
| 4.1–4.4 — secrets, mail, config | honuware `services`/`platform`; **Square wiring → app** (§1.8.3 #1) |
| 4.5 — email branding | split: framework mail builders → honuware (pulled forward, §1.8.3 #4); payment/scheduling builders → app |
| 5 — provisioning & migrations | framework functions in honuware; `--create-tenant`/`--migrate` CLI + composed `DatabaseInfo` stay in the app's `database_helper` |
| 6 — scheduler | engine stays `honuware_scheduler` (foundation-only, tenancy-agnostic); tenant iteration + job catalog in app-side `scheduler/main.cpp` |
| 7 — frontend branding | app (frontend sharing deferred per the component plan's Q7); the `/api/site_info` endpoint itself is framework |
| 8 — infra | ops (unchanged) |

Practical consequence: most of this project is **cross-repo work** — honuware and knottyyoga edited together with `FETCHCONTENT_SOURCE_DIR_HONUWARE` pointed at the local checkout, and a honuware version bump when each slice stabilizes. The component plan's Q6 discussion priced this in ("a few weeks of editing both repos together"); the phases here are already sliced so each is a clean bump point.

### 1.8.3 Design changes forced (or gifted) by the component boundaries

1. **`TenantResources` gains an app extension point (Square).** Component Phase 1.6 removes `SquareClient` from the framework `EndpointAuthHelper` façade: the framework exposes session/cookies/db/secrets/mail, and the app supplies a derived accessor for Square. Tenancy follows the same shape — the framework `TenantResources` holds `{DatabaseHelper, TransactionProvider, SecretsHelper, MailHelper, site config/branding}`, and `TenantResourceRegistry` is constructed with an **app-supplied factory** (`std::function<std::shared_ptr<TenantResources>(const TenantContext&)>`, wired in `main.cpp` — the composition root the component plan's Phase 3.2 establishes). knottyyoga's factory returns a derived `KnottyTenantResources` adding the per-tenant `SquareClient` (built from tenant secrets using the `honuware_square` client component); the app-derived endpoint helper preserves the `helper.GetSquareClient()` call-site ergonomics. The framework never references Square. (Phases 2.1, 3.2, 4.2 updated.)
2. **The scheduler engine stays tenancy-agnostic.** After component Phase 1.7 the engine consumes a data-driven `std::vector<ScheduledJob>` and links foundation only — it cannot read the control DB, and shouldn't. Multi-tenancy is expressed in the *job list*: `ScheduledJob` grows per-job request headers and login credentials; app-side `scheduler/main.cpp` reads the active-tenant list from the control DB (the app-side main keeps app-layer deps anyway) and builds catalog × tenants, each job carrying `X-Honuware-Site: <site_key>`. (Phase 6 updated.)
3. **Provisioning and migrations take composed inputs.** Component Phase 2.3 splits `MakeDatabaseInfo()` into framework-table + app-table assembly composed by the app. So the framework provisioning function is shaped `ProvisionTenant(controlProvider, TenantSpec, const DatabaseInfo&)`, and the per-tenant migration loop takes a composed migration list — **framework migrations and app migrations are two streams, applied framework-first, tracked in `schema_migrations` under separate id namespaces** so honuware upgrades and app changes can't collide. The CLI surface stays in `knottyyoga_database_helper`. (Phases 5.1/5.2 updated.)
4. **Framework mail-builder branding moves earlier.** `person_verify_mail.cpp` (and any other framework mail body) hardcodes "Knotty Yoga" today but is slated to move into the *public* honuware repo — it can't ship with the brand baked in. The `TenantBranding` parameterization of the **framework** builders therefore becomes a prerequisite of the extraction and should be executed during the componentization work (its Phase 1.3 secrets-defaults registration is the natural mechanism; a single-tenant consumer registers one branding). Phase 4.5 here then covers only the app-side builders (payment/scheduling) plus sourcing the branding struct per-tenant. **Action: fold this item into the component plan's Phase 1 when executing it.**
5. **Naming.** Everything new in this plan is honuware-branded (Q2/Q3 below): header `X-Honuware-Site`, control DB `honuware_control`, env vars `HONUWARE_CONTROL_DB_NAME` and `HONUWARE_FIXED_SITE_KEY`. Existing env vars this plan references (`KNOTTYYOGA_ORIGIN_SECRET`, `KNOTTYYOGA_TRUST_PROXY`, `KNOTTYYOGA_SECRET_KEY`, `KNOTTYYOGA_DEV_CORS_ORIGIN`, `KNOTTYYOGA_ALLOW_DESTRUCTIVE`) belong to code that moves into honuware and will be renamed `HONUWARE_*` (new name read first, old name as fallback) per the component plan's convention — this plan uses the new names.
6. **What componentization hands tenancy for free.** Parameterized database name (component 1.2), secrets defaults via registration (1.3), the de-Square'd `EndpointAuthHelper` (1.6), the data-driven scheduler (1.7), the framework/app `DatabaseInfo` split (2.3), `GlobalDatabaseTestSupport` taking composed inputs (2.6), and the "everything app-specific enters at the composition root" audit (3.2). These were previously implicit prep inside this plan's phases; by the time this plan executes they already exist, which shrinks Phases 2–4 noticeably.

---

# Part 2 — Implementation Plan

Phases are ordered lowest-layer-first. Within each phase, subsections are numbered and also ordered lowest-layer-first. Every code change lists its test work as a checkbox (per the "always add tests" rule). The isolation mechanism (Model C dbname vs Model B search_path) is confined to Phase 2.2 so the rest of the plan is mechanism-agnostic. Each phase also notes **where the code lands** after the honuware extraction (repo map in §1.8.2); during this project the two repos are edited together via the `FETCHCONTENT_SOURCE_DIR_HONUWARE` override.

> **Build/test/git:** I will not build the C++ server, run the suite, or run git — you do those. Each phase is sized so you can build + run tests between phases.

## Phase 1 — Control plane & tenant model (lowest layer: new control DB + registry)

Goal: a queryable source of truth that maps a CloudFront site key to a tenant + its database name, plus an in-memory resolver. No request path touches this yet. **Lands in:** honuware `platform` (framework db_schema / table_helpers / business_logic).

### 1.1 Control-database schema (`db_schema` layer)
- [ ] Add `db_schema/tenants.{h,cpp}` defining the `tenants` table with `BIGSERIAL id` PK and columns: `site_key` (UNIQUE, the value CloudFront sends in `X-Knotty-Site`), `database_name` (UNIQUE), `display_name`, `status` (e.g. `active`/`suspended`, default `active`), `created_us`, `updated_us`. Follow the column-constant + `MakeTenantsTable(DatabaseInfo)` pattern used by `classes.cpp`.
- [ ] Decide the control DB's schema assembly: a dedicated, minimal `MakeControlDatabaseInfo()` (it needs only `tenants`, and optionally `schema_migrations` for its own evolution) rather than reusing the full `MakeDatabaseInfo()`.
- [ ] Add both files to `db_schema/CMakeLists.txt`.

### 1.2 Tenants table helper (`table_helpers` layer)
- [ ] Add `sql_util/table_helpers/tenants.{h,cpp}` — `TableHelpers::Tenants` with `LookupBySiteKey(transaction, siteKey) → optional<TenantRow>`, `ListActive(transaction) → vector<TenantRow>`, `Insert(transaction, …)`, `SetStatus(transaction, siteKey, status)`. Prefer `DbCrud` helpers; custom SQL only where `DbCrud` can't express it.
- [ ] Add `tenants_test.cpp`: insert/lookup, unknown site key returns empty, duplicate site_key/database_name throws, `ListActive` filters by status. Add helper + test to `table_helpers/CMakeLists.txt`.

### 1.3 Control-database bootstrap (`database_helper` layer)
- [ ] Add a code path to create + populate the control DB (create database `honuware_control`, create the `tenants` table). Model it on the existing `CreateAndPopulateDatabases()` structure but minimal. The function itself is framework (honuware); the app's `database_helper` CLI invokes it.
- [ ] Add a `MakeControlDatabaseHelper()` convenience (wraps `MakeProductionDatabaseHelper("honuware_control")`), reading an optional `HONUWARE_CONTROL_DB_NAME` env override (fallback `honuware_control`).
- [ ] Tests for the control-DB info assembly (table present, columns/constraints as expected) mirroring existing `make_database_info`-style tests.

### 1.4 Tenant resolver (business-logic layer, no request coupling yet)
- [ ] Add `business_logic/tenancy/tenant_context.h` — a small immutable `TenantContext { int64_t tenantId; std::string siteKey; std::string databaseName; std::string displayName; }`.
- [ ] Add `business_logic/tenancy/tenant_resolver.{h,cpp}` — `TenantResolver` holding the control `TransactionProvider`; `Resolve(siteKey) → optional<TenantContext>` backed by an in-memory cache (map + mutex) with explicit `Invalidate(siteKey)` / `InvalidateAll()` for onboarding/suspension. Start with cache-on-read; no TTL needed if invalidation is wired into provisioning (Phase 5).
- [ ] `tenant_resolver_test.cpp`: resolves a seeded tenant, caches (second lookup doesn't re-query — inject a counting provider), unknown key returns empty, `Invalidate` forces a re-query, suspended tenant is not resolved (or resolved with status so the edge can 403). Wire into a new `business_logic/tenancy/CMakeLists.txt`.

## Phase 2 — Per-tenant connection & transaction routing (database-access layer)

Goal: turn "one connection baked at startup" into "a lazily-built, cached per-tenant `TransactionProvider`," behind an interface that hides Model C vs B.

### 2.1 Tenant resource registry (database-access + DI layer)
- [ ] Add `business_logic/tenancy/tenant_resources.{h,cpp}` — `TenantResources` owns the per-tenant `{DatabaseHelper, TransactionProviderPtr}` (helpers from Phase 4 are added to this struct later). Add `TenantResourceRegistry` with `GetOrCreate(const TenantContext&) → shared_ptr<TenantResources>` (map keyed by tenantId, mutex-guarded, lazy). Document the connection-count guardrail (one connection per active tenant; revisit with PgBouncer beyond ~50 tenants — Open Question 4).
- [ ] `tenant_resources_test.cpp`: `GetOrCreate` builds once and caches (same pointer on second call), distinct tenants get distinct providers. Use the test transaction-provider seam so no real second DB is needed.

### 2.2 Pluggable isolation mechanism (the ONLY place Model C vs B differs)
- [ ] Define how a tenant's `TransactionProvider` acquires its connection:
  - **Model C (recommended):** `MakeProductionTransactionProvider(MakeProductionDatabaseHelper(tenant.databaseName))` — a distinct connection per database. No SQL-layer change.
  - **Model B (fallback):** a `TransactionProvider` wrapper that issues `SET search_path = <schema>` (via `SET LOCAL search_path`) as the first statement of every transaction on a shared connection. Implement as a thin decorator around the existing provider so it's swappable.
- [ ] Keep this behind a single factory function (`MakeTenantTransactionProvider(TenantContext)`) so switching models is a one-file change. Add a test asserting the factory yields a provider bound to the expected database name (Model C) / sets search_path (Model B) — use the existing transaction-provider test seams.

### 2.3 Wire the registry into `WebApp`
- [ ] `WebApp` gains a `TenantResolver` + `TenantResourceRegistry` (constructed in `main.cpp` from the control DB) **in addition to** the existing globals for now (don't rip out the single-tenant path yet — keep tests green; the legacy global provider becomes the fallback used only by tenant-agnostic endpoints like health).
- [ ] Update `main.cpp` to build the control helper + resolver + registry and pass them to `WebApp`. Log the count of active tenants at startup.
- [ ] Add/extend `web_app` construction tests to cover the new members (the existing `EndpointTestHelper` will need to provide a test resolver/registry — see Phase 3.4).

## Phase 3 — Request-edge tenant resolution (endpoints / auth layer)

Goal: every request resolves its tenant before touching the DB, and endpoints transparently use the tenant's provider. This is the layer where the multiplexing actually happens.

### 3.1 Read the site header
- [ ] Decide the header name: `X-Knotty-Site` (documented constant in one header). Read it via `req.get_header_value(...)` — same mechanism as `cloudfront_origin_guard.cpp`.
- [ ] Add a dev/local escape hatch: when the header is absent and a `KNOTTYYOGA_DEV_SITE_KEY` env var is set (or a single-tenant fallback flag), resolve to that tenant — so local `ng serve` + a single dev DB keeps working without CloudFront. (Mirrors the existing `KNOTTYYOGA_DEV_CORS_ORIGIN` dev-only pattern.)

### 3.2 Resolve tenant in `EndpointAuthHelper` before session init
- [ ] In `EndpointAuthHelper::Initialize()` (`endpoints/endpoint_auth_helper.cpp`): (1) read the site header; (2) `TenantResolver::Resolve`; (3) on miss → record the failure and surface a clean error (see 3.3); (4) `TenantResourceRegistry::GetOrCreate`; (5) store the resolved `TenantContext` + `shared_ptr<TenantResources>` on the helper; (6) initialize the session against the **tenant** provider.
- [ ] Add `EndpointAuthHelper::GetTransactionProvider()` to return the **tenant** provider (today it delegates to `WebApp`). Add `GetTenantContext()`, and tenant-scoped `GetSecretsHelper()/GetSquareClient()/GetMailHelper()` accessors (these return the tenant resources; Phase 4 fills them in — until then they can delegate to the global helper so the build stays green).

### 3.3 Tenant-resolution failure handling
- [ ] Define behavior for missing/unknown/suspended tenant on a normal `/api/*` request: respond `400`/`421 Misdirected Request` with a JSON error (`{"error":"unknown_site"}`), rate-limit the log like the origin guard does. Never fall through to a default tenant in prod (that would cross-serve data).
- [ ] Keep `/api/health` (and any other allow-listed, tenant-agnostic routes) working without a tenant — they must not call the tenant accessors. Audit the allow-list.

### 3.4 Migrate endpoints + test harness to the tenant provider
- [ ] Mechanically replace `webApp.GetTransactionProvider()` / `app_.GetTransactionProvider()` usage in endpoint and business-logic call sites with `endpointAuthHelper.GetTransactionProvider()` (the tenant one). This is the broadest-but-shallowest change; do it endpoint-by-endpoint. Most endpoints already go through `EndpointAuthHelper`, so the edit is localized.
- [ ] Update `endpoints/endpoint_test_helper.{h,cpp}` so tests construct a `TenantResolver`/registry pointing at the existing single test database under a known dev site key, and inject the `X-Knotty-Site` header (or use the dev fallback). **Goal: existing endpoint tests pass with minimal per-test edits** — ideally the test helper defaults a tenant so current tests need zero changes. Add a focused test proving two different site keys route to two different providers (can use the same physical test DB twice under two control rows to validate routing without a second schema).
- [ ] Add edge tests: missing header → error; unknown site → error; suspended → error; health works headerless.

## Phase 4 — Per-tenant services (secrets, payments, mail, site config)

Goal: the helpers that currently bake a single tenant's identity become per-tenant, sourced from the tenant DB and cached in `TenantResources`. Each subsection is lower-layer (util) before higher-layer (business-logic email bodies).

### 4.1 Per-tenant `SecretsHelper`
- [ ] Build the tenant's `SecretsHelper` from its own `DatabaseHelper` inside `TenantResources` (lazy). Keep the global at-rest `KNOTTYYOGA_SECRET_KEY` master key (it decrypts each tenant's `config_secrets.value`). Confirm `SecretsHelper` needs no interface change — only a different `DatabaseHelper`.
- [ ] Tests: a `SecretsHelper` built against tenant-A resources reads tenant-A values; ensure no cross-tenant bleed (two control rows → two helpers → distinct values). Reuse the test DB with two seeded secret sets if a second DB is impractical (Open Question 5).

### 4.2 Per-tenant `SquareClient`
- [ ] Move Square client construction out of `main.cpp` startup into `TenantResources` (lazy): read `square_access_token` + `square_environment` from the **tenant** secrets; build the client (sandbox/prod per tenant). Payment endpoints/business logic obtain the client via `endpointAuthHelper.GetSquareClient()`.
- [ ] Tests: tenant on sandbox vs tenant on prod yield correctly-configured clients; payment business-logic tests inject a test Square client per tenant (existing `MakeTestSquareClient` seam).

### 4.3 Per-tenant `MailHelper`
- [ ] Build `MailHelper` per tenant from tenant SMTP/SES secrets + sender identity. Email-sending business logic obtains it via the tenant accessor.
- [ ] Tests: sender identity / SMTP config reflect the tenant; existing mail tests adapted to the tenant accessor (test mail seam).

### 4.4 Split `ServerConfig` into global vs per-tenant
- [ ] Extract the genuinely-global bits (prod mode, trust proxy) into a process-wide `DeploymentConfig` initialized once at startup. Move per-tenant bits (website address, login address, CORS origin) into the tenant's `TenantResources`/site config, sourced from tenant secrets.
- [ ] Rework CORS: since each tenant is same-origin behind its own CloudFront, prod CORS is effectively a no-op; replace the single pinned-origin registration with a dynamic check that validates the request `Origin` against the **resolved tenant's** allowed origin (and keep the dev `KNOTTYYOGA_DEV_CORS_ORIGIN` path). Remove the `ServerConfig` global-singleton assumption.
- [ ] Tests: cookie `Domain`/`Secure` derive from the tenant's website address (extend the existing `SessionTest.InitializeFromLoginProdMode…` test to assert per-tenant domain); CORS accepts the tenant origin and rejects others.

### 4.5 Parameterize email templates by tenant branding
- [ ] Introduce a `TenantBranding { studioName; senderName; senderAddress; websiteUrl; }` assembled from tenant secrets, threaded into each mail-body builder. Replace hardcoded `"Knotty Yoga"` / `"Knotty Yoga and Spa"` literals in the ~15 `*_mail.cpp` builders with `FormatString` placeholders (per the project's template-constant convention; keep `NormalizeCrLf`).
- [ ] Tests: each mail builder with an existing `*_test.cpp` gets a case asserting the branding placeholder is substituted (studio name appears, no hardcoded literal). Enumerate the mail builders and check off one-by-one so none is missed.

## Phase 5 — Provisioning & migrations (database_helper / ops)

Goal: stand up and evolve tenant databases repeatably.

### 5.1 `--create-tenant` command
- [ ] Add a `knottyyoga_database_helper --create-tenant --site-key=<k> --db-name=<n> --display-name=<…>` mode: create the tenant database, run `CreateAndPopulateDatabases()` against it, provision the scheduler service account (`EnsureSchedulerServiceAccount`), then insert the `tenants` row in the control DB and `Invalidate` the resolver cache. Guard destructive re-create with the existing `KNOTTYYOGA_ALLOW_DESTRUCTIVE` gate.
- [ ] A companion `--seed-tenant-secrets --site-key=<k> --file=secrets.json` step (or reuse the existing secret-seeding workflow) to set that tenant's Square/mail/website secrets. Document the order: create-tenant → seed-secrets → tenant live.
- [ ] Tests for the control-row insert + idempotency (re-running create-tenant for an existing site key is a clean no-op or explicit error, not a duplicate).

### 5.2 Per-tenant migration loop
- [ ] Extend `--migrate` to iterate every active tenant from the control DB and run `MigrationRunner::ApplyPending(BuildAllMigrations())` against **each** tenant database (plus the control DB's own migration list). Per-tenant transaction semantics already hold (each migration in its own txn). Log per-tenant progress; on a tenant failure, record it and continue or abort per Open Question 6.
- [ ] Tests: a fixture migration list applies to N tenant DBs (or N control rows over the test DB) and is idempotent on re-run; a failing migration in one tenant is reported without silently skipping the rest unrecorded.

### 5.3 Onboarding runbook
- [ ] Document the end-to-end onboard: create DB + populate → seed secrets → create CloudFront distribution + S3 bundle + ACM cert + DNS → set the distribution's `X-Knotty-Site` (and shared `X-Origin-Secret`) origin headers → smoke-test `/api/health` then a tenant request. (Cross-references Phase 8 and `Deploying to AWS.md`.)

## Phase 6 — Scheduler multi-tenant

Goal: background jobs run for every tenant, not just one.

### 6.1 Tenant iteration
- [ ] `knottyyoga_helper` reads the active-tenant list from the control DB (or a config file mirroring it) at startup and on a refresh interval.

### 6.2 Per-tenant authenticated calls
- [ ] The scheduler's `api_client` sends `X-Knotty-Site: <site_key>` (and the existing `X-Origin-Secret`) on every call, and logs in per tenant as that tenant's `scheduler@knottyyoga.local` (each tenant DB has its own service-account row from Phase 5.1). Loop: for each active tenant → login → run the job set → logout/expire.
- [ ] Decide service-account password scope: shared `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` across tenants (simplest) vs per-tenant (Open Question 7). Tests for header injection + per-tenant login sequencing (existing scheduler test seams).

## Phase 7 — Frontend per-tenant branding

Goal: one Angular bundle serves all tenants; branding arrives at boot from the API. (Full theming is Website Makeover Phase 5; this is the minimal hook.)

### 7.1 `/api/site_info` endpoint (backend first, per layering)
- [ ] Add `endpoints/site_info.cpp` returning the resolved tenant's public branding (`display_name`, `logo_url`, `website_url`, theme tokens if available) — sourced from tenant secrets/branding; unauthenticated; cacheable. Reads the tenant via the same edge resolution.
- [ ] `site_info_test.cpp`: returns the resolved tenant's branding; unknown site → error; no auth required.

### 7.2 ServerAccess method
- [ ] Add `getSiteInfo()` to the `ServerAccess` interface, `ServerAccessNetwork`, `ServerAccess.mock.ts`, and the proxy; add `ServerAccess.mock.spec.ts` coverage (per the frontend testing rule).

### 7.3 Replace hardcoded branding
- [ ] Add an `APP_INITIALIZER` that fetches `/api/site_info` before render and exposes it via a `SiteConfigService`. Refactor `header.component.ts` (logo + alt text), `footer.component.html` (address/email/tagline/copyright), and `shared/constants/about.ts` to read from `SiteConfigService` with the current values as fallback.
- [ ] Component spec updates for header/footer/about reading from the service (per the "test every component change" rule).

### 7.4 Theming hand-off
- [ ] Note the dependency on Website Makeover Phase 5 (DB-driven theme tokens + `/api/site_theme`); align `site_info` so it can carry or coexist with theme tokens rather than duplicating them.

## Phase 8 — Infrastructure (per-tenant CloudFront / DNS / headers)

Goal: the AWS wiring that makes one origin serve many branded sites. (Ops, owned by you; code is ready after Phases 1–7.)

### 8.1 Per-tenant distribution
- [ ] Per tenant: S3 bundle (same artifact), CloudFront distribution, ACM cert (us-east-1), Route 53 records — following `Deploying to AWS.md` Phase 4/5 conventions.

### 8.2 Header injection
- [ ] On each distribution's `/api/*` behavior, set origin custom headers: the shared `X-Origin-Secret` (existing) **and** that tenant's `X-Knotty-Site=<site_key>`.

### 8.3 Origin-secret strategy
- [ ] Keep `X-Origin-Secret` deployment-wide for now (origin protection, not identity). Record the option to make it per-tenant later for defense-in-depth (Open Question 8).

---

# Part 3 — Open Questions

These are decisions I need from you rather than ones I should silently make. I've recommended a default for each so implementation can proceed if you don't object.

Mason- Please look at [[Splitting the server up into components]] and update this document based on the componentization effort.

1. **Isolation model — confirm Model C (database-per-tenant).** Part 1 recommends C with B as a swappable fallback and rejects A. *Default: proceed with C.* Confirm, or pick B/A.
	- Mason- I'll go with your recommendation.
2. **Header name & semantics.** I propose `X-Knotty-Site: <site_key>` where `site_key` is a short stable slug (e.g. `knotty`, `acme`). *Default: `X-Knotty-Site`.* OK, or different name/value (e.g. send the database name directly, or a UUID)?
	- Mason- Per [[Splitting the server up into components]], let's go with X-Honuware-Site
3. **Control-plane location.** A dedicated `knottyyoga_control` database on the same RDS instance holding `tenants`. *Default: dedicated control DB.* Alternative: a JSON/env tenant manifest for the very first soft-launch (simpler, not dynamic) — acceptable, or go straight to the control DB?
	- Mason- Let's go with honuware_control and do the control db.
4. **Connection-count ceiling.** One persistent libpqxx connection per active tenant. Fine to dozens; beyond ~50 we'd add PgBouncer or a bounded/evicting pool. *Default: one-per-tenant now, document the PgBouncer trigger.* Any near-term tenant-count target that would change this?
	- Mason- Do you think one connection for tenant will be too limited? Would it be worth it to have a pool of connections per tenant?
5. **Test strategy for multi-tenant routing.** The suite uses one shared test DB (`test_knottyyoga`). To test routing without standing up many DBs, I'll register **two control rows pointing at the same physical test DB under two site keys** and assert routing/caching/independence. *Default: that approach.* Or do you want a second real test database provisioned in the harness?
	- Mason- What do you recommend?
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

---

# Implementation Log

(Checkboxes above are unchecked — this plan is written but not yet implemented. As we implement, each item gets checked off here per the "mark completed" workflow. Backend layers land before frontend within each phase, per the established convention.)