---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/13/2026
Version: 0.3
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

The Bookable Service Foundation.md file currently has the scheduling algorithm detailed but it has some issues. The core thing is that we need to think of a model based on maximizing the number of complete scheduling units that can be fit into an open block of time. Here are the constraints:

- The core unit of scheduling is an hour
- We allow booking 60min, 90min, and 120min blocks which are 1x, 1.5x, and 2x
- There is a mandatory buffer window between blocks
- The idea is that if one of these units is cancelled, we should allow it to be rebooked flexibly
- 60min and 120min are highly compatible if we require a double buffer between 120min massages. By requiring a double buffer, if a 120min massage gets cancelled, that slot can be filled with either another 120min massage OR two 60min massages with a buffer after each in the same amount of time as a 120min massage with a double buffer.
- 90min massage is complicated
	- For scheduling a 90min massage, there are two conditions:
		- Previous massage was the first massage or the block or not a 90min massage
			- Require a double buffer after this massage
		- Previous massage was a 90min massage
			- Previous 90min massage had a double buffer after it
				- Require just a single buffer after this massage
			- Previous 90min massage did not have a double buffer after it
				- Require a double buffer after this massage
	- This will cause concurrent pairs of 90min massages to occupy three hours with three buffers
		- If one of the pair members is cancelled, it can only be replaced with another 90min massage, a 60min massage is not allowed since that would cause a 30min gap
		- If both are cancelled, they can be replaced with either three 60min massages OR a 120min and 60min massage (in either order)
	- The complicated case is if we have three 90min massages A, B, C booked in a row
		- This would look like A, double buffer, B, single buffer, C, double buffer
		- If B is cancelled, it can only be replaced with either a 90min massage
		- If B is cancelled and either A or B as well, either block can be replaced with:
			- Two 90min massages
			- Three 60min massages
			- A 60min and 120min massage in either order
		- The key is that that alternating double buffers for 90min massages makes them compatible with 60min and 120min massages
	- A 90min massage next to another 90min massage is compatible with 60min and 90min massage
	- A 90min massage sandwiched between 60min or 120min massages is essentially an island and can only be replaced with another 90min massage if it is cancelled
	- A 120min massage that is cancelled with no adjacent free slots can never be replaced by a 90min massage as that would create a 30min hole
	- A 90min massage can only be booked into an available window if there is either exactly 90min plus single or double buffer depending on previous massage state OR the window is at least 180min plus 3xbuffer long
	- Please note that buffer requirements after a booking don't count if this is the last booking in an availability block (meaning that there isn't 60min left after the last booking in this block)

The other thing I would like to alter is to tweak requirements at the end of an availability window. In general, if we have a window that is less than 120min but can fit a 90min massage, we require the 90min massage and don't allow the 60min massage. For the last block of time in an availability window if it is less than 120min but greater than 90min, I would like to allow either a 60min or 90min massage as 60min is the most popular and pairs well for availability.

# Scheduling Algorithm

## Core Constants

- **Slot alignment**: 5 minutes (all start times snap to 5-minute boundaries)
- **Base unit**: 60 minutes
- **Buffer**: 10 minutes default (configurable per-provider, minimum 10 minutes)
- **Variants**: 60min (1x), 90min (1.5x), 120min (2x)

## Shift Model

A **shift** is a continuous block of provider availability that becomes a concrete scheduling unit once the first booking is made. The shift captures a snapshot of all settings at booking time so that subsequent changes to facility or provider preferences don't retroactively alter existing bookings.

### Shift Lifecycle
1. **Virtual**: Before any booking exists, the shift is virtual — computed from the provider's availability blocks and current settings. Settings changes take effect immediately.
2. **Materialized**: When the first booking is made in a shift, a shift record is created in the database capturing:
   - Provider person ID, facility ID
   - Shift start/end times
   - Effective buffer minutes
   - Lunch start/end times (if applicable)
   - Setup/teardown buffer minutes
   - Max time hole minutes
3. **Frozen**: Once materialized, the shift's settings don't change even if provider/facility settings are updated. New bookings within this shift use the snapshotted settings.

### Shift Boundary Buffers (Setup/Teardown)
Configurable per-facility with zero defaults:
- **Start-of-shift buffer**: minutes before the first booking can start (default: 0)
- **End-of-shift buffer**: minutes after the last booking must end by (default: 0)

These reduce the effective availability window. Example: 8:00–4:30 shift with 5min setup and 5min teardown → effective booking window is 8:05–4:25.

