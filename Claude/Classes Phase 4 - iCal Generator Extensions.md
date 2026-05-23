---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/23/2026
Version: 0.1
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

This is a pure-library phase (no DB, no endpoints) with email-helper updates layered on top:

1. `util/ical_generator.h/cpp` — struct + generator extensions.
2. `util/ical_generator_test.cpp` — golden-text + behavior tests.
3. Email helpers (`business_logic/scheduling/*_mail.cpp`, `business_logic/payment/*_mail.cpp`) — wire the new fields.
4. Endpoint tests — assert UID + STATUS appear in queued mail attachments.

No frontend work in this phase.

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

- [ ] Add `std::string uid` — required by RFC 5545. Format is the caller's choice; helper builders in §4 centralize the conventions.
- [ ] Add `std::string status` — `""` (omit field), `"CONFIRMED"`, `"CANCELLED"`. `"CANCELLED"` emits `STATUS:CANCELLED`.
- [ ] Add `std::string rrule` — emitted verbatim as `RRULE:<value>` when non-empty. Caller is responsible for RFC 5545 grammar; helper builders in §4 build common patterns.
- [ ] Add `std::string organizerEmail` and `std::string organizerName` — emit as `ORGANIZER;CN=<name>:mailto:<email>`.
- [ ] Add `std::string attendeeEmail` and `std::string attendeeName` — emit as `ATTENDEE;CN=<name>;RSVP=FALSE:mailto:<email>`.
- [ ] Add `std::string sequence` (default `"0"`) — emitted as `SEQUENCE:<n>`. Incremented on each update so calendar apps accept the latest version.
- [ ] Existing fields stay: `title`, `startTimeUs`, `endTimeUs`, `timezone`, `location`, `description`. None renamed.
- [ ] Regression test: an all-default `ICalEvent` produces the same byte sequence as today plus a new `DTSTAMP` line. (Update the existing golden-text tests accordingly.)

## 3. Extend `GenerateICalendar` in `util/ical_generator.cpp`

### 3.1 Single-event path (existing function)
- [ ] Emit `UID:<uid>\r\n` after the `BEGIN:VEVENT`.
- [ ] Emit `DTSTAMP:<now_utc>\r\n` — `now()` in `YYYYMMDDTHHMMSSZ` UTC form. RFC 5545 mandatory.
- [ ] Emit `SEQUENCE:<sequence>\r\n` after DTSTAMP.
- [ ] If `status` non-empty: emit `STATUS:<status>\r\n`.
- [ ] If `rrule` non-empty: emit `RRULE:<rrule>\r\n`.
- [ ] If `organizerEmail` non-empty: emit `ORGANIZER;CN=<organizerName>:mailto:<organizerEmail>\r\n`.
- [ ] If `attendeeEmail` non-empty: emit `ATTENDEE;CN=<attendeeName>;RSVP=FALSE:mailto:<attendeeEmail>\r\n`.

### 3.2 New multi-event overload
- [ ] Add `std::string GenerateICalendar(const std::vector<ICalEvent>& events)`.
- [ ] One `BEGIN:VCALENDAR` / `END:VCALENDAR` wrapping multiple `BEGIN:VEVENT` / `END:VEVENT` blocks.
- [ ] If any event has a non-empty `timezone` AND a non-empty `rrule`, emit one `VTIMEZONE` block per distinct `timezone` value before the first VEVENT (de-dup the set).

### 3.3 VTIMEZONE block emission
- [ ] Helper function `EmitVTimezone(const std::string& ianaTz, int64_t referenceUs, std::ostream& out)`.
- [ ] Use the existing `date` library plus the time-zone DB (`date/tz.h`) to look up DST transitions in the window `[referenceUs - 1y, referenceUs + 2y]`.
- [ ] Emit `STANDARD` and `DAYLIGHT` sub-blocks with `DTSTART`, `TZOFFSETFROM`, `TZOFFSETTO`, `TZNAME`, and `RRULE:FREQ=YEARLY;...` for the recurring transitions. For zones without DST (e.g. `America/Phoenix`), emit only `STANDARD`.
- [ ] For non-recurring entries (`rrule.empty()`), continue to use `DTSTART:<utcZ>` / `DTEND:<utcZ>` form — simpler, works fine for one-offs.
- [ ] For recurring entries when timezone is set, emit `DTSTART;TZID=<tz>:<localwall>` / `DTEND;TZID=<tz>:<localwall>` referencing the embedded `VTIMEZONE`.

