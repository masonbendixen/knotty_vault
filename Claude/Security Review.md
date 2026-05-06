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

## Decisions Locked In

These were resolved in the Open Questions section below. Recording them here so the plan body is self-contained:

| # | Topic | Decision |
|---|---|---|
| 1 | CSRF approach | Double-submit cookie (`csrft` non-HttpOnly + `X-CSRF-Token` header) + Origin allow-list. |
| 2 | Verification email link | Points at SPA route `/verify?email=…&secret=…`; SPA POSTs to `/api/verify` and `history.replaceState`s the URL bar. Default verification window reduced to **1 hour**. |
| 3 | Argon2id strength | `MODERATE` (3 ops, ~256 MB). Tunable via `kAuthArgon2OpsLimit` secret. |
| 4 | Dev CORS | No-op by default (Angular proxy makes everything same-origin); opt-in via `KNOTTYYOGA_DEV_CORS_ORIGIN` env var for direct API testing. |
| 5 | Lockout thresholds | Per-email 10/15-min → 30-min lockout. Per-IP 50/15-min → 1-h block. Verify 30/15-min/IP → 30-min. Remember 50/15-min/IP → 1-h. All secret-driven. Generic 401/429 responses; never reveal lockout reason. |
| 6 | Encryption key | v1: env var `KNOTTYYOGA_SECRET_KEY` injected by ECS task definition from AWS Secrets Manager. v2 (deferred): AWS KMS + envelope encryption. |
| 7 | Mason/Tyler hardcode | Delete outright; admins are minted via seed data + role-management UI. |
| 8 | Sensitive column protection | Column-level redact map populated in `create_database.cpp`, enforced at the JSON boundary in `database_rest_helper.cpp`. Reconsider if `people` is ever exposed to anonymous reads. |
| 9 | Secrets admin UI | Dedicated endpoint + UI; values masked, never returned to client; writes audited to `admin_alerts`. |
| 10 | SameSite | Stay with `Lax` for all auth cookies. `Strict` would break email-driven `/verify` flow. |
| 11 | Rate-limit persistence | PostgreSQL with **write-behind** via existing `ThreadPool`. Synchronous read for the threshold check; synchronous lockout decision; async write of attempt records. Multi-instance ECS safe; login latency unchanged (Argon2id dominates). |
| 12 | `/api/me` GET | Switch from POST to GET (no contract to break — pre-deploy). |
| 13 | Ownership checks | Verified correct on `change_purchase_recipient` and `gift_permissions`; just adding regression tests. |
| 14 | Phase ordering | Pull Phase 7 (security headers) and Phase 12.1 (startup guards) forward as 1b/1c. Pull Phase 6 (cookie hygiene) ahead of Phase 4 (CSRF) so the shared clear-cookies helper exists when CSRF starts using it. |

## Phase 1 — Schema integrity & lookup performance

**Why first**: every later auth fix sits on top of these tables. Adding indices + uniqueness now also avoids painful migrations after data accumulates.

### 1.1 Add unique index on `sessions.uuid` (db_schema layer)
- [ ] Add `AddIndex(kSessionsUuid, /*unique=*/ true)` (or equivalent in the schema DSL) in `db_schema/sessions.cpp::MakeSessionsTable`
- [ ] Verify `db_schema/db_and_table_operations.cpp` emits `CREATE UNIQUE INDEX` correctly (extend if no unique-index helper exists)
- [ ] Add a metadata-level unit test in `db_schema/sessions_test.cpp` (or the equivalent) asserting the index is present in `DatabaseInfo`
- [ ] Add a `db_and_table_operations_test.cpp` regression that DDL output includes `UNIQUE INDEX` when the column is so flagged

### 1.2 Add unique index on `device_tokens.uuid`
- [ ] `db_schema/device_tokens.cpp::MakeDeviceTokensTable` — add unique index on `kDeviceTokensUuid`
- [ ] Schema test in `db_schema/device_tokens_test.cpp` (or `database_metadata_test.cpp`) asserting uniqueness

### 1.3 Add `UNIQUE(person_id)` to `email_verifications`
- [ ] `db_schema/email_verifications.cpp` — add unique constraint on `kEmailVerificationsPersonId`
- [ ] Update `sql_util/table_helpers/email_verifications.cpp::AddEmailVerificationByEmail/Id` to do a `DELETE` of any existing row for the person *in the same transaction* before insert (so resending a verification is still allowed). Add a unit test that issues two consecutive `AddEmailVerificationByEmail` calls and asserts only one row exists for that person.
- [ ] Backfill plan note: existing data is dev-only, schema reset acceptable

### 1.4 Add `UNIQUE(person_id, role_id)` to `role_assignments`
- [ ] `db_schema/role_assignments.cpp` — composite unique
- [ ] Update `sql_util/table_helpers/role_assignments.cpp::AddRoleAssignment` test to assert duplicate insert throws the unique-violation
- [ ] Add an `AddRoleAssignmentDuplicate` test that confirms the constraint fires

### 1.5 Add `UNIQUE(role_id, permission_id)` to `role_permissions`
- [ ] `db_schema/role_permissions.cpp` — composite unique
- [ ] Mirror test additions in `sql_util/table_helpers/role_permissions_test.cpp`

