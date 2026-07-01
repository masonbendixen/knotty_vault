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