## Buffer Rules

The buffer model serves two purposes: (1) providing a break between clients for the provider, and (2) maximizing rebookability when a slot is cancelled. With a 10-minute minimum buffer, providers always get at least 10 minutes between clients.

### 60-Minute Bookings
- Always require **1 buffer** (10min) after the booking
- Total occupied: **70min**
- A cancelled 60min slot frees 70 minutes — enough for exactly one more 60min + buffer

### 120-Minute Bookings
- Always require **2 buffers** (20min) after the booking
- Total occupied: **140min**
- A cancelled 120min slot frees 140 minutes — enough for exactly two 60min bookings (60+10+60+10 = 140min)
- This is the key **subdivisibility guarantee**: a 120min block is always perfectly replaceable by two 60min blocks

### 90-Minute Bookings (Context-Dependent Buffers)
The 90-minute variant requires context-dependent buffers because 90 is not an exact multiple of 60. The buffer after a 90min booking depends on the **previous booking** in the sequence:

**Rule: Alternating double/single buffers for consecutive 90-minute bookings**

1. **First 90min in a block, or previous was NOT 90min** → **double buffer** (20min) after this 90min
2. **Previous was 90min with double buffer** → **single buffer** (10min) after this 90min
3. **Previous was 90min with single buffer** → **double buffer** (20min) after this 90min

This creates an alternating pattern for consecutive 90-min bookings:

```
90min, 20min buffer, 90min, 10min buffer, 90min, 20min buffer, ...
  A      (double)      B     (single)      C      (double)
```

**Why this works:**
- A pair of 90min bookings (A + B) occupies: 90 + 20 + 90 + 10 = 210 minutes (3 hours 30 minutes, 3 buffers)
- If both A and B are cancelled, the 210 minutes can hold:
  - Three 60min bookings: 60+10+60+10+60+10 = 210 ✓
  - One 120min + one 60min: 120+20+60+10 = 210 ✓
  - One 60min + one 120min: 60+10+120+20 = 210 ✓
  - Two 90min bookings (same as before): 90+20+90+10 = 210 ✓
- If only one of the pair is cancelled, only a 90min replacement fits (60min would leave a 30min hole)

**Island 90-minute bookings:**
- A 90min booking sandwiched between 60min or 120min bookings is an "island" — if cancelled, it can ONLY be replaced by another 90min booking
- A 120min booking that is cancelled with no adjacent free slots can NEVER be replaced by a 90min booking (would create a 30min hole)

### Buffer at End of Availability Window
- If the remaining time after the last booking is **less than 60 minutes** (the base duration), no buffer is required — the booking is simply the last one in the window
- This applies regardless of variant duration

### Summary Table

| Variant | Buffer Count | Buffer Duration (10min base) | Total Occupied | Condition |
|---------|-------------|------------------------------|----------------|-----------|
| 60min   | 1           | 10min                        | 70min          | Always |
| 90min   | 2           | 20min                        | 110min         | First in block, or previous was non-90min, or previous 90min had single buffer |
| 90min   | 1           | 10min                        | 100min         | Previous was 90min with double buffer |
| 120min  | 2           | 20min                        | 140min         | Always |

## Lunch Break System

Providers working shifts at or above a configurable threshold require an unpaid lunch break. The shift clock time is extended by the lunch duration so the provider still works the full shift hours.

### Configuration

| Setting | Level | Default | Constraints |
|---------|-------|---------|-------------|
| Shift threshold requiring lunch | Facility (from secrets/config) | 6 hours | Must be > 0 |
| Lunch length | Facility (from secrets/config) | 30 min | Minimum 30 min |
| Provider lunch length override | Per-provider | (uses facility default) | Must be >= facility lunch length |

### Cascade: Facility Default → Provider Override
The facility sets default values that apply to all providers. Providers can override **upward** (e.g., take a longer lunch or larger buffer) but cannot go below the facility minimum. Providers are only paid for the facility default — extra time is their personal choice.

### Lunch Placement Algorithm

When a shift meets or exceeds the threshold, the algorithm determines optimal lunch placement **before any bookings are made**. The lunch break creates two independent availability windows (pre-lunch and post-lunch).

1. **Extend shift**: Add lunch duration to the total clock time. An 8hr shift with 30min lunch runs 8:00 AM – 4:30 PM on the clock, with the provider working 8 hours.

