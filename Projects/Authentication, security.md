---
fileClass: Project
Category: Implementation
Status: Active
Author: Mason Bendixen
Reviewers: 
Date: 9/11/2025
Version: 0.1
tags: 
---
# Overview
We need authentication and role based security. This is very fundamental.

# Background Research

> [!summary]
> - **Passwords:** Argon2id on the server (libsodium). Passwords are sent in **cleartext inside TLS** during register/login (standard).
> - **Sessions:** **Stateful** cookie `sid` → DB `sessions` lookup. Do **not** rotate per request; use sliding expiration.
> - **Keep me signed in:** device-bound, rotating **remember-me** cookie `rmt`.
> - **CSRF:** SameSite cookies + **double-submit** header (`X-CSRF-Token`) on unsafe methods.
> - **CORS:** exact origin + `Access-Control-Allow-Credentials: true` if SPA and API are different origins.
> - **Concurrency:** timeouts don’t kill sessions; parallel requests are OK; rotate remember tokens atomically.

---

## 1) Workflows (step-by-step)

### 1.1 First-time sign-up (email + password + verify)
**Client → Server**  
`POST /auth/register { email, password }` over HTTPS.  
- Password is plaintext inside TLS (don’t pre-hash in the browser).

**Server**
1. Validate; Argon2id hash → store in `users.password_hash`.
2. Create verification token or code → store **hash(token)** + expiry in `email_verifications`.
3. Email verification link and/or code.

**Verify**
- `POST /auth/verify { user_id, token }` or `{ email, code }`.
- On success:
  - Set `email_verified_at = now()`.
  - Create **session** and set cookies:
    - `sid=SID_OPAQUE` with secure attributes (see section 4).
    - `csrft=CSRF_TOKEN` (non-HttpOnly) for CSRF header.
    - Optional `rmt=TOKEN_ID.SECRET` for remember-me.

---

### 1.2 Login (later sign-ins)
**Client → Server**  
`POST /auth/login { email, password, remember?: boolean }` over HTTPS.  
- Password is sent only at explicit login.

**Server**
- Verify Argon2id → create `sessions` row → set `sid` + `csrft`.
- If `remember` → create `device_tokens` row and set `rmt`.

---

### 1.3 Auto-login (keep me signed in)
**Client on app start**
1. `GET /me` with credentials.
2. If `401` and `rmt` present → `POST /auth/remember`.

**Server**
- Validate `rmt` (compare **hash(secret)**; check expiry/revocation).
- **Rotate** `rmt`, mint new `sid`, return user.
- No password is sent.

---

### 1.4 Logout
`POST /auth/logout` → revoke current session; clear `sid`.
Optionally also revoke this device’s `rmt` (or “log out everywhere”).

---

## 2) Rotation, timeouts, concurrency

- **Rotate `sid`** on:
  - Login
  - Privilege changes (change email/password, enable 2FA)
  - Optional timed refresh (e.g., every 6–12h) via a refresh endpoint
- **Do not rotate per request.** Use **sliding expiration** (update `last_seen_at` when stale, e.g., >5m).
- **Timeouts:** a timed-out request does not invalidate the session. Use `Idempotency-Key` for retriable POSTs.
- **Parallel requests:** multiple concurrent calls with the same `sid` are fine. Throttle `last_seen_at` writes.
- **Remember-token races:** rotate in a transaction; call `/auth/remember` from a single bootstrap path; optionally allow a short grace window for the old `sid`.

---

## 3) Database (Postgres)

```sql
create extension if not exists pgcrypto;
create extension if not exists citext;

-- Users
create table users (
  id                uuid primary key default gen_random_uuid(),
  email             citext unique not null,
  password_hash     text not null,              -- libsodium Argon2id encoded string
  email_verified_at timestamptz,
  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now()
);

-- Email verification (store only a hash of the token/code)
create table email_verifications (
  user_id      uuid primary key references users(id) on delete cascade,
  token_hash   bytea not null,
  expires_at   timestamptz not null,
  attempts     int not null default 0
);

-- Stateful sessions (short-lived)
create table sessions (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references users(id) on delete cascade,
  created_at   timestamptz not null default now(),
  last_seen_at timestamptz not null default now(),
  expires_at   timestamptz not null,           -- e.g., now() + interval '8 hours'
  revoked      boolean not null default false
);

-- Device-bound remember-me tokens (rotating)
-- Cookie holds TOKEN_ID.SECRET; DB stores only a hash(SECRET)
create table device_tokens (
  token_id     uuid primary key default gen_random_uuid(),
  user_id      uuid not null references users(id) on delete cascade,
  secret_hash  bytea not null,                  -- Argon2id over SECRET (or HMAC + Argon2id)
  created_at   timestamptz not null default now(),
  last_used_at timestamptz,
  expires_at   timestamptz not null,            -- e.g., now() + interval '30 days'
  revoked      boolean not null default false
);

-- Optional: login throttling
create table login_attempts (
  email        citext not null,
  ip           inet not null,
  attempted_at timestamptz not null default now()
);
create index on login_attempts (email);
create index on login_attempts (ip);
```

---

## 4) HTTP details

### 4.1 Cookies

- `sid` (session cookie, short-lived)  
  Attributes: `HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=28800` (8h)  
  Purpose: identifies server-side session (row in `sessions`).

- `rmt` (remember-me, long-lived, rotating)  
  Attributes: `HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=2592000` (30d)  
  Value format: `TOKEN_ID.SECRET` (opaque random). DB stores only **hash(SECRET)**.

- `csrft` (CSRF helper, non-HttpOnly)  
  Attributes: `SameSite=Lax; Path=/; Max-Age=28800`  
  Client reads this and echoes it in header `X-CSRF-Token` on unsafe methods.

> [!tip]
> If SPA and API are same-site, you can consider `SameSite=Strict` for `sid`/`rmt` if UX allows.

---

### 4.2 CSRF (double-submit pattern)

- On login/session creation, set random `csrft` cookie (non-HttpOnly).
- Angular interceptor reads `csrft` and sets `X-CSRF-Token: CSRF_TOKEN` on `POST/PUT/PATCH/DELETE`.
- Server verifies:
  1) `X-CSRF-Token` equals the `csrft` cookie, and
  2) `Origin` (and/or `Referer`) header matches your allowed origin(s).
- Pair with `SameSite=Lax` cookies and HTTPS-only.

---

### 4.3 CORS (only if SPA and API are different origins)

Add these headers on API responses:

```
Access-Control-Allow-Origin: https://YOUR-SPA-ORIGIN
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: Content-Type, X-CSRF-Token
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Vary: Origin
```

Preflight (`OPTIONS`) should return status `204` with the same headers.

> [!warning]
> Do not use `*` for `Access-Control-Allow-Origin` when `Access-Control-Allow-Credentials: true` is set. Use the exact SPA origin.

---

### 4.4 Security headers (production)

- `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`
- `Content-Security-Policy: default-src 'self'; connect-src 'self' https://YOUR-API-ORIGIN;` (tailor to your app)
- `Referrer-Policy: strict-origin-when-cross-origin`
- `X-Content-Type-Options: nosniff`
- `Permissions-Policy: camera=(), microphone=(), geolocation=()` (as needed)

---

## 5) Server (Crow/C++) key snippets

> These are sketches to drop into your codebase; wire them to your DB and routing.

### 5.1 libsodium (Argon2id)

```cpp
#include <sodium.h>
#include <stdexcept>
#include <string>

std::string hash_password(const std::string& pwd) {
  char out[crypto_pwhash_STRBYTES];
  if (crypto_pwhash_str(out, pwd.c_str(), pwd.size(),
        crypto_pwhash_OPSLIMIT_INTERACTIVE,
        crypto_pwhash_MEMLIMIT_INTERACTIVE) != 0) {
    throw std::runtime_error("pwhash failed");
  }
  return std::string(out);
}

bool verify_password(const std::string& hash, const std::string& pwd) {
  return crypto_pwhash_str_verify(hash.c_str(), pwd.c_str(), pwd.size()) == 0;
}
```

### 5.2 Secure random + URL-safe encoding (sketch)

```cpp
#include <sodium.h>
#include <string>

std::string random_bytes(size_t n) {
  std::string s(n, '\0');
  randombytes_buf(s.data(), n);
  return s;
}

// Implement or import a helper for URL-safe base64 encoding.
std::string base64url_encode(const std::string& raw);
```

### 5.3 Set-Cookie helper

```cpp
#include "crow.h"
#include <string>

void set_cookie(crow::response& res, const std::string& name, const std::string& val, const std::string& attrs) {
  res.add_header("Set-Cookie", name + "=" + val + "; " + attrs);
}
```

### 5.4 CORS preflight and headers (pattern)

```text
Access-Control-Allow-Origin: https://YOUR-SPA-ORIGIN
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: Content-Type, X-CSRF-Token
Access-Control-Allow-Methods: GET,POST,PUT,PATCH,DELETE,OPTIONS
Vary: Origin
```

### 5.5 CSRF double-submit check (sketch)

```text
Parse Cookie header to get csrft; compare to X-CSRF-Token.
Also verify Origin/Referer for unsafe methods.
bool csrf_ok(request) { /* implementation */ }
```

### 5.6 Login handler outline (sets sid, csrft, optional rmt)

```text
POST /auth/login
- Parse JSON { email, password, remember }
- Verify password (Argon2id)
- Insert session (expires_at = now + 8h), generate SID_OPAQUE and CSRF_TOKEN
- Set-Cookie: sid=SID_OPAQUE; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=28800
- Set-Cookie: csrft=CSRF_TOKEN; SameSite=Lax; Path=/; Max-Age=28800
- If remember:
    - Create device_tokens row with SECRET hash
    - Set-Cookie: rmt=TOKEN_ID.SECRET; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=2592000
- Return 200 with {"ok": true}
```

---

## 6) Angular (TypeScript) pieces

### 6.1 Interceptor (credentials + CSRF header)

```text
- Read 'csrft' cookie in the browser
- For POST/PUT/PATCH/DELETE, set header: X-CSRF-Token: CSRF_TOKEN
- Always send requests with { withCredentials: true }
```

### 6.2 Auth service + bootstrap flow

```text
login(email, password, remember?)
me()
remember()
logout()

App init sequence:
1) me()
2) if 401 → remember()
3) me() again (or proceed unauthenticated)
```

---

## 7) What’s on the wire (reference)

### 7.1 Login

```text
POST /auth/login
Content-Type: application/json

{ "email": "user@example.com", "password": "PASSWORD", "remember": true }

HTTP/1.1 200 OK
Set-Cookie: sid=SID_OPAQUE; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=28800
Set-Cookie: rmt=TOKEN_ID.SECRET; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=2592000
Set-Cookie: csrft=CSRF_TOKEN; SameSite=Lax; Path=/
Content-Type: application/json

{"ok": true}
```

### 7.2 Authenticated unsafe request

```text
POST /api/items
X-CSRF-Token: CSRF_TOKEN
Cookie: sid=SID_OPAQUE

{ ...payload... }
```

### 7.3 Auto-login via remember

```text
POST /auth/remember
Cookie: rmt=TOKEN_ID.SECRET

HTTP/1.1 200 OK
Set-Cookie: sid=NEW_SID_OPAQUE; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=28800
Set-Cookie: rmt=NEW_TOKEN_ID.NEW_SECRET; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=2592000
Content-Type: application/json

{"ok": true}
```

---

## 8) Implementation checklist

- [ ] Add libsodium; implement Argon2id hash/verify and secure RNG
- [ ] Create tables: `users`, `email_verifications`, `sessions`, `device_tokens` (+ optional `login_attempts`)
- [ ] Implement endpoints: `/auth/register`, `/auth/verify`, `/auth/login`, `/me`, `/auth/remember`, `/auth/logout`
- [ ] Set cookies: `sid` (HttpOnly/Secure/SameSite=Lax), `csrft` (non-HttpOnly), `rmt` (HttpOnly, long-lived)
- [ ] Angular interceptor: `X-CSRF-Token` + `withCredentials: true`
- [ ] App bootstrap: `GET /me` → if 401 then `/auth/remember` → `GET /me`
- [ ] CORS config (if cross-origin) + preflight handling
- [ ] Security headers (HSTS, CSP, etc.); login rate-limit; `Idempotency-Key` for critical POSTs

---

## 9) Libraries to use

- **Server (C++/Crow):** libsodium (Argon2id + RNG), libpqxx (Postgres), a small base64url helper
- **Client (Angular):** built-in HttpClient; optional zxcvbn for password strength meter UX

---

> [!tip]
> - Add TOTP 2FA for step-up actions.
> - Add Passkeys (WebAuthn) to reduce password prompts on returning devices.
> - Use JWT only if you need cross-service/third-party integration; otherwise prefer stateful cookie sessions.

# Understanding concepts
- Sessions
	- These are short lived
	- Table has id, user_id (foreign key into users), created at timestamp, last_seen at timestamp, expires at timestamp, and a bool for revoke
	- What is the session cookie?
	- This is apparently set as a cookie with sid=SID_OPAQUE
	- I asked ChatGPT
	> Can you explain the session id to me? I don't see anyting regarding a SID being stored in the database. All we have is the id, user, created, last seen, expires at, and a revoke. What is the session cookie? What is SID_OPAQUE? Is that value stored on the server? How is it validated?
	- The answer is that the UUID id is the SID. It is a value that expires after a certain interval like eight hours. The client just keeps the value and sends it back with each request.
	- On every request, we look up the value in the table to make sure it hasn't expired or been revoked (which would cause us to respond with 401). If we haven't updated the last seen within a certain interval, like five minutes, queue a work item to update the last seen async.
	- We rotate the session ID on interesting events like login or privilege changing things like login, email/password change, turning on two factor auth, or logout
- Device Tokens (device bound remember me token RMT)
	- These are long lived
	- Table has token id (random UUID), user id foreign key to users table, secret hash Argon2id over SECRET, created at timestamp, last used at timestamp, expires at timestamp, and revoked. The expires at is long like 30 days.
	- Value format is TOKEN_ID.SECRET (DB only stores hash of secret)
	- Where does the secret come from? Is this the client or server? What is the value?
	- I asked ChatGPT
	> Can you explain the device token / remember me token? When is this added to the system? On first login? What is the SECRET? Does it originate on the client or the server? What is the value? So this can be used in lieu of login and will cause a new session to be created?
	- This is created on login when the user clicks remember me. It is generated on the server and sent as a cookie to the browser but only the hash is stored on the server. This is a high entropy random string of 32 bytes (256 bits). Not derived from any user info.
	- This is a relatively long lived cookie and can be used by the client to generate sessions without the need to login.
	- This is important and blocks replay attacks. Rotate this token every time that it is used to prevent replay attacks. When creating the new token, update the last used at. On logout, invalidate all device tokens for that user.
	```c++
	// Pseudocode: POST /auth/remember
auto rmt = get_cookie(req, "rmt");            // "tokenId.secret"
auto [token_id, secret] = split_dot(rmt);

auto t = db.device_tokens.find(token_id);
if (!t || t->revoked || now() >= t->expires_at) return 401;

if (!argon2id_verify(t->secret_hash, secret)) return 401;

// Rotate + mint session atomically
db.tx([&]{
  t->revoked = true; db.update(t);

  auto new_token_id = uuid4();
  auto new_secret   = random_b64url(32);
  auto new_hash     = argon2id(new_secret);
  db.insert_device_token(user_id, new_token_id, new_hash, now()+30d);

  auto sid = create_session(user_id, now()+8h);

  set_cookie(res, "sid", sid, "HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=28800");
  set_cookie(res, "rmt", new_token_id + "." + new_secret,
             "HttpOnly; Secure; SameSite=Lax; Path=/auth/remember; Max-Age=2592000");
});
return 200;
	```
- csrft
	- I don't see entries for this in the database
	- Asking ChatGPT
	> I don't know what csrft is. I don't see an entry for this in the table. Can you explain it?
	- This is a server side token generated and sent in a non-HttpOnly cookie (ie. one that script can read) and then the client sends it back as an HTTP header and a non-HTTP cookie. The server makes sure these match. Cross site scripting attacks can't read this cookie so they can't pull this off.
```c++
bool csrf_ok(const crow::request& req) {
  std::string hdr = req.get_header_value("X-CSRF-Token");
  std::string cookies = req.get_header_value("Cookie");
  std::string cookie = parse_cookie_value(cookies, "csrft"); // implement robustly
  if (hdr.empty() || cookie.empty() || hdr != cookie) return false;

  // Also verify Origin/Referer for POST/PUT/PATCH/DELETE:
  // std::string origin = req.get_header_value("Origin");
  // if (!allowed_origin(origin)) return false;

  return true;
}
```
# Stages
- Check in database tables / schema
	- Done
- Update users table with timestamps
	- Done
- Find library to send email and incorporate it with helper
- Integrate libsodium (Argon2id) with helpers
	- [[libsodium]]
	- Done
