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
- [ ] `db_schema/user_notification_preferences.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL UNIQUE REFERENCES people(id)`
  - `weekly_digest_enabled BOOLEAN NOT NULL DEFAULT TRUE`
  - `digest_send_dow INT NOT NULL DEFAULT 0`  (0=Sunday..6=Saturday)
  - `digest_send_hour_local INT NOT NULL DEFAULT 12`  (0..23, local to user's facility)
  - `last_digest_sent_us BIGINT NOT NULL DEFAULT 0`
  - `created_us`, `updated_us`

### 2.2 `ical_feed_tokens` table
- [ ] `db_schema/ical_feed_tokens.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL UNIQUE REFERENCES people(id)`
  - `token_hash TEXT NOT NULL`  — Argon2id or SHA-256 hash of the random token (consistent with `sessions.token_hash` pattern used elsewhere)
  - `created_us`, `last_used_us`, `revoked_us` (nullable)
- [ ] Unique on `person_id` keeps it simple: one active token at a time. Regeneration = update the hash + invalidate the prior.

### 2.3 Wire into DB init
- [ ] `make_database_info.cpp` + `create_database.cpp` `CreateTables()`.
- [ ] CMakeLists.

## 3. Table Helpers

### 3.1 `TableHelpers::UserNotificationPreferences`
- [ ] `GetOrCreateForPerson(Transaction&, personId)` — idempotent fetch-or-create with defaults.
- [ ] `UpdatePreferences(Transaction&, personId, const KeyValueTable& updates)`.
- [ ] `SetLastDigestSent(Transaction&, personId, sentUs)` — idempotency mutator.
- [ ] `GetUsersDueForDigest(Transaction&, asOfUs)` → list of `personId` whose local Sunday-noon has just passed and `last_digest_sent_us < <this Sunday's local midnight in UTC>`.
- [ ] Tests for fetch-or-create, update, due-list correctness.

### 3.2 `TableHelpers::ICalFeedTokens`
- [ ] `GetOrCreateForPerson(Transaction&, personId)` — returns existing if not revoked, else creates.
- [ ] `RegenerateToken(Transaction&, personId)` → returns the new raw token (only time it's exposed in cleartext).
- [ ] `LookupPersonByToken(Transaction&, tokenString)` → personId or 0. Hashes the input and looks up.
- [ ] `RecordUse(Transaction&, personId)` updates `last_used_us`.
- [ ] Tests, including hash-mismatch returning 0.

## 4. Business Logic

### 4.1 `WeeklyDigestHelper`
Files: `business_logic/scheduling/weekly_digest_helper.h/.cpp/_test.cpp`.

- [ ] `struct WeeklyDigestRow { int64_t sessionStartUs; int64_t sessionEndUs; std::string className; std::string facilityName; std::string roomName; std::vector<std::string> instructorNames; std::string subtitle; /* "Included — your template", "Paid: 6-Week Aerial 101", "One-off addition", "Guest pass for Friend" */ }`.
- [ ] `struct WeeklyDigestData { int64_t personId; std::string personFirstName; std::string personEmail; std::string ianaTz; std::vector<WeeklyDigestRow> rows; }`.
- [ ] `WeeklyDigestData BuildDigestForPerson(Transaction&, int64_t personId, int64_t weekStartUs)`. Algorithm:
  1. Resolve user's primary facility timezone (or the first facility if multi).
  2. Compute `weekStartUs` as the Monday 00:00 in that TZ; `weekEndUs` as Sunday 23:59:59 in that TZ.
  3. Templated occurrences: `GetTemplateOccurrencesForWeek(personId, weekStartUs)` — derives the week's occurrences for each template-entry slot (via `ClassScheduleHelper::GetDerivedSessionsForRange`), drops `attendance_template_exceptions.attending=false`, includes one-off `attending=true` additions. (NO `bookings` / `event_sessions` join for these — they're booking-less.)
  4. Add paid bookings: `bookings WHERE person_id=? AND purchase_id IS NOT NULL AND session start in window AND status IN ('confirmed', 'waitlisted')` (these have persisted `event_sessions`). De-dupe against step 3 by (`class_schedule_slot_id`, `occurrence_date_us`).
  5. **(OQ-P6-1 — yes)** Add upcoming paid services / events the user has booked (existing `BookingHelper::GetBookingsForPerson` upcoming filter), same surface, each row subtitled e.g. "Massage with Provider X — 60min".
  6. Sort by `sessionStartUs`.
  7. Return.
- [ ] `bool SendDigestForPerson(Transaction&, MailHelper*, int64_t personId, int64_t weekStartUs)`:
  1. Build the data.
  2. **(OQ-P6-2 — no empty digests)** If `rows.empty()` → skip (don't send empty digests; don't pester a disengaged user).
  3. Format the HTML body via `FormatString` with a template constant; wrap with `NormalizeCrLf`.
  4. Build the combined iCal: vector of `ICalEvent` (one per row). Paid-booking rows use `BuildBookingUid(bookingId)`; templated booking-less rows use `BuildTemplateUid(classScheduleSlotId, personId)` + occurrence date. Call `GenerateICalendar(events)` from Phase 4.
  5. Queue email with `.ics` attachment.
  6. Update `UserNotificationPreferences::SetLastDigestSent(personId, now)`.
  7. Sync SQL before any `ThreadPool::Queue` (per memory `feedback_sync_sql_before_threadpool_queue.md`).
- [ ] `int SendPendingDigests(Transaction&, MailHelper*, int64_t asOfUs)`:
  - Calls `UserNotificationPreferences::GetUsersDueForDigest(asOfUs)`.
  - For each user, computes the relevant `weekStartUs` and calls `SendDigestForPerson`.
  - Returns the count sent.
- [ ] Tests with `TestMailHelper`:
  - Empty digest skipped.
  - Templated session + one-off addition + paid booking all appear with correct subtitles.
  - Exception-skip removes a templated session.
  - `last_digest_sent_us` updated → re-running `SendPendingDigests(sameAsOfUs)` sends zero new mails.
  - Multi-VEVENT iCal attachment has the right count + UIDs.

### 4.2 `PersonalICalFeedHelper`
- [ ] `business_logic/scheduling/personal_ical_feed_helper.h/.cpp/_test.cpp`.
- [ ] `std::string GenerateFeedForPerson(Transaction&, int64_t personId, int64_t windowStartUs, int64_t windowEndUs)`:
  - Reuses `BuildDigestForPerson` logic to assemble the row set, but over a longer window (default: today + 90 days).
  - Returns the multi-VEVENT iCal text.
- [ ] Headers to set on the response: `Content-Type: text/calendar; charset=utf-8`, `X-PUBLISHED-TTL: PT1H`, `Cache-Control: max-age=3600`, `Refresh-Interval;value=PT1H`.

### 4.3 Email template
- [ ] `business_logic/scheduling/weekly_digest_mail.h/.cpp` — template constants for HTML body and plain-text fallback. Use `FormatString` with `{placeholder}` substitution per CLAUDE.md.
- [ ] Plain-text section lists rows in `Tue Mar 5  6:00–7:00pm  Vinyasa Flow (Studio A) — Instructor Sara` format.
- [ ] Test: golden-text comparison.

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
