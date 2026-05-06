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
- **H3. No CSRF / Origin / Referer enforcement on payment, purchase, and gift-permission endpoints.** `endpoints/{purchase_create,purchase_pay_card,payments,cards,change_purchase_recipient,gift_permissions}.cpp`. Combined with C1, these are forgery-prone; combined with absent ownership checks (verify these exist) they're escalation-prone. Audit `change_purchase_recipient` and `gift_permission_{accept,deny}` for ownership checks against `session.GetPersonId()`.
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
- **M12. `change_purchase_recipient` and `gift_permission_*` ownership checks need verification.** Listed for completeness — the audit could not confirm these always check `session.GetPersonId()` against the row owner.

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

### 3.3 `/api/verify` — switch to POST body
- [ ] Same shape of change as 3.1 for `endpoints/verify.cpp`. The verify URL is still constructable from the email (the email contains a link), so the email body needs the link to point at a small client page that POSTs the secret rather than calling the backend directly. Two ways:
  - (a) Keep the email-link a GET to the SPA at `/verify?secret=...&email=...`; the SPA reads the params and POSTs to `/api/verify`. (recommended)
  - (b) Allow GET fallback on the server but require a CSRF-equivalent nonce embedded in the token itself
- [ ] Update `business_logic/auth/person_verify_mail.cpp` template to point at the SPA `/verify` route, not the API
- [ ] Add the SPA route + component or reuse an existing one
- [ ] Tests: backend `verify_test.cpp` for the new POST shape; frontend component spec for the new component

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

### 3.8 Strip sensitive columns from generic CRUD reads
- [ ] Add a column-redaction map in `database_helper/create_database.cpp` (a new `PopulateAdminColumnRedactions` step) — `(people, password_hash)`, `(device_tokens, secret_hash)`, `(email_verifications, token_hash)`, `(sessions, uuid)`
- [ ] Wire the map through `endpoints/endpoint_auth_helper.h/cpp` and apply it inside `sql_util/json/database_rest_helper.cpp::JsonFromDataResults` (or one level up, before the data leaves the server)
- [ ] Tests: `database_rest_helper_test.cpp::JsonFromDataResultsRedactsConfiguredColumns`, plus `endpoints/get_row_test.cpp::GetRowPeopleHidesPasswordHash`

### 3.9 Verify ownership on object-mutating endpoints
- [ ] `endpoints/change_purchase_recipient.cpp` — confirm it checks the row's `recipient_person_id` (or buyer) against `session.GetPersonId()` before mutating; add the check if missing; add a `_NotOwnerForbidden` test
- [ ] `endpoints/gift_permissions.cpp` (Accept/Deny/Delete) — same check, ensure only the intended grantee/grantor can act; tests for each of the three negative cases
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

## Phase 5 — Rate limiting & brute-force defense

### 5.1 `login_attempts` table
- [ ] `db_schema/login_attempts.{h,cpp}` — `(id BIGSERIAL, email_lower citext NOT NULL, ip inet NOT NULL, attempted_at BIGINT NOT NULL, success BOOLEAN NOT NULL)` with indexes on `(email_lower, attempted_at DESC)` and `(ip, attempted_at DESC)`
- [ ] Register in `make_database_info.cpp` and `create_database.cpp::CreateTables`. Do **not** add to the admin CRUD list.
- [ ] `sql_util/table_helpers/login_attempts.{h,cpp,_test.cpp}` — `RecordAttempt(transaction, email, ip, success)`, `RecentFailureCountForEmail(transaction, email, windowMicros)`, `RecentFailureCountForIp(transaction, ip, windowMicros)`, `PurgeOlderThan(transaction, ageMicros)`

### 5.2 Per-email and per-IP login throttling
- [ ] Secrets: `kAuthLoginMaxFailuresPerEmailPerWindow` (default 10), `kAuthLoginFailureWindowInMicros` (default 15 min in µs), `kAuthLoginMaxFailuresPerIpPerWindow` (default 50), `kAuthAccountLockoutAfterFailures` (default 10), `kAuthAccountLockoutDurationInMicros` (default 30 min)
- [ ] `business_logic/auth/person.cpp::VerifyPassword` (or a wrapper `Login`) — before the password check, consult `LoginAttempts` for both keys and `people.locked_until`; reject early with a generic "invalid credentials" error if blocked. After the check, record the attempt (success or failure), and on failure increment `people.failed_login_attempts`, setting `locked_until` if the threshold is hit. On success, zero the counter and clear `locked_until`.
- [ ] Tests: `person_test.cpp::LoginRateLimitedPerEmail`, `LoginRateLimitedPerIp`, `LoginAccountLockoutAfterRepeatedFailures`, `LoginSuccessClearsLockout`

### 5.3 IP plumbing
- [ ] `endpoints/login.cpp` — extract the client IP via `proxy_trust.cpp`'s helper (which already prefers `X-Forwarded-For` only when `KNOTTYYOGA_TRUST_PROXY` is set, falling back to the connection peer otherwise) and pass it to `Login`
- [ ] Add a unit test on `proxy_trust_test.cpp` that confirms the trust gate is honored