### 3.4 Long-line folding (RFC 5545 §3.1)
- [ ] After all lines are emitted, post-process the output: any line exceeding 75 octets is folded by inserting `CRLF + space` at the 75-octet boundary, then continuing.
- [ ] Take care with UTF-8 — count octets, not characters; never split inside a multi-byte sequence (back off to the previous codepoint boundary).
- [ ] Several real calendar clients (some Microsoft Outlook variants in particular) reject unfolded long DESCRIPTION lines.

## 4. Helper functions for common patterns

Add as free functions in `util/ical_generator.h/cpp` (same namespace).

- [ ] `std::string BuildBookingUid(int64_t bookingId)` → `"booking-<id>@knottyyoga.com"`.
- [ ] `std::string BuildSessionUid(int64_t sessionId)` → `"session-<id>@knottyyoga.com"`.
- [ ] `std::string BuildTemplateUid(int64_t scheduleId, int64_t personId)` → `"schedule-<scheduleId>-person-<personId>@knottyyoga.com"`.
- [ ] `std::string BuildWeeklyRRule(const std::vector<int>& daysOfWeek, int64_t untilUs)` → `"FREQ=WEEKLY;BYDAY=<MO,TU,...>;UNTIL=<utcZ>"`. Day index 0..6 → `SU`, `MO`, `TU`, `WE`, `TH`, `FR`, `SA`.
- [ ] `std::string BuildBiweeklyRRule(const std::vector<int>& daysOfWeek, int64_t untilUs)` → same as weekly but `INTERVAL=2`.
- [ ] `std::string BuildCustomRRule(int intervalDays, int64_t untilUs)` → `"FREQ=DAILY;INTERVAL=<n>;UNTIL=<utcZ>"`.
- [ ] `std::string FoldLine(std::string_view line)` (test-only helper if useful) — folds one logical line per RFC 5545 §3.1.

## 5. Extend `util/ical_generator_test.cpp`

Golden-text tests are the easiest to read and review.

- [ ] Test: an all-default event (regression). Expect the today-equivalent output plus the new `DTSTAMP`, `UID:""` (omitted if empty — pick policy), `SEQUENCE:0`. Decide: omit UID line when empty OR emit `UID:` blank line? Spec says UID is mandatory. Recommend: REJECT generation with empty UID via a `assert`/log warning so we catch unset UIDs at dev time. Update existing call sites to always set UID via helpers.
- [ ] Test: status `"CANCELLED"` emits `STATUS:CANCELLED`.
- [ ] Test: weekly RRULE: `BuildWeeklyRRule({2}, until)` → expected RRULE string; full output contains exactly one `RRULE:` line.
- [ ] Test: multi-event overload with three events emits one VCALENDAR wrapping three VEVENTs.
- [ ] Test: recurring event in `America/Los_Angeles` emits a `VTIMEZONE` block with STANDARD+DAYLIGHT, and the VEVENT's DTSTART uses `;TZID=America/Los_Angeles:<local>`.
- [ ] Test: recurring event in `America/Phoenix` (no DST) emits a VTIMEZONE with only STANDARD.
- [ ] Test: long DESCRIPTION (200+ chars) is folded with `CRLF + space` continuation.
- [ ] Test: UTF-8-safe fold — a multi-byte codepoint near octet 75 does not get split.

## 6. Update existing email paths to use the new fields

**Note:** confirmation / cancellation mails already attach `.ics` today via `GenerateICalendar`. This subsection is about feeding them the new struct fields, not bolting on attachments from scratch.

### 6.1 Call sites that need `uid` populated
- [ ] `endpoints/book_event.cpp` (or wherever it queues the confirmation email) — `uid = BuildBookingUid(bookingId)`.
- [ ] `endpoints/book_service.cpp`.
- [ ] `endpoints/cart_checkout.cpp` — one UID per booking inside the cart.
- [ ] `business_logic/payment/payment_helper.cpp` — anywhere it queues a `.ics` for a fresh booking.
- [ ] `endpoints/staff_upgrade_session.cpp`.

