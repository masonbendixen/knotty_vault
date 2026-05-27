---
fileClass: Project
Category: Claude
Status: Design discussion
Authors: Mason Bendixen
Last Updated: 5/27/2026
Version: 0.1
tags:
---
# Overview

Phase 1 (`Classes Phase 1 - Catalog and Schedule Authoring.md` §357) flagged that the current `class_schedules` model is too thin and asks for a redesign modeled on `price_schedules`:

- Versioned **implementations** per class with `valid_from_us` / `valid_to_us` / `priority` (default 3); higher priority wins on overlap; same priority + overlapping windows is rejected at save.
- Multiple **time slots** per implementation as (`day_of_week`, `start_time`, `duration`) tuples — same class can run morning + evening on the same day, and different days can have different times / durations.
- Schedule **exceptions go away**: holidays / Memorial Day / closure weeks become short, high-priority implementations.
- Recurrence-pattern enum probably collapses to "weekly is implicit"; biweekly / custom are flagged for removal.
- Admin UI needs to swap raw IDs for friendly-name autocomplete dropdowns, time picker, day-of-week dropdown, default duration 60min.

This doc is the **design discussion + change-management plan**. Section 1 fleshes out the model. Section 2 lists open questions Mason needs to answer. Section 3 is the doc-update plan that runs *after* the open questions resolve — DO NOT touch Phase 1 / parent / siblings until the design is locked.

---

# 1. Proposed Design

## 1.1 Naming

The cleanest mapping to the `price_schedules` precedent:

| Today | Proposed | Notes |
|-------|----------|-------|
| `class_schedules` (one row = class + facility + room + recurrence + days + start_time + duration) | `class_schedules` (one row = the **implementation** — class + window + priority) | Repurposed — keeps the existing FK column name (`event_sessions.class_schedule_id`) intact. |
| n/a | `class_schedule_slots` (one row = day-of-week + start_time + duration + facility + room + capacity override) | New — each row is a single recurring slot. |

Alternatives considered: `class_timetables` / `class_timetable_entries` (fresh naming, no rename collision); `class_schedule_versions` / `class_schedule_entries`. **Recommendation: `class_schedules` + `class_schedule_slots`** because (a) the price-schedule parallel is explicit, (b) the implementation IS the schedule for that window, (c) external code (frontend, FKs, manage routes) only needs to learn one new noun (`slot`) rather than two.

Pinning the name "implementation" only in conversation / docs is also fine — the table can be `class_schedules` and the slot rows are the "implementation contents".

## 1.2 New table shape

**`class_schedules`** (REPURPOSED — the "implementation" / versioned container):

| Column                          | Type                                    | Notes                                                                                                                |
| ------------------------------- | --------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `id`                            | BIGSERIAL PK                            |                                                                                                                      |
| `class_id`                      | BIGINT NOT NULL FK                      | One class per implementation.                                                                                        |
| `name`                          | TEXT NOT NULL                           | Admin-visible label ("Default", "Holiday Week 2026", "Memorial Day").                                                |
| `priority`                      | INTEGER NOT NULL DEFAULT 3              | Higher wins on overlap. Same priority + overlap = save rejected.                                                     |
| `valid_from_us`                 | BIGINT NOT NULL                         |                                                                                                                      |
| `valid_to_us`                   | BIGINT NULL                             | NULL = open-ended. Closed when a same-priority successor is created.                                                 |
| `product_id`                    | BIGINT NOT NULL FK                      | See "What `product_id` is for" below. Default placement is at the implementation level — see OQ-CSI-2.               |
| `is_series` / `series_*` fields | (unchanged from current Phase 1 design) | One-off workshops and paid series stay on the implementation row — they ARE the schedule + window for that offering. |
| `is_active`                     | BOOLEAN NOT NULL DEFAULT TRUE           | Soft-delete flag, separate from time-window.                                                                         |
| `created_us` / `updated_us`     | BIGINT NOT NULL                         |                                                                                                                      |

