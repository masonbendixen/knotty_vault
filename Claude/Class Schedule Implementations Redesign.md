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

## 1.4 Lazy session instantiation (replaces materialization)

Mason's note on §123: "why do we want materialization? … we shouldn't need to 'materialize' anything." Agreed — the materialization model is a holdover from the flat-schedule design. With implementations + slots, the schedule IS the source of truth and persisted session rows are *deltas*, not the schedule itself.

**New model — derived sessions with lazy persistence:**

- A **derived session instance** is identified by the tuple (`class_schedule_slot_id`, `occurrence_date`). Walking days in `[from, to)`, the calendar/catalog computes derived instances by looking up the active implementation per (class, day) and expanding its matching slots.
- A row exists in `event_sessions` (or its replacement — see "table-naming wrinkle" below) ONLY when one of these triggers fires:
  1. Admin / instructor cancels the specific class instance (status='cancelled', reason, etc.)
  2. Admin / approved-trade changes the instructor for that one instance (writes to `event_session_staffing`)
  3. Admin / instructor attaches a per-session note (CS-7)
  4. A student books a paid offering (workshop / series instance / intro / guest pass)
  5. A student marks attendance-template intent that the system needs to persist (only for paid offerings — see Mason's note on §140 about no-muss-no-fuss for membership-included)
  6. Staff checks a student in (creates a `bookings` row + an `event_sessions` row if one didn't exist)
  7. Staff marks a no-show on a student who'd indicated planned attendance for a paid offering
- Absence of a row in `event_sessions` for `(class_schedule_slot_id, occurrence_date)` ⇒ "scheduled, nothing special, nothing recorded yet". Calendar / catalog render it using slot defaults from the active implementation.

**Why this is better than materialization:**

- No "materialization horizon" to advance — calendar queries are always correct out to infinity (within the active impl's window).
- No "blow away future sessions when implementation changes" cascade. New high-priority impl just *wins the lookup* for the dates it covers.
- Mason's "soft bookings for membership-included classes can vanish without muss or fuss" (§140 note) becomes trivially true — there's no booking row to manage in the first place.
- The implementation truly IS the source of truth, mirroring `price_schedules` (which we don't "materialize" — we look up the active price at query time).

**Costs / things this introduces:**

- Calendar / catalog queries are more complex: walk dates, look up active impl per day, expand slots, left-join persisted rows for cancellations / subs / notes. The query is a derived view rather than a simple `SELECT FROM event_sessions`. (Workable — encapsulate in a helper.)
- Bookings need a target. When a student books a workshop session that doesn't have an `event_sessions` row yet, the booking flow creates the row first. Same for check-in.
- Stable identity for "the Monday 6pm Knotty Yoga on 2026-06-15": the tuple (`class_schedule_slot_id`, `occurrence_date`). This is durable as long as the slot row isn't deleted from the implementation. If the slot IS deleted (admin re-edits the impl), persisted session rows that reference it become *abandoned* (their slot no longer derives them) — those rows still exist and represent real attendance / bookings, but the calendar wouldn't naturally surface them on the new impl. We need a small "orphaned sessions" admin view for that case.
- Capacity / room-conflict checks at booking time must compute against *both* derived (other slot occurrences for that date+room) AND persisted (already-booked-into rows) sessions. A `RoomOccupancyHelper` that returns "all derived + persisted sessions overlapping this time window in this room" cleanly handles it.

### Table-naming wrinkle

Today's `event_sessions` is shared between classes, services, and one-off events. Three options:

- **A. Keep `event_sessions` for everything**, but stop pre-populating class rows. Rows exist for class instances only after one of the seven triggers fires. Backwards-compatible with services / events; minimum churn. **Recommendation.**
- **B. Split out a separate `class_session_instances` table** for the lazy class rows; keep `event_sessions` for services / events. Cleaner conceptually but doubles the rendering paths in calendar / my-bookings.
- **C. Rename `event_sessions` → `session_instances`** and treat the lazy creation as the default semantics for ALL three (classes, services, events). Largest churn, but conceptually purest.

Recommend **A**. See OQ-CSI-12 if Mason wants to revisit.

### Discussion: when the active implementation changes mid-window

Open question worth its own discussion since it touches multiple paths:

- **Holiday week pre-booked workshop**: a paid workshop sits on the default impl's Tuesday-6pm slot. Admin lands a high-priority "Holiday Week" impl whose Tuesday-6pm slot is *different* (different instructor, different room) OR is *missing entirely*. The pre-paid booking has a persisted `event_sessions` row pointing at the default impl's slot. With the lazy model:
  - The booked row stays exactly where it is. The high-priority impl doesn't auto-overwrite it.
  - The calendar derivation for that date picks the high-priority impl's view; the persisted booked row is "orphaned" relative to the new impl but still real.
  - UI must decide which to show. Best answer: show the *persisted* row (it's what's actually happening), suppress the derived row, and flag both sides in the admin "orphaned" view so admin can reconcile manually.
- **Membership-included drop-in template**: per Mason's §140 note, there's no booking row, so nothing to preserve. The next-week calendar just derives from the new impl. Clean.
- **Cancellation of a slot via empty high-priority impl**: derivation for those dates returns no slots → calendar shows no class → existing persisted bookings for those dates (if any) become orphaned and must be cancelled+refunded by admin. The redesign doesn't auto-cancel; that's still an admin action because refunds need authorization.

Mason- I don't think workshops / event sessions should be affected but class schedules. They are fulfilling kind of different needs. Events / workshops aren't part of a class schedule and shouldn'

## 1.5 Booked-session preservation (under the lazy model)

Most of the old §1.5 disappears. With lazy session creation:

- Membership-included recurring class attendance: no rows to preserve — there's nothing to clean up when an impl changes (per Mason's §140 note).
- Paid bookings (workshops, series, intro, guest pass): the persisted `event_sessions` rows are unaffected by impl changes. They stay where they were booked. If a new high-priority impl conceptually overrides them, the admin sees them in the "orphaned" view and decides whether to cancel-and-refund.
- Per-instance admin actions (cancellation notes, instructor subs): same — those rows persist and stay correct.

The only "blow away" action is when admin explicitly deletes the orphaned row. That's a deliberate cancellation, handled by the existing `SessionCancellationHelper` (refund-on-cancel for paid bookings).

## 1.6 Recurrence-pattern collapse

Drop the `recurrence_pattern` column entirely. The implementation table is itself "weekly with these slot rows". No biweekly, no custom. Mason's concerns about biweekly are valid (odd/even week definition is ambiguous and depends on facility convention); there's no demonstrated user need for either pattern.

If the studio ever needs "every 3 weeks" or "first Saturday of each month" patterns, add them then. Until then, complex cadences are expressible by stringing together short-window implementations.

## 1.7 What this means for "scheduling exceptions"

Today's plan has scheduling exceptions (Phase 10) as a separate path: a `scheduling_exceptions` row cascade-cancels affected `event_sessions`.

With this redesign, the **per-class schedule exception** path collapses entirely:

- "No Monday-evening Knotty Yoga the week of Dec 23" = create a high-priority implementation with `valid_from = Dec 23 00:00`, `valid_to = Dec 30 00:00`, slots = (everything except Mon evening).
- "Memorial Day modified schedule" = same pattern, narrow window.
- "Just kill this one session" = stays as a per-session `SessionCancellationHelper` call (per-instance is below the granularity of an implementation).

**Studio-wide closure** (re-examining Mason's pushback on §160): closures work the same way — a high-priority empty implementation closes a class for the date range. The only wrinkle is that closing the *whole studio* requires creating one empty high-priority impl per class. With ~6 classes today that's tolerable manually; longer-term, a "Close studio" admin action can batch-create the empty impls in one click. I was over-counting the cost in the original draft — the model genuinely handles closures the same as any other override. OQ-CSI-4 is downgraded to "Do you want the batch-create-on-close convenience UI in Phase 1 or defer to Phase 10?".

## 1.8 Admin UI changes

Per Mason's note:

- Drop raw IDs. Class / Facility / Room / Product / Instructor → autocomplete dropdowns showing friendly names with the ID hidden.
- Slot row UI: day-of-week dropdown (Sun..Sat), time picker for start (NOT separated hour/minute inputs), duration input defaulting to 60min, instructor autocomplete (nullable / "TBD" allowed), optional predecessor-slot picker (slot autocomplete scoped to other slots on the same `day_of_week` within the same implementation — that's how same-day chaining is expressed). Sorted list with add / remove buttons.
- Calendar-day preview: pick a date → show which implementation is active that day for which classes, with its resolved slot list (priority-resolved). This is the "easy to see what the active class schedule will be on a given day" view from Mason's note.
- Implementation list view: per-class, sortable by `valid_from_us` and `priority`, with visual highlighting of which implementation is currently active.
- Orphaned-sessions admin view (per §1.4 wrinkle): list of persisted `event_sessions` rows whose `class_schedule_slot_id` is no longer present in the currently-active implementation for that date. Lets admin spot the "I edited the impl and a paid booking is now stranded" case and decide cancel-and-refund vs leave-as-is.

This is a meaningful UI surface area — likely a multi-component redesign of the Phase 1 admin page.

**Skill-level requirement entry** (responding to Mason's note on §173): per the §1.2 recommendation, skill requirements stay per-class (Phase 3 design unchanged). The skill-requirement multiselect therefore lives on the *class* edit form, not the slot. The "requires previous slot" / predecessor entry is on the slot form, as covered above. See OQ-CSI-11 if Mason wants to revisit per-slot skill requirements.

---

# 2. Open Questions for Mason

Tag each with a decision before Section 3 (doc updates) kicks off.

- [ ] **OQ-CSI-1 (Naming)**: Repurpose `class_schedules` to be the implementation-level + add `class_schedule_slots` (recommendation)? Or rename to `class_timetables` / `class_timetable_slots` for clarity? Or another name?
- [ ] **OQ-CSI-2 (Where does `product_id` live)**: On the implementation (recommendation — one product per implementation, all slots inherit pricing) or per-slot (lets a class have different pricing for the morning vs evening slot under one implementation)? See the "What `product_id` is for" subsection in §1.2 for what the column actually carries.
- [ ] **OQ-CSI-3 (Where does `facility_id` live)**: On the slot (recommendation — lets a class run at different facilities within one implementation) or on the implementation (simpler — one facility per implementation)?
- [ ] **OQ-CSI-4 (Studio-wide closures — convenience UX)**: Per §1.7's re-examination, closures handle naturally via per-class empty high-priority impls. The remaining question is purely UX:
  - (a) Ship a "Close studio for date range" admin action in Phase 1 that batch-creates empty impls across all classes in one click.
  - (b) Defer the convenience action to Phase 10; in Phase 1 admin manually creates the per-class empty impl.
  - Recommendation: **(b)** — Phase 1 is already big; the manual path works fine with 6 classes; ship the batch action with the other scheduling-exceptions work in Phase 10.
- [ ] **OQ-CSI-5 (Materialization UX)** — *superseded by OQ-CSI-12*. The lazy-instantiation model in §1.4 removes the materialization concept entirely. Leaving this as a stub so the numbering doesn't shift.
- [ ] **OQ-CSI-6 (Drop biweekly + custom)**: Confirm we're removing `recurrence_pattern` entirely?
- [ ] **OQ-CSI-7 (Time entry UX)**: Mason's existing memory `feedback_date_time_pickers.md` says "times must use hour pickers" and Phase 1 §6.3 ships separated hour + minute inputs. Slot start times often have minute precision (5:45 PM yoga) — do we need a full time picker (HH:MM) for slot entry, overriding the hour-only convention here? Recommendation: **yes, full time picker** because class start times in the wild are not hour-aligned (e.g. 5:45, 6:15).
- [ ] **OQ-CSI-8 (Slot uniqueness)**: Allow duplicate (`implementation_id`, `day_of_week`, `start_time_minutes`, `location_room_id`) tuples or reject? Recommendation: **reject identical full tuples** because that's almost certainly a data-entry mistake — different rooms at the same time are different rows, and different start times are different rows. The same room + same start_time + same day = duplicate.
- [ ] **OQ-CSI-9 (Series + workshops)**: `is_series` + series_* fields stay on the implementation row (a single implementation IS the series window)? Or split series into a separate table? Recommendation: **keep on the implementation** — a series is just an implementation with `is_series=true`, `valid_from = series_start`, `valid_to = series_end + 1 day`, slots = the series's day/time pattern. The "implementation IS the series for that window" framing is clean.
- [ ] **OQ-CSI-10 (Phase 1 already merged?)**: Phase 1 is marked done end-to-end (most checkboxes are checked). Is this a "redesign before Phase 2 lands" plan (rewrite migrations, re-do tests) or a "Phase 1.5 migration" plan (new tables alongside, deprecation)? Recommendation: **rewrite Phase 1 in place** — pre-deploy, no production state to defend against per `feedback_no_premature_defensive_code.md`. But Mason should confirm there's no deployed environment that needs a migration path.
- [ ] **OQ-CSI-11 (Skill-level requirements per-class or per-slot)** — *new, prompted by Mason's note on §77.* Today's Phase 3 design keys skill prerequisites off `class_id` (a "class" includes its required skills). With per-slot skill requirements, "Beginner Acro" Saturday morning and "Advanced Acro" Tuesday night could share a class row but have different prerequisites. Recommendation: **per-class (no change to Phase 3)** because (a) prerequisites are a property of "what the class is", and (b) genuinely different skill levels of the same activity should be different classes (different products too — different pricing typically applies). Confirm with Mason that this matches his mental model.
- [ ] **OQ-CSI-12 (Lazy instantiation vs materialization)** — *new, prompted by Mason's note on §123.* Adopt the lazy-instantiation model described in §1.4 (recommendation) or keep an explicit pre-materialization step? The recommendation is **lazy** because it (a) eliminates an entire job + admin button, (b) makes implementation changes correctly take effect with no cleanup cascade, and (c) faithfully mirrors how `price_schedules` work today. Costs: more complex calendar query, "orphaned session" handling (covered in §1.4 "Discussion"). Confirm.
- [ ] **OQ-CSI-13 (Table strategy under lazy model)** — *new.* Per §1.4 "Table-naming wrinkle": (A) keep `event_sessions` for everything and just stop pre-populating class rows (recommendation, minimum churn); (B) split out a separate `class_session_instances` table for lazy class rows; (C) rename `event_sessions` → `session_instances` and adopt lazy as the default for services / events too. Recommendation: **A**.
- [ ] **OQ-CSI-14 (Instructor scheduling at slot vs per-session)** — *new, prompted by Mason's note on §77.* Adding `instructor_person_id` to the slot row means "this is the regularly-scheduled instructor for this time slot". Per-session substitutions still ride on `event_session_staffing` (which gets populated when a sub is recorded — under the lazy model, that creates the `event_sessions` row at the same time). Confirm this two-tier model (slot = default instructor, persisted session = override) matches Mason's intent, vs. always sourcing the instructor from `event_session_staffing` even for the default case.

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
  - §2.2 `class_schedules` — strip down to implementation columns (per §1.2 above); drop `predecessor_class_schedule_id`.
  - §2.2a (new) `class_schedule_slots` table including `instructor_person_id` and `predecessor_class_schedule_slot_id`.
  - §2.3 `event_sessions` extensions — change `class_schedule_id` column to be `class_schedule_slot_id` (slot identity, not impl identity), and add an `occurrence_date_us` (date-truncated for the day this row pins). Keep `class_id` as a denormalized convenience. The composite (`class_schedule_slot_id`, `occurrence_date_us`) becomes the natural key for derived-vs-persisted lookups under §1.4's lazy model.
  - §2.4 wire into init pipeline — add the new slots table to `make_database_info.cpp` + `CreateTables()` + `db_schema/CMakeLists.txt`.
- [ ] §3 Table Helpers — rewrite:
  - `TableHelpers::ClassSchedules` — drop `GetSchedulesPotentiallyConflictingInRoom` (moved to the slot helper); add `GetActiveImplementation(classId, atUs)`, `GetImplementationsByClass(classId)`, `GetImplementationsOverlapping(classId, fromUs, toUs)`.
  - `TableHelpers::ClassScheduleSlots` (new) — full CRUD; `GetSlotsByImplementation(scheduleId)`, `GetSlotsByImplementationAndDay(scheduleId, dayOfWeek)`, `GetActiveSlotsForClassOnDay(classId, dateUs)`, `GetSlotsPotentiallyConflictingInRoom(roomId, dayOfWeek, startTimeMinutes, durationMinutes)`.
  - `TableHelpers::EventSessions` — add `LookupBySlotAndDate(slotId, occurrenceDateUs)` for the lazy lookup path; add `GetOrphanedClassSessions(classId, fromUs, toUs)` for the orphaned-session admin view.
  - Tests for all three with the new sort orders + conflict-detection semantics.
- [ ] §4 Business Logic — rewrite `ClassScheduleHelper`:
  - `CreateImplementation(req)` / `UpdateImplementation(...)` (validates overlap + priority + same-priority-no-overlap rule).
  - `AddSlot(scheduleId, slot)` / `UpdateSlot(slotId, ...)` / `DeleteSlot(slotId)` (slot CRUD).
  - **No `MaterializeFutureSessions`** under the lazy model. Replaced by `EnsureSessionExists(slotId, occurrenceDateUs)` — idempotent, called by the booking / check-in / cancel / sub paths in §1.4's trigger list.
  - `GetDerivedSessionsForRange(classId, fromUs, toUs)` — walks dates, resolves active impl per day, expands slots, left-joins persisted `event_sessions` rows. The single helper that calendar / catalog queries call into.
  - `GetActiveImplementationView(classId, dateUs)` — backs the "preview on date X" UI.
  - `GetOrphanedSessionsForRange(classId, fromUs, toUs)` — finds persisted rows whose `class_schedule_slot_id` no longer derives from the active impl for that date.
  - Drop `recurrence_pattern` validation entirely.
  - Test updates: all of `class_schedule_helper_test.cpp` needs re-casting. New tests for priority-resolution, overlap rejection, no-slot (closure) impls, multi-slot per day, per-day different times, lazy ensure-session idempotency, orphan detection, derived-vs-persisted left-join.
- [ ] §5 Endpoints — re-cast:
  - `POST /api/admin/class_schedule` (implementation create) — body shape changes.
  - `POST /api/admin/class_schedule/<id>/slot` (add slot), `PUT /api/admin/class_schedule_slot/<slotId>`, `DELETE /api/admin/class_schedule_slot/<slotId>`.
  - `GET /api/admin/class_schedules?class_id=<id>` — list implementations for a class.
  - `GET /api/admin/class_schedule_preview?class_id=<id>&date_us=<t>` — resolved active impl + slot list.
  - `GET /api/admin/orphaned_class_sessions?class_id=<id>&from_us=&to_us=` — backs the orphaned-session admin view.
  - **Remove** `POST /api/admin/class_schedule/<id>/materialize` — no longer applicable under lazy instantiation.
- [ ] §6 Frontend — significant rewrite of the admin UI:
  - Implementation list per class with priority + window indicators (currently-active highlighted).
  - Slot editor: sorted list, day-of-week dropdown, time picker (per OQ-CSI-7), duration input, facility / room / instructor autocomplete dropdowns, optional predecessor-slot picker.
  - "Schedule on date X" preview view.
  - Orphaned-session admin view + per-row "cancel + refund" action (delegates to `SessionCancellationHelper`).
  - Remove the materialize dialog component entirely.
  - All component specs updated.
- [ ] §7 Admin Metadata — add `class_schedule_slots` registration (all eleven steps).
- [ ] §10 Tests Summary — expand to call out the new slot tests + priority tests + lazy-instantiation tests + orphan-detection tests; remove materialize tests.
- [ ] §11 Cross-Layer Acceptance Criteria — rewrite around: "admin creates default impl with three slots, adds one holiday-week empty-impl override, calendar shows holiday-week behavior during the window, reverts after, and no admin click-to-materialize is required at any point."
- [ ] §12 Resolved Questions — append OQ-CSI-1..14 resolutions.
- [ ] Mason-note in §357 — replaced by the rewritten body; can become a "Design Pivot Notes" appendix linking to this doc as the design source.

## 3.3 Sibling phase doc updates

- [ ] **Phase 2 (Membership-Gated Drop-In)** — bigger impact than originally thought under the lazy model. `EventSessionHelper::GetVisibleEventSessions` now needs to return *derived* sessions (no row exists yet) alongside persisted ones. The booking flow under §1.4's trigger #4 must call `EnsureSessionExists` to materialize the row at booking time. Per-tier price resolution still flows through the implementation's `product_id`. Expected impact: moderate — Phase 2 doc needs a "lazy ensure on booking" subsection.
- [ ] **Phase 4 (iCal Generator Extensions)** — no model touches. No update needed.
- [ ] **Phase 5 (Attendance Templates)** — bigger impact. Per Mason's §140 note, membership-included template entries don't create any persisted rows at all (no booking, no `event_sessions`). The template entry becomes pure intent. Bind template entries to (`person_id`, `class_schedule_slot_id`) — if the slot is overridden by a higher-priority impl on a given week, the user simply gets nothing for that week (matches their experience: the class isn't running this week). Per-instance exceptions (AT-5 / AT-6) likewise stay as pure data without backing bookings. Phase 5's entire "auto-create bookings on materialization" subsection disappears. Open question for Phase 5: does the template entry follow the slot or follow the class? Recommendation: **follow the slot** — slots have stable identity; if admin deletes a slot, the template entry is orphaned and surfaced in the user's portal as "this slot no longer exists, want to pick a new one?".
- [ ] **Phase 6 (Weekly Digest)** — bigger impact. The digest's "this week's classes" lookup must derive from impl + slots PLUS persisted rows, not just `SELECT FROM bookings`. For membership-included template attendance there are no booking rows; the digest must compute "today the user has a Monday 6pm Knotty Yoga via their template against the active impl's Monday 6pm slot". Add a `WeeklyDigestHelper::GetTemplateOccurrencesForWeek(personId, weekStartUs)` that walks the user's template entries against the active impls.
- [ ] **Phase 7 (Class Series and Workshops)** — series are implementations with `is_series=true`. Phase 7 buys a whole series → ensures `event_sessions` rows for every occurrence in the series window (paid bookings can't be lazy — they're real money). So series buys remain materializing-at-purchase. Phase 7 doc adds a "series purchase = ensure all sessions at purchase time" rule. The series-min-attendees auto-cancel job runs against the persisted rows from that purchase.
- [ ] **Phase 8 (Staff Check-in)** — small impact. Check-in is trigger #6 from §1.4 — staff check-in calls `EnsureSessionExists` if the row doesn't exist. The "people who attended this class in the last 4 weeks" lookup joins through `event_sessions.class_id` as before (denormalized column unchanged). Add a paragraph noting the ensure-on-checkin step.
- [ ] **Phase 9 (Attendance History)** — small impact. History reads from persisted rows only (no derived view needed — attendance only exists for sessions that were checked in, which means they were ensured). No model touch beyond keying off `class_schedule_slot_id` instead of `class_schedule_id` if any join uses that column.
- [ ] **Phase 10 (Scheduling Exceptions and Shift Trades)** — major rewrite. Per-class exceptions collapse to implementations. Studio-wide closure batch action is the OQ-CSI-4 deferred work — Phase 10 ships the convenience UI. Instructor substitution becomes trigger #2 from §1.4 — sub action calls `EnsureSessionExists` then writes `event_session_staffing`. Shift trades likewise. Document the ensure-on-action pattern as the Phase 10 invariant.
- [ ] **Phase 11 (Signup Windows and Reminders)** — small impact. "Sessions available to book on date X" now derives from the active impl + slots. The reminder system queries derived sessions to figure out when a user can book.
- [ ] **Phase 12 (Specialty Instructor Cost)** — costs are per-implementation or per-session; per-implementation makes more sense with the redesign. Update §12.1 schema to key off `class_schedule_id` (the implementation) instead of "schedule". Optionally also support per-slot cost overrides (different rates for the morning vs evening slot of a specialty instructor).
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