### 6.2 Cancellation paths
- [ ] `BookingCancellationMail` — emit a CANCELLED `.ics` with:
  - same UID as the original booking confirmation (so the calendar client matches it)
  - `status = "CANCELLED"`
  - `sequence = "1"` (or higher; centralize via a `bookings.calendar_sequence` column if we expect multiple updates — see open questions)
- [ ] `SessionCancellationMail` — one cancellation `.ics` per attendee with the matching UID.
- [ ] `WaitlistPromotionMail` — fresh `BuildBookingUid(newBookingId)` + `status = "CONFIRMED"`.
- [ ] `ProviderCancelledSessionMail` / `ProviderChangeClientMail` (provider-side cancellation paths) — `STATUS:CANCELLED` where appropriate.

### 6.3 Backwards-compatibility note
- [ ] Calendar entries created prior to this phase did NOT have UIDs (generator didn't emit one). When those bookings are cancelled, the cancellation `.ics` will arrive with a UID the calendar app has never seen → the calendar app will treat it as a new (cancelled) event and likely show nothing. This is fine — they're old; no user expectations break.

## 7. Tests

### 7.1 Generator unit tests
- [ ] As listed in §5.

### 7.2 Email-helper tests
- [ ] Update existing tests in `business_logic/scheduling/booking_confirmation_mail_test.cpp` (and friends): assert the queued mail body's attachment contains the expected `UID:<booking-id>@knottyyoga.com` substring.
- [ ] Add new tests for the cancellation path: confirm the attachment contains `STATUS:CANCELLED` and the same UID as the original confirmation.

### 7.3 Endpoint tests
- [ ] `book_event_test.cpp` already covers the success path. Add an assertion that the captured outbound mail has a `text/calendar` attachment with a non-empty UID.
- [ ] Cancellation endpoint test asserts the `STATUS:CANCELLED` attachment.

## 8. No DB / no endpoint changes

- [ ] Confirm we don't need a `bookings.calendar_sequence` column for Phase 4. If a single update beyond the initial confirmation suffices, `sequence = "1"` for any update is enough. If we expect multiple updates (rescheduled booking → cancelled → re-confirmed → re-cancelled), add the column. See open questions.

## 9. Frontend

- [ ] No UI change. Verify the user portal email-receipt history still renders cleanly with the (slightly larger) attachments.

## 10. Cross-Layer Acceptance Criteria

- [ ] After Phase 4 lands, booking a paid offering (workshop, series instance, intro workshop) produces an attached `.ics` that includes a stable UID matching the booking's id. Cancelling the same booking sends a follow-up `.ics` with the same UID + `STATUS:CANCELLED`, and a real calendar app (Apple Calendar / Google Calendar) shows the event auto-disappearing after the second attachment is processed.
- [ ] Sending a multi-event bundle via the new overload (used by Phase 6) produces a single VCALENDAR that Apple/Google parse without errors.
- [ ] An event with `RRULE` + `timezone=America/Los_Angeles` shows the correct local time year-round in a calendar app (i.e. DST transitions are handled).

## 11. Open Questions

- **OQ-P4-1.** Should `bookings` get a `calendar_sequence INT NOT NULL DEFAULT 0` column we increment on each update, so the email helper always sends an accurate `SEQUENCE`? Recommended: yes — small risk to add the column; saves a class of "calendar client ignored my update because the sequence didn't move" bugs.
- **OQ-P4-2.** Where should the `date` library's tzdata live in production? If the tz-aware library reads tzdata files from disk, ensure the AWS task definition mounts /usr/share/zoneinfo or includes the equivalent. Verify the existing service does this for the scheduler.
- **OQ-P4-3.** For unset UID at call sites — assert + crash, log + skip emission, or fall back to a synthetic random UID? Recommended: log + fall back to `synthetic-<uuid>@knottyyoga.com` so a missing UID never blocks an email. Add a regression test for the fallback.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 4.
- Existing iCal use: `book_event`, `book_service`, `cart_checkout`, `payment_helper`, `staff_upgrade_session`.
- Consumers: [[Classes Phase 5 - Attendance Templates]] (uses `RRULE` + `BuildTemplateUid`), [[Classes Phase 6 - Weekly Digest]] (uses multi-VEVENT overload), [[Classes Phase 7 - Class Series and Workshops]] (per-instance `STATUS:CANCELLED` on min-not-met).
