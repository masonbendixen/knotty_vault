---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/6/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Please take a look at:

C:\Users\mason\Documents\Obsidian\Knotty Yoga\Projects\Authentication, security.md

I would like you to review this document and then the code for my authentication implementation (largely in src/business_logic/auth) and do a detailed security review. Please also look at my endpoints for the authentication being done as well as all of the places I build SQL statements for SQL injection issues. I'd really like a comprehensive review of my design documents and code base to harden the server. Please start with listing the issues and other things to worry about. Then we can work on phases of implementation to fix these issues.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here

# Findings Summary

A multi-agent review of `business_logic/auth/`, `endpoints/`, all SQL building paths, the Crow bootstrap, the database schema, and the Angular frontend produced the issues below. They are organized by severity. Each is referenced by `file:line`. The plan in the next section turns these into phased, layered work items with checkboxes.

## Critical
- **C1. CSRF protection is entirely missing.** No endpoint validates `X-CSRF-Token` and no client-side interceptor injects it. The design document calls for double-submit cookie + Origin/Referer check; neither side is wired. Cookie auth without CSRF on POST/PUT/DELETE means any other origin a logged-in user visits can forge state-changing requests (login, password change, payment). Files: every `endpoints/*.cpp` POST handler; `ui/src/app/shared/services/server-access.network.ts` (no interceptor reads `csrft`).
- **C2. Password is sent in the URL path on `/api/register`.** `endpoints/register.cpp` registers the route `/api/register/{first_name}/{last_name}/{email}/{password}`. URLs end up in access logs (CloudFront, any reverse proxy), browser history, the `Referer` header on outbound links, and `crow::request.url`. Fix is to switch the endpoint to a POST with a JSON body and update `ui/src/app/.../server-access.network.ts:206` to match.
- **C3. Hardcoded admin role assignment by first name.** `endpoints/account_activation.cpp:65-82` grants the admin role to anyone whose first name is "Mason" or "Tyler". Anyone can register `firstName=Mason` and become admin.
- **C4. Staff endpoints check only `IsLoggedIn`, not staff role/permission.** Multiple `/api/staff/*` endpoints (`staff_checkin`, `staff_create_quick_account`, `staff_dropin_booking`, `staff_search_people`, `staff_upgrade_session`, `staff_upgrade_options`, `staff_upcoming_checkins`) allow any authenticated user (i.e., any registered customer) to perform staff operations including arbitrary check-ins, account creation, and PII enumeration via `staff_search_people`.
- **C5. No login rate limiting or account lockout.** `endpoints/login.cpp` calls `VerifyPassword` with no per-IP / per-email throttle, no exponential backoff, no `failed_login_attempts` counter on `people`, and no `login_attempts` table. The design doc includes a `login_attempts` table that was never implemented. Brute-force is unconstrained. Same applies to `/api/verify` and `/api/remember`.
- **C6. `config_secrets.value` is stored in plaintext.** `db_schema/config_secrets.cpp` defines `value` as a plain string, and the table includes Gmail app passwords, Square access tokens, and other live credentials. Any database dump or admin-CRUD read of this table exposes them. The table is also exposed via `admin_top_level_tables` (`database_helper/create_database.cpp`).
- **C7. `sessions.uuid` and `device_tokens.uuid` have no unique constraint or index.** `db_schema/sessions.cpp` and `db_schema/device_tokens.cpp` declare `uuid` as `NOT NULL DEFAULT gen_random_uuid()` with no `UNIQUE` and no explicit index. Every authenticated request executes `LookupRowByValue(kSessionsUuid, uuid)`, which becomes a sequential scan as the table grows — a DoS vector and a logical correctness issue (two sessions could in principle share the same UUID).

