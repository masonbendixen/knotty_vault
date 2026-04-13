---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/13/2026
Version: 0.2
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

Please copy the existing scheduling algorithm into this document with the examples and override it to support the changes listed in this document but leave all the rest unchanged. After we iterate and get it completed, we can work on an implementation plan. Note that the referenced document and algorithm are already implemented so the implementation plan will be modifying the existing code base to implement the tweaked algorithm. After we have implemented the implementation plan, we should modify Bookable Service Foundation.md to remove the scheduling algorithm from that document and replace it with a reference to this document. This should be the final step in the implementation plan.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Scheduling Algorithm

## Core Constants

- **Slot alignment**: 5 minutes (all start times snap to 5-minute boundaries)
- **Base unit**: 60 minutes
- **Buffer**: 10 minutes default (configurable per-provider, minimum 10 minutes)
- **Variants**: 60min (1x), 90min (1.5x), 120min (2x)

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

### Lunch Placement Algorithm

When a shift meets or exceeds the threshold, the algorithm determines optimal lunch placement:

1. **Extend shift**: Add lunch duration to the total clock time. An 8hr shift with 30min lunch runs 8:00 AM – 4:30 PM on the clock, with the provider working 8 hours.

2. **Constraint**: Lunch must begin **between the 2nd and 5th hours** of the shift (measured from shift start, in working time). This ensures neither the morning nor afternoon portion is unreasonably short or long.

3. **Calculate split candidates**: Compute 60min slots with buffers from the start of the shift. Each buffer boundary between slots is a candidate lunch start point (within the 2nd–5th hour constraint).

4. **Select optimal placement**: For each candidate, calculate the working time before lunch and working time after lunch. Compute the delta of each half from the midpoint of total working time. Pick the candidate with the **smallest maximum delta** — this creates the most balanced split.

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

## Slot Generation Algorithm

### Step 1: Build Free Windows
For each provider, determine free windows by subtracting existing bookings from availability blocks. **NEW**: Also subtract the computed lunch break from the availability, creating two separate working windows (pre-lunch and post-lunch) for shifts that require lunch.

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

This prevents booking a 90min into a slot where the remaining time is awkward (e.g., 155 minutes — not enough for two 90s, and a lone 90 would leave a gap that can only hold a 60min, wasting the potential for a 120min or two 60s).

## Worked Examples

### Example 1: 7-hour 20 minute window (8:00 AM – 3:20 PM, no lunch)

**60min bookings**: 8:00, 9:10, 10:20, 11:30, 12:40, 13:50 = 6 slots
- Last booking: 13:50+60=14:50. Buffer ends 15:00. Remaining: 15:20-15:00 = 20min < 60min → 6 slots is the max
- Wait, 15:00+20min remaining. Let's verify: 6×70=420min=7:00. Window=7:20=440min. 440-420=20min leftover. So slot 6 at 13:50 ends at 14:50, buffer 15:00. Remaining 20min → no more. ✓

**120min bookings**: 8:00 (ends 10:00, buffer 10:20), 10:20 (ends 12:20, buffer 12:40), 12:40 (ends 14:40, buffer 15:00). Remaining 20min. 3 slots.
- Or mix: 120min at 8:00, then 60min at 10:20, 11:30, 12:40, 13:50. = 1×120 + 4×60.

**90min**: Check at 8:00 — window is 440min. 90+20=110 at start, remaining 330. 330 ≥ 210? Yes → allowed.
- 90min at 8:00 (double buffer) → ends 9:30, buffer ends 9:50
- Next start: 9:50. 90min (previous 90 with double buffer) → single buffer → ends 11:20, buffer ends 11:30
- Next start: 11:30. 90min (previous 90 with single buffer) → double buffer → ends 13:00, buffer ends 13:20
- Next start: 13:20. 90min (previous 90 with double buffer) → single buffer → ends 14:50, buffer ends 15:00
- Remaining: 20min < 60min → done. 4 × 90min slots.

### Example 2: Cancelled 120min in a full day

Original schedule:
```
60min(8:00-9:00) buf(9:00-9:10) 120min(9:10-11:10) buf(11:10-11:30) 60min(11:30-12:30) ...
```

120min at 9:10 gets cancelled. Free window: 9:10 – 11:30 (140 min).

At 9:10:
- 60min: 9:10–10:10, buffer ends 10:20. Remaining: 11:30–10:20 = 70min ≥ 70min (minSlot) ✓
- 120min: 9:10–11:10, buffer ends 11:30. Remaining = 0 ✓ (perfect fit)
- 90min: Window is 140min. 90+20=110, remaining 30min. Not enough for second slot. 90+buffer=110 ≠ 140. Not allowed.

At 10:20:
- 60min: 10:20–11:20, buffer ends 11:30. Remaining = 0 ✓