- Find a  base64url_encode library
	- See libsodium
	- Done
- Find a good library to generate random text
	- See libsodium
	- Done
- Create a table to store secrets (like email app password) and then a way to set those secrets eventually.
	- Done
- Register workflow (server)
	- Endpoint for register (first name, last name, email, password)
	- RegisterUser
		- Check for duplicates
		- Add to table
		- Send confirmation email with link and code
	- Done
- Confirmation workflow (server)
	- Endpoint for confirmation with email
		- [[mailio]]
	- Validates table entry and within time limit and add validation entry to database
	- Redirect to login
	- Done
- Login workflow (server)
	- Endpoint for login taking email / password / remember me
	- Generate session and a RMT if they said remember me true
	- Respond with correct header and cookies
	- Done
- Remember workflow (server)
	- Validate that cookie matches hash in database. Create new device token and session and send both in response.
	- Done
- Change email, password, and name (server)
	- Validate authenticated, update database, invalidate session and device token
	- Should we auto redo the device token and session or make them login again?
- Logout (server)
	- Invalidate all device tokens for this user and kill sessions
	- Done
- API to fetch information about current user (can't just give open access to users table)
	- Done
- Design role based security
	- Done
- Register workflow (client side)
	- Create page to do client login
	- Direct to page to check email
- Login workflow (client side)
	- email / password with remember me
	- Show user name and maybe picture if we enable photo support
- Update UI to show pages based on permissions
	- Hide dashboard for non admin users
- Remember me workflow (client side)
	- Call remember me URL and then function like login workflow if successful
- Change email, password, and name (client side)
- Logout (client side)
- Implement csrft
- Implement login throttling

# Implementation
## What I'm working on 9/12
- Check in database tables / schema
	- Created branch auth_1_database
	- Plan
		- Add types: timestamp, timestamptz
		- To ColumnInfo, add after nullable a default that defaults to nothing and doesn't output.
		- Update GetSqlType()
		- Add bool IsDefault(string& defaultValue) to ColumnInfo
		- In db_and_table_operations, check for IsDefault()
		- To database_info.h DatabaseInfo, add AddColumnNotNullableWithDefault and add constextprs for kDatabaseInfoDefaultZero, kDatabaseInfoDefaultFalse, kDatabaseInfoDefaultNow, kDatabaseInfoDefaultGenRandomUuid
		- Update People with timestamps
		- Update Classes to have an integer ID
		- Add email_verifications table but give it it's own primary key
			- Should token_hash be binary or base64 encoded?
		- Add sessions
			- Give it a unique ID
		- Add device_tokens
			- Give it a unique ID
		- Add to CreateTables() create_database.cpp
	- Implementation
		- DatabaseCommon.h - add timestamp, timestamptz
		- In column_info.h
			- Added default to constructor
			- Added default_ member
			- Added IsDefault
			- Added to ==
		- In column_info.cpp
			- Added to constructor
			- Added to SqlTypeFromDatabaseType
		- In db_and_table_operations.cpp
			- Added to GenerateCreateTableSql
		- In database_info.h
			- Added AddColumnNotNullableWithDefault
			- Added kDatabaseInfoDefaultZero
			- Added kDatabaseInfoDefaultFalse
			- Added kDatabaseInfoDefaultNow
			- Added kDatabaseInfoDefaultGenRandomUuid
		- In database_info.cpp
			- Added DatabaseInfoImpl::AddColumnNotNullableWithDefault
			- Added DatabaseInfo::AddColumnNotNullableWithDefault
		- In people.h/.cpp
			- Added kPeopleEmailVerifiedAt, kPeopleCreatedAt. and kPeopleUpdatedAt
			- Added corresponding AddColumnNotNullableWithDefault calls
		- in classes.h
			- This already has an integer ID. Not sure why the client doesn't
		- In email_verifications.h/cpp
			- Added kEmailVerificationsTable
			- Added kEmailVerificationsId, kEmailVerificationsPersonId, kEmailVerificationsTokenHash, kEmailVerificationsExpiresAt, kEmailVerificationsAttempts
		- In sessions.h/cpp
			- Added kSessionsTable
			- Added kSessionsId, kSessionsUuid, kSessionsPersonId, kSessionsCreatedAt, kSessionsLastSeenAt, kSessionsExpiresAt, kSessionsRevoked
		- In device_tokens.h/cpp
			- Added kDeviceTokensTable
			- Added kDeviceTokensId, kDeviceTokensUuid, kDeviceTokensPersonId, kDeviceTokensSecretHash, kDeviceTokensCreatedAt, kDeviceTokensLastUsedAt, kDeviceTokensExpiresAt, kDeviceTokensRevoked
	- Completed

What I'm working on 9/23
- Find library to send email and incorporate it with helper
	- [[mailio]]
	- To send email from gmail, you need to use OAth 2.0 or an app password. These are the tradeoffs:
		- Oath2
			- Granular, revocable, short-lived tokens; safer for teams and production.
			- Works where app passwords are blocked.
			- Requires OAuth client setup, token storage, and refresh logic.
		- App Password
			- Easiest to get running for a personal account.
			- Broad access, long-lived secret; weaker operational security.
			- Often disallowed in organizations.
		- It looks like GMail is trying to move away from App Passwords but they are currently supported so that is probably the better solution. 
			- Oath2 involved logging in and then getting a short lived access token that usually has a lifetime of around an hour and then you use that to obtain a refresh_token that is long lived and stored securely. This would involve a flow logging into gmail with the actual password, getting the access token, using that to get a refresh token, and then storing that in the database as a secret.
	- Need to have an email address and an app password
	- Created: knottyyogaandspa@gmail.com
		- Q8tXf2!zL0rV9bGk
		- Have to turn on two factor auth
			- Settings / Security / 2-Step Vefification
		- Searched for app password
			- Name chosen: website
			- App password:
				- ctoj sngx uenn lbdz
	- The plan
		- Create a MailHelper class
			- Takes server, port, password
			- Add a method to send text and get that working with a dummy unit test to my gmail
			- Add support for HTML
			- Maybe add support for HTML with images
		- Implementation
			- Created branch auth_2_mailio
			- Got mailio integrated, building, and tests passing
			- I was using masonbendixen@gmail.com instead of knottyyogaandspa@gmail.com so I was getting: Mail sender rejection.
			- Switched to the right email and things work. I can use either port 465 / LOGIN or 587 / START_TLS.
			- ChatGPT says to prefer port 465

What I'm working on 9/29
- Integrate libsodium (Argon2id) with helpers
	- [[libsodium]]
	- Current version is 1.0.20
	- Current version on Conan is libsodium/1.0.20
	- Access via: libsodium::libsodium
	- First phase is just getting the stuff into Conan
		- Created branch auth_3_libsodium_conan
		- Completed with rebuild
	- Second phase is AuthHelper
		- Methods for HashPassword, VerifyPassword, Base64Encode, RandomText
		- Created branch: auth_4_auth_helper
		- Making unit tests with this Copilot prompt:
		> Can you write unit tests for AuthHelper. Can you have one HashPasswordBasic that calls HashPassword to generate a hash and then verifies it with VerifyPassword. Can you have another one called HashPasswordInvalid that hashes a password and then it uses VerifyPassword with another password and make sure it fails. Can you make a test RandomBytesBasic that calls RandomBytes and verifies the result is the requested number of bytes. Can you do another Base64Encode that passes in a blob and verifies that it gets base64 encoded to the correct value?
		- Tests passing

What I'm working on 9/30
- Create a refactor of the codebase
	- Split sql_util
		- table_helpers - admin\_\*, allowed_tables.\*, 
		- schema - column\_info.\*, database\_info.\*, foreign_key_manager.\*, table\_info.\*
		- database_access - database\_helper.\*, database_crud_helper\*, database\_common.h, database\_metadata.\*, database\_util.\*, db\_and\_table\_operations\*
		- json - database\_rest\_helper\*
	- Remove crud, database_json, sql_schemas, user_features
	- Created branch: auth_5_cleanup
		- Completed
- Create a table to store secrets (like email app password) and then a way to set those secrets eventually.
	- Table: config_secrets
		- id / name / value
	- Have default values
	- ConfigSecrets class that fetches things from the table and sets things
	- Created branch: auth_6_config_secrets
	- Completed

What I'm working on 10/1
- Register workflow (server)
	- Endpoint for login (first name, last name, email, password)
	- RegisterUser
		- Check for duplicates
		- Add to table
		- Send confirmation email with link and code
	- The plan
		- Phase 1
			- Put the table_helpers in a TableHelper namespace
		- Phase 2
			- Make SecretsHelper interface and then test and non test versions of this
		- Phase 3
			- Make MailHelper interface and then test and non test version of this
		- Phase 4
			- Move auth from under util to top level and make a namespace for it
		- Phase 5
			- Create People class under table_helpers with basic CRUD level support including IsPerson and UpdatePerson that modifies updated_at
		- Phase 6
			- Create Person class that wraps the people table
				- struct PersonInfo(email, first, last, pass_hash)
				- static bool IsPerson(email) method
				- static Person RegisterPerson(info)
					- Sends email
				- static Person LookupPerson(email)/(id)
				- static VerifyEmail(info)
				- bool VerifyPassword(password)
				- bool Update(email, info)
				- static void DeletePerson(email)/(id)
	- The implementation
		- Phase 1 - Put the table_helpers in a TableHelper namespace
			- Created branch: auth_7_table_helpers
		- Phase 2 - Make SecretsHelper interface and then test and non test versions of this
			- Created branch: auth_8_secrets
			- Used this Copilot prompt to implement the main class:
			> Can you implement the interface SecretsHelper in this file in a class called SecretsHelperImpl that is in an anonymous namespace and delegates the implementation to ConfigSecrets? Can you implement MakeSecretsHelper using this class?
			- Used this Copilot prompt to implement the test class:
			> Can you implement the interface SecretsHelper in this file in a class called SecretsHelperTestImpl that is in an anonymous namespace. It should contain a KeyValueTable from types.h that it uses to implement the methods. Can you implement MakeTestSecretsHelper using this class?
		- Phase 3 - Make MailHelper interface and then test and non test version of this
			- Created branch: auth_9_mail
			- Added secrets for mail server and port
			- Submitted
		- Move on the Phase4!
		- Phase 4 - Move auth from under util to top level and make a namespace for it
			- Created branch: auth_10_auth_refactor
		- Phase 5 - Create People class under table_helpers with basic CRUD level support including IsPerson and UpdatePerson that modifies updated_at
			- Note about escaping SQL
				- I want to be able to pass in now()
				- I'm currently using txn.esc to escape strings
				- This is unfortunately wrong. I should be using txn.quote() for values and txn.quote_name() for column / table names, BUT I should actually just be using exec_params with $1, $2, ... for values.
				- Can pass an array of params to exec_params() like so
				```c++
pqxx::work tx{conn};

std::vector<std::string> tags = {"alpha","beta","gamma"};

pqxx::params ps;           // empty, grow it dynamically
ps.reserve(tags.size());   // optional
ps.append_multi(tags);     // appends each element as its own param

auto r = tx.exec_params(
  "INSERT INTO my_table(tag1, tag2, tag3) VALUES ($1,$2,$3)",
  ps
);
tx.commit();
				```
			- Plan for Updating DB Crud
				- Cleanup phase
					- Change EscSqlString to use txn.quote()
					- Add EscSqlTableName to use txn.quote_name()
					- Add EscSqlColName to use txn.quote_name()
					- Update usages
				- SQL value support
					- Add variants of insert / update that take a list of supported SQL keywords as values
				- Switch to exec_params
					- Build a class ParamHelper that takes a list of parameters, SQL keywords, and outputs replaceable params and then builds the pqxx::params from an output array that it builds.
			- Implementation for Updating DB Crud
				- Cleanup phase
					- Created branch: auth_11_crud_cleanup
				- SQL value support
					- Created branch: auth_12_sql_keywords
				- Switch to exec_params
					- Created branch: auth_13_exec_params
					- Here are the things with parameters
						- LookupRowByValue: $1 - value
						- GetRowsByColumn: $1 - pageSize, $2 - page \* pageSize
							- Remove commented out code
						- GetRow: $1 - value
						- DeleteRow: $1 - value
						- DeleteRow multivalue needs support like add / update
						- Checking in but not complicated params and needs cleanup
					- Created branch: auth_14_exec_params_complex
					- Moving to variable parameters:
						- This affects: AddRowToTable, AddRowToTableFetchPrimaryKey, UpdateRow, DeleteRow
						- Class ExecParamsHelper
							- string AddParam(string value) // String returned is replacement param
							- pqxx::params GetParams() const // Fetches the params to pass to exec_params
						- Need to add test cases for SQL keywords
						- Need to do cleanup
					- Do cleanup
						- We were taking extra params when we were passing params around instead of doing placeholders. We can simplify.
						- Created branch: auth_15_exec_params_cleanup
					- Add tests for the SQL keyword support.
						- I added tests for the SQL but not the actual execution. Want to make sure the now() support actually works.
						- Make an AddPeopleWithTimestampTable()
						```c++
#include "date/date.h"      // https://github.com/HowardHinnant/date
#include <chrono>
#include <sstream>
#include <stdexcept>
#include <string_view>

inline std::int64_t to_epoch_millis(std::string_view s)
{
    using namespace std::chrono;
    std::istringstream is{std::string(s)};
    date::sys_time<milliseconds> tp;

    // Try common formats (Postgres default and ISO 8601)
    // 1) "YYYY-MM-DD HH:MM:SS[.fff][±HH:MM]"
    if (is >> date::parse("%F %T%Ez", tp)) return tp.time_since_epoch().count();

    // Reset and try 2) "YYYY-MM-DDTHH:MM:SS[.fff]Z"
    is.clear(); is.str(std::string(s));
    if (is >> date::parse("%FT%TZ", tp))   return tp.time_since_epoch().count();

    // Reset and try 3) "YYYY-MM-DDTHH:MM:SS[.fff][±HH:MM]"
    is.clear(); is.str(std::string(s));
    if (is >> date::parse("%FT%T%Ez", tp)) return tp.time_since_epoch().count();

    throw std::runtime_error("Unrecognized timestamp format: " + std::string(s));
}
						```
						- Here is sample code
						- [https://conan.io/center/recipes/date?version=3.0.4](https://conan.io/center/recipes/date?version=3.0.4)
						- date/3.0.4
						- Cmake target name: date::date
					- Add date library support
						- Created branch: auth_16_date
					- Add test cases for SQL keywords
						- Created branch: auth_17_sql_keywords
						- The format Postgres uses for timestamps wasn't supported by my parsing library so I had ChatGPT write a function to parse the strings. Tests are now passing.

What I'm working on 10/8
- Phase 5 - Create People class under table_helpers with basic CRUD level support including IsPerson and UpdatePerson that modifies updated_at
	- Created branch: auth_18_people
	- Copilot query to help implement this file:
	> Using db_schema/people.h and other cpp files in this directory for guidance, can you attempt to implement the CRUD methods in this class?
	- Copilot query for generating tests:
	> Using db_schema/people.h, people.h, and the other *_test.cpp files in this directory as a guide, can you implement a test for every public method in the class People? Can you name the simple confirmation of functionality {MethodName}Basic but also add a separate test for each edge / error case right below each method's Basic test names {MethodName}{EdgeCaseScenario}? For things that throw an exception, please do the scenario in a try catch and verify the exception text. If you are unsure how to test something but know there is a case, can you stub out the test and put a comment about what needs to be tested and a TODO comment for me so I can do it on my pwn?
	- This got close but it was using an older revision of the file for some reason instead of the actual version. Fixed to a degree with this prompt:
	> You generated tests based on an old version of people.h. In particular, there is no PersonInfo. Please redo the tests to use the class People as the people.h currently exists.
	- It put (void) in front of all the calls to void methods. Fixed with:
	> You put (void) in front of all calls to a void method. Can you remove these?
	- It messed up the method, VerifyPersonEmailBasic. Trying to fix with this prompt:
	> You made a mistake with VerifyPersonEmailBasic. The third parameter that is passed to VerifyPersonEmail is the created at that was automatically created by the SQL method now() when AddPerson was called. Please use GetPersonById or Email to look up the person just added and fetch the created at value to pass here. Save the old created at and updated at values from the first GetPersonBy* call and then look the data up again and make sure that created at did not change but that updated at is now later and that verified at was null before and is now set to a value later than created at sql_util/database_access/database_crud_helpers_test.cpp in the AddRowToTableSqlKeywords and the use of DateTimeUtil::StringToEpochMillis for how to do these timestamp comparisons.
	- It didn't test the updated at for the update tests. Fix with this Copilot prompt:
	> Can you use a similar approach for UpdatePersonBasic and UpdatePasswordBasic to snapshot updated at at CreatePerson time and then verify that the value changed and is later after the update? Can you also modify AddPersonBasic to verify that created at and updated at exist and are the same value and that verified at is null?
	- There is an issue that RunInTransaction swallows errors. My test case for duplicate insertions swallows the error. This is a systemic issue though. Put a todo and a work item to follow up on this.
	- Need to add more error test cases for VerifyPersonEmail. Here is the Copilot prompt to do so:
	> I need you to add some more test cases below VerifyPersonEmailNonexistentId fir VerifyPersonEmail. Please call them VerifyPersonEmailEmailMismatch, VerifyPersonEmailCreatedAtMismatch, and VerifyPersonEmailEmailVerifiedAlreadySet. For each case, simulate the correct error and make sure a runtime error with the right exception text is thrown. Please look in the implemention of VerifyPersonEmail for the exception text.
	- Done
- Make a change to People table to have verify email take a time period in milliseconds that the verification must happen within
	- Hardest part honestly is generating appropriate test data
	- The timestamp string starts with a four digit year. Easiest thing is to generate something a year earlier and do one time window too short and another two years
	- Add a secret kAuthVerifyEmailTimeLimitInMillis that defaults to one week
	- Implementation
		- Created branch: auth_19_verify_window
		- Completed
- Phase 6
	- Create Person class that wraps the people table
		- struct PersonInfo(email, first, last, pass_hash)
		- static bool IsPerson(email) method
		- static Person RegisterPerson(info)
			- Sends email
		- static Person LookupPerson(email)/(id)
		- static VerifyEmail(info)
		- bool VerifyPassword(password)
		- bool Update(email, info)
		- static void DeletePerson(email)/(id)
	- Plan
		- Use these things:
			- class People
			- SecretsHelperPtr
			- AuthHelper - Hash/VerifyPassword
	- Implementation
		- Created branch: auth_20_person
		- Copilot query to flesh out this class:
		> In this file, please stub out and implement the functions declared in person.h to the best of your ability. For things that you can't figure out how to implement, please just leave a stub and a comment. Please create people_ with the passed in DatabaseHelper. Please lookup kAuthVerifyEmailTimeLimitInMillis using the SecretHelper and also create a constant like kTwoYearsInMillis in people_test.cpp that you use if the lookup of the secret fails to pass into VerifyPersonEmail for timeWindowToVerifyInMillis. For the passwordHash parameter to AddPerson, please use AuthHelper::HashPassword. For VerifyPassword, please lookup the person via people_ and then pass the password hash retrieved and the passed in password to AuthHelper::VerifyPassword. Delegate most funcitonality to people_.
		- This worked surprisingly well. 
		- Copilot query to do the testing:
		> In this file, can you generate unit tests for the PersonHelper class? For each method, please create a separate tests that verifies a basic, positive use case and name each test {method_name}Basic. Please use sql_util/table_helpers/people_test.cpp for an inspiration. You can also use the People class to help with testing. Please use MakeTestMailHelper() in util/mail/mail_helper_test_util.h and MakeTestSecretsHelper() in secrets/secrets_helper_test_util.h to "inject" dependent test objects and help validate the results. Please create a constant test HTML template like kMailTemplate but with all the replacement parameters filled in to verify the right email sent. For PreliminaryRegisterPerson, have the basic case test that the email is sent and then create a PreliminaryRegisterPersonNoEmail test with the email flag set to false. For any additional tests you add, add them after the Basic test for that function so all the tests for a given function are grouped together. For IsPerson() add a test for a person that does not exist and add another for one that the person has been created with PreliminaryRegisterPerson but not completed with VerifyPersonEmail. This class handles users registering with a website but they are not full users until they get the email link sent by PreliminaryRegisterPerson and then call back via the link that gets routed to a call to VerifyPersonEmail. For LookupPerson (by it and email), VerifyPassword, UpdateInfo, and UpdatePassword, add a case for someone registered with PreliminaryRegisterPerson but not verified and call each test {method_name}NotVerified. For all of those cases, also do another test case for the user not existing at all and call it {method_name}NoPerson. For LookupPerson (by it and email), VerifyPassword, UpdateInfo, and UpdatePassword, use CreateFullyValidatedUser to add a user for the normal, positive test cases. For VerifyPassword, add a test case VerifyPasswordWrongPassword that passes in a different password than that created with CreateFullyValidatedUser. For UpdatePassword, make sure that the test case creates a user with CreateFullyValidatedUser, update the password and verify that VerifyPassword succeeds for the new password and fails for the original one.
		- Completed

What I'm working on 10/20
- Modify WebApp
	- Plan
		- Have endpoints/web_app.h WebApp class take a SecretHelperPtr and MailHelperPtr and add access methods for these.
		- Modify src/main.cpp to create the real versions of these
		- Modify endpoints/endpoint_test_helper to create test versions of these, pass them into the webapp constructor but have derived class accessors to the real class.
	- Implementation
		- Created branch: auth_21_web_app_update
		- Need to make a MakeMailHelper that takes a SecreteHelperPtr and calls the other version with parameters.
		- get_row_test.cpp is making it's own WebApp instead of using EndPointTestHelper
- Register workflow (server)
	- Endpoint for register (first name, last name, email, password)
	- Plan
		- Add endpoint /api/register/{first_name}/{last_name}/{email}/{password}
		- Call PersonHelper::PreliminaryRegisterPerson
		- Verify email gets set
		- Copilot query from delete_item.cpp
		> Create three files in this directory modelled after this file (delete_item.cpp), delete_item.cpp, and delete_item_test.cpp. Name them register.h, register.cpp, and register_test.cpp. This is adding another endpoint like this endpoint (/api/delete_item) but the new endpoint will be /api/register. It will take four string parameters: first_name, last_name, email, and password. In register.cpp, Have there be functions HandlePost and a SetupRouting class just like delete_item.cpp but have Register instead of DeleteItem. There is no need to call IsTableAllowed(). Implement Register by creating a PersonHelper object (auth/person.h) and passing in the SecretsHelperPtr and MailHelperPtr from WebApp's GetSecretsHelper() and GetMailHelper() and then calling PreliminaryRegisterPerson with sendEmail set to true. Create a test file having the TEST methods use RegisterTest and have a RegisterBasic test method that validates basic functionality. Use delete_item_test.cpp and auth/person_test.cpp for guidance and inspiration.
	- Implementation
		- Created branch: auth_22_register
		- Chatgpt pretty much wrote this entirely
- Confirmation workflow (server)
	- Endpoint for confirmation with email
		- [[mailio]]
	- Validates table entry and within time limit and add validation entry to database
	- Redirect to login
	- Plan
		- Endpoint is /api/confirm/{id}/{email}/{created_at}
		- Call PersonHelper::VerifyPersonEmail
		- Copilot query from register.cpp:
		> Create three files in this directory modelled after this file (register.cpp), register.h, and register_test.cpp. Name them verify.h, verify.cpp, and verify_test.cpp. This is adding another endpoint like this endpoint (/api/register) the the new endpoint will be /api/verify. It will take three string parameters: id, email, created_at. In verify.cpp, have there be functions HandlePost and a SetupRouting class like register.cpp but have them call Verify instead of Register. Implement Verify by creating a PersonHelper object (auth/person.h) and passing in the SecretsHelperPtr and MailHelperPtr from WebApp's GetSecretsHelper() and GetMailHelper() and then calling VerifyPersonEmail. Create a test file having the TEST methods use VerifyTest and have a VerifyBasic test method that validated basic functionality. Use register_test.cpp and auth/person_test.cpp for guidance and inspiration.
		> Can you add a test VerifyInvalidId that checks to see that the right exception is thrown when passing an id that is not an integer?
	- Implementation
		- Created branch: auth_23_verify
		- Created the files with ChatGPT but the created at needs to be URL encoded in the current form which is messing things up. I think the better solution is to convert to millis. I need to change this in a bunch of places:
			- PreliminaryRegisterPerson
			- VerifyPersonEmail
		- I will check this in with a todo and then modify the code to use integers for timestamps
- Convert to storing timestamps as microseconds
	- Plan for now_us()
		- Switch to storing types as BIGINT
		- Create this stored proc and use it anywhere we currently use now():
		```sql
CREATE OR REPLACE FUNCTION now_us()
RETURNS bigint
LANGUAGE sql
AS $$
  SELECT (extract(epoch FROM clock_timestamp()) * 1000000)::bigint
$$;
		```
		- Make a stored_procedures under sql_util
		- Make a create_stored_procedures.h/cpp with a CreateStoredProceduresBeforeTables(DatabaseHelper databaseHelper)
		- Add a now_us.h, now_us.cpp, now_us_test.cpp to create the stored procedure with a CreateNowUs(DatabaseHelper databaseHelper)
		- Call this from CreateStoredProceduresBeforeTables
		- Also add a CreateStoredProceduresAfterTables
		- Add both of these to CreateDatabase
	- Implementation for now_us()
		- Created branch: auth_24_now_us
		- Created CreateNowUs sproc
		- Copilot prompt to write tests:
		> In this file, can you write unit tests for CreateNowUs? Can you use sql_util\database_access\database_crud_helpers_test.cpp as an example? I'd like a TEST method using CreateNowUsTest with the test method name CreateNowUsBasic and call CreateNowUs() and then make two separate calls to select now_us() and verify that they are greater than zero and less than an hour apart. It returns microseconds since the epoch.
	- Where do we use timestamps?
		- TIMESTAMPTZ
			- device_tokens.cpp - Not used yet
			- email_verifications.cpp (Note- we have a table for email verification... to create a secret and hash) - not used yet
			- people.cpp - 
			- sessions.cpp - Not used yet
		- MakePeopleTable()
			- A lot of tests create and use this function- factor out to a new function called MakeTestPeopleTable() and put in test utility code
			- Places we need to make a change:
				- auth/person_test.cpp
				- endpoints/register_test.cpp
				- endpoints/verify_test.cpp
				- table_helpers/people_test.cpp
		- Places that use: StringToEpochMillis
			- auth/person_test.cpp
			- sql_util/table_helpers/people.cpp
			- sql_util/table_helpers/people_test.cpp
		- Things that need explicit updating:
			- sql_util/table_helpers/people.cpp
				- All the places that call StringToEpochMillis() or use now()
			- auth/person.cpp
				- The code that pulls the created at out
			- endpoints/verify_test.cpp
				- Need to adjust to the correct created at
			- table_helpers/people.cpp
				- Uses DateTimeUtil::StringToEpochMillis
			- table_helpers/people_test.cpp
				- Uses DateTimeUtil::StringToEpochMillis
				- Extensive time usage
	- The plan
		- Update all the Schema files
			- device_tokens.cpp - Not used yet
			- email_verifications.cpp (Note- we have a table for email verification... to create a secret and hash) - not used yet
			- people.cpp - 
			- sessions.cpp - Not used yet
		- Add MakeTestPeopleTable() method
			- On test/src/util/database_test_helper.h
		- Update all the test files using MakePeopleTable() to use this method
		- Look at all the places the use now() / StringToEpochMillis for explicit updating
	- The implementation
		- Created branch: auth_25_epoch
		- Added DB_TYPE_BIGINT to DatabaseTypes and ColumnInfo::SqlTypeFromDatabaseType
		- Added TestDatabaseUtil::MakeTestPeopleTable()
		- Added TestDatabaseUtil::AddPerson()
		- Added test/src/util/json_test_util.h/cpp
		- Need to compare DataResults ignoring columns. Here is a Copilot query to do this:
		> In util/types.h, there is a struct DataResults that has two fields. One is a string array with a sorted list of column names. The second is an array of rows of values where each entry in the array has a row of values corresponding to data items matching the columnName by index in the first column. I want to take two of these in and a list of columns to ignore and compare that all the values are equal minus those specified in columnsToRemove. Notice that in util/types.h, there is an IndexOOfColunm method that takes a column name and returns the index of the column or -1. First off, walk the columns of the first DataResults and add each column that is not removed to a hashset. Walk the second and make sure each column not removed is in the hashset of the first while incrementing a counter. If any are not present or you finish and the counter for the second doesn't match the count of the first hashset, they are different. After that, walk each row of each DataResults and walk throught the columns comparing items to make sure they match minus the removed columns. Implement this code in the body of CompareDataResultsMinusColumnsHelper. in test_helper_test.cpp, please implement the four test functions to test the case described by the test name. Please note that you are implementing CompareDataResultsMinusColumnsHelper, but that is a helper function so the function being tested is the outer function CompareDataResultsMinusColumns.
		- Need to compare DataResults encoded in JSON ignoring columns. Here is a Copilot query to do this:
		> I need you to do basically the same thing as last time, comparing two DataResults structs minus columns; however, this time the DataResults are encoded in JSON. The Crow micro web server C++ framework has C++ JSON wrapper object implemented in crow/json.h and the type is crow::json::wvalue. You can see the data getting encoded in sql_util/json/database_rest_helper.cpp in the function JsonFromDataResults. util/json_util.h/cpp has a lot of functions I wrote for dealing with these JSON objects that you can use for context and should try to use. Please implement CompareJsonDataResultsMinusColumnsHelper and then also implement the four stubbed out test functions in json_test_util_test.cpp based on the name of the test. Like last time, this is a helper function used by the outer function CompareJsonDataResultsMinusColumns so please test in terms of that function.
		- Need a simpler function that just does objects. Here is the Copilot prompt:
		> I need to do something similar but quite a bit simpler. Instead of a DataResults object encoded in JSON, this is just simple JSON objects with just key value pairs (named data basically) and the fieldsToRemove refers to keys in key value pairs to not use for comparison. Can you do something similar to the last two examples and implement the body of CompareJsonObjectMinusColumnsHelper. In json_test_util_test.cpp, there are four test methods CompareJsonObjectMinusColumnsHelper* that I'd like you to implement as well. Just note that CompareJsonObjectMinusColumnsHelper is a helper so CompareJsonObjectMinusColumns is the function you will actually be testing.
		- Only two tests are still failing
			- [  FAILED  ] GetRowsByColumnTest.GetRowsByColumnBasic
			- [  FAILED  ] GetTableRowsTest.GetTableRowsBasic
			- Got these working
		- Checked this bitch in!!!

What I'm working on 10/28
- Need to modify this workflow to use the email verifications table
- Need to go back and fix up the verify workflow now that we switched to dates
- Modify workflow to use email verifications table
	- Create a certain number of random bytes (64?)
	- Generate a hash of this and store this in email verifications
	- Base64 encode this and send this in the email
	- On the verification side, Base64 decode this, generate the hash, and compare this to the email verifications
	- Add an admin alerts table with an id, date, and notification
	- In the same transaction, take the email and base64 encoded data and generates the hash, looks up the email verifications to see if the hashes match. If they match, remove the entry from email verifications (if within the time window) and add the verified timestamp to the people table. If failed, increase the attempts count. If the count goes over a threshold, log to admin alerts.
	- Work stages
		- Stage 1- Add admin alerts table and table helpers class to wrap this
		- Stage 2- Refactor database_crud_helpers.h with copies of all methods in a NoTransation namespace that take a DatabaseHelper and pqxx::work& trans as arguments and the normal versions just call these.
		- Stage 3- Add EmailVerifications table helper with operations on the class including the operation to do the register / verify type work and accessing the database outside of a transaction but taking a transaction.
		- Stage 4- Comment out person.h and the endpoints to get the lower layers working.
		- Stage 5- Modify people.h to do the work outside of a transaction and to just update the email verified at
		- Stage 6- Modify person.h to create the transactions and do the combined work
		- Stage 7- Modify the endpoints to use the new functionality
	- Stage 1- Add admin alerts table and table helpers class to wrap this
		- Created branch: auth_27_admin_alerts
		- Created a bunch of functions in database_util.h to implement
			- Implement these
			- Add tests
				- Copilot prompt to add tests:
				> I added the functions: RunSqlStatementReturningDataResults, RunSqlStatementReturningOneRow, RunSqlStatementReturningOneValue, RunSqlStatement. Can you add tests to database_util_test.cpp at the end that start with DatabaseUtilTest and then have the function name with Basic appended for the test name. Please use MakePeopleTable() like the other tests in this file. For validating DataResults, note that you can use ElementsAre for the items in the sorted columns and individual rows in the datatable. You can also specify an ORDER BY in the SQL query you generate to force the order of the rows. Also note that these are helper functions so the real functions to test are the variadic template functions with the same name so please pass the parameters in directly as params to the call instead of manually creating an ExecParamsHelper. Also note that this is PostGres SQL for generating SQL statements with replaceable parameters. For RunSqlStatementReturningDataResults and RunSqlStatementReturningOneRow, please generate a test named function name with NoResults appended to the name that does not produce any results.
			- Rewrite database_crud_helpers to use these and get rid of ExecParamsHelper
				- Done!
			- Write tests for get_admin_alerts_in_window_test
				- Need to write test for get_admin_alerts_in_window. Here is a Copilot query to do so:
				> Please write a test for CreateGetAdminAlertsInWindow in the file get_admin_alerts_in_window_test.cpp. Please use now_us_test.cpp as a guide. Please be aware of sql_util/database_access/database_util.h and sql_util/database_access/database_crud_helpers.h as well as sql_util/table_helpers/admin_alerts.h for using AdminAlerts to do AddAdminAlert(). Please put the tests under CreateGetAdminAlertsInWindowTest and name the basic functionality test CreateGetAdminAlertsInWindowBasic that just adds a couple of tests and then checks that everything added in the last ten minutes shows up. Then make another test named CreateGetAdminAlertsInWindowOutOfRange that adds two alerts but then do DbCrud::UpdateRow() to make the second alert a week in the past and query for the last ten minutes again and make sure only the first entry shows up. You might need to make one call initially before the update to get the ids so you can do the update.
				- Done :)
			- Finish admin_alerts and add testing
				- Functionality is implemented. Add testing. Copilot query for this:
				> Please create tests for the class AdminAlerts in the file you create in this folder: admin_alerts_test.cpp. Please update CMakeLists.txt appropriately. For each public method in AdminAlerts, please test basic functionality in the test section AdminAlertsTest and name each separate test function name with Basic appended. Please use both other _test.cpp files in this directory as examples and also look at sql_util/stored_procedures/get_admin_alerts_in_window_test.cpp for examples. Please note that you will need to install that stored procedure with CreateGetAdminAlertsInWindow.
				- Done!
			- TODO: DatabaseCrudHelpers- we might be passing / returning arrays of column names that are no longer needed. Do a check on this.
				- GetRowsByColumn no longer uses columnNames

What I'm working on 10/31
- Stage 2- Refactor database_crud_helpers.h with copies of all methods in a NoTransation namespace that take a DatabaseHelper and pqxx::work& trans as arguments and the normal versions just call these.
	- Created branch: auth_28_no_transaction
	- This is a bigger change than I expected. Basically, I should be having a single transaction at root level. I also should just reuse or do a single create database and reuse a single connection to this database for all tests. I can also create tables unlogged to speed things up and then abort the transaction at the end of each test. Here is some example code:
	```c++
#include <pqxx/pqxx>
#include <chrono>
#include <string>

template <typename TestBody>
void run_isolated_test(pqxx::connection& conn, TestBody body) {
    pqxx::work tx{conn};

    // 1) Cheap durability settings for this test only
    tx.exec0("SET LOCAL synchronous_commit = OFF");
    tx.exec0("SET LOCAL default_transaction_isolation = 'read committed'"); // cheapest

    // 2) Create a unique schema for the test
    auto now = std::chrono::steady_clock::now().time_since_epoch().count();
    std::string schema = "t_" + std::to_string(now); // or a UUID
    tx.exec0("CREATE SCHEMA " + tx.quote_name(schema));

    // 3) Route unqualified names into our schema (then pg_temp, then public)
    tx.exec0("SET LOCAL search_path = " + tx.quote_name(schema) + ", pg_temp, public");

    // 4) Create objects you need (prefer TEMP or UNLOGGED)
    // TEMP => zero WAL, auto-dropped; UNLOGGED => minimal WAL.
    tx.exec0(R"SQL(
        CREATE UNLOGGED TABLE accounts(
            id BIGSERIAL PRIMARY KEY,
            name TEXT NOT NULL
        );
    )SQL");

    // If you can: use TEMP for scratch tables (even cheaper).
    tx.exec0("CREATE TEMP TABLE scratch (k int, v text) ON COMMIT DROP");

    // 5) Create any functions/procedures needed for the test
    // (DDL is transactional; they disappear on ROLLBACK.)
    tx.exec0(R"SQL(
        CREATE FUNCTION add_account(n text) RETURNS bigint AS $$
        BEGIN
          INSERT INTO accounts(name) VALUES (n) RETURNING id INTO STRICT RESULT;
        END;
        $$ LANGUAGE plpgsql;
    )SQL");

    // 6) Run the actual tested code against this tx.
    // Your production code takes a pqxx::work&; call it here:
    body(tx);  // e.g., your API that accepts pqxx::work&

    // 7) Throw away everything (DDL + DML)
    tx.abort(); // or just let an exception roll it back
}
	```
	- Plan
		- Create a GlobalDatabaseTestSupport singleton object
			- This is created in main and initialized
			- This will drop and recreate the database
			- This will create the persistent connection to the database that is shared by all tests
			- Done
		- Create a Transaction interface class that Takes an ExecParams class and does the various Exec operations. This allows for different transaction template types to be used between prod and test.
		- Done
		- Make DatabaseHelper hold a shared_ptr to a DatabaseHelperBase and have DatabaseHelperTest and DatabaseHelperProd that derive from that
			- Have Database helper have an bool IsTest() const method
			- Use the shared connection from GlobalDatabaseTestSupport for the test version 
			- Have an RunInTransaction that passes in a Transaction&
				- For prod, this will use the expensive transaction and commit
				- For test, this will use a cheap transaction and abort
			- Done
		- Modify database_util to use this
			- Done
		- Modify database_crud_helpers to use this
			- Done
		- Move up the layers
			- Start with table_helper in particular admin alerts and make sure the stored procedure tests are firing
			- Table Helpers are done
			- Stored Procedures are done
			- DatabaseRESTHelper is done
			- Need to do: enpoints, auth, and test main
				- auth down
		- Issue- Create a transaction provider interface that provides a transaction. Have the WebApp take this as a parameter and have a default one for production that creates a transaction and have a utility class that creates one of these in the handler and then commits it. For the endpoint test cases, use TestDatabaseUtil::RunInTransaction and then pass the Transaction to EndpointTestHelper. Create a dummy transaction provider that does nothing for the handler in the test case as far as creating and committing / aborting the transaction but DOES provide the transaction that is the same transaction as in the test code.
			- Note I can should get rid of the transaction description to RunInTransaction
			- Create interface TransactionProvider that provides a RunInTransaction() method
			- Create TestTransactionProvider that implements the interface but takes a pointer to a Transaction in the constructor that it uses and provided by EntpointTestHelper to the constructor
			- Create ProductionTransactionProvider that implements the interface and creates a transaction that it implements. Wire this into the normal app setup. Have it take a DatabaseHelper in the constructor to generate the transaction like below:
			- Here is the old way we created a transaction:
			```c++
			void DatabaseHelper::RunInTransaction(
    std::string_view description, DatabaseFunc databaseFunc) {
    try {
        std::string transactionDescription =
            "Running transaction: " + std::string(description);
        pqxx::work trans(*connection_, transactionDescription);
        databaseFunc(trans);
        trans.commit();
    }
    catch (const std::exception& e) {
        LogError() << "Exception: " << description << " failed with: "
            << e.what() << std::endl;
    }
}
			```
			- WebApp also takes a DatabaseHelper as a constructor parameter so it would be 
	- Got it code complete... now we compile!
	- All tests are passing
	- Things to do:
		- Get the meta database test back online
			- Follow up on that post checkin
		- The table verify stuff
			- Follow up on that post checkin
		- Get the register / verify tests and content completed
			- This was based on the timestamp stuff. Do in a separate checkin
		- Look at any other todos
		- Code review

# What I'm working on 11/4
- Get the meta database tests back online
	- Created branch: auth_29_ddl
	- Completed
- Need to modify this workflow to use the email verifications table
- Need to go back and fix up the verify workflow now that we switched to dates
- Modify workflow to use email verifications table
	- Create a certain number of random bytes (64?)
	- Generate a hash of this and store this in email verifications
	- Base64 encode this and send this in the email
	- On the verification side, Base64 decode this, generate the hash, and compare this to the email verifications
	- Add an admin alerts table with an id, date, and notification
	- In the same transaction, take the email and base64 encoded data and generates the hash, looks up the email verifications to see if the hashes match. If they match, remove the entry from email verifications (if within the time window) and add the verified timestamp to the people table. If failed, increase the attempts count. If the count goes over a threshold, log to admin alerts.
	- Work stages
		- Stage 1- Add admin alerts table and table helpers class to wrap this
			- Done
		- Stage 2- Refactor database_crud_helpers.h with copies of all methods in a NoTransation namespace that take a DatabaseHelper and pqxx::work& trans as arguments and the normal versions just call these.
			- Done
		- Stage 3- Add EmailVerifications table helper with operations on the class including the operation to do the register / verify type work and accessing the database outside of a transaction but taking a transaction.
		- Stage 4- Comment out person.h and the endpoints to get the lower layers working.
		- Stage 5- Modify people.h to do the work outside of a transaction and to just update the email verified at
		- Stage 6- Modify person.h to create the transactions and do the combined work
		- Stage 7- Modify the endpoints to use the new functionality

# What I'm working on 11/5
- Stage 3 - Add EmailVerifications table helper with operations on the class including the operation to do the register / verify type work and accessing the database outside of a transaction but taking a transaction.
	- Created branch: auth_30_email
	- The plan:
		- All these take a transaction
		- kEmailVerificationExpirationWindowInMicros - secret
		- kEmailVerificationAttemptLimit - secret
		- AddEmailVerificationByEmail(email, tokenHash)
		- AddEmailVerificationById(personId, tokenHash)
		- IdList ListExpiredEmailVerifications()
		- DeleteEmailVerification(id)
		- IdList GetEmailVerificationsOverNumberOfAttempts(int numberOfAttempts)
		- KeyValueTable GetEmailVerificationInfo(id)
		- KeyValueTable GetEmailVerificationInfoByEmail(email)
		- bool DoEmailVerification(email, tokenHash)
	- Implementation
		- Attempt to describe this in Copilot:
		> Please create a class in the directory called EmailVerifications. Place the class declaration in the file email_verifications.h, the member implementation in email_verifications.cpp, and the tests in email_verifications_test.cpp. Please update CMakeLists.txt accordingly. Please use people.h/people.cpp/people_test.cpp as templates for this. Please use the same namespace and indent in the same way as in that file. Please name variables the same way as named in that file (ie. base instances of a class based on the full name of the class with a lower case first letter instead of doing the shortening that you like to do). This is kind of a wrapper for the table defined in db_schema/email_verifications.h/.cpp. It also uses the secrets defined in secrets/secret_keys.h kEmailVerificationExpirationWindowInMicros and kEmailVerificationAttemptLimit. This table is used as part of user sign up for the system and basically tracks an email being sent to the user with a random set of bytes and validates the response click in the email for the given email for the user and that the hash of this data matches the hash stored in the table. kEmailVerificationExpirationWindowInMicros is a time window in microseconds in which this verification must be validated before it is no longer allowed to do so. kEmailVerificationAttemptLimit tracks the maximum of account activation attempts to allow before blocking activation. And logging an alert. This is basically a CRUD wrapper class for the table with a hint of business logic mixed in. Please give it two create methods AddEmailVerificationByEmail(email, tokenHash) and AddEmailVerificationById(personId, tokenHash). The first looks up the email using the People CRUD database class and then calls the second with the person id. Both methods should create an entry in the table and allow the time to be filled in automatically. StringArray ListExpiredEmailVerifications() is another method that should return a list of email verifications whose time is past the current time allowed. DeleteEmailVerification(id) takes an id for an email verification and removes it from the table. StringArray GetEmailVerificationsOverNumberOfAttempts(int numberOfAttempts) returns a list of ids of email verifications whose number of failed attempts has exceeded kEmailVerificationAttemptLimit. KeyValueTable GetEmailVerificationInfo(id) returns a KeyValueTable with the various column names as keys and the various column values as the matching values in the table for the given id. GetEmailVerificationInfoByEmail(email) is the same thing but looks up the id of the persion by email using People and then calls the other method. bool DoEmailVerification(email, tokenHash) looks up that the email verification for the given email (use the People class to turn email into person id) and validates that it has not exceeded kEmailVerificationAttemptLimit, the the time is within kEmailVerificationExpirationWindowInMicros, and that the given hash code matches. If that is true, it deletes the entry and returns true. Otherwise, it increases the failed attempt count, logs to the alerts table if the number exceeds kEmailVerificationAttemptLimit, and returns false. I didn't list it here for brevity, but please make all these methods take a Transaction& transaction as the first parameter. In the test file, please create a CreateEmailVerificationTable function that creates the people table, alerts table, and email verifications tables. Please also create a CreatedFilledSecrets function that creates a test secrets provider with default values passed as parameters for boundary case testing but that default to sensible values for normal tests. Please have each method throw an exception if an error case happens. For each method, create a EmailVerifactionsTest test function called method name with Basic appended and test normal functionality. Underneath this Basic test method for each method add edge case tests named method name with error case description appended.
		- Done

What I'm working on 11/7
- Work stages
	- Stage 1- Add admin alerts table and table helpers class to wrap this
		- Done
	- Stage 2- Refactor database_crud_helpers.h with copies of all methods in a NoTransation namespace that take a DatabaseHelper and pqxx::work& trans as arguments and the normal versions just call these.
		- Done
	- Stage 3- Add EmailVerifications table helper with operations on the class including the operation to do the register / verify type work and accessing the database outside of a transaction but taking a transaction.
		- Done
	- Stage 4- Comment out person.h and the endpoints to get the lower layers working.
		- Not sure I need to do this
	- Stage 5- Modify people.h to do the work outside of a transaction and to just update the email verified at
		- Done
	- Stage 6- Modify person.h to create the transactions and do the combined work
	- Stage 7- Modify the endpoints to use the new functionality
- Need to update person.h to use email verifications, the transaction stuff is already done
	- The plan
		- In GenerateMailBody, don't pass in the id or created at, take a base64 encoded secret hash and use that and the email for the URL
		- In PersonHelper::PreliminaryRegisterPerson
			- Create 64 bytes of random data
			- Hash this
			- Base64 encode the hash
			- Add to email verifications with base64 encoded hash
			- Call GenerateMailBody with the hash and not the id or created at
		- In PersonHelper::VerifyPersonEmail
			- Take in email and the base64 encoded secret, unbase64 encode this, hash it, and the base64 encode that and then call email verifications to verify and, if successful, then update the people table.
	- Implementation
		- Created branch: auth_31_email_secret
		- I don't have access to the secret put in the email. Split the test into two fragments and do a prefix and suffix match.
		- The URL is base64 encoded but that is not URL safe. Need to URL encode / decode
		- Completed

# What I'm working on 11/12
- Stages for previous change is done. Complete cleanup items and then get back to larger tasks.
- Make function that takes a lambda and calls for all secrets and values
- Factor the mail related stuff out to a helper
- Make function that takes a lambda and calls for all secrets and values
	- Created branch: auth_32_secret_cleanup
- Factor the mail related stuff out to a helper
	- Created branch: auth_33_mail_cleanup
	- Copilot prompt for the test
	> Using person_test.cpp in this folder as an example, please generate a test file for person_verify_mail.h/cpp. Please place the tests inside an anonumous namespace inside Auth::Mail. Please have the test functions under PersonVerifyMailTest. For each of the three function in the header, please create a test called function name with Basic appended. For each test, outside of the test, create a string_view test content with the expected result and call the function with sensible values and the test content having static text with those values filled in based on the templates in the cpp file.
	- Completed
- What to work on next:
	- Login workflow (server)
		- Endpoint for login taking email / password / remember me
		- Generate session and a RMT if they said remember me true
		- Respond with correct header and cookies

# What I'm working on 11/13
- Work that needs to be done
	- Phase 1 - Create secrets for:
		- Duration that a session lives
		- Duration that a device token (rmt token) is good for
		- Duration since updating last seen for session to queue a work item to upate it
		- Done
	- Phase 2 - Convert hashes for tables to strings to make dealing with them and testing easier
		- Done
	- Phase 3 - Table helper for sessions
		- Done
	- Phase 4 - Table helper for device tokens
		- Done
	- Phase 5 - Add CreateSession() method to PersonHelper that adds the appropriate entries to the table for a given user
		- Done
	- Phase 6 - Create async work queue
		- Done
	- Phase 7 - Add SessionUsed() method to PersonHelper that checks secret and queues the async update of the last seen if that threshold is exceeded
		- Done
	- Phase 8 - Create EndpointAuthHelper class that takes the resp/req objects and wire this into all the existing endpoints
		- Done
	- Phase 9 - Login workflow with username, password, and remember bool that creates a session
		- Done
	- Phase 10 - Modify EndpointAuthHelper to validate the user based on session (and eventually RMT) (and returns 401 otherwise)
		- Done
	- Phase 11 - Create a Session class that has the current user id in it and is present in EndpointAuthHelper and available for every endpoint
		- Done
	- Phase 12 - Modify login to be a post and have a remember me that is used to create the device token if set
		- Done
	- Phase 13 - Add CreateDeviceToken() method to PersonHelper that adds the appropriate entries to the table for a given user and returns the secret
	- Phase 14 - Add a Remember() endpoint that takes the device token, validates it, and that it is not expired and generates a new device token and session and responds to the user
- Phase 1 - Create secrets for:
	- Created branch: auth_34_auth_secrets
	- Submitted
- Phase 2 - Convert hashes for tables to strings to make dealing with them and testing easier
	- Created branch: auth_35_hash_to_string
	- Submitted
- Phase 3 - Table helper for sessions
	- The plan
		- int AddSession(int personId, int64_t microsUntilExpires)
		- KeyValueTable LookupSessionById(int id)
		- KeyValueTableArray LookupSessionByPerson(int personId)
		- void UpdateLastSeen(int id)
		- void RemoveSessionsForUser(int personId)
		- void RemoveSession(int sessionId)
	- The implementation:
		- Created branch: auth_36_sessions
		- Copilot prompt:
		> Please create a class in this folder called Sessions. Please place the declaration in sessions.h, the implementation in sessions.cpp, and the tests in sessions_test.cpp. Please make sure to update CMakeLists.txt. Please base this on the People class in people.h/cpp and put it in the same namespace. This is a CRUD wrapper for the Postgres table in db_schema/sessions.h/cpp. I want a method int AddSession(int personId, int64_t microsUntilExpires) that allows the default column values for most things to be filled in but populates kSessionsExpiresAt with now() + microsUntilExpires. Make sure to add any expression you use to do this to the allowedSqlKeywords. This function returns the id of the session that is returned by AddRowToTableFetchIntPrimaryKey(). Have a method KeyValueTable LookupSessionById(int id) that does exactly what the name says and another similar one KeyValueTableArray LookupSessionByPerson(int personId). Note that there can be multiple sessions for the same user if the user logs in from different machines. Have another method void UpdateLastSeen(int id) that updates kSessionsLastSeenAt with now() in microseconds. Have a method void RemoveSessionsForUser(int personId) that removes any sessions from the table that match the given username and another void RemoveSession(int sessionId) that removes the entry, if present, that matches the given id. For the test file, put all the tests in the same namespace as the implementation but then in an anonymous namespace. Please make a test for each public method names method name with Basic appended that exercises a normal positive case flow. Underneath the Basic test for each method, add tests with any checks for edge cases and exceptions. For AddSession(), add a test AddSessionUserNotPresent to make sure the foreign key missing Postgres error is thrown. For both LookupSessionById and LookupSessionByPerson add NotFound cases. Same for UpdateLastSeen. All of these should throw an exception. RemoveSessionsForUser should not throw an exception for a non existent person. Add a NotFound test case that just makes sure things return normally. RemoveSession should not throw an exception for a missing session. Please add a test case for this. For each of the removal an update cases, please have the tests validate things at the database level.
		- Completed
- Phase 4 - Table helper for device tokens
	- The plan
		- int AddDeviceToken(int personId, string_view secretHash, int64_t microsUntilExpires)
		- KeyValueTable LookupDeviceTokenById(int id)
		- KeyValueTableArray LookupDeviceTokensByPerson(int personId)
		- KeyValueTable LookupDeviceTokenBySecretHash(string_view secretHash)
		- bool IsValid(int id)
		- void Revoke(int id)
		- bool UpdateDeviceToken(int id, string_view secretHash, int64_t microsUntilExpires)
		- void RemoveDeviceTokenById(int id)
		- void RemoveDeviceTokensForUser(int personId)
	- The implementation
		- Created branch: auth_37_device_tokens
		- Copilot prompt
		> Please create a class in this folder called DeviceTokens. Please place the declaration in device_tokens.h, the implementation in device_tokens.cpp, and the tests in device_tokens_test.cpp. Please make sure to update CMakeLists.txt. Please base this on the class you just created Sessions in sessions.h/cpp and put it in the same namespace. This is a CRUD wrapper for the Postgres table in db_schema/device_tokens.h/cpp. Add a method int AddDeviceToken(int personId, string_view secretHash, int64_t microsUntilExpires) that uses the default values for most things but populates kDeviceTokensExpiresAt with now() + microsUntilExpires and add this expression to allowedSqlKeywords. Please add these methods:  KeyValueTable LookupDeviceTokenById(int id), KeyValueTableArray LookupDeviceTokensByPerson(int personId), and KeyValueTable LookupDeviceTokenBySecretHash(string_view secretHash) that do pretty much exactly what the method names suggest. Add the method bool IsValid(int id) that verifies the given device token exists and that now() is not after expires at and that it is not revoked. Add void Revoke(int id) that sets kDeviceTokensRevoked to true. Add bool UpdateDeviceToken(int id, string_view secretHash, int64_t microsUntilExpires) that does an IsValid() check and then updates secretHash, kDeviceTokensExpiresAt to now() + microsUntilExpires, sets kDeviceTokensLastUsedAt to now() in micros since epoch (you will need to add expressions used to allowedSqlKeywords). Please note that you need to use row() around any update with a tuple syntax like I modified your changes in sessions.cpp. Add void RemoveDeviceTokenById(int id) that removes the entry for the given id but does not do anything if the id does not exist. Add void RemoveDeviceTokensForUser(int personId) that removes all device tokens for the given user. For the test file, put all the tests in the same namespace as the implementation but then in an anonymous namespace. Please make a test for each public method names method name with Basic appended that exercises a normal positive case flow. Underneath the Basic test for each method, add tests with any checks for edge cases and exceptions. For AddDeviceToken() add a AddDeviceTokenPersonNotFound test for a person id that does not exist to make sure the correct Postgres foriegn key exception gets thrown. For the three Lookup methods, add NotFound variants that validate that an exception is thrown. For IsValid() add test cases for a NotFound, Revoked, and Expired. For Revoke() make sure that IsValid() returns false after. Also, add a NotFound case for Revoke that verifies that it throws an exception. For UpdateDeviceToken, add a NotFound case that throws an exception, an Expired case that verifies that false is returned, and a Revoked case that also returns false. For each of these failed cases, verify that nothing is changed in the database. For the Remove cases, make sure that nothing is thrown and that execution proceeds normally even if the device token or user specified does not exist. For each of the removal an update cases, please have the tests validate things at the database level. Please note the extra tables and stored procedures I set up in CreatePeopleAndSessionsTables and make sure to do the same thing (minus the sessions table) in this test file.
		- Done
# What I'm working on 11/17
- Phase 5 - Add CreateSession() method to PersonHelper that adds the appropriate entries to the table for a given user
	- The Plan
		- Lookup the secret kAuthSessionMaxDuractioninMicros
		- bool CreateSession(string_view email, string& uuidString)
		- Lookup person id from email
		- Call Sessions::AddSession with the person id and kAuthSessionMaxDuractioninMicros value and fetch the id
		- Fetch and return the uuid
		- Also add bool LookupPersonBySession(string_view sessionUuid, int& personId)
	- Implementation
		- Created branch: auth_38_create_session
		- Copilot prompt
		> Can you implement CreateSessionToken and LookupPersonIdBySessionToken as well as the tests for these methods in person_test.cpp? For CreateSessionToken, please lookup the secret kAuthSessionMaxDuractioninMicros, convert that to an integer, lookup the person id from the email, and call the table helper in table_helpers/session.h. After fetching the id, look up the session to get the uuid to return in the output parameter. Besides the Basic positive test case, please add a NotFound test case. Please validate the the appropriate entries are added at the database level. For the timestamps, before making the call, please fetch now() and make sure that the timestamps added are later and that revoked is null or false. For LookupPersonIdBySessionToken, you just need to treat the value passed in as the UUID and lookup the persion id and make sure it has not expired or been revoked. Please add a test case for NotFound as well as a Revoked test case for a session that has been revoked (which should return false) and an Expired test for a session that has expired.
		- Completed
- Phase 6 - Create async work queue
	- The plan
		- Class is names ThreadPool
		- Boost thread pool
			- \#include <boost/asio.hpp>
			- boost::asio::thread_pool pool(8);
			- boost::asio::post(pool, [] {...});
			- pool.join();
		- Have static methods GetInstance() and Shutdown() to make it easier to do testing
		- Have Queue and Join methods
	- Implementation
		- Created branch: auth_39_thread_pool
		- Copilot prompt
		> In this folder, please create three classes thread_pool.h, thread_pool.cpp, and thread_pool_test.cpp. Create a class called ThreadPool. Make the default constructor private with a default implementation and make the copy constructor and assignment operator deleted. Make the destructor default implementation too. Have a static unique_ptr\<ThreadPool> that is the s_instance and static accessors GetInstance() that creates the instance if it does not exist and a static Shutdown() that calls join and the frees the static instance. Have methods Queue that takes a std::function<void()> and a Join method. Internally use the boost::asio::thread_pool with 8 threads. Have queue do post and then have Join() call join on the thread_pool object. Create a simple ThreadPoolTest for QueueBasic() test that queues two threads and makes sure both complete. Have a JoinBasic() that queues multiple workers and then verfifies that calling Join() waits for all the threads to complete. Please put the tests inside an anonymous namespace. Look at types_test.cpp for an example of how to do testing. Please update CMakeLists.txt.
		- Completed

# What I'm working on 11/18
- Phase 7 - Add SessionUsed() method to PersonHelper that checks secret and queues the async update of the last seen if that threshold is exceeded
	- The plan
		- SessionUsed() takes the token and looks it up to see if it exists. If it does, it queues a lambda by value with the session id to update the timestamp. Only issue is that this needs a transaction. 
		- We already have a transaction provider interface. Take this as a parameter to the method and then pass in transaction provider as a parameter that gets routed to the worker item. Can probably pass this by reference as a raw reference.
		- We are looking at the secret kAuthSessionLastSeenUpdateDurationInMicros
	- Implementation
		- Created branch: auth_40_session_last_seen
		- Inside PersonHelper::SessionUsed, Copilot prompt to complete method
		> Can you finish implementing PersonHelper::SessionUsed. Please lookup the information for the given session and compare kSessionsLastSeenAt plus the secret kAuthSessionLastSeenUpdateDurationInMicros to see if now() is later than that. If so, please use util/thread_pool.h to do ThreadPool::GetInstance().Queue() to pass a lambda in that passes a reference to the transaction provider by value as well as the session id by value and then does a RunInTransaction that does Session::UpdateLastSeen.
		- Copilot prompt for tests
		> Can you please implement SessionUsedBasic and SessionUsedNotFound? For SessionUsedBasic, can you use test/src/util/test_transaction_provider.h with the MakeTestTransactionProvider() with the transaction from RunInTransaction to pass in the transaction provider. Make a call to util/thread_pool.h ThreadPool::GetInstance().Shutdown() to do a Join and make the test hermetic. Fetch the SecretHelper and do AddSecret() with kAuthSessionLastSeenUpdateDurationInMicros to set it to a relatively short duration to make it quicker in the test to force an update to the last seen. Validate that the value gets updated by doing call to now() after creating the session and then validating that it gets updated. For the SessionUsedNotFound case, verify that an exception is thrown in the case that the token is not found.
		- Need a test for the case that the window is long and we don't need to do the update
		> Can you implement the test SessionUsedNoUpdate? It will be very similar to SessionUsedBasic but set kAuthSessionLastSeenUpdateDurationInMicros to a long window and verify that kSessionsLastSeenAt does not change.
	- Done
- Phase 8 - Create EndpointAuthHelper class that takes the resp/req objects and wire this into all the existing endpoints
	- I can put these at the start of the CROW_ROUTE lambda \(const crow::request& req, crow::response& res, ...
	    - if crow::response is present, it must be the second parameter
	- Have the constructor take raw references to these
	- Give is Response()/Request() accessors
	- Give it an App() accessor
	- Give it GetDatabaseInfo()
	- Give it GetSecretHelper()
	- Give it GetMailHelper()
	- Give it GetTransactionProvider()
	- Implementation
		- Created branch: auth_41_endpoint_helper
		- Copilot query
		> Please create a class in this folder named EndpointAuthHelper and have the declaration live in endpoint_auth_helper.h, the implementation in endpoint_auth_helper.cpp, and the test file in endpoint_auth_helper_test.cpp. Please update CMakeLists.txt accordingly. Have the constructor take \(WebApp& app, const crow::request& req, crow::response& res\) as parameters. Have a method void Initialize() that takes no parameters and has an empty body for now in the implementation file. Please have a single test TEST(EndpointAuthHelperTest, InitializeBasic) that will, for now, just have an empty body. Have members app_, request_, and response_ that store the references in the constructor and accessors App(), Request(), and Response() that give access to the underlying members. Have accessors GetDatabaseInfo(), GetSecretHelper(), GetMailHelper(), and GetTransactionProvider() that call down to the same methods on the app_. Please put the test in an anonymous namespace.
		- Have the class up and running, now wire into existing endpoints.
		- Got add_item.h/cpp working. Use them to have Copilot do this for all of the other classes. Check them out
		- Need to modify get_row.cpp get_row_test.cpp to use the handle_full
			- We currently call Endpoints::GetRow directly and validate that the json value returned
			- For the others, we do:
				- crow::request req;
				- crow::response resp;
				- req.url = "/api/stuff...";
				- endpointTestHelper.GetWebApp().GetApp().handle_full(req, resp);
				- json_util::WvalueFromText(resp.body)
				- ASSERT_EQ(resp.code, 200);
				- Fixed this so everything is now consistent
		- Here is a Copilot prompt
		> Look at the changes I have made to add_item.h/cpp. I'd like you to make similar changes to all the handler type functions in this folder: add_item_fetch_primary_key, db_schema, delete_item, get_row, get_rows_by_column, get_table_rows, register, update_item, verify. For each .h file, please switch from including web_app.h and to endpoint_auth_helper.h and replace WebApp* webApp with EndpointAuthHelper& endpointAuthHelper. In each cpp file, please add the include and make each HandlePost take a webApp, request, and response like the example. Create an instance of EndpointAuthHelper and initialize it. Instead of returning crow::response objects, switch to assigning like in the example. Make each CROW_ROUTE lambda take both the response and request as the first two parameters (before any other parameters) and pass all the needed parameters to HandlePost. For the actual function being called from inside HandlePost, make the first parameter a EndpointAuthHelper instead of a WebApp pointer. Swith all the calls currently on webApp to the calls to the helper. Please name variables the same way as in the example.
		- Getting two test failures now:
			- AddItemFetchPrimaryKeyTest.AddItemFetchPrimaryKeyBasic
			- GetRowsByColumnTest.GetRowsByColumnBasic
			- Needed to clear() the response object between calls. The old assignment method did this automatically.
		- Done

```C++
#include <crow.h>
#include <crow/middlewares/cors.h>
#include <crow/middlewares/cookie_parser.h>

using App = crow::App<crow::CookieParser, crow::CORSHandler>;

int main() {
    App app;

    // Toggle however you like (env var, compile flag, etc.)
    const bool is_prod = false;

    // --- CORS config ---
    auto& cors = app.get_middleware<crow::CORSHandler>();
    if (is_prod) {
        cors
            .global()
                .origin("https://app.example.com")  // your SPA origin
                .methods("GET"_method, "POST"_method, "OPTIONS"_method)
                .headers("Content-Type", "Authorization")
                .allow_credentials();               // Access-Control-Allow-Credentials: true
    } else {
        // Dev: Angular at http://localhost:4200, API at http://localhost:8080
        cors
            .global()
                .origin("http://localhost:4200")    // frontend dev server
                .methods("GET"_method, "POST"_method, "OPTIONS"_method)
                .headers("Content-Type", "Authorization")
                .allow_credentials();
    }

    // --- Login route: set cookie ---
    CROW_ROUTE(app, "/login").methods(crow::HTTPMethod::Post)
    ([&app, is_prod](const crow::request& req, crow::response& res) {
        // TODO: real authentication and secure random session id
        std::string session_id = "abc123";

        using Cookie      = crow::CookieParser::Cookie;
        using SameSitePol = Cookie::SameSitePolicy;

        auto& ctx = app.get_context<crow::CookieParser>(req);

        // Create/set cookie
        auto& c = ctx.set_cookie("session_id", session_id);
        c.path("/");        // send on all paths
        c.httponly();       // JS can't read it

        if (is_prod) {
            c.domain("example.com");   // covers api.example.com, app.example.com, etc.
            c.secure();                // HTTPS only
            c.same_site(SameSitePol::None);  // cross-site SPA <-> API
        } else {
            // Dev: still cross-site (different ports), but usually on plain HTTP
            // Browsers *prefer* SameSite=None + Secure; localhost is often special-cased.
            c.same_site(SameSitePol::None);
            // no domain() => host-only cookie for the API host
        }

        // Optional: cookie lifetime (in seconds)
        // c.max_age(24 * 60 * 60);

        res.code = 200;
        res.write("Logged in");
        res.end();
    });

    // --- Authenticated route: parse/read cookie ---
    CROW_ROUTE(app, "/me").methods(crow::HTTPMethod::Get)
    ([&app](const crow::request& req, crow::response& res) {
        auto& ctx = app.get_context<crow::CookieParser>(req);

        // Returns empty string if cookie is not present
        std::string session_id = ctx.get_cookie("session_id");

        if (session_id.empty()) {
            res.code = 401;
            res.write("No session cookie");
            res.end();
            return;
        }

        // Here you’d look up the session in your store / verify signature etc.
        res.code = 200;
        res.write("Hello, session " + session_id);
        res.end();
    });

    app.port(8080).multithreaded().run();
}
```
On the client
```typescript
this.http.post('http://localhost:8080/login', body, {
  withCredentials: true,
}).subscribe(...);

this.http.get('http://localhost:8080/me', {
  withCredentials: true,
}).subscribe(...);

```
# What I'm working on 11/20
- Phase 9 - Login workflow with username, password, and remember bool that creates a session
	- Subphases
		- Sub 1 - create secret for if in production mode defaults to false
			- Done
		- Sub 2 - create ServerConfig class that is a singleton that has an InitializeTestMode() method as well as an Initialize(transaction) that reads the secret. This gets initialized in main for regular server and test. Have it also take the app to initialization and do the CORS initialization.
			- Done
		- Sub 3 - Add Login(string_view email, string_view password, bool remember) method to PersonHelper and just validates the credentials
			- Done
		- Sub 4 - Create CookieManager interface with a CookieInfo struct that you can pass with a SetCookie(string_view name, string_view value, const CookieInfo& cookieInfo) and a GetCookies() that returns a std::unordered_map<string, string>. Have a test and non test implemenation that is intialized with a response and a request.
			- Done
		- Sub 5 - Create a Session class with IsLoggedIn(), InitializeFromLogin(transaction, token, CookieManagerPtr), bool InitializeFromFromCookie(transaction, CookieManagerPtr) and methods about user information (person id)
			- Done
		- Sub 6 - Wire the session class into EndpointAuthHelper but not wired into the system
			- Done
		- Sub 7 - Add login endpoint that calls to the method on PersonHelper
			- Done
		- Sub 8 - Wire Session object into Initialize() to read from the cookie manager and do the equivalent of a login
		- Done
- Sub 1 - create secret for if in production mode defaults to false
	- Created branch: auth_42_prod_mode
	- Done
- Sub 2 - create ServerConfig class that is a singleton that has an InitializeTestMode() method as well as an Initialize(transaction) that reads the secret. This gets initialized in main for regular server and test. Have it also take the app to initialization and do the CORS initialization.
	- Created branch: auth_43_server_config
	- Copilot prompt:
	>Create a class called ServerConfig that lives in this folder with the declaration in server_config.h, the implementation in server_config.cpp, and the tests in server_config_test.cpp. Please update CMakeLists.txt accordingly and just add the files, don't do any extra cleanup. Please put the code in the Auth namespace and format things like the other files in this folder. Please give it a bool member prodMode_ and testMode_ that both default to false. Have a default basic contstructor and destructor and delete the copy constructor and assignment operator. Please make the constructor private and have a static private variable s_instance that is a singleton unique_ptr\<ServerConfig>. Have a static GetInstance() method that throws a run time error exception if this value is null. Have a private static GetInstancePrivate() that throws an exception if the s_instance is not set and then does a new ServerConfig and then sets s_instance to this value. Have a static Initialize\(Transaction& transaction, Secrets::SecretHelperPtr secretHelper\) that calls GetInstancePrivate() and then looks up the kServerProductionMode flag secret and uses util/types.h's StringToBool to convert to a bool that sets prodMode_ based on the value and testMode_ to false. Have a static InitializeTestMode() that calls GetInstancePrivate() and then sets testMode_ to true and prodMode_ to false. Have a static Shutdown() method that clears / destroys the static instance. Have accessors IsProdMode() const and IsTestMode() const that return the corresponding member variables. Put the tests (ServerConfigTest) in an anonymous namespace inside Auth. Have a InitializeProdMode test that fills in a secret with kServerProductionMode set to true and then validates that after doing Initialize() IsProdMode() returns true and IsTestMode() returns false. Have a InitializeDevMode test that fills in a secret with kServerProductionMode set to false and then validates that after doing Initialize() IsProdMode() returns false and IsTestMode() returns false. Have a test InitializeTestModeBasic that calls InitializeTestMode() and then validates IsProdMode() returns false and IsTestMode() returns true.
	- Done
- Sub 3 - Add Login(string_view email, string_view password, bool remember) method to PersonHelper and just validates the credentials
	- We don't actually need this method. We already have VerifyPassword and CreateSessionToken
	- Done
- Sub 4 - Create CookieManager interface with a CookieInfo struct that you can pass with a SetCookie(string_view name, string_view value, const CookieInfo& cookieInfo) and a GetCookies() that returns a std::unordered_map<string, string>. Have a test and non test implemenation that is intialized with a response and a request.
	- The plan
		```c++
		namespace crow {
    struct cookie {
        std::string key, value;
        std::string domain;
        std::string path = "/";
        std::string same_site;   // "Strict", "Lax", "None"
        bool secure = false;
        bool http_only = false;
        int max_age = -1;        // seconds, -1 = unset

        cookie(std::string k, std::string v)
            : key(std::move(k)), value(std::move(v)) {}
    };
}
		```
		- Create a struct CookieProperties {
			- std::string domain;
			- std::string path = "/";
			- std::string sameSite;
			- bool secure = false;
			- bool httpOnly = false;
			- int64_t maxAgeInMicros = -1;
		- };
	- The implementation
		- Created branch: auth_44_cookie_manager
		- Copilot prompt
		> Please create a set of files cookie_manager.h, cookie_manager.cpp, cookie_manager_test.cpp, cookie_manager_test_util.h, and cookie_manager_test_util.cpp. Place these files in this folder and put the contents in the Auth namespace. Please update CMakeLists.txt accordingly and just add these files, don't attempt cleanup of the file. In cookie_manager.h, please put this struct:
struct CookieProperties {
std::string domain;
std::string path = "/";
std::string sameSite;
bool secure = false;
bool httpOnly = false;
int64_t maxAgeInMicros = -1;
};
Also put an interface CookieManager with a virtual, default destructor, and the copy constructor and assignment operator set to delete. Give it a pure virtual method SetCookie\(string_view key, string_view value, const CookieProperties& cookiteProperties\) and another pure virtual method std::unordered_map\<std::string, std::string> GetCookies() const; Put a using CookieManagerPtr = std::unique_ptr\<CookieManager> as well as a declaration for CookieManagerPtr MakeCookieManager\(const crow::request& req, crow::response& resp\). In cookie_manager.cpp, please create a class CookieManagerImpl in an anonymous namespace that publicly derives from CookieManager and takes a request and response as constructor parameters that it saves the references to member variables. Implement SetCookie / GetCookies in terms of crow cookies. Please fill in the related crow::cookie fields based on whether each value in CookieProperties is empty. Convert the maxAgeInMicros to seconds or leave it as -1. Implement MakeCookieManager using this class. In cookie_manager_test_util.h, please put a Test namespace inside Auth \(Auth::Test\) and place the class CookieManagerTest the publicly derives from CookieManager in this namespace. Override the methods of CookieManager and then add the method std::unordered_map\<std::string, CookieProperties\> GetCookieProperties(). Give it two members, std::unordered_map\<std::string, std::string> cookies_ and std::unordered_map\<std::string, CookieProperties>. Put a using CookieManagerTestPtr = std::unique_ptr\<CookieManagerTest>; in this file as well as a CookieManagerTestPtr MakeCookieManagerTest\(\). In cookie_manager_test_util.cpp, please implement the methods of CookieManagerTest using the various members as well as implementing MakeCookieManagerTest. In cookie_manager_test.cpp, please add a the tests in an anonymous namespace inside Auth::Test and then put tests under CookieManagerUtilTest and create a test for each public method with Basic appended to the name that creates a CookieManagerTestPtr and then does the various operations and validates basic functionality.
- Done

# What I'm working on 11/21
- PersonHelper cleanup to remove the things from the constructor and pass in as parameters where I can:
	- Created branch: auth_45_person_helper_cleanup
	- secrets_ is used in:
		- PreliminaryRegisterPerson
		- VerifyPersonEmail
		- CreateSessionToken
		- SessionUsed
	- mailHelper_ is used in:
		- PreliminaryRegisterPerson
	- Done
- Sub 5 - Create a Session class with IsLoggedIn(), InitializeFromLogin(transaction, token, CookieManagerPtr), bool InitializeFromFromCookie(transaction, CookieManagerPtr) and methods about user information (person id)
	- Created branch: auth_46_session
	> Create a class called Session in this folder that lives in the files session.h, session.cpp, and session_test.cpp. Put everything in the Auth namespace and format everything like you did for cookie_manager.h/cpp. Update CMakeLists.txt and just add the files, don't do any other cleanup. Make the Session class have a default destructor and make the copy constructor and assignment operator as delete. Make the constructor take a DatabaseHelper databaseHelper and store it in a member named databaseHelper_. Have a member int personId_ that defaults to -1. Have a const method IsLoggedIn() that returns true if personId_ is not -1. Also have a const method int GetPersonId() that returns personId_. Have a method void InitializeFromLogin(Transaction& transaction, SecretsHelperPtr secrets, string_view sessionToken, CookieManagerPtr cookieManager) that creates a PersonHelper and uses LookupPersonIdBySessionToken() to convert sessionToken into a personId that it stores in the member and then calls SetCookie() with the key "session_token" and the sessionToken. When creating the CookieProperties to pass to the call, set the path to "/", httpOnly to true, and sameSite to CookieSameSitePolicy::None. Call ServerConfig::GetInstance().IsProdMode() and if it is prod mode, lookup secrets->LookupSecret\(Secrets::kWebsiteAddress\) for domain, set secure to true, and . For maxAgeInMicros, lookup kAuthSessionMaxDuractioninMicros as a string and then convert it to a int64_t and set maxAgeInMicros. Then call CookieManager::SetCookie. Add another method bool InitializeFromFromCookie(Transaction& transaction, CookieManagerPtr cookieManager). Lookup to see if there is a cookie named "session_token". If there is not, return false. If there is, fetch the value and then make a PersonHelper and call LookupPersonIdBySessionToken. If that fails, return false. Lookup the session in the Sessions TableHelper and make sure it is not Otherwise, store the personId returned and return true. Put the tests in the test file in an anonymous namespace and all the tests under SessionTest. Create a test InitializeFromLoginBasic that does ServerConfig::InitializeTestMode() \(do a call to Shutdown() first\) and then does TestDatabaseUtil MakeTestPeopleTable and then AddPerson with a dummy person and then do PersonHelper CreateSessionToken to populate a valid session for this person. Look at the test files for these classes to make sure the appropriate database tables are created. Use MakeCookieManagerTest(). Test InitializeFromLogin to make sure basic functionality is working and that the CookieProperties has the right values. Do a test InitializeFromLoginNotFound that passes in a token that is garbage and verifies that the correct exception is thrown for a user not being found. Create a test InitializeFromFromCookieBasic that verifies that the user is set and IsLoggedIn returns true for being initialized from a token. Create another test InitializeFromFromCookieNotPresent that returns false when no cookie is present. Then another test InitializeFromFromCookieInvalid that has a cookie but the value is not valid and verify it returns false.
	- Done
- Sub 6 - Wire the session class into EndpointAuthHelper but not wired into the system
	- Created branch: auth_47_wire_session
	- Done

# What I'm working on 11/22
- Sub 7 - Add login endpoint that calls to the method on PersonHelper
	- The plan
		- login is a post that takes a string email, string password, bool remember as a post
		- It calls PersonHelper::VerifyPassword
			- On success it then calls, CreateSessionToken
				- On success it then calls, Session::InitializeFromLogin
		- add_item_fetch_primary_key.cpp is an example of a post
	- The implementation
		- Created branch: auth_48_login
		- Copilot prompt
		> In this folder, please create another HTTP endpoint for my webserver named login. It will live in this folder in these files: login.h, login.cpp, and login_test.cpp. Please lookup other endpoints in this folder for context. For instance, add_item_fetch_primary_key.cpp is an example of an endpoint that does a post. The URL for this endpoint will be "/api/login". The post will have three fields: string email, string password, and bool remember. The helper function called from HandlePost will be void Login\(const crow::request& req, crow::response& resp, const crow::json::rvalue& message\). It will create a PersonHelper object and then call, VerifyPassword on that with the email and password. On success, it will call CreateSessionToken. On success to that call, it will create a Session object and call InitializeFromLogin. On success to that it will lookup the secret kWebsiteAddress and then use the response object to do an HTTP redirect to the value for kWebsiteAddress. On failure to any of these, it will set the response to HTTP access denied. In the test file, place the tests under the same namespace as the implementation code (Endpoints) but also put the tests under the anonymous namespace. All the tests should be under LoginTest. Please add a test LoginBasic that exercises the basic positive case workflow. Look at verify_test.cpp for an example of how to do test for an endpoint and then the tests for auth/person_helper_test.cpp for how to setup and test the results of the VerifyPassword and CreateSessionToken functionality and then auth/session_test.cpp for how to validate that InitializeFromLogin succeeds. Please use the actual files instead of your history of the files since I have hand edited many of the things after your context. Please update CMakeLists.txt but just with the three new files. Please also update web_app.cpp with g_Login = &Endpoints::Login. Please test things at the database level. Please add a test for LoginNotFound for a user that does not exist.
		- Need to wire cookie manager in correctly to the system
		- Done
- Sub 8 - Wire Session object into Initialize() to read from the cookie manager and do the equivalent of a login
	- The plan
		- Well, I already wired cookie manager into EndpointAuthHelper. Now I need to wire in the session object.
		- Session needs a DatabaseHelper. EndpointAuthHelper has a WebApp so it definitely has that.
		- Make it a member EndpointAuthHelper and then try to do Session::InitializeFromFromCookie inside Initialize
	- Implementation
		- Created branch: auth_49_auth_cookie
		- Done

# What I'm working on 11/24
- Phase 13 - Add CreateDeviceToken() method to PersonHelper that adds the appropriate entries to the table for a given user and returns the secret
	- The plan
		- bool CreateDeviceToken\(Transaction& transacion, Secrets::SecretsHelperPtr secrets, std::string_view email, std::string& tokenId, std::string& base64EncodedSecret\)
		- DeviceTokens::AddDeviceToken(Transaction& transaction, int personId, string_view secretHash, int64_t microsUntilExpires)
		- We need to create the secret, hash it, make the call, and return the secret
		- The value we return to the user is UUID.base64EncodedSecret
			- The first value is the the UUID as a string generated by the record insert
			- The second is 32 bytes of random generate binary base64 encoded
			- Generate new values on each rotation
		- We don't deal with cookies at this level so just return the values needed
		- The secret we use is: kAuthDeviceTokenMaxDurationInMicros
		- We lookup the person by email, create a 32 blob, hash it, base64 encode the hash, base64 encode the value, lookup the secret, lookup now_us(), add the secret to that, and add the value to the table helper and then return the values
	- The implementation
		- Created branch: auth_51_device_token
		- Stubbed out method and tests
		- Copilot prompt:
		> Can you implement the stubbed out method CreateDeviceToken? Use CreateSessionToken as a template. Make an AuthHelper and use RandomBytes to generate a blob of 32 bytes. Then create a hash of that with HashBinary. Then base64 encode the hash and base64 encode the blob. Lookup the secret kAuthDeviceTokenMaxDurationInMicros and convert that to an int64_t. Look in SessionUsed for how to make a call to SELECT now_us() to fetch the current database time in microseconds and then add that value to the secret looked up. Use TableHelpers::DeviceTokens and make one and then call AddDeviceToken (lookup the personId from the email) and pass the base64 encoded hash in for secretHash and the added value for microsUntilExpires. Look up the value just inserted with LookupDeviceTokenById and fetch kDeviceTokensUuid. Return the kDeviceTokensUuid via outTokenId and the base64 encoded 32 byte blob in outBase64EncodedSecret. Test the basic functionality in the stubbed out test CreateDeviceTokenBasic. Add tests for CreateDeviceTokenNotFound and CreateDeviceTokenRevoked. Look at LookupPersonIdBySessionTokenNotFound and LookupPersonIdBySessionTokenRevoked for guidance.
		- Done

# What I'm working on 11/25
- Write DataResults accessors
	- Created branch: auth_52_cleanup
	- Copilot prompt:
	> Please implement GetDataResultsValue using IndexOfColumn. Please implement GetDataResultsValueAsInt/64 in terms of GetDataResultsValue and then using the conversion function but let it throw on failure. For GetDataResultsValueAsBool, use GetDataResultsValue and StringToBool.
	- Test query:
	> Please implement the tests starting with GetDataResultsValue. Add relevant tests for the various function referenced.
	- Done

# What I'm working on 11/26
- Change DeviceTokens::UpdateDeviceToken to generate a new UUID
	- Created branch: auth_53_device_token_uuid
	- row()
	- GEN_RANDOM_UUID()
	- Done
- Add a bool PersonHelper::TryLoginWithDeviceToken\(Transaction& transaction, Secrets::SecretsHelperPtr secrets, std::string_view tokenId, base64EncodedSecret, std::string& outTokenId, std::string& outBase64EncodedSecret\)
	- Created branch: auth_54_try_login_device_token
	- Factor out the secret generation stuff and expiration from PersonHelper::CreateDeviceToken
	- Base64decode the secret, generate the hash, lookup by the hash, if it is present, then lookup the UUID and make sure that matches. If that matches, make sure to check DeviceToken::IsValid to test for expiration or revokation. If all of that succeeds, do UpdateDeviceToken and return true
	- Copilot query:
	> Can you implement TryLoginWithDeviceToken? Please use CreateDeviceToken as an example. Inside a try / catch, create a TableHelpers::DeviceTokens object. Create an Auth object. base64decode the base64EncodedSecret. Take that secret and do HashBinary on it. Take that hash and use DeviceTokens::LookupDeviceTokenBySecretHash to fetch the device token. Once you have that, lookup kDeviceTokensUuid and make sure that value matches tokenId. After that, check DeviceTokens::IsValid. If all that succeeds and we have a valid token, call DeviceTokenHelper and then call DeviceToken::UpdateDeviceToken. Make sure the two output params get set with the values returned by DeviceTokenHelper.
	- Copilot query for implementing tests 
	> Can you implement TryLoginWithDeviceTokenBasic and the other three tests below it for TryLoginWithDeviceToken? TryLoginWithDeviceTokenBasic is the normal workflow. Use CreateDeviceTokenBasic as an example. Note that you will need to call CreateDeviceToken in order to do TryLoginWithDeviceToken. The other tests have the error condition in the name to test for.
	- Done
- Fix login so that it doesn't create a separate Session object
	- Created branch: auth_55_fix_login_session
	- Done
- Add DeviceToken support to Session
	- Created branch: auth_56_initialize_from_device_token
	- Add a bool remember to InitializeFromLogin
		- Have that cause the device token to be created and the cookie set (factor setting the cookie into a helper function)
		- Call PersonHelper::CreateDeviceToken
		- Call SetDeviceTokenCookie
	- Add an InitializeFromDeviceToken\(Transaction& transaction, Secrets::SecretsHelperPtr secrets, CookieManagerPtr cookieManager\)
		- This will check for the cookie for the device token, parse it, and then call TryLoginWithDeviceToken. On success, it will call CreateSessionToken, InitializeFromLogin, and then the helper to set the cookie
	- Created a SetDeviceTokenCookie helper
		- This sets the device token cookie from either InitializeFromLogin or InitializeFromDeviceToken
		- This is implemented in InitializeFromLoginRemember
		> Can you implement the test InitializeFromLoginRemember? You can base it off of InitializeFromLoginBasic. It will be pretty similar but you will pass true for the remember flag. Check to make sure that an entry with the same person id gets added to the device tokens table in addition to the session token. Can you modify InitializeFromLoginBasic to make sure an entry is NOT added to the device_tokens table?
	- Created SetCookieHelper
		- This is common code shared with setting the session token and the device token
		- This is implemented
	- Need to touch up call sites for InitializeFromLogin, add tests for remember functionality and add tests for InitializeFromDeviceToken
	> Can you implement the test InitializeFromDeviceTokenBasic? Please look at InitializeFromFromCookieBasic, InitializeFromLoginRemember, InitializeFromLoginBasic, and the function InitializeFromDeviceToken itself. You can also look at the test in person_test.cpp TryLoginWithDeviceTokenBasic. Please validate things at the database level and make sure that the device token cookie is present when the call is made AND that it changes after the call. Also, make sure that the session cookie is set as well. This is a remember me token for authentication that is used to do a login via device token which also causes a new session to be created and for the device token to be rotated in the database (old one replaced) and in the cookie sent back to the user. Also, create a test InitializeFromDeviceTokenNotFound for a token sent in the cookie that is not in the database, InitializeFromDeviceTokenExpired for an expired token, and then another for InitializeFromDeviceTokenRevoked for a device token that has been revoked. Please add any necessary headers but don't otherwise reformat or change existing tests.
	- Need to add a test for remember true to login test
	- Copilot prompt for login test
	> Can you implement the test LoginRemember? It should be very similar to LoginBasic but will pass true to remember for the remember JSON field for body. Look at auth/session_test.cpp in InitializeFromDeviceTokenBasic for more context. Make sure that the device token is set in the database and in the cookie after. Can you also modify LoginBasic to make sure that device_token is not set in the cookie after the call? Please make no other changes to LoginBasic.
	- Done
- Phase 14 - Add a Remember() endpoint that takes the device token, validates it, and that it is not expired and generates a new device token and session and responds to the user. Also add a me() endpoint that returns 401 if the user can't be validated of is expired of revoked.
	- The plan
		- Add /api/remember that does Session::InitializeFromDeviceToken
		- Add /api/me that does Session::InitializeFromFromCookie and returns 401 otherwise to cause a call to /remember from the client or /login
	- The implementation
		- Created branch: auth_57_remember
		- Copilot prompt
		> I want to create a new endpoint for my webserver that does the device token / remember me function. Please place the files in this folder with the declaration for the function in remember.h, the implementation in remember.cpp, and the tests in remember_test.cpp. Please use login.h/cpp and login_test.cpp as templates. Put everything in the same namespace (Endpoints) and the tests in an anonymous namespace. The function should be bool Remember\(EndpointAuthHelper& endpointAuthHelper, const crow::request& req, crow::response& resp\). Please return 401 if this function fails or an exception is thrown. Please return the same text regardless (ie. don't return the exception text). I don't want hackers to know why it failed. The crow route should be "/api/remember". Please use the same structure as login.cpp with a SetupRouting and handle post that call Remember. Please update CMakeLists.txt with these files and make no other changes to that file. Please add the function to web_app.h with the header. Please return 200 on success. Inside Remember, please call GetSession() on EndpointAuthHelper and then call InitializeFromDeviceToken on that. Please create a test under RememberTest called RememberBasic that evaluates the normal workflow and verifies that 200 is returned. Please look at auth/session_test.cpp under InitializeFromDeviceTokenBasic and InitializeFromLoginRemember for help for the test. Mainly just make sure to set a device token before the call and make sure that the session and device token are different after (if you set a session token beforehand). Create another test called RememberInvalid with an invalid device token and make sure that 401 is returned.
		- Completed /remember now work on /me
		- Created branch: auth_58_me
		- Copilot query:
		> I want to create a new endpoint for my webserver that does the session token / auto login functionality. Please place the files in this folder with the declaration for the function in me.h, the implementation in me.cpp, and the tests in me_test.cpp. Please use remember.h/cpp and remember_test.cpp as templates. Put everything in the same namespace (Endpoints) and the tests in an anonymous namespace. The function should be bool Me\(EndpointAuthHelper& endpointAuthHelper, const crow::request& req, crow::response& resp\). Please return 401 if this function fails or an exception is thrown. Please return the same text regardless (ie. don't return the exception text). I don't want hackers to know why it failed. The crow route should be "/api/me". Please use the same structure as remember.cpp with a SetupRouting and handle post that call Me. Please update CMakeLists.txt with these files and make no other changes to that file. Please add the function to web_app.h with the header. Please return 200 on success. Inside Me, please call GetSession() on EndpointAuthHelper and then call InitializeFromFromCookie on that. Please create a test under MeTest called MeBasic that evaluates the normal workflow and verifies that 200 is returned. Please look at auth/session_test.cpp under InitializeFromFromCookieBasic for help for the test. Mainly just make sure to set a session token before the call . Create another test called MeInvalid with an invalid session token and make sure that 401 is returned.
		- Done!
		- Server auth is done!

# What I'm working on 12/4
- Implement logout
	- Invalidate all device tokens and kill sessions
	- The plan
		- Lookup the current user
		- Add a Logout(int personId) to PersonHelper
			- Call DeviceTokens::RemoveDeviceTokensForUser and Sessions::RemoveSessionsForUser
		- Add a /logout endpoint that calls Logout
			- Remove the cookies
	- The implmentation
		- First stage: the changes to PersonHelper
			- Created branch: auth_59_logout_person
			- Copilot prompt:
			> Can you implement PersonHelper::LogoutPerson in this file and the test LogoutPersonBasic in person_test.cpp? For LogoutPerson, you need to call DeviceTokens::RemoveDeviceTokensForUser and Sessions::RemoveSessionsForUser. For LogoutPersonBasic, look at the tests for RemoveDeviceTokensForUserBasic and RemoveSessionsForUserBasic in sql_util/table_helpers in the files session_test.cpp and device_tokens_test.cpp. Validate that entries are removed from both the sessions and device_tokens tables. Looke at CreateDeviceTokenBasic and CreateSessionTokenBasic for how to add device and session tokens. This is basically the workflow for my website that a user has logged in with a remember token and established a session and then wants to logout and have the device token and session tokens removed. Please don't try to cleanup other functions in the files.
			- Done
		- Second stage: add the endpoint
			- Copilot prompt:
			> I want to create a new endpoint for my webserver that does session / device token logout. Please place the files in this folder with the declaration for the function in logout.h, the implementation in logout.cpp, and the tests in logout_test.cpp. Please use remember.h/cpp and remember_test.cpp as templates. Put everything in the same namespace (Endpoints) and the tests in an anonymous namespace. The function should be void Logout\(EndpointAuthHelper& endpointAuthHelper, const crow::request& req, crow::response& resp\). Please return 200 even if this function fails or an exception is thrown (Logout should never fail). The crow route should be "/api/logout". Please use the same structure as remember.cpp with a SetupRouting and handle post that call Logout. Please update CMakeLists.txt with these files and make no other changes to that file. Please add the function to web_app.h with the header. Please return 200 on success. Inside Logout, please call PersonHelper::Logout and remove the session_token and device_token cookies. Please create a test under LogoutTest called LogoutBasic that evaluates the normal workflow and verifies that 200 is returned. Please look at auth/person_test.cpp under LogoutPersonBasic for help for the test. In addition to the database validation in LogoutPersonBasic, please also make sure the cookies are gone after the call.
			- Done

# What I'm working on 12/8
- Things to do on the server
	- API to fetch information about current user (can't just give open access to users table)
	- Design role based security
- API to fetch information about current user (can't just give open access to users table)
	- The plan
		- Add /get_user_info endpoint that calls PersonHelper LookupPerson after calling Session::IsLoggedIn and Session::GetPersonId
		- Use util/json_util.h for the JsonFromKeyValueTable to build a KeyValueTable from PersonInfo converting the camelCase names in PersonInfo to snake case. Return the JSON like endpoints/get_row.cpp
	- The implementation
		- Created branch: auth_60_get_user_info
		- Copilot query
		> I want to create a new endpoint for my webserver that fetches user information for the currently logged in user. Please place the files in this folder with the declaration for the function in get_user_info.h, the implementation in get_user_info.cpp, and the tests in get_user_info_test.cpp. Please use remember.h/cpp and remember_test.cpp as templates as well as get_row.cpp and get_row_test.cpp. Put everything in the same namespace (Endpoints) and the tests in an anonymous namespace. The function should be crow::wvalue GetUserInfo\(EndpointAuthHelper& endpointAuthHelper, const crow::request& req, crow::response& resp\). Please return 401 if this function fails but don't return the exception text since I don't want a hacker to know why this failed. The crow route should be "/api/get_user_info". Please use the same structure as remember.cpp with a SetupRouting and handle post that call GetUserInfo. Please update CMakeLists.txt with these files and make no other changes to that file. Please add the function to web_app.h with the header. Please return 200 on success. Inside GetUserInfo, please call EnpointAuthHelper::GetSession() to fetch the session. Use IsLoggedIn and then GetPersonId to fetch the person or throw an exception. From there, Call PersonHelper::LookupPerson. Take the PersonInfo and build a KeyValueTable but use the snake case version of the camel case names of the fields in PersonInfo to insert all three values into the KeyValueTable. Return the JSON using util/json_util.h's JsonFromKeyValueTable. Use get_row.cpp for how to return JSON to the user. Please create a test under GetUserInfoTest called GetUserInfoBasic that evaluates the normal workflow and verifies that 200 is returned. Please look at get_row_test.cpp for how to validate the JSON returned. Create another test GetUserInfoNotLoggedIn to test the case that there is no user logged in and that 401 is returned (which you should return on an exception being thrown). Please notice that the various Cookie related helpers are in the Auth namespace. When you generate code, you keep missing this namespace. Please look at the changes I had to make to logout.cpp and logout_test.cpp to see the namespace and header includes I had to fix in what you generated. In other words, use the contents of the files themselves instead of your context of what you generated.
		- Done
- Design role based security
	- Tables
		- roles
			- id
			- name
			- description
		- permissions
			- id
			- name
			- description
		- role_permissions
			- id
			- role_id
			- permission_id
		- role_assignments
			- id
			- person_id
			- role_id
	- TableHelpers
		- Roles
			- int AddRole(Transaction & transaction, string_view name, string_view description)
			- KeyValueTable GetRole(Transaction& transaction, int id)
			- KeyValueTable GetRole(Transaction& transaction, string_view name)
			- KeyValueArray GetRoles(Transaction& transaction)
			- void SetName(Transaction& transaction, int id, string_view name)
			- void SetDescription(Transaction& transaction, string_view description)
			- void DeleteRole(int id)
		- Permissions
			- int AddPermission(Transaction & transaction, string_view name, string_view description)
			- KeyValueTable GetPermission(Transaction& transaction, int id)
			- KeyValueTable GetPermission(Transaction& transaction, string_view name)
			- KeyValueArray GetPermissions(Transaction& transaction)
			- void SetName(Transaction& transaction, int id, string_view name)
			- void SetDescription(Transaction& transaction, string_view description)
			- void DeletePermission(int id)
		- RolePermissions
			- int AddRolePermission(Transaction& transaction, int roleId, int permissionId)
			- KeyValueTable GetRolePermission(Transaction& transaction, int id)
			- KeyValueTableArray GetRolePermissionsForRole(Transaction& transaction, int roleId)
			- KeyValueTableArray GetRolePermissionsForPermission(Transaction& transaction, int permissionId)
			- KeyValueTableArray GetRolePermissions(Transaction& transaction)
			- void DeleteRolePermission(Transaction& transaction, int id)
		- RoleAssignments
			- int AddRoleAssignment(Transaction& transaction, int personId, int roleId)
			- KeyValueTable GetRoleAssignments(Transaction& transaction, int id)
			- KeyValueTableArray GetRoleAssignmentsForPerson(Transaction& transaction, int personId)
			- KeyValueTableArray GetRoleAssignmentsForRole(Transaction& transaction, int roleId)
			- KeyValueTableArray GetRoleAssignments(Transaction& transaction)
			- void DeleteRoleAssignment(Transaction& transaction, int id)
		- PersonHelper
			- StringArray GetRolesForUser(int personId)
			- StringArray GetPermissionsForUser(int personId)
		- Session
			- bool ActiveUserHasRole(string_view roleName)
			- bool ActiveUserHasPermission(string_view permissionName)

# What I'm workin on 12/9
- Add Tables
	- The plan
		- roles
			- id
			- name
			- description
		- permissions
			- id
			- name
			- description
		- role_permissions
			- id
			- role_id
			- permission_id
		- role_assignments
			- id
			- person_id
			- role_id
	- The implementation
		- Created branch: auth_61_role_tables
		- Copilot query
		> I am adding role based security to my webserver. In the db_schema folder, each set of .h/.cpp file pairs defines the schema for a table in the database. If you look at a file like people.h/people.cpp, you will see that the header defines an identifier for the table name with k prepended to the table name with the first letter capitalized and then the word Table appended and then the constant. The fields all are named with k, the capitalized table name, and then the camel case word conversion of the snake case string literals. There is also a void Make{table_name}Table function declared. Inside the cpp file, you see the implementation of this function. It starts with a primary key and then adds other fiels based on their type and if they are nullable, unique, etc. If you look in sessions.cpp, you will see AddColumnForeignKeyRef used to add a foriegn key ref. With this background, I need you to add four tables for me. Each of these should be in it's own .h/.cpp file pair. The roles table has id as its primary key, and two simple string colums: name and description. The permissions table has id as it's primary key and two simple string columns: name and description. The role_permissions table has id as its primary key and role_id and permission_id are foreign key references to the roles and permssions tables. The role_assignments table has id as its primar key and person_id and role_id are foreighn key references to the people and roles tables. Please look at the syntax of the other tables in this folder and use the same conventions and syntax, namespaces, as the other tables. Please add each of these new files to CMakeLists.txt but only add the files, please don't try to change anything else. Also, please maintain the same comment style for namespaces even if you normally would want to change it to get rid of the trailing curly brace.
	- Done
-  TableHelpers
	- Roles
		- int AddRole(Transaction & transaction, string_view name, string_view description)
		- KeyValueTable GetRole(Transaction& transaction, int id)
		- KeyValueTable GetRole(Transaction& transaction, string_view name)
		- KeyValueArray GetRoles(Transaction& transaction)
		- void SetName(Transaction& transaction, int id, string_view name)
		- void SetDescription(Transaction& transaction, string_view description)
		- void DeleteRole(int id)
	- Permissions
		- int AddPermission(Transaction & transaction, string_view name, string_view description)
		- KeyValueTable GetPermission(Transaction& transaction, int id)
		- KeyValueTable GetPermission(Transaction& transaction, string_view name)
		- KeyValueArray GetPermissions(Transaction& transaction)
		- void SetName(Transaction& transaction, int id, string_view name)
		- void SetDescription(Transaction& transaction, string_view description)
		- void DeletePermission(int id)
	- RolePermissions
		- int AddRolePermission(Transaction& transaction, int roleId, int permissionId)
		- KeyValueTable GetRolePermission(Transaction& transaction, int id)
		- KeyValueTableArray GetRolePermissionsForRole(Transaction& transaction, int roleId)
		- KeyValueTableArray GetRolePermissionsForPermission(Transaction& transaction, int permissionId)
		- KeyValueTableArray GetRolePermissions(Transaction& transaction)
		- void DeleteRolePermission(Transaction& transaction, int id)
	- RoleAssignments
		- int AddRoleAssignment(Transaction& transaction, int personId, int roleId)
		- KeyValueTable GetRoleAssignments(Transaction& transaction, int id)
		- KeyValueTableArray GetRoleAssignmentsForPerson(Transaction& transaction, int personId)
		- KeyValueTableArray GetRoleAssignmentsForRole(Transaction& transaction, int roleId)
		- KeyValueTableArray GetRoleAssignments(Transaction& transaction)
		- void DeleteRoleAssignment(Transaction& transaction, int id)
	- Implementation
		- Roles
			- Created branch: auth_62_roles
			> Using session.h/.cpp/test.cpp and people.h/.cpp/test.cpp as templates, please add a CRUD wrapper for this table:
			
			Roles
				int AddRole(Transaction & transaction, string_view name, string_view description)
			    KeyValueTable GetRole(Transaction& transaction, int id)
			    KeyValueTable GetRole(Transaction& transaction, string_view name)
			    KeyValueArray GetRoles(Transaction& transaction)
			    void SetName(Transaction& transaction, int id, string_view name)
			    void SetDescription(Transaction& transaction, string_view description)
			    void DeleteRole(int id)
			    
			Please use the same namespace and naming conventions as the other files (including leaving the same comment for namespace with the trailing curly brace). Please use sql_util/database_access/database_crud_helpers.h and util/types.h including KeyValueTableArrayFromDataResults. Put the class declaraion in roles.h, the implementation in roles.cpp, and the tests in roles_test.cpp. For the tests, please put the files in the anonymous namespace. The table being wrapped is in db_schema/roles.h. Use those identifiers. Please create a test fore each function under RolesTest with a function name based on the method name with Basic appended to exercise basic functionality. For the GetRole, add GetRoleIdNotFound and GetRoleNameNotFound to test that an exception is thrown for those cases. Add a GetRolesNoRoles to verify that an empty collection is returned when there are no roles. Add a SetNameNotFound to verify an exception is thrown. Add a SetDescriptionNotFound that also verifies an exception is thrown. Add a DeleteRole not found that verifies that an exception is NOT thrown. Validate all these tests at the database level. Please add the files to CMakeLists.txt and only add the files. Make no other changes.
		- Done
		- Permissions
			- Create branch: auth_63_permissions
			- Copilot prompt:
			> Using roles.h/.cpp/test.cpp and people.h/.cpp/test.cpp as templates, please add a CRUD wrapper for this table:
			
			Permissions
             int AddPermission(Transaction & transaction, string_view name, string_view description)
             KeyValueTable GetPermission(Transaction& transaction, int id)
             KeyValueTable GetPermission(Transaction& transaction, string_view name)
             KeyValueArray GetPermissions(Transaction& transaction)
             void SetName(Transaction& transaction, int id, string_view name)
             void SetDescription(Transaction& transaction, string_view description)
             void DeletePermission(int id)
             
             Please use the same namespace and naming conventions as the other files (including leaving the same comment for namespace with the trailing curly brace). Please use sql_util/database_access/database_crud_helpers.h and util/types.h including KeyValueTableArrayFromDataResults. Put the class declaraion in permissions.h, the implementation in permissions.cpp, and the tests in permissions_test.cpp. For the tests, please put the files in the anonymous namespace. The table being wrapped is in db_schema/permissions.h. Use those identifiers. Please create a test for each function under PermissionsTest with a function name based on the method name with Basic appended to exercise basic functionality. For the GetPermission, add GetPersmissionIdNotFound and GetPersmissionNameNotFound to test that an exception is thrown for those cases. Add a GetPersmissionsNoPermissions to verify that an empty collection is returned when there are no permissions. Add a SetNameNotFound to verify an exception is thrown. Add a SetDescriptionNotFound that also verifies an exception is thrown. Add a DeletePermissionNotFound that verifies that an exception is NOT thrown. Validate all these tests at the database level. Please add the files to CMakeLists.txt and only add the files. Make no other changes.
         - Done
	 - RolePermissions
		 - Created branch: auth_64_role_permissions
		 - Copilot prompt:
		 > Using roles.h/.cpp/test.cpp and permissions.h/.cpp/test.cpp and role_permissions.h/.cpp/test.cpp as templates, please add a CRUD wrapper for this table:
		 > 
		 > RoleAssignments
		 >   int AddRoleAssignment(Transaction& transaction, int personId, int roleId)
		 >   KeyValueTable GetRoleAssignments(Transaction& transaction, int id)
		 >   KeyValueTableArray GetRoleAssignmentsForPerson(Transaction& transaction, int personId)
		 >   KeyValueTableArray GetRoleAssignmentsForRole(Transaction& transaction, int roleId)
		 >   KeyValueTableArray GetRoleAssignments(Transaction& transaction)
		 >   void DeleteRoleAssignment(Transaction& transaction, int id)
		 >   
		 >   Please use the same namespace and naming conventions as the other files (including leaving the same comment for namespace with the trailing curly brace). Please use sql_util/database_access/database_crud_helpers.h and util/types.h including KeyValueTableArrayFromDataResults. Put the class declaraion in role_assignments.h, the implementation in role_assignments.cpp, and the tests in role_assignments_test.cpp. For the tests, please put the files in the anonymous namespace. The table being wrapped is in db_schema/role_assignments.h. Use those identifiers. Please create a test for each function under RoleAssignmentsTest with a function name based on the method name with Basic appended to exercise basic functionality. For the GetRoleAssignments family, add GetRoleAssignmentsNotFound, GetRoleAssignmentsForRoleNotFound, and GetRoleAssignmentsForPersonNotFound to test that an exception is thrown for those cases. Add a GetRoleAssignmentsNoRoleAssignments to verify that an empty collection is returned when there are no role assignments. Add a DeleteRoleAssignmentNotFound that verifies that an exception is NOT thrown. Validate all these tests at the database level. Please add the files to CMakeLists.txt and only add the files. Make no other changes.
		- Done
	- RoleAssignments
		- Created branch: auth_65_role_assignments
		- Copilot prompt:
		> Using roles.h/.cpp/test.cpp and permissions.h/.cpp/test.cpp as templates, please add a CRUD wrapper for this table:
		 > 
		 > RolePermissions
		 >   int AddRolePermission(Transaction& transaction, int roleId, int permissionId)
		 >   KeyValueTable GetRolePermission(Transaction& transaction, int id)
		 >   KeyValueTableArray GetRolePermissionsForRole(Transaction& transaction, int roleId)
		 >   KeyValueTableArray GetRolePermissionsForPermission(Transaction& transaction, int permissionId)
		 >   KeyValueTableArray GetRolePermissions(Transaction& transaction)
		 >   void DeleteRolePermission(Transaction& transaction, int id)
		 >   
		 >   Please use the same namespace and naming conventions as the other files (including leaving the same comment for namespace with the trailing curly brace). Please use sql_util/database_access/database_crud_helpers.h and util/types.h including KeyValueTableArrayFromDataResults. Put the class declaraion in role_permissions.h, the implementation in role_permissions.cpp, and the tests in role_permissions_test.cpp. For the tests, please put the files in the anonymous namespace. The table being wrapped is in db_schema/role_permissions.h. Use those identifiers. Please create a test for each function under RolePermissionsTest with a function name based on the method name with Basic appended to exercise basic functionality. For the GetRolePermission family, add GetRolePersmissionNotFound, GetRolePersmissionForRoleNotFound, and GetRolePermissionForPermissionNotFound to test that an exception is thrown for those cases. Add a GetRolePersmissionsNoRolePermissions to verify that an empty collection is returned when there are no role permissions. Add a DeleteRolePermissionNotFound that verifies that an exception is NOT thrown. Validate all these tests at the database level. Please add the files to CMakeLists.txt and only add the files. Make no other changes.
		 - Done
 - PersonHelper
	- StringArray GetRolesForUser(int personId)
	- StringArray GetPermissionsForUser(int personId)
	- Created branch: auth_66_person_permissions
	- Copilot prompt:
	> Please add these functions to PersonHelper:
	> 
	> StringArray GetRolesForUser(int personId)
	> StringArray GetPermissionsForUser(int personId)
	> 
	> Please use sql_util/table_helpers/role_assignments.h and role_permissions.h to implement these functions as well as roles.h and permissions.h. You will need to lookup the RoleAssignments for the given person and then use the role id to lookup the names for each role in the Roles to return an array of role names. You will need to do the same thing but use each role to lookup the RolePermissions to find the set of permissions for the given user and then use Permissions for each id to get the name to return back as an array for GetPermissionsForUser. Add the functions to the bottom of the existing public functions in person_helper.h and then add the implementation to the end of person_helper.cpp. Add tests to person_helper_test.cpp at the end of the file. Add tests with the name of the function with Basic appended to exercise ordinary functionality to the end of the file. Feel free to add helpers as needed at the start of the the file for the new tables for roles, permissions, etc but try not to change the existing code. Please add NotFound test variants for both functions to validate an exception is thrown for users that are not found.
	- Done
- Session
	- bool ActiveUserHasRole(string_view roleName)
	- bool ActiveUserHasPermission(string_view permissionName)
	- Created branch: auth_66_session_role_security
	- Copilot prompt:
	> Please add these functions to Session:
	> 
	> bool ActiveUserHasRole(string_view roleName)
	> bool ActiveUserHasPermission(string_view permissionName)
	> 
	> Please use sql_util/table_helpers/role_assignments.h and role_permissions.h to implement these functions as well as roles.h and permissions.h. You will need to lookup the role id for the given name in the Roles table and then use RoleAssignments for the given person to see if that role id is present for ActiveUserHasRole (use IsLoggedIn / GetPersonId on the session). For ActiveUserHasPermission, you will need to do the same thing but look through all the roles for the active user and then use role assignments to lookup roles and then use RolePermissions to look for the given permission after you have used Permissions to convert the given permission name to an id. Add the functions to the bottom of the existing public functions in session.h and then add the implementation to the end of session.cpp. Add tests to session_test.cpp at the end of the file. Add tests with the name of the function with Basic appended to exercise ordinary functionality to the end of the file. Feel free to add helpers as needed at the start of the the file for the new tables for roles, permissions, etc but try not to change the existing code. Please add NotFound test variants for both functions to validate an exception is thrown for users that are not found. Please add all the various includes for the various db_schema files too. You missed that for the last one.
	- Done
- TODO: Add roles and permissions to user info and then we are done with the server!!!!
	- Move to a new document for the client side stuff

# What I'm working on 12/10
- Add roles and permissions to /get-user_info
	- Use PersonHelper GetRolesForUser and GetPermissionsForUser
	- Created branch: auth_67_get_user_info_role_based_security
	- Copilot prompt for JSON test functions:
	> I have added the functions StringFromBool, JsonFromStringArray, JsonFromArrayOfJson, and JsonFromDataResults to json_util.h/.cpp. Can you add unit tests to the end of json_util_test.cpp? For each function, add a test function named with the name of the function with the word Basic appended. Please test normal execution flow. Use the other code in this file for how to test Crow's JSON support. Please do not make any other changes to the test file.
	- Copilot prompt
	> I would like to add two JSON keys to the JSON returned from /api/get_user_info. One is called "roles" and has the roles returned by PersonHelper::GetRolesForUser and the other is "permissions" and has the roles return by PersonHelper::getPermissionsForUser. Please do this in GetUserInfo after the call to JsonFromKeyValueTable. Use util/json_util.h's json_util::JsonFromStringArray to convert the string arrays to JSON that can be used to be inserted with the string operator[] on wvalue. Please modify the test in get_user_info_test.cpp GetUserInfoBasic to convert the resultant string to JSON with json_util's WvalueFromText and then RvalueFromWvalue. Use auth/person_test.cpp and util/json_util_test.cpp as examples for how to test this and maybe rewrite the test to validate at the JSON rvalue level instead of doing HasSubstr.
	- Done
- We have completed auth on the server!!!!!
- Make a new document for the client
	- [[Client Side Authentication and Security]]

# Branches
- auth_1_database
	- Submitted
- auth_2_mailio
	- Submitted
- auth_3_libsodium_conan
	- Submitted
- auth_4_auth_helper
	- Submitted
- auth_5_cleanup
	- Submitted
- auth_6_config_secrets
	- Submitted
- auth_7_table_helpers
	- Submitted
- auth_8_secrets
	- Submitted
- auth_9_mail
	- Submitted
- auth_10_auth_refactor
	- Submitted
- auth_11_crud_cleanup
	- Submitted
- auth_12_sql_keywords
	- Submitted
- auth_13_exec_params
	- Submitted
- auth_14_exec_params_complex
	- Submitted
- auth_15_exec_params_cleanup
	- Submitted
- auth_16_date
	- Submitted
- auth_17_sql_keywords
	- Submitted
- auth_18_people
	- Submitted
- auth_19_verify_window
	- Submitted
- auth_20_person
	- Submitted
- auth_21_web_app_update
	- Submitted
- auth_22_register
	- Submitted
- auth_23_verify
	- Submitted
- auth_24_now_us
	- Submitted
- auth_25_epoch
	- Submitted
- auth_26_decode
	- Submitted
- auth_27_admin_alerts
	- Submitted
- auth_28_no_transaction
	- Submitted
- auth_29_ddl
	- Submitted
- auth_30_email
	- Submitted
- auth_31_email_secret
	- Submitted
- auth_32_secret_cleanup
	- Submitted
- auth_33_mail_cleanup
	- Submitted
- auth_34_ auth_secrets
	- Submitted
- auth_35_hash_to_string
	- Submitted
- auth_36_sessions
	- Submitted
- auth_37_device_tokens
	- Submitted
- auth_38_create_session
	- Submitted
- auth_39_thread_pool
	- Submitted
- auth_40_session_last_seen
	- Submitted
- auth_41_endpoint_helper
	- Submitted
- auth_42_prod_mode
	- Submitted
- auth_43_server_config
	- Submitted
- auth_44_cookie_manager
	- Submitted
- auth_45_person_helper_cleanup
	- Submitted
- auth_46_session
	- Submitted
- auth_47_wire_session
	- Submitted
- auth_48_login
	- Submitted
- auth_49_auth_cookie
	- Submitted
- auth_50_split_string
	- Submitted
- auth_51_device_token
	- Submitted
- auth_52_cleanup
	- Submitted
- auth_53_device_token_uuid
	- Submitted
- auth_54_try_login_device_token
	- Submitted
- auth_55_fix_login_session
	- Submitted
- auth_56_initialize_from_device_token
	- Submitted
- auth_57_remember
	- Submitted
- auth_58_me
	- Submitted
- auth_59_logout_person
	- Submitted
- auth_60_get_user_info
	- Submitted
- auth_61_role_tables
	- Submitted
- auth_62_roles
	- Submitted
- auth_63_permissions
	- Submitted
- auth_64_role_permissions
	- Submitted
- auth_65_role_assignments
	- Submitted
- auth_66_session_role_security
	- Submitted
- auth_67_get_user_info_role_based_security
	- Submitted

# Requirements
- Functional and non-functional requirements go here

# High-Level Architecture
Description of high level design goes here

# Detailed Design
APIs, classes, data structures, algorithms, database design, security, performance

# Alternatives Considered
- List other possible alternative solutions and why they were not chosen

# Future Work
- List of things out of the scope of this design that probably still need to be tackled
