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

Three-level hierarchy (LOCKED per discussion on Mason's §214 note):

```
classes  (marketing identity — name, description, photo, kind)
   └── class_instances  (a run — own product, validity window, optional series-bundle)
          └── class_schedules  (versioned impls under one instance — priority + window)
                 └── class_schedule_slots  (recurring day/time tuples)
```

**`classes`** (mostly unchanged — adds `kind` enum):

| Column | Type | Notes |
|--------|------|-------|
| `id` / `name` / `description` / `default_capacity` / `is_active` / `created_us` / `updated_us` | (existing Phase 1 columns) | |
| `kind` | TEXT NOT NULL DEFAULT `'recurring'` | Enum: `recurring` | `workshop` | `series`. Discriminates catalog rendering: recurring → "upcoming sessions from the active impl on the perpetual instance"; workshop / series → "list of upcoming instances (runs)". Note: `recurring` membership classes still ride the instance layer for symmetry (the perpetual instance is 1:1 with the class). |

`product_id` is NOT on `classes` — it lives on `class_instances` so that product migrations (changing membership tier permissions, cancellation policy, etc.) happen by closing one instance and opening a new one, without touching the marketing identity.

**`class_instances`** (NEW — the per-run / per-offering layer):

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL PK | |
| `class_id` | BIGINT NOT NULL FK | The parent marketing identity. |
| `name` | TEXT NOT NULL | Admin-visible label — "Perpetual" for recurring classes; "May 2026", "Fall 2026 with Guest Mary", "Aug 15 2026 Inversion Workshop" for runs. Surfaces on admin and (for workshops / series) in catalog as a sub-title. |
| `valid_from_us` | BIGINT NOT NULL | Start of this run. |
| `valid_to_us` | BIGINT NULL | End of this run. NULL = open-ended (the perpetual recurring case). |
| `product_id` | BIGINT NOT NULL FK | See "What the product carries" below. Per-instance so product migrations are clean (close old instance + open new one). Per-tier pricing flows through `product_prices` × `price_schedules` — pure price changes don't require new instances. |
| `is_active` | BOOLEAN NOT NULL DEFAULT TRUE | Soft-delete flag. |
| `created_us` / `updated_us` | BIGINT NOT NULL | |

Validation at save:

- For `classes.kind = 'recurring'`, at most one instance per class with `is_active = TRUE` AND `valid_to_us = NULL` (the perpetual instance). Migrations close one and open another.
- Two `is_active` instances under the same class with overlapping `[valid_from_us, valid_to_us)` ranges → reject. Instances don't overlap; impls (under instances) DO overlap (priority resolves).
- `valid_to_us <= valid_from_us` → reject.

**What the product on `class_instances.product_id` carries** (carrying forward from the prior §1.2 subsection but at the instance layer now):

1. **Per-permission pricing** via `product_prices` × `price_schedules`. Price evolves over time via `price_schedules` without needing instance migration — exactly the same way today's products work. The buyer pays the active price at the moment of purchase.
2. **Visibility permissions** — which tiers can see the offering in the catalog.
3. **Booking permissions** — which tiers can book.
4. **Cancellation policy** — refund tiers for paid bookings.
5. **Advance booking windows** — per-tier "you can book this N days before the session".
6. **Product variants** — for parallel shapes of an instance (single seat / couple seat / family seat). Variants apply to instances; temporal succession does NOT use variants (it uses instance close+open).

**`class_schedules`** (REPURPOSED — the impl / versioned schedule under an instance):

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL PK | |
| `class_instance_id` | BIGINT NOT NULL FK | The parent instance. Active-impl resolution is scoped to an instance (not directly to a class). |
| `name` | TEXT NOT NULL | "Default schedule", "Memorial Day", "Holiday Week". Admin-visible. |
| `priority` | INTEGER NOT NULL DEFAULT 3 | Higher wins on overlap *within the same instance*. Same priority + overlap (same instance) = rejected. |
| `valid_from_us` | BIGINT NOT NULL | Must fall within the parent instance's window. |
| `valid_to_us` | BIGINT NULL | Must fall within the parent instance's window. NULL = up to the instance's `valid_to_us` (or open if the instance is open). |
| `is_active` | BOOLEAN NOT NULL DEFAULT TRUE | Soft-delete flag, separate from time-window. |
| `created_us` / `updated_us` | BIGINT NOT NULL | |

Removed since the original Phase 1 design: `class_id` (now reached via instance), `product_id` (now on instance), `facility_id` / `location_room_id` / `recurrence_pattern` / `days_of_week` / `start_time_minutes` / `duration_minutes` / `effective_from_us` / `effective_to_us` / `capacity` override (all moved to slots), `predecessor_class_schedule_id` (moved to slots), `is_series` / `series_*` (live on the Phase 7 `class_series_instances` augmentation table).

Validation at save:

- Same `class_instance_id` + same `priority` + overlapping `[valid_from_us, valid_to_us)` ranges → reject (`OVERLAPPING_SAME_PRIORITY`).
- Impl window must lie within the parent instance's window.
- `valid_to_us <= valid_from_us` → reject.
- Empty `class_schedule_slots` is *allowed* — the "closure / studio dark this week" pattern (high-priority impl with no slots = no sessions for those days within the parent instance).

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

## 1.3 Active-instance + active-implementation resolution

Resolution is now two steps under the three-level hierarchy:

**Step 1 — Active instance for a class at moment `t`:**

```sql
SELECT *
FROM class_instances
WHERE class_id = $1
  AND is_active = TRUE
  AND valid_from_us <= $2
  AND (valid_to_us IS NULL OR $2 < valid_to_us)
LIMIT 1;
```

(Instances don't overlap for the same class — at most one is active. If none, the class has no current run.)

**Step 2 — Active implementation under that instance at moment `t`:**

```sql
SELECT *
FROM class_schedules
WHERE class_instance_id = $1
  AND is_active = TRUE
  AND valid_from_us <= $2
  AND (valid_to_us IS NULL OR $2 < valid_to_us)
ORDER BY priority DESC, valid_from_us DESC
LIMIT 1;
```

Helper method shape:

```cpp
std::optional<int64_t> GetActiveInstanceId(Transaction&, int64_t classId, int64_t atUs);
std::optional<int64_t> GetActiveImplementationId(Transaction&, int64_t classInstanceId, int64_t atUs);

// Convenience that does both lookups, since most callers want the slot list for a (class, date):
std::vector<KeyValueTable> GetActiveSlotsForClassOnDay(
    Transaction&, int64_t classId, int64_t dateUs);
```

Validation invariants (see §1.2 for the full list at each layer):

- Instances don't overlap per class; impls under one instance DO overlap and priority resolves.
- Impl windows must lie within their parent instance's window.
- Empty slot set on an impl = closure / dark days for that span.

## 1.4 Lazy session instantiation (replaces materialization)

Mason's note on §123: "why do we want materialization? … we shouldn't need to 'materialize' anything." Agreed — the materialization model is a holdover from the flat-schedule design. With implementations + slots, the schedule IS the source of truth and persisted session rows are *deltas*, not the schedule itself.

**New model — derived sessions with lazy persistence:**

- A **derived session instance** is identified by the tuple (`class_schedule_slot_id`, `occurrence_date`). Walking days in `[from, to)`, the calendar/catalog computes derived instances by (1) looking up the active `class_instances` for the class on that day, (2) looking up the active `class_schedules` impl under that instance, then (3) expanding the impl's slots matching `EXTRACT(DOW FROM date)`.
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

### Recommendation: three-level hierarchy — `classes` → `class_instances` → `class_schedules`

LOCKED per discussion on Mason's §214 note. The intermediate `class_instances` layer captures "a run / a unit of purchase", and impls live UNDER an instance instead of directly under a class. This:

- Naturally scopes holiday overrides to a specific run (the Memorial Day empty-Monday impl belongs to the May 2026 series instance, not to the class as a whole, and doesn't risk bleeding into the October run).
- Makes pricing per-instance (different runs can have different products) while letting pure price changes flow through `price_schedules` without instance churn.
- Lets product *migrations* (permission rule changes, cancellation policy changes, kind shifts) happen cleanly: close the old instance, open a new one with the new product. Marketing identity on `classes` stays put.
- For recurring membership classes the instance layer collapses to a permanent 1:1 row (`valid_to_us = NULL`). Slight ceremony but consistent.

The three shapes:

- **Recurring membership class** = `classes` row "Knotty Yoga" (`kind='recurring'`) + ONE perpetual `class_instances` row (`valid_to_us = NULL`, product is the membership-included `kind='class'` product) + multiple `class_schedules` impls over time (default + holiday overrides via priority) + slots like Mon/Wed 6pm under the default impl.
- **Workshop** = `classes` row "Inversion Workshop" (`kind='workshop'`) + one `class_instances` row per run ("Aug 15 2026", "Mar 22 2027"), each with its own product (typically `kind='workshop'`) + one bounded `class_schedules` impl per instance (single slot, narrow window). Repeated runs are additional instance + impl pairs under the same `classes` row.
- **Series** = `classes` row "Intro to Partner Acro" (`kind='series'`) + one `class_instances` row per run ("May 2026", "October 2026"), each with its own `kind='class_series'` product + a base `class_schedules` impl per instance (the default schedule for that run, e.g. Mon/Wed 6–7pm for the run's window) + zero-or-more higher-priority impls under the same instance for holiday overrides (Memorial Day Monday empty) + one `class_series_instances` augmentation row per `class_instances` (Phase 7) carrying `min_attendees` / `min_by_us` / `prorated_signups_allowed`.

### Why a single `classes` row across runs

Same reasoning as the prior recommendation, walked back at the `classes` level only (per-run identity now lives on `class_instances.name`, not on a separate `classes` row):

- Lets admin write "Intro to Partner Acro" copy once and have it apply to every upcoming instance.
- Lets the catalog naturally surface a list of upcoming instances under one page.
- Keeps the recurring vs workshop vs series distinction explicit on `classes.kind`.

Per-instance distinctiveness ("Fall 2026 with Guest Mary") lives on `class_instances.name` and is rendered as a sub-title under the parent class's marketing copy. No need to fork the `classes` row.

### How the catalog renders each kind

- `kind='recurring'`: class detail page renders upcoming sessions derived from the perpetual instance's active impl. No list of "runs" — there's only ever the one.
- `kind='workshop'` or `kind='series'`: class detail page renders the marketing copy + a list of **upcoming `class_instances`** (the runs). Each instance links to its own panel / detail showing its slot schedule + per-tier price (from its product).

### Where series-instance bundle fields live (Phase 7)

A new `class_series_instances` table (Phase 7) augments `class_instances` 1:1 for series-specific fields that don't fit on the foundational instance row:

| Column | Purpose |
|--------|---------|
| `id` | PK |
| `class_instance_id` | 1:1 with the parent instance. |
| `min_attendees` | Min headcount for the run to proceed. |
| `min_by_us` | Cutoff for the min check. |
| `min_not_met_policy` | `auto_cancel_refund` | `proceed` | `admin_decides`. |
| `prorated_signups_allowed` | Boolean. |
| `created_us` / `updated_us` | |

Recommended over merging these fields onto `class_instances` (which Phase 1 owns) because:

- Phase 1 stays minimal — series-specific stuff lives in Phase 7 where the series purchase machinery is built.
- Recurring + workshop instances don't carry always-NULL series fields.
- Mirrors the existing `event_sessions` / `bookable_service_sessions` split pattern (foundational row + specialized augment).

Pricing of a series purchase = (count of derived session occurrences in the instance's window from the instance's active impls) × (per-tier base from `product_prices` for the instance's product). Mason's Labor Day case naturally falls out: a high-priority empty-Monday impl under the May 2026 instance drops the occurrence count by one → price drops by one base.

### What this changes in Phase 1

Phase 1 owns scheduling + the instance layer. The series-bundle augmentation (`class_series_instances`) and the series-purchase flow are Phase 7.

Concretely for Phase 1:

- **Add `classes.kind` enum** (`recurring` | `workshop` | `series`).
- **Add `class_instances` table** (the new middle layer with `product_id` per instance).
- **Restructure `class_schedules`** to point at `class_instances` rather than `classes`. Drop `product_id`, `class_id`, `is_series`, and all `series_*` columns.
- **Admin UI** must support: per-class instance list + create instance + per-instance impl list + create impl + slot editor. The three-level nav is intentional.
- **The intro workshop (M-12)** is a workshop in this framing: `classes.kind='workshop'` + one `class_instances` row per offering (typically run a few times per year), with non-member-allowed product permissions on each instance's product.

See the consolidated Open Questions section for what's still open vs locked.

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

Per Mason's notes, with the three-level hierarchy:

- Drop raw IDs. Class / Facility / Room / Product / Instructor → autocomplete dropdowns showing friendly names with the ID hidden.
- **Class detail (admin)**: the page for a single `classes` row shows its kind (recurring / workshop / series), marketing metadata, and its **list of `class_instances`**. For recurring, this is the perpetual instance + any closed predecessor instances from past product migrations. For workshops + series, this is the list of runs (upcoming + past).
- **Instance detail (admin)**: the page for a single `class_instances` row shows the instance's name, window, product (with link), and its **list of `class_schedules` impls** under that instance. "Add new impl" creates a new bounded or open-ended impl scoped to this instance.
- **Implementation detail (admin)**: the page for a single impl shows its priority + window + the **slot editor**. Slot row UI: day-of-week dropdown (Sun..Sat), time picker for start (NOT separated hour/minute inputs), duration input defaulting to 60min, instructor autocomplete (nullable / "TBD" allowed), optional predecessor-slot picker (slot autocomplete scoped to other slots on the same `day_of_week` within the same implementation — that's how same-day chaining is expressed). Sorted list with add / remove buttons.
- **Calendar-day preview**: pick a date → for each class, show which instance is active that day, which impl is active under that instance, and the resolved slot list (priority-resolved within the instance). This is the "easy to see what the active class schedule will be on a given day" view from Mason's note.
- **Impl-save sweep notice**: when saving an impl that orphans future-date persisted rows on the affected instance, the save action surfaces a confirmation ("X future admin notes / subs will be removed by this change") before proceeding. The sweep is scoped to the impl's parent instance — impls under one instance never sweep rows under another. No standalone orphan-recovery UI per §1.4 / Mason's §175 note.

This is a meaningful UI surface area — likely a multi-component redesign of the Phase 1 admin page.

**Skill-level requirement entry** (responding to Mason's note on §173): per the §1.2 recommendation, skill requirements stay per-class (Phase 3 design unchanged). The skill-requirement multiselect therefore lives on the *class* edit form, not the slot. The "requires previous slot" / predecessor entry is on the slot form, as covered above. See OQ-CSI-11 if Mason wants to revisit per-slot skill requirements.

---

# 2. Open Questions for Mason

## 2.0 Locked decisions (already resolved in conversation; recorded here so Section 3 can cite them)

- **L-1 Three-level hierarchy.** `classes` → `class_instances` → `class_schedules` → `class_schedule_slots`. Per Mason's §214 note + Q1–Q5 follow-up.
- **L-2 Middle layer name.** `class_instances` (per Mason's Q1).
- **L-3 `product_id` location.** On `class_instances`, not on `classes` or `class_schedules`. Per Mason's Q2 + Q5. Pure pricing changes flow through `price_schedules` automatically; product migrations (permission / cancellation policy / kind shifts) happen by close-old-instance + open-new-instance. Product variants apply for parallel shapes (single / couple / family seats).
- **L-4 Instance layer always present.** Per Mason's Q3. Recurring membership classes have a perpetual 1:1 instance (`valid_to_us = NULL`); no asymmetric skip.
- **L-5 Series-bundle augmentation table.** Phase 7 ships `class_series_instances` as a 1:1 augmentation of `class_instances` carrying `min_attendees` / `min_by_us` / `min_not_met_policy` / `prorated_signups_allowed`. Per Mason's Q4. (Supersedes the prior `class_series_offerings` naming in earlier drafts.)
- **L-6 `is_series` + `series_*` columns dropped from `class_schedules` in Phase 1.** Lives in the Phase 7 augmentation table instead.
- **L-7 Shared `classes` row across runs.** Workshops + series share one marketing-identity `classes` row; per-run distinction lives on `class_instances.name`. (Walks back the much-earlier "separate `classes` row per run" recommendation.)
- **L-8 Per-class exceptions collapse to high-priority impls.** Per Mason's §1.7 reframe. Per-class scheduling exceptions are no longer a separate concept; the impl + priority model covers them.
- **L-9 No global "studio closure" lever.** Per Mason's §205 note. Closures are per-class empty high-priority impls; workshops on the closed date are unaffected (they're under their own class). Phase 10's batch UI is a class-multiselect.

## 2.1 Still-open questions

Tag each with a decision before Section 3 (doc updates) kicks off.

- [ ] **OQ-CSI-1 (Naming)**: Use `class_schedules` (impls) + `class_schedule_slots` + `class_instances` (recommendation)? Or rename the impl + slot pair to `class_timetables` / `class_timetable_slots` to avoid the slight noun overload now that "instance" is the headline middle layer? Recommendation: **keep `class_schedules` / `class_schedule_slots`** — the parallel to `price_schedules` is still the cleanest external naming, and "instance" reads as the unit-of-purchase layer, not the schedule layer.
	- Mason- I'm confused. Can you list the two options for the three different entities?
- [ ] **OQ-CSI-2 (Product location)** — **RESOLVED in L-3.** Product is on `class_instances`. Listed here for backward reference.
- [ ] **OQ-CSI-3 (Where does `facility_id` live)**: On the slot (recommendation — lets a class run at different facilities within one implementation) or on the implementation (simpler — one facility per implementation)?
	- Mason- I'll go with your recommendation.
- [ ] **OQ-CSI-4 (Closure batch UX)**: Closures are per-class empty high-priority impls; there is no "studio is closed" global lever (workshops can still run, per L-9). Pure UX question:
  - (a) Ship a "Close these N classes for this date range" multiselect admin action in Phase 1 (creates an empty impl under each selected class's active instance).
  - (b) Defer the convenience UI to Phase 10; in Phase 1 admin manually creates per-class empty impls.
  - Recommendation: **(b)** — Phase 1 is already big; ship the batch action with the other scheduling-exception work in Phase 10.
  - Mason- I'll go with your recommendation.
- [ ] **OQ-CSI-5 (Materialization UX)** — *superseded by OQ-CSI-12*. Lazy-instantiation in §1.4 removes materialization entirely. Stub.
- [ ] **OQ-CSI-6 (Drop biweekly + custom)**: Confirm we're removing `recurrence_pattern` entirely?
	- Mason- Yes.
- [ ] **OQ-CSI-7 (Time entry UX)**: Existing memory `feedback_date_time_pickers.md` says "times must use hour pickers" and Phase 1 §6.3 ships separated hour + minute inputs. Slot start times often have minute precision (5:45 PM yoga) — do we need a full time picker (HH:MM) for slot entry, overriding the hour-only convention here? Recommendation: **yes, full time picker** because class start times in the wild are not hour-aligned (e.g. 5:45, 6:15).
	- Mason- yes this things don't need to start on hour boundaries.
- [ ] **OQ-CSI-8 (Slot uniqueness)**: Allow duplicate (`class_schedule_id`, `day_of_week`, `start_time_minutes`, `location_room_id`) tuples or reject? Recommendation: **reject identical full tuples** because that's almost certainly a data-entry mistake — different rooms at the same time are different rows, and different start times are different rows. The same room + same start_time + same day = duplicate.
- [ ] **OQ-CSI-9 (Drop `is_series` / `series_*` from `class_schedules`)** — **RESOLVED in L-5 / L-6.** Drops happen in Phase 1; Phase 7 owns the `class_series_instances` augmentation table. Listed here for backward reference.
- [ ] **OQ-CSI-10 (Phase 1 already merged?)**: Phase 1 is marked done end-to-end (most checkboxes are checked). Is this a "redesign before Phase 2 lands" plan (rewrite migrations, re-do tests) or a "Phase 1.5 migration" plan (new tables alongside, deprecation)? Recommendation: **rewrite Phase 1 in place** — pre-deploy, no production state to defend against per `feedback_no_premature_defensive_code.md`. But Mason should confirm there's no deployed environment that needs a migration path.
- [ ] **OQ-CSI-11 (Skill-level requirements per-class or per-slot)**: Today's Phase 3 design keys skill prerequisites off `class_id` via `class_skill_requirements`. With per-slot skill requirements, "Beginner Acro" Saturday morning and "Advanced Acro" Tuesday night could share a class row but have different prerequisites. Recommendation: **per-class (no change to Phase 3)** because (a) prerequisites are a property of "what the class is", and (b) genuinely different skill levels of the same activity should be different classes. Confirm.
- [ ] **OQ-CSI-12 (Lazy instantiation vs materialization)**: Adopt the lazy-instantiation model described in §1.4 (recommendation) or keep an explicit pre-materialization step? Recommendation: **lazy** — eliminates a job + admin button, makes impl changes correctly take effect with no cleanup cascade, mirrors how `price_schedules` work today. Costs covered in §1.4. Confirm.
- [ ] **OQ-CSI-13 (Table strategy under lazy model)**: Per §1.4 "Table-naming wrinkle": (A) keep `event_sessions` for everything and just stop pre-populating class rows (recommendation, minimum churn); (B) split out a separate `class_session_instances` table for lazy class rows; (C) rename `event_sessions` → `session_instances` and adopt lazy as the default for services / events too. Recommendation: **A**.
- [ ] **OQ-CSI-14 (Instructor scheduling at slot vs per-session)**: Adding `instructor_person_id` to the slot row means "this is the regularly-scheduled instructor for this time slot". Per-session substitutions still ride on `event_session_staffing` (the sub action creates the `event_sessions` row at the same time under the lazy model). Confirm this two-tier model (slot = default, persisted session = override) vs always sourcing instructor from `event_session_staffing` even for the default.
- [ ] **OQ-CSI-15 (Shared `classes` row across runs)** — **RESOLVED in L-7.** One `classes` row per offering identity; per-run distinction on `class_instances.name`. Listed here for backward reference.
- [ ] **OQ-CSI-16 (Series bundle table)** — **RESOLVED in L-5.** Phase 7 ships `class_series_instances` as a 1:1 augmentation of `class_instances`. Listed here for backward reference.
- [ ] **OQ-CSI-17 (Closure scope)** — **RESOLVED in L-9.** No global closure lever; per-class only. Listed here for backward reference.
- [ ] **OQ-CSI-18 (Add `classes.kind` enum)**: Add a `classes.kind` enum (`recurring` | `workshop` | `series`) defaulting to `recurring` so catalog / admin UI can discriminate rendering paths. Confirm.
- [ ] **OQ-CSI-19 (Impl-save sweep semantics)**: Per §1.4, the impl-save flow auto-deletes orphaned future-date `event_sessions` rows scoped to the impl's parent instance that hold only admin actions (notes, manual instructor subs). Any row with a `purchase_id` blocks the save with a "cancel-and-refund first" message. Confirm this sweep-on-save model rather than a standalone orphan-recovery view.
- [ ] **OQ-CSI-20 (Recurring-class instance migration UX)** — *new, follow-up from Mason's Q5.* When admin needs to migrate a recurring class to a new product (membership permission rule change, etc.) — what's the UX? Options:
  - (a) "Edit class" form has a "migrate to new product effective DATE" action that closes the current perpetual instance with `valid_to_us=DATE` and opens a new one with `valid_from_us=DATE` and the new `product_id`. Admin picks the new product from a dropdown. Slots default to copying from the closing instance's latest impl (admin can edit).
  - (b) Admin manually closes the old instance and creates the new one through generic CRUD. No special "migration" affordance.
  - Recommendation: **(a)** because the migration is a real concept worth surfacing — admin shouldn't have to remember to copy slots forward.
- [ ] **OQ-CSI-21 (Slot copy-forward on impl create)** — *new, follow-up.* When admin creates a new impl under an existing instance (e.g. a holiday override), should the impl start empty, or pre-populated with the slots of the previous active impl (so admin only edits the differences)? Recommendation: **start with a "copy from existing impl" picker** — admin chooses to copy from the default impl and then edits, or to start empty (for a closure). Speeds up the common case (Memorial Day override = copy default + delete Monday slot).

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
- [ ] §2 Database Schema — full rewrite under the three-level hierarchy (L-1):
  - §2.1 `classes` table — **add a `kind` enum (`recurring` | `workshop` | `series`) defaulting to `recurring`** per L-7 / OQ-CSI-18. Workshops + series share their `classes` row across runs.
  - §2.2 (new) `class_instances` table — the middle layer per L-1 / L-2 / L-4. Columns: `id`, `class_id`, `name`, `valid_from_us`, `valid_to_us` (nullable for perpetual), `product_id` (per L-3 — product lives here, not on impls or classes), `is_active`, `created_us`, `updated_us`. Validation: at most one perpetual `valid_to_us=NULL` instance per recurring class; no overlapping active instances per class.
  - §2.3 `class_schedules` — repurposed as impls **under instances**. Columns: `id`, `class_instance_id` (FK; was previously planned as `class_id`), `name`, `priority`, `valid_from_us`, `valid_to_us`, `is_active`, `created_us`, `updated_us`. Drop `product_id` (now on instance), `class_id` (reached via instance), `predecessor_class_schedule_id` (moved to slot), and `is_series` + all `series_*` columns (per L-5 / L-6 — series bundle machinery is in Phase 7's `class_series_instances`).
  - §2.4 (new) `class_schedule_slots` table including `instructor_person_id` and `predecessor_class_schedule_slot_id`.
  - §2.5 `event_sessions` extensions — change to `class_schedule_slot_id` (slot identity) + `occurrence_date_us` (date-truncated for the day this row pins). Keep `class_id` as a denormalized convenience. The composite (`class_schedule_slot_id`, `occurrence_date_us`) is the natural key for derived-vs-persisted lookups under §1.4's lazy model. (Optional second denormalized convenience: `class_instance_id` for instance-scoped queries — TBD if needed.)
  - §2.6 wire into init pipeline — add the new tables to `make_database_info.cpp` + `CreateTables()` + `db_schema/CMakeLists.txt`. Ordering: `classes` → `class_instances` → `class_schedules` → `class_schedule_slots` → `event_sessions` (FKs resolve cleanly).
- [ ] §3 Table Helpers — rewrite for the three-level hierarchy:
  - `TableHelpers::ClassInstances` (new) — full CRUD; `GetActiveInstance(classId, atUs)`, `GetInstancesByClass(classId)` (active + closed), `GetUpcomingInstances(classId, asOfUs)` (for workshop / series catalog list).
  - `TableHelpers::ClassSchedules` — pivots to instance-scoped queries. `GetActiveImplementation(classInstanceId, atUs)`, `GetImplementationsByInstance(classInstanceId)`, `GetImplementationsOverlapping(classInstanceId, fromUs, toUs)`. Drop class-id-keyed queries from earlier drafts.
  - `TableHelpers::ClassScheduleSlots` (new) — full CRUD; `GetSlotsByImplementation(scheduleId)`, `GetSlotsByImplementationAndDay(scheduleId, dayOfWeek)`, `GetActiveSlotsForClassOnDay(classId, dateUs)` (does the two-step instance + impl resolution internally), `GetSlotsPotentiallyConflictingInRoom(roomId, dayOfWeek, startTimeMinutes, durationMinutes)`.
  - `TableHelpers::EventSessions` — add `LookupBySlotAndDate(slotId, occurrenceDateUs)` for the lazy lookup path; add `GetOrphanedFutureSessionsForInstance(classInstanceId, asOfUs)` for the sweep-on-impl-save logic (scoped to instance per the §1.8 sweep rule).
  - Tests for all four with the new sort orders + conflict-detection semantics + instance-scoped resolution.
- [ ] §4 Business Logic — split into `ClassInstanceHelper` + `ClassScheduleHelper`:
  - `ClassInstanceHelper` (new) — `CreateInstance(req)` / `UpdateInstance(...)` / `CloseInstance(instanceId, validToUs)` / `MigrateRecurringClassToNewProduct(classId, newProductId, effectiveAtUs)` — the L-3 / OQ-CSI-20 migration shortcut that closes the perpetual instance and opens a new one with the new product, optionally copying slots forward.
  - `ClassScheduleHelper` — `CreateImplementation(instanceId, req)` / `UpdateImplementation(...)` (validates that impl window lies within the parent instance's window + same-priority-no-overlap rule scoped to the parent instance). `UpdateImplementation` triggers the impl-save sweep (delete future-date orphaned `event_sessions` rows under the affected instance; refuse the save if any orphan carries a `purchase_id`). Optional `CreateImplementationFromExisting(...)` to support the OQ-CSI-21 "copy slots forward" shortcut.
  - `AddSlot(scheduleId, slot)` / `UpdateSlot(slotId, ...)` / `DeleteSlot(slotId)` (slot CRUD). Slot deletion also triggers the sweep.
  - **No `MaterializeFutureSessions`** under the lazy model. Replaced by `EnsureSessionExists(slotId, occurrenceDateUs)` — idempotent, called by the booking / check-in / cancel / sub paths in §1.4's trigger list.
  - `GetDerivedSessionsForRange(classId, fromUs, toUs)` — walks dates, resolves active instance per day, resolves active impl under that instance, expands slots, left-joins persisted `event_sessions` rows. The single helper calendar / catalog queries call into.
  - `GetActiveScheduleView(classId, dateUs)` — backs the "preview on date X" UI (shows both the active instance and its active impl with resolved slots).
  - `SweepOrphanedFutureSessions(classInstanceId, asOfUs)` — internal helper called by `UpdateImplementation` / `AddSlot` / `DeleteSlot`. Returns (deletedCount, blockedRows) so the endpoint can surface the count or the refusal. Scoped to instance per §1.8.
  - Drop `recurrence_pattern` validation entirely.
  - Test updates: all of `class_schedule_helper_test.cpp` needs re-casting; new `class_instance_helper_test.cpp`. Tests for priority-resolution scoped to instance, overlap rejection within an instance, impl window must lie within instance window, no-slot (closure) impls, multi-slot per day, per-day different times, lazy ensure-session idempotency, sweep-on-save deletes admin actions, sweep-on-save refuses when purchase row exists, derived-vs-persisted left-join, recurring-class product migration close+open, OQ-CSI-21 slot copy-forward.
- [ ] §5 Endpoints — re-cast for the three-level hierarchy:
  - `POST /api/admin/class_instance` (instance create — body has `class_id`, `name`, window, `product_id`).
  - `PUT /api/admin/class_instance/<id>` / `DELETE /api/admin/class_instance/<id>` (close/soft-delete).
  - `POST /api/admin/class/<classId>/migrate_product` — convenience for OQ-CSI-20 (closes current perpetual instance, opens new one with the new product; body picks the effective date + new product + slot copy-forward flag).
  - `GET /api/admin/class_instances?class_id=<id>` — list instances under a class.
  - `POST /api/admin/class_instance/<instanceId>/schedule` (impl create under an instance).
  - `PUT /api/admin/class_schedule/<id>` / `DELETE /api/admin/class_schedule/<id>` (impl update / soft-delete; updates trigger the sweep and return `{deletedOrphanCount, blockedRows}`).
  - `POST /api/admin/class_schedule/<id>/slot` / `PUT /api/admin/class_schedule_slot/<slotId>` / `DELETE /api/admin/class_schedule_slot/<slotId>`.
  - `GET /api/admin/class_schedules?class_instance_id=<id>` — list impls under an instance.
  - `GET /api/admin/class_schedule_preview?class_id=<id>&date_us=<t>` — resolved active instance + active impl + slot list.
  - **Remove** `POST /api/admin/class_schedule/<id>/materialize` — no longer applicable under lazy instantiation.
  - **No standalone orphan-listing endpoint** — sweep is part of the impl-save / slot-mutation responses.
- [ ] §6 Frontend — significant rewrite of the admin UI for the three-level hierarchy:
  - **Class detail (admin)**: page per `classes` row showing kind (recurring / workshop / series) + marketing metadata + instance list.
  - **Instance detail (admin)**: page per `class_instances` row showing name + window + product + impl list. "Add new impl" creates a new impl under this instance.
  - **Implementation detail (admin)**: page per impl showing priority + window + slot editor.
  - **Slot editor**: sorted list, day-of-week dropdown, time picker (per OQ-CSI-7), duration input, facility / room / instructor autocomplete dropdowns, optional predecessor-slot picker.
  - **Migration action** on a recurring `classes` row: "Migrate to new product effective DATE" form (OQ-CSI-20) — closes the perpetual instance, opens a new one with the chosen product, optionally copies forward slots from the closing instance's latest impl.
  - **Copy-from picker** on impl create (OQ-CSI-21): "start empty" or "copy slots from <impl name>".
  - **"Schedule on date X" preview view** — shows active instance + active impl + resolved slots for the date.
  - **Impl-save confirmation modal**: when the save will trigger a non-zero orphan sweep, show "X future admin notes/subs will be removed by this change" with a confirm button. If the sweep is blocked by `purchase_id` rows, surface those with "cancel and refund first" actions.
  - Remove the materialize dialog component entirely.
  - All component specs updated.
- [ ] §7 Admin Metadata — add `class_instances` AND `class_schedule_slots` registration (all eleven steps each). `class_instances` is nested under `classes`; `class_schedules` becomes nested under `class_instances` (was previously nested under `classes`); `class_schedule_slots` is nested under `class_schedules`.
- [ ] §10 Tests Summary — expand to call out the new instance tests + slot tests + priority tests + lazy-instantiation tests + sweep-on-save tests + recurring-class-migration tests + slot-copy-forward tests; remove materialize tests.
- [ ] §11 Cross-Layer Acceptance Criteria — rewrite around three flows:
  1. Admin creates a `kind='recurring'` class with its perpetual instance (auto-created or wizard-driven) + a default impl with three slots, then adds a holiday-week empty-impl override under the same instance; calendar shows holiday-week behavior during the window and reverts after, with no admin click-to-materialize.
  2. Admin creates a `kind='workshop'` class, then creates two instances under it (Aug 15 2026 + Mar 22 2027) each with its own bounded impl + product; catalog detail page shows the marketing copy + both upcoming runs.
  3. Admin migrates a recurring class to a new product effective Sept 1 2026 via the migration action; the perpetual instance closes Sept 1 and a new one opens with the new product; the new instance auto-inherits the prior instance's latest impl's slots.
- [ ] §12 Resolved Questions — append the L-1..L-9 locked decisions and OQ-CSI-1..21 entries.
- [ ] Mason-note in §357 — replaced by the rewritten body; can become a "Design Pivot Notes" appendix linking to this doc as the design source.

## 3.3 Sibling phase doc updates

- [ ] **Phase 2 (Membership-Gated Drop-In)** — bigger impact than originally thought under the lazy model. `EventSessionHelper::GetVisibleEventSessions` now needs to return *derived* sessions (no row exists yet) alongside persisted ones. The booking flow under §1.4's trigger #4 must call `EnsureSessionExists` to materialize the row at booking time. Per-tier price resolution still flows through the implementation's `product_id`. Expected impact: moderate — Phase 2 doc needs a "lazy ensure on booking" subsection.
- [ ] **Phase 4 (iCal Generator Extensions)** — no model touches. No update needed.
- [ ] **Phase 5 (Attendance Templates)** — bigger impact. Per Mason's §140 note, membership-included template entries don't create any persisted rows at all (no booking, no `event_sessions`). The template entry becomes pure intent. Bind template entries to (`person_id`, `class_schedule_slot_id`) — if the slot is overridden by a higher-priority impl on a given week, the user simply gets nothing for that week (matches their experience: the class isn't running this week). Per-instance exceptions (AT-5 / AT-6) likewise stay as pure data without backing bookings. Phase 5's entire "auto-create bookings on materialization" subsection disappears. Under the three-level hierarchy: template entries are keyed by slot, and a slot deletion (via impl edit or instance migration) leaves a template entry "stale" — surfaced in the user's portal as "this slot no longer exists, pick a new one?".
- [ ] **Phase 6 (Weekly Digest)** — bigger impact. The digest's "this week's classes" lookup must derive from impl + slots PLUS persisted rows, not just `SELECT FROM bookings`. For membership-included template attendance there are no booking rows; the digest must compute "today the user has a Monday 6pm Knotty Yoga via their template against the active impl's Monday 6pm slot". Add a `WeeklyDigestHelper::GetTemplateOccurrencesForWeek(personId, weekStartUs)` that walks the user's template entries against the active impls.
- [ ] **Phase 7 (Class Series and Workshops)** — bigger impact, fundamental reframe per §1.5a. The Phase 7 doc currently treats series as "`class_schedules.is_series=true`"; this is dropped. Instead:
  - Each `classes` row of `kind='series'` shares marketing identity across all its runs. Each run is a `class_instances` row under that class (multiple per series — May 2026, October 2026, etc.).
  - The bundled product (`kind='class_series'`, per-tier per-session base price) lives on `class_instances.product_id`, not on any series-specific table.
  - A new `class_series_instances` table (L-5) augments `class_instances` 1:1 for series-only fields: `min_attendees`, `min_by_us`, `min_not_met_policy`, `prorated_signups_allowed`.
  - Each instance has one base `class_schedules` impl describing the run's default Mon/Wed schedule, plus zero-or-more higher-priority impls for holiday overrides (Mason's Labor Day Monday case) scoped to THAT instance only.
  - Price at purchase = (count of derived session occurrences in the instance's window from the instance's active impls) × (per-tier base from the instance's product). Empty high-priority impls under the instance naturally reduce the count.
  - Series purchase = ensure `event_sessions` rows for every derived occurrence at purchase time (paid bookings can't be lazy — they're real money). Sessions get `series_purchase_id` set so the min-attendees auto-cancel job can find them.
  - Workshops follow the same shape — `classes` row of `kind='workshop'` + one `class_instances` per run + `kind='workshop'` product per instance + one bounded impl per instance — minus the `class_series_instances` augmentation.
  - The M-12 intro workshop is treated identically to any other workshop, with non-member-allowed product permissions on each instance's product.
- [ ] **Phase 8 (Staff Check-in)** — small impact. Check-in is trigger #6 from §1.4 — staff check-in calls `EnsureSessionExists` if the row doesn't exist. The "people who attended this class in the last 4 weeks" lookup joins through `event_sessions.class_id` as before (denormalized column unchanged). Add a paragraph noting the ensure-on-checkin step.
- [ ] **Phase 9 (Attendance History)** — small impact. History reads from persisted rows only (no derived view needed — attendance only exists for sessions that were checked in, which means they were ensured). No model touch beyond keying off `class_schedule_slot_id` instead of `class_schedule_id` if any join uses that column.
- [ ] **Phase 10 (Scheduling Exceptions and Shift Trades)** — major rewrite. Per-class exceptions collapse to implementations. Per Mason's §205 note + OQ-CSI-17, the closure batch UI is "close these N recurring classes for this window" (class-multiselect), NOT a global "studio is closed" lever — workshops scheduled during the window are unaffected. Instructor substitution becomes trigger #2 from §1.4 — sub action calls `EnsureSessionExists` then writes `event_session_staffing`. Shift trades likewise. Document the ensure-on-action pattern as the Phase 10 invariant.
- [ ] **Phase 11 (Signup Windows and Reminders)** — small impact. "Sessions available to book on date X" now derives from the active impl + slots. The reminder system queries derived sessions to figure out when a user can book.
- [ ] **Phase 12 (Specialty Instructor Cost)** — costs are typically per-instance (a specialty instructor is hired for a specific run / workshop / series). Update §12.1 schema to key off `class_instance_id` rather than the original "schedule" framing. Optionally also support per-slot cost overrides if the same instance has the specialty teacher only on certain days.
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
