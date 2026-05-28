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

Classes Phase 5 - Attendance Templates

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

**Should-have, core engagement.** Users mark which class schedules they normally plan to attend → that's their **attendance template** (per-schedule-entry, not per-instance). Templates are a *personal fitness-planning tool*, NOT a reservation: they do NOT create `booking` rows, do NOT consume `event_sessions.capacity`, and do NOT participate in the waitlist. They drive the user homepage today-classes feed, the weekly digest (Phase 6), and per-instance exception notes that surface to the instructor.

**Why this scope:** Mason explicitly framed templates as "fitness goals" — "I plan to work out Monday, Thursday, Saturday on a good week." Bookings come into existence only when (a) staff checks the user in at the door (Phase 8), or (b) the user paid for a workshop / series / intro / guest pass.

**Prerequisites:**
- Phase 1 (class catalog + the three-level `classes` → `class_instances` → `class_schedules` → `class_schedule_slots` model, lazy-derived sessions).
- Phase 2 (membership-gated visibility — `GetClassesVisibleToPerson`).
- Phase 3 (skill levels — the eligible-classes grid is filtered by skill requirements).
- Phase 4 (iCal RRULE support — template-add email uses single VEVENT + RRULE).

> ### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]]
> The redesign makes this phase strictly *simpler* and confirms its no-bookings stance, but changes the binding key:
> - **Template entries bind to `class_schedule_slot_id`, not `class_schedule_id`.** A slot (day-of-week + start time + room + instructor under an impl) has stable identity; the impl is a versioned container that changes over time. "I attend the Monday 6pm Knotty Yoga" = a template entry on that slot.
> - **There is no materialization hook.** Sessions are derived on the fly (lazy model, OQ-CSI-12). The homepage / digest / calendar compute "is this on my template?" by evaluating the user's template-entry slots against the **derived** occurrences for the week — NOT by joining persisted `event_sessions` rows (most have none). Delete §4.4's "session-materialization hook" entirely.
> - **Exceptions key off (`class_schedule_slot_id`, `occurrence_date_us`)**, not `event_session_id` — the occurrence usually has no persisted row.
> - **Stale-slot handling:** if a slot referenced by a template entry is deleted (impl edit / product migration), surface it in the user portal as "this slot no longer exists — pick a new one".
> - Everything else (no bookings, no capacity, no waitlist) is unchanged and now even cleaner — there are no soft-booking rows to manage on impl change (Mason's §140 note).

**Outcome:**
- User can view a weekly grid of classes they're eligible to attend, mark/unmark template entries.
- Adding a template entry sends a single confirmation email with a recurring `RRULE`-based `.ics`.
- Per-instance exceptions: "I'm out this Tuesday" with an optional note for the instructor (surfaces via staff portal + N-7 daily digest).
- Per-instance additions: "I'll attend this Thursday too" (one-off).
- Homepage today-classes feed shows checkmarks for templated/added today, unchecked rows for eligible-but-unmarked classes; one click flips state.

## Layering & Conventions

Lowest layer first:

1. `db_schema/` — three new tables.
2. `sql_util/table_helpers/` — three new helpers.
3. `business_logic/scheduling/` — `AttendanceTemplateHelper`.
4. `endpoints/` — six new endpoints.
5. Angular: My Schedule page, calendar overlay, homepage today-classes feed, per-instance exception dialog.
6. Admin metadata.
7. Tests at every layer.

## 1. Pre-Coding Design Decisions

### 1.1 Locked-in semantics (resolved per parent doc §2.8)
- [x] Template = personal fitness-planning, NOT a booking. NO `booking` rows created from templates.
- [x] NO `event_sessions.capacity` consumption from templates.
- [x] NO waitlist participation from templates.
- [x] Templates ONLY apply to membership-included recurring classes (NOT paid workshops / series — those are explicit bookings).

### 1.2 Forward sync
- [x] Adding a template entry walks forward through already-materialized future sessions but DOES NOT create bookings. It records the schedule-level template entry; the homepage and digest compute "is this on my template?" by joining `event_sessions.class_schedule_id → attendance_template_entries`.
- [x] Removing a template entry is just a row delete. No cascading booking cancellation (there were no bookings).

### 1.3 Per-instance exception semantics
- [x] Exception = "I won't attend this Tuesday" (attending=false) OR "I will attend this Thursday in addition to my template" (attending=true). One row covers both.
- [x] Exception rows have an optional note routed to the instructor.
- [x] Same-day predecessor (SL-11) auto-cancel: if a user cancels a paid booking on the predecessor class, the dependent booking is silently cancelled (SL-11 lives in `BookingHelper`, not in this phase — but the homepage display in this phase must respect it for consistency).

### 1.4 Email cadence
- [x] On template add: one confirmation email with recurring RRULE iCal (Phase 4 BuildWeeklyRRule).
- [x] On template remove: NO email (silent).
- [x] On exception (skip): NO email; the note goes to the instructor's staff portal feed + daily digest per N-7.
- [x] On exception (one-off add): NO email; shows up in the next Sunday digest (Phase 6).

## 2. Database Schema

### 2.1 `attendance_templates` table
- [ ] `db_schema/attendance_templates.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL UNIQUE REFERENCES people(id)`  — one template per person
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us`, `updated_us`

### 2.2 `attendance_template_entries` table
- [ ] `db_schema/attendance_template_entries.h/.cpp`:
  - `id BIGSERIAL PK`
  - `template_id BIGINT NOT NULL REFERENCES attendance_templates(id)`
  - `class_schedule_id BIGINT NOT NULL REFERENCES class_schedules(id)`
  - `created_us`
  - `UNIQUE (template_id, class_schedule_id)`
- [ ] Index on (`template_id`) for "all my entries" reads.
- [ ] Index on (`class_schedule_id`) for "who has this on their template?" reads (instructor view, materialization hook in §4.4).

### 2.3 `attendance_template_exceptions` table
- [ ] `db_schema/attendance_template_exceptions.h/.cpp`:
  - `id BIGSERIAL PK`
  - `template_id BIGINT NOT NULL REFERENCES attendance_templates(id)`
  - `event_session_id BIGINT NOT NULL REFERENCES event_sessions(id)`
  - `attending BOOLEAN NOT NULL` — false = "skip this instance"; true = "one-off add"
  - `note TEXT NOT NULL DEFAULT ''`
  - `created_us`, `updated_us`
  - `UNIQUE (template_id, event_session_id)`
- [ ] Index on (`event_session_id`) for instructor's "exception notes for my class today" view.

### 2.4 Wire into DB init
- [ ] `make_database_info.cpp` adds three `Make*Table()` calls after `class_schedules`, `event_sessions`, `people`.
- [ ] `create_database.cpp` `CreateTables()` adds three `CreateTable()` calls.
- [ ] CMakeLists for `db_schema/` and `sql_util/table_helpers/`.

## 3. Table Helpers

### 3.1 `TableHelpers::AttendanceTemplates`
- [ ] CRUD + `GetOrCreateTemplateForPerson(Transaction&, int64_t personId)` — idempotent fetch-or-create. Tests.

### 3.2 `TableHelpers::AttendanceTemplateEntries`
- [ ] `AddEntry(Transaction&, templateId, scheduleId)` — idempotent (no-op if exists).
- [ ] `DeleteEntry(Transaction&, templateId, scheduleId)`.
- [ ] `GetEntriesForTemplate(Transaction&, templateId)` → vector.
- [ ] `GetTemplateIdsForSchedule(Transaction&, scheduleId)` → list of template IDs (for "who has this on their template" lookups; joins back to `attendance_templates → people` in business logic).
- [ ] Tests for CRUD + idempotency + unique-constraint behavior.

### 3.3 `TableHelpers::AttendanceTemplateExceptions`
- [ ] `SetException(Transaction&, templateId, eventSessionId, attending, note)` — UPSERT.
- [ ] `DeleteException(Transaction&, templateId, eventSessionId)`.
- [ ] `GetExceptionsForTemplate(Transaction&, templateId, fromUs, toUs)`.
- [ ] `GetExceptionsForSession(Transaction&, eventSessionId)` → list (for the instructor's "see who's skipping" view).
- [ ] Tests.

## 4. Business Logic — `AttendanceTemplateHelper`

Place in `business_logic/scheduling/attendance_template_helper.h/.cpp/_test.cpp`.

### 4.1 Eligible-classes resolver
- [ ] `struct EligibleScheduleInfo { int64_t classScheduleId; int64_t classId; std::string className; std::string classPhotoUrl; int64_t facilityId; std::string facilityName; std::vector<int> daysOfWeek; int64_t startTimeMinutes; int64_t durationMinutes; bool onTemplate; std::string subtitle; }`.
- [ ] `std::vector<EligibleScheduleInfo> GetEligibleSchedulesForPerson(Transaction&, int64_t personId)`. Algorithm:
  1. Pull active class_schedules (joined to active classes).
  2. Filter to schedules whose product the user has booking permission for (M-6, via `CatalogHelper::ResolveBestPriceForPerson` returning `isIncluded=true`).
  3. Filter further by skill requirements: call `SkillLevelHelper::PersonMeetsClassRequirements(personId, classId)`; drop schedules where `meetsAll=false`.
  4. Decorate with `onTemplate` boolean and (Phase 13) tag list.
  5. Order by day-of-week + start-time-minutes for the weekly grid.
- [ ] Tests cover: member with included class, member missing skill, member without permission.

### 4.2 Template entry add / remove
- [ ] `AddTemplateEntry(Transaction&, personId, scheduleId)`:
  1. Validate eligibility (must be in `GetEligibleSchedulesForPerson`); reject otherwise with `NOT_ELIGIBLE`.
  2. `GetOrCreateTemplateForPerson`, then `AttendanceTemplateEntries::AddEntry` (idempotent).
  3. Build a recurring iCal using `BuildTemplateUid(scheduleId, personId)` + `BuildWeeklyRRule(daysOfWeek, untilUs=min(schedule.effective_to, +1y))`.
  4. Queue a confirmation email via `MailHelper` with the `.ics` attachment.
  5. Return `{ok=true, templateEntryId}`.
- [ ] `RemoveTemplateEntry(Transaction&, personId, scheduleId)`:
  1. Find template, delete the entry row.
  2. No email.
  3. Return `{ok=true}`.
- [ ] Tests: add idempotent, add rejected when ineligible, remove no-op when not present.

### 4.3 Per-instance exception
- [ ] `SetException(Transaction&, personId, eventSessionId, attending, note)`:
  1. Resolve template; if none yet, `GetOrCreateTemplateForPerson`.
  2. UPSERT the exception row.
  3. No email; the note flows to the instructor via the staff portal feed (5.3 / N-7).
- [ ] `RemoveException(Transaction&, personId, eventSessionId)` — delete row.

### 4.4 Materialization hook
- [ ] When `RecurringSessionHelper` (called via `ClassScheduleHelper::MaterializeFutureSessions`) creates new `event_sessions` for a schedule, NO booking is created — per the design lock-in in §1.1. The hook here is read-only: ensure `EventSessionHelper::GetVisibleEventSessions` joins to template entries so the "is on my template" flag works going forward.
- [ ] No new code path — verify the join is in place. Add an integration test that materializes a new session and asserts the templated user's homepage feed shows it as checked.

### 4.5 Homepage feed
- [ ] `struct TodayClassEntry { int64_t eventSessionId; int64_t classId; std::string className; std::string classPhotoUrl; int64_t startUs; int64_t endUs; std::string facilityName; std::string roomName; std::vector<std::string> instructorNames; bool onTemplate; bool exceptionAttending; bool exceptionSkipping; std::string exceptionNote; std::string perInstanceNote; }`.
- [ ] `std::vector<TodayClassEntry> GetTodayClassesForPerson(Transaction&, int64_t personId, int64_t nowUs, std::string_view ianaTz)`. Algorithm:
  1. Compute today's date in `ianaTz`.
  2. Pull `event_sessions` whose start_us falls within "today" (start of facility day → end of facility day).
  3. Filter to sessions whose `class_schedule_id` is eligible for the user (eligibility resolver from 4.1).
  4. Left-join `attendance_template_entries` to set `onTemplate`.
  5. Left-join `attendance_template_exceptions` to set `exceptionAttending` / `exceptionSkipping` / `exceptionNote`.
  6. Left-join `event_sessions.per_instance_note` (when Phase 13 adds it; Phase 5 leaves this blank).
  7. Return sorted by `startUs`.
- [ ] Tests cover: templated session checked, non-templated eligible session unchecked, exception-skip displayed struck-through.

### 4.6 Instructor exception-notes view
- [ ] `struct InstructorExceptionNote { int64_t eventSessionId; int64_t personId; std::string personName; bool attending; std::string note; int64_t createdUs; }`.
- [ ] `std::vector<InstructorExceptionNote> GetExceptionNotesForInstructorPerson(Transaction&, int64_t instructorPersonId, int64_t fromUs, int64_t toUs)`. Joins:
  - `event_session_staffing` rows where `person_id = instructorPersonId AND role IN ('instructor', 'lead instructor')`.
  - To `attendance_template_exceptions WHERE event_session_id IN (those sessions) AND created_us BETWEEN fromUs AND toUs`.
- [ ] Used by:
  - Staff portal page (5.3) — "Notes from your students this week".
  - Daily digest job (N-7) — collects fresh notes from the last 24h and emails the instructor.

### 4.7 KeyValueTable conversions
- [ ] Add converters for `EligibleScheduleInfo`, `TodayClassEntry`, `InstructorExceptionNote` in `scheduling_key_value_table.h/.cpp`.

## 5. Endpoints

### 5.1 User endpoints
- [ ] `GET /api/me/eligible_schedules` → list of `EligibleScheduleInfo`.
- [ ] `GET /api/me/template` → `{ entries: [...], exceptions: [...] }`.
- [ ] `POST /api/me/template/entry` body `{ schedule_id }`.
- [ ] `DELETE /api/me/template/entry/<scheduleId>`.
- [ ] `POST /api/me/template/exception` body `{ event_session_id, attending, note? }`.
- [ ] `DELETE /api/me/template/exception/<eventSessionId>`.
- [ ] `GET /api/me/today_classes` → list of `TodayClassEntry` (facility ID query param if user belongs to multiple facilities — default to "all").

### 5.2 Instructor staff portal endpoint
- [ ] `GET /api/staff/me/exception_notes?from=<us>&to=<us>` → list of `InstructorExceptionNote`. Permission: `staff` role.

### 5.3 Daily digest scheduled job
- [ ] `POST /api/admin/send_instructor_exception_digests` — idempotent, called daily by `knottyyoga_helper`. Iterates instructors with fresh notes in the past 24h and queues a digest email each. (See [[Scheduled Jobs]] for the helper pattern.)

### 5.4 Routing + permissions
- [ ] All new endpoints registered in `web_app.cpp`.
- [ ] All user endpoints require logged-in session; gate on session.IsLoggedIn().
- [ ] Async email queueing → make sure tests `ThreadPool::Shutdown()` before the next DB read.

## 6. Frontend

### 6.1 My Schedule page — eligible-classes grid
- [ ] `ui/src/app/pages/account/my-schedule/my-schedule.component.*/.spec.ts`.
- [ ] Weekly grid (7 columns × time-of-day rows) of eligible classes with checkboxes.
- [ ] Checking → calls `addTemplateEntry`; unchecking → `removeTemplateEntry`.
- [ ] Empty state for "no eligible classes yet — talk to staff about a skill evaluation or upgrade your membership".

### 6.2 Calendar overlay
- [ ] In the existing `pages/calendar/` views, render template-attending classes with a filled fill, one-off addition with a star icon, exception-skip with strike-through + tooltip showing the note.
- [ ] Spec coverage for all three states.

### 6.3 Homepage today-classes feed
- [ ] `ui/src/app/pages/home/today-classes/today-classes.component.*/.spec.ts`.
- [ ] List of today's classes — checked rows for template/one-off, unchecked rows for eligible-not-claimed.
- [ ] Click to flip state via `setException(eventSessionId, true)` or `setException(false)`.
- [ ] Inline "I can't make it" button → opens dialog with optional note field.
- [ ] Optimistic UI updates with rollback on error.

### 6.4 Exception-note dialog
- [ ] `exception-note-dialog.component.*/.spec.ts` — input field + save + cancel.

### 6.5 Instructor staff portal — exception notes
- [ ] `ui/src/app/pages/portal/staff/exception-notes/exception-notes.component.*/.spec.ts`.
- [ ] Lists fresh notes from members for sessions this instructor is teaching this week.

### 6.6 `ServerAccess` extensions
- [ ] `getEligibleSchedules()`, `getTemplate()`, `addTemplateEntry(scheduleId)`, `removeTemplateEntry(scheduleId)`, `setException(eventSessionId, attending, note?)`, `removeException(eventSessionId)`, `getTodayClasses()`, `getInstructorExceptionNotes(from, to)`.
- [ ] Update `ServerAccess.mock.spec.ts`.

### 6.7 Types
- [ ] `ui/src/app/shared/types/template.types.ts`: `EligibleSchedule`, `TemplateEntry`, `TemplateException`, `TodayClassEntry`, `InstructorExceptionNote`.

## 7. Admin Metadata

- [ ] Three new tables nested as follows:
  - `attendance_templates` → top-level (admin can list all templates), permission `manage_users` or `admin`.
  - `attendance_template_entries` → nested under `attendance_templates` keyed by `template_id`.
  - `attendance_template_exceptions` → nested under `attendance_templates` keyed by `template_id`.
- [ ] Column data info, friendly names, table friendly names, display templates.
- [ ] Generally admins won't edit these directly; nesting is mostly for support / debugging.

## 8. Scheduled job integration

- [ ] Add daily 09:00-local job to `knottyyoga_helper`: calls `POST /api/admin/send_instructor_exception_digests`. Idempotent.
- [ ] Job config secret `instructor_digest_send_hour_local` default 9.

## 9. Tests-Required Summary

- [ ] Table helper tests (CRUD + uniqueness + idempotency).
- [ ] `attendance_template_helper_test.cpp`:
  - eligible-classes filter correctness (member, non-member, missing skill)
  - add template entry sends confirmation email with RRULE iCal containing the right UID and days-of-week
  - re-add is idempotent (no duplicate email)
  - remove no-op when not present
  - exception upsert flips between attending/skipping
  - today_classes returns the right onTemplate / exception state
  - instructor exception-notes view filters by staffing rows
- [ ] Endpoint tests for all eight new endpoints.
- [ ] Frontend specs: my-schedule, calendar overlay, today-classes, exception dialog, instructor exception notes, mock service.
- [ ] Manual-testing-helper commands: `add_template_entry <person_id> <schedule_id>`, `simulate_exception_note <person_id> <event_session_id> <note>`, `send_instructor_digest <instructor_person_id>`.

## 10. Cross-Layer Acceptance Criteria

A logged-in member with:
- Active Gold Membership granting `gold_member` permission
- "Aerial Basics" skill level (so they pass the skill gate)
- A class schedule "Aerial 101" recurring Mon/Wed 7-8pm

Should be able to:
- [ ] Open `/my/account/my-schedule`, see "Aerial 101" with a checkbox in the Mon & Wed slots.
- [ ] Check both → API calls land, two `attendance_template_entries` created (one per schedule, deduplicated to one if both days are on the same schedule).
- [ ] Receive ONE confirmation email with a single iCal `VEVENT` + `RRULE=FREQ=WEEKLY;BYDAY=MO,WE;UNTIL=...`.
- [ ] On Monday morning, see "Aerial 101 at 7pm — Studio A" with a checkmark on `/my/home`.
- [ ] Click "I can't make it this Monday" → exception dialog → enter "out of town" → save. Homepage row turns to struck-through; an `attendance_template_exception(attending=false, note='out of town')` is created.
- [ ] The assigned instructor sees the note the next morning in their staff-portal exception-notes page AND in the daily digest email.

## 11. Open Questions

- **OQ-P5-1.** When a class schedule is deactivated (Phase 1), should existing template entries be auto-deleted? Recommended: yes, atomic with deactivation; otherwise stale entries pile up. The user is not emailed about the auto-delete — they'll just see the class disappear from their grid.
- **OQ-P5-2.** Across multiple facilities, should `eligible_schedules` and `today_classes` filter by the user's "home" facility, or surface all facilities? Recommended: all facilities by default; UI offers a filter chip. The studio is single-facility today but the model supports multi.
- **OQ-P5-3.** When a user's membership lapses, do their template entries get auto-deleted? Recommended: no — leave them; if they renew within N days the template is still there. UI shows entries with "no longer eligible" badge for any schedule whose permission they don't currently hold.

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 5.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 3 - Skill Levels]], [[Classes Phase 4 - iCal Generator Extensions]].
- Consumers: [[Classes Phase 6 - Weekly Digest]] (uses template entries + exceptions to build the digest), [[Classes Phase 8 - Staff Check-in]] (uses template membership for the pre-pop list).
- Scheduler: [[Scheduled Jobs]].
