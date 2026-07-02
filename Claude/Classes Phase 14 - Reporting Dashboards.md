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

Classes Phase 14 - Reporting Dashboards

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

**Nice-to-Have.** Three read-only admin reports that give operational visibility into the studio:

- **AR-1 Schedule grid** — facility × day × time-slot view of the master schedule with capacity + booked counts color-coded.
- **AR-2 Enrollment trends** — per-class attendance over the last N weeks; line/bar chart.
- **AR-3 Instructor load** — already covered structurally in Phase 10 (`GetInstructorLoad`); this phase adds the polished UI + filtering.

Open-seat heatmap (AR-4) and refund-effectiveness (AR-5) are Could-Have and deferred to [[Classes Phase 15 - Could Haves Batch]].

**Prerequisites:**
- Phase 1 (class_schedules + event_sessions with class linkage).
- Phase 8 (bookings populated by check-in for attendance counts).
- Phase 10 (`GetInstructorLoad` foundation).

**Outcome:**
- Three new admin endpoints + three new admin pages with charts.
- All read-only — no schema or mutation paths.

## Layering & Conventions

Lowest layer first:

1. `business_logic/scheduling/reporting_helper.h/.cpp` — pure SQL aggregation helpers.
2. `endpoints/` — three read-only admin endpoints.
3. Angular admin pages using **Angular Material + minimal CSS-only bar/line visualizations — NO chart dependency added (resolved OQ-P14-1).** `ng2-charts` (Chart.js wrapper) is a deferred follow-up only if admins later want richer interactivity.
4. Tests.

No new tables; no new table helpers (uses existing reads).

### Resolved decisions
- **OQ-P14-1 (resolved):** charts are hand-rolled CSS-only (Material) — no new chart library. AR-1 already uses a CSS grid; AR-2/AR-3 use CSS bar/line visuals.
- **OQ-P14-2 (resolved):** AR-2 weekly buckets key off `event_sessions.start_time_us` in **facility-local TZ** (`AT TIME ZONE f.timezone`), the same TZ-aware approach as Phase 9 attendance history.

## 1. AR-1 Schedule Grid

