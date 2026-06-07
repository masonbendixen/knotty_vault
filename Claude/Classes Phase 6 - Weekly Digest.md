---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 6/5/2026
Version: 0.2
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please use your built in tools for read only operations on the filesystem or just say yes but do NOT prompt me when performing work that only reads the filesystem. I want you to run to completion (putting questions to be answered) but DO NOT FUCKING PROMPT ME. Please leave this Overview alone and build the plan in the following sections.

Classes Phase 6 - Weekly Digest

Please use the code base and the document Classes, schedules, and attendance.md. Please use these documents and the code base for context as well:

- [[Payment Design Document]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Event Polish- Scheduling Should Have Items]]
- [[Multi seat and bundled products]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Product, Event, and Subscription Admin Portal]]
- [[Provider Portal]]
- [[Scheduled Jobs]]
- [[Vouchers and Refunds]]

For each document, please extract the corresponding section from [[Classes, schedules, and attendance]] and place the information from that section of the document here as well as expanded details for the implementation as well as checkboxes to mark off as we complete them. For each piece, please start with the lowest layer of the system moving to higher layers on the server (db schema, table helpers, other helpers, business logic, and endpoints) and then client side work. Make sure to test everything that you can possibly test especially changes to existing files with tests.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here

## Phase Summary

**Should-have.** Sunday-noon (facility-TZ-aware) digest email for every active member, listing their this-week template attendance + one-off additions + paid bookings (workshops / series / intro / guest pass / regular events / services). One combined `.ics` attachment with per-occurrence VEVENTs (per parent doc WD-3). Strict opt-out support (notification preferences table). Also provides the per-user subscribable iCal feed URL (WD-6) — a `webcal://` URL the user adds to their calendar app once for auto-updating.

**Prerequisites:**
- Phase 1 (three-level schedule model + `ClassScheduleHelper::GetDerivedSessionsForRange`).
- Phase 4 (iCal generator extensions — multi-event overload, UID helpers).
- Phase 5 (attendance templates + exceptions — drive the recurring-classes part of the digest).
- Existing `MailHelper` + `FormatString` + `NormalizeCrLf` patterns (per CLAUDE.md).
- Existing `knottyyoga_helper` daemon ([[Scheduled Jobs]]).

### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]]
The "this week's recurring class attendance" part of the digest CANNOT be a `SELECT FROM bookings` — membership-included template attendance creates no booking rows and no `event_sessions` rows (lazy model). The digest must:
- **Derive the week's templated occurrences.** Add `WeeklyDigestHelper::GetTemplateOccurrencesForWeek(personId, weekStartUs)` that, for each of the user's `attendance_template_entries` (slot-keyed), walks the week via `ClassScheduleHelper::GetDerivedSessionsForRange` to find the slot's occurrences, then overlays `attendance_template_exceptions` by (`class_schedule_slot_id`, `occurrence_date_us`) to drop skips and add one-offs.
- **Union with persisted paid bookings.** Workshops / series / intro / guest pass / events / services DO have persisted `event_sessions` + `bookings`; those come from the existing booking query. De-dupe by occurrence.
- The multi-VEVENT `.ics` is built from the unioned occurrence list. UID for a templated (booking-less) occurrence uses `BuildTemplateUid(classScheduleSlotId, personId)` + the occurrence date (Phase 4 helper, slot-keyed per the redesign); paid bookings use `BuildBookingUid`.

**Outcome:**
- New `user_notification_preferences` table with defaults.
- `WeeklyDigestHelper` builds + sends idempotently.
- Admin endpoint `POST /api/admin/send_weekly_digests` is hourly-tickled by `knottyyoga_helper`.
- New `/my/account/preferences` page with a "Weekly digest" toggle.
- New `webcal://` subscribable URL endpoint with token-based auth.

## Layering & Conventions

Lowest layer first:

1. `db_schema/` — `user_notification_preferences`, `ical_feed_tokens` (Phase 6 introduces both since they go together).
2. `sql_util/table_helpers/` — two new helpers.
3. `business_logic/scheduling/` — `WeeklyDigestHelper`; `PersonalICalFeedHelper`.
4. `endpoints/` — admin send-digests endpoint + user preferences endpoints + the public-but-tokenized iCal feed endpoint.
5. Scheduled jobs in `knottyyoga_helper`.
6. Angular: preferences page; "your subscribable calendar URL" panel.
7. Admin metadata.
8. Tests at every layer.