### 1.6 Make `people.email` case-insensitive
- [ ] Switch `db_schema/people.cpp::kPeopleEmail` to `citext` (preferred — requires `CREATE EXTENSION citext` in `database_helper/create_database.cpp`'s pre-table phase)
- [ ] If `citext` requires more plumbing than desired, alternative: keep `text` but add `CHECK (email = lower(email))` and lowercase on every write in `sql_util/table_helpers/people.cpp::AddPerson` / `UpdatePerson`
- [ ] `people_test.cpp` — add `AddPersonEmailCaseInsensitive` test that inserts `Mason@x.com`, then asserts `LookupPersonByEmail("mason@x.com")` returns the same row, and that a second insert with mixed case throws on the unique constraint

### 1.7 Add lockout columns to `people`
- [ ] `db_schema/people.cpp` — add `failed_login_attempts BIGINT NOT NULL DEFAULT 0` and `locked_until BIGINT NULL` (microseconds since epoch, NULL = not locked)
- [ ] Don't expose these via the generic CRUD `admin_top_level_tables` mapping — only the auth path writes them
- [ ] Defer the *use* of these columns to Phase 5; this phase only adds the schema slot

### 1.8 Remove `config_secrets` from generic CRUD exposure
- [ ] `database_helper/create_database.cpp::PopulateAdminTopLevelTables` — remove `config_secrets` (and any other table holding live credentials) from the admin-CRUD allow set
- [ ] If admins currently rely on the generic table editor for secrets, replace with a dedicated `/api/admin/secrets` endpoint in Phase 8 that masks the value and audits writes
- [ ] `endpoints/get_row_test.cpp` — add a regression that `GET /api/get_row/config_secrets/...` returns 403/400

## Phase 2 — Auth primitives hardening (lower layers, no behavior changes user-visible)

### 2.1 Constant-time hash comparison helper
- [ ] Add `bool AuthHelper::ConstantTimeEqual(string_view, string_view)` in `business_logic/auth/auth_helper.h/cpp` backed by `sodium_memcmp` (returns false if sizes differ)
- [ ] Tests in `auth_helper_test.cpp`: `ConstantTimeEqualMatch`, `ConstantTimeEqualLengthMismatch`, `ConstantTimeEqualMismatch`

### 2.2 Use the helper for the email-verification token-hash compare
- [ ] `sql_util/table_helpers/email_verifications.cpp::DoEmailVerification` — replace the `storedTokenHash == tokenHash` check with `AuthHelper::ConstantTimeEqual`
- [ ] Update / add a test confirming a mismatch still increments attempts and returns false

### 2.3 Atomic email-verification attempt accounting
- [ ] Replace the read-then-write attempts logic in `email_verifications.cpp::DoEmailVerification` with a single `UPDATE email_verifications SET attempts = attempts + 1 WHERE id = $1 AND attempts < $2 AND expires_at > now_us() RETURNING attempts, token_hash, person_id` — branch on whether a row was returned
- [ ] On success-path the row is then `DELETE`d in the same transaction
- [ ] Add `email_verifications_test.cpp::DoEmailVerificationConcurrentAttempts` that simulates two interleaved verification attempts on the same row and asserts the total attempt count is exactly 2 (not 3 / not 1) and that `attemptLimit` is honored

### 2.4 Atomic device-token rotation
- [ ] Add a new helper `bool DeviceTokens::ConsumeAndRotate(Transaction&, string_view oldSecretHash, string_view newSecretHash, int64_t newExpiresAtMicros, /*out*/ int& personId)` that runs a single `UPDATE device_tokens SET secret_hash=$2, uuid=gen_random_uuid(), last_used_at=now_us(), expires_at=$3 WHERE secret_hash=$1 AND NOT revoked AND now_us() < expires_at RETURNING person_id, uuid` and returns true only if a row was returned
- [ ] `business_logic/auth/person.cpp::TryLoginWithDeviceToken` — switch from the read+update pattern to `ConsumeAndRotate`
- [ ] Test: `device_tokens_test.cpp::ConsumeAndRotateBasic`, `ConsumeAndRotateRevoked`, `ConsumeAndRotateExpired`, `ConsumeAndRotateUnknown`, plus a contention test that does two parallel rotations against the same secret hash and asserts exactly one wins

### 2.5 Argon2id parameter bump (and make tunable)
- [ ] Add secrets `kAuthArgon2OpsLimit` and `kAuthArgon2MemLimitKb` (defaults: `MODERATE` ≡ 3 ops, 262144 KB)
- [ ] `auth_helper.cpp::HashPassword` — read those secrets, fall back to `MODERATE` when missing
- [ ] `auth_helper_test.cpp` — keep the existing fast tests (override the secret to `INTERACTIVE` for speed); add a `HashPasswordRespectsOpsLimitSecret` test asserting the cost parameter actually plumbed through
- [ ] Note: existing hashes remain valid — `crypto_pwhash_str_verify` reads parameters from the encoded hash, so old logins still work

### 2.6 Zeroize sensitive buffers on the password path
- [ ] Use `sodium_memzero` on the plaintext password copy after hashing/verifying inside `auth_helper.cpp` (best-effort, not a hard guarantee)
- [ ] No new tests required (hard to assert; manual inspection)

## Phase 3 — Endpoint authentication & authorization corrections

### 3.1 Move `/api/register` from URL path to JSON body — backend
- [ ] Change route in `endpoints/register.cpp` to `POST /api/register` reading `{ first_name, last_name, email, password }` from `req.body` JSON
- [ ] Validate each field; preserve existing `PreliminaryRegisterPerson` call
- [ ] `endpoints/register_test.cpp` — flip every test to body-based POST; add `RegisterMissingFieldBadRequest`
- [ ] No legacy fallback — old URL form is removed

### 3.2 `/api/register` — frontend
- [ ] `ui/src/app/shared/services/server-access.network.ts:206` (and the corresponding mock + spec) — switch to `POST /api/register` with JSON body, drop `withCredentials` for register? (no — keep `withCredentials` so the cookie set on the response sticks)
- [ ] `ui/src/app/auth/register/register.component.ts` — verify the call site
- [ ] Update `ui/src/app/shared/services/server-access.mock.ts` and `server-access.mock.spec.ts`
- [ ] `register.component.spec.ts` — confirm it still asserts the success/failure branches

### 3.3 `/api/verify` — SPA-routed POST with 1-hour TTL (Decision #2)
The verification email link points at the SPA, not the API. The SPA reads the secret from the query string, immediately POSTs to `/api/verify`, then scrubs the URL via `history.replaceState` so the secret doesn't linger in browser history.

**Backend:**
- [ ] `endpoints/verify.cpp` — change route to `POST /api/verify` reading `{ email, secret }` from JSON body. Drop the URL-path-secret variant entirely.
- [ ] `verify_test.cpp` — flip every test to body-based POST; add `VerifyMissingFieldBadRequest`
- [ ] Add the verify endpoint to the CSRF-exempt allow-list in Phase 4 (it's a bootstrap endpoint — the user has no `csrft` cookie yet) but enforce a strict `Origin` check there
- [ ] **Reduce the verification window default** — `business_logic/auth/secret_keys.h::kEmailVerificationExpirationWindowInMicros` default → **1 hour** (3,600,000,000 µs). Existing behavior is configurable via secret; only the default changes. Update the corresponding test value in `email_verifications_test.cpp` if it asserts on the default.

**Email content:**
- [ ] `business_logic/auth/person_verify_mail.cpp` — template's `{verify_link}` becomes `https://{kWebsiteAddress}/verify?email={url-encoded-email}&secret={url-encoded-base64url-secret}`. No `/api/` prefix; this is the SPA route, not the API.
- [ ] `person_verify_mail_test.cpp` — update the expected URL shape

**Frontend:**
- [ ] Add Angular route `/verify` → new `VerifyComponent` under `ui/src/app/auth/verify/`
- [ ] On `ngOnInit`: read `email` and `secret` from `ActivatedRoute.snapshot.queryParamMap`, call `serverAccess.verify(email, secret)` (POST `/api/verify`), and `history.replaceState({}, '', '/verify-success')` immediately after the POST is fired (don't wait for the response — the secret is what we want out of the URL bar)
- [ ] On success: route to `/login` with a "verified, please log in" toast
- [ ] On failure: route to `/login` with a generic "verification failed or expired" toast — do not echo the server message
- [ ] Tests: `verify.component.spec.ts` covering success, failure, missing-params; `server-access.mock.spec.ts` for the new `verify(email, secret)` mock method

### 3.4 `/api/me` — switch to GET
- [ ] `endpoints/me.cpp` registers `crow::HTTPMethod::Get`
- [ ] `me_test.cpp` and the `me.mock` frontend update
- [ ] Important interaction with Phase 4: `/api/me` is read-only so a CSRF check should not be required — confirm the eventual CSRF middleware skips safe methods

### 3.5 Remove hardcoded "Mason"/"Tyler" admin grant
- [ ] `endpoints/account_activation.cpp:65-82` — delete the name-based branch entirely
- [ ] Replace with: nothing — admins are minted via the seed-data path in `database_helper/create_database.cpp` or via the role-management UI
- [ ] Update / add `account_activation_test.cpp::AccountActivationDoesNotGrantAdminByName` that registers `firstName="Mason"` and asserts no `role_assignments` rows for the admin role exist after activation
- [ ] Confirm the seed path: ensure `create_database.cpp` provisions at least one admin user so that initial setup still works

### 3.6 Staff endpoints — enforce a `staff_access` permission
- [ ] Choose the permission name (`staff_access`) and seed it in `database_helper/create_database.cpp` (add the row to `permissions` and the `admin → staff_access` link in `role_permissions`)
- [ ] Add a helper `EndpointAuthHelper::RequirePermission(string_view name, crow::response& resp)` that returns `false` and writes `ErrorResponse::NotAuthorized` if the active user lacks the permission, so each endpoint becomes a one-liner
- [ ] Apply to: `endpoints/staff_checkin.cpp`, `staff_create_quick_account.cpp`, `staff_dropin_booking.cpp`, `staff_search_people.cpp`, `staff_upgrade_session.cpp`, `staff_upgrade_options.cpp`, `staff_upcoming_checkins.cpp`, and any other `staff_*.cpp` discovered while editing
- [ ] Tests: each `*_test.cpp` gets a `*NotStaff` case asserting 403 for a logged-in non-staff user, and a `*StaffOK` case verifying the existing flow still works once the role is granted

### 3.7 Image endpoint authorization
- [ ] `endpoints/get_photo.cpp` — require both `IsLoggedIn` and that the requested `(table, item_id)` is one the active user is allowed to read. Reuse `IsTableAllowed` and add an item-scoped check helper if any tables expose private images. Document a small allow-list of tables for which any logged-in user can fetch images.
- [ ] `endpoints/get_scaled_photo.cpp` — add the same auth check; replace the dedicated connection with the shared `TransactionProvider`
- [ ] Tests: `get_photo_test.cpp::GetPhotoForbiddenTable`, `GetPhotoUnauthenticated` (already exists?), `get_scaled_photo_test.cpp` similar

### 3.8 Strip sensitive columns from generic CRUD reads via column-level redact map (Decision #8)
**Approach:** column-level redact map, not a separate `people_credentials` table. Rationale:
1. **No surgery in the auth path.** `PersonHelper::VerifyPassword` reads `people.password_hash` directly today; splitting tables forces a join in the most security-critical code path. High risk for marginal gain.
2. **Single source of truth.** The map lives next to the existing admin metadata in `create_database.cpp`. A test ensures any column flagged as redacted never appears in a JSON response.
3. **Enforced at the JSON boundary**, so it covers every endpoint that uses `JsonFromDataResults` (the entire generic CRUD layer + any handler that uses it).
4. **The exception** that would change this decision: if we ever expose `people` to anonymous reads (e.g., a public studio directory). At that point a separate-table layout is worth the surgery. Until then, redact map is the right tradeoff.

**Schema-side / metadata layer:**
- [ ] In `sql_util/table_helpers/`, add a new helper `admin_column_redactions.{h,cpp,_test.cpp}` modeled after `admin_column_data_info.h/cpp`. Stores `(table_name, column_name)` rows in a new `admin_column_redactions` DB table.
- [ ] `db_schema/admin_column_redactions.{h,cpp}` — define the table; treat the same way as the other admin metadata tables.
- [ ] Register in `make_database_info.cpp` and `create_database.cpp::CreateTables` (do NOT add to the admin CRUD allow set — this metadata is read-only at runtime).

**Population:**
- [ ] In `database_helper/create_database.cpp`, add `PopulateAdminColumnRedactions(...)` that inserts the canonical redact list:
  - `(people, password_hash)` — never leak credential material
  - `(device_tokens, secret_hash)` — never leak token material
  - `(email_verifications, token_hash)` — never leak verification material
  - `(sessions, uuid)` — never leak session cookies via JSON; only the cookie path should ever emit it
  - `(login_attempts, *)` after Phase 5 — entire table; consider redacting at the table level rather than column level if cleaner
- [ ] Call `PopulateAdminColumnRedactions` from `CreateAndPopulateDatabases` next to the other `Populate*` calls.

**Enforcement at the JSON boundary:**
- [ ] In `sql_util/json/database_rest_helper.cpp`, the `DatabaseRESTHelper` (or its caller in `endpoint_auth_helper.cpp`) reads the redact map at startup into an in-memory `std::set<std::pair<std::string, std::string>>`.
- [ ] Modify `JsonFromDataResults(tableName, results)` to take the redact set and skip any column whose `(tableName, columnName)` is in the set. Same for `KeyValueTableToJson` overloads that know the source table.
- [ ] For endpoints that don't have a single source table (joins, custom JSON), the call site is responsible — but the audit shows the generic CRUD endpoints all flow through `JsonFromDataResults`, so this catches them.

**Tests:**
- [ ] `admin_column_redactions_test.cpp::AddAndListBasic`, plus a `RedactionMapContainsExpectedDefaults` test asserting `(people, password_hash)` etc. are present after `CreateAndPopulateDatabases`
- [ ] `database_rest_helper_test.cpp::JsonFromDataResultsRedactsConfiguredColumns` — call with `(tableName=people)` and confirm `password_hash` is absent from the JSON
- [ ] `endpoints/get_row_test.cpp::GetRowPeopleHidesPasswordHash` — end-to-end via `handle_full`
- [ ] `endpoints/get_table_rows_test.cpp::GetTableRowsPeopleHidesPasswordHash` — same for the list variant
- [ ] Negative test: `JsonFromDataResultsKeepsNonRedactedColumns` so we know the redaction is targeted, not accidentally aggressive

### 3.9 Lock down ownership-sensitive endpoints with regression tests
- [x] (Verified on follow-up) `endpoints/change_purchase_recipient.cpp:99-106` already rejects when `payerPersonId != loggedInPersonId`
- [x] (Verified on follow-up) `business_logic/payment/gift_permission_helper.cpp::AcceptRequest/DenyRequest` reject when the active user isn't the grantee, and `Revoke` requires either grantor or grantee
- [ ] Add `change_purchase_recipient_test.cpp::PostChangePurchaseRecipientNotPayerForbidden` (a logged-in non-payer attempts to change the recipient and gets 403)
- [ ] Add `gift_permissions_test.cpp::AcceptNotGranteeForbidden`, `DenyNotGranteeForbidden`, `RevokeOutsiderForbidden`
- [ ] `endpoints/set_user_info.cpp` and `endpoints/update_user_password.cpp` — confirm they always operate on `session.GetPersonId()` and never on a person_id read from the request body; add a test that a request body explicitly passing another person's ID is ignored

## Phase 4 — CSRF (cross-cutting; lower layers + endpoints + client)

### 4.1 Generate and set the `csrft` cookie alongside the session cookie
- [ ] `business_logic/auth/session.cpp::InitializeFromLogin` and `InitializeFromDeviceToken` — after setting the session cookie, generate a 32-byte random base64url string via `AuthHelper::RandomBytes` + `Base64Encode`, set as `csrft` cookie with `Path=/`, `SameSite=Lax`, `Secure` in prod, **NOT HttpOnly** (so the SPA can read it)
- [ ] Add the value to the `Session` object so handlers that need to compare don't have to re-parse cookies
- [ ] Tests in `session_test.cpp` asserting the cookie is present with the right attributes after each initialization path

### 4.2 Server-side CSRF middleware
- [ ] Add `CsrfGuard` middleware in `endpoints/csrf_guard.h/cpp` that, on POST/PUT/PATCH/DELETE, reads the `csrft` cookie and the `X-CSRF-Token` header, compares them via `AuthHelper::ConstantTimeEqual`, and rejects with 403 on mismatch. Skip on GET/HEAD/OPTIONS.
- [ ] Also enforce: `Origin` header (when present) is in the prod CORS allow-list; `Referer` (if present) starts with the same allowed origin
- [ ] Exempt the bootstrap endpoints (`/api/login`, `/api/register`, `/api/remember`) — they cannot have a CSRF cookie yet — but enforce a strict `Origin` check for those
- [ ] Wire the middleware into `App = crow::App<crow::CookieParser, crow::CORSHandler, CsrfGuard>` in `web_app.h`/`web_app.cpp`
- [ ] Tests in `csrf_guard_test.cpp`: missing-header, mismatch, match, GET-bypass, login-bypass, origin-mismatch on login

### 4.3 Frontend CSRF interceptor
- [ ] Add `ui/src/app/shared/interceptors/csrf.interceptor.ts` that, on every outgoing request whose method is in `[POST, PUT, PATCH, DELETE]`, reads the `csrft` cookie via `document.cookie` and adds `X-CSRF-Token: <value>`
- [ ] Register in `ui/src/app/app.config.ts` ahead of `ErrorInterceptor`
- [ ] Tests: `csrf.interceptor.spec.ts` covering header-attached-on-POST, no-header-on-GET, no-cookie-no-header

### 4.4 Logout must also clear `csrft`
- [ ] `endpoints/logout.cpp` clears `session_token`, `device_token`, **and `csrft`** with matching attributes (see also H10 / Phase 6)

## Phase 5 — Rate limiting & brute-force defense (PostgreSQL with write-behind via ThreadPool — Decision #11)

**Architecture rationale:** in-memory rate limiting on each ECS task means a multi-instance attacker spreads load across N tasks for N× allowance. Worse, ECS auto-scales — the "limit" floats with the fleet. Persistent state in PostgreSQL is the only multi-instance-safe option. To keep login latency tight, we **write attempt records asynchronously** via the existing `ThreadPool` (the same one used by `SessionUsed` for last-seen updates). Reads are synchronous (one indexed `SELECT count(*)`).

**Key invariant:** the *recording* of a failure is async; the *decision to lock* is sync (in the same transaction as the password check). This way two parallel attempts at the threshold can over-count by at most one (acceptable), but the lockout, once decided, is committed before the response goes back.

### 5.1 `login_attempts` table
- [ ] `db_schema/login_attempts.{h,cpp}` — columns: `id BIGSERIAL PK`, `email_lower citext NOT NULL`, `ip inet NOT NULL`, `attempted_at BIGINT NOT NULL` (microseconds since epoch), `success BOOLEAN NOT NULL`, `kind TEXT NOT NULL` (one of `login`, `verify`, `remember`)
- [ ] Indexes: `(email_lower, attempted_at DESC)` and `(ip, attempted_at DESC)`
- [ ] Register in `make_database_info.cpp` and `create_database.cpp::CreateTables`. Do **not** add to the admin CRUD list (sensitive PII; access only via dedicated admin reports if ever needed).
- [ ] `sql_util/table_helpers/login_attempts.{h,cpp,_test.cpp}`:
  - `void RecordAttempt(Transaction&, string_view emailLower, string_view ip, string_view kind, bool success)`
  - `int64_t RecentFailureCountForEmail(Transaction&, string_view emailLower, string_view kind, int64_t windowMicros)`
  - `int64_t RecentFailureCountForIp(Transaction&, string_view ip, string_view kind, int64_t windowMicros)`
  - `void PurgeOlderThan(Transaction&, int64_t ageMicros)` — for the periodic cleanup job

### 5.2 Synchronous gate before password verify (per-email + per-IP)
- [ ] Add secrets:
  - `kAuthLoginMaxFailuresPerEmailPerWindow` (default 10)
  - `kAuthLoginFailureWindowInMicros` (default 15 min in µs)
  - `kAuthLoginMaxFailuresPerIpPerWindow` (default 50)
  - `kAuthAccountLockoutAfterFailures` (default 10)
  - `kAuthAccountLockoutDurationInMicros` (default 30 min)
- [ ] Add a `business_logic/auth/login_gate.{h,cpp}` helper:
  - `enum class LoginGateResult { Allow, RateLimitedEmail, RateLimitedIp, AccountLocked }`
  - `LoginGateResult CheckBeforeVerify(Transaction&, SecretsHelperPtr, string_view email, string_view ip)` — runs three small queries: `people.locked_until` (combined with the password-hash lookup the auth path was going to do anyway), per-email failure count, per-IP failure count. Returns the first failing reason, or `Allow`.
- [ ] `business_logic/auth/person.cpp::VerifyPassword` (or a new `LoginPerson` wrapper) — call `CheckBeforeVerify` before doing Argon2id; on rate-limited result, return early with a generic failure (don't even verify the password — saves CPU and avoids timing leaks via the password-verify branch).
- [ ] Tests on `login_gate_test.cpp` covering each branch.

### 5.3 Synchronous lockout decision (sticky on the row)
- [ ] On a failed password verify, in the same transaction:
  - `UPDATE people SET failed_login_attempts = failed_login_attempts + 1, locked_until = CASE WHEN failed_login_attempts + 1 >= $threshold THEN now_us() + $duration ELSE locked_until END WHERE id = $id` (one statement so it's atomic)
  - If the `RETURNING` shows `locked_until` was just set, also enqueue an `admin_alerts` row via the same async pipe (Phase 9) — but the lockout itself is already committed
- [ ] On a successful password verify: `UPDATE people SET failed_login_attempts = 0, locked_until = NULL WHERE id = $id`
- [ ] Tests on `person_test.cpp`: `LoginAccountLockoutAtThreshold`, `LoginSuccessClearsLockout`, `LoginAccountStaysLockedDuringWindow`, `LoginAccountUnlocksAfterWindow`

### 5.4 Asynchronous attempt recording (write-behind via ThreadPool)
- [ ] After the synchronous password-verify path returns to the endpoint, enqueue a single lambda on `ThreadPool::GetInstance().Queue(...)` that:
  - Captures the `TransactionProvider&` by reference (the singleton lives at least as long as the request handler)
  - Captures `email`, `ip`, `kind`, `success` by value
  - Inside the lambda: `RunInTransaction { LoginAttempts::RecordAttempt(...) }`
- [ ] The lambda is fire-and-forget; the HTTP response goes back to the client without waiting. Login latency budget is unchanged (Argon2id at ≈250ms dominates).
- [ ] Tests in `person_test.cpp` use `ThreadPool::GetInstance().Shutdown()` (which calls `Join`) at the end of the test to make assertions deterministic — same pattern as `SessionUsedBasic`.
- [ ] Multi-instance correctness: every task reads the latest count from PG; no shared in-memory state needed.

### 5.5 IP plumbing
- [ ] `business_logic/auth/proxy_trust.cpp` — confirm/extend `ResolveClientIp(crow::request&)` so it returns the right client IP regardless of whether we're behind CloudFront or running locally. Reads `KNOTTYYOGA_TRUST_PROXY` env var (already in place); when set, prefers `X-Forwarded-For` last entry; otherwise falls back to the connection peer address from `req.remote_ip_address` (or whatever Crow exposes).
- [ ] `endpoints/login.cpp`, `endpoints/verify.cpp`, `endpoints/remember.cpp` — call `ResolveClientIp` and pass into the gate / async recorder.
- [ ] Tests on `proxy_trust_test.cpp` for both trust-on and trust-off modes.

### 5.6 Verification & remember-me throttling
- [ ] Reuse the same gate machinery for `/api/verify` (kind=`verify`, default 30 failures / 15 min / IP, 30-min block) and `/api/remember` (kind=`remember`, default 50 failures / 15 min / IP, 1 h block).
- [ ] New secrets: `kAuthVerifyMaxFailuresPerIpPerWindow`, `kAuthVerifyFailureWindowInMicros`, `kAuthRememberMaxFailuresPerIpPerWindow`, `kAuthRememberFailureWindowInMicros` (defaults as above).
- [ ] Tests on `verify_test.cpp::VerifyRateLimitedPerIp` and `remember_test.cpp::RememberRateLimitedPerIp`.

### 5.7 Generic error shape (no enumeration)
- [ ] All auth-failure paths return the same response shape: `401 {"error":"invalid_credentials"}` for any failure (wrong password, unknown email, account locked) below the rate-limit threshold; `429 {"error":"too_many_attempts"}` only when rate-limited or locked.
- [ ] Specifically, do **not** distinguish "account is locked" from "wrong password" in 401 responses — both look identical from the outside. The 429 means "stop trying for now"; no detail about why.
- [ ] Test `person_test.cpp::LoginUnknownUserSameAsWrongPassword` (timing — pad with a no-op Argon2id verify against a sentinel hash so unknown-email and wrong-password take similar wall-clock time). Optional but worth doing.

### 5.8 Periodic cleanup
- [ ] Add a `ThreadPool` job in `main.cpp` that, every hour, calls `LoginAttempts::PurgeOlderThan(transaction, kAuthLoginAttemptsRetentionInMicros)` (default: 30 days). Prevents unbounded growth.
- [ ] Test in `login_attempts_test.cpp::PurgeOlderThanBasic`.

## Phase 6 — Cookie hygiene cleanup

### 6.1 Logout clears cookies with attributes that match the originals
- [ ] `endpoints/logout.cpp` — refactor to call a shared `Auth::ClearAuthCookies(cookieManager, secrets, isProd)` helper that knows the canonical attributes (path, domain, secure, httpOnly, sameSite) and writes `Max-Age=0` for `session_token`, `device_token`, and `csrft`
- [ ] Drop `SameSite=None`; use the same `Lax` value used at set-time
- [ ] Tests in `logout_test.cpp` asserting cookie attributes on the response match the set-time attributes (use the `CookieManagerTest::GetCookieProperties()` map)

### 6.2 Defensive: CookieManager warns on `SameSite=None` without `Secure`
- [ ] In the production `CookieManagerImpl::SetCookie`, log an error if `sameSite == None && !secure` (fail-fast in dev/test). Cheap insurance against re-introduction.
- [ ] Test in `cookie_manager_test.cpp`

## Phase 7 — Security headers, error handling, server banner

### 7.1 SecurityHeaders middleware
- [ ] `endpoints/security_headers.{h,cpp}` middleware that, on every response, sets:
  - `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` (only in prod mode)
  - `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; base-uri 'none'` (tune in real deploy)
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: strict-origin-when-cross-origin`
  - `Permissions-Policy: camera=(), microphone=(), geolocation=()`
  - Override `Server: Knotty Yoga` (or omit entirely)
- [ ] Wire into `App` middleware list in `web_app.h`
- [ ] Tests confirming headers are present on a sample of endpoints (auth, public, admin)

### 7.2 Vary: Origin on CORS responses
- [ ] `business_logic/auth/server_config.cpp::ConfigureCors` — add `Vary: Origin` header (Crow's CORS middleware may already do this; verify and add if not)
- [ ] `server_config_test.cpp` — assert the header is present in a CORS-handled response

### 7.3 Global Crow error handler
- [ ] In `endpoints/web_app.cpp`, register a fallthrough that converts uncaught exceptions to `500 internal_server_error` with body `{"error":"internal_server_error"}` and logs the exception text server-side only
- [ ] Audit endpoints that currently echo `e.what()` (e.g., `endpoints/db_schema.cpp:31`) and replace with the generic shape via `ErrorResponse::InternalError`
- [ ] Tests: `web_app_test.cpp` (or per-endpoint) — throw inside a handler, assert response is generic and the exception text is *not* in the body

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

# Open Questions

These are the decisions I want your input on rather than guessing:

1. **CSRF approach** — happy with the double-submit cookie pattern (`csrft` non-HttpOnly + `X-CSRF-Token` header) per your design doc, or do you prefer the alternative of a per-session CSRF token stored on the server and signed?
	- Mason- Which would you recommend and why?
	- Claude — **Double-submit cookie.** Reasons:
		1. **Stateless on the server.** No new per-session row to track — fits your stateless-ECS design (you flagged this in Q11). The signed-token alternative needs either a server-side store or an HMAC key managed in the same way as Phase 8's secret key.
		2. **Crow already has `CookieParser`**, so reading/writing `csrft` is a couple of lines. The signed-token variant needs you to maintain an HMAC key, signature format, and rotation story.
		3. **It's the industry default for SPA-cookie auth** (Django CSRF middleware, Rails `protect_from_forgery`, ASP.NET anti-forgery). Lots of prior art for the failure modes.
		4. The one weakness of double-submit (a same-origin XSS can read the cookie and forge requests) is **not** materially worse than signed tokens — at that point an XSS can also read your other in-page state. CSP (Phase 7) is the right defense for that, not signed CSRF tokens.
		- **Defense in depth** I'd add on top of the double-submit: also reject when `Origin` header is present and not in the prod allow-list. That's already in plan item 4.2.
	- Mason- I'll go with the double submit :)
2. **Verification email link target** — for moving `/api/verify` off URL-path-with-secret (item 3.3): is it OK if the verification link in the email points at the SPA (e.g., `/verify?email=...&secret=...`) and the SPA POSTs to the API? That removes the secret from server logs but does put it in browser history of whoever opens the email.
	- Mason- What do you think and suggest?
	- Claude — **Yes, SPA-routed.** The email link is a GET to `/verify?email=...&secret=...`; the SPA reads the params and immediately POSTs to `/api/verify`, then routes the user to the next step (login or dashboard).
		- The "secret in browser history" concern is real but small: the secret is single-use, and the email itself is already visible to anyone with the user's mailbox.
		- The mitigations to layer on:
			1. **Single-use enforcement** is already there (`DoEmailVerification` deletes the row on success; Phase 2.3 makes attempt counting atomic).
			2. **Short TTL** — `kEmailVerificationExpirationWindowInMicros` (already a secret). Recommend setting it to 1 hour for new registrations, not the current "default several days" if that's still in place; check the default.
			3. After the SPA POSTs, **`history.replaceState({}, '', '/verify-success')`** so the secret is replaced in the browser's URL bar and history within milliseconds.
		- The alternative ("mail link points directly at `GET /api/verify` because it's idempotent") puts the secret in CloudFront access logs, your server access logs, and any CDN log retention. Worse than browser history, which is local-only.
	- Mason- Let's stay with SPA-routed and I'm okay reducing the window to one hour.
3. **Argon2id strength** — is `MODERATE` (≈250ms login on a modern server) acceptable, or do you want `INTERACTIVE` to keep logins snappier?
	- Mason- Is there a security risk going with the faster option?
	- Claude — **Recommend `MODERATE`, but it's not a huge deal either way for a yoga-studio app.** Risk model:
		- The thing Argon2id defends against is **offline brute-force after a database compromise**. Online guessing is bounded by network round-trips and the rate-limiter (Phase 5).
		- Rough numbers per password guess on a single modern CPU core: `INTERACTIVE` ≈ 50ms, `MODERATE` ≈ 250ms. Translate to hardware an attacker actually has (cheap GPU farm, ~1000 cores effective): `INTERACTIVE` ≈ 20k guesses/sec, `MODERATE` ≈ 4k guesses/sec.
		- Against a strong random password (10+ chars from a 70-char alphabet ≈ 60 bits), both make brute force impractical.
		- Against a **weak** password (top 10k common, ~13 bits), `INTERACTIVE` cracks in ~0.5s, `MODERATE` in ~2.5s. Neither helps. Real defense for weak passwords is the breach-list check you can add later (e.g., HaveIBeenPwned password range API).
		- **Net:** `MODERATE` adds ~200ms to logins (imperceptible to users since it's once per 8h) and 5× the offline-attack cost. Cheap insurance.
		- The plan already makes the parameter a secret (`kAuthArgon2OpsLimit`), so you can A/B if you're worried.
	- Mason- Let's stay with moderate then.
4. **CORS in dev mode** — the prod path is correct; should I leave the comment-only dev path as-is (relying on Angular proxy) or wire dev CORS so the API works without the Angular proxy too?
	- Mason- I definitely want ng-serve to continue working. What do you recommend here?
	- Claude — **Keep the no-CORS-in-dev path as the default** (so `ng serve` + Angular proxy keeps working unchanged), and **add an opt-in** that activates only when an env var is set:
		- New env var `KNOTTYYOGA_DEV_CORS_ORIGIN`. If set in dev mode, `ServerConfig::ConfigureCors` adds that exact origin with `allow_credentials()`. If unset (the default), the dev path stays a no-op like today.
		- That way:
			1. Day-to-day with `ng serve`: no change. Proxy makes everything same-origin, no CORS needed.
			2. Direct API testing (curl from another origin, a second SPA running on a different port, exploratory work in Postman with a different origin): you set the env var and it just works.
			3. Prod path is unchanged.
		- Document the env var in `CLAUDE.md` and in `proxy.conf.json` comments.
	- Mason- Your recommendation sounds great.
5. **Lockout duration / threshold** — is 10 failures / 15-min window / 30-min lockout per email acceptable, or do you want different numbers? Same question for IP.
	- Mason- Do these seem okay to you? Should we make these configurable via server secrets?
	- Claude — **Yes, all configurable via secrets** (the plan already lists them: `kAuthLoginMaxFailuresPerEmailPerWindow`, `kAuthLoginFailureWindowInMicros`, `kAuthLoginMaxFailuresPerIpPerWindow`, `kAuthAccountLockoutAfterFailures`, `kAuthAccountLockoutDurationInMicros`). Recommended defaults:
		- **Per email:** 10 failures / 15 min → 30-min soft lockout. After 3 lockouts in 24 h, hard-lock requiring email-based unlock. (The 24h+lockout-counter logic can be a follow-up; not in v1.)
		- **Per IP:** 50 failures / 15 min → 1 h temporary block. Wider net to catch credential stuffing across many accounts from the same IP.
		- **Verify endpoint:** 30 failures / 15 min / IP → 30 min block. Looser because legitimate users may click multiple verification links from the same IP.
		- **Remember endpoint:** 50 failures / 15 min / IP → 1 h block.
		- **Tuning** — start with these defaults, watch `admin_alerts` for "account locked" events, adjust. The whole point of using secrets is you can tune without redeploying.
		- **Important UX note:** when lockout is hit, the response should be a generic 401 ("invalid credentials") or 429 ("too many attempts") — never reveal *which* user is locked. Plan item 5.5 covers this.
	- Mason- That sounds great.
6. **Encryption key storage** — for Phase 8, can we depend on a `KNOTTYYOGA_SECRET_KEY` env var, or is there a key-management story (KMS, HSM) you want to plug into instead?
	- Mason- What do you recommend? Is there a better solution for letting AWS manage this?
	- Claude — **Two-tier recommendation**, ship the simple one now and upgrade later:
		1. **v1 (now): env var fed from AWS Secrets Manager.** In your ECS task definition, set the container env var `KNOTTYYOGA_SECRET_KEY` from a Secrets Manager entry (`secrets:` block in the task definition). The container sees a normal env var; AWS handles secret storage, rotation, and IAM. **No code changes** beyond reading the env var. This is the standard pattern for ECS and is what most teams ship as "v1".
		2. **v2 (later, when it matters): AWS KMS + envelope encryption.** Generate a KMS Customer Master Key. On startup, the server uses the AWS SDK to call `GenerateDataKey`, which returns a plaintext data key + an encrypted version. You encrypt secrets with the data key and store the *encrypted* data key alongside (or in `config_secrets`). On read, you call `Decrypt` on the encrypted data key. Benefits: KMS rotates the master key without code changes, IAM controls who can decrypt, every decrypt is audited in CloudTrail.
		- **Why not v2 now:** adds AWS SDK dependency, network calls in startup hot path, IAM plumbing, and tests need a mock KMS client. For a small studio app, v1 (env-var-from-Secrets-Manager) is the right risk/effort tradeoff.
		- **What v1 needs in the codebase:** literally just `std::getenv("KNOTTYYOGA_SECRET_KEY")` and base64-decode it to 32 bytes. No AWS SDK in the C++ build.
		- **One non-obvious thing:** Secrets Manager → ECS env var injection happens at task-start time. If you rotate the secret, ECS won't pick it up until the next task revision. That's fine for an at-rest encryption key (which you rarely rotate); not fine for short-lived API tokens (which is why `kSquareAccessToken` etc. live in `config_secrets`, not env vars — they can be rotated via the admin UI without an ECS deploy).
	- Mason- I'll go with your recommendation but please update the documentation accordingly.
7. **Removing the "Mason"/"Tyler" hardcode (3.5)** — am I OK to delete this outright assuming the seed-data path mints at least one admin? If you currently bootstrap admin only via that hardcode, I need to add a seed-data admin first.
	- Mason- I did this during bootstrapping to get things working but now we build those accounts and give them the right permissions so this is no longer needed. Feel free to delete.
8. **Generic CRUD redaction (3.8)** — is the column-level redact map acceptable, or do you want sensitive columns moved to entirely separate tables (e.g., `people_credentials`) so they can never accidentally be selected?
	- Mason- What do you recommend?
	- Claude — **Column-level redact map.** Reasons:
		1. **No schema migration / no code surgery in the auth path.** Today `people.password_hash` is read directly by `PersonHelper::VerifyPassword`. Splitting it into `people_credentials` means the auth path needs a join (or a second lookup), and so does every other place that touches the password (registration, password change, hash format upgrades). All of that is risky surgery in the most security-sensitive code path you have.
		2. **The redact map is auditable in one place.** A new dev adding a sensitive column adds it to the redact map next to the column declaration; there's a single test that fails if a configured-redact column appears in a JSON response. Splitting tables is auditable too, but only if every developer remembers the convention every time.
		3. **The "physical separation" win you'd be paying for is small in your model.** It buys "no risk of accidental SELECT \*" — but you don't write SELECT \* in this codebase; the JSON layer reflects against the table's columns. The redact pass runs at the same point.
		- **The exception that would change my mind:** if you ever expose a *public, unauthenticated* read on `people` (a public studio directory, instructor bios, etc.), I'd want secrets in a separate table. You'd be one config bug away from leaking hashes. Until then, the redact map is the better tradeoff.
		- **Concrete recommendation:** put the map in `database_helper/create_database.cpp` next to the existing admin metadata population, key on `(table, column) → redaction-policy`, and apply in `sql_util/json/database_rest_helper.cpp::JsonFromDataResults` so the policy is enforced at the JSON boundary, not in each endpoint.
	- Mason- Okay, let's go with the column-level redact map. Please add all of this to the documentation though.
9. **`config_secrets` admin UI (8.2)** — is it acceptable to drop the generic table editor for secrets in favor of a dedicated UI that masks values, or do you need to keep the generic editor for now?
	- Mason- I'm fine with moving this to a dedicated UI. I just needed something to get up and running.
10. **SameSite policy** — your current code uses `Lax`. Are you open to `Strict` for `session_token` (UX cost: cross-site links into the app log the user out)? `Lax` is fine; just confirming.
	- Mason- Honestly, I was just getting things working. I need the local dev scenario to keep working though. What would you recommend here? I am okay with strict but I need ng-serve with the proxy continuing to work for dev scenario work.
	- Claude — **Stay with `Lax`.** Specifically:
		1. **`Lax` is sufficient against CSRF that SameSite is meant to prevent.** Browsers downgrade to no-cookies on cross-site **POST/PUT/PATCH/DELETE/`fetch()` from another origin**. The only thing `Strict` adds is "no cookies on top-level navigations from another origin" — i.e. clicking a link from email or a Google search result into your app shows the user logged out, which is a UX cliff for no real security gain (we already have CSRF middleware in Phase 4, security headers in Phase 7, and CloudFront origin guard).
		2. **`Strict` is hostile to email-driven flows** like `/verify` (Phase 3.3). The link from the verification email is a top-level navigation; with `Strict` the cookie isn't sent. With `Lax` it is.
		3. **`Lax` works the same way through the Angular proxy in dev.** Same-origin requests aren't subject to SameSite at all, so the proxy makes this a non-issue. `Strict` would also work in dev but I'd rather match dev to prod behavior.
		- **The compromise to consider** if you want the strictest viable posture: `SameSite=Strict` for the `session_token` cookie *only*, keep `Lax` for `device_token` (so cross-site nav from email still triggers `/api/remember` re-auth) and `csrft`. The cost is the user gets a one-time re-auth via the device token whenever they land in the app from outside. Not awful UX. But honestly, **stick with `Lax`** until you have a reason to go further.
	- Mason- Let's stick with Lax.
11. **Rate limiting persistence** — `login_attempts` in PostgreSQL is simple and correct but adds write load. Are you OK with that, or do you want an in-memory rate limiter keyed off IP/email for the hot path with the DB used only for permanent lockouts?
	- Mason- I'm okay doing that in memory if it speeds things up and doesn't cause issues. Does this cause an issue if we move to ECS and have multiple server instances? The server is stateless currently and that is an important design. If we need to hit the database, can we have this write on a worker thread but still return immediately to the client?
	- Claude — Your instinct is exactly right. **Use PostgreSQL with write-behind via the existing `ThreadPool`.** Keep the server stateless across instances. Concrete shape:
		1. **In-memory only does NOT work for multi-instance ECS.** Each task has its own counter, so an attacker spreads requests across N tasks and gets N× the allowance. Worse, ECS auto-scales — your "limit" floats with the fleet. So in-memory is out.
		2. **Pure synchronous DB-write per attempt** is correct but adds an extra `INSERT` (and an `UPDATE people SET failed_login_attempts = …`) on every login. Login isn't a hot path, but each request adding two writes is wasteful.
		3. **Recommended pattern (write-behind):**
			- **Synchronous read** of "recent failure count for (email, window)" via one indexed `SELECT count(*)`. Fast, cached by Postgres' shared buffers within the window.
			- **Synchronous read** of `people.locked_until`. Single column on a row you're about to read for password verify anyway — effectively free (combine into one query).
			- Reject early if either threshold is hit.
			- Run the password verify (the slow part — Argon2id, ~50–250ms).
			- **Asynchronous write** of the attempt record + the per-user counter via `ThreadPool::GetInstance().Queue(...)`. The lambda captures a `TransactionProvider` reference and the relevant ints by value, runs `RunInTransaction`, returns. The HTTP response goes back to the client without waiting.
			- The user-visible login latency budget is unchanged (Argon2id dominates by orders of magnitude).
		- **One subtlety:** if the threshold is at the boundary, two parallel attempts could both pass the synchronous read before either's async write lands. Acceptable for rate-limiting (worst case attacker gets +1 attempt past the limit). Not acceptable for the hard-lock decision — when the synchronous read shows "this is the 10th failure", the lockout `UPDATE` should be **synchronous** (in the same transaction as the lockout decision). Async only the *recording* of failures, not the decision to lock.
		- **`admin_alerts` event for "account locked"** is also written by the same async worker; one DB round-trip total per attempt.
		- **Multi-instance safe**: all state lives in Postgres; any task can read the current count.
	- Mason- That sounds great. I'll go with your recommendation but please add all of this to the documentation.
12. **`/api/me` GET (3.4)** — switching from POST to GET technically changes the public API contract. Confirm you want this; otherwise leave it as POST and just exempt it from CSRF for being read-only.
	- Mason- I'm okay moving this to post. We haven't deployed yet. So there is no contract to break. Hence why I'm doing this review now :)
13. **`change_purchase_recipient` / `gift_permissions` ownership** — please confirm whether these endpoints already check ownership; if you remember, save me the grep.
	- Mason- I don't remember. Please look this up.
	- Claude — Looked. **Both already enforce ownership correctly.**
		- `endpoints/change_purchase_recipient.cpp:99-106` — fetches the purchase row, reads `payer_person_id`, and rejects with `NotAuthorized("Only the payer can change the recipient")` if it doesn't equal `session.GetPersonId()`.
		- `business_logic/payment/gift_permission_helper.cpp::AcceptRequest` (lines 71-107) and `DenyRequest` (109-134) both check `actualGranteeId != granteePersonId` and throw `invalid_argument("Only the grantee can …")`.
		- `gift_permission_helper.cpp::Revoke` (136-154) allows either grantor or grantee, throws otherwise.
		- I downgraded the H3 finding and rewrote plan item 3.9 to "verified — add regression tests for the negative cases". Findings M12 is resolved.
14. **Phase ordering** — would you prefer to interleave phases (e.g., do Phase 7 security headers very early since they're zero-risk) or strictly sequential as written? The plan is bottom-up by layering, but some phases are independent.
	- Mason- I don't have an opinion here. What do you recommend?
	- Claude — **Slightly re-ordered**, pulling the cheap-and-independent wins forward:
		- **Phase 1** (schema integrity & lookup performance) — first; everything later depends on these tables/indices.
		- **Phase 1b — pull forward Phase 7** (security headers + Crow error handler + server banner). Single-middleware change, no behavior risk, immediate prod-readiness win. Would be embarrassing to ship without.
		- **Phase 1c — pull forward Phase 12.1** (fail-loud-in-prod startup checks) and the `KNOTTYYOGA_TRUST_PROXY` / `KNOTTYYOGA_ORIGIN_SECRET` assertions. Tiny, isolated, prevents an entire class of "we deployed without `X` set" foot-guns.
		- **Phase 2** (auth primitives — constant-time compare, atomic verify counter, atomic device-token rotation, Argon2id bump). Low layer, supports later phases. Adds the helpers Phase 4 needs.
		- **Phase 3** (endpoint authn/authz — register-as-POST, kill the Mason/Tyler hardcode, staff role gate, image authz, redact map). Medium risk, high value, mostly endpoint-level.
		- **Phase 6** (cookie hygiene cleanup — logout cookie attributes). Pull forward — it's a small fix that removes the SameSite=None-without-Secure bug. Doing it before CSRF means the new `csrft` cookie can be cleared correctly by the same shared helper.
		- **Phase 4** (CSRF middleware + frontend interceptor). Now that auth primitives exist and cookie clearing is fixed.
		- **Phase 5** (rate limiting + lockout). Needs the lockout columns from Phase 1 and the async pattern is the same as what's used elsewhere; can run in parallel with Phase 4 if you want.
		- **Phase 8** (encryption at rest + admin secrets endpoint). Independent; can slot in any time after Phase 1, but nice to do after Phase 5 so there's a stable observability baseline.
		- **Phase 9** (auth event log + admin_alerts delivery). Builds on phases 5 and 8.
		- **Phase 10** (frontend bootstrap + returnUrl allow-list). Independent of backend phases; can be done by anyone in parallel.
		- **Phase 11** (latent SQL-concat in `create_database.cpp` cleanup). Pure code-quality cleanup — bottom of the stack. Or pair with Phase 1 since you're touching that file anyway.
		- **Why this order**: the early phases (1, 1b, 1c) are nearly risk-free and produce a server that's noticeably more production-ready immediately. The auth primitives (2) unlock the high-value middleware work (4, 5). Phase 10 (frontend) can be parallelized to a different person's work queue if you ever pick up help.
	- Mason- That sounds great to me.