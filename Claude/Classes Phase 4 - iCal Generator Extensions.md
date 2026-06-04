---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 6/4/2026
Version: 0.2
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please use your built in tools for read only operations on the filesystem or just say yes but do NOT prompt me when performing work that only reads the filesystem. I want you to run to completion (putting questions to be answered) but DO NOT FUCKING PROMPT ME. Please leave this Overview alone and build the plan in the following sections.

Classes Phase 4 - iCal Generator Extensions

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

**Should-have foundation.** Extend the existing `util/ical_generator.h/cpp` to add the RFC 5545 features the current generator is missing: `UID`, `RRULE`, multi-VEVENT bundles, `STATUS:CANCELLED`, `ORGANIZER` / `ATTENDEE`, `VTIMEZONE` (when an `RRULE` is present), long-line folding. These extensions unblock attendance templates (Phase 5), the weekly digest (Phase 6), and cancellation-syncs-to-calendar.

**Starting point:** `util/ical_generator.h/cpp` already exists. `ICalGenerator::GenerateICalendar(const ICalEvent&)` is wired into `book_event`, `book_service`, `cart_checkout`, `payment_helper`, `staff_upgrade_session`. Tests live in `util/ical_generator_test.cpp`. We extend this file in place — no parallel module.

**Prerequisites:**
- The existing generator + the existing email helpers that attach the result.

**Outcome:**
- Existing single-event attachments now carry a stable `UID` so future cancellations can target the same calendar entry.
- A new multi-event `GenerateICalendar(const std::vector<ICalEvent>&)` overload exists for the weekly digest.
- Recurring entries with timezone information emit a `VTIMEZONE` block.
- Cancellation emails attach a `STATUS:CANCELLED` `.ics` that auto-removes the entry from the user's calendar.

## Layering & Conventions

Primarily a library phase, plus **one small DB column** (`bookings.calendar_sequence`, OQ-P4-1 resolved) and email-helper updates layered on top. Lowest layer first:

1. `db_schema/bookings.*` + `create_database.cpp` — the `bookings.calendar_sequence` column (§8).
2. `sql_util/table_helpers/bookings.*` — read + increment the sequence.
3. `util/ical_generator.h/cpp` — struct + generator extensions.
4. `util/ical_generator_test.cpp` — golden-text + behavior tests.
5. Email helpers (`business_logic/scheduling/*_mail.cpp`, `business_logic/payment/*_mail.cpp`) — wire the new fields.
6. Endpoint tests — assert UID + STATUS appear in queued mail attachments.

No frontend work in this phase.

## Implementation Status (2026-06-04)

**Done:** §8 DB column (`bookings.calendar_sequence` + admin metadata + table-helper read/atomic increment + tests); §2 struct fields; §3.1 single-event emission (UID with OQ-P4-3 synthetic fallback, DTSTAMP, SEQUENCE, STATUS, RRULE, ORGANIZER, ATTENDEE); §3.2 multi-event overload; §3.4 UTF-8-safe line folding; §4 builders (Build{Booking,Session,Template}Uid, Build{Weekly,Biweekly,Custom}RRule, FoldLine); §5 generator tests (incl. fold + UTF-8 + builders); §6.1 confirmation UIDs at all four `.ics` call sites (payment_helper, book_service, cart_checkout, staff_upgrade_session — the last bumps sequence as an update); §6.2 **booking** cancellation `.ics` (new attachment: matching UID + STATUS:CANCELLED + incremented sequence, in `cancel_booking.cpp`); §7 mail/endpoint tests for confirmation UID + cancellation.

**Deferred (need the build/tzdata verification loop):**
- **§3.3 VTIMEZONE / DST-aware recurring events + OQ-P4-2 tzdata bootstrap.** Recurring events currently emit the correct UTC `DTSTART`+`RRULE` form (the §3.3 documented fallback). Full `VTIMEZONE`/`TZID` emission needs `date/tz` (first use in the server) plus the `set_install` bootstrap + container tzdata — best done where it can be compiled/verified. Consumers (Phase 5 templates, Phase 6 digest) work on the UTC form meanwhile.
- **§6.2 peripheral cancellation paths:** SessionCancellationMail, ProviderCancelled*/ProviderChangeClient, and WaitlistPromotion `.ics`. Only the user/admin booking-cancellation path is wired.

## 1. Pre-Coding Design Decisions