2. **Constraint**: Lunch must begin **between the 2nd and 5th hours** of the shift (measured from shift start, in working time).

3. **Calculate split candidates**: Compute 60min slots with buffers from the start of the shift. Each buffer boundary between slots is a candidate lunch start point (within the 2nd–5th hour constraint).

4. **Select optimal placement**: For each candidate, calculate working time before lunch and working time after lunch. Compute the delta of each half from the midpoint of total working time. Pick the candidate with the **smallest maximum delta** — this creates the most balanced split.

5. **Lunch subsumes buffer**: No additional buffer is required before or after the lunch break. The lunch IS the break.

### Worked Examples (10min buffer, 30min lunch)

#### 8-hour shift (480min working, 510min clock)
Midpoint of working time: 240min (4:00)

| Slots Before Lunch | Pre-Lunch Work | Post-Lunch Work | Delta A | Delta B | Max Delta |
|--------------------:|---------------:|----------------:|--------:|--------:|----------:|
| 3 | 3×60 + 2×10 = 200min (3:20) | 280min (4:40) | 40 | 40 | **40** |
| 4 | 4×60 + 3×10 = 270min (4:30) | 210min (3:30) | 30 | 30 | **30** ← Winner |

**Result**: Lunch after the 4th client. Schedule: 4 slots (8:00–12:30), lunch (12:30–1:00), remaining slots (1:00–4:30).

#### 7-hour shift (420min working, 450min clock)
Midpoint: 210min (3:30)

| Slots Before Lunch | Pre-Lunch Work | Post-Lunch Work | Delta A | Delta B | Max Delta |
|--------------------:|---------------:|----------------:|--------:|--------:|----------:|
| 2 | 2×60 + 1×10 = 130min (2:10) | 290min (4:50) | 80 | 80 | **80** |
| 3 | 3×60 + 2×10 = 200min (3:20) | 220min (3:40) | 10 | 10 | **10** ← Winner |
| 4 | 4×60 + 3×10 = 270min (4:30) | 150min (2:30) | 60 | 60 | **60** |

**Result**: Lunch after the 3rd client.

#### 6-hour shift (360min working, 390min clock)
Midpoint: 180min (3:00)

| Slots Before Lunch | Pre-Lunch Work | Post-Lunch Work | Delta A | Delta B | Max Delta |
|--------------------:|---------------:|----------------:|--------:|--------:|----------:|
| 2 | 2×60 + 1×10 = 130min (2:10) | 230min (3:50) | 50 | 50 | **50** |
| 3 | 3×60 + 2×10 = 200min (3:20) | 160min (2:40) | 20 | 20 | **20** ← Winner |
| 4 | 4×60 + 3×10 = 270min (4:30) | 90min (1:30) | 90 | 90 | **90** |

**Result**: Lunch after the 3rd client.

#### 5-hour shift (300min working) — no lunch required
Below the 6-hour threshold. No lunch break inserted.

#### Split shifts — no lunch
If a provider has two separate availability blocks (e.g., 8-12 and 1-5), these are treated as independent shifts. Neither requires lunch if under the threshold individually.

## Walk-In Booking Rules

Walk-in bookings are staff-initiated bookings for customers who arrive without a reservation. They must respect the availability system to prevent schedule holes and provider conflicts.

### Provider Walk-In Settings
- **Accepts walk-ins**: boolean per-provider (default: true)
- **Walk-in minimum booking window**: facility-level setting, default 15 minutes. A walk-in cannot be booked for a slot starting less than this many minutes from now.

### Walk-In Constraints
1. **Respect availability**: Walk-ins must go through the normal availability system. No `skipAvailabilityCheck`. Available slots are shown to the staff member.
2. **No booking immediately after in-session provider**: If a provider is currently with a client, the open slot immediately following that client **cannot** be booked as a walk-in. Rationale: therapists commonly extend sessions when no one is booked after them. If the therapist finishes on time, the front desk can ask them if they want to take the walk-in and use the admin override (see below).
3. **Minimum booking window**: The slot must start at least `walk_in_min_buffer_minutes` from now.
4. **Admin override**: Admin/staff can override the "already started" constraint to book a slot that has technically started (within the buffer period). This handles the case where a therapist finishes on time and is ready for the next client.

## Mid-Session Extension (Upgrades)

It's common for a therapist to extend a session if no client is booked after them and the current client wants more time. This is handled as a session upgrade.

### Upgrade Paths
- 60min → 90min
- 60min → 120min
- 90min → 120min