## 1. Pre-Coding Design Decisions

### 1.1 Locked-in (resolved per parent doc §2.9, WD-3, WD-6)
- [x] Multi-VEVENT per session (NOT a single RRULE) — exception-accurate.
- [x] iCal feed authenticated via random token (not JWT), hashed at rest, user-regenerable.
- [x] `X-PUBLISHED-TTL: PT1H` (1 hour) on the feed.
- [x] Sunday 12:00 in facility-local time by default; per-user override of day-of-week / hour.

### 1.2 Idempotency
- [x] Tracks `last_digest_sent_us` per user; the send-pending-digests job only sends once per (user, week-bucket).
- [x] Hourly job at the `knottyyoga_helper` level — endpoint is safe to re-trigger.

### 1.3 Multi-facility
- [x] For a user belonging to multiple facilities, send a single digest covering all facilities; render each row with the facility name. Out-of-scope to send a per-facility digest in Phase 6.

## 2. Database Schema

### 2.1 `user_notification_preferences` table
- [x] `db_schema/user_notification_preferences.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL UNIQUE REFERENCES people(id)`
  - `weekly_digest_enabled BOOLEAN NOT NULL DEFAULT TRUE`
  - `digest_send_dow INT NOT NULL DEFAULT 0`  (0=Sunday..6=Saturday)
  - `digest_send_hour_local INT NOT NULL DEFAULT 12`  (0..23, local to user's facility)
  - `last_digest_sent_us BIGINT NOT NULL DEFAULT 0`
  - `created_us`, `updated_us`

### 2.2 `ical_feed_tokens` table
- [x] `db_schema/ical_feed_tokens.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL UNIQUE REFERENCES people(id)`
  - `token_hash TEXT NOT NULL`  — Argon2id or SHA-256 hash of the random token (consistent with `sessions.token_hash` pattern used elsewhere)
  - `created_us`, `last_used_us`, `revoked_us` (nullable)
- [x] Unique on `person_id` keeps it simple: one active token at a time. Regeneration = update the hash + invalidate the prior.

### 2.3 Wire into DB init
- [x] `make_database_info.cpp` + `create_database.cpp` `CreateTables()`.
- [x] CMakeLists.

## 3. Table Helpers

### 3.1 `TableHelpers::UserNotificationPreferences`
Files: `sql_util/table_helpers/user_notification_preferences.h/.cpp/_test.cpp`.
- [x] `GetForPerson(Transaction&, personId)` → row or empty.
- [x] `GetOrCreateForPerson(Transaction&, personId)` — idempotent fetch-or-create with defaults (returns the row `KeyValueTable`).
- [x] `UpdatePreferences(Transaction&, personId, const KeyValueTable& updates)` — applies only the editable subset (`weekly_digest_enabled`, `digest_send_dow`, `digest_send_hour_local`); ignores protected keys; bumps `updated_us`; creates the row first if missing.
- [x] `SetLastDigestSent(Transaction&, personId, sentUs)` — idempotency mutator (creates row first if missing).
- [x] **`GetEnabledDigestPreferences(Transaction&)`** replaces the planned `GetUsersDueForDigest(asOfUs)` here. **Layering decision:** the precise "due right now?" test needs each user's *facility timezone* (not in this table) plus the per-week idempotency-window math. That belongs in business logic, so this table helper exposes only the SQL-expressible slice — every row with `weekly_digest_enabled = TRUE`, oldest-first — and `WeeklyDigestHelper::SendPendingDigests` (§4) applies the TZ gating + idempotency window over those candidates. The "due-list correctness" test moves to §4.
- [x] Tests: fetch-or-create defaults, idempotency, update (editable + ignored-protected-keys), create-on-update-when-missing, set-last-sent, enabled-filter ordering.

### 3.2 `TableHelpers::ICalFeedTokens`
Files: `sql_util/table_helpers/ical_feed_tokens.h/.cpp/_test.cpp`.
**Layering decision:** mirrors the email-verification split — the table helper is pure CRUD over the token **hash** only; it never generates a raw token or hashes one. Generating the random token + BLAKE2b hash (`AuthHelper`) lives in the business-logic layer (§4), exactly as email-verification token generation lives in `PersonHelper`, not in `TableHelpers::EmailVerifications`. (Table helpers must not depend upward on `business_logic/auth`.)
- [x] `GetForPerson(Transaction&, personId)` → row (active or revoked) or empty.
- [x] `CreateOrReplaceTokenHash(Transaction&, personId, tokenHash)` → replaces any prior row (regeneration), returns new id. Backs the business-logic `RegenerateToken`, which generates the raw token + hash and passes the hash here.
- [x] `LookupPersonByTokenHash(Transaction&, tokenHash)` → personId or 0 (non-revoked only). Backs the business-logic `LookupPersonByToken`, which hashes the input first.
- [x] `RevokeForPerson(Transaction&, personId)` → stamps `revoked_us` without deleting (added; supports revoke without regenerate).
- [x] `RecordUse(Transaction&, personId)` updates `last_used_us`.
- [x] Tests: round-trip create+lookup, hash-mismatch/empty → 0, regeneration replaces prior token (old hash → 0), revoke disables lookup but keeps row, regenerate-after-revoke reactivates, record-use stamps, per-person scoping.

## 4. Business Logic

### 4.0 Prerequisite — Phase 4 iCal helper extension (DONE)
- [x] Added `ICalGenerator::BuildTemplateOccurrenceUid(classScheduleSlotId, personId, occurrenceDateUs)` to `util/ical_generator.h/.cpp` (+ test). Slot-keyed, per-occurrence UID for the booking-less templated occurrences — the existing `BuildTemplateUid` is schedule-keyed and reserved for the single recurring-RRULE confirmation invite, so a new function avoids overloading its meaning. The redesign note's "`BuildTemplateUid(slot, person)` + date" is realized by this dedicated builder.
- [x] Added `TableHelpers::Facilities::GetFirstActiveFacilityTimezone()` (+ tests) — the studio-default timezone fallback.

### 4.1 `WeeklyDigestHelper` (DONE)
Files: `business_logic/scheduling/weekly_digest_helper.h/.cpp/_test.cpp`.

- [x] `struct WeeklyDigestRow` — extended beyond the original sketch with `displayWhen` (pre-formatted local time, so the mail generator stays a pure string builder) and the UID-source fields `classScheduleSlotId` / `occurrenceDateUs` / `bookingId` (the `.ics` is built straight from the row list).
- [x] `struct WeeklyDigestData { personId; personFirstName; personEmail; ianaTz; windowStartUs; windowEndUs; rows; }`.
- [x] **Reuse decision:** templated occurrences come from `AttendanceTemplateHelper::GetUpcomingClassesForPerson` (Phase 5) — it already derives + decorates every eligible occurrence (className/facility/room/instructors/onTemplate/exception state) over a range. The digest filters that to `attendingViaTemplate = onTemplate && !exceptionSkipping` plus one-off `exceptionAttending` additions. No separate `GetTemplateOccurrencesForWeek` was needed.
- [x] `ResolvePersonTimezone(tx, personId)` — template-slot facility → upcoming-booking facility → first active facility → "UTC". (No `people.facility_id` exists; a member's facility is inferred from activity.)
- [x] `CollectRowsForWindow` / `BuildDigestForWindow` / `BuildDigestForPerson(weekStartUs)`. weekEnd = next local Monday (DST-aware via SQL `AT TIME ZONE`). Paid bookings via `BookingHelper::GetBookingsForPerson(upcoming)` (covers events **and** services, OQ-P6-1), de-duped vs templated by persisted `event_session_id` and by (start, className). Sorted by start.
- [x] `static BuildICalEvents(data)` — one VEVENT per row; paid → `BuildBookingUid`, templated → `BuildTemplateOccurrenceUid`.
- [x] `SendDigestForPerson(tx, personId, weekStartUs)` — empty-skip (OQ-P6-2), HTML+text bodies, `weekly_digest.ics` attachment, records `last_digest_sent_us`. Mail held as a constructor member (matching `AttendanceTemplateHelper`) rather than a `MailHelper*` param; no `ThreadPool::Queue` is used here (synchronous send), so the sync-SQL-before-queue rule doesn't apply.
- [x] `SendPendingDigests(tx, asOfUs)` — iterates `UserNotificationPreferences::GetEnabledDigestPreferences`, computes each member's local **send anchor** (most recent past dow+hour in their tz) and previewed **week start** (next local Monday) via SQL tz math, and sends when `last_digest_sent_us < anchor`. **Marks the bucket with `asOfUs`** after each due member (sent or empty) → idempotent. (The original plan's `GetUsersDueForDigest` tz math lives here, in business logic, because it needs the facility timezone — see §3.1 layering note.)
- [x] Tests with `TestMailHelper`: empty skipped; templated occurrence; one-off addition; exception-skip removes a templated occurrence; paid booking subtitle; union+sort; send-with-`.ics`; multi-VEVENT + booking/slot UIDs; `SendPendingDigests` sends-then-idempotent; disabled member skipped; before-send-time not due.

### 4.2 `PersonalICalFeedHelper` (DONE)
- [x] `business_logic/scheduling/personal_ical_feed_helper.h/.cpp/_test.cpp`.
- [x] `GenerateFeedForPerson(tx, personId, windowStartUs, windowEndUs)` — reuses `WeeklyDigestHelper::BuildDigestForWindow` over a long window (`kDefaultWindowUs` = 90 days) and returns the multi-VEVENT iCal text.
- [ ] Headers (`Content-Type: text/calendar`, `X-PUBLISHED-TTL: PT1H`, `Cache-Control: max-age=3600`) — set by the §5.3 endpoint, not the helper.
- [x] Tests: empty → valid event-less VCALENDAR; templated+paid in a 1-week window → 2 VEVENTs with both UID forms; 90-day window → many occurrences; exception-skip respected.

### 4.3 Email template (DONE)
- [x] `business_logic/scheduling/weekly_digest_mail.h/.cpp` — `GenerateWeeklyDigestHtmlBody` + `GenerateWeeklyDigestTextBody`, `FormatString` template constants, `NormalizeCrLf` on both bodies.
- [x] Plain-text rows: `Tue Mar 5  6:00-7:00pm  Vinyasa Flow (Studio A) - Instructor Sara` (ASCII hyphens to keep golden tests encoding-safe; room name falls back to facility name).
- [x] Test: substring/golden assertions on both bodies + CRLF + room-fallback + greeting-only-when-empty.

## 5. Endpoints

### 5.1 Admin send-digests endpoint
- [ ] `endpoints/admin_send_weekly_digests.h/cpp` + test:
  - `POST /api/admin/send_weekly_digests` — calls `WeeklyDigestHelper::SendPendingDigests(now)`. Returns `{ sent_count: N }`. Idempotent.
  - Permission: `admin` (service account inherits it).

### 5.2 User preferences endpoints
- [ ] `endpoints/get_my_notification_preferences.h/cpp` + test:
  - `GET /api/me/notification_preferences` → preferences row.
- [ ] `endpoints/update_my_notification_preferences.h/cpp` + test:
  - `PUT /api/me/notification_preferences` body `{ weekly_digest_enabled?, digest_send_dow?, digest_send_hour_local? }`.

### 5.3 Personal iCal feed endpoint
- [ ] `endpoints/get_my_ical_feed.h/cpp` + test:
  - **(OQ-P6-3 — opaque)** `GET /api/me/ical_feed.ics?token=<...>` — opaque path (NOT email-bearing); public route (no session) gated on the token.
  - Looks up token → personId; if invalid → 404 (NOT 401 — keep the URL low-information for crawlers).
  - Calls `PersonalICalFeedHelper::GenerateFeedForPerson(personId, now, now+90d)`.
  - Sets the iCal headers from §4.2.
  - Records `last_used_us`.
- [ ] `endpoints/regenerate_ical_feed_token.h/cpp` + test:
  - `POST /api/me/ical_feed/regenerate` — returns the new raw `webcal://...` URL.
- [ ] Note: the `webcal://` URL is constructed client-side by replacing `https://` with `webcal://` on the URL the API returns.

### 5.4 Routing
- [ ] All registered in `web_app.cpp`.

## 6. Scheduled job integration

- [ ] Add hourly job to `knottyyoga_helper`: `POST /api/admin/send_weekly_digests`. Hourly cadence is fine because the endpoint is idempotent and `GetUsersDueForDigest` filters precisely. Helper logs the returned `sent_count`.
- [ ] No config secrets needed — defaults in the DB column suffice.

## 7. Frontend

### 7.1 Notification preferences page
- [ ] `ui/src/app/pages/account/notification-preferences/notification-preferences.component.*/.spec.ts`.
- [ ] Toggle for weekly digest; day-of-week picker; hour-of-day picker (per memory `feedback_date_time_pickers.md` — hour-of-day uses a real hour picker, not a free text field).
- [ ] Mat-card border per memory `feedback_mat_card_border.md`; back nav per memory `feedback_account_page_layout.md`.

### 7.2 Calendar feed panel
- [ ] On the preferences page (or a separate `/my/account/calendar-feed` page), show the user's `webcal://...` URL with copy-to-clipboard button + "Regenerate URL" action.
- [ ] Explanatory text: "Paste this into Google Calendar / Apple Calendar / Outlook to subscribe. Your calendar will auto-update as your bookings change."
- [ ] Spec.

### 7.3 `ServerAccess` extensions
- [ ] `getMyNotificationPreferences()`, `updateMyNotificationPreferences(prefs)`, `regenerateICalFeedToken()`, `getMyICalFeedUrl()` (returns the URL with token in query string).
- [ ] Update `ServerAccess.mock.spec.ts`.

### 7.4 Types
- [ ] `ui/src/app/shared/types/notification.types.ts`: `NotificationPreferences`, `ICalFeedInfo`.

## 8. Admin Metadata

- [ ] `user_notification_preferences` → `admin_nested_tables` under `people` keyed by `person_id`, permission `manage_users` or `admin`.
- [ ] `ical_feed_tokens` → nested under `people`; token hash is **redacted** in the admin view (per the existing `admin_column_redactions` pattern).

## 9. Tests-Required Summary

- [ ] Table helper tests for both new tables.
- [ ] `weekly_digest_helper_test.cpp`: empty skipped, full digest, exception-skip, paid booking, idempotency.
- [ ] `personal_ical_feed_helper_test.cpp`: 90-day window, exception-skip respected, includes both templated + paid items.
- [ ] Endpoint tests for all five new endpoints (`admin_send_weekly_digests`, `get/update_my_notification_preferences`, `get_my_ical_feed`, `regenerate_ical_feed_token`).
- [ ] Mail helper test: assert HTML body + plain-text body + attachment count + filename `weekly_digest.ics`.
- [ ] Frontend specs for preferences page, calendar-feed panel, mock service.

## 10. Cross-Layer Acceptance Criteria

A logged-in member with:
- Templated "Vinyasa Flow" Mon/Wed 7pm
- Paid booking for "6-Week Aerial 101" Tue 8pm
- An exception "skipping this Wed"

Sunday 12:00 local time (which crosses the hourly trigger):
- [ ] Member receives a digest listing **three** rows for the week: Mon 7pm Vinyasa, Tue 8pm Aerial, and NO Wed entry. (The exception removed the Wed slot.)
- [ ] Email has a `text/calendar` attachment with three VEVENTs and correct UIDs.
- [ ] Re-running the hourly trigger an hour later sends zero new digests (idempotent).

A member opens `/my/account/notification-preferences`, clicks the calendar-feed copy button, pastes the `webcal://...` URL into Apple Calendar:
- [ ] Apple Calendar resolves the URL, shows the next 90 days of templated + paid sessions in a separate subscribed calendar.
- [ ] Adding a new paid booking → within 1 hour the new event appears in the subscribed calendar.

## 11. Open Questions — ALL RESOLVED (2026-06-05)

- [x] **OQ-P6-1 (resolved — yes, include paid services/events).** The digest covers paid services / events (massage, etc.) on the same surface, each row subtitled e.g. "Massage with Provider X — 60min". Already reflected in §4.1 step 5 + the Phase Summary. Mason: "I'll go with your recommendation."
- [x] **OQ-P6-2 (resolved — NO empty digests).** When a user has zero rows for the week, no digest is sent (empty digests are noise; don't pester a disengaged user). Already reflected in §4.1 `SendDigestForPerson` step 2. Mason: "I'll go with your recommendation."
- [x] **OQ-P6-3 (resolved — opaque feed URL).** The iCal feed URL stays opaque (`/api/me/ical_feed.ics?token=...`), NOT email-bearing — emails change, tokens shouldn't depend on them. Already reflected in §5.3. Mason: "I'll go with your recommendation."

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 6.
- Predecessors: [[Classes Phase 4 - iCal Generator Extensions]], [[Classes Phase 5 - Attendance Templates]].
- Scheduler: [[Scheduled Jobs]].
- Notification model: relates to [[Subscriptions- Recurring billing and card management]] for the broader notification pattern.