Removed from the current Phase 1 design: `facility_id`, `location_room_id`, `recurrence_pattern`, `days_of_week`, `start_time_minutes`, `duration_minutes`, `effective_from_us` / `effective_to_us`, `capacity` override. These move down to the slot table OR (for the date-range fields) become the `valid_from_us` / `valid_to_us` on the implementation itself.

Also removed (per Mason's note on §59): `predecessor_class_schedule_id` — predecessor relationships are about *specific class times* requiring attendance at *another specific class time* (e.g., "Acro Level 2 at 7pm requires you to be in Acro Level 1 at 6pm the same day"). That's a slot-to-slot relationship, not an implementation-to-implementation one. Moved down to `class_schedule_slots.predecessor_class_schedule_slot_id`.

**What `product_id` is for** (answering Mason's note on §59): in the current Phase 1 design, the schedule row references a `kind='class'` product. The product carries:

1. **Per-permission pricing** (via `product_prices` × `price_schedules`) — what each membership tier pays. For included-with-membership recurring classes the per-tier price is $0; for workshops / series / intro it's the real ticket price.
2. **Visibility permissions** (via `product_visibility_permission`) — which membership tiers can even see the offering in the catalog.
3. **Booking permissions** (via `product_booking_permission`) — which membership tiers can actually book it.
4. **Cancellation policy** (via `products.cancellation_policy_id`) — refund tiers for paid bookings.
5. **Advance booking windows** (via `product_booking_windows`) — per-tier "you can book this N days before the session".

In short: `product_id` is the bridge between "this thing on the schedule" and "the entire pricing / access-control / refund machinery already built in the Payment Design layer". Without it, every class would need its own parallel access-control fields. With it, classes inherit everything from the existing product infrastructure.

Whether `product_id` belongs on the implementation or on the slot is OQ-CSI-2. The recommendation is **implementation-level** because: (a) different time-of-day slots within the same implementation are almost always the same offering at the same price, and (b) if a workshop time vs. a drop-in time really need different pricing they're conceptually different classes (different product, different schedule).

**`class_schedule_slots`** (NEW):

| Column                              | Type                                   | Notes                                                                                                                                                                  |
| ----------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                | BIGSERIAL PK                           |                                                                                                                                                                        |
| `class_schedule_id`                 | BIGINT NOT NULL FK                     | Cascade-delete from the implementation.                                                                                                                                |
| `day_of_week`                       | SMALLINT NOT NULL CHECK (0..6)         | 0=Sun..6=Sat.                                                                                                                                                          |
| `start_time_minutes`                | INTEGER NOT NULL CHECK (0..1439)       | Local-TZ minutes-after-midnight at the slot's facility.                                                                                                                |
| `duration_minutes`                  | INTEGER NOT NULL CHECK (>0) DEFAULT 60 |                                                                                                                                                                        |
| `facility_id`                       | BIGINT NOT NULL FK                     | Per-slot so a class can run at different facilities under one implementation.                                                                                          |
| `location_room_id`                  | BIGINT NOT NULL FK                     | Per-slot for the same reason.                                                                                                                                          |
| `instructor_person_id`              | BIGINT NULL FK                         | Person scheduled to teach this slot. Nullable for "TBD" (admin hasn't assigned yet). Per-session subs / trades still live on `event_session_staffing` as today.        |
| `predecessor_class_schedule_slot_id` | BIGINT NULL FK (self-ref)             | Per Mason's note on §77 — chain dependent class times (Acro 2 @ 7pm requires Acro 1 @ 6pm). Phase 3 (SL-11) is the consumer; the column exists from day 1 to avoid migrations. |
| `capacity_override`                 | INTEGER NULL                           | NULL = use `classes.default_capacity`.                                                                                                                                 |
| `created_us` / `updated_us`         | BIGINT NOT NULL                        |                                                                                                                                                                        |

No unique constraint on (`class_schedule_id`, `day_of_week`, `start_time_minutes`) — Mason explicitly called out "multiple time slots for the same class on the same day" (morning + evening) — so the same day-of-week appears multiple times. (But see OQ-CSI-8 — recommending we still reject exact duplicate tuples as data-entry errors; distinct rows must differ on at least `location_room_id` or `start_time_minutes`.)

**Skill-level requirements** (answering the second half of Mason's note on §77): the current Phase 3 design (`Classes Phase 3 - Skill Levels.md` §3.2) keys skill requirements off `class_id` via a `class_skill_requirements` table. The argument for keeping it per-class is that skill prerequisites are a property of *what the class is*, not *when it runs* — if "Advanced Acro" requires the Inversion skill, that's true at every slot, not just the Tuesday slot. The argument for per-slot is more nuanced — a beginner-friendly Saturday morning slot vs. an advanced Tuesday-night slot of the same class. In practice, that's two different classes (with two different products), not two slots of one class. **Recommendation: leave skill-level requirements per-class (Phase 3 owns this).** See OQ-CSI-11 if Mason wants to revisit.

## 1.3 Active-implementation resolution

For a given `class_id` at moment `t`:

```sql
SELECT *
FROM class_schedules
WHERE class_id = $1
  AND is_active = TRUE
  AND valid_from_us <= $2
  AND (valid_to_us IS NULL OR $2 < valid_to_us)
ORDER BY priority DESC, valid_from_us DESC
LIMIT 1;
```

Validation at save / publish time:

- Same `class_id` + same `priority` + overlapping `[valid_from_us, valid_to_us)` ranges → reject (`OVERLAPPING_SAME_PRIORITY`).
- `valid_to_us <= valid_from_us` → reject.
- Empty `class_schedule_slots` is *allowed* — that's the "closure / studio dark this week" pattern (high-priority implementation with no slots = no sessions for the covered days).

Helper method shape:

```cpp
std::optional<int64_t> GetActiveImplementationId(Transaction&, int64_t classId, int64_t atUs);
std::vector<KeyValueTable> GetActiveImplementationSlotsForDay(
    Transaction&, int64_t classId, int64_t dateUs);  // returns slots for the class's active impl on that day
```

## 1.4 Materialization

The current Phase 1 design pre-materializes `event_sessions` via an admin "Materialize through date" button. With implementations + slot tuples, the rule changes:

> **For each day D in `[from_us, through_us)` and each class C, look up the active implementation for (C, D). For each slot in that implementation matching `EXTRACT(DOW FROM D)`, create an `event_sessions` row if no equivalent row exists.**

Two UX options:

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A. Keep "Click to materialize"** | Admin runs the materializer per-class on demand. | Conservative — same UX as current Phase 1. | Easy to forget. Creating a holiday-week implementation requires a second click to apply it. |
| **B. Auto-materialize on a rolling horizon** | Background job keeps N weeks of `event_sessions` materialized for every active implementation. Saving a new implementation also re-runs the materializer over its window. | "I create an implementation and the calendar updates" feels right. | New scheduled job + edge cases (booked sessions that fall out of the new active implementation). |

Recommendation: **B** for production UX, but keep an admin "materialize now" button for ops + the test helper. The Phase 1 doc currently has the materialize button as a first-class feature — this can stay but become an "advance the horizon now" operation.

Mason- why do we want materialization? I think that the advantage of doing class schedules with active times and priority with only one active at a time is that we shouldn't need to "materialize" anything. I think that we should have a class slot instance that associates a given slot with a certain day and instantiates it "just in time" in response to one of these operations:
- Noting the class has been cancelled
- Noting a change in the instructor
- A student noting their intention to attend a specific class instance beyond their schedule template
- A student being signed in as attending a given class
- Noting a no show for a class that a student marked themselves as planning to attend that they did not attend
What do you think about this versus materialization?

## 1.5 Booked-session preservation

When a new high-priority implementation lands and overrides slots that already materialized as `event_sessions`:

- Sessions with `booked_count = 0` AND no attendance-template entries → safe to delete (and re-materialize from the new implementation).
- Sessions with `booked_count > 0` OR with template entries → preserve them and surface in the response (`sessionsKeptDueToBookings`) so admin can manually cancel/reschedule. This matches the resolved OQ-P1-3 rule from Phase 1.

Cancellations triggered by an implementation change are still admin-initiated cancellations → full refunds for paid bookings (Phase 2 / Phase 10 rules apply).

Mason- I think that this same pattern moves to instantiations. Also, there are classes that are included in memberships that users don't pay extra to attend. These don't require any kind of cancellation or refund logic. We can just get rid of the "thanks for letting me know you plan on coming" soft bookings with no muss, no fuss.

## 1.6 Recurrence-pattern collapse

Drop the `recurrence_pattern` column entirely. The implementation table is itself "weekly with these slot rows". No biweekly, no custom. Mason's concerns about biweekly are valid (odd/even week definition is ambiguous and depends on facility convention); there's no demonstrated user need for either pattern.

If the studio ever needs "every 3 weeks" or "first Saturday of each month" patterns, add them then. Until then, complex cadences are expressible by stringing together short-window implementations.

## 1.7 What this means for "scheduling exceptions"

Today's plan has scheduling exceptions (Phase 10) as a separate path: a `scheduling_exceptions` row cascade-cancels affected `event_sessions`.

With this redesign, the **per-class schedule exception** path collapses entirely:

- "No Monday-evening Knotty Yoga the week of Dec 23" = create a high-priority implementation with `valid_from = Dec 23 00:00`, `valid_to = Dec 30 00:00`, slots = (everything except Mon evening).
- "Memorial Day modified schedule" = same pattern, narrow window.
- "Just kill this one session" = stays as a per-session `SessionCancellationHelper` call (per-instance is below the granularity of an implementation).

The **studio-wide closure** path is harder — see OQ-CSI-4.

Mason- Is studio wide closure harder with this model? I think we just create a high priority but empty class schedule for closure. Can you explain why this wouldn't handle that case?

## 1.8 Admin UI changes

Per Mason's note:

- Drop raw IDs. Class / Facility / Room / Product → autocomplete dropdowns showing friendly names with the ID hidden.
- Slot row UI: day-of-week dropdown (Sun..Sat), time picker for start (NOT separated hour/minute inputs), duration input defaulting to 60min. Sorted list with add / remove buttons.
- Calendar-day preview: pick a date → show which implementation is active that day for which classes, with its resolved slot list (priority-resolved). This is the "easy to see what the active class schedule will be on a given day" view from Mason's note.
- Implementation list view: per-class, sortable by `valid_from_us` and `priority`, with visual highlighting of which implementation is currently active.

This is a meaningful UI surface area — likely a multi-component redesign of the Phase 1 admin page.

Mason- We also need to tackle the requires previous slot and skill level stuff (if we don't defer that to Classes Phase 3 - Skill Levels.md)

---

# 2. Open Questions for Mason

Tag each with a decision before Section 3 (doc updates) kicks off.

- [ ] **OQ-CSI-1 (Naming)**: Repurpose `class_schedules` to be the implementation-level + add `class_schedule_slots` (recommendation)? Or rename to `class_timetables` / `class_timetable_slots` for clarity? Or another name?
- [ ] **OQ-CSI-2 (Where does `product_id` live)**: On the implementation (recommendation — one product per implementation, all slots inherit pricing) or per-slot (lets a class have different pricing for the morning vs evening slot under one implementation)?
- [ ] **OQ-CSI-3 (Where does `facility_id` live)**: On the slot (recommendation — lets a class run at different facilities within one implementation) or on the implementation (simpler — one facility per implementation)?
- [ ] **OQ-CSI-4 (Studio-wide closures)**: Three choices:
  - (a) Batch-create empty implementations per class when admin marks a date range closed (auto-fanout in business logic).
  - (b) Keep a separate `facility_closures` mechanism (or reuse the existing `scheduling_exceptions` table) that operates above the implementation layer and short-circuits materialization.
  - (c) Both — UI is "close studio" but implementation is (a).
  - Recommendation: **(b)** because closures aren't really "schedule changes" — they apply across all classes and don't need per-class slot bookkeeping. Phase 10 still owns the cascade.
- [ ] **OQ-CSI-5 (Pre-materialization UX)**: Auto-materialize on a rolling horizon (Option B in §1.4) or keep the admin button (Option A)? Recommendation: **B with the button kept as a manual nudge**.
- [ ] **OQ-CSI-6 (Drop biweekly + custom)**: Confirm we're removing `recurrence_pattern` entirely?
- [ ] **OQ-CSI-7 (Time entry UX)**: Mason's existing memory `feedback_date_time_pickers.md` says "times must use hour pickers" and Phase 1 §6.3 ships separated hour + minute inputs. Slot start times often have minute precision (5:45 PM yoga) — do we need a full time picker (HH:MM) for slot entry, overriding the hour-only convention here? Recommendation: **yes, full time picker** because class start times in the wild are not hour-aligned (e.g. 5:45, 6:15).
- [ ] **OQ-CSI-8 (Slot uniqueness)**: Allow duplicate (`implementation_id`, `day_of_week`, `start_time_minutes`) rows or reject? Recommendation: **reject identical tuples** because that's almost certainly a data entry mistake — different facilities/rooms at the same time should be different `location_room_id` values, which makes the tuple unique anyway.
- [ ] **OQ-CSI-9 (Series + workshops)**: `is_series` + series_* fields stay on the implementation row (a single implementation IS the series window)? Or split series into a separate table? Recommendation: **keep on the implementation** — a series is just an implementation with `is_series=true`, `valid_from = series_start`, `valid_to = series_end + 1 day`, slots = the series's day/time pattern. The "implementation IS the series for that window" framing is clean.
- [ ] **OQ-CSI-10 (Phase 1 already merged?)**: Phase 1 is marked done end-to-end (most checkboxes are checked). Is this a "redesign before Phase 2 lands" plan (rewrite migrations, re-do tests) or a "Phase 1.5 migration" plan (new tables alongside, deprecation)? Recommendation: **rewrite Phase 1 in place** — pre-deploy, no production state to defend against per `feedback_no_premature_defensive_code.md`. But Mason should confirm there's no deployed environment that needs a migration path.

---

# 3. Doc Update Plan (BLOCKED until Section 2 resolves)

Once the open questions are answered, the following docs need updating. Order matters — parent first (it sets the use cases the children specialize), then Phase 1 (the schema authority), then siblings that reference the schema.

## 3.1 Parent doc updates (`Classes, schedules, and attendance.md`)

- [ ] §1 Context Recap — update the "gaps" list (#94) to note the implementation-versioned schedule model instead of "no class_schedules concept yet".
- [ ] §2.2 Class Schedule Authoring (CS-1..CS-8) — rewrite to describe implementations + slots instead of single-row schedules. CS-1 becomes "create an implementation"; CS-3 / CS-4 get the priority + window semantics; new use case CS-9 "preview active schedule on date X". CS-8 (multi-occupancy rooms) still applies but the materializer now reasons about slots.
- [ ] §2.12 Scheduling Exceptions — restructure to reflect that per-class exceptions collapse into implementations; only studio-wide closures remain a separate concept (assuming OQ-CSI-4 lands on option b).
- [ ] §3 Design Principles — add a new principle P-7 (or similar) for "versioned class-schedule implementations with priority-based override".
- [ ] §4 Alternatives Considered — add "Single flat schedule table" to the rejected list with the rationale.
- [ ] §5 Bucketing — no change expected; the use cases still belong in the same buckets.
- [ ] §6 Phase 1 — replace §1.1 / §1.2 / §1.3 / §1.4 schema + helper outlines with the new model.
- [ ] §6 Phase 10 — Scheduling Exceptions: change "per-class exception" wording to "studio closure" or remove that path entirely (depends on OQ-CSI-4); keep instructor substitution + shift trades unchanged.
- [ ] §7.5 Backwards-compat note — explicitly note the rewrite-vs-migrate decision from OQ-CSI-10.

## 3.2 Phase 1 doc updates (`Classes Phase 1 - Catalog and Schedule Authoring.md`)

This is the biggest rewrite — most §2..§7 needs touching.

- [ ] §1 Pre-Coding Design Decisions — add the resolved OQ-CSI-1..10 entries. Keep §1.1 / §1.3 (taxonomy + room conflict policy still apply).
- [ ] §2 Database Schema — full rewrite:
  - §2.1 `classes` table — unchanged.
  - §2.2 `class_schedules` — strip down to implementation columns (per §1.2 above).
  - §2.2a (new) `class_schedule_slots` table.
  - §2.3 `event_sessions` extensions — unchanged.
  - §2.4 wire into init pipeline — add the new slots table to `make_database_info.cpp` + `CreateTables()` + `db_schema/CMakeLists.txt`.
- [ ] §3 Table Helpers — rewrite:
  - `TableHelpers::ClassSchedules` — drop `GetSchedulesPotentiallyConflictingInRoom` (moved to the slot helper); add `GetActiveImplementation(classId, atUs)`, `GetImplementationsByClass(classId)`, `GetImplementationsOverlapping(classId, fromUs, toUs)`.
  - `TableHelpers::ClassScheduleSlots` (new) — full CRUD; `GetSlotsByImplementation(scheduleId)`, `GetSlotsByImplementationAndDay(scheduleId, dayOfWeek)`, `GetActiveSlotsForClassOnDay(classId, dateUs)`, `GetSlotsPotentiallyConflictingInRoom(roomId, dayOfWeek, startTimeMinutes, durationMinutes)`.
  - Tests for both with the new sort orders + conflict-detection semantics.
- [ ] §4 Business Logic — rewrite `ClassScheduleHelper`:
  - `CreateImplementation(req)` / `UpdateImplementation(...)` (validates overlap + priority).
  - `AddSlot(scheduleId, slot)` / `UpdateSlot(slotId, ...)` / `DeleteSlot(slotId)`.
  - `MaterializeFutureSessions(classId, throughDateUs)` — now walks day-by-day, asks for the active impl per day, materializes slot tuples for that day.
  - `GetActiveImplementationView(classId, dateUs)` — backs the "preview on date X" UI.
  - Drop `recurrence_pattern` validation entirely.
  - Test updates: all of `class_schedule_helper_test.cpp` needs to be re-cast around the new shape. Add tests for priority-resolution, overlap rejection, no-slot (closure) impls, multi-slot per day, per-day different times.
- [ ] §5 Endpoints — re-cast:
  - `POST /api/admin/class_schedule` (implementation create) — body shape changes.
  - `POST /api/admin/class_schedule/<id>/slot` (add slot), `PUT /api/admin/class_schedule_slot/<slotId>`, `DELETE /api/admin/class_schedule_slot/<slotId>`.
  - `GET /api/admin/class_schedules?class_id=<id>` — list implementations for a class.
  - `GET /api/admin/class_schedule_preview?class_id=<id>&date_us=<t>` — resolved active impl + slot list.
  - `POST /api/admin/class_schedule/<id>/materialize` — likely re-keys to `POST /api/admin/class/<classId>/materialize` since materialization is now per-class.
- [ ] §6 Frontend — significant rewrite of the admin UI:
  - Implementation list per class with priority + window indicators.
  - Slot editor: sorted list, day-of-week dropdown, time picker (per OQ-CSI-7), duration input, facility / room autocomplete dropdowns.
  - "Schedule on date X" preview view.
  - All component specs updated.
- [ ] §7 Admin Metadata — add `class_schedule_slots` registration (all eleven steps).
- [ ] §10 Tests Summary — expand to call out the new slot tests + priority tests.
- [ ] §11 Cross-Layer Acceptance Criteria — rewrite around: "admin creates default impl + one holiday-week impl; calendar shows holiday-week slots during that window; reverts after."
- [ ] §12 Resolved Questions — append OQ-CSI-1..10 resolutions.
- [ ] Mason-note in §357 — replaced by the rewritten body; can become a "Design Pivot Notes" appendix linking to this doc as the design source.

## 3.3 Sibling phase doc updates

- [ ] **Phase 2 (Membership-Gated Drop-In)** — §2.4 references `EventSessionHelper::GetVisibleEventSessions` surfacing `class_id` / `class_name`. The session row still has those columns; only the materialization upstream changed. Expected impact: minor — a paragraph noting the materializer is now per-day implementation-aware.
- [ ] **Phase 4 (iCal Generator Extensions)** — no model touches. No update needed.
- [ ] **Phase 5 (Attendance Templates)** — `attendance_template_entries.class_schedule_id` (§5.2) now references an implementation. Behavior should still work — a template entry binds to an implementation; if a higher-priority impl supersedes for some weeks, the materializer's slot resolution drives what gets booked. But: when the lower-priority impl resumes, the template entry is still bound to the original impl, and the user-facing semantics ("I'm signed up for Monday 6pm Knotty Yoga") need to follow the *class*, not the impl. Recommendation in Phase 5 update: bind template entries to (`person_id`, `class_id`, `slot_pattern`) rather than `class_schedule_id`. **Open question worth its own line in Phase 5's redesign**.
- [ ] **Phase 6 (Weekly Digest)** — derives from session rows; no model touch.
- [ ] **Phase 7 (Class Series and Workshops)** — series are implementations with `is_series=true`. The Phase 7 doc currently says "series uses `class_schedules.is_series=true`" — that still works since the implementation table is `class_schedules`. Expected impact: small. Verify the series-min-attendees auto-cancel job operates on the implementation's window.
- [ ] **Phase 8 (Staff Check-in)** — operates on `event_sessions`; no model touch. The "people who attended this class in the last 4 weeks" lookup still joins via `class_schedule_id → class_id`. Verify the join still works under the new model (it does — `event_sessions.class_id` is the denormalized convenience column kept from Phase 1).
- [ ] **Phase 9 (Attendance History)** — joins through `class_id`; no model touch.
- [ ] **Phase 10 (Scheduling Exceptions and Shift Trades)** — major rewrite of §10.x scheduling-exceptions sections. The per-class exception path collapses (admin uses implementations instead); only studio-wide closures remain. Instructor substitution + shift-trade sections unchanged.
- [ ] **Phase 11 (Signup Windows and Reminders)** — derives from session rows; no model touch.
- [ ] **Phase 12 (Specialty Instructor Cost)** — costs are per-implementation or per-session; per-implementation makes more sense with the redesign. Update §12.1 schema to key off `class_schedule_id` (the implementation) instead of "schedule".
- [ ] **Phases 13–16** — no expected model touches.

## 3.4 Misc

- [ ] Update `feedback_*.md` memory: probably nothing here yet — these are all already-merged decisions. Once OQ-CSI-10 is resolved, decide whether to add a memory entry noting "class_schedules is the implementation table, slots are separate".
- [ ] `MEMORY.md` index — no entry change unless we add new feedback memories.

---

# 4. Sequencing & Effort

Once Section 2 resolves:

1. **Parent doc** — half a session.
2. **Phase 1 rewrite** — a full session; touches schema, helpers, business logic, endpoints, frontend, admin metadata, tests. The code rewrite alongside (since Phase 1 is already implemented) is bigger than the doc rewrite — likely several sessions.
3. **Sibling docs** — a session for all of them combined; most are paragraph-level touches except Phase 5 (template binding semantics) and Phase 10 (exception path collapse) which need careful rethinking.

---

# 5. Cross-References

- Parent: [[Classes, schedules, and attendance]]
- Being redesigned: [[Classes Phase 1 - Catalog and Schedule Authoring]]
- Analog: `price_schedules` table + `CatalogHelper::GetActivePriceScheduleId` — see `server/knottyyoga_server/src/db_schema/price_schedules.h` and `business_logic/payment/catalog_helper.cpp:180`.
- Touched siblings: [[Classes Phase 2 - Membership-Gated Drop-In]], [[Classes Phase 5 - Attendance Templates]], [[Classes Phase 7 - Class Series and Workshops]], [[Classes Phase 10 - Scheduling Exceptions and Shift Trades]], [[Classes Phase 12 - Specialty Instructor Cost]].