### Rules
1. **Staff/admin only**: Upgrades are initiated by staff, not customers.
2. **Don't cancel the original booking**: Cancelling would affect bundled pricing. Instead, add an upgrade line item to the purchase.
3. **Charge the delta**: The upgrade line item charges only the price difference between the original and new duration.
4. **Override availability checks**: Upgrades may violate "no schedule hole" rules, which is acceptable because the alternative is dead time for the therapist.
5. **Update the session**: Modify `bookable_service_sessions.end_time_us` and `buffer_end_us` to reflect the new duration.

## Slot Generation Algorithm

### Step 1: Build Free Windows
For each provider, determine free windows by subtracting existing bookings from availability blocks. The lunch break is subtracted too, creating two separate working windows (pre-lunch and post-lunch) for shifts that require lunch. Setup/teardown buffers reduce the effective window at shift boundaries.

### Step 2: Determine Context for Each Free Window
Before generating slots for a free window, determine the **preceding booking context**:
- What type of booking (if any) immediately precedes this free window?
- If it was a 90min booking, was its buffer a single or double buffer?
- If the free window starts at the beginning of an availability block (or after a lunch break), there is no preceding context.

This context is needed to determine the buffer requirement for a 90min booking placed at the start of the free window.

### Step 3: Generate Valid Start Times

Start times are generated at `window_start`, `window_start + minSlot`, `window_start + 2*minSlot`, etc., where `minSlot = baseDuration + baseBuffer` (70 minutes for 60min + 10min buffer), all rounded to 5-minute boundaries.

#### Bidirectional Hole Prevention

The hole check applies in **both directions** — a slot is only valid if it does not create an unusable gap either before or after it. An unusable gap is a gap that is greater than zero but smaller than `minSlot` (the smallest bookable unit including its buffer).

**Before the slot (leading gap):**
- The gap between the window start (or previous booking's buffer end) and this slot's start time must be either **exactly 0** or **>= minSlot**
- Valid start times from window start: `window_start`, `window_start + minSlot`, `window_start + 2*minSlot`, etc.

**After the slot (trailing gap):**
- The remaining time between this slot's buffer end and the window end (or next booking's start) must be either **exactly 0** or **>= minSlot**
- If the remaining time is > 0 but < minSlot, the slot is **rejected** because it would create an unusable hole

**Example**: Free window 8:00 AM – 10:20 AM (140 minutes), minSlot = 70 min:

| Start Time | Leading Gap | Valid? | Reason |
|-----------|-------------|--------|--------|
| 8:00 AM   | 0 min       | ✓      | Window start, no gap before it |
| 8:05 AM   | 5 min       | ✗      | Leaves 5-min gap before it (< 70min) |
| ...       | ...         | ✗      | All gaps < 70min |
| 9:10 AM   | 70 min      | ✓      | Exactly minSlot from 8:00 — enough for one 60min+buffer before it |
| 10:20 AM  | 140 min     | ✗      | Past window end |

So valid start times are **8:00 AM** and **9:10 AM** only.

At 8:00 AM, for a 60min variant: end = 9:00, buffer end = 9:10. Trailing gap: 10:20 - 9:10 = 70 min. 70 ≥ 70 (minSlot) → ✓

At 8:00 AM, for a 120min variant: end = 10:00, buffer end = 10:20. Remaining = 0 → ✓ (perfect fit)

At 9:10 AM, for a 60min variant: end = 10:10, buffer end = 10:20. Remaining = 0 → ✓

Result: The 140-minute window offers either one 120min OR two 60min — the subdivisibility guarantee holds.

### Step 4: Generate Slots per Start Time
For each valid start time, try each variant (longest first):

1. **Calculate the buffer** for this variant at this position:
   - 60min: always 1 buffer (10min)
   - 120min: always 2 buffers (20min)
   - 90min: depends on what precedes this slot (see buffer rules above)

2. **Check fit**: `startTime + duration + buffer <= windowEnd` (or if last slot, buffer waived if less than 60min remains after the booking)

3. **Check trailing hole**: remaining time after buffer must be 0 or >= minSlot (same as current, but using the context-dependent buffer)

### Step 5: End-of-Window Best-Fit Rule

At the last valid start time in a window:
- If remaining time is **>= 120min + double buffer (140min)**: offer all fitting variants normally
- If remaining time is **>= 90min + applicable buffer** but **< 140min**: offer **both 90min AND 60min** (not just the longest)
- If remaining time is **>= 60min** but **< 90min + applicable buffer**: offer only 60min
- The rationale: 60min is the most popular variant and pairs best for availability. Restricting to only 90min at the end of a window unnecessarily limits options.