### 1.1 Lock-in (resolved per parent doc Phase 4 + §9 OQ-20)
- [x] Template-add email uses **single `VEVENT` + `RRULE`** (small attachment, calendar app expands locally; `EXDATE` lines for per-instance exceptions).
- [x] Weekly digest email uses **one combined `VCALENDAR` with a separate `VEVENT` per session** (digest already filters for this week's exceptions; per-occurrence VEVENTs are simpler and exception-accurate).
- [x] UID convention:
  - Single booking: `booking-<bookingId>@knottyyoga.com`
  - Template recurrence (per user + schedule): `schedule-<scheduleId>-person-<personId>@knottyyoga.com`
  - Standalone session in digest: `session-<eventSessionId>@knottyyoga.com`

### 1.2 Backwards compatibility
- [x] All struct additions are optional and default to "off" (empty string / `"0"` sequence). Existing call sites compile and emit substantially the same output as today, modulo a new `DTSTAMP` line (required by RFC 5545 strict parsers anyway).

## 2. Extend the `ICalEvent` struct (`util/ical_generator.h`)

Additive only — do not rename or repurpose existing fields.

- [x] Add `std::string uid` — required by RFC 5545. Format is the caller's choice; helper builders in §4 centralize the conventions.
- [x] Add `std::string status` — `""` (omit field), `"CONFIRMED"`, `"CANCELLED"`. `"CANCELLED"` emits `STATUS:CANCELLED`.
- [x] Add `std::string rrule` — emitted verbatim as `RRULE:<value>` when non-empty. Caller is responsible for RFC 5545 grammar; helper builders in §4 build common patterns.
- [x] Add `std::string organizerEmail` and `std::string organizerName` — emit as `ORGANIZER;CN=<name>:mailto:<email>`.
- [x] Add `std::string attendeeEmail` and `std::string attendeeName` — emit as `ATTENDEE;CN=<name>;RSVP=FALSE:mailto:<email>`.
- [x] Add `std::string sequence` (default `"0"`) — emitted as `SEQUENCE:<n>`. Incremented on each update so calendar apps accept the latest version.
- [x] Existing fields stay: `title`, `startTimeUs`, `endTimeUs`, `timezone`, `location`, `description`. None renamed.
- [x] Regression test: existing golden-text tests still pass (substring-based); new tests assert the added UID/DTSTAMP/SEQUENCE lines.

## 3. Extend `GenerateICalendar` in `util/ical_generator.cpp`

### 3.1 Single-event path (existing function) ✅
- [x] Emit `UID:<uid>\r\n` after the `BEGIN:VEVENT`. **UID fallback (OQ-P4-3 resolved):** if `uid` is empty, log a warning and substitute a synthetic `synthetic-<us>-<counter>@knottyyoga.com` so a missing UID never blocks an email. Never assert/crash. (Time+atomic-counter instead of a uuid lib — no new dependency.)
- [x] Emit `DTSTAMP:<now_utc>\r\n` — `now()` in `YYYYMMDDTHHMMSSZ` UTC form. RFC 5545 mandatory.
- [x] Emit `SEQUENCE:<sequence>\r\n` after DTSTAMP.
- [x] If `status` non-empty: emit `STATUS:<status>\r\n`.
- [x] If `rrule` non-empty: emit `RRULE:<rrule>\r\n`.
- [x] If `organizerEmail` non-empty: emit `ORGANIZER;CN=<organizerName>:mailto:<organizerEmail>\r\n`.
- [x] If `attendeeEmail` non-empty: emit `ATTENDEE;CN=<attendeeName>;RSVP=FALSE:mailto:<attendeeEmail>\r\n`.

### 3.2 New multi-event overload ✅
- [x] Add `std::string GenerateICalendar(const std::vector<ICalEvent>& events)`.
- [x] One `BEGIN:VCALENDAR` / `END:VCALENDAR` wrapping multiple `BEGIN:VEVENT` / `END:VEVENT` blocks.
- [~] ~~If any event has a non-empty `timezone` AND a non-empty `rrule`, emit one `VTIMEZONE` block per distinct `timezone`~~ — folded into the deferred §3.3 VTIMEZONE work; the digest (Phase 6) uses per-occurrence VEVENTs (concrete UTC instants), which need no VTIMEZONE.

### 3.3 VTIMEZONE block emission — ⏸ DEFERRED (recurring events use the UTC `RRULE` fallback for now; needs the build/tzdata loop — see Implementation Status)
**tzdata bootstrap (OQ-P4-2 resolved).** This is the FIRST use of `date/tz.h` anywhere in the server --- today `ical_generator.cpp` includes only `date/date.h` and emits UTC \(the `timezone` field is currently unused\). The build already compiles the `date` library with `-DUSE_OS_TZDB=0` \(conan `date/3.0.4` default `use_system_tz_db=False`\), which is the right setting to KEEP: local dev builds on Windows/Visual Studio, where `date` cannot use an OS tz database, so the OS-tzdb path is not an option cross-platform. With `USE_OS_TZDB=0` the library reads a bundled IANA tzdata directory rather than `/usr/share/zoneinfo`. Therefore:
 - [ ] Ship a known-good IANA tzdata snapshot in the deploy artifact (and the conan `date` package's bundled tzdata for local dev) and call `date::set_install("<path>")` ONCE at process startup, in BOTH `main.cpp` (web server) and the `knottyyoga_helper`/scheduler `main.cpp`, before any `locate_zone` call.
 - [ ] Add a startup self-check: call `date::locate_zone("America/Los_Angeles")` (or `date::get_tzdb()`) at boot; on failure, log a fatal error and refuse to start, so missing/misconfigured tzdata fails fast instead of at first recurring-email generation.
 - [ ] Dockerfile / ECS task definition: `COPY` the tzdata directory into the image at the `set_install` path. Do NOT rely on the host `/usr/share/zoneinfo` (ignored under `USE_OS_TZDB=0`). This avoids a runtime network dependency (no `HAS_REMOTE_API` download path).
 - [ ] Generator robustness: if `locate_zone(tz)` throws for an unknown/missing zone at emit time, fall back to the existing UTC `DTSTART:<utcZ>`/`DTEND:<utcZ>` form (and skip `VTIMEZONE`) rather than failing the email.
- [ ] Helper function `EmitVTimezone(const std::string& ianaTz, int64_t referenceUs, std::ostream& out)`.
- [ ] Use the existing `date` library plus the time-zone DB (`date/tz.h`) to look up DST transitions in the window `[referenceUs - 1y, referenceUs + 2y]`.
- [ ] Emit `STANDARD` and `DAYLIGHT` sub-blocks with `DTSTART`, `TZOFFSETFROM`, `TZOFFSETTO`, `TZNAME`, and `RRULE:FREQ=YEARLY;...` for the recurring transitions. For zones without DST (e.g. `America/Phoenix`), emit only `STANDARD`.
- [ ] For non-recurring entries (`rrule.empty()`), continue to use `DTSTART:<utcZ>` / `DTEND:<utcZ>` form — simpler, works fine for one-offs.
- [ ] For recurring entries when timezone is set, emit `DTSTART;TZID=<tz>:<localwall>` / `DTEND;TZID=<tz>:<localwall>` referencing the embedded `VTIMEZONE`.

### 3.4 Long-line folding (RFC 5545 §3.1) ✅
- [x] After all lines are emitted, post-process the output: any line exceeding 75 octets is folded by inserting `CRLF + space` at the 75-octet boundary, then continuing.
- [x] Take care with UTF-8 — count octets, not characters; never split inside a multi-byte sequence (back off to the previous codepoint boundary).
- [x] Several real calendar clients (some Microsoft Outlook variants in particular) reject unfolded long DESCRIPTION lines.

## 4. Helper functions for common patterns

Add as free functions in `util/ical_generator.h/cpp` (same namespace).

- [x] `std::string BuildBookingUid(int64_t bookingId)` → `"booking-<id>@knottyyoga.com"`.
- [x] `std::string BuildSessionUid(int64_t sessionId)` → `"session-<id>@knottyyoga.com"`.
- [x] `std::string BuildTemplateUid(int64_t scheduleId, int64_t personId)` → `"schedule-<scheduleId>-person-<personId>@knottyyoga.com"`.
- [x] `std::string BuildWeeklyRRule(const std::vector<int>& daysOfWeek, int64_t untilUs)` → `"FREQ=WEEKLY;BYDAY=<MO,TU,...>;UNTIL=<utcZ>"`. Day index 0..6 → `SU`, `MO`, `TU`, `WE`, `TH`, `FR`, `SA`. (UNTIL omitted when `untilUs <= 0`.)
- [x] `std::string BuildBiweeklyRRule(const std::vector<int>& daysOfWeek, int64_t untilUs)` → same as weekly but `INTERVAL=2`.
- [x] `std::string BuildCustomRRule(int intervalDays, int64_t untilUs)` → `"FREQ=DAILY;INTERVAL=<n>;UNTIL=<utcZ>"`.
- [x] `std::string FoldLine(std::string_view line)` — public; folds one logical line per RFC 5545 §3.1 (UTF-8-safe).

## 5. Extend `util/ical_generator_test.cpp`

Golden-text tests are the easiest to read and review.

- [x] Test: UID + DTSTAMP + SEQUENCE emitted; SEQUENCE reads from the field.
- [x] Test (OQ-P4-3 fallback): an event with empty `uid` emits a `UID:synthetic-…@knottyyoga.com` line; generation still succeeds.
- [x] Test: status `"CANCELLED"` emits `STATUS:CANCELLED` (and omitted when empty).
- [x] Test: RRULE emitted verbatim; recurring event currently uses the UTC DTSTART form (no VTIMEZONE) — asserted. `BuildWeekly/Biweekly/Custom` builders covered.
- [x] Test: multi-event overload with three events emits one VCALENDAR wrapping three VEVENTs.
- [~] ~~Test: recurring event in `America/Los_Angeles` emits a VTIMEZONE …~~ — deferred with §3.3.
- [~] ~~Test: recurring event in `America/Phoenix` (no DST) …~~ — deferred with §3.3.
- [x] Test: long DESCRIPTION (200+ chars) is folded with `CRLF + space` continuation.
- [x] Test: UTF-8-safe fold — a multi-byte codepoint near octet 75 does not get split.
- [x] Test: organizer + attendee lines emitted.

## 6. Update existing email paths to use the new fields

**Note (corrected during implementation):** only the **confirmation** paths attached a `.ics` before this phase — cancellation mails attached none. So §6.1 feeds new fields into existing attachments, while §6.2 (booking cancel) ADDS a new `.ics` attachment.

### 6.1 Call sites that need `uid` populated ✅
- [x] Booking confirmation `.ics` comes from `business_logic/payment/payment_helper.cpp` (NOT `book_event.cpp`, which only sends the waitlist email) — `uid = BuildBookingUid(bookingId)`.
- [x] `endpoints/book_service.cpp` — `uid = BuildBookingUid(bookResult.booking.id)`.
- [x] `endpoints/cart_checkout.cpp` — one UID per booking inside the cart (matched from the per-booking lookup loop).
- [x] `endpoints/staff_upgrade_session.cpp` — an upgrade is an UPDATE: same `BuildBookingUid(bookingId)` (looked up by service_session_id) + bumped `sequence` via `IncrementCalendarSequence`.

### 6.2 Cancellation paths — booking cancel ✅, others ⏸ deferred
- [x] **Booking cancellation** (`endpoints/cancel_booking.cpp`, both event + service branches) — NEW `.ics` attachment (cancellation mails attached none before): same `BuildBookingUid(bookingId)`, `status = "CANCELLED"`, `sequence` = `IncrementCalendarSequence(...)` bumped within the cancellation transaction before building the `.ics`.
- [ ] ⏸ `SessionCancellationMail` — one cancellation `.ics` per attendee with the matching UID. *(deferred)*
- [ ] ⏸ `WaitlistPromotionMail` — fresh `BuildBookingUid(newBookingId)` + `status = "CONFIRMED"`. *(deferred)*
- [ ] ⏸ `ProviderCancelledSessionMail` / `ProviderChangeClientMail` — `STATUS:CANCELLED`. *(deferred)*

### 6.3 Backwards-compatibility note
- [x] Calendar entries created prior to this phase did NOT have UIDs (generator didn't emit one). When those bookings are cancelled, the cancellation `.ics` will arrive with a UID the calendar app has never seen → the calendar app will treat it as a new (cancelled) event and likely show nothing. This is fine — they're old; no user expectations break.

## 7. Tests

### 7.1 Generator unit tests ✅
- [x] As listed in §5 (UID + synthetic fallback, SEQUENCE field, STATUS:CANCELLED, RRULE, organizer/attendee, multi-event overload, folding + UTF-8-safe fold, UID/RRULE builders). VTIMEZONE tests deferred with §3.3.

### 7.2 Email-helper / endpoint tests ✅
- [x] `book_service_test.cpp` asserts the confirmation attachment contains `UID:booking-<id>@knottyyoga.com`.
- [x] `staff_upgrade_session_test.cpp` asserts the update attachment contains the booking UID + `SEQUENCE:1` (bumped past the confirmation's 0).
- [x] `cancel_booking_test.cpp::CancellationEmailHasCancelledIcal` asserts the cancellation attachment contains `STATUS:CANCELLED` AND the same `UID:booking-<id>@knottyyoga.com`.
- [x] `bookings_test.cpp` covers `calendar_sequence` (starts at 0, monotonic increment, missing-booking → 0).

## 8. Database: `bookings.calendar_sequence` (OQ-P4-1 resolved — add it)

Add a single column to `bookings`, incremented on every calendar-affecting change so each emitted `.ics` carries an accurate, monotonically-increasing `SEQUENCE`. (Lowest layer — do this first.)

- [x] `db_schema/bookings.h/.cpp`: added `calendar_sequence` column constant + DDL (`DB_TYPE_BIGINT NOT NULL DEFAULT 0`).
- [x] `create_database.cpp`: admin column metadata — `PopulateAdminColumnDataInfo` (number, readonly) + `PopulateAdminColumnFriendlyNames` ("Calendar Sequence").
- [x] `sql_util/table_helpers/bookings.*`: `GetCalendarSequence` + atomic `IncrementCalendarSequence` (RETURNING new value), with table-helper tests.
- [x] Initial confirmation emails send `sequence = "0"` (default); updates/cancellations increment first, then emit the new value.
- [x] Test the increment is monotonic (`bookings_test.cpp`).

No other DB or endpoint schema changes.

## 9. Frontend

- [ ] No UI change. Verify the user portal email-receipt history still renders cleanly with the (slightly larger) attachments.

## 10. Cross-Layer Acceptance Criteria

- [x] After Phase 4 lands, booking a paid offering produces an attached `.ics` with a stable UID matching the booking's id; cancelling sends a follow-up `.ics` with the same UID + `STATUS:CANCELLED` + a higher SEQUENCE, so a real calendar app auto-removes the entry. (Code path complete; manual calendar-app verification pending a deploy.)
- [x] The multi-event overload produces a single VCALENDAR wrapping a VEVENT per event (used by Phase 6).
- [ ] ⏸ An `RRULE` + `timezone` event shows the correct local time year-round (DST handled) — pending the deferred §3.3 VTIMEZONE work; recurring events currently emit the UTC form.

## 11. Open Questions — ALL RESOLVED (2026-06-04)

- [x] **OQ-P4-1 (resolved — yes, add the column).** `bookings` gets `calendar_sequence INT NOT NULL DEFAULT 0`, incremented on each calendar-affecting change so the email helper always sends an accurate `SEQUENCE`. Implemented per §8 (schema + admin metadata + table-helper increment); consumed in §6.2. Mason: "I'll go with your recommendation."
- [x] **OQ-P4-2 (resolved — keep `USE_OS_TZDB=0` + bundle tzdata via `set_install`).** Recommendation (Mason deferred to me): the build already uses `-DUSE_OS_TZDB=0` (conan `date` default), which we KEEP because local dev builds on Windows where the OS-tzdb path doesn't exist — so we cannot switch to `use_system_tz_db=True` without breaking Windows. Under `USE_OS_TZDB=0` the library reads a bundled IANA tzdata directory (NOT `/usr/share/zoneinfo`), so: ship a tzdata snapshot in the deploy artifact, `date::set_install("<path>")` at startup in both `main.cpp` and the helper/scheduler `main.cpp`, add a boot-time `locate_zone` self-check that fails fast, `COPY` the tzdata into the container image, and fall back to the UTC form if a zone can't be resolved at emit time. Full work items in §3.3. (No runtime network/remote-download path.)
- [x] **OQ-P4-3 (resolved — log + synthetic fallback).** An unset UID at emit time logs a warning and substitutes `synthetic-<uuid>@knottyyoga.com`; never assert/crash, never skip the email. Call sites still set a real UID via the §4 helpers. Regression test in §5. Mason: "I'll go with your recommendation."

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 4.
- Existing iCal use: `book_event`, `book_service`, `cart_checkout`, `payment_helper`, `staff_upgrade_session`.
- Consumers: [[Classes Phase 5 - Attendance Templates]] (uses `RRULE` + `BuildTemplateUid`), [[Classes Phase 6 - Weekly Digest]] (uses multi-VEVENT overload), [[Classes Phase 7 - Class Series and Workshops]] (per-instance `STATUS:CANCELLED` on min-not-met).