## High
- **H1. Generic CRUD endpoints can reach sensitive tables.** `endpoints/{add_item,update_item,delete_item,get_row,get_rows_by_column,get_table_rows,get_filtered_table_rows,get_row_by_values}.cpp` rely on `IsTableAllowed()` (the union of `admin_top_level_tables`, `admin_nested_tables`, and the per-permission map). The `people` table is in this set, and the endpoints have no per-column redaction, so a `GET /api/get_row/people/id/N` returns `password_hash` to any user who has read access to the `people` table. `database_helper/create_database.cpp` `PopulateAdminTopLevelTables` and `PopulateAdminTablePermissions` need a column-level allow/deny list, or the endpoints need a redact pass.
- **H2. Image endpoints lack object-level authorization.** `endpoints/get_photo.cpp` requires only `IsLoggedIn`; any logged-in user can fetch any image keyed by `(table, item_id)`. `endpoints/get_scaled_photo.cpp` does **no** auth and creates a fresh DB connection per request — both an authz gap and a connection-exhaustion DoS vector.
- **H3. No CSRF / Origin / Referer enforcement on payment, purchase, and gift-permission endpoints.** `endpoints/{purchase_create,purchase_pay_card,payments,cards,change_purchase_recipient,gift_permissions}.cpp`. Combined with C1, these are forgery-prone. *Ownership checks confirmed correct* on follow-up: `change_purchase_recipient.cpp:99-106` rejects when `payerPersonId != loggedInPersonId`; `gift_permission_helper.cpp::AcceptRequest/DenyRequest/Revoke` verify the requesting person is the grantee (or grantor for revoke). Risk is therefore CSRF-only, addressed in Phase 4.
- **H4. Token-hash and verification-token comparisons use plain `==` (timing-leak surface).** `business_logic/auth/person.cpp:233` compares the email-verification stored hash to the supplied hash with `std::string::operator==`. Use `sodium_memcmp` or compare hashes only after a constant-time function. This also applies to any device-token-secret comparison done outside `crypto_pwhash_str_verify` (which is constant-time itself).
- **H5. Email-verification attempts increment is a TOCTOU race.** `sql_util/table_helpers/email_verifications.cpp` reads `attempts`, evaluates `attempts < limit`, then issues a separate `UPDATE` to bump `attempts` on failure. Two concurrent attackers can both observe the same low value and exceed the limit. Replace with a single `UPDATE ... SET attempts = attempts + 1 WHERE ... RETURNING ...` and decide based on the returned new count, or take a `SELECT ... FOR UPDATE` lock first.
- **H6. Device-token rotation is not atomic.** `business_logic/auth/person.cpp::TryLoginWithDeviceToken` checks the secret hash, then `UpdateDeviceToken` writes a fresh secret. Two parallel requests with the same device cookie can both pass validation before either writes. Wrap the read+update in a single SQL statement (e.g. `UPDATE ... SET secret_hash=$new, uuid=gen_random_uuid() WHERE secret_hash=$old AND NOT revoked AND now_us() < expires_at RETURNING ...`) and only mint a session on a non-empty RETURNING.
- **H7. No security headers on responses.** No middleware sets `Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, or `X-Frame-Options`. Crow also emits its `Server: Crow/...` banner.
- **H8. `email_verifications` allows multiple pending tokens per person.** `db_schema/email_verifications.cpp` does not enforce `UNIQUE(person_id)` (the original design did). A user — or attacker — can stack pending tokens for the same email. Either rotate (delete-then-insert) on every `AddEmailVerification*` call or add the unique constraint.
- **H9. `people.email` is a plain `text UNIQUE`, not case-insensitive.** `db_schema/people.cpp`. `Mason@Example.com` and `mason@example.com` are distinct rows. Switch to `citext` (preferred) or normalize to lower-case on insert and add a `CHECK (email = lower(email))`.
- **H10. Logout cookie-clearing has the wrong attributes.** `endpoints/logout.cpp` sets `SameSite=None` without `Secure` (browsers reject the attribute), and does not echo the `domain` and `secure` flags that were used at set-time. As a result the original cookies are not necessarily evicted. Match the attributes used in `business_logic/auth/session.cpp::SetCookieHelper`.
- **H11. No global Crow error handler.** `endpoints/web_app.cpp` and `main.cpp` do not register `app.error_handler(...)`. An exception escaping any handler returns Crow's default 500 page, which can include stack details depending on the build flavor. Several endpoints already echo exception text directly (`endpoints/db_schema.cpp:31` returns `e.what()`).

## Medium
- **M1. `register` and `verify` use GET-style URL semantics for state-changing operations.** Even after C2 is fixed, `/api/verify/{email}/{secret}` is conceptually a side-effect-bearing GET. Change to POST with body for both endpoints; this also removes the secret from logs.
- **M2. `me` is a POST.** `endpoints/me.cpp` registers POST. `me` is the canonical safe/idempotent read; should be GET. Same for any other status-style endpoint.
- **M3. `proxy_trust` is opt-in only via env var.** `business_logic/auth/proxy_trust.cpp:11`. If `KNOTTYYOGA_TRUST_PROXY` is unset behind CloudFront, the server uses the wrong client IP for any future rate-limit / abuse logic. Fail startup loudly (with override) when prod mode is on but `KNOTTYYOGA_TRUST_PROXY` is unset.
- **M4. CloudFront origin-secret guard silently disables when env var is unset.** `endpoints/cloudfront_origin_guard.cpp:27`. Acceptable for dev, dangerous if deployed without setting the secret. Fail startup in prod mode if `KNOTTYYOGA_ORIGIN_SECRET` is missing.
- **M5. Argon2id is using `OPSLIMIT_INTERACTIVE`.** `business_logic/auth/auth_helper.cpp:16-17`. Fine for low-value accounts; `MODERATE` (3 ops, 256 MB) is recommended. Make it a secret so it can be tuned without a redeploy.
- **M6. `role_assignments` and `role_permissions` allow duplicate `(parent, child)` pairs.** `db_schema/role_assignments.cpp`, `db_schema/role_permissions.cpp` — no unique constraint on the natural key. Add one.
- **M7. `admin_alerts` is logged but never delivered.** No code path emails / pages / surfaces alerts to a human; security-relevant events stay invisible. Either wire a polling job that emails admin on new rows, or surface them in the admin UI with a daily digest email.
- **M8. Open-redirect on the login page.** `ui/src/app/auth/login.component.ts:54-75` reads `returnUrl` and calls `router.navigateByUrl(returnUrl)` without an allow-list, accepting absolute URLs.
- **M9. No `APP_INITIALIZER` calling `/api/me` → `/api/remember`.** `ui/src/app/app.config.ts`. After page refresh the user appears logged out until they hit a protected route; the design doc explicitly calls for this bootstrap pattern. Reduces UX friction and avoids surprise 401 redirects on first action.
- **M10. CORS sets `Access-Control-Allow-Origin: <site>` but does not emit `Vary: Origin`** in `business_logic/auth/server_config.cpp::ConfigureCors`. Caches in front of the API can serve cross-origin responses to the wrong origin.
- **M11. SQL string concatenation in `database_helper/create_database.cpp:2031,2049,2062`.** Currently safe (the spliced values are hard-coded), but it's a divergence from the codebase's parameterization discipline and a foot-gun for the next maintainer.
- **M12. (Resolved on follow-up.)** `change_purchase_recipient.cpp:99-106` and `gift_permission_helper.cpp::AcceptRequest/DenyRequest/Revoke` already check ownership. Plan item 3.9 reduces to adding regression tests so this stays true.

## Low
- **L1. Argon2id hash buffer not zeroed after use.** `business_logic/auth/auth_helper.cpp` returns the hash by value but does not call `sodium_memzero` on intermediate buffers. Lowest priority, real-world impact is small.
- **L2. `people` table has `must_change_password` but no force-rotate-after-N-days secret.** Optional.
- **L3. No log line on successful auth events.** Hampers incident response. Log `login_success`, `login_failure`, `device_token_used`, `password_changed`, `email_verified` at INFO with the person_id and remote IP.
- **L4. Crow `Server:` banner leaks framework + version.** `business_logic/auth/server_config.cpp` or response middleware can override.
- **L5. `email_verifications.token_hash` is not indexed.** Lookups are by `person_id` (the FK), so this is fine today, but if a future code path looks up by hash it will scan.
- **L6. Idle-session client-side timeout is absent.** Server-side TTL (8h via `kAuthSessionMaxDuractioninMicros`) handles it; client-side warning would be polish.

## Notes (deliberately not findings)
- SQL injection: the parameterization audit found **no exploitable injection** in runtime code. `EscSqlTableName/ColumnName` validate identifiers against `^[a-zA-Z_][a-zA-Z0-9_]*$` and otherwise quote, the JSON CRUD layer rejects unknown column names against `DbMeta::ListColumns`, and `allowedSqlKeywords` is never populated from user input. Keep this discipline.
- Cookie attributes on the *set* path are correct (`HttpOnly`, `SameSite=Lax`, `Secure` in prod, domain from `kWebsiteAddress` in prod). Only the clear path (logout) is wrong (H10).
- Argon2id verification uses `crypto_pwhash_str_verify`, which is constant-time — passwords are fine. The non-constant-time issue (H4) is for the *email-verification token hash*, not the password.

# Implementation Plan

The fixes are grouped into phases that respect the layering of the system: each phase starts at the lowest layer touched (db schema → table helpers → business logic → endpoints → frontend) before moving up. Tests are mandatory for every code-bearing item per `feedback_always_test.md` / `feedback_test_every_cpp_change.md`.

## Resolved Decisions

The architectural decisions below were settled before drafting the per-phase work items; their rationale is baked into the relevant phase body (column on the right). Recording them here so the plan is self-contained.

| Topic | Decision | Phase |
|---|---|---|
| CSRF model | Double-submit cookie (`csrft` non-HttpOnly + `X-CSRF-Token` header) plus `Origin` allow-list. Stateless, fits multi-instance ECS, industry default for SPA-cookie auth. | 4 |
| Verification email | Email link points at SPA route `/verify?email=…&secret=…`. SPA reads params, POSTs to `/api/verify`, `history.replaceState`s the URL bar to scrub the secret. Default verification window reduced to **1 hour**. | 3.3 |
| Argon2id strength | `MODERATE` (3 ops, ~256 MB). ~5× the offline-attack cost of `INTERACTIVE` for ~200 ms extra latency once per 8 h. Tunable via `kAuthArgon2OpsLimit` / `kAuthArgon2MemLimitKb` secrets. | 2.5 |
| Dev CORS | No-op by default (Angular proxy keeps everything same-origin). Opt-in via `KNOTTYYOGA_DEV_CORS_ORIGIN` env var when developers want to hit the API directly without the proxy. | 7.2 |
| Lockout thresholds | Per-email 10 fail / 15 min → 30 min lockout. Per-IP 50/15-min → 1 h block. Verify 30/15-min/IP → 30 min. Remember 50/15-min/IP → 1 h. All secret-driven; generic 401/429 responses never reveal lockout reason. | 5 |
| Secrets-at-rest key | **v1 (now):** env var `KNOTTYYOGA_SECRET_KEY`, injected by ECS task definition from AWS Secrets Manager. **v2 (deferred):** AWS KMS + envelope encryption. v1 needs only `std::getenv` + base64 — no AWS SDK in the C++ build. | 8 |
| Mason/Tyler hardcode | Delete outright. Admins are now minted via seed data and the role-management UI. | 3.5 |
| Sensitive column protection | Column-level redact map populated in `create_database.cpp`, enforced at the JSON boundary in `database_rest_helper.cpp`. Reconsider only if `people` is ever exposed to anonymous reads. | 3.8 |
| Secrets admin UI | Dedicated endpoint + UI; values masked, never returned to client; every write audited to `admin_alerts`. Generic-CRUD editor for `config_secrets` is removed. | 8.3 |
| SameSite policy | Stay with `Lax` for all auth cookies. `Strict` would break email-driven `/verify` flow. | 4.1, 6 |
| Rate-limit persistence | PostgreSQL via the existing `ThreadPool` with **write-behind**: synchronous read for the threshold check, synchronous lockout decision, async write of attempt records. Multi-instance ECS safe; login latency unchanged because Argon2id dominates. | 5 |
| `/api/me` HTTP method | Switch from POST to GET. No contract break — pre-deploy. | 3.4 |
| Ownership checks on `change_purchase_recipient` / `gift_permissions` | Verified correct on follow-up. Plan reduces to adding regression tests for the negative cases. | 3.9 |

## Implementation Order

Phases are numbered by topic (so the doc reads as a logical grouping), but they ship in the order below. Earlier work is risk-free / unblocks later work; nothing here forces a serial bottleneck on Mason once Phase 1 is in.

1. **Phase 1** — Schema integrity & lookup performance. Everything later depends on these tables/indices.
2. **Phase 7** *(pulled forward)* — Security headers, error handler, server banner. Single middleware change, zero behavior risk, immediate prod-readiness win.
3. **Phase 12.1** *(pulled forward)* — Fail-loud startup guards. Tiny, isolated, prevents whole classes of "deployed without `X` set" foot-guns.
4. **Phase 2** — Auth primitives hardening. Adds the constant-time helper that Phase 4 depends on, plus atomic verification / device-token rotation.
5. **Phase 3** — Endpoint authn/authz corrections. Register-as-POST, kill the Mason/Tyler hardcode, staff role gate, image authz, redact map.
6. **Phase 6** *(pulled ahead of Phase 4)* — Cookie hygiene cleanup. The shared `ClearAuthCookies` helper lands here so Phase 4 can reuse it for `csrft`.
7. **Phase 4** — CSRF middleware (server) + interceptor (client).
8. **Phase 5** — Rate limiting + lockout. Can run in parallel with Phase 4.
9. **Phase 8** — Secrets at rest + admin secrets endpoint. Independent; slot in any time after Phase 1.
10. **Phase 9** — Auth event log + admin_alerts delivery. Builds on phases 5 and 8.
11. **Phase 10** — Frontend bootstrap + returnUrl allow-list. Independent of backend phases; can be parallelized.
12. **Phase 11** — Latent SQL-concat cleanup in `create_database.cpp`. Pure code-quality cleanup; pair with Phase 1 if convenient.

## Phase 1 — Schema integrity & lookup performance

**Why first**: every later auth fix sits on top of these tables. Adding indices + uniqueness now also avoids painful migrations after data accumulates.

### 1.1 Add unique index on `sessions.uuid` (db_schema layer)
- [x] Added `AddUniqueConstraint(kSessionsTable, kSessionsUuid)` in `db_schema/sessions.cpp::MakeSessionsTable`. (Used the existing table-level unique-constraint DSL rather than introducing a separate `AddIndex` primitive — Postgres implements `UNIQUE` via a unique index, so the lookup performance and correctness goals are met.)
- [x] Verified the existing `db_and_table_operations.cpp::GenerateCreateTableSql` emits `UNIQUE (col)` correctly for table-level constraints. No DSL extension needed.
- [x] Added `db_schema/sessions_test.cpp` asserting the constraint is present in `DatabaseInfo` and that the DDL emits `UNIQUE (uuid)`.
- [x] Added `db_and_table_operations_test.cpp::GenerateCreateTableSqlSingleColumnUniqueConstraint` regression — confirms the table-level `UNIQUE (col)` round-trips correctly when paired with `NOT NULL DEFAULT GEN_RANDOM_UUID()` (the sessions/device_tokens uuid pattern).

### 1.2 Add unique index on `device_tokens.uuid`
- [x] Added `AddUniqueConstraint(kDeviceTokensTable, kDeviceTokensUuid)` in `db_schema/device_tokens.cpp::MakeDeviceTokensTable`.
- [x] Added `db_schema/device_tokens_test.cpp` asserting uniqueness at both metadata and DDL level.

### 1.3 Add `UNIQUE(person_id)` to `email_verifications`
- [x] Added `AddUniqueConstraint(kEmailVerificationsTable, kEmailVerificationsPersonId)` in `db_schema/email_verifications.cpp`.
- [x] Rewrote `sql_util/table_helpers/email_verifications.cpp::AddEmailVerificationById` to `DELETE` any existing row for the person *in the same transaction* before insert (replaces the old throw-on-existing behavior). Replaced the `AddEmailVerificationByIdAlreadyExists` test with `AddEmailVerificationByIdReplacesExisting` and added `AddEmailVerificationByEmailReplacesExisting`. Both assert exactly one row remains, the old PK is gone, and the row carries the new hash.
- [x] Added `db_schema/email_verifications_test.cpp`.
- [x] Backfill: dev-only data, no migration needed.

### 1.4 Add `UNIQUE(person_id, role_id)` to `role_assignments`
- [x] Added `AddUniqueConstraint(kRoleAssignmentsTable, kRoleAssignmentsPersonId, kRoleAssignmentsRoleId)` in `db_schema/role_assignments.cpp`.
- [x] Added `RoleAssignmentsTest::AddRoleAssignmentDuplicate` in `sql_util/table_helpers/role_assignments_test.cpp` — asserts `EXPECT_THROW` on duplicate `(person_id, role_id)`.
- [x] Added `db_schema/role_assignments_test.cpp`.

### 1.5 Add `UNIQUE(role_id, permission_id)` to `role_permissions`
- [x] Added `AddUniqueConstraint(kRolePermissionsTable, kRolePermissionsRoleId, kRolePermissionsPermissionId)` in `db_schema/role_permissions.cpp`.
- [x] Added `RolePermissionsTest::AddRolePermissionDuplicate` in `sql_util/table_helpers/role_permissions_test.cpp`.
- [x] Added `db_schema/role_permissions_test.cpp`.

### 1.6 Make `people.email` case-insensitive (via `DB_TYPE_CITEXT`)
**Approach:** Add Postgres's `citext` extension and surface it as a first-class `DB_TYPE_CITEXT` in the schema DSL. Every comparison, unique check, and index lookup on a citext column is automatically case-insensitive at the DB level — no per-call discipline required. The reverse-engineering path is generalized at the same time so the next user-defined Postgres type (hstore, ltree, custom enums, PostGIS geometry, …) drops into the DSL with one enum addition instead of needing a special case in the metadata reader. This is the platform-quality move.

**DSL extension (lower layer first):**
- [x] Added `DB_TYPE_CITEXT` to `DatabaseTypes` in `sql_util/database_common.h`.
- [x] `sql_util/schema/column_info.cpp::SqlTypeFromDatabaseType` maps `DB_TYPE_CITEXT` → `"CITEXT"`.
- [x] `sql_util/schema/column_info_test.cpp` — extended `SqlTypeFromDatabaseTypeBasic` and added `GetSqlTypeCitext` covering both the static helper and instance-level `GetSqlType()`.
- [x] `sql_util/database_access/database_metadata.cpp::DatabaseTypesFromDatabaseColumnInfo` — refactored to two `static const` maps: `kPrimaryLookup` (keyed on `data_type`, now includes `"CITEXT"`) and `kUserDefinedLookup` (keyed on `udt_name`, consulted when `data_type == "USER-DEFINED"`). Unknown user-defined types throw with the offending `udt_name` in the message so the next maintainer knows what to add.
- [x] `sql_util/database_access/database_metadata_test.cpp` — added five tests: `…CitextViaPrimaryLookup`, `…CitextViaUdtNameFallback`, `…UnknownUserDefinedThrows` (pure-unit), and `ListColumnsCitextRoundTrip` + `DatabaseInfoFromDatabaseWithCitextColumn` (live DB).
- [x] `sql_util/database_access/db_and_table_operations_test.cpp::GenerateCreateTableSqlEmitsCitext` — DDL emit regression confirming `email CITEXT UNIQUE`.

**Schema bootstrap:**
- [x] Created `sql_util/database_access/extensions.{h,cpp}` with `Extensions::CreateExtensions(transaction)` and `Extensions::GenerateCreateExtensionsSql()`. Internally a `kExtensionNames` array — adding the next extension (pgcrypto, hstore, …) is one line. Production path loops one statement per extension because `RunSqlStatement` uses pqxx `exec_params0` (single-statement only); the test batch path uses `pqxx::work::exec` which supports multi-statement strings.
- [x] `sql_util/database_access/extensions_test.cpp` — `CitextExtensionPresentAfterBootstrap` verifies the extension exists in `pg_extension` after the test bootstrap; `GenerateCreateExtensionsSqlIncludesCitext` covers the DDL string.
- [x] `database_helper/create_database.cpp` calls `Extensions::CreateExtensions(transaction)` before `CreateStoredProceduresBeforeTables` and `CreateTables`.
- [x] `test/src/util/global_database_test_support.cpp::SetupAllTables` — `GenerateCreateExtensionsSql()` is the first thing prepended to the batch.

**Apply to `people.email`:**
- [x] `db_schema/people.cpp::MakePeopleTable` — `kPeopleEmail` flipped from `DB_TYPE_STRING` to `DB_TYPE_CITEXT`.
- [x] Added `db_schema/people_test.cpp` with `EmailColumnIsCitext` and `GenerateCreateTableSqlEmitsCitextEmail`. Registered in `db_schema/CMakeLists.txt`.
- [x] `sql_util/table_helpers/people_test.cpp::AddPersonEmailCaseInsensitive` — inserts `Mason@Example.com`, asserts both `mason@example.com` and `MASON@EXAMPLE.COM` lookups return the same row, that the stored email retains its original case, and that a second insert with `mason@example.com` throws on the unique constraint.

**Notes left for future cleanup:**
- The DDL generator skips `NOT NULL` when a column is marked unique (`db_and_table_operations.cpp:46-50`). For `people.email` that means the emitted DDL is `email CITEXT UNIQUE` with no `NOT NULL`. Existing pre-1.6 quirk — not a regression — flagging in case strict NOT NULL semantics matter later.
- The single-column-uuid round-trip caveat from Phase 1.1–1.5 still stands (defaults are lost in the metadata reverse-engineering for `sessions.uuid` / `device_tokens.uuid`). 1.6 fixed user-defined-type round-trip, but the defaults gap is an independent future fix.

### 1.7 Add lockout columns to `people`
- [x] Added `kPeopleFailedLoginAttempts` (BIGINT NOT NULL DEFAULT 0) and `kPeopleLockedUntil` (BIGINT NULL — microseconds since epoch, NULL = not locked) to `db_schema/people.{h,cpp}`.
- [x] Did NOT add entries in `PopulateAdminColumnDataInfo` / `PopulateAdminColumnFriendlyNames` — the columns are auth-internal and stay invisible to the admin UI. The remaining JSON-level exposure via the generic CRUD endpoints is closed by the redact map in Phase 3.8 (which will list both columns alongside `password_hash`).
- [x] `db_schema/people_test.cpp` — added `FailedLoginAttemptsColumnIsBigintWithZeroDefault`, `LockedUntilColumnIsNullableBigint`, `GenerateCreateTableSqlEmitsLockoutColumns`.
- [x] `sql_util/table_helpers/people_test.cpp::AddPersonInitializesLockoutColumns` — confirms a freshly added person has `failed_login_attempts=0` and `locked_until` NULL/empty.
- [x] Updated skip-list comparisons in 9 test files so the two new columns are excluded from full-row equality checks: `sql_util/table_helpers/people_test.cpp`, `sql_util/json/database_rest_helper_test.cpp`, and the endpoint tests `add_item`, `add_item_fetch_primary_key`, `delete_item`, `update_item`, `get_filtered_table_rows`, `get_rows_by_column`, `get_row_by_values`.
- [x] Phase 5 will wire up reads/writes; this phase only added the schema slots. **Latent maintenance smell flagged:** the skip-list pattern is brittle — every column added to `people` requires touching ~25 test sites. If this happens again, worth introducing a `kPeopleAutoSetColumns` constant so it's a one-line update.

### 1.8 Remove `config_secrets` from generic CRUD exposure
- [x] `database_helper/create_database.cpp::PopulateAdminTopLevelTables` — `config_secrets` removed from the admin-CRUD allow set. (`sessions`, `device_tokens`, `email_verifications` were never in the set.)
- [x] Also removed `config_secrets` admin metadata: column data info (`PopulateAdminColumnDataInfo`), column friendly names (`PopulateAdminColumnFriendlyNames`), and table friendly name (`PopulateAdminTableFriendlyNames`). The table still exists and is still seeded; the bootstrap `INSERT INTO config_secrets …` path is unchanged.
- [x] Frontend mock + integration spec updated to match: `ServerAccess.mock.ts` no longer includes `config_secrets` in the schema or root_tables, and the integration spec asserts `config_secrets` is **not** present (positive negative-assertion). Generic admin UI iterates `getDbSchema().root_tables`, so config_secrets simply stops appearing.
- [x] `endpoints/get_row_test.cpp` — added two regressions: `GetRowConfigSecretsAnonymousForbidden` and `GetRowConfigSecretsAdminForbidden`. Both expect HTTP 400 with `Table(config_secrets) is not an allowed table.` The admin variant adds a sentinel admin table to `admin_top_level_tables` and sanity-checks that the admin login path works against the sentinel before asserting config_secrets is rejected — so the regression catches an accidental re-add to `admin_top_level_tables`, not just a broken admin login.
- [x] Dedicated `/api/admin/secrets` endpoint with masking and audit logging: deferred to Phase 8.3 as designed.

**Notes:**
- Dev/ops tools under `src/test_helper/*` access `config_secrets` via direct SQL (not generic CRUD), so they're unaffected.
- `SecretsHelper` (the in-process secrets accessor used by the auth path, mailer, Square client, etc.) reads/writes the table directly via its own table helper. It's unaffected by the CRUD allow-set change.

## Phase 2 — Auth primitives hardening (lower layers, no behavior changes user-visible)

### 2.1 Constant-time hash comparison helper
- [x] Added static `AuthHelper::ConstantTimeEqual(string_view, string_view)` backed by `sodium_memcmp`. Length mismatch short-circuits to false (the size of one operand isn't sensitive in our threat model). Empty/empty returns true.
- [x] Tests cover `Match`, `Mismatch` (including a one-byte-different-at-end variant), `LengthMismatch`, `EmptyEqualsEmpty`, and `HandlesEmbeddedNull` (guards against a future refactor swapping in a strcmp-style compare).

### 2.2 Use the helper for the email-verification token-hash compare
- [x] `sql_util/table_helpers/email_verifications.cpp::DoEmailVerification` now compares `storedTokenHash` to `tokenHash` via `Auth::AuthHelper::ConstantTimeEqual`.
- [x] Added `DoEmailVerificationConstantTimeHashCompare` test that uses an embedded-NUL hash to confirm byte-exact comparison without C-string truncation.

### 2.3 Atomic email-verification attempt accounting
- [x] Replaced SELECT-then-UPDATE with a single conditional `UPDATE email_verifications SET attempts = attempts + 1 WHERE person_id = $1 AND attempts < $2 AND expires_at > now_us() RETURNING id, token_hash, attempts`. The counter saturates at the limit (no off-by-one over-count), the WHERE filter blocks expired rows, and the RETURNING clause feeds the constant-time hash compare.
- [x] On success, the row is `DELETE`d in the same transaction so a concurrent caller can't replay it.
- [x] On failure when `newAttempts >= attemptLimit`, fire the `admin_alerts` row.
- [x] Updated `DoEmailVerificationTooManyAttempts` to assert `attempts == 2` (saturates at the limit; old behavior went to limit+1).
- [x] Added `DoEmailVerificationConcurrentAttempts`: two sequential failed attempts → exactly `attempts == 2`.
- [x] Added `DoEmailVerificationNoRowReturnsFalse`: missing verification row returns false rather than throwing — keeps failure modes indistinguishable from wrong-hash / expired / over-limit.

### 2.4 Atomic device-token rotation
- [x] Added `DeviceTokens::ConsumeAndRotate(transaction, oldSecretHash, oldUuid, newSecretHash, microsUntilExpires, outPersonId, outNewUuid)`. Single SQL: `UPDATE device_tokens SET secret_hash=$3, uuid=gen_random_uuid(), last_used_at=now_us(), expires_at=now_us()+<micros> WHERE secret_hash=$1 AND uuid::text=$2 AND NOT revoked AND now_us() < expires_at RETURNING person_id, uuid`. WHERE matches both the secret hash AND the uuid (the public side of the cookie) — defense in depth against captured-hash-with-stale-uuid attacks.
- [x] `PersonHelper::TryLoginWithDeviceToken` now mints the replacement secret/hash up front, then calls `ConsumeAndRotate`. The old read+UUID-compare+IsValid+UPDATE+lookup chain is gone.
- [x] Tests: `ConsumeAndRotateBasic` (rotation + state mutation), `ConsumeAndRotateUnknownSecretHash`, `ConsumeAndRotateUuidMismatch` (defense-in-depth — same hash, wrong uuid still fails), `ConsumeAndRotateRevoked`, `ConsumeAndRotateExpired`, `ConsumeAndRotateContentionOnlyOneWins` (two sequential calls — first wins, second fails because the WHERE no longer matches).

### 2.5 Argon2id parameter bump (and make tunable)
- [x] Added `kAuthArgon2OpsLimit` and `kAuthArgon2MemLimitKb` to `secret_keys.h`; production defaults are MODERATE (3 ops, 262144 KB) in `secret_values.cpp`. Both registered in `FillInSecretsStringView`.
- [x] `AuthHelper` now has three forms of `HashPassword`: instance method with default MODERATE cost, instance method with explicit `(opsLimit, memLimitBytes)`, and static `HashPasswordWithSecrets(transaction, secrets, password)` that reads the two new secrets and falls back to MODERATE if either is missing/empty/unparseable. Bounds-checked against libsodium's MIN/MAX so a misconfigured secret can't drive cost below the floor.
- [x] `PersonHelper::PreliminaryRegisterPerson` now uses `HashPasswordWithSecrets`. `CreateFullyValidatedUser` and `UpdatePassword` got an optional `Secrets::SecretsHelperPtr secrets` parameter — production callers pass it (Argon2 driven by secrets); test/dev-tool callers omit it (fall back to a fast INTERACTIVE-cost hash). Bootstrap (`PopulatePeople` in `create_database.cpp`) and `endpoints/update_user_password.cpp` now pass secrets.
- [x] **Test-suite speed preserved.** `SecretsHelperTestImpl` overrides `kAuthArgon2OpsLimit`/`kAuthArgon2MemLimitKb` to INTERACTIVE values (2 ops, 64 MB) right after loading prod defaults. Every test that obtains a SecretsHelper via `MakeTestSecretsHelper()` (or via `EndpointTestHelper`) automatically gets fast Argon2 hashes without per-test setup.
- [x] Tests: `HashPasswordRespectsExplicitCost` (explicit args plumb through), `HashPasswordWithSecretsRespectsOpsLimitSecret` (secrets drive cost; encoded hash carries the right t= and m= parameters), `HashPasswordWithSecretsFallsBackToModerateOnMissingSecrets`, `HashPasswordWithSecretsFallsBackToModerateOnNullSecrets`, `HashPasswordWithSecretsRespectsTestHelperInteractiveOverride` (locks in the test-helper override).
- [x] Existing hashes still verify: `crypto_pwhash_str_verify` reads cost from the encoded hash, so users with INTERACTIVE-cost hashes from earlier still log in fine until their next password change.

### 2.6 Zeroize sensitive buffers on the password path
- [x] `AuthHelper::HashPassword` calls `sodium_memzero` on the local libsodium output buffer after copying it to the return string, both on success and on error. Best-effort — the returned `std::string` still lives in the caller's storage, but the local stack buffer doesn't linger.

## Phase 3 — Endpoint authentication & authorization corrections

### 3.1 Move `/api/register` from URL path to JSON body — backend
- [x] Route in `endpoints/register.cpp` is now `POST /api/register` reading `{ first_name, last_name, email, password }` from `req.body` JSON via `Json::Value::FromText`. Each missing/empty field returns `ValidationError` (400). `register.h` signature updated to take `const Json::Value& message`. No legacy URL-path form remains.
- [x] Field validation preserved; `PreliminaryRegisterPerson` call path unchanged.
- [x] `endpoints/register_test.cpp` — added `BuildRegisterBody` / `IssueRegisterRequest` helpers, flipped every existing test to body-based POST, added regressions: `RegisterMissingBody`, `RegisterMissingFieldFirstName/LastName/Email/Password`, `RegisterEmptyFieldRejected`, `RegisterNoLongerAcceptsLegacyPathParameters`.
- [x] Side effect: `gift_permissions_endpoint_test::RegistrationConvertsInvitationToGiftPermission` was using the legacy GET URL — rewritten to POST + JSON body.

### 3.2 `/api/register` — frontend
- [x] `ServerAccessNetwork.register(...)` now does `http.post('/api/register', { first_name, last_name, email, password }, { withCredentials: true })`. `withCredentials` retained so the session cookie set in the response sticks.
- [x] `register.component.ts` call site verified — already passes the four fields, no shape change needed.
- [x] Mock + spec parity: `ServerAccess.mock.ts::register` and `ServerAccess.mock.spec.ts` track the new shape.
- [x] `register.component.spec.ts` — existing success/failure branch assertions still cover the body-based call.

### 3.3 `/api/verify` — SPA-routed POST with 1-hour TTL (Decision #2)
The verification email link points at the SPA, not the API. The SPA reads the secret from the query string, immediately POSTs to `/api/verify`, then scrubs the URL via `history.replaceState` so the secret doesn't linger in browser history.

**Backend:**
- [x] `endpoints/verify.cpp` — `POST /api/verify` reading `{ email, secret }` from JSON body. URL-path-secret variant gone. Failures return generic `ValidationError` (400) — caller can't distinguish "no row" / "expired" / "wrong secret" / "attempt limit hit".
- [x] `verify_test.cpp` — every test flipped to body-based POST; added `VerifyMissingBody`, `VerifyMissingFieldEmail/Secret`, `VerifyWrongSecretReturnsGenericValidationError`, `VerifyNoLongerAcceptsLegacyPathParameters`.
- [ ] Add the verify endpoint to the CSRF-exempt allow-list in Phase 4 (it's a bootstrap endpoint — the user has no `csrft` cookie yet) but enforce a strict `Origin` check there. *(deferred to Phase 4)*
- [x] **Verification window default reduced** — `kEmailVerificationExpirationWindowInMicrosValue` in `util/secrets/secret_values.cpp` is `"3600000000"` (one hour). Still secret-overridable; only the default changed.

**Email content:**
- [x] `business_logic/auth/person_verify_mail.cpp` — template URL is now `{base}{kWebsiteActivationLink}?email={url-encoded-email}&secret={url-encoded-secret}`. `kWebsiteActivationLinkValue` is `"verify"` (no `/api/` prefix). `person.cpp::PreliminaryRegisterPerson` swapped to `kWebsiteAddressLogin` so the link points at the SPA host, not the API host.
- [x] `person_verify_mail_test.cpp` and `person_verify_mail_test_util.cpp` — expected URL shape is `https://example.com/verify?email=user@example.com&secret=ABC123`.

**Frontend:**
- [x] Angular route `/verify` registered in `ui/src/app/pages/auth/auth.routes.ts:12` → new `VerifyComponent` under `ui/src/app/pages/auth/verify/`.
- [x] `verify.component.ts::ngOnInit` reads `email` + `secret` from `ActivatedRoute.snapshot.queryParamMap`, fires `serverAccess.verify(email, secret)`, and calls `history.replaceState({}, '', '/verify-success')` **before** awaiting the response — the secret is gone from the URL bar by the time the POST resolves.
- [x] On success: navigate to `/login` with the verified toast.
- [x] On failure (or missing params): navigate to `/login` with a generic "verification failed or expired" toast. Server-supplied messages are never echoed.
- [x] Tests: `verify.component.spec.ts` covers success, failure, missing-params, and the URL-scrub-before-response timing assertion. `ServerAccess.mock.spec.ts` covers the new `verify(email, secret)` mock method.

### 3.4 `/api/me` — switch to GET
- [x] `endpoints/me.cpp` registers `crow::HTTPMethod::Get` (HandlePost renamed to HandleGet for clarity).
- [x] `me_test.cpp` updated to use GET.
- [x] Frontend: `ServerAccessNetwork.ts::me()` switched from `http.post` to `http.get`. Phase 4's eventual CSRF middleware will naturally skip safe methods.

### 3.5 Remove hardcoded "Mason"/"Tyler" admin grant
- [x] `endpoints/account_activation.cpp` — name-based branch deleted; comment in the source explains why.
- [x] `account_activation_test.cpp::AccountActivationDoesNotGrantAdminByName` (firstName=Mason) and `…AndNameTyler` (firstName=Tyler) — both register/activate the user and assert no admin role assignment lands. Belt-and-suspenders so a future regression on either hardcoded name is caught.
- [x] Existing seed-data path in `create_database.cpp::PopulateRoleAssignments` continues to mint admins.

### 3.6 Staff endpoints — enforce a `staff_access` permission
- [x] Added `kPermissionStaffAccess = "staff_access"` to `db_schema/permissions.h`.
- [x] Seeded the permission row in `create_database.cpp::PopulatePermissions` and linked `admin → staff_access` in `PopulateRolePermissions` (id 8 — order matters, comments call it out).
- [x] Added `EndpointAuthHelper::RequirePermission(transaction, name, resp)` — returns true on allow; on deny, writes `NotAuthenticated` (401) when not logged in or `NotAuthorized` (403) when missing the permission. Catches the underlying `runtime_error` from `Session::ActiveUserHasPermission` so an unseeded permission name fails closed without leaking the message.
- [x] Wired into all seven staff endpoints inside the existing `RunInTransaction` block: `staff_checkin.cpp`, `staff_create_quick_account.cpp`, `staff_dropin_booking.cpp`, `staff_search_people.cpp`, `staff_upgrade_session.cpp`, `staff_upgrade_options.cpp`, `staff_upcoming_checkins.cpp`.
- [x] Added `EndpointTestHelper::GrantPermissionToPerson(transaction, personId, name)` so each test setup is a one-line addition. Updates the existing `Setup*` helpers in all seven `staff_*_test.cpp` files so the existing happy-path tests continue to exercise success.
- [x] Added `NotStaffForbidden` regression tests for `staff_create_quick_account` and `staff_search_people` (the two highest-value gates — account creation and PII enumeration). Both expect HTTP 403 when a logged-in customer hits the endpoint without the permission.
- [ ] **Follow-up**: add `NotStaffForbidden` tests for the other five (`staff_checkin`, `staff_dropin_booking`, `staff_upgrade_session`, `staff_upgrade_options`, `staff_upcoming_checkins`). The gate is wired in production code and the existing tests grant the permission, so the success path is covered; only the negative regression is missing for these five. Pattern is identical to the two that exist — copy/paste with the right URL.

### 3.7 Image endpoint authorization
- [x] `endpoints/get_photo.cpp` — kept the `IsLoggedIn` check, added `IsTableAllowed` so the active user must have read access to the requested table. Anonymous callers still get 401; authenticated users without permission to the table get 400.
- [x] `endpoints/get_scaled_photo.cpp` — fully rewritten: removed the dedicated DB connection (a connection-exhaustion DoS vector), routed through `EndpointAuthHelper`, added the same `IsLoggedIn + IsTableAllowed` gate. Public-image flows (home page carousel) keep using the dedicated `home_page_photos` endpoint.
- [x] Tests: `GetPhotoForbiddenTable` (logged-in non-admin gets 400), pre-existing `GetPhotoNotLoggedIn` still passes. `get_scaled_photo_test.cpp` rewritten to go through `handle_full` for full-stack coverage — added `Success`, `Returns304WhenETagMatches`, `NotFound`, `NotLoggedIn`, `ForbiddenTable`. Updated the existing test setup helpers to grant admin (matching the new permission requirement).

### 3.8 Strip sensitive columns from generic CRUD reads via column-level redact map (Decision #8)
**Approach:** column-level redact map, not a separate `people_credentials` table. Rationale:
1. **No surgery in the auth path.** `PersonHelper::VerifyPassword` reads `people.password_hash` directly today; splitting tables forces a join in the most security-critical code path. High risk for marginal gain.
2. **Single source of truth.** The map lives next to the existing admin metadata in `create_database.cpp`. A test ensures any column flagged as redacted never appears in a JSON response.
3. **Enforced at the JSON boundary**, so it covers every endpoint that uses `JsonFromDataResults` (the entire generic CRUD layer + any handler that uses it).
4. **The exception** that would change this decision: if we ever expose `people` to anonymous reads (e.g., a public studio directory). At that point a separate-table layout is worth the surgery. Until then, redact map is the right tradeoff.

**Schema-side / metadata layer:**
- [x] `db_schema/admin_column_redactions.{h,cpp}` — table with `(id BIGSERIAL, table_name, column_name)`. Intentionally NOT FK'd to `admin_top_level_tables` so we can redact tables that are deliberately invisible to the admin CRUD list (sessions, device_tokens, email_verifications). Constants exposed via `kAdminColumnRedactionsTable / Id / TableName / ColumnName`.
- [x] `sql_util/table_helpers/admin_column_redactions.{h,cpp,_test.cpp}` — `Add/Delete/IsRedacted/GetAdminColumnRedactions/LoadColumnRedactionSet`. Defines the canonical `using ColumnRedactionSet = std::set<std::pair<std::string, std::string>>` next to the helper. Tests cover `AddAndListBasic`, `IsRedactedReturnsTrueOnlyForExactPair`, `DeleteRemovesEntry`, `LoadColumnRedactionSetEmptyByDefault`, and `RedactionMapContainsExpectedDefaults`.
- [x] Registered in `make_database_info.cpp::MakeDatabaseInfo` and `create_database.cpp::CreateTables`. Not added to admin top-level / nested lists — the table is read-only at runtime.

**Population:**
- [x] `database_helper/create_database.cpp::PopulateAdminColumnRedactions` seeds the four canonical entries via `TableHelpers::AdminColumnRedactions::AddAdminColumnRedaction`:
  - `(people, password_hash)`
  - `(device_tokens, secret_hash)`
  - `(email_verifications, token_hash)`
  - `(sessions, uuid)`
- [x] Wired into `PopulateTables` next to `PopulateAdminColumnDataInfo`.
- [ ] `(login_attempts, *)` deferred to Phase 5 (login_attempts table doesn't exist yet).

**Enforcement at the JSON boundary:**
- [x] `WebApp` constructor takes `TableHelpers::ColumnRedactionSet`; `WebApp::GetColumnRedactions()` returns it. `main.cpp` loads the set via `AdminColumnRedactions::LoadColumnRedactionSet` once at startup and passes it to the `WebApp` ctor; `EndpointTestHelper` does the same in its initialiser list (so tests that insert redaction rows BEFORE constructing the helper get those redactions applied to their `WebApp`).
- [x] `EndpointAuthHelper::GetColumnRedactions()` delegates to the WebApp accessor.
- [x] `DatabaseRESTHelper` constructor now takes an optional `ColumnRedactionSet` (default `{}` — empty, all columns visible). Private `ApplyRedactions(tableName, KeyValueTable&)` and `ApplyRedactions(tableName, KeyValueTableArray&)` strip any column whose `(tableName, columnName)` is in the set. Applied inside `GetTableValues`, `GetRowsByColumn`, `GetRowsByColumnWithCount`, `GetRowsByValuesWithCount`, `GetRow`, `GetRowByValues`, `GetTableValue`.
- [x] All five generic-CRUD read endpoints updated to construct the helper with the redaction set: `get_row.cpp`, `get_table_rows.cpp`, `get_rows_by_column.cpp`, `get_row_by_values.cpp`, `get_filtered_table_rows.cpp`. The other DatabaseRESTHelper consumers (`add_item`, `add_item_fetch_primary_key`, `update_item`, `delete_item`, `db_schema`, `get_fk_options`, `resolve_fk_display`) are write-paths or metadata/PK-only paths where row data is not echoed back, so no redaction is needed.
- [x] **Note on the design doc's prescription**: the doc said to overload `JsonFromDataResults(tableName, results, redactSet)`. I implemented the equivalent at the `DatabaseRESTHelper` layer instead — every callsite that has a `tableName` is already inside that class, and filtering before constructing `DataResults` keeps `util/json_util.cpp` agnostic of the redaction concept. Same security outcome, smaller blast radius.

**Tests:**
- [x] `admin_column_redactions_test.cpp::AddAndListBasic`, `IsRedactedReturnsTrueOnlyForExactPair`, `DeleteRemovesEntry`, `LoadColumnRedactionSetEmptyByDefault`, `RedactionMapContainsExpectedDefaults` (asserts the four canonical entries on the loaded set).
- [x] `database_rest_helper_test.cpp::JsonFromDataResultsRedactsConfiguredColumns` (column header AND every cell scanned for the secret values), `GetRowRedactsConfiguredColumns`, `GetRowByValuesRedactsConfiguredColumns`, `RedactionsAreTableScoped`.
- [x] Negative: `JsonFromDataResultsKeepsNonRedactedColumns` proves the filter is a no-op when the set is empty.
- [x] **End-to-end `handle_full` redaction tests for every modified endpoint** (every endpoint that constructs `DatabaseRESTHelper` with the redaction set is covered):
  - `endpoints/get_row_test.cpp::GetRowPeopleHidesPasswordHash` — admin session, full round-trip. Asserts: 200, no `password_hash` column, admin's email present, no cell value beginning with `$argon2`.
  - `endpoints/get_table_rows_test.cpp::GetTableRowsPeopleHidesPasswordHash` — same shape for the list endpoint.
  - `endpoints/get_rows_by_column_test.cpp::GetRowsByColumnPeopleHidesPasswordHash` — paginated/sorted variant.
  - `endpoints/get_row_by_values_test.cpp::FetchPeopleHidesPasswordHash` — POST-with-filter-pairs variant.
  - `endpoints/get_filtered_table_rows_test.cpp::FilteredPeopleHidesPasswordHash` — covers BOTH the no-filter and with-filter branches inside one test, since `GetRowsByValuesWithCount` takes both paths.
  - Each test scans every cell for the secret hash value as a belt-and-suspenders check (defends against a future bug that strips the column header but leaves the data).

### 3.9 Lock down ownership-sensitive endpoints with regression tests
- [x] (Verified on follow-up) `endpoints/change_purchase_recipient.cpp:99-106` already rejects when `payerPersonId != loggedInPersonId`
- [x] (Verified on follow-up) `business_logic/payment/gift_permission_helper.cpp::AcceptRequest/DenyRequest` reject when the active user isn't the grantee, and `Revoke` requires either grantor or grantee
- [x] `change_purchase_recipient_test.cpp::ChangeRecipientNotPayer` — already existed pre-Phase-3, exact regression we wanted (logged-in stranger tries to change recipient → 403).
- [x] `gift_permissions_test.cpp::AcceptWrongPerson` already existed (pinned the `AcceptNotGrantee` case). Added `DenyNotGranteeForbidden` and `RevokeOutsiderForbidden` to round out the trio.
- [x] `set_user_info_test.cpp::SetUserInfoIgnoresPersonIdInBody` — the most important regression: two users A and B; logged in as A, POST a body that explicitly sets `person_id` and `id` to B's id along with new field values. Assert A's row was modified (the session user, never the body's id) and B's row is untouched. `update_user_password.cpp` already operates on `session.GetPersonId()` only — verified by inspection; no body field maps to person_id.

## Phase 4 — CSRF (cross-cutting; lower layers + endpoints + client)

### 4.1 Generate and set the `csrft` cookie alongside the session cookie
- [x] `Session::SetCsrfTokenCookie` generates a 32-byte random value via `AuthHelper::RandomBytes(32) + Base64Encode`, stores it in the new `csrfToken_` member, and sets the `csrft` cookie. Called from the existing `InitializeFromLogin` (and therefore also from `InitializeFromDeviceToken`, which delegates to `InitializeFromLogin`).
- [x] Cookie attributes: `Path=/`, `SameSite=Lax`, `Max-Age` matched to the session-cookie's Max-Age, `Secure` + `Domain` in prod, **`HttpOnly=false`** so the SPA can read it.
- [x] `Session::GetCsrfToken()` accessor exposes the value to test code; production code never needs it server-side because the SPA reads the cookie directly.
- [x] Tests in `session_test.cpp`: `InitializeFromLoginSetsCsrfTokenCookie` (attributes + Max-Age + value-matches-getter), `InitializeFromLoginCsrfTokenIsFreshOnEveryCall` (no token reuse across logins), `InitializeFromLoginProdModeCsrfTokenCookieHasSecureAndDomain`, `InitializeFromDeviceTokenSetsCsrfTokenCookie`.

### 4.2 Server-side CSRF middleware
- [x] `endpoints/csrf_guard.{h,cpp}` — `CsrfGuard` is a Crow middleware modeled on `CloudFrontOriginGuard`. `before_handle` short-circuits on GET/HEAD/OPTIONS and on the bootstrap allow-list (`/api/login`, `/api/register`, `/api/remember`, `/api/verify`, including with query strings). Otherwise it parses the `Cookie` header, pulls `csrft`, compares against the `X-CSRF-Token` header via `Auth::AuthHelper::ConstantTimeEqual`, and rejects with `403 application/json {"error":"csrf_token_missing_or_invalid"}` on miss. Rejection logging throttled to one line per minute (mirrors `CloudFrontOriginGuard`).
- [x] Wired into `web_app.h::AppType` as the LAST middleware: `crow::App<CloudFrontOriginGuard, crow::CookieParser, crow::CORSHandler, CsrfGuard>`. Order matters — running after `CookieParser` is defensive even though we parse the raw header ourselves.
- [x] **Test integration**: `EndpointTestHelper` calls `webApp_.GetApp().get_middleware<CsrfGuard>().SetEnabled(false)` after construction so the 80+ existing endpoint tests that set `session_token` cookies don't all need to also synthesize matching CSRF tokens. The dedicated `csrf_guard_test.cpp` re-enables it explicitly.
- [x] Tests in `csrf_guard_test.cpp`: enable/disable toggling, GET/HEAD/OPTIONS bypass, all four bootstrap paths bypass (including with query string), `BootstrapPrefixIsExactMatchOnly` (prevents `/api/login_evil` slipping through), missing-cookie/missing-header/mismatch all rejected with 403 + the right JSON body, matching pair allowed, PUT/PATCH/DELETE all enforced, multi-pair Cookie header parsing, `EmptyCsrftCookieValueRejectedEvenWithEmptyHeader` (locks the empty-string-isn't-a-match invariant).
- [ ] **Origin / Referer enforcement deferred** — the design doc called for additional `Origin` / `Referer` checks (especially on the bootstrap endpoints where the CSRF token doesn't yet exist). Deferred to a follow-up because (a) the production CORS allow-list isn't centrally configured yet and (b) the SameSite=Lax cookie attributes already block the cross-origin POST attack vector for token-bearing requests. Token-based protection is the load-bearing defense; Origin would be defense-in-depth.

### 4.3 Frontend CSRF interceptor
- [x] `ui/src/app/shared/interceptors/csrf.interceptor.ts` — class-based interceptor (matches the existing `ErrorInterceptor` pattern). On POST/PUT/PATCH/DELETE, parses `document.cookie` for `csrft` and clones the request with `X-CSRF-Token: <value>`. No-op when the method is safe or when the cookie isn't set (so unauthenticated bootstrap requests still go through).
- [x] Registered in `ui/src/app/app.config.ts` BEFORE `ErrorInterceptor` (interceptors run in registration order on the request path).
- [x] Tests in `csrf.interceptor.spec.ts`: header-attached on POST/PUT/PATCH/DELETE, no header on GET/HEAD/OPTIONS, no header when cookie absent, correct lookup with multiple cookies, partial-name cookies (`csrft_other`) NOT treated as a match.

### 4.4 Logout must also clear `csrft`
- [x] `endpoints/logout.cpp` adds a third `cookieManager->SetCookie("csrft", "", csrfCp)` after the `session_token` and `device_token` clears. The clear properties mirror the original cookie's `httpOnly=false` so the browser parses the same Set-Cookie shape and replaces the live cookie cleanly.
- [x] Test in `logout_test.cpp::LogoutClearsCsrfCookie` — seeds a live csrft, hits `/api/logout`, asserts the cookie value is empty AND the clear-time properties match the set-time attributes (`HttpOnly=false`, `Path=/`, `Max-Age=0`).

## Phase 5 — Rate limiting & brute-force defense (PostgreSQL with write-behind via ThreadPool — Decision #11)

**Architecture rationale:** in-memory rate limiting on each ECS task means a multi-instance attacker spreads load across N tasks for N× allowance. Worse, ECS auto-scales — the "limit" floats with the fleet. Persistent state in PostgreSQL is the only multi-instance-safe option. To keep login latency tight, we **write attempt records asynchronously** via the existing `ThreadPool` (the same one used by `SessionUsed` for last-seen updates). Reads are synchronous (one indexed `SELECT count(*)`).

**Key invariant:** the *recording* of a failure is async; the *decision to lock* is sync (in the same transaction as the password check). This way two parallel attempts at the threshold can over-count by at most one (acceptable), but the lockout, once decided, is committed before the response goes back.

### 5.1 `login_attempts` table
- [x] `db_schema/login_attempts.{h,cpp}` — columns: `id BIGSERIAL PK`, `email_lower CITEXT`, `ip TEXT`, `attempted_at BIGINT` (microseconds since epoch), `success BOOL`, `kind TEXT`. **Stored ip as TEXT, not Postgres `inet`**, so the schema-helper layer stays type-agnostic; we only ever equality-match and count.
- [x] Three indexes via `CreateLoginAttemptsIndexes(transaction)` called from `CreateTables`: `(email_lower, kind, attempted_at DESC)`, `(ip, kind, attempted_at DESC)`, and `(attempted_at)` for the purge job. Without these, the gate's `SELECT count(*)` on every login goes sequential and turns into a DoS in its own right.
- [x] Registered in `make_database_info.cpp::MakeDatabaseInfo` and `create_database.cpp::CreateTables`. NOT added to the admin CRUD list (sensitive PII).
- [x] `sql_util/table_helpers/login_attempts.{h,cpp,_test.cpp}` — full surface: `RecordAttempt` (uses `now_us()` at insert time so write-behind latency doesn't backdate), `RecentFailureCountForEmail/Ip` (success rows excluded; kind-scoped), `PurgeOlderThan`. Defensive: zero/negative window/age returns 0 / no-op rather than throw or wipe everything.
- [x] Tests: 9 cases including `RecordAttemptInsertsRow`, `EmailLowerIsCaseInsensitive` (CITEXT collation), `RecentFailureCountForEmailExcludesSuccesses`, `RecentFailureCountForEmailIsScopedByKind`, `RecentFailureCountForEmailRespectsWindow`, `RecentFailureCountForIpScopedByIp`, `ZeroOrNegativeWindowReturnsZero`, `PurgeOlderThanBasic`, `PurgeOlderThanZeroIsNoop`.

### 5.2 Synchronous gate before password verify (per-email + per-IP)
- [x] Added all 11 new secrets in `secret_keys.h` + `secret_values.cpp`: `kAuthLoginMaxFailuresPerEmailPerWindow=10`, `kAuthLoginMaxFailuresPerIpPerWindow=50`, `kAuthLoginFailureWindowInMicros=15min`, `kAuthAccountLockoutAfterFailures=10`, `kAuthAccountLockoutDurationInMicros=30min`, `kAuthVerifyMaxFailuresPerIpPerWindow=30`, `kAuthVerifyFailureWindowInMicros=15min`, `kAuthRememberMaxFailuresPerIpPerWindow=50`, `kAuthRememberFailureWindowInMicros=15min`, `kAuthLoginAttemptsRetentionInMicros=30days`.
- [x] `business_logic/auth/login_gate.{h,cpp,_test.cpp}` — three entry points instead of one: `CheckBeforeLogin(tx, secrets, email, ip)`, `CheckBeforeVerify(tx, secrets, ip)`, `CheckBeforeRemember(tx, secrets, ip)`. Login consults `people.locked_until`, per-email, and per-IP; verify and remember are per-IP only (no authenticated email at those stages). Email check takes precedence over IP when both would trip — operator gets the more specific signal in logs. Empty IP skips the per-IP check (so a misconfigured deploy without proxy plumbing doesn't false-positive). Missing secrets fall back to defensive defaults.
- [x] **The gate runs INSIDE `login.cpp` / `verify.cpp` / `remember.cpp` BEFORE the auth check**, not inside `PersonHelper::VerifyPassword`. Rationale: VerifyPassword stays usable from places that don't want side effects (e.g. tests asserting on hash semantics). The endpoint layer is the right place to attach gate + IP plumbing + async recorder, since that's where the request lives.
- [x] 16 tests in `login_gate_test.cpp`: each branch (Allow / AccountLocked / RateLimitedEmail / RateLimitedIp), precedence (email before ip; account-locked before failure counts), expired-lockout-is-ignored, unknown-email-falls-through-to-IP-check, missing-secrets-fall-back, empty-IP skips, plus dedicated coverage of `CheckBeforeVerify` and `CheckBeforeRemember` including kind scoping.

### 5.3 Synchronous lockout decision (sticky on the row)
- [x] New high-level wrapper `PersonHelper::LoginPerson(tx, secrets, email, password) -> LoginAttemptResult` (struct: `success`, `justLocked`, `personId`, `failedLoginAttempts`, `lockedUntil`). Single atomic UPDATE in both branches:
  - **Success path**: `UPDATE people SET failed_login_attempts = 0, locked_until = NULL WHERE id = $1 RETURNING failed_login_attempts, COALESCE(locked_until, 0)`
  - **Failure path**: `UPDATE people SET failed_login_attempts = failed_login_attempts + 1, locked_until = CASE WHEN failed_login_attempts + 1 >= $threshold THEN now_us() + $duration ELSE locked_until END WHERE id = $1 RETURNING ...` — one statement so two parallel verifies at the threshold over-count by at most one (acceptable per the design doc's invariant).
  - `VerifyPassword` is left untouched so existing tests/callers still work.
- [x] `justLocked = (newAttempts == threshold) AND (newLockedUntil > 0)` — fires exactly on the crossing, not on subsequent failures past the threshold. Hooks into the admin_alerts pipe in Phase 9 (deferred — admin_alerts table exists, but the alert insert call is not yet wired).
- [x] Tests in `person_test.cpp`: `LoginPersonSuccessClearsFailureCount` (resets pre-seeded failures), `LoginPersonWrongPasswordIncrementsCount` (under threshold), `LoginAccountLockoutAtThreshold` (3 wrong = locked, justLocked=true, lockedUntil>0), `LoginSuccessClearsLockout` (post-expired lock), `LoginAccountStaysLockedDuringWindow` (documents that LoginPerson is intentionally lock-agnostic — gate's job to refuse), `LoginAccountUnlocksAfterWindow` (backdated lock), `LoginPersonUnknownEmailReturnsFailureWithNoSideEffect` (no row created, no row touched).

### 5.4 Asynchronous attempt recording (write-behind via ThreadPool)
- [x] `login.cpp::EnqueueAttemptRecord` — captures `TransactionProvider`, `DatabaseHelper`, `email`, `ip`, `kind`, `success` by value into a `ThreadPool::GetInstance().Queue(...)` lambda. The lambda opens a fresh transaction inside `provider->RunInTransaction` and inserts via `LoginAttempts::RecordAttempt`. Wrapped in try/catch so DB blips don't crash the worker.
- [x] Same pattern in `verify.cpp::EnqueueVerifyAttempt` and `remember.cpp::EnqueueRememberAttempt` (5.6 below).
- [x] Gate-refused requests (429) **don't** enqueue a row — the existing rows are why we got the 429; another would re-extend the lockout window. Sub-threshold failures DO enqueue.
- [x] Tests use `ThreadPool::GetInstance().Shutdown()` to flush queued work before asserting (mirrors `SessionUsedBasic`). The test transaction provider routes the async write back into the test's outer transaction so it's both visible AND rolled back at end-of-test (no cross-test pollution).
- [x] Multi-instance correctness preserved: nothing in-memory; every gate read goes to Postgres.

### 5.5 IP plumbing
- [x] `Auth::ResolveClientIp(req)` added to `proxy_trust.{h,cpp}` — when the proxy is trusted AND `X-Forwarded-For` is set, returns the first entry (the actual viewer); otherwise falls back to `req.remote_ip_address` (Crow's connection peer). Returns empty when neither source has a value; the gate / recorder treat empty as "skip per-IP path" rather than as a sentinel.
- [x] Wired into all three endpoints: `login.cpp`, `verify.cpp`, `remember.cpp`.
- [x] 4 new tests in `proxy_trust_test.cpp::ResolveClientIpTest`: `PrefersForwardedHeaderWhenProxyTrusted`, `FallsBackToConnectionPeerWhenProxyUntrusted`, `FallsBackToConnectionPeerWhenForwardedHeaderMissing`, `ReturnsEmptyWhenNeitherSourceAvailable`.

### 5.6 Verification & remember-me throttling
- [x] `verify.cpp` and `remember.cpp` call `LoginGate::CheckBeforeVerify` / `CheckBeforeRemember` before their existing auth checks. Refused requests get the same generic `TooManyAttempts` 429 as login (Phase 5.7). Both endpoints async-record outcomes via per-endpoint enqueuers (`EnqueueVerifyAttempt`, `EnqueueRememberAttempt`) keyed by `kind=verify` / `kind=remember`. Remember passes empty `email_lower` since there's no authenticated email at that point.
- [x] Secrets added (5.2 above): defaults `30/15min` for verify, `50/15min` for remember.
- [x] Tests: `verify_test.cpp::VerifyRateLimitedPerIp` (3-row pre-seed → 429), `VerifyRecordsFailureAttemptAsync` (sub-threshold failure records); `remember_test.cpp::RememberRateLimitedPerIp`, `RememberRecordsFailureAttemptAsync`. Each test sets `req.remote_ip_address` directly so `ResolveClientIp`'s connection-peer fallback returns a non-empty IP for the synthesized request.

### 5.7 Generic error shape (no enumeration)
- [x] New `ErrorResponse::TooManyAttempts` (HTTP 429, type `too_many_attempts`). Returned for ALL gate refusals: per-email burst, per-IP burst, AND sticky account-lockout. Caller can't tell the buckets apart; operator gets the breakdown in server logs.
- [x] Wrong-password and unknown-email both return identical 401 `invalid_credentials` with the same `detail` text. `LoginTest::LoginWrongPasswordSameShapeAsUnknownEmail` pins the response shape — pulls both response bodies and asserts type/detail equality.
- [ ] **Wall-clock timing parity (deferred)**: padding the unknown-email branch with a no-op Argon2 verify against a sentinel hash. The design doc marks this "optional"; the response-shape test above covers the more impactful side-channel. Defer to a follow-up.

### 5.8 Periodic cleanup
- [x] `POST /api/admin/cleanup_login_attempts` (`endpoints/admin_cleanup_login_attempts.{h,cpp}`). Auth: login + `manage_subscriptions`. Reads `kAuthLoginAttemptsRetentionInMicros` from secrets with a defensive 30-day fallback when missing/unparseable. Calls `LoginAttempts::PurgeOlderThan` and returns `{"login_attempts_deleted": N}`. 5 endpoint tests (401, 403, nothing-to-clean, deletes-aged-rows-leaves-fresh, missing-secret-falls-back).
- [x] Registered as a daily scheduled job in `Scheduler::BuildStandardJobs`. New `loginAttemptsCleanupSeconds = 86400` field on `JobIntervals`, new `--login_attempts_cleanup_interval` flag in `scheduler/main.cpp`, included in `LogConfigSummary`. `BuildStandardJobsTest` count bumped from 10 to 11; new `LoginAttemptsCleanupCanBeDisabled` test pins the disable-with-zero-interval contract. Phase 12 of `Scheduled Jobs.md`.
- [x] Manual trigger via `knottyyoga_test_helper`: new `cleanup_login_attempts` command (alias `cla`) under the Configuration category. Calls the table helper directly (no HTTP roundtrip, no service-account login). Useful after load tests / attack drills where the table needs purging without waiting for the scheduler tick. Documented as scenario 25 in `Manual Testing Helper Executable.md`.
- [x] `login_attempts_test.cpp::PurgeOlderThanBasic` and `::PurgeOlderThanZeroIsNoop` already pin the helper-side behavior; the new endpoint test is the thin wrapper around them.

## Phase 6 — Cookie hygiene cleanup

### 6.1 Logout clears cookies with attributes that match the originals
- [x] New `business_logic/auth/auth_cookies.{h,cpp,_test.cpp}` with two functions: `BuildAuthCookieProperties(transaction, secrets, isProd, httpOnly, maxAgeMicros)` builds the canonical `CookieProperties` for any auth cookie (Path=/, SameSite=Lax, Secure+Domain in prod, configurable HttpOnly), and `ClearAuthCookies(transaction, secrets, cookieManager)` invokes it three times with `Max-Age=0` to clear `session_token` (HttpOnly), `device_token` (HttpOnly), and `csrft` (NOT HttpOnly).
- [x] `session.cpp::SetCookieHelper` and `Session::SetCsrfTokenCookie` refactored to use `BuildAuthCookieProperties` so set-time and clear-time can never drift again (single source of truth for the attribute matrix).
- [x] `endpoints/logout.cpp` refactored — old inline `CookieProperties` block (which used SameSite=None at clear-time, mismatching session.cpp's Lax at set-time and causing browsers to refuse the deletion) replaced by a single `Auth::ClearAuthCookies(...)` call.
- [x] 9 tests in `auth_cookies_test.cpp`: `BuildAuthCookiePropertiesTestModeHttpOnly`, `BuildAuthCookiePropertiesTestModeNotHttpOnly`, `BuildAuthCookiePropertiesProdModeHasSecureAndDomain`, `ClearAuthCookiesEmitsAllThreeCookies`, `ClearAuthCookiesUsesMaxAgeZero`, `ClearAuthCookiesUsesSameSiteLax` (regression for the actual bug), `ClearAuthCookiesPreservesHttpOnlyDistinction`, `ClearAuthCookiesProdModeHasSecureAndDomain`, `ClearAuthCookiesNullCookieManagerIsNoop`, plus `SetAndClearAttributesMatchExceptForMaxAge` which pins the load-bearing invariant directly.
- [x] `logout_test.cpp::LogoutClearedCookiesUseSameSiteLax` — endpoint-level regression that seeds a SameSite=None cookie (mimicking what a browser had pre-fix) and asserts the logout overwrites it with Lax for all three cookies, with HttpOnly preserved per cookie type.

### 6.2 Defensive: CookieManager warns on `SameSite=None` without `Secure`
- [x] `Auth::Detail::CheckCookieAttributes(key, props)` in `cookie_manager.h/cpp` returns a non-empty diagnostic string when SameSite=None && !secure, empty otherwise. Production `CookieManagerImpl::SetCookie` calls it on every cookie write and emits `LogError() << "[cookie_manager] event=invalid_attributes ..."` when it returns a warning. Advisory only — the cookie is still set so behavior is unchanged for callers, but the operator gets a greppable line in the journal.
- [x] 5 tests in `cookie_manager_test.cpp`: `SameSiteNoneWithoutSecureIsFlagged` (warning includes cookie name + "SameSite=None" + "Secure"), `SameSiteNoneWithSecureIsAccepted` (legitimate cross-origin case), `SameSiteLaxAcceptsAnySecureValue` (project default works on plain HTTP), `SameSiteStrictAcceptsAnySecureValue`, `OtherAttributesDoNotInfluenceTheCheck` (HttpOnly/Path/Domain/Max-Age all irrelevant — only SameSite+Secure matter).

## Phase 7 — Security headers, error handling, server banner

### 7.1 SecurityHeaders middleware
- [x] `endpoints/security_headers.{h,cpp}` — Crow middleware modeled on `CsrfGuard`. Emits in `after_handle` so headers land on every response (handler-set, error-set, anything else). Always-on headers: `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: camera=(), microphone=(), geolocation=()`, `Server: Knotty Yoga` (banner override). Prod-only: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` and the canonical `Content-Security-Policy` from the design doc.
- [x] Defensive: when `Auth::ServerConfig::GetInstance()` throws (early-startup paths, some test fixtures), the middleware falls back to "not prod" rather than crashing — the always-on headers still emit; the prod-only ones are skipped.
- [x] Wired into `web_app.h::AppType` as the LAST middleware. Crow runs `after_handle` in REVERSE declaration order, so `SecurityHeaders.after_handle` fires FIRST and the headers land before any other middleware mutates the response.
- [x] **Test integration**: `EndpointTestHelper` calls `webApp_.GetApp().get_middleware<SecurityHeaders>().SetEnabled(false)` after construction — same trick used for `CsrfGuard` (Phase 4.2) — so the 100+ pre-existing endpoint tests don't have to assert on or strip the new headers. The dedicated `security_headers_test.cpp` re-enables it explicitly.
- [x] 9 tests in `security_headers_test.cpp`: `EnabledByDefault`, `SetEnabledFlipsState`, `DisabledMiddlewareIsANoOp`, `TestModeEmitsAlwaysOnHeaders` (exact strings for all four always-on headers), `EmitsVaryOrigin` (Phase 7.2 — see below), `TestModeOmitsHstsAndCsp`, `NoServerConfigOmitsHstsAndCsp` (defensive), `ProdModeEmitsHstsAndCsp` (full integration through `EndpointTestHelper`+`ServerConfig::Initialize`; spot-checks load-bearing CSP directives `default-src 'self'`, `frame-ancestors 'none'`, `base-uri 'none'`), `IntegrationHeadersLandOnRealResponse` (drives `/api/health` through `handle_full` with the middleware re-enabled), `EndpointTestHelperLeavesMiddlewareDisabled` (regression — the default-off behavior the test fixture promises).

### 7.2 CORS hardening (`Vary: Origin` + opt-in dev CORS)
- [x] `Vary: Origin` is added by `SecurityHeaders.after_handle` (Crow's `CORSHandler` does NOT add it itself). Always-on so shared caches don't serve a CORS response keyed for one origin to a request from a different origin. Pinned by `SecurityHeadersTest::EmitsVaryOrigin`.
- [x] Dev-mode CORS opt-in: `KNOTTYYOGA_DEV_CORS_ORIGIN` env var. When set (e.g. `http://localhost:4200`), `server_config.cpp::Initialize` registers that origin with `allow_credentials()`, methods `GET/POST/OPTIONS`, headers `Content-Type, Authorization, X-CSRF-Token`. Unset → dev mode stays no-CORS (Angular proxy keeps everything same-origin — long-standing default).
- [x] Production CORS allow-list also extended to include `X-CSRF-Token` so the SPA's CSRF interceptor (Phase 4.3) can attach the header on cross-origin POSTs without a preflight rejection.
- [x] Documented `KNOTTYYOGA_DEV_CORS_ORIGIN` (and the rest of the process-level env vars) in a new `## Environment Variables` section in `CLAUDE.md`. **`ui/proxy.conf.json` comments deferred** — JSON doesn't support comments and converting to `proxy.conf.js` is out of scope for this PR; the env var is documented in the only file that supports it.
- [x] 4 new tests in `server_config_test.cpp::ServerConfigTest`: `DevCorsOriginUnsetLeavesCorsAtDefault` (default `Access-Control-Allow-Origin: *`, no credentials), `DevCorsOriginSetRegistersThatOrigin` (specific origin echoed; `Access-Control-Allow-Credentials: true`), `DevCorsOriginEmptyValueIsTreatedAsUnset` (empty env var = unset), `DevCorsOriginIgnoredInProdMode` (prod always pins to `kWebsiteAddress`, never the dev env var even if set to a hostile value). `DevCorsOriginEnvScope` RAII guard scrubs the env var on entry/exit so leaked state can't bleed across tests.

### 7.3 Global Crow error handler
- [x] `ErrorResponse::InternalError(serverContext)` refactored: response body is ALWAYS the generic `"An unexpected error occurred. Please try again later"`, never echoes the argument. The argument is logged server-side via `[error_response] event=internal_error_context context="..."` so operators can correlate. Pre-existing call sites (`catch (std::exception& e) { resp = ErrorResponse::InternalError(e.what()); }`) stop leaking exception text without code change.
- [x] Audited the codebase for raw `crow::response(NNN, e.what())` patterns. Found exactly 2: `endpoints/db_schema.cpp:31` and `endpoints/add_item_fetch_primary_key.cpp:25`. Both replaced with `ErrorResponse::InternalError(e.what())`. The `add_item_fetch_primary_key.cpp` site also fixed an unrelated `crow::response(400, "No message body.")` to use `ErrorResponse::BadRequest` for consistency.
- [x] **No global Crow exception handler added** — Crow doesn't expose an `app.exception_handler(...)` API. The existing per-endpoint try/catch pattern catches everything; the gap was the *content* of the response body, not the absence of catching. With `InternalError` always-generic, the throw-inside-handler scenario the design doc calls out is fully covered: every handler that wraps in try/catch + `InternalError` now produces the generic shape automatically.
- [x] 3 new tests in `error_response_test.cpp`: `InternalErrorDoesNotEchoServerContext` (passes a sensitive SQL error string, asserts NEITHER the structured `detail` nor the raw response body contains any of it; spot-checks `password_hash`, `ERROR:`, `LINE 1`), `InternalErrorEmptyServerContextStillGeneric`, `InternalErrorDefaultArgIsGeneric` (default-arg overload matches explicit empty-string).

## Phase 8 — Secrets at rest (env-var-from-Secrets-Manager v1; KMS later — Decision #6)

**Architecture (v1):** the master encryption key lives in an env var `KNOTTYYOGA_SECRET_KEY`. In production, ECS task definitions inject this value from AWS Secrets Manager via the `secrets:` block on the container — the container sees a normal env var; AWS handles secret storage, IAM, and rotation lifecycle. **No AWS SDK in the C++ build.** Just `std::getenv` + base64-decode at startup.

**Architecture (v2, deferred):** AWS KMS + envelope encryption. Generate a Customer Master Key. On startup, call `GenerateDataKey` to get a plaintext data key + an encrypted version. Encrypt secrets with the data key; store the encrypted data key alongside. On read, call `Decrypt` on the encrypted data key. Pros: KMS rotates the master without code change, IAM controls who can decrypt, every decrypt audited in CloudTrail. Cons: AWS SDK dependency, a network call in startup hot path, IAM plumbing, mock KMS client for tests. Defer until audit pressure or compliance requires it.

**Why v1 first:** zero external dependencies, ships in days not weeks, gets us off "plaintext credentials in PG" today. The day we want v2, the encryption boundary is already in place — we just swap the key source.

### 8.1 Encrypt `config_secrets.value`
- [ ] Add a singleton `business_logic/auth/secrets_at_rest.{h,cpp}` that reads `std::getenv("KNOTTYYOGA_SECRET_KEY")` once at startup, base64-decodes to 32 bytes, and exposes `Encrypt(plaintext)` / `Decrypt(ciphertext)` using `crypto_secretbox_easy` (libsodium). Store as `base64(nonce || ciphertext)` so the value column stays a `TEXT`.
- [ ] In test mode, `SecretsAtRest` accepts an injected key (use a fixed test key in `endpoint_test_helper.cpp`).
- [ ] Wire `SecretsAtRest` through `sql_util/table_helpers/config_secrets.cpp::AddSecret` / `LookupSecret`. Existing callers (`SecretsHelper`) continue to see plaintext; encryption is fully internal.
- [ ] **Migration**: one-shot startup pass in `database_helper/create_database.cpp` (or a new `migrate_secrets.{h,cpp}` called from `main.cpp`) that reads every row, detects format (any value not starting with the `v1:` prefix is treated as legacy plaintext), encrypts it, and rewrites. Idempotent: running twice is a no-op.
- [ ] Tests:
  - `secrets_at_rest_test.cpp::EncryptDecryptRoundTrip`, `DecryptOfTamperedCiphertextThrows`, `DecryptWithWrongKeyThrows`
  - `config_secrets_test.cpp::AddSecretEncryptsValue` (assert the raw row is not the plaintext) and `LookupSecretReturnsPlaintext`
  - `secrets_helper_test.cpp` — existing tests should still pass unchanged (callers see plaintext)

### 8.2 Startup validation (Decision #6 v1 + Phase 12.1)
- [ ] `main.cpp` after `ServerConfig::Initialize`: when `IsProdMode()` is true, fail-fast if `KNOTTYYOGA_SECRET_KEY` is unset/empty/wrong-length. Log a clear message ("ECS task definition must inject KNOTTYYOGA_SECRET_KEY from Secrets Manager") and exit non-zero.
- [ ] In dev/test mode, fall back to a fixed dev key if the env var is unset (with a `[DEV]` warning log) so local builds don't require the env var.
- [ ] Document the AWS Secrets Manager → ECS task definition wiring in a new ops note in the Obsidian vault: create entry `Claude/Deploying — secrets and env vars.md` with the JSON snippet for the task definition's `secrets:` block.

### 8.3 Locked-down admin secrets endpoint (Decision #9)
- [ ] (Replaces the deletion in 1.8) `endpoints/admin_secrets.{h,cpp}` — `GET /api/admin/secrets` returns `[{name}, ...]` (names only, no values, never the key). `PUT /api/admin/secrets/{name}` body `{value}` writes a value via `SecretsHelper`, which encrypts via `SecretsAtRest` (Phase 8.1). Both require the `admin` permission.
- [ ] Audit every write: insert an `admin_alerts` row with `kind=secret_changed`, `description={name} updated by person_id={...}`. Don't include the value — names only.
- [ ] **Never** return secret values to the client, even to admins. The right pattern is "set new value, blind-write" — if the admin needs to verify it, they do it via the feature that uses the secret (e.g., send a test email).
- [ ] Frontend: replace the generic-CRUD secrets editor with a new component under `ui/src/app/admin/secrets/`. Lists names, has a "Set Value" form per row that masks the input and POSTs to the new endpoint.
- [ ] Tests:
  - `admin_secrets_test.cpp::GetReturnsNamesOnly`, `PutRequiresAdmin`, `PutEmitsAdminAlert`, `PutEncryptsAtRest`
  - Frontend: `secrets.component.spec.ts` — masked input, post-on-save, refresh-list-on-success

## Phase 9 — Observability for security events

### 9.1 Auth event log
- [ ] Add a `auth_events` table or extend `admin_alerts` with a `kind` column. Persist: `login_success`, `login_failure`, `password_changed`, `email_verified`, `device_token_used`, `device_token_revoked`, `account_locked`, `role_assigned`, `role_revoked` with `(person_id, ip, user_agent, when_us, detail_json)`.
- [ ] Emit from the relevant business-logic helpers (`PersonHelper::VerifyPassword`, `Login`, `UpdatePassword`, `LogoutPerson`, `TryLoginWithDeviceToken`, role-assignment helpers).
- [ ] Tests: per-helper assertion that an event row is added on the relevant code path

### 9.2 admin_alerts notification
- [ ] One simple delivery mechanism: a polling `ThreadPool` job started in `main.cpp` that, every N minutes, looks at the new admin_alerts since last run and, if any are above a severity, mails them to the address in a new secret `kAdminAlertsRecipient`.
- [ ] Test: insert a row, run one tick of the loop, assert the test mailer captured a message

## Phase 10 — Frontend follow-ups

### 10.1 APP_INITIALIZER bootstrap
- [ ] Add `APP_INITIALIZER` to `ui/src/app/app.config.ts` that calls `AuthService.tryTokenLogin()` (which already exists) and resolves regardless of result
- [ ] Update `auth.service.ts::tryTokenLogin` to: call `/api/me` → on 401, call `/api/remember` → on success, call `/api/me` again → populate `authData$`
- [ ] Tests in `auth.service.spec.ts` covering the four branches (200/200, 401/200/200, 401/401, 200/_)

### 10.2 returnUrl allow-listing
- [ ] In `ui/src/app/auth/login/login.component.ts`, validate `returnUrl`: must start with `/`, must not start with `//` or `/\\`, must match one of `['/my', '/admin', '/manage', '/staff', '/calendar', '/shop', '/account']` (or a regex of those prefixes)
- [ ] Default to `/` if invalid; do not silently redirect to a foreign origin
- [ ] Tests: `login.component.spec.ts::returnUrlValidationRejectsAbsoluteUrls`, `RejectsProtocolRelative`, `AllowsKnownPath`

### 10.3 Frontend table-name guards
- [ ] Even though server enforcement is now correct (Phase 3.8), audit `ui/src/app/.../edit-db-table.component.ts` to ensure `selectedDbTable.table_name` is sourced from server-provided lists, not raw user input
- [ ] No new tests if it's already correct; otherwise add a spec

### 10.4 Idle warning (optional polish)
- [ ] Display a "your session will expire soon" snackbar 5 minutes before `kAuthSessionMaxDuractioninMicros` elapses on the client. Optional — defer.

## Phase 11 — Cleanup of latent SQL-concat in DB initializer

### 11.1 Replace concatenation in `create_database.cpp`
- [ ] `database_helper/create_database.cpp:2031,2049,2062` — switch each to `transaction.RunSqlStatementReturningOneValue(... $1 ...)`
- [ ] No new tests required — existing init runs on every test fixture

## Phase 12 — Operational guard rails

### 12.1 Fail loud in prod when guards aren't configured
- [ ] In `main.cpp` after `ServerConfig::Initialize`, when `IsProdMode()` is true:
  - assert `KNOTTYYOGA_ORIGIN_SECRET` is set
  - assert `KNOTTYYOGA_TRUST_PROXY` is set
  - assert `KNOTTYYOGA_SECRET_KEY` is set (Phase 8)
  - assert `kWebsiteAddress` secret is set
  - throw and exit nonzero on any miss
- [ ] Test in `server_config_test.cpp` for the prod-validation path (mock the env via test-mode flag)

### 12.2 Log on first request that the server is in prod mode and that the guards are active
- [ ] Single startup log line summarizing `prod=true|false, csrf_guard=on, origin_secret=set, proxy_trust=set, security_headers=on`. Helps catch misconfiguration in deploy.

# Future Considerations

Items that came up while drafting this plan but were intentionally scoped out. Track here so they're not forgotten:

- **Hard lockout escalation.** After N soft lockouts in 24 h, mark the account hard-locked and require an email-based unlock link. Useful as a tier-2 brute-force defense once Phase 5's soft lockout is in production and we have failure-pattern data. Adds a `lockout_count_24h` field to `people` and a small unlock-via-email flow.
- **HaveIBeenPwned breach-list check.** At registration and password change, hit HIBP's k-anonymity range API with the first 5 chars of the password's SHA-1 and reject if the suffix is in the returned list. Real defense against weak passwords (Argon2id doesn't help with these). Optional for v1.
- **Idle-session client-side warning.** Snackbar 5 min before `kAuthSessionMaxDuractioninMicros` elapses. Polish, not security.
- **Server-side per-IP rate limit on `/api/admin/*` writes.** Less load-bearing than auth endpoints but worth a smaller, per-IP-only throttle once admin functionality grows.