### Step 6: 90-Minute Availability Constraints
A 90min booking can only be offered at a start time if ONE of these holds:
1. The free window from that start time has **exactly** 90min + applicable buffer (single or double), making it a perfect fit
2. The free window from that start time has **at least** 210min (enough for a pair of 90min bookings in the alternating pattern: 90+20+90+10 = 210)
3. The start time is at the **end of window** and the remaining time fits a 90min booking (with or without buffer per end-of-window rules)

## Worked Examples

### Example 1: 7-hour 20 minute window (8:00 AM – 3:20 PM, no lunch)

**60min bookings**: 8:00, 9:10, 10:20, 11:30, 12:40, 13:50 = 6 slots
- 6×70 = 420min = 7:00. Window = 440min. Remaining 20min < 60min → 6 is the max.

**120min bookings**: 8:00 (ends 10:00, buffer 10:20), 10:20 (ends 12:20, buffer 12:40), 12:40 (ends 14:40, buffer 15:00). Remaining 20min. 3 slots.

**90min at 8:00**: Window = 440min. 90+20=110 at start, remaining 330. 330 ≥ 210? Yes → allowed.
- A at 8:00 (double buffer) → ends 9:30, buffer 9:50
- B at 9:50 (single buffer, prev 90 had double) → ends 11:20, buffer 11:30
- C at 11:30 (double buffer, prev 90 had single) → ends 13:00, buffer 13:20
- D at 13:20 (single buffer, prev 90 had double) → ends 14:50, buffer 15:00
- Remaining: 20min → done. 4 × 90min.

### Example 2: Cancelled 120min in a full day

Original: `60min(8:00-9:00) buf(9:00-9:10) 120min(9:10-11:10) buf(11:10-11:30) 60min(11:30-12:30) ...`

120min at 9:10 cancelled. Free window: 9:10 – 11:30 (140 min).

At 9:10:
- 60min: ends 10:10, buffer 10:20. Remaining: 70min ≥ 70 ✓
- 120min: ends 11:10, buffer 11:30. Remaining = 0 ✓
- 90min: 140min window. 90+20=110, remaining 30 < 70. Not exact fit (110 ≠ 140). Not allowed.

At 10:20: 60min: ends 11:20, buffer 11:30. Remaining = 0 ✓

Result: Either one 120min OR two 60min. Subdivisibility guarantee holds.

### Example 3: End-of-window with 110 minutes remaining

Last valid start at 3:10 PM. Window ends 5:00 PM. Remaining: 110 min.

- 120min: 3:10+120=5:10 > 5:00. Doesn't fit.
- 90min: 3:10+90=4:40. Double buffer (20min): buffer ends 5:00. Remaining 0 ✓. End-of-window: 110 >= 110 and < 140 → offer **both 90min and 60min**.
- 60min: 3:10+60=4:10. Buffer 10min. Buffer ends 4:20. Remaining 40min < 60min → last slot ✓

**Both 60min and 90min offered.**

### Example 4: Three consecutive 90min, middle cancelled

```
90min(8:00-9:30) double-buf(9:30-9:50) 90min(9:50-11:20) single-buf(11:20-11:30) 90min(11:30-13:00) double-buf(13:00-13:20)
```

Middle cancelled. Free window: 9:50 – 11:30 (100 min). Preceding: 90min with double buffer.

- 90min: ends 11:20, single buffer (10min). Buffer 11:30. Remaining = 0 ✓
- 60min: ends 10:50, buffer 11:00. Remaining 30min → hole < 70min. NOT valid.
- 120min: doesn't fit.

Result: Only 90min fits.

### Example 5: Two adjacent 90min cancelled

Both 8:00-9:30 and 9:50-11:20 cancelled. Free window: 8:00 – 11:30 (210 min). No preceding context.

At 8:00: 120min ✓ (remaining 70 ≥ 70), 90min ✓ (210 = exactly 2×90 + 3 buffers), 60min ✓ (remaining 140 ≥ 70)
At 9:10: 60min ✓, 120min ✓ (ends 11:10, buffer 11:30, remaining 0)
At 10:20: 60min ✓ (ends 11:20, buffer 11:30, remaining 0)