### 5.4 Verification & remember-me throttling
- [ ] Reuse the same per-IP machinery for `/api/verify` and `/api/remember` (looser limits — e.g., 30 failures / 15 min / IP) — protects against device-token stuffing
- [ ] Tests on `verify_test.cpp` and `remember_test.cpp`

### 5.5 Generic 401/429 error shape
- [ ] All auth-failure paths return the same JSON body and status (`401 invalid_credentials` or `429 too_many_attempts` after lockout) so attackers can't distinguish "wrong password" from "account locked"
- [ ] Test that `429` is returned only after the threshold is exceeded

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

## Phase 8 — Secrets at rest

### 8.1 Encrypt `config_secrets.value`
- [ ] Add a master encryption key bootstrapped from env var `KNOTTYYOGA_SECRET_KEY` (32 bytes, base64). Refuse to start in prod mode if it's missing.
- [ ] In `sql_util/table_helpers/config_secrets.cpp` (or a new wrapper layer in `business_logic/auth/secrets_at_rest.{h,cpp}`), encrypt with `crypto_secretbox_easy` (libsodium) on write and decrypt on read. Store ciphertext + nonce concatenated and base64-encoded.
- [ ] Migration: a one-shot startup pass that reads each row, detects whether it's already in the new format (nonce|ciphertext prefix), and rewrites if not.
- [ ] Tests: `config_secrets_test.cpp` round-trip; `secrets_helper_test.cpp` ensures the higher layer continues to read the cleartext

### 8.2 Locked-down admin secrets endpoint
- [ ] (Replaces the deletion in 1.8) `endpoints/admin_secrets.{h,cpp}` — `GET` returns `{name}` only (no values), `PUT` writes a value (audited to `admin_alerts`). Requires `admin` role.
- [ ] Frontend adjustments: change the admin "secrets" UI to use this endpoint instead of generic CRUD on `config_secrets`
- [ ] Tests on both ends

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
2. **Verification email link target** — for moving `/api/verify` off URL-path-with-secret (item 3.3): is it OK if the verification link in the email points at the SPA (e.g., `/verify?email=...&secret=...`) and the SPA POSTs to the API? That removes the secret from server logs but does put it in browser history of whoever opens the email.
	- Mason- What do you think and suggest?
3. **Argon2id strength** — is `MODERATE` (≈250ms login on a modern server) acceptable, or do you want `INTERACTIVE` to keep logins snappier?
	- Mason- Is there a security risk going with the faster option?
4. **CORS in dev mode** — the prod path is correct; should I leave the comment-only dev path as-is (relying on Angular proxy) or wire dev CORS so the API works without the Angular proxy too?
	- Mason- I definitely want ng-serve to continue working. What do you recommend here?
5. **Lockout duration / threshold** — is 10 failures / 15-min window / 30-min lockout per email acceptable, or do you want different numbers? Same question for IP.
	- Mason- Do these seem okay to you? Should we make these configurable via server secrets?
6. **Encryption key storage** — for Phase 8, can we depend on a `KNOTTYYOGA_SECRET_KEY` env var, or is there a key-management story (KMS, HSM) you want to plug into instead?
	- Mason- What do you recommend? Is there a better solution for letting AWS manage this?
7. **Removing the "Mason"/"Tyler" hardcode (3.5)** — am I OK to delete this outright assuming the seed-data path mints at least one admin? If you currently bootstrap admin only via that hardcode, I need to add a seed-data admin first.
	- Mason- I did this during bootstrapping to get things working but now we build those accounts and give them the right permissions so this is no longer needed. Feel free to delete.
8. **Generic CRUD redaction (3.8)** — is the column-level redact map acceptable, or do you want sensitive columns moved to entirely separate tables (e.g., `people_credentials`) so they can never accidentally be selected?
	- Mason- What do you recommend?
9. **`config_secrets` admin UI (8.2)** — is it acceptable to drop the generic table editor for secrets in favor of a dedicated UI that masks values, or do you need to keep the generic editor for now?
	- Mason- I'm fine with moving this to a dedicated UI. I just needed something to get up and running.
10. **SameSite policy** — your current code uses `Lax`. Are you open to `Strict` for `session_token` (UX cost: cross-site links into the app log the user out)? `Lax` is fine; just confirming.
	- Mason- Honestly, I was just getting things working. I n
11. **Rate limiting persistence** — `login_attempts` in PostgreSQL is simple and correct but adds write load. Are you OK with that, or do you want an in-memory rate limiter keyed off IP/email for the hot path with the DB used only for permanent lockouts?
12. **`/api/me` GET (3.4)** — switching from POST to GET technically changes the public API contract. Confirm you want this; otherwise leave it as POST and just exempt it from CSRF for being read-only.
13. **`change_purchase_recipient` / `gift_permissions` ownership** — please confirm whether these endpoints already check ownership; if you remember, save me the grep.
14. **Phase ordering** — would you prefer to interleave phases (e.g., do Phase 7 security headers very early since they're zero-risk) or strictly sequential as written? The plan is bottom-up by layering, but some phases are independent.