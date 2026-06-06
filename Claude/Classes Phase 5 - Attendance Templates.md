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

> **Access redesign note (2026-05-31, [[Permission-based class access redesign]] §4.5):** The attendance facts this phase deals with feed **SL-10** — the attendance-threshold permission grant ("attended ≥N classes last/this month → a time-boxed permission some classes require"). No access-model change here, but the SL-10 monthly grant job (Phase 3) **depends on this phase's attendance data**, and the permissions it grants flow into the closure-expanded `GetEffectivePermissionIds` → the shared `Scheduling::ClassAccessHelper` gate. Note the dependency.

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

### 1.2 Forward sync (lazy model)
- [x] Adding a template entry records the slot-level template entry only — it creates NO bookings and NO `event_sessions` rows. The homepage and digest compute "is this on my template?" by evaluating the user's template-entry slots against the **derived** occurrences for the week (`ClassScheduleHelper::GetDerivedSessionsForRange`), NOT by joining persisted rows.
- [x] Removing a template entry is just a row delete. No cascading booking cancellation (there were no bookings).

### 1.3 Per-instance exception semantics
- [x] Exception = "I won't attend this Tuesday" (attending=false) OR "I will attend this Thursday in addition to my template" (attending=true). One row covers both.
- [x] Exception rows have an optional note routed to the instructor.
- [x] Same-day predecessor (SL-11) auto-cancel: if a user cancels a paid booking on the predecessor class, the dependent booking is silently cancelled (SL-11 lives in `BookingHelper`, not in this phase — but the homepage display in this phase must respect it for consistency).