Result: Three 60min, or 120+60, or 60+120, or two 90min — all fit in 210min. ✓

### Example 6: 8-hour shift with lunch

Shift: 8:00 AM – 4:30 PM (8hr working, 30min lunch after 4th client).

Pre-lunch: 8:00 – 12:30 (270min). Slots: 8:00, 9:10, 10:20, 11:30 → 4 × 60min.
Lunch: 12:30 – 1:00 (no buffer before/after).
Post-lunch: 1:00 – 4:30 (210min). Slots: 1:00, 2:10, 3:20 → 3 × 60min.

Total: 7 × 60min clients in an 8-hour shift.

# Resolved Design Decisions

These were open questions that have been answered:

1. **Buffer overrides scale with provider**: 10min is the default. The buffer concept is what matters, not the specific number. Provider overrides scale all buffer rules proportionally.

2. **Room-based products unchanged**: Buffer rules and lunch only apply to provider-based products.

3. **Buffer stored on sessions**: `buffer_end_us` on `bookable_service_sessions` records the buffer assigned at booking time.

4. **Lunch creates two independent windows**: Lunch is calculated before bookings and creates two separate availability windows. No booking can span across a lunch break.

5. **Split shifts = no lunch**: Two separate availability blocks are independent shifts. Neither requires lunch unless individually over the threshold.

6. **No migration needed**: System has not deployed yet. No existing data to migrate.

7. **Provider preferences visible to admins**: Admin portal shows provider preferences (buffer, lunch, time hole). This is the start of a broader "staff preferences" feature class.

8. **Cascade direction**: Facility sets defaults and min/max constraints. Providers can override upward but are only paid for facility default time. Provider preference > facility default (within bounds).

9. **Setup/teardown**: Add capability with zero defaults. Support now for future use, not enabled initially.

10. **No shifts > 8 hours**: Business policy. Emergency coverage uses a separate shift with appropriate gap.

11. **Lunch invisible to customers**: Customers see availability blocks. Lunch just creates a gap — customer doesn't know why.

12. **Shift entity snapshots settings**: Once the first booking exists in a shift, settings are frozen. Un-booked shifts use current settings until the first booking materializes the shift.

13. **Walk-ins respect availability**: Walk-in bookings go through the normal availability system. No skipping availability checks. Provider must opt-in to walk-ins.

14. **Mid-session extensions don't cancel original**: Add upgrade line item, charge delta, update session times. Can override availability checks.

# Open Questions

1. **Walk-in "in-session" detection**: To prevent booking the slot immediately after a provider currently in session, we need to reliably determine if a provider is in session right now. The simplest approach: check if the current time falls within a booked session's `start_time_us` to `end_time_us` range. Is this sufficient, or do we need check-in status as a more reliable indicator?
	- Mason- let's just do start_time_us / end_time_us

2. **Upgrade product variant linking**: When extending 60→90min, we need the 90min variant's pricing. Should the upgrade create a new purchase item referencing the 90min variant with a negative adjustment for the already-paid 60min? Or should it create a special "upgrade" item type with just the delta price?
	- Mason- I'm on the fence about this one. I don't want to cancel the existing one since that g

3. **Upgrade and existing entitlements**: A 60min booking creates an entitlement. When upgraded to 90min, should the original entitlement be modified, or should a new supplemental entitlement be created?

4. **Walk-in provider opt-in default**: Should "accepts walk-ins" default to true or false? If true, providers who don't want walk-ins need to opt out. If false, walk-ins are an opt-in feature. Recommendation: default true — walk-ins are a standard part of the business.

5. **Shift materialization trigger**: Should the shift record be created at the moment of the first booking, or at the start of the shift's clock time? Creating at booking time means the settings snapshot happens earlier, which is safer. But if a booking is cancelled and the shift has no bookings again, should the shift record be deleted (returning to virtual state)?

6. **Lunch visibility in provider portal**: The lunch break should be visible to providers in their schedule view. Should it appear as a distinct "Lunch" block, or just as unavailable time? Distinct block seems more user-friendly and provides a clear visual indicator.

# Discussion & Suggestions

## 1. Rest Break Compliance
Some states require paid 10-minute rest breaks for every 4 hours worked. With 10min buffers between clients, providers get regular micro-breaks. For a 7-slot day (7 hours of client time with 6 buffers), the provider gets 60 minutes of buffer time spread throughout the day plus a 30-minute lunch. This likely satisfies most rest break requirements, but you may want to verify with a labor attorney for your specific jurisdiction.

