---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/13/2026
Version: 0.1
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
- **Buffer**: 5 minutes (configurable per-variant, overridable per-provider)
- **Variants**: 60min (1x), 90min (1.5x), 120min (2x)

## Buffer Rules

The buffer model is designed to maximize rebookability — when a slot is cancelled, the freed time should be fillable with the maximum number of alternative configurations.

### 60-Minute Bookings
- Always require **1 buffer** (5min) after the booking
- A cancelled 60min slot frees 65 minutes — enough for exactly one more 60min + buffer

### 120-Minute Bookings
- Always require **2 buffers** (10min) after the booking
- A cancelled 120min slot frees 130 minutes — enough for exactly two 60min bookings (60+5+60+5 = 130min)
- This is the key **subdivisibility guarantee**: a 120min block is always perfectly replaceable by two 60min blocks

### 90-Minute Bookings (Context-Dependent Buffers)
The 90-minute variant requires context-dependent buffers because 90 is not an exact multiple of 60. The buffer after a 90min booking depends on the **previous booking** in the sequence:

**Rule: Alternating double/single buffers for consecutive 90-minute bookings**

1. **First 90min in a block, or previous was NOT 90min** → **double buffer** (10min) after this 90min
2. **Previous was 90min with double buffer** → **single buffer** (5min) after this 90min
3. **Previous was 90min with single buffer** → **double buffer** (10min) after this 90min

This creates an alternating pattern for consecutive 90-min bookings:

```
90min, 10min buffer, 90min, 5min buffer, 90min, 10min buffer, ...
  A      (double)      B     (single)      C      (double)
```

**Why this works:**
- A pair of 90min bookings (A + B) occupies: 90 + 10 + 90 + 5 = 195 minutes = 3 hours + 15 minutes (3 buffers)
- If both A and B are cancelled, the 195 minutes can hold:
  - Three 60min bookings: 60+5+60+5+60+5 = 195 ✓
  - One 120min + one 60min: 120+10+60+5 = 195 ✓
  - One 60min + one 120min: 60+5+120+10 = 195 ✓
  - Two 90min bookings (same as before): 90+10+90+5 = 195 ✓
- If only one of the pair is cancelled, only a 90min replacement fits (60min would leave a 30min hole)

**Island 90-minute bookings:**
- A 90min booking sandwiched between 60min or 120min bookings is an "island" — if cancelled, it can ONLY be replaced by another 90min booking
- A 120min booking that is cancelled with no adjacent free slots can NEVER be replaced by a 90min booking (would create a 30min hole)

### Buffer at End of Availability Window
- If the remaining time after the last booking is **less than 60 minutes** (the base duration), no buffer is required — the booking is simply the last one in the window
- This applies regardless of variant duration

### Summary Table

| Variant | Buffer Count | Buffer Duration (5min base) | Total Occupied | Condition |
|---------|-------------|----------------------------|----------------|-----------|
| 60min   | 1           | 5min                       | 65min          | Always |
| 90min   | 2           | 10min                      | 100min         | First in block, or previous was non-90min, or previous 90min had single buffer |
| 90min   | 1           | 5min                       | 95min          | Previous was 90min with double buffer |
| 120min  | 2           | 10min                      | 130min         | Always |

## Slot Generation Algorithm

### Step 1: Build Free Windows
For each provider, determine free windows by subtracting existing bookings from availability blocks (unchanged from current implementation).

### Step 2: Determine Context for Each Free Window
**NEW**: Before generating slots for a free window, determine the **preceding booking context**:
- What type of booking (if any) immediately precedes this free window?
- If it was a 90min booking, was its buffer a single or double buffer?
- If the free window starts at the beginning of an availability block, there is no preceding context.

This context is needed to determine the buffer requirement for a 90min booking placed at the start of the free window.

### Step 3: Generate Valid Start Times
Same as current: start times are generated at `window_start`, `window_start + minSlot`, `window_start + 2*minSlot`, etc., where `minSlot = baseDuration + baseBuffer` (65 minutes for 60min + 5min buffer), all rounded to 5-minute boundaries.

