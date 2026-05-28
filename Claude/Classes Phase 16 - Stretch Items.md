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

Classes Phase 16 - Stretch Items

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

**Stretch / long-term.** Items that are valuable but not differentiators today. Each is an independent feature; promote individual items back to earlier phases if they become hot.

Items from parent §5.5:
- **R-1..R-4** Reliability tracking ("indicated attending but didn't attend" rolling rate + soft warning + admin reliability score; harder penalties stay manual).
- **R-5** Consecutive no-show auto-suspend.
- **PR-1..PR-4** Specialty instructor full payroll model (rates, snapshots, payroll report, CSV export). Builds on [[Classes Phase 12 - Specialty Instructor Cost]].
- **SI-5** Payroll CSV export — subset of PR.
- **AR-6** Series min-attendees risk dashboard.

## Layering & Conventions

Lowest layer first per item; full test coverage.

## 1. Reliability Tracking (R-1..R-5)

### 1.1 Background
Per parent §2.17: track per-user "indicated attending but didn't attend" rate; surface in admin user-detail; configurable soft-warning threshold; harder penalties manual. P-5 guarantees attendance records are trustworthy (no self-attribution).

### 1.2 Data sources
- **Indicated attendance** = attendance template entries + per-instance "attending=true" exceptions.
- **Actual attendance** = `bookings.checked_in_us IS NOT NULL`.
- **Implicit no-show** for membership-included recurring class: the user is "indicated" (template or one-off add) but no `booking` row was created at check-in time.

### 1.3 Database
- [ ] Optional materialized rollup table `person_reliability_rollups`:
  - `person_id BIGINT PK REFERENCES people(id)`
  - `indicated_count_last_30d INT`
  - `attended_count_last_30d INT`
  - `reliability_pct INT`  (0..100)
  - `last_computed_us BIGINT`
- [ ] Update via daily scheduled job.

### 1.4 Business logic
- [ ] `ReliabilityHelper::ComputeReliabilityForPerson(personId, fromUs, toUs)`:
  1. Pull template entries → derive expected sessions in the window (excluding `attending=false` exceptions; including `attending=true` exceptions).
  2. Cross-reference against `bookings WHERE person_id=? AND checked_in_us BETWEEN fromUs AND toUs`.
  3. Compute `attended / indicated` → reliability_pct.
- [ ] `ComputeAllUsersAndRollup(asOfUs)` — scheduled job entry point.

### 1.5 R-3 Soft warning
- [ ] When `reliability_pct < soft_warning_threshold` (configurable secret, default 50%), send a "We've noticed..." email at most once per 30 days per user.

### 1.6 R-5 Consecutive no-show auto-suspend
- [ ] Track consecutive no-shows in the rollup. If `consecutive_no_shows >= secret consecutive_no_show_cap` (default 3), suspend the user's ability to template-claim new entries for `suspend_days` (default 14). Membership-included class access via "just show up at the door" is unaffected.

### 1.7 Frontend
- [ ] Admin user-detail page: reliability score badge + history sparkline.
- [ ] User homepage: gentle nudge banner if reliability drops below 60%.

### 1.8 Scheduled job
- [ ] Daily 04:00 local: `POST /api/admin/compute_reliability_rollups`. Idempotent.

### 1.9 Tests
- [ ] Helper tests for the rollup math; mail-helper assertion for the once-per-30d soft warning; suspension and lift via scheduled job.

## 2. Specialty Instructor Payroll (PR-1..PR-4 + SI-5)

### 2.1 Database
- [ ] `payroll_periods` table — defines a payroll window (start_us, end_us, status). Admin creates one per period (e.g. monthly).
- [ ] `payroll_entries` table — one row per (instructor, payroll_period, event_session) with computed pay snapshot.

### 2.2 Business logic
- [ ] `PayrollHelper::ComputePeriod(periodId)` — iterates sessions in the window; calls `SpecialtyCostHelper::ComputeInstructorPayCents` for each; inserts `payroll_entries`.
- [ ] `PayrollHelper::ExportCsv(periodId)` — flattens to a CSV blob.

### 2.3 Endpoints
- [ ] `POST /api/admin/payroll/period` — create a period.
- [ ] `POST /api/admin/payroll/period/<id>/compute` — run the computation.
- [ ] `GET /api/admin/payroll/period/<id>/csv` — returns CSV.

### 2.4 Frontend
- [ ] Admin payroll page: list of periods, "Compute", "Download CSV" actions.

### 2.5 Tests
- [ ] Helper + endpoint + CSV golden-text comparison.

## 3. AR-6 Series Min-Attendees Risk Dashboard

### 3.1 Business logic
- [ ] `SeriesRiskHelper::GetSeriesUnderMinByThreshold(asOfUs)` — returns series runs (`class_instances`) within their `class_series_instances.min_by_us` window whose confirmed count is below `class_series_instances.min_attendees`. Includes days-until-deadline. (Per the schedule redesign, series fields live on the `class_series_instances` augmentation, not on `class_schedules`.)

### 3.2 Endpoint
- [ ] `GET /api/admin/series_risk`. Permission `manage_class_schedule`.

### 3.3 Frontend
- [ ] Admin dashboard widget: list of at-risk series with progress bar (confirmed / min) and days remaining.

### 3.4 Tests
- [ ] Helper + endpoint + frontend spec.

## 4. Promote individual stretch items as needed

When a stretch item is ready to ship, copy its sub-section into its own dedicated `Classes Phase ... .md` doc using the template, expand with more concrete schema / endpoint / UI sketches per the patterns in Phases 1-14, and remove it here.

## 5. Open Questions

- **OQ-P16-1.** Reliability score window — 30 days vs 60 days vs configurable? Recommended: 30 days, configurable via secret.
- **OQ-P16-2.** Payroll period granularity — monthly default, but admin can create any window? Recommended: yes, fully flexible.
- **OQ-P16-3.** What's the studio's policy on suspended users? Hard block their template-add, soft warn, or just surface in admin? Defer — make it admin-visible first, then layer in policy once Mason has data on no-show rates.

## 6. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 16.
- Builds on: [[Classes Phase 5 - Attendance Templates]] (indicated attendance source), [[Classes Phase 8 - Staff Check-in]] (actual attendance source), [[Classes Phase 12 - Specialty Instructor Cost]] (payroll cost snapshots), [[Classes Phase 7 - Class Series and Workshops]] (series-risk source).