## 2. Overtime Tracking
The lunch extension means an 8-hour shift runs 8.5 hours on the clock. This should NOT count as overtime. The system should track working time (excluding lunch) separately from clock time. This distinction matters for payroll compliance.

## 3. Cancellation Window for Walk-Ins
Walk-ins have a 15-minute minimum booking window. But should there also be a cancellation policy? If a walk-in is booked and the customer changes their mind 5 minutes later, the therapist may have already started preparing. Consider: walk-in bookings could have a shorter or different cancellation policy than pre-booked appointments.

## 4. Extension Notification
When a therapist extends a session, the front desk system should be aware so they don't try to book walk-ins into the now-occupied slot. The extension should immediately update the session times in the database and recalculate availability. Should the front desk get a real-time notification (e.g., via websocket or polling)?

## 5. Provider Schedule Transparency
With the shift model, providers should see their materialized shift settings in their portal. If a provider changes their buffer from 10 to 15 minutes, they should understand that existing shifts keep the old setting while future un-booked shifts will use the new setting. A clear visual distinction (e.g., "Settings locked for this shift — first booking was made on April 10") would help avoid confusion.

## 6. Walk-In Queue
For busy periods, there may be multiple walk-in customers waiting. Should the system support a simple queue or waitlist for walk-ins? This is different from the event waitlist — it's more of a "next available" queue. Not necessarily for this implementation, but worth considering for the data model.

## 7. Grace Period for "In-Session" Walk-In Block
The rule that walk-ins can't book the slot after an in-session provider should have a time window. If a provider's session ended 2 minutes ago and they haven't checked in the next client, they're technically not "in session" but may still be wrapping up. Consider: the in-session block should extend for buffer-minutes after the session's end_time_us, not just during the session itself.

# Implementation Plan

## Phase 1: Data Model — Shifts and Configuration
*Add the shift entity, facility config, and provider preferences*

### 1. Database: Shift table
- [ ] Create `provider_shifts` table: id, provider_person_id, facility_id, shift_start_us, shift_end_us, lunch_start_us (nullable), lunch_end_us (nullable), effective_buffer_minutes, setup_buffer_minutes, teardown_buffer_minutes, max_time_hole_minutes, created_us
- [ ] Schema in `db_schema/`, table helper in `sql_util/table_helpers/`
- [ ] Tests for CRUD operations

### 2. Database: Facility scheduling config in secrets
- [ ] Add secrets: `scheduling_lunch_threshold_minutes` (default 360), `scheduling_lunch_length_minutes` (default 30), `scheduling_min_buffer_minutes` (default 10), `scheduling_setup_buffer_minutes` (default 0), `scheduling_teardown_buffer_minutes` (default 0), `scheduling_walkin_min_buffer_minutes` (default 15)
- [ ] Tests for secrets lookup with defaults

### 3. Database: Provider preferences
- [ ] Add columns to `provider_type_assignments`: `preferred_buffer_minutes` (nullable), `preferred_lunch_minutes` (nullable), `accepts_walkins` (boolean, default true), `preferred_setup_minutes` (nullable), `preferred_teardown_minutes` (nullable)
- [ ] Table helper methods for get/set
- [ ] Validation: preferred values must be >= facility minimums
- [ ] Tests

### 4. Update default buffer from 5min to 10min
- [ ] Update `product_variants` default buffer values in database helper / seed data
- [ ] Tests

## Phase 2: Algorithm — Lunch Placement
*Implement the lunch break calculation and shift splitting*

### 1. Lunch placement calculator (pure function)
- [ ] Create `CalculateLunchPlacement(shiftDurationMinutes, bufferMinutes, lunchMinutes, lunchThresholdMinutes)` → returns minute offset into shift where lunch starts, or nullopt if no lunch needed
- [ ] Balanced-split algorithm: try each slot boundary, compute max delta, pick smallest
- [ ] Constraint: lunch must start between hour 2 and hour 5
- [ ] Tests for 5hr (no lunch), 6hr, 7hr, 8hr shifts, edge cases

### 2. Integration with availability computation
- [ ] In `ComputeAvailableSlots`, detect shifts meeting lunch threshold
- [ ] Calculate lunch placement, split availability into pre-lunch and post-lunch windows
- [ ] Apply setup/teardown buffers to window boundaries
- [ ] Tests

## Phase 3: Algorithm — Context-Dependent Buffer Calculation
*Modify buffer calculation and slot generation for the new rules*