Result: Either one 120min OR two 60min. Subdivisibility guarantee holds.

### Example 3: End-of-window with 110 minutes remaining

Last valid start time at 3:10 PM. Window ends at 5:00 PM. Remaining: 110 minutes.

- 120min: 3:10+120=5:10 > 5:00. Doesn't fit.
- 90min: 3:10+90=4:40. Double buffer (20min): buffer ends 5:00. Remaining after: 0 ✓. But also check end-of-window rule: 110 >= 90+20=110 and 110 < 140. → Offer **both 90min and 60min**.
- 60min: 3:10+60=4:10. Buffer: 10min. Buffer ends 4:20. Remaining: 40min < 60min → last slot. ✓

**Both 60min and 90min offered.** User can choose the popular 60min.

### Example 4: Three consecutive 90min, middle cancelled

```
90min(8:00-9:30) double-buf(9:30-9:50) 90min(9:50-11:20) single-buf(11:20-11:30) 90min(11:30-13:00) double-buf(13:00-13:20)
```

Middle (9:50-11:20) cancelled. Free window: 9:50 – 11:30 (100 min).

Preceding context: Previous was 90min with double buffer.

At 9:50:
- 90min: 9:50+90=11:20. Single buffer (10min, prev 90 had double). Buffer ends 11:30. Remaining = 0. Perfect fit ✓
- 60min: 9:50+60=10:50. Buffer: 10min. Buffer ends 11:00. Remaining: 11:30-11:00 = 30min. Gap > 0 but < 70min → NOT valid.
- 120min: 9:50+120=11:50 > 11:30. Doesn't fit.

Result: Only 90min fits. Correct — 60min would leave a 30min hole.

### Example 5: Two adjacent 90min cancelled

Both 8:00-9:30 and 9:50-11:20 cancelled. Free window: 8:00 – 11:30 (210 min).

No preceding context (start of block).

At 8:00:
- 120min: ends 10:00, buffer ends 10:20. Remaining: 11:30-10:20 = 70min ≥ 70 ✓
- 90min: Window=210min. 90+20=110, remaining=100. Is 100 exactly 90+10? Yes (second 90 gets single buffer). Total 110+100=210 ✓
- 60min: ends 9:00, buffer ends 9:10. Remaining: 11:30-9:10 = 140min ≥ 70 ✓

At 9:10:
- 60min: ends 10:10, buffer ends 10:20. Remaining: 70min ≥ 70 ✓
- 120min: ends 11:10, buffer ends 11:30. Remaining = 0 ✓

At 10:20:
- 60min: ends 11:20, buffer ends 11:30. Remaining = 0 ✓

Result: Three 60min, or 120+60, or 60+120, or two 90min — all fit in 210min. ✓

### Example 6: 8-hour shift with lunch (10min buffer, 30min lunch)

Shift: 8:00 AM – 4:30 PM (8hr working, 30min lunch).
Lunch placement: after 4th client (see lunch algorithm above).

Pre-lunch window: 8:00 AM – 12:30 PM (270min = 4.5hr working time)
- Slots: 8:00, 9:10, 10:20, 11:30 → 4 × 60min

Lunch: 12:30 PM – 1:00 PM (no buffer before/after)

Post-lunch window: 1:00 PM – 4:30 PM (210min = 3.5hr working time)
- Slots: 1:00, 2:10, 3:20 → 3 × 60min
- Remaining after 3:20+60+10 = 4:30. Remaining = 0. ✓

Total: 7 × 60min clients in an 8-hour shift.

# Open Questions

1. **Provider buffer overrides**: The current system allows per-provider buffer overrides (e.g., a provider may require 15min instead of 10min). The 90min alternating double/single buffer logic scales with the provider's effective buffer. (i.e., if override is 15min, single buffer = 15min and double buffer = 30min.)
	- Mason- Yes, 10min is just a default. Buffer is the concept and whatever is configured is used. It is the buffer that is relevant, not the ten minutes.

2. **Room-based products**: The room availability algorithm (`room_availability_helper.cpp`) uses a completely different slot generation model (fixed 15-minute intervals, no buffers, capacity-based). The new buffer rules only apply to provider-based products.
	- Mason- Yes, this does not apply to room based products.

3. **Existing bookings**: When checking the context of preceding bookings, we need to know what buffer was assigned to each existing 90min session. This is stored on the `bookable_service_sessions` table via `buffer_end_us`. The difference between `buffer_end_us` and `end_time_us` reveals the buffer duration assigned at booking time.
	- Mason- I will go with your recommendation.