### 1.4 Email cadence
- [x] On template add: one confirmation email with recurring RRULE iCal (Phase 4 `BuildWeeklyRRule` / `BuildTemplateUid`). **Phase 4 note:** the `.ics` recurring entry uses the UTC `DTSTART`+`RRULE` form (Phase 4's VTIMEZONE/DST work was deferred). If we want DST-correct local recurrence, decide between (a) finishing Phase 4 §3.3 VTIMEZONE, or (b) emitting **bounded per-occurrence VEVENTs** (concrete UTC instants, no VTIMEZONE) via the Phase 4 multi-event overload. See §1.6.
- [x] On template remove: NO email (silent).
- [x] On exception (skip): NO email; the note goes to the instructor's staff portal feed + daily digest per N-7.
- [x] On exception (one-off add): NO email; shows up in the next Sunday digest (Phase 6).

### 1.5 Template lifecycle (OQ-P5-1 / -3 resolved 2026-06-04)
- [x] **Class deactivation → cascade delete (OQ-P5-1).** When a class (or its active instance) is deactivated in Phase 1, delete that class's template **entries + exceptions** atomically with the deactivation. No email. Hooked into the Phase-1 deactivation business logic; do NOT scatter the delete into the deactivation endpoint (layering).
- [x] **Slot deleted on impl edit / product migration → surface as stale** (redesign note) — narrower than deactivation; the entry's slot FK row is gone, so the user portal shows "this slot no longer exists — pick a new one". (Distinct from OQ-P5-1: deactivation is a soft `is_active=false` on the class and would NOT cascade via FK, so it needs the explicit hook above; a hard slot delete is the stale-surface case.)
- [x] **Membership lapse → keep entries, badge them (OQ-P5-3).** No deletion; `GetTemplateForPerson` decorates each entry with `currentlyEligible` (re-evaluated via the live `ClassAccessHelper` gate). A renewal restores eligibility with zero user action.

### 1.6 Recurring-`.ics` shape — OPEN sub-decision (deferred from Phase 4)
- [ ] Decide whether the template-add `.ics` uses (a) a single open-ended `RRULE` VEVENT (needs Phase 4 §3.3 VTIMEZONE for DST correctness — currently UTC-anchored, so it drifts an hour across DST), or (b) **bounded per-occurrence VEVENTs** for the next N weeks via the Phase 4 multi-event overload (DST-correct, no VTIMEZONE, but re-sent/extended periodically). Recommendation: **(b)** — simpler, DST-correct, and lets us drop Phase 4 §3.3 entirely. Confirm before §4.2 is built.

## 2. Database Schema

> **Implementation deviation (slot FK).** §2.2/§2.3 below say `class_schedule_slot_id ... REFERENCES class_schedule_slots(id)`, but the implemented schema makes `class_schedule_slot_id` a **plain `BIGINT` with NO foreign key** (via `AddColumnSimple`). Rationale matches `predecessor_class_schedule_slot_id`: when an impl is edited or a product migrated its slots are hard-deleted; a FK would block that delete (or force a cascade that silently drops the entry). With no FK the slot delete proceeds and the entry/exception survives as **stale**, which the user portal surfaces as "this slot no longer exists — pick a new one" (§1.5). `template_id` keeps its FK to `attendance_templates`.

### 2.1 `attendance_templates` table
- [x] `db_schema/attendance_templates.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL UNIQUE REFERENCES people(id)`  — one template per person (FK + `AddUniqueConstraint`)
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us`, `updated_us`

### 2.2 `attendance_template_entries` table
- [x] `db_schema/attendance_template_entries.h/.cpp`:
  - `id BIGSERIAL PK`
  - `template_id BIGINT NOT NULL REFERENCES attendance_templates(id)`
  - `class_schedule_slot_id BIGINT NOT NULL` — plain BIGINT, **no FK** (see deviation note); binds to the stable-identity slot, NOT the impl
  - `created_us`
  - `UNIQUE (template_id, class_schedule_slot_id)`
- [x] Index on (`template_id`) for "all my entries" reads (`CreateAttendanceTemplateEntriesIndexes`, prod-only path).
- [x] Index on (`class_schedule_slot_id`) for "who has this slot on their template?" reads (instructor view).

### 2.3 `attendance_template_exceptions` table
- [x] `db_schema/attendance_template_exceptions.h/.cpp`:
  - `id BIGSERIAL PK`
  - `template_id BIGINT NOT NULL REFERENCES attendance_templates(id)`
  - `class_schedule_slot_id BIGINT NOT NULL` — plain BIGINT, **no FK** (see deviation note); the slot the exception applies to
  - `occurrence_date_us BIGINT NOT NULL` — the specific day (the occurrence usually has no persisted `event_sessions` row, so we key off the derived-occurrence identity, NOT `event_session_id`)
  - `attending BOOLEAN NOT NULL` — false = "skip this instance"; true = "one-off add"
  - `note TEXT NOT NULL DEFAULT ''`
  - `created_us`, `updated_us`
  - `UNIQUE (template_id, class_schedule_slot_id, occurrence_date_us)`
- [x] Index on (`class_schedule_slot_id`, `occurrence_date_us`) for instructor's "exception notes for my class today" view (`CreateAttendanceTemplateExceptionsIndexes`, prod-only path).

### 2.4 Wire into DB init
- [x] `make_database_info.cpp` adds three `Make*Table()` calls (after `MakeBookingRequirementOverridesTable`; templates → entries → exceptions, parent-before-children) + the three includes.
- [x] `create_database.cpp` `CreateTables()` adds three `CreateTable()` calls + the two `CreateXxxIndexes()` calls + the three includes.
- [x] `db_schema/CMakeLists.txt` lists all six `.h/.cpp` files. (Table-helper CMake entries land with §3.)

## 3. Table Helpers

> **Param reconciliation (done at implementation).** The §3 signatures originally said `scheduleId` / `eventSessionId`, but per the redesign note the binding key is `class_schedule_slot_id` (+ `occurrence_date_us` for exceptions) — which is what the §2 schema actually stores. The implemented helpers therefore take `classScheduleSlotId` / `occurrenceDateUs`, not `scheduleId` / `eventSessionId`. Bool values are stored as `true`/`false` literals (DbCrud allowed-keyword set) and read back as Postgres `t`/`f`.

### 3.1 `TableHelpers::AttendanceTemplates`
- [x] `GetOrCreateTemplateForPerson(Transaction&, int64_t personId)` — idempotent fetch-or-create. Plus `GetTemplate(id)`, `GetTemplateForPerson(personId)`, `SetActive(id, bool)` (CRUD). Tests in `attendance_templates_test.cpp` (idempotency, per-person isolation, get-by-id/person, empty-when-none, active toggle).

### 3.2 `TableHelpers::AttendanceTemplateEntries`
- [x] `AddEntry(Transaction&, templateId, classScheduleSlotId)` — idempotent (returns existing id, no duplicate; UNIQUE-guarded).
- [x] `DeleteEntry(Transaction&, templateId, classScheduleSlotId)` — no-op when absent.
- [x] `GetEntriesForTemplate(Transaction&, templateId)` → `KeyValueTableArray`, oldest first.
- [x] `GetTemplateIdsForSlot(Transaction&, classScheduleSlotId)` → `std::vector<int64_t>` (reconciled from `GetTemplateIdsForSchedule`; "who has this slot on their template?" — joins back to `attendance_templates → people` in business logic).
- [x] Tests in `attendance_template_entries_test.cpp` (add/get, idempotency, delete + no-op, reverse-lookup by slot, per-template isolation).

### 3.3 `TableHelpers::AttendanceTemplateExceptions`
- [x] `SetException(Transaction&, templateId, classScheduleSlotId, occurrenceDateUs, attending, note)` — UPSERT (INSERT … ON CONFLICT … DO UPDATE, refreshes `updated_us`), returns row id.
- [x] `DeleteException(Transaction&, templateId, classScheduleSlotId, occurrenceDateUs)` — no-op when absent.
- [x] `GetExceptionsForTemplate(Transaction&, templateId, fromUs, toUs)` — half-open `[fromUs, toUs)` range on `occurrence_date_us`, ordered by occurrence then id.
- [x] `GetExceptionsForSlotOccurrence(Transaction&, classScheduleSlotId, occurrenceDateUs)` → list across all templates (reconciled from `GetExceptionsForSession`; the instructor's "who's skipping / dropping in for this class" view).
- [x] Tests in `attendance_template_exceptions_test.cpp` (insert, upsert-in-place, delete + no-op, half-open range, cross-template occurrence span, empty-when-none).

## 4. Business Logic — `AttendanceTemplateHelper`

Place in `business_logic/scheduling/attendance_template_helper.h/.cpp/_test.cpp`. **DONE.**

> **Reconciliations made at implementation (carry into §5/§6):**
> - All params are `class_schedule_slot_id` (+ `occurrence_date_us`), never `schedule_id`/`event_session_id`.
> - `EligibleScheduleInfo` is **slot-level** (one row per day/time slot, single `dayOfWeek`), not schedule-level with a `daysOfWeek[]` — the grid renders a checkbox per slot.
> - Class photo is a `classHasPhoto` **bool** (matching `ScheduleSlotView`), not a `classPhotoUrl` — the frontend builds the URL.
> - §4.6 keys instructor notes off the **slot's `instructor_person_id`** (the recurring teacher), NOT `event_session_staffing` rows — most templated occurrences are lazy (no persisted `event_sessions` row), so a staffing join would miss them. The window filters on `created_us` (when the student wrote the note) per the plan's algorithm.
> - **Supporting table-helper additions (with tests):** `ClassScheduleSlots::GetSlotsByInstructor`; `AttendanceTemplateEntries::DeleteEntriesForSlot`; `AttendanceTemplateExceptions::DeleteExceptionsForSlot` + `GetExceptionsForSlotByCreated`.

### 4.1 Eligible-classes resolver
- [x] `struct EligibleScheduleInfo` — **slot-level** (`classScheduleSlotId`, `classScheduleId`, `classId`, `className`, `classHasPhoto`, `facilityId`, `facilityName`, `dayOfWeek`, `startTimeMinutes`, `durationMinutes`, `onTemplate`, `subtitle`).
- [x] `std::vector<EligibleScheduleInfo> GetEligibleSchedulesForPerson(Transaction&, int64_t personId, std::optional<int64_t> facilityId = {})`. Algorithm:
  1. Pull active class_schedule **slots** (joined up to active class_schedules → instances → classes), per the redesign note (bind on the stable slot, not the impl).
  2. **Single unified access check (Phase 3 §1.4):** `Scheduling::ClassAccessHelper::CheckAccess(classId, personId)` — this already AND-combines membership permission, skill literals, and attendance-threshold permissions (closure-expanded). Keep only slots whose class the viewer can access (`allowed=true`). *(Replaces the old separate `CatalogHelper::ResolveBestPriceForPerson` + the superseded `SkillLevelHelper::PersonMeetsClassRequirements` steps — both folded into the one gate by Phase 3.)*
  3. Restrict to **recurring** (membership-included) classes per §1.1 — templates don't apply to paid workshops/series.
  4. **(OQ-P5-2)** Include **all facilities** by default; when `facilityId` is provided, narrow to that facility.
  5. Decorate with `onTemplate` (slot id ∈ the user's `attendance_template_entries`) and (Phase 13) tag list.
  6. Order by day-of-week + start-time-minutes for the weekly grid.
- [x] Tests cover: member with included (open) class eligible, member without the gating permission blocked / granted member allowed, and the `facilityId` filter narrowing.

### 4.2 Template entry add / remove
- [x] `AddTemplateEntry(Transaction&, personId, classScheduleSlotId)` → `AddTemplateEntryResult {ok, errorCode, templateEntryId}`:
  1. `INVALID_SLOT` if the slot row doesn't exist; eligibility = "slot appears in `GetEligibleSchedulesForPerson`" else `NOT_ELIGIBLE`.
  2. `GetOrCreateTemplateForPerson`, then `AttendanceTemplateEntries::AddEntry` (idempotent).
  3. Recurring iCal via `BuildTemplateUid(classScheduleId, personId)` + `BuildWeeklyRRule({dayOfWeek}, untilUs=now+1y)`; DTSTART = first derived occurrence within 14 days.
  4. Queue the confirmation email via `MailHelper` with the `.ics` (only when mail is configured AND the entry is newly added — a re-add sends nothing).
- [x] `RemoveTemplateEntry(Transaction&, personId, classScheduleSlotId)` — finds the template, deletes the entry row, no email; no-op (returns true) when the person has no template.
- [x] **`GetTemplateForPerson`** → `PersonTemplateView { templateId, entries[], exceptions[] }`; each entry carries `currentlyEligible` (OQ-P5-3, re-evaluated against `ClassAccessHelper::CheckAccess`) and `slotExists` (redesign §1.5 stale-slot).
- [x] **`DeleteTemplateEntriesForClass(Transaction&, classId)`** (OQ-P5-1): sweeps all instances → impls → slots of the class and deletes every template's entries **and** exceptions on those slots. To be called from the Phase-1 deactivation business logic (no email).
- [x] Tests: add idempotent + no duplicate email, add rejected when ineligible (`NOT_ELIGIBLE`) / invalid slot (`INVALID_SLOT`), remove no-op when no template, stale-slot surfaced, `currentlyEligible=false` after the permission is revoked (entry kept), `DeleteTemplateEntriesForClass` removes entries + exceptions.

### 4.3 Per-instance exception
- [x] `SetException(Transaction&, personId, classScheduleSlotId, occurrenceDateUs, attending, note)` → exception row id. Auto-creates the template (`GetOrCreateTemplateForPerson`), then UPSERTs keyed by (`template_id`, `class_schedule_slot_id`, `occurrence_date_us`). No email.
- [x] `RemoveException(Transaction&, personId, classScheduleSlotId, occurrenceDateUs)` — deletes the row; no-op when the person has no template.

### 4.4 Derived-occurrence evaluation (NO materialization hook)
- [x] No materialization hook. `onTemplate` is computed by matching the user's template-entry slot ids against the **derived** occurrences (`ClassScheduleHelper::GetDerivedSessionsForRange`), then overlaying `attendance_template_exceptions` by (`class_schedule_slot_id`, `occurrence_date_us`). Implemented inside §4.5.
- [x] Covered by `TodayClassesReflectsTemplateAndException`: derives the day's occurrence, asserts checked when templated, and a skip-exception flips `exceptionSkipping`.

### 4.5 Homepage feed
- [x] `struct TodayClassEntry` — adds `classScheduleSlotId` + `occurrenceDateUs` (+ keeps `eventSessionId` = persisted id or 0), `classHasPhoto` bool, `classId`, `className`, `startUs`/`endUs`, `facilityId`/`facilityName`, `locationRoomId`/`roomName`, `instructorNames[]`, `onTemplate`, `exceptionAttending`/`exceptionSkipping`/`exceptionNote`, `perInstanceNote` (blank, Phase 13).
- [x] `std::vector<TodayClassEntry> GetTodayClassesForPerson(Transaction&, int64_t personId, int64_t nowUs, std::string_view ianaTz, std::optional<int64_t> facilityId = {})`. **(OQ-P5-2)** all facilities by default; `facilityId` narrows. Algorithm:
  1. Compute today's date in `ianaTz`.
  2. **Derive** today's occurrences via `ClassScheduleHelper::GetDerivedSessionsForRange` for the facility day window (each carries its `class_schedule_slot_id` + `occurrence_date_us`; a persisted `event_sessions` row may or may not exist).
  3. Filter to occurrences whose slot is eligible for the user (eligibility resolver from 4.1).
  4. Set `onTemplate` by matching the occurrence's `class_schedule_slot_id` against the user's `attendance_template_entries`.
  5. Overlay `attendance_template_exceptions` by (`class_schedule_slot_id`, `occurrence_date_us`) to set `exceptionAttending` / `exceptionSkipping` / `exceptionNote`.
  6. Left-join any persisted `event_sessions` per-instance note (when Phase 13 / CS-7 adds it; Phase 5 leaves blank).
  7. Return sorted by `startUs`. (Exception overlay reflects only the viewer's own exception; cancelled occurrences are dropped.)
- [x] Tests cover: templated occurrence checked, non-templated eligible occurrence unchecked, exception-skip flips `exceptionSkipping`, and the `facilityId` filter narrows.

### 4.6 Instructor exception-notes view
- [x] `struct InstructorExceptionNote { int64_t classScheduleSlotId; int64_t occurrenceDateUs; int64_t personId; std::string personName; bool attending; std::string note; int64_t createdUs; }` (slot+occurrence keyed, reconciled from `eventSessionId`).
- [x] `std::vector<InstructorExceptionNote> GetExceptionNotesForInstructorPerson(Transaction&, int64_t instructorPersonId, int64_t fromUs, int64_t toUs)`. Reconciled join (see §4 note): the instructor's **slots** (`ClassScheduleSlots::GetSlotsByInstructor`) → `attendance_template_exceptions` on those slots created in `[fromUs, toUs)` → resolve the noting person via `template_id → attendance_templates.person_id`. Newest-first.
- [x] Used by: staff portal page (5.2) + daily digest job (5.3 / N-7).
- [x] Test: filters to the instructor's own slots only, by `created_us` window.

### 4.7 KeyValueTable conversions
- [x] Added converters (+ array variants) for `EligibleScheduleInfo`, `TemplateEntryView`, `TemplateExceptionView`, `TodayClassEntry`, `InstructorExceptionNote` in `scheduling_key_value_table.h/.cpp`. (`PersonTemplateView` is composed by the endpoint: `template_id` + nested `entries[]`/`exceptions[]` arrays, matching the `ClassDetail`/`RequirementGroup` nesting pattern.)

## 5. Endpoints

### 5.1 User endpoints
> Per the redesign note the binding key is `class_schedule_slot_id` + `occurrence_date_us` (not `schedule_id` / `event_session_id`) — implemented with slot+occurrence keys throughout. Each endpoint is one `.cpp` (+`.h`+`_test.cpp`); response arrays are wrapped as `{ "items": [...] }` via `SqlUtil::KeyValueTableArrayToJson`. **DONE.**
- [x] `GET /api/me/eligible_schedules?facility_id=<id>` → `{items:[EligibleSchedule]}`. `facility_id` optional (OQ-P5-2).
- [x] `GET /api/me/template` → `{ template_id, entries:[…], exceptions:[…] }`; entries carry `currently_eligible` + `slot_exists`.
- [x] `POST /api/me/template/entry` body `{ class_schedule_slot_id }` → `{ok, template_entry_id}`. Constructs the helper **with mail** so the confirmation `.ics` goes out. `NOT_ELIGIBLE`→403, `INVALID_SLOT`→404, missing field→400.
- [x] `DELETE /api/me/template/entry/<int>` → `{ok}` (no-op when no template).
- [x] `POST /api/me/template/exception` body `{ class_schedule_slot_id, occurrence_date_us, attending, note? }` → `{exception_id}`.
- [x] `DELETE /api/me/template/exception/<int>/<int>` → `{ok}`.
- [x] `GET /api/me/today_classes?facility_id=<id>&tz=<iana>` → `{items:[TodayClassEntry]}`. **Reconciliation:** the endpoint resolves "now" from `now_us()` and takes the studio IANA timezone via an optional `tz` query param (default `UTC`); `facility_id` optional (OQ-P5-2).

### 5.2 Instructor staff portal endpoint
- [x] `GET /api/staff/me/exception_notes?from=<us>&to=<us>` → `{items:[InstructorExceptionNote]}`. Gated on `kPermissionStaffAccess` via `RequirePermission` (401 anon / 403 missing). `from`/`to` required (400 otherwise). Resolves the instructor as the logged-in `session.GetPersonId()`.

### 5.3 Daily digest scheduled job
- [x] `POST /api/admin/send_instructor_exception_digests` → `{sent:N}`. Gated on `kPermissionManageClassSchedule`. Computes a 24h window from `now_us()` and calls `AttendanceTemplateHelper::SendInstructorExceptionDigests` (business logic owns the fan-out + mail). Not deduplicated (no sent-ledger — the daily cron passes one 24h window). **Supporting additions:** `AttendanceTemplateExceptions::GetExceptionsCreatedInWindow` (table helper, +test) and `instructor_exception_digest_mail.{h,cpp}` (FormatString + `NormalizeCrLf`, +test).

### 5.4 Routing + permissions
- [x] All nine endpoints registered in `web_app.cpp` (include + reference variable) and listed in `endpoints/CMakeLists.txt`.
- [x] All `/api/me/*` endpoints gate on `session.IsLoggedIn()` → 401; staff/admin endpoints use `RequirePermission`.
- [x] **No `ThreadPool` needed:** the confirmation + digest emails are sent **synchronously** inside the business logic (matching `PaymentHelper`/`SessionCancellationHelper`), so tests don't need `ThreadPool::Shutdown()`. Endpoint tests inject `TestMailHelper` and assert on `GetMailHelper()->GetMessages()`.
- [x] Shared inline test fixture (`endpoints/template_test_fixture.h`) provides `LoginUser` + `CreateRecurringSlot` to keep the nine `_test.cpp` files self-contained without duplicating setup.

## 6. Frontend  — **DONE except §6.2 (calendar overlay, deferred).**

> Routing note: the account area is mounted at `/my/*` (not `/my/account/*`), so the pages live at `/my/my-schedule` and `/my/today`. The plan's `pages/home/today-classes` / `pages/portal/staff` paths were reconciled to the actual `pages/account/` and `pages/staff/` locations.

### 6.1 My Schedule page — eligible-classes grid
- [x] `ui/src/app/pages/account/my-schedule/my-schedule.component.*` + `.spec.ts`.
- [x] Day-column grid of eligible slots with checkboxes (clearer than a fixed time-row grid).
- [x] Checking → `addTemplateEntry`; unchecking → `removeTemplateEntry` (optimistic + rollback).
- [x] **(OQ-P5-2)** Facility filter chips; default "All facilities".
- [x] **(OQ-P5-3)** "Kept on your schedule" section badges `currently_eligible=false` entries "No longer eligible" and `slot_exists=false` entries "this time slot no longer exists — pick a new one".
- [x] Empty state ("no eligible classes yet — talk to staff…").

### 6.2 Calendar overlay — **DEFERRED**
- [ ] Render template-attending / one-off / skip states in the existing `pages/calendar/` views. Deferred: it requires integrating into the existing calendar component (not yet explored) and is the least self-contained piece. Tracked as a follow-up; everything else in §6 is independent of it.

### 6.3 Homepage today-classes feed
- [x] `ui/src/app/pages/account/today-classes/today-classes.component.*` + `.spec.ts` (placed in the account area since there's no logged-in home dashboard; reachable from the account dashboard + `/my/today`).
- [x] List of today's occurrences — checkmark + highlight when attending, strike-through when skipping.
- [x] "I'll be there" → `setException(slot, occ, true)`; "I can't make it" → dialog → `setException(slot, occ, false, note)`. Slot+occurrence keyed (reconciled from `eventSessionId`).
- [x] Optimistic UI with rollback on error.

### 6.3b Multi-day planning feed (Mason's request)
- [x] A member can mark skips / drop-ins **ahead of time** (e.g. before a vacation, or when reviewing their weekly email) instead of only same-day. `ui/src/app/pages/account/upcoming-classes/upcoming-classes.component.*` + `.spec.ts`; route `/my/upcoming` + account-dashboard "Plan My Classes" card. A scrollable list of the next 4 weeks **grouped by day** (sticky day headers), each occurrence with the same "I'll be there" / "I can't make it" + note-dialog controls and optimistic updates.
- [x] Backend: `GET /api/me/upcoming_classes?from=&to=&facility_id=` → `AttendanceTemplateHelper::GetUpcomingClassesForPerson`. Refactored the today + upcoming feeds onto a shared `ResolveClassesInRange` core (derive eligible occurrences in a range, overlay onTemplate + the viewer's exceptions). Endpoint + business + (reused) `TodayClassEntry` shape; tests at both layers.

### 6.4 Exception-note dialog
- [x] `today-classes/exception-note-dialog.component.*` + `.spec.ts` — note textarea + save/cancel; returns `{ note }`.

### 6.5 Instructor staff portal — exception notes
- [x] `ui/src/app/pages/staff/exception-notes/exception-notes.component.*` + `.spec.ts`; route `/staff/exception-notes` + staff-dashboard "Student Notes" tile. Pulls the last 7 days via `getInstructorExceptionNotes`.

### 6.6 `ServerAccess` extensions
- [x] All eight methods added to the interface + `ServerAccessNetwork` (with boolean/`instructor_names` normalization) + `ServerAccessProxy` + `ServerAccessMock`.
- [x] `ServerAccess.mock.spec.ts` — added an "attendance templates" describe block (eligible, idempotent add, ineligible reject, remove, exception upsert/remove, today-feed state, instructor-notes window, 401-when-logged-out).

### 6.7 Types
- [x] `ui/src/app/shared/types/template.types.ts`: `EligibleSchedule`, `TemplateEntry`, `TemplateException`, `PersonTemplate`, `TodayClassEntry`, `InstructorExceptionNote`, `AddTemplateEntryResult`. Re-exported from `shared/types/ServerAccess.ts`.

### 6.8 Bespoke admin support page (Mason's request)
- [x] `ui/src/app/pages/manage/attendance-templates/attendance-templates-admin.component.*` + `.spec.ts`; route `/manage/attendance-templates` + manage-dashboard "Attendance Templates" tile. Read-only browser: per template it shows the **member's name + email**, and each entry as **class name · day · time** (not raw slot ids); exceptions show class + occurrence date + skip/drop-in + note.
- [x] Backed by a dedicated resolved endpoint **`GET /api/admin/attendance_templates?q=<search>`** (gated `manage_class_schedule`) → `AttendanceTemplateHelper::GetAllTemplatesForAdmin(searchQuery)` (resolves person via `people` + per-slot class/day/time via `ResolveSlotContext`). Supporting: `AttendanceTemplates::GetAllTemplates` + `SearchTemplatesByPerson` table helpers, `AdminTemplateView`/`AdminTemplateEntryView`/`AdminTemplateExceptionView` structs + converters, `getAdminAttendanceTemplates(query?)` across all 4 ServerAccess files. (Replaced the earlier raw `getTableRows` approach which only exposed ids.)
- [x] **Search-first UI (scales past a flat list):** a `mat-autocomplete` search box that queries members by **first name, last name, or email** (server-side `ILIKE`, capped at 25 suggestions); selecting a member shows their resolved template. No "list everyone" dump.

## 7. Admin Metadata — **DONE**

- [x] All three tables registered in `create_database.cpp`: `PopulateAllowedTables` (generic CRUD allow-list), `PopulateAdminTopLevelTables` (all three), `PopulateAdminNestedTables` (entries + exceptions nest under templates), `PopulateAdminTablePermissions` (gated on `manage_class_schedule`, id 9).
- [x] `PopulateAdminColumnDataInfo`, `PopulateAdminColumnFriendlyNames`, `PopulateAdminTableFriendlyNames`, `PopulateAdminTableDisplayTemplates` for all three tables (number/bool/date/text edit types; FK display templates).
- [x] Both the generic CRUD "Manage Data" support AND the bespoke §6.8 page are available. (These Populate* seeders run only in the real DB-creation path, so a DB reset via `knottyyoga_database_helper` is needed to pick them up.)

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
  - **(OQ-P5-1)** `DeleteTemplateEntriesForClass` removes entries + exceptions on class deactivation
  - **(OQ-P5-2)** `facilityId` filter narrows eligible_schedules / today_classes; omitted → all facilities
  - **(OQ-P5-3)** an entry stays after the permission is revoked, with `currentlyEligible=false` (not deleted)
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

## 11. Open Questions — ALL RESOLVED (2026-06-04)

- [x] **OQ-P5-1 (resolved — yes, cascade-delete).** When a class is deactivated (Phase 1 sets `classes.is_active=false`; the redesign's narrower slot-delete on impl-edit is handled separately by stale-slot surfacing), its template **entries AND exceptions** are deleted atomically with the deactivation, no email — the class simply disappears from the user's grid. Wired as a hook in the Phase-1 class/instance deactivation path (§1.5 + §4.2). Mason: "I'll go with your recommendation."
- [x] **OQ-P5-2 (resolved — all facilities, UI filter chip).** `eligible_schedules` and `today_classes` surface ALL facilities by default; the endpoints accept an optional `facility_id` filter and the UI offers a filter chip. Single-facility today, but the model is multi-facility-ready. Mason: "I'll go with your recommendation."
- [x] **OQ-P5-3 (resolved — keep entries, badge them).** A membership lapse does NOT delete template entries — they survive so a renewal restores the template. `GET /api/me/template` decorates each entry with `currentlyEligible` (re-evaluated against the live access gate); the UI shows a "no longer eligible" badge for entries whose permission/skill the user no longer holds. Mason: "I'll go with your recommendation."

## 12. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 5.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 3 - Skill Levels]], [[Classes Phase 4 - iCal Generator Extensions]].
- Consumers: [[Classes Phase 6 - Weekly Digest]] (uses template entries + exceptions to build the digest), [[Classes Phase 8 - Staff Check-in]] (uses template membership for the pre-pop list).
- Scheduler: [[Scheduled Jobs]].
