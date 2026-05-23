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
3. Angular admin pages using Material + the existing chart library (or chart.js if not yet present — flagged in open questions).
4. Tests.

No new tables; no new table helpers (uses existing reads).

## 1. AR-1 Schedule Grid

### 1.1 Business logic
- [ ] `struct ScheduleGridCell { int dayOfWeek; int startMinutes; int durationMinutes; int64_t classScheduleId; std::string className; int64_t capacity; int64_t bookedCount; double fillRate; }`.
- [ ] `std::vector<ScheduleGridCell> GetScheduleGrid(Transaction&, int64_t facilityId, int64_t dateFromUs, int64_t dateToUs)`:
  - For each active `class_schedule` at the facility:
    - For each materialized session in the date range: compute `bookedCount = SELECT COUNT(*) FROM bookings WHERE event_session_id=? AND status IN ('confirmed', 'attended')`.
    - Average / sum to per-schedule cell.
  - Return.

### 1.2 Endpoint
- [ ] `GET /api/admin/schedule_grid?facility_id=&date_from=&date_to=`. Permission `manage_class_schedule`. Endpoint test.

### 1.3 Frontend
- [ ] `ui/src/app/pages/portal/manage/reports/schedule-grid/schedule-grid.component.*/.spec.ts`.
- [ ] CSS grid layout (7 columns × N rows of time slots) with cells colored by fill-rate (green ≥80%, amber 30-80%, red <30%).
- [ ] Date-range picker; facility selector.

## 2. AR-2 Enrollment Trends

### 2.1 Business logic
- [ ] `struct EnrollmentTrendPoint { int64_t classId; std::string className; int64_t weekStartUs; int64_t attendeeCount; int64_t capacity; }`.
- [ ] `std::vector<EnrollmentTrendPoint> GetEnrollmentTrend(Transaction&, int64_t classId, int weeksBack)`. Returns weekly buckets for the last N weeks for one class (or `classId=0` for all classes overlaid).

### 2.2 Endpoint
- [ ] `GET /api/admin/enrollment_trend?class_id=&weeks_back=`. Permission `manage_class_schedule`. Endpoint test.

### 2.3 Frontend
- [ ] `enrollment-trend.component.*/.spec.ts`.
- [ ] Line chart over weeks; tooltip per week with attendee count.

## 3. AR-3 Instructor Load (UI polish)

### 3.1 Reuse Phase 10's `GetInstructorLoad`
- [ ] No new business logic.

### 3.2 Frontend
- [ ] `instructor-load-dashboard.component.*/.spec.ts`.
- [ ] Sortable table + bar chart visualization; date range picker + facility filter.

## 4. `ServerAccess`

- [ ] `getScheduleGrid(facilityId, from, to)`, `getEnrollmentTrend(classId, weeksBack)`, `getInstructorLoad(facilityId, from, to)` (reuse Phase 10's API).
- [ ] Update `ServerAccess.mock.spec.ts`.

## 5. Types

- [ ] `reports.types.ts`: `ScheduleGridCell`, `EnrollmentTrendPoint`.

## 6. Admin Metadata

- [ ] No new tables.

## 7. Tests-Required Summary

- [ ] Business-logic tests for both new aggregation helpers (correctness of week bucketing, capacity / booked count joins).
- [ ] Endpoint tests for both new endpoints.
- [ ] Frontend specs for all three pages including chart-component-mock-render.

## 8. Cross-Layer Acceptance Criteria

- [ ] Admin opens the schedule grid for "this week" → sees all class schedules at Studio Main with cells colored by fill rate.
- [ ] Admin clicks Aerial 101 → goes to enrollment trend page; sees a line chart of the last 12 weeks of attendance, hovers a Tuesday in week 5 to see attendee count.
- [ ] Admin opens instructor load → sees Sara taught 12 classes with 87 total attendees this month.

## 9. Open Questions

- **OQ-P14-1.** Which chart library? Recommend: use Angular Material + a minimal CSS-only bar/line visualization for AR-2/AR-3 to avoid adding a chart dependency. If admin wants richer interactivity, add `ng2-charts` (Chart.js wrapper) in a follow-up.
- **OQ-P14-2.** AR-2 weekly bucketing: bucket by `event_sessions.start_time_us` in facility-local TZ? Recommend yes; same TZ-aware approach as Phase 9 attendance history.

## 10. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 14.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 8 - Staff Check-in]], [[Classes Phase 10 - Scheduling Exceptions and Shift Trades]].
- Could-have follow-ons: AR-4 / AR-5 in [[Classes Phase 15 - Could Haves Batch]].