4. **Lunch and existing bookings**: When a shift has existing bookings AND meets the lunch threshold, the lunch placement should be calculated from the original shift availability (not from free windows between existing bookings). The lunch position is determined once at schedule generation time and treated like an unavailable block. If a provider has bookings that span across where the lunch would go, the lunch cannot be placed there — should the algorithm shift the lunch to the next valid gap, or should it flag a scheduling conflict?
	- Mason- The lunch is part of the booking process and is calculated before bookings are allowed. The lunch spot is generated and placed. The before and after lunch are separate windows (other than the first window ending on a nice alignment boundary) that are basically independent.

5. **Lunch for split shifts**: If a provider has two availability blocks (e.g., 8-12 and 1-5), should lunch be calculated per-block or for the combined working time? If the gap between blocks already serves as a lunch, no additional lunch is needed.
	- Mason- They are two separate shifts at that point with no lunch. 

6. **Minimum buffer change impact on existing data**: The current system uses 5min default buffer. Changing to 10min default means existing provider configurations may need updating. Should we migrate existing 5min buffers to 10min, or grandfather them in? Provider overrides that were explicitly set to a specific value should be respected.
	- Mason- we have not deployed yet and are still in development so there is nothing to migrate.

7. **Provider preferences visibility**: The staff portal additions (preferred lunch length, preferred buffer, time hole setting) — should these be visible to admins in the manage portal as well? Can admins override provider preferences?
	- Mason- It would be nice to know the staff preferences to the admins. Knowing their buffer length preferences could help with planning shifts that end smoothly with the buffer preferences. I can imagine there will be a number of staff preferences that would be useful for admins to know so this could be the start for that class of data.

# Discussion & Suggestions

## Additional Considerations for the Algorithm

### 1. Setup/Teardown Time at Shift Boundaries
Should there be a configurable "no booking" period at the very start and end of a shift? For example, 5-10 minutes for a therapist to set up their room at the beginning and clean up at the end. This is different from buffer (which is between clients). Currently, the first booking starts right at the shift start.
- Mason- I think we should add the capability but set both the start and end buffer to zero by default. My general feeling is that this is more of a shift scheduling thing and we should never schedule shifts with no buffer between them but it's generally assumed the therapist will be there a little early if they need to set up and stay a little late if they need to do cleanup. But it would be expensive to add later so let's do the support now even though I don't plan on enabling it currently.

### 2. Short Breaks vs. Lunch
The 10min buffer between clients provides a micro-break. But for very long shifts (10+ hours), should there be mandatory short breaks in addition to lunch? Some jurisdictions require a paid 10-15 min break for every 4 hours worked. This could be implemented as a second tier of the lunch system — a shorter break that doesn't need the balanced-split algorithm.
- Mason- I don't plan on having people work shifts longer than 8 hours. I feel like massage is hard and it just isn't something that people can sustainably do for those lengths. However, if there is an emergency and someone needs to cover for someone in a pinch and work a really long day, we can schedule this as a separate shift with an appropriate buffer after the 8 hours to make this sustainable / legal.

### 3. Lunch Visibility to Customers
The lunch break should appear as unavailable time in the slot search results. Currently, the system generates free windows by subtracting bookings from availability. The lunch break should be subtracted too, effectively splitting a long availability block into two shorter ones.
- Mason- The customer just sees availability blocks. As discussed before, the lunch break will just be blocked out and essentially create two separate availability windows so the client won't necessarily be aware that there is a lunch break. They will just see that there is no availability at that time. For all they know, the previous person did a 30min longer massage. Is there a reason you think that the person would need to know this information?

### 4. Provider Preference Defaults from Facility
The provider preferences (buffer, lunch length, time hole) should cascade: system default → facility override → provider preference. This gives facility managers control while allowing provider customization within bounds.
- Mason- I feel like it should be the opposite (minus facility maximum / minimum values). The facility sets a default (like a buffer window of 10min) but the provider can override that if they choose. However, they are only paid for the facility default values. So if they increase their buffer to 20min, that's fine, but they aren't going to be paid for the extra ten minutes because that was their choice.

### 5. Impact on Walk-In Staff Bookings
The staff check-in walk-in flow currently uses `skipAvailabilityCheck` for drop-in bookings. These bookings should still respect lunch breaks (you can't book a client during the provider's lunch), but buffer validation is already skipped. The lunch break should be a hard constraint even for walk-ins.
- Mason- It does? Um... that's news to me and was not intended. A walk-in should not disrupt the availability system and we should only show windows that are available. Also, we need two settings for providers for walk in. One is if they allow walk in customers, and, if so, what kind of buffer window to allow for booking. For the website, we have a minimum window for which booking is allowed. If a therapist sees they have no client in their first slot, they might not come in until their second slot. A therapist might see a gap in their schedule and decide to take a call, go run an errand, take a class at the studio, use the spa. We can't have someone walk in three min before a slot, have it booked, and expect a therapist to be ready. I think we should also have a facility min buffer for booking a walkin massage and I think it should default to 15min. Regardless, booking walkins should respect the availability system. Otherwise, that would create all kind of weird schedule holes.