### 1.1 Business logic ✅ (2026-07-01)
- [x] `struct ScheduleGridCell` — implemented in `business_logic/scheduling/reporting_helper.{h,cpp}` (new `ReportingHelper`, the phase's aggregation home). Extended the plan struct with fields the grid UI needs and that fall out of the reuse approach: `classScheduleSlotId` (stable grid-cell identity / frontend key), `facilityId` + `facilityName` (so an all-facility grid is usable), and `sessionCount` (occurrences in range — the fill-rate denominator alongside `capacity`). Final fields: `classScheduleSlotId, classScheduleId, classId, className, dayOfWeek, startMinutes, durationMinutes, facilityId, facilityName, capacity, sessionCount, bookedCount, fillRate`.
- [x] `std::vector<ScheduleGridCell> GetScheduleGrid(Transaction&, int64_t facilityId, int64_t dateFromUs, int64_t dateToUs)`:
  - **Cell = one recurring slot** (a day/time tuple), not one whole implementation — the struct's day/start/duration are slot-level, and "facility × day × time-slot" is the natural grid unit. `classScheduleId` is carried as the parent implementation.
  - **Reuses `ClassScheduleHelper::GetDerivedSessionsForRange`** per active class (mirrors `CalendarHelper::AppendClassItems`) rather than re-deriving instance/impl windows in raw SQL. This correctly counts **all** occurrences in range (persisted + purely-derived), respects the active instance/implementation windows, and **excludes cancelled** occurrences (`ds.status == "cancelled"`). `facilityId` 0 = all facilities; otherwise filters `ds.facilityId`.
  - **`bookedCount`** sums held seats over the slot's occurrences; only materialized occurrences (`persistedEventSessionId != 0`) can carry bookings, counted via a new table-helper method (below) using `status IN ('confirmed','attended')` — so waitlisted/cancelled/no_show don't inflate fill.
  - **`fillRate` = bookedCount / (capacity × sessionCount)**, clamped to 0 when the denominator is 0. `capacity` comes from the derived session (slot `capacity_override` ?? class `default_capacity`). NOT today-forward-clamped — a report can cover past ranges (unlike the calendar).
  - Ordered by `day_of_week, start_minutes, class name, slot id`.
- [x] **New table-helper method** `TableHelpers::Bookings::CountConfirmedOrAttendedForSession(tx, eventSessionId)` (`bookings.{h,cpp}`) — the plan's `COUNT(*) … status IN ('confirmed','attended')`, kept in the table-helper layer (per `feedback_no_sql_in_business_logic`, the reporting helper composes it rather than issuing CRUD SQL itself). Tests in `bookings_test.cpp` (held-only + session-scoped; empty/unknown → 0).
- [x] **KVT converter** `ScheduleGridCellToKeyValueTable` / `…sToKeyValueTableArray` in `scheduling_key_value_table.{h,cpp}` (mirrors `InstructorLoadRow`). `fill_rate` emitted as a locale-independent fixed-point string (`snprintf %.6f`); all other fields via `StringFromInt`. Converter tests in `scheduling_key_value_table_test.cpp`.
- [x] **Tests:** `reporting_helper_test.cpp` (6) — basic cell fields + counts (held vs waitlisted, persisted + derived occurrence), facility filter, cancelled-occurrence excluded from both counts, fill-rate 0 when no bookings, slot absent when range misses its weekday, sort order (day then start). Uses far-future Monday base dates so derivation is deterministic and now-independent. Registered `reporting_helper.{h,cpp}` + test in `business_logic/scheduling/CMakeLists.txt`.

### 1.2 Endpoint ✅ (2026-07-01)
- [x] `GET /api/admin/schedule_grid?facility_id=&date_from=&date_to=` (`endpoints/admin_schedule_grid.{h,cpp}`) → `{ cells: [...] }`. Thin handler modeled on `admin_instructor_load`: strict int64 param parse (400 on missing/non-numeric/backwards range; `facility_id` optional, 0/omitted = all), permission gate **`manage_class_schedule`**, delegates to `ReportingHelper::GetScheduleGrid`, converts via the KVT array. Registered in `web_app.cpp` (include + `g_GetAdminScheduleGrid`) and `endpoints/CMakeLists.txt`.
- [x] **Endpoint test** `admin_schedule_grid_test.cpp` (4) — 403 without permission, 400 on the invalid-param matrix, cell with capacity/session_count/booked_count/fill_rate (held vs waitlisted), facility filter (only-A vs all). Finds cells by `class_schedule_slot_id` (seed-noise-robust).
- **Backend `knottyyoga_tests` to be built + run by Mason.**

### 1.3 Frontend ✅ (2026-07-01)
- [x] **`pages/manage/schedule-grid/schedule-grid.component.*` (+ `.spec.ts`, 12 cases)** at route **`/manage/schedule-grid`**. NOTE: the plan's `pages/portal/manage/reports/…` path is stale — the actual admin-report convention is `pages/manage/…` (modeled on the Phase 10 `instructor-load` component, the closest analog: date-range pickers + facility selector + `getTableRows('facilities')`).
- [x] **7-column CSS grid** (`grid-template-columns: repeat(7,1fr)`), one column per weekday (Sun–Sat), each stacked with its slot **tiles sorted by start time** (then class name). A tile shows the start time (`minutesToTimeStr`), class name, and `booked/capacity · NN%`, with a `matTooltip` (class · facility · N sessions · booked/capacity). Cells **coloured by fill rate** via `fillLevel()`: green `.fill-high` ≥80%, amber `.fill-mid` 30–80%, red `.fill-low` <30% (shared by the tiles and the **legend**). Columns precomputed in `rebuildColumns()` after each load.
- [x] **Date-range picker + facility selector** — defaults to **today → today + 7 days** ("this week"); loads on init before the admin touches the picker; re-fetches on `dateChange`/`selectionChange`. Backwards range → inline "Pick a valid date range." (no server call). Loading / empty ("No scheduled classes in this range.") / error ("Failed to load schedule grid", clears the grid) states + back-to-Manage link.
- [x] **Manage-dashboard card** "Schedule Grid" (icon `grid_view`) → `/manage/schedule-grid`; route registered in `manage.routes.ts`; dashboard spec asserts the card + route.
- **Frontend `ng test` green for the affected specs: schedule-grid 12/12; ServerAccess.mock + manage-dashboard suite 518/518.**

### 1.4 `ServerAccess` + types (AR-1 slice of §4/§5) ✅ (2026-07-01)
- [x] **`getScheduleGrid(dateFromUs, dateToUs, facilityId?) → ScheduleGridCell[]`** across interface (`types/ServerAccess.ts`) / network (`GET /api/admin/schedule_grid`, **coerces `fill_rate` string→number** since `KeyValueTableToJson` only numifies integers) / proxy (`serialize`) / mock (in-memory 3-cell seed across 2 facilities with high/mid/low fill; 401 logged-out, 400 backwards range). Mock spec: cells + facility filter, backwards-range 400, logged-out 401.
- [x] **New type `ScheduleGridCell`** in `shared/types/scheduling.types.ts` (re-exported from the ServerAccess barrel). `getInstructorLoad`/`getEnrollmentTrend` (§4) belong to AR-3/AR-2 and are untouched here (`getInstructorLoad` already existed from Phase 10).

### 1.5 Manual Test Guide — AR-1 Schedule Grid (website only)

**Navigation (verified against `mockHeaderResponse.ts` — the builder used in BOTH mock and real modes via `HeaderService.emitHeaderData` — the dashboard HTML, and `manage.routes.ts`):** the admin entry is a **top-level nav dropdown titled `Admin`** (only rendered when logged in as `isAdmin` OR holding `manage_products`). Open the **Admin** dropdown → click **Manage Products** (this is the sub-menu item; `goTo: '/manage'`). For a full admin the dropdown also shows **Manage Data** (→ `/admin`) above it; a manage-products-only user sees just **Manage Products**. `/manage` is the dashboard of cards. On that dashboard, click the **Schedule Grid** card (`mat-card-title` = **Schedule Grid**, avatar icon `grid_view`, subtitle "Weekly class schedule coloured by how full each slot is") → route **`/manage/schedule-grid`**. Backend gate: `manage_class_schedule`.

> **Correction (2026-07-02):** an earlier version of this section claimed "Manage Products" was a top-level link and "NOT a dropdown." That was wrong. It is a sub-item **inside the `Admin` dropdown**. Verified in `mockHeaderResponse.ts` lines 75–199.

**Fill-rate colours (the legend on the page):** green = ≥80% full, amber = 30–80%, red = <30%. Fill = booked ÷ (capacity × number of occurrences) over the selected range.

#### Path A — Fast path (mock mode, no backend)
The mock is always "logged in" and seeds three cells.
1. Terminal from `...\knottyyoga\ui`: `ng serve -c local` → open `http://localhost:4200`.
2. Top nav **Admin** dropdown → **Manage Products** → on the `/manage` dashboard click the **Schedule Grid** card (`/manage/schedule-grid`).
3. The board shows 7 day-columns (Sun–Sat). With the default range you'll see:
   - **Mon** column: **Knotty Yoga** at **10:00 AM** — **green** tile reading `27/15 · 90%`; below it **Partner Acrobatics** at **6:00 PM** — **amber** tile reading `10/10 · 50%`.
   - **Wed** column: **Aerial 101** at **9:00 AM** — **red** tile reading `3/12 · 13%`.
   - All other day-columns show a "—" placeholder.
4. Hover any tile → tooltip reads e.g. "Partner Acrobatics · Studio Main · 2 sessions · 10/10 booked (50%)".
5. **Date range:** the **From** and **To** date-picker fields re-fetch on change; set **To** to a date before **From** → the inline red message **"Pick a valid date range."** appears and no server call is made. (Mock returns the same three cells for any valid range — the tiles are keyed by weekday, not the actual dates.)
6. **Facility** field (dropdown): in mock mode it offers only **All facilities** (the mock seeds no facilities table), which shows all three tiles. The facility filter is exercised in Path B.

#### Path B — Full stack (real backend)
**Prereqs:** reset the DB with `knottyyoga_database_helper`, start the C++ server, `ng serve` (real backend / `development` config — **not** `-c local`), and log in as a user holding **manage_class_schedule** (an admin/manager).

**Step 1 — Create a recurring class + slot so the grid has a cell.** Top nav **Admin** dropdown → **Manage Products** → on the `/manage` dashboard click the **Class Schedules** card (`/manage/class-schedules`). The page is a 4-level nesting: **Class → Class Instances → Class Schedules → Class Schedule Slots**.
- Click **Add class** (button text `add Add class`). In the **Add class** dialog fill: **Name** = `Test Grid Class`, **Description** = `Grid demo`, **Default capacity** = `15`, **Kind** = `Recurring` (dropdown; options Recurring / Workshop / Series), **Tags** = leave empty (field only appears if tags exist), leave **Active (shown in the public catalog)** checkbox ticked. Click **Save**.
- In the class list, select **Test Grid Class**.
- Click **Add class instance**. In the **Add class instance** dialog: **Name** = `2026 Run`, **Valid from** = today's date (date picker), **Valid to (blank = perpetual)** = leave blank, **Product** = pick any existing class product (dropdown). Click **Save**.
- Under that instance click **Add class schedule**. In the **Add class schedule** dialog: **Name** = `Default`, **Priority** = `3` (dropdown), **Valid from** = today's date, **Valid to (blank = open)** = leave blank, **Copy slots from class schedule** = leave `— none —`. Click **Save**.
- Under that schedule click **Add class schedule slot**. In the **Add class schedule slot** dialog: **Day** = `Monday` (dropdown), **Start time** = `10:00 AM` (time picker), **Duration (min)** = `60`, **Facility** = pick a facility (dropdown), **Room** = pick a room (dropdown; populates from the chosen facility), **Instructor (optional)** = leave blank (autocomplete search), **Requires attending (optional)** = leave `— none —`, **Capacity override (optional)** = leave blank. Click **Save**.

**Step 2 — Open the grid.** Top nav **Admin** dropdown → **Manage Products** → dashboard → **Schedule Grid** card. Set the **From** field to the Monday of the current week (or today if today is ≤ Monday) and the **To** field a few days later so the Monday falls in the range. The **Mon** column shows a **Test Grid Class** tile at **10:00 AM** — **red**, reading `0/15 · 0%` (nothing booked yet).

**Step 3 — Add bookings to raise the fill.** Booked count = bookings on the materialized occurrence whose status is **confirmed** or **attended** (waitlisted / cancelled / no-show don't count). Get some by booking the class as a member through the public booking flow, or by marking attendees present on the staff check-in page. Re-open **Admin → Manage Products → Schedule Grid** for the same range → the tile's `booked/capacity · %` rises and its colour shifts red → amber (≥30%) → green (≥80%).

**Step 4 — Facility filter.** With classes at more than one facility, choose a facility in the **Facility** field → only that facility's tiles remain; **All facilities** shows every facility's tiles.

**Step 5 — Range behaviour.** Set **From**/**To** to a range that skips Monday → the tile disappears; set **To** before **From** → "Pick a valid date range." Cancelled occurrences are excluded from both the session and booked counts.

**Note:** the tile's `booked/capacity` text is the summed booked count over the range against the **per-occurrence** capacity, so with multiple occurrences booked can read higher than capacity (e.g. `27/15`) — the **percentage and colour** are the accurate signal (they divide by capacity × occurrences).

## 2. AR-2 Enrollment Trends

### 2.1 Business logic
- [ ] `struct EnrollmentTrendPoint { int64_t classId; std::string className; int64_t weekStartUs; int64_t attendeeCount; int64_t capacity; }`.
- [ ] `std::vector<EnrollmentTrendPoint> GetEnrollmentTrend(Transaction&, int64_t classId, int weeksBack)`. Returns weekly buckets for the last N weeks for one class (or `classId=0` for all classes overlaid). **Weeks are bucketed by `event_sessions.start_time_us` in facility-local TZ (`AT TIME ZONE f.timezone`), resolved OQ-P14-2** — same TZ-aware approach as Phase 9, so an 11pm-Pacific session on a week boundary lands in the correct local week, not the UTC one.

### 2.2 Endpoint
- [ ] `GET /api/admin/enrollment_trend?class_id=&weeks_back=`. Permission `manage_class_schedule`. Endpoint test.

### 2.3 Frontend
- [ ] `enrollment-trend.component.*/.spec.ts`.
- [ ] **CSS-only** line/bar visualization over weeks (Material, no chart dependency — resolved OQ-P14-1); tooltip per week with attendee count.

## 3. AR-3 Instructor Load (UI polish)

### 3.1 Reuse Phase 10's `GetInstructorLoad`
- [ ] No new business logic.

### 3.2 Frontend
- [ ] `instructor-load-dashboard.component.*/.spec.ts`.
- [ ] Sortable table + **CSS-only** bar visualization (Material, no chart dependency — resolved OQ-P14-1); date range picker + facility filter.

## 4. `ServerAccess`

- [ ] `getScheduleGrid(facilityId, from, to)`, `getEnrollmentTrend(classId, weeksBack)`, `getInstructorLoad(facilityId, from, to)` (reuse Phase 10's API).
- [ ] Update `ServerAccess.mock.spec.ts`.

## 5. Types

- [ ] `reports.types.ts`: `ScheduleGridCell`, `EnrollmentTrendPoint`.

## 6. Admin Metadata

- [ ] No new tables.

## 7. Tests-Required Summary

- [ ] Business-logic tests for both new aggregation helpers (capacity / booked count joins; **week bucketing in facility-local TZ — a session straddling midnight UTC lands in the correct local week, resolved OQ-P14-2**).
- [ ] Endpoint tests for both new endpoints.
- [ ] Frontend specs for all three pages, asserting the **CSS-only** Material visualizations render (no chart library — resolved OQ-P14-1).

## 8. Cross-Layer Acceptance Criteria

- [ ] Admin opens the schedule grid for "this week" → sees all class schedules at Studio Main with cells colored by fill rate.
- [ ] Admin clicks Aerial 101 → goes to enrollment trend page; sees a line chart of the last 12 weeks of attendance, hovers a Tuesday in week 5 to see attendee count.
- [ ] Admin opens instructor load → sees Sara taught 12 classes with 87 total attendees this month.

## 9. Open Questions

Both resolved (Mason, 2026-06-09) and folded into the plan above (Layering ▸ Resolved decisions + the cited sections).

- **OQ-P14-1. — RESOLVED (Mason: "that sounds fine").** No chart dependency — AR-2/AR-3 use minimal CSS-only Material bar/line visuals; `ng2-charts` deferred to a follow-up only if needed. Folded into Layering, §2.3, §3.2, §7.
- **OQ-P14-2. — RESOLVED (Mason: "go with your recommendation").** AR-2 weekly buckets key off `event_sessions.start_time_us` in facility-local TZ (`AT TIME ZONE f.timezone`), same as Phase 9. Folded into Layering, §2.1, §7.

## 10. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 14.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 8 - Staff Check-in]], [[Classes Phase 10 - Scheduling Exceptions and Shift Trades]].
- Could-have follow-ons: AR-4 / AR-5 in [[Classes Phase 15 - Could Haves Batch]].