**Bidirectional hole prevention** remains unchanged: a start time is only valid if the gap before it (from window start or previous slot's buffer end) is either 0 or >= minSlot.

### Step 4: Generate Slots per Start Time
For each valid start time, try each variant (longest first):

1. **Calculate the buffer** for this variant at this position:
   - 60min: always 1 buffer
   - 120min: always 2 buffers
   - 90min: depends on what precedes this slot (see buffer rules above)

2. **Check fit**: `startTime + duration + buffer <= windowEnd` (or if last slot, buffer waived if less than 60min remains after the booking)

3. **Check trailing hole**: remaining time after buffer must be 0 or >= minSlot (same as current, but using the context-dependent buffer)

### Step 5: End-of-Window Best-Fit Rule (MODIFIED)

**Current behavior**: At the last valid start time, offer only the longest fitting variant (best-fit).

**New behavior**: At the last valid start time in a window:
- If remaining time is **>= 120min + double buffer**: offer all fitting variants normally
- If remaining time is **>= 90min + applicable buffer** but **< 120min + double buffer**: offer **both 90min AND 60min** (not just 90min)
- If remaining time is **>= 60min** but **< 90min + applicable buffer**: offer only 60min
- The rationale: 60min is the most popular variant and pairs best for availability. Restricting to only 90min at the end of a window unnecessarily limits options.

### Step 6: 90-Minute Availability Constraints
A 90min booking can only be offered at a start time if ONE of these holds:
1. The free window from that start time has **exactly** 90min + applicable buffer (single or double), making it a perfect fit
2. The free window from that start time has **at least** 180min + 3×buffer (enough for a pair of 90min bookings in the alternating pattern)
3. The start time is at the **end of window** and the remaining time fits a 90min booking (with or without buffer per end-of-window rules)

This prevents booking a 90min into a slot where the remaining time is awkward (e.g., 155 minutes — not enough for two 90s, and a lone 90 would leave a 60-minute gap that can only hold a 60min, wasting the potential for a 120min or two 60s).

## Worked Examples

### Example 1: 5 hours 20 minutes window (8:00 AM – 1:20 PM)

**60min bookings**: 8:00, 9:05, 10:10, 11:15, 12:20 = 5 slots
- Last booking ends 1:20 PM, window ends 1:20 PM — perfect fit

**120min bookings**: 8:00, 10:10, 12:20 — but 12:20+120=2:20 PM > 1:20 PM, so only 2 slots at 8:00 and 10:10
- After 10:10+120+10 = 12:20, remaining = 60min → fits one 60min

**90min at 8:00**: Check constraint — window is 320min. 90+10=100 at start, remaining 220min. 220 >= 180+15=195? Yes → allowed
- 90min at 8:00 (double buffer, first in block) → buffer ends 9:40
- Next start: 9:45 (rounded to 5-min boundary after 9:40)
- At 9:45: 90min (previous was 90min with double buffer) → single buffer → buffer ends 11:20
- Next start: 11:20
- At 11:20: 90min (previous was 90min with single buffer) → double buffer → buffer ends 1:00
- Remaining: 1:00 to 1:20 = 20min < 60min → no more slots
- Total: 3 × 90min slots

### Example 2: Cancelled 120min in a full day (8:00 AM – 4:00 PM)

Original schedule:
```
60min(8:00-9:00) buf(9:00-9:05) 120min(9:05-11:05) buf(11:05-11:15) 60min(11:15-12:15) ...
```

120min at 9:05 gets cancelled. Free window: 9:05 – 11:15 (130 min).

Available at 9:05:
- 60min: 9:05–10:05, buffer ends 10:10. Remaining 11:15–10:10 = 65min ≥ 65min (minSlot) ✓
- 120min: 9:05–11:05, buffer ends 11:15. Remaining = 0 ✓ (perfect fit)
- 90min: Check constraint — window is 130min. 90+10=100, remaining 30min < 65min. Not enough for a second slot. Is it exactly 90+buffer? 90+10=100 ≠ 130. Not allowed.

Next start: 10:10 (9:05 + 65min):
- 60min: 10:10–11:10, buffer ends 11:15. Remaining = 0 ✓
- 90min: 10:10+90=11:40 > 11:15. Doesn't fit.
- 120min: 10:10+120=12:10 > 11:15. Doesn't fit.

Result: The 130min window offers either one 120min OR two 60min — exactly the subdivisibility guarantee.

### Example 3: End-of-window with 110 minutes remaining

Availability window ends at 6:00 PM. Last valid start time at 4:10 PM. Remaining: 110 minutes.

**Current algorithm**: Only offers 90min (best fit, longest that fits).

**New algorithm**: 
- 120min: 4:10+120=6:10 > 6:00. Doesn't fit.
- 90min: 4:10+90=5:40. Buffer: double (10min, assuming first/non-90 preceding). 5:50 PM. Remaining after buffer: 10min < 60min → buffer waived at end of window. Fits ✓
- 60min: 4:10+60=5:10. Buffer: 5min. 5:15 PM. Remaining after: 45min < 60min → this is the last slot. Fits ✓

**Both 60min and 90min are offered** at 4:10 PM. This is the key change — the user can choose the popular 60min option.

### Example 4: Three consecutive 90min bookings then middle cancelled

```
90min(8:00-9:30) double-buf(9:30-9:40) 90min(9:40-11:10) single-buf(11:10-11:15) 90min(11:15-12:45) double-buf(12:45-12:55)
```

Middle booking (9:40-11:10) cancelled. Free window: 9:40 – 11:15 (95 min).

Preceding context: Previous booking was 90min with double buffer.

At 9:40:
- 90min: 9:40+90=11:10. Buffer: single (5min, because previous was 90min with double buffer). Buffer ends 11:15. Remaining = 0. Perfect fit ✓
- 60min: 9:40+60=10:40. Buffer: 5min. Buffer ends 10:45. Remaining: 11:15-10:45 = 30min. That's a gap > 0 but < 65min. NOT valid (creates unusable hole).
- 120min: 9:40+120=11:40 > 11:15. Doesn't fit.

Result: Only 90min can fill the gap — correct, as a 60min would create a 30min hole.

### Example 5: Two adjacent 90min cancelled (A and B from Example 4)

Both 8:00-9:30 and 9:40-11:10 cancelled. Free window: 8:00 – 11:15 (195 min).

No preceding context (start of availability block).

At 8:00:
- 120min: 8:00+120=10:00. Buffer: double (10min). Buffer ends 10:10. Remaining: 11:15-10:10 = 65min ≥ 65min ✓
- 90min: Check — window is 195min. 90+10=100, remaining 95min. 95 < 195 (3×90+3×5). But is it exactly 90+buffer? 195-100=95, which is exactly 90+5. So the second 90min would get single buffer, occupying 95min. Total: 100+95=195. Fits ✓
- 60min: 8:00+60=9:00. Buffer: 5min. Buffer ends 9:05. Remaining: 11:15-9:05 = 130min ≥ 65min ✓

At 9:05 (second start time, if 60min booked at 8:00):
- 60min: 9:05+60=10:05. Buffer: 5min. Buffer ends 10:10. Remaining: 65min ≥ 65min ✓
- 120min: 9:05+120=11:05. Buffer: 10min. Buffer ends 11:15. Remaining: 0 ✓

At 10:10 (third start time):
- 60min: 10:10+60=11:10. Buffer: 5min. Buffer ends 11:15. Remaining: 0 ✓

Result: The 195min window can hold three 60min, or 120min+60min, or 60min+120min, or two 90min — all the expected configurations.

# Open Questions

1. **Provider buffer overrides**: The current system allows per-provider buffer overrides (e.g., a provider may require 10min instead of 5min). Should the new 90min alternating double/single buffer logic also scale with provider overrides? (i.e., if override is 10min, single buffer = 10min and double buffer = 20min?) **Assumed yes — the effective buffer is already used as the base.**

2. **Room-based products**: The room availability algorithm (`room_availability_helper.cpp`) uses a completely different slot generation model (fixed 15-minute intervals, no buffers, capacity-based). The new buffer rules only apply to provider-based products. **Assumed room-based products are unchanged.**

3. **Existing bookings**: When checking the context of preceding bookings, we need to know what buffer was assigned to each existing 90min session. Should this be stored on the `bookable_service_sessions` table (e.g., a `buffer_type` column), or derived from the sequence at query time? **Storing on the session is safer and more performant — the buffer was determined at booking time and shouldn't change.**

# Implementation Plan

## Phase 1: Data Model — Store Buffer on Sessions
*Add a column to track the buffer type assigned at booking time*

### 1. Database schema update
- [ ] Add `buffer_end_us` column to `bookable_service_sessions` table (already exists per schema). Verify this is populated correctly at booking time. The difference between `buffer_end_us` and `end_time_us` tells us the buffer duration.

### 2. Verify booking flow populates buffer correctly
- [ ] Check `ServiceBookingHelper::BookService` and `CartCheckoutHelper::Checkout` — they create `bookable_service_sessions` rows. Verify `buffer_end_us` is set to `end_time_us + total_buffer`.
- [ ] If not already correct, fix to use the new context-dependent buffer for 90min bookings.

## Phase 2: Algorithm — Context-Dependent Buffer Calculation
*Modify `CalculateTotalBufferMinutes` and `ComputeSlotsForFreeWindow` to implement the new rules*

### 1. New buffer calculation function
- [ ] Create `CalculateBufferForVariant(variantDurationMinutes, baseDurationMinutes, effectiveBufferMinutes, precedingBookingDurationMinutes, precedingBookingBufferMinutes)` that implements the context-dependent buffer rules:
  - 60min → always 1× buffer
  - 120min → always 2× buffer
  - 90min → double buffer (2×) if preceding is non-90min or preceding 90min had single buffer; single buffer (1×) if preceding 90min had double buffer
  - Non-standard durations → current proportional rule (exact multiples get proportional, others get 1×)
- [ ] Tests for all buffer calculation cases

### 2. Update ComputeSlotsForFreeWindow signature
- [ ] Add `precedingBookingContext` parameter (duration and buffer of the booking that ends at the start of this free window, or nullopt if window starts at availability block start)
- [ ] Thread this context through to the buffer calculation for the first slot in each window

### 3. Update slot-to-slot context propagation
- [ ] When generating multiple start times within a window, track what the "previous" booking at each start time would be (the slot generated at the prior start time)
- [ ] For each variant at each start time, calculate its buffer based on what would precede it

### 4. Implement 90-minute availability constraints
- [ ] 90min is only offered if: (a) exactly 90min+buffer fits perfectly, OR (b) window has >= 180min+3×buffer from this start time, OR (c) it's the last slot in the window (end-of-window rule)
- [ ] Tests for all constraint cases

### 5. Implement modified end-of-window rule
- [ ] At the last valid start time, if remaining time >= 90min+buffer but < 120min+double_buffer, offer BOTH 60min and 90min (not just the longest)
- [ ] Tests for end-of-window scenarios

### 6. Extensive algorithm tests
- [ ] Port all existing `ComputeSlotsForFreeWindow` tests to the new signature
- [ ] Add tests for: alternating 90min buffers, cancelled middle 90min, cancelled pair of 90min, 90min island, end-of-window 60+90 both offered, 90min constraint rejection

## Phase 3: Integration — Update Free Window Context in ComputeAvailableSlots
*Pass preceding booking context from the main function to the slot generator*

### 1. Update ComputeAvailableSlots
- [ ] When iterating through free windows between existing bookings, extract the preceding booking's duration and buffer from the booked interval data
- [ ] Pass this context to `ComputeSlotsForFreeWindow`

### 2. Update booking creation
- [ ] Ensure `ServiceBookingHelper` and `CartCheckoutHelper` calculate and store the context-dependent buffer when creating new sessions
- [ ] Tests for booking creation with correct buffers

### 3. Integration tests
- [ ] Test: existing 90min booking → free window → new 90min gets correct buffer
- [ ] Test: existing 60min booking → free window → new 90min gets double buffer
- [ ] Test: full day with mix of bookings and cancellations

## Phase 4: Documentation Update
*Move algorithm documentation from BSF doc to this document*

### 1. Update Bookable Service Foundation.md
- [ ] Remove the scheduling algorithm section (lines 140-199 approximately)
- [ ] Replace with a reference to this document: "See [[Bookable Service Scheduling Algorithm]] for the complete scheduling algorithm description."
- [ ] Verify no other references in BSF doc that need updating