### 6. Historical Consistency
When a provider changes their buffer or lunch preference, existing future bookings should NOT be retroactively recalculated. The buffer assigned at booking time is stored on the session and remains fixed. Only new bookings use the updated preferences.
- Mason- Yeah, past bookings should use the old settings. I feel like it would be useful to trac

### 7. Minimum Buffer Enforcement
The document specifies minimum buffer of 10min. But what if a legacy provider has a 5min override? We should either:
- Enforce the 10min minimum at the API level (reject overrides below 10min)
- Or display a warning but allow it for backwards compatibility

I'd recommend enforcing the minimum — it's a business policy decision that the buffer serves as a break.

# Implementation Plan

## Phase 1: Configuration — Lunch and Buffer Settings
*Add facility-level lunch settings and update buffer defaults*

### 1. Database: Add lunch configuration to secrets/config
- [ ] Add secrets: `scheduling_lunch_threshold_minutes` (default 360), `scheduling_lunch_length_minutes` (default 30), `scheduling_min_buffer_minutes` (default 10)
- [ ] Tests for secrets lookup with defaults

### 2. Database: Add provider lunch preference
- [ ] Add `preferred_lunch_minutes` column to `provider_type_assignments` table (nullable — null means use facility default)
- [ ] Add table helper method for getting/setting the value
- [ ] Validation: must be >= facility lunch length when set
- [ ] Tests

### 3. Update default buffer from 5min to 10min
- [ ] Update `product_variants` default buffer values in database helper / seed data
- [ ] Ensure existing provider buffer overrides below 10min are handled (enforce minimum)
- [ ] Tests

## Phase 2: Algorithm — Lunch Placement
*Implement the lunch break calculation and shift splitting*

### 1. Lunch placement calculator (pure function)
- [ ] Create `CalculateLunchPlacement(shiftDurationMinutes, bufferMinutes, lunchMinutes, lunchThresholdMinutes, lunchWindowStartHour, lunchWindowEndHour)` → returns minute offset into shift where lunch starts, or nullopt if no lunch needed
- [ ] Implements the balanced-split algorithm from this document
- [ ] Tests for 6hr, 7hr, 8hr, 5hr (no lunch), edge cases

### 2. Integration with availability computation
- [ ] In `ComputeAvailableSlots`, after loading provider availability, detect shifts meeting the lunch threshold
- [ ] Calculate lunch placement for each qualifying shift
- [ ] Split the availability window into pre-lunch and post-lunch windows
- [ ] Lunch window treated as unavailable (no slots generated in it)
- [ ] Tests

## Phase 3: Algorithm — Context-Dependent Buffer Calculation
*Modify buffer calculation and slot generation for the new rules*

### 1. New buffer calculation function
- [ ] Create `CalculateBufferForVariant(variantDurationMinutes, baseDurationMinutes, effectiveBufferMinutes, precedingBookingDurationMinutes, precedingBookingBufferMinutes)` implementing context-dependent rules
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
- [ ] Update all existing `ComputeSlotsForFreeWindow` tests for new signature and 10min buffer
- [ ] Add tests for all new scenarios from worked examples

## Phase 4: Integration — Booking Creation and Context
*Update booking flow to store correct buffers and pass context*

### 1. Update booking creation
- [ ] `ServiceBookingHelper` and `CartCheckoutHelper` calculate context-dependent buffer when creating sessions
- [ ] Store correct `buffer_end_us` on `bookable_service_sessions`
- [ ] Tests

### 2. Pass preceding context in ComputeAvailableSlots
- [ ] Extract preceding booking duration and buffer when building free windows
- [ ] Pass to `ComputeSlotsForFreeWindow`
- [ ] Integration tests

## Phase 5: Staff Portal — Provider Preferences UI
*Add UI for providers to manage their scheduling preferences*

### 1. API endpoints
- [ ] `GET /api/provider/preferences` — returns current buffer, lunch, time hole settings
- [ ] `PUT /api/provider/preferences` — updates preferences with validation
- [ ] Tests

### 2. Provider portal UI
- [ ] Add "Scheduling Preferences" section to provider portal
- [ ] Fields: preferred buffer (min 10min), preferred lunch length (min = facility min), max time hole
- [ ] Validation and save
- [ ] Component tests

## Phase 6: Documentation Update
*Move algorithm documentation from BSF doc to this document*

### 1. Update Bookable Service Foundation.md
- [ ] Remove the scheduling algorithm section
- [ ] Replace with a reference to this document: "See [[Bookable Service Scheduling Algorithm]] for the complete scheduling algorithm description."
- [ ] Verify no other references need updating