### 1. New buffer calculation function
- [ ] Create `CalculateBufferForVariant(variantDurationMinutes, baseDurationMinutes, effectiveBufferMinutes, precedingDurationMinutes, precedingBufferMinutes)` implementing context-dependent rules
- [ ] Tests for all buffer cases

### 2. Update ComputeSlotsForFreeWindow
- [ ] Add `precedingBookingContext` parameter
- [ ] Thread context through start time generation and slot generation
- [ ] Implement slot-to-slot context propagation within a window
- [ ] Tests

### 3. 90-minute availability constraints
- [ ] Implement the three conditions under which 90min can be offered
- [ ] Tests

### 4. Modified end-of-window rule
- [ ] Offer both 60min and 90min at end of window when either fits
- [ ] Tests

### 5. Port and extend all existing algorithm tests
- [ ] Update all existing tests for new signature and 10min buffer
- [ ] Add tests for all worked examples

## Phase 4: Integration — Booking Creation, Shifts, and Context
*Update booking flow for shift materialization and context-dependent buffers*

### 1. Shift materialization on first booking
- [ ] When creating a booking, check if a shift record exists for this provider/availability block
- [ ] If not, create one with snapshotted settings
- [ ] If yes, use the shift's snapshotted settings for buffer calculation
- [ ] Tests

### 2. Update booking creation with context-dependent buffers
- [ ] `ServiceBookingHelper` and `CartCheckoutHelper` calculate and store correct `buffer_end_us`
- [ ] Pass preceding context from existing bookings in the shift
- [ ] Tests

### 3. Pass preceding context in ComputeAvailableSlots
- [ ] Extract preceding booking duration and buffer when building free windows
- [ ] Pass to `ComputeSlotsForFreeWindow`
- [ ] Integration tests

## Phase 5: Walk-In Booking Constraints
*Fix walk-in flow to respect availability and add new constraints*

### 1. Remove skipAvailabilityCheck from walk-in flow
- [ ] Update `staff_dropin_booking.cpp` to use normal availability checking
- [ ] Add walk-in minimum booking window check
- [ ] Add in-session provider blocking (no booking slot after currently in-session provider)
- [ ] Tests

### 2. Provider walk-in settings
- [ ] Filter available providers by `accepts_walkins` for walk-in requests
- [ ] API endpoint to toggle walk-in acceptance
- [ ] Tests

### 3. Admin override for "already started" slots
- [ ] Allow admin to book a slot where start_time_us is in the past (within buffer period)
- [ ] Tests

## Phase 6: Mid-Session Extension (Upgrades)
*Allow staff to extend a session duration mid-appointment*

### 1. Upgrade endpoint
- [ ] Create `POST /api/staff/upgrade_session/{sessionId}` with target variant
- [ ] Calculate price delta from original variant to target variant
- [ ] Create upgrade purchase item (staff/admin only visible)
- [ ] Update session `end_time_us` and `buffer_end_us`
- [ ] Override availability checks
- [ ] Tests

### 2. Upgrade UI
- [ ] Add "Extend Session" option on staff check-in / provider portal active session view
- [ ] Show available upgrade paths (60→90, 60→120, 90→120)
- [ ] Confirm with price delta
- [ ] Tests

## Phase 7: Staff Portal — Provider Preferences UI
*Add UI for providers to manage scheduling preferences*

### 1. API endpoints
- [ ] `GET /api/provider/preferences` — returns current buffer, lunch, time hole, walk-in settings
- [ ] `PUT /api/provider/preferences` — updates preferences with validation
- [ ] Tests

### 2. Provider portal UI
- [ ] Add "Scheduling Preferences" section to provider portal
- [ ] Fields: preferred buffer (min = facility min), preferred lunch length (min = facility min), max time hole, accepts walk-ins
- [ ] Show lunch break in schedule view as distinct block
- [ ] Validation and save
- [ ] Component tests

### 3. Admin visibility
- [ ] Show provider preferences in manage portal staff view
- [ ] Read-only view for admins (providers set their own preferences)
- [ ] Tests

## Phase 8: Documentation Update
*Move algorithm documentation from BSF doc to this document*

### 1. Update Bookable Service Foundation.md
- [ ] Remove the scheduling algorithm section
- [ ] Replace with reference: "See [[Bookable Service Scheduling Algorithm]] for the complete scheduling algorithm description."
- [ ] Verify no other references need updating
