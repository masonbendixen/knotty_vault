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

Mason flagged confusion on §173: workshops and one-off events don't "extend into perpetuity"; they have a single date range and there's no holiday-week override to worry about. Right — I conflated two cases in the original write-up. Let me re-frame:

- **Workshops / one-off events**: a workshop has its own bounded impl (single slot, narrow window). There is no "default impl" to override. A holiday landing during a workshop's window doesn't change the workshop because the workshop IS its own impl — no other impl coexists with it for that class. The orphan-on-impl-change concern doesn't apply.
- **Series**: same — a series has its own bounded impl. Holiday overrides during a series window are a separate question (see §1.5a below for Mason's "Labor Day skipped during a Fall Series" use case) but they apply *within* the series's own impl management, not by an external holiday-week impl reaching in.
- **Recurring membership-included classes**: this is where impl overrides actually matter. The "default Monday 6pm Knotty Yoga" recurring class has a default impl, and a higher-priority "Holiday Week" impl can replace it for a date range. Since membership-included attendance creates no persisted rows (per Mason's §140 note), there's almost nothing to preserve on impl change — the next week just derives from the new impl.

So the orphan story shrinks to almost nothing:

- **Recurring membership-included classes**: no bookings exist (template intent only, no persisted booking rows). The only persisted rows for future dates would be admin-attached notes (CS-7) or manual instructor subs against a specific date — and per Mason's §175 note, those are ephemeral; the appropriate response when an impl change supersedes them is to **just delete the stale future-date rows on impl save**. No admin orphan-recovery UI needed. The new impl owns the future from the moment it's saved.
- **Workshops / series**: their impls don't overlap with anything (they're bounded one-off impls under their own `classes` rows per §1.5a). There's no "higher-priority impl reaches in and orphans my paid workshop" scenario because no other impl coexists with the workshop's impl for that class.
- **Past sessions** (already attended / checked in): impl changes never reach backward in time. Past rows are untouched.

**Net design**: on impl save, sweep persisted future-date `event_sessions` rows for that class where `class_schedule_slot_id` is no longer in the new active impl. For rows whose only content is admin actions (notes, manual subs) and which carry no `purchase_id` / `booking`, delete. For rows that DO carry a `purchase_id` (which shouldn't happen for recurring classes by design, since membership-included classes don't create bookings — but defensive check), refuse the impl save and surface those rows to admin for explicit cancel-and-refund first. Mason's "we don't need orphans" applies because the membership-class case generates only deletable rows.

The "orphaned sessions admin view" from the prior draft is dropped. The sweep is automatic.

## 1.5 Booked-session preservation (under the lazy model)

Most of the old §1.5 disappears. With lazy session creation + §1.4's automatic orphan sweep:

- **Membership-included recurring class attendance**: no bookings exist (template intent only). Any stale future-date admin actions (notes, manual subs) are auto-deleted on impl save per Mason's §175 note. Nothing to preserve.
- **Paid bookings (workshops, series, intro, guest pass)**: workshops and series live under their own `classes` rows with their own bounded impls — no other impl coexists with them, so the "orphan a paid booking via impl change" path doesn't exist in practice. Their persisted `event_sessions` rows are simply unaffected by impl edits to OTHER classes.
- **Defensive guard**: the impl-save sweep refuses to delete any row carrying a `purchase_id` and instead asks admin to cancel-and-refund first via `SessionCancellationHelper`. This catches any accidental cross-wired case (e.g., a recurring-class row that somehow ended up with a paid booking — shouldn't happen, but worth a guard).

No standing admin "orphan recovery" UI is needed. Cancel-and-refund of a specific paid offering remains the existing `SessionCancellationHelper` flow.

## 1.5a Recurring class vs workshop vs series — how each uses this infrastructure

Mason's §185 note flagged that recurring membership classes and paid bundle offerings (series, workshops, intro) have fundamentally different lifecycles, and asked whether they share infrastructure or split. Walking through it carefully:

### The three offering shapes

| Offering | Lifecycle | Pricing | Persistence | User intent |
|----------|-----------|---------|-------------|-------------|
| **Recurring membership class** | Forever (no end date); evolves via versioned impls | $0 to user; gated by tier permission | Lazy — no booking unless cancelled / sub'd / noted | Attendance template; show up on the day |
| **Workshop** | Single date, single slot, narrow window | Per-tier ticket; non-members allowed (M-12 intro is a workshop) | Persisted at purchase | Pay once, attend once |
| **Series** | Bounded date range, multiple slots/occurrences within range | Per-tier lump sum, derived from per-session base × occurrence count | Persisted at purchase across all session occurrences | Pay once, attend all (or pro-rated subset) |

### Recommendation: shared schedule infrastructure; the `classes` row is the marketing identity, and each impl is a single run

Per Mason's §209 note: a series like "Intro to Partner Acro" runs repeatedly (Fall 2026, Spring 2027, Summer 2027), and the marketing identity (name, description, photo) is shared across all runs. The series catalog should have a page for "Intro to Partner Acro" with upcoming instances listed underneath. Same logic applies to workshops that get repeated — "Inversion Workshop" run in March and again in October share marketing identity. So:

> **The `classes` row is the offering's marketing identity. Each run is a separate bounded `class_schedules` impl under that same class.**

This unifies the three shapes:

- **Recurring membership class** = `classes` row "Knotty Yoga" + `kind='class'` product + open-ended `class_schedules` impl (`valid_to_us = NULL`) + slots like Mon/Wed 6pm. Multiple impls over time express the schedule's evolution + holiday overrides via priority.
- **Workshop** = `classes` row "Inversion Workshop" + `kind='workshop'` product + one bounded `class_schedules` impl per run (single slot, narrow window). Repeated runs are additional impls under the same `classes` row, each with its own bounded window. The catalog shows the workshop's marketing page with a list of upcoming runs.
- **Series** = `classes` row "Intro to Partner Acro" + `kind='class_series'` product + one bounded `class_schedules` impl per instance (with the instance's day/time pattern) + one `class_series_offerings` row per impl (Phase 7). Repeated runnings ("Fall 2026", "Spring 2027") are additional impl + offering pairs under the same `classes` row. Holiday overrides for a specific instance (Mason's Labor Day case) are extra higher-priority impls layered on top of that instance's default impl.

### How the catalog renders each kind

- The recurring class detail page renders upcoming sessions derived from the active impl, no list of "runs".
- The workshop / series catalog detail page renders the marketing copy + a list of **upcoming runs** (the bounded impls). Each run links to its own detail page or expanded panel showing its slot schedule + (for series) the per-tier price for that instance.
- A `classes.kind` enum (`recurring` | `workshop` | `series`) discriminates the rendering path. Phase 1 introduces the enum; Phase 7 fills in the series-instance UI.

### Why a single `classes` row across runs (and walk-back of the prior "separate row per run" recommendation)

The original §1.5a draft said each workshop run / series instance got its own `classes` row. Mason's §209 note correctly pushes back: that forces marketing copy to be duplicated across runs, which both wastes admin effort and means edits to the description don't propagate. Sharing the `classes` row across runs:

- Lets admin write "Intro to Partner Acro" copy once and have it apply to every upcoming run.
- Lets the catalog naturally surface a list of upcoming runs under one page.
- Keeps the recurring class vs workshop vs series distinction implicit in the impl pattern (open-ended vs bounded) and the product kind, NOT in the `classes` row.

The only thing that varies per run is the impl's `[valid_from, valid_to]` window, the slot pattern (if it differs across runs), and the per-instance series pricing in `class_series_offerings` (Phase 7). Marketing identity stays shared.

If a particular run of a series legitimately needs distinct copy ("this run includes a guest instructor"), the impl gets a `name` field (which it already has — "Fall 2026 with Guest Mary") that the catalog renders as a sub-title under the parent class's marketing copy. No need to fork the `classes` row.

This affects OQ-CSI-15, which is reframed below.

### Where series-instance pricing lives

A new `class_series_offerings` table (introduced in Phase 7, not Phase 1) holds the per-instance bundle machinery. One row per impl that's a series instance:

| Column | Purpose |
|--------|---------|
| `id` | PK |
| `class_schedule_id` | The impl this offering corresponds to (1:1). The parent `classes` row reaches back through `class_schedules.class_id`. |
| `product_id` | `kind='class_series'`. Per-tier pricing on the product is the **per-session base** (not the lump-sum total — see below). |
| `min_attendees` / `min_by_us` / `min_not_met_policy` | The min-cancel mechanism. |
| `prorated_signups_allowed` | Boolean. |
| `created_us` / `updated_us` | |

Price at purchase = (count of derived session occurrences in this impl's window from the active impls for this class) × (per-tier base from `product_prices`). Mason's Labor Day case naturally falls out: admin creates a high-priority empty-Monday impl over Labor Day → that Monday's slot doesn't derive → occurrence count drops by one → price drops by one base.

### What this changes in Phase 1

Phase 1 owns recurring-class scheduling. Workshops and series are *enabled* by this infrastructure (their schedule impls use the same table), but their *product machinery* (series purchasing, min-attendee policy, prorating) is Phase 7's job. Phase 1 doesn't need to ship the `class_series_offerings` table or the series-purchase flow.

Concretely for Phase 1:

- **Add `classes.kind` enum** (`recurring` | `workshop` | `series`) to discriminate catalog rendering. Defaults to `recurring`. Workshops + series detail pages list their upcoming runs (bounded impls); recurring class pages render the active impl's slot schedule.
- **Drop `is_series` + all `series_*` columns from the Phase 1 `class_schedules` table** — they were leftovers from the original plan that conflated "the schedule" with "the bundle product". Phase 7 introduces `class_series_offerings` separately.
- **Workshops + series share the `classes` row across runs**, per Mason's §209 note. Each run is a separate bounded `class_schedules` impl. Phase 1's admin UI must support creating multiple bounded impls under one `classes` row.
- **The intro workshop (M-12)** is a workshop in this framing: a `classes` row of `kind='workshop'`, with one or more bounded impls (one per run), and a `kind='workshop'` product. Non-member-allowed product permissions distinguish it from other workshops.

See OQ-CSI-9 (now reframed), the new OQ-CSI-15 (reframed), OQ-CSI-16, and OQ-CSI-18 (new — the `classes.kind` enum) for confirmation points.

## 1.6 Recurrence-pattern collapse

Drop the `recurrence_pattern` column entirely. The implementation table is itself "weekly with these slot rows". No biweekly, no custom. Mason's concerns about biweekly are valid (odd/even week definition is ambiguous and depends on facility convention); there's no demonstrated user need for either pattern.

If the studio ever needs "every 3 weeks" or "first Saturday of each month" patterns, add them then. Until then, complex cadences are expressible by stringing together short-window implementations.

## 1.7 What this means for "scheduling exceptions"

Today's plan has scheduling exceptions (Phase 10) as a separate path: a `scheduling_exceptions` row cascade-cancels affected `event_sessions`.

With this redesign, the **per-class schedule exception** path collapses entirely:

- "No Monday-evening Knotty Yoga the week of Dec 23" = create a high-priority implementation with `valid_from = Dec 23 00:00`, `valid_to = Dec 30 00:00`, slots = (everything except Mon evening).
- "Memorial Day modified schedule" = same pattern, narrow window.
- "Just kill this one session" = stays as a per-session `SessionCancellationHelper` call (per-instance is below the granularity of an implementation).

**"Studio closure"** (re-examining Mason's pushback on §160 and §205): there is no global "studio is closed" lever. Per Mason's §205 note, even on a closure day the studio might still run a workshop or a specialty class. So:

- Each recurring class independently gets (or doesn't get) a high-priority empty impl for the closure window. Admin picks which classes are affected.
- Workshops scheduled during the closure window are NOT suppressed — they're their own impls under their own `classes` rows (per §1.5a), and the closure impls only apply to the recurring classes that were closed.
- The convenience UI is therefore "Close these N classes for this date range" with a class-multiselect — NOT "close the studio for this date range".

OQ-CSI-4 is reframed: do we ship the "close-these-classes" batch action in Phase 1 or Phase 10? Recommendation remains **defer to Phase 10** (Phase 1 already large; per-class manual workflow works for the small class roster today).

## 1.8 Admin UI changes

Per Mason's notes:

- Drop raw IDs. Class / Facility / Room / Product / Instructor → autocomplete dropdowns showing friendly names with the ID hidden.
- Slot row UI: day-of-week dropdown (Sun..Sat), time picker for start (NOT separated hour/minute inputs), duration input defaulting to 60min, instructor autocomplete (nullable / "TBD" allowed), optional predecessor-slot picker (slot autocomplete scoped to other slots on the same `day_of_week` within the same implementation — that's how same-day chaining is expressed). Sorted list with add / remove buttons.
- Class detail (admin): the page for a single `classes` row shows its kind (recurring / workshop / series), its impls list, and per-class metadata. For workshops + series, "impls" reads as "upcoming and past runs". For recurring, "impls" reads as "schedule versions over time".
- Implementation list view: per-class, sortable by `valid_from_us` and `priority`, with visual highlighting of which implementation is currently active. "Add new impl" supports both "new run" (for workshop / series — bounded) and "schedule replacement / override" (for recurring — open-ended or higher-priority bounded).
- Calendar-day preview: pick a date → show which implementation is active that day for which classes, with its resolved slot list (priority-resolved). This is the "easy to see what the active class schedule will be on a given day" view from Mason's note.
- Impl-save sweep notice: when saving an impl that orphans future-date persisted rows on the affected class, the save action surfaces a confirmation ("X future admin notes / subs will be removed by this change") before proceeding. No standalone orphan-recovery UI per §1.4 / Mason's §175 note — the sweep is part of the impl-save flow itself.

This is a meaningful UI surface area — likely a multi-component redesign of the Phase 1 admin page.

**Skill-level requirement entry** (responding to Mason's note on §173): per the §1.2 recommendation, skill requirements stay per-class (Phase 3 design unchanged). The skill-requirement multiselect therefore lives on the *class* edit form, not the slot. The "requires previous slot" / predecessor entry is on the slot form, as covered above. See OQ-CSI-11 if Mason wants to revisit per-slot skill requirements.

---

# 2. Open Questions for Mason

Tag each with a decision before Section 3 (doc updates) kicks off.

- [ ] **OQ-CSI-1 (Naming)**: Repurpose `class_schedules` to be the implementation-level + add `class_schedule_slots` (recommendation)? Or rename to `class_timetables` / `class_timetable_slots` for clarity? Or another name?
- [ ] **OQ-CSI-2 (Where does `product_id` live)**: On the implementation (recommendation — one product per implementation, all slots inherit pricing) or per-slot (lets a class have different pricing for the morning vs evening slot under one implementation)? See the "What `product_id` is for" subsection in §1.2 for what the column actually carries.
- [ ] **OQ-CSI-3 (Where does `facility_id` live)**: On the slot (recommendation — lets a class run at different facilities within one implementation) or on the implementation (simpler — one facility per implementation)?
- [ ] **OQ-CSI-4 (Closure batch UX)**: Per §1.7's re-examination and Mason's §205 note, closures are per-class empty high-priority impls; there is no "studio is closed" global lever (workshops can still run). The remaining question is purely UX:
  - (a) Ship a "Close these N classes for this date range" multiselect admin action in Phase 1.
  - (b) Defer the convenience UI to Phase 10; in Phase 1 admin manually creates per-class empty impls.
  - Recommendation: **(b)** — Phase 1 is already big; ship the batch action with the other scheduling-exception work in Phase 10.
- [ ] **OQ-CSI-5 (Materialization UX)** — *superseded by OQ-CSI-12*. The lazy-instantiation model in §1.4 removes the materialization concept entirely. Leaving this as a stub so the numbering doesn't shift.
- [ ] **OQ-CSI-6 (Drop biweekly + custom)**: Confirm we're removing `recurrence_pattern` entirely?
- [ ] **OQ-CSI-7 (Time entry UX)**: Mason's existing memory `feedback_date_time_pickers.md` says "times must use hour pickers" and Phase 1 §6.3 ships separated hour + minute inputs. Slot start times often have minute precision (5:45 PM yoga) — do we need a full time picker (HH:MM) for slot entry, overriding the hour-only convention here? Recommendation: **yes, full time picker** because class start times in the wild are not hour-aligned (e.g. 5:45, 6:15).
- [ ] **OQ-CSI-8 (Slot uniqueness)**: Allow duplicate (`implementation_id`, `day_of_week`, `start_time_minutes`, `location_room_id`) tuples or reject? Recommendation: **reject identical full tuples** because that's almost certainly a data-entry mistake — different rooms at the same time are different rows, and different start times are different rows. The same room + same start_time + same day = duplicate.
- [ ] **OQ-CSI-9 (Series + workshops — REVISED)**: *Reframed per Mason's §185 note + §1.5a.* The original "is_series + series_* on the implementation row" was conflating scheduling with the bundle-product concept. Recommendation now: **drop `is_series` and all `series_*` columns from `class_schedules` in Phase 1.** Schedule impls describe pure scheduling. Series purchasing, min-attendee policy, and pro-rating live in a separate `class_series_offerings` table introduced in Phase 7. Confirm.
- [ ] **OQ-CSI-10 (Phase 1 already merged?)**: Phase 1 is marked done end-to-end (most checkboxes are checked). Is this a "redesign before Phase 2 lands" plan (rewrite migrations, re-do tests) or a "Phase 1.5 migration" plan (new tables alongside, deprecation)? Recommendation: **rewrite Phase 1 in place** — pre-deploy, no production state to defend against per `feedback_no_premature_defensive_code.md`. But Mason should confirm there's no deployed environment that needs a migration path.
- [ ] **OQ-CSI-11 (Skill-level requirements per-class or per-slot)** — *new, prompted by Mason's note on §77.* Today's Phase 3 design keys skill prerequisites off `class_id` (a "class" includes its required skills). With per-slot skill requirements, "Beginner Acro" Saturday morning and "Advanced Acro" Tuesday night could share a class row but have different prerequisites. Recommendation: **per-class (no change to Phase 3)** because (a) prerequisites are a property of "what the class is", and (b) genuinely different skill levels of the same activity should be different classes (different products too — different pricing typically applies). Confirm with Mason that this matches his mental model.
- [ ] **OQ-CSI-12 (Lazy instantiation vs materialization)** — *new, prompted by Mason's note on §123.* Adopt the lazy-instantiation model described in §1.4 (recommendation) or keep an explicit pre-materialization step? The recommendation is **lazy** because it (a) eliminates an entire job + admin button, (b) makes implementation changes correctly take effect with no cleanup cascade, and (c) faithfully mirrors how `price_schedules` work today. Costs: more complex calendar query, "orphaned session" handling (covered in §1.4 "Discussion"). Confirm.
- [ ] **OQ-CSI-13 (Table strategy under lazy model)** — *new.* Per §1.4 "Table-naming wrinkle": (A) keep `event_sessions` for everything and just stop pre-populating class rows (recommendation, minimum churn); (B) split out a separate `class_session_instances` table for lazy class rows; (C) rename `event_sessions` → `session_instances` and adopt lazy as the default for services / events too. Recommendation: **A**.
- [ ] **OQ-CSI-14 (Instructor scheduling at slot vs per-session)** — *new, prompted by Mason's note on §77.* Adding `instructor_person_id` to the slot row means "this is the regularly-scheduled instructor for this time slot". Per-session substitutions still ride on `event_session_staffing` (which gets populated when a sub is recorded — under the lazy model, that creates the `event_sessions` row at the same time). Confirm this two-tier model (slot = default instructor, persisted session = override) matches Mason's intent, vs. always sourcing the instructor from `event_session_staffing` even for the default case.
- [ ] **OQ-CSI-15 (Separate `classes` rows for workshops and series)** — *new, prompted by Mason's §185 note.* Per §1.5a, a workshop / series gets its own `classes` row distinct from any recurring class it might thematically descend from. This means "Inversion Workshop Aug 15" and "Vinyasa Fall Series Sept–Oct 2026" each have their own row in `classes` with their own name / description / photo. Alternative: share a `classes` row with the related recurring class. Recommendation: **separate rows** because marketing identity differs and the same activity might spawn multiple bounded offerings per year. Confirm.
- [ ] **OQ-CSI-16 (Where the series bundle machinery lives)** — *new, prompted by Mason's §185 note.* Per §1.5a, the series-purchase mechanics (`min_attendees`, `min_by_us`, `min_not_met_policy`, `prorated_signups_allowed`, `per_session_base_cents` per tier) belong in a new `class_series_offerings` table introduced in Phase 7, NOT on `class_schedules` in Phase 1. Confirm that Phase 1 ships purely scheduling and Phase 7 owns the bundle product. Open follow-up: does the series's session set derive from the *same* `class_schedules` impls the recurring class uses, or does the series have its OWN impls under its own `classes` row? Recommendation: **own impls under its own `classes` row** so admin can independently shape the series schedule without disturbing the recurring class (e.g., add the Labor Day empty-Monday impl to the series's class without affecting the regular Vinyasa Flow recurring schedule). Confirm.
- [ ] **OQ-CSI-17 (Closure scope confirmation)** — *new, prompted by Mason's §205 note.* Per §1.7's reframe, there is no global "studio closure" lever; closures are per-class empty high-priority impls and workshops happening on the closure date are unaffected. The Phase 10 batch UI is therefore a class-multiselect ("close these N recurring classes for this window"). Confirm there's no scenario where Mason wants a truly global suppress-everything closure.

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
  - §2.1 `classes` table — unchanged. Note that workshops and series get their own `classes` rows per §1.5a / OQ-CSI-15.
  - §2.2 `class_schedules` — strip down to implementation columns (per §1.2 above); drop `predecessor_class_schedule_id`; **drop `is_series` and all `series_*` columns** (per §1.5a / OQ-CSI-9 reframe — series bundle machinery moves to `class_series_offerings` in Phase 7).
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
- [ ] **Phase 7 (Class Series and Workshops)** — bigger impact, fundamental reframe per §1.5a. The Phase 7 doc currently treats series as "`class_schedules.is_series=true`"; this is dropped. Instead:
  - Each series gets its OWN `classes` row + its OWN bounded `class_schedules` impl(s) under that class (default impl plus any holiday-override impls for the series window — Mason's Labor Day Monday case).
  - A new `class_series_offerings` table holds the series-bundle product fields: `class_id`, `valid_from_us`, `valid_to_us`, `product_id` (`kind='class_series'`), `per_session_base_cents` (per tier via `product_prices`), `min_attendees`, `min_by_us`, `min_not_met_policy`, `prorated_signups_allowed`.
  - Price at purchase = (count of derived session occurrences in window from the series's active impls) × (per-tier base). Empty high-priority impls during the series window naturally reduce the count.
  - Series purchase = ensure `event_sessions` rows for every derived occurrence at purchase time (paid bookings can't be lazy — they're real money). Sessions get `series_purchase_id` set so the min-attendees auto-cancel job can find them.
  - Workshops follow the same shape (own `classes` row + bounded impl + `kind='workshop'` product), minus the per-session pricing and min-attendees machinery.
  - The M-12 intro workshop is treated identically to any other workshop, with non-member-allowed product permissions.
- [ ] **Phase 8 (Staff Check-in)** — small impact. Check-in is trigger #6 from §1.4 — staff check-in calls `EnsureSessionExists` if the row doesn't exist. The "people who attended this class in the last 4 weeks" lookup joins through `event_sessions.class_id` as before (denormalized column unchanged). Add a paragraph noting the ensure-on-checkin step.
- [ ] **Phase 9 (Attendance History)** — small impact. History reads from persisted rows only (no derived view needed — attendance only exists for sessions that were checked in, which means they were ensured). No model touch beyond keying off `class_schedule_slot_id` instead of `class_schedule_id` if any join uses that column.
- [ ] **Phase 10 (Scheduling Exceptions and Shift Trades)** — major rewrite. Per-class exceptions collapse to implementations. Per Mason's §205 note + OQ-CSI-17, the closure batch UI is "close these N recurring classes for this window" (class-multiselect), NOT a global "studio is closed" lever — workshops scheduled during the window are unaffected. Instructor substitution becomes trigger #2 from §1.4 — sub action calls `EnsureSessionExists` then writes `event_session_staffing`. Shift trades likewise. Document the ensure-on-action pattern as the Phase 10 invariant.
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
