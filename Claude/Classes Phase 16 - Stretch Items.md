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
- **CAL-1** Wire the public Calendar page to live class data (it's currently a hardcoded mock). Consolidates the calendar deferrals from Phases 2/10/11/13. See §4.

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
  - `indicated_count INT` — over the configured reliability window (resolved OQ-P16-1; window length is a secret, default 30 days, so the count is window-relative, not hard-coded to 30)
  - `attended_count INT` — over the same window
  - `reliability_pct INT`  (0..100)
  - `consecutive_no_shows INT`
  - `window_days INT` — the window length this rollup was computed against (so the UI can label it)
  - `last_computed_us BIGINT`
- [ ] Update via daily scheduled job.
- [ ] **`person_suspensions` table (resolved OQ-P16-3 — enforced hard block):** backs both the R-5 auto-suspend and admin conduct bans.
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL REFERENCES people(id)`
  - `source TEXT NOT NULL` — allowed-value constants `kSuspensionSource{NoShowAuto,AdminConduct}` (app-layer CHECK)
  - `reason TEXT NOT NULL DEFAULT ''`
  - `starts_us BIGINT NOT NULL`
  - `ends_us BIGINT NULL` — NULL = **indefinite** (conduct bans stay until an admin lifts them)
  - `created_by_person_id BIGINT NULL REFERENCES people(id)` — the admin for conduct bans; NULL for the auto job
  - `lifted_us BIGINT NULL`, `lifted_by_person_id BIGINT NULL REFERENCES people(id)`
  - `created_us BIGINT NOT NULL`
  - Partial index for the active-suspension lookup (`WHERE lifted_us IS NULL`).

### 1.4 Business logic
- [ ] **Configurable window (resolved OQ-P16-1):** the reliability window length comes from a `reliability_window_days` config secret, **default 30** (seeded in `create_database.cpp`, read via `SecretsHelper`). `ComputeAllUsersAndRollup` resolves `fromUs = asOfUs - reliability_window_days × 86400_000_000` and stamps `window_days` on each rollup.
- [ ] `ReliabilityHelper::ComputeReliabilityForPerson(personId, fromUs, toUs)`:
  1. Pull template entries → derive expected sessions in the window (excluding `attending=false` exceptions; including `attending=true` exceptions).
  2. Cross-reference against `bookings WHERE person_id=? AND checked_in_us BETWEEN fromUs AND toUs`.
  3. Compute `attended / indicated` → reliability_pct.
- [ ] `ComputeAllUsersAndRollup(asOfUs, SecretsHelper&)` — scheduled job entry point; resolves the window from the secret.

### 1.5 R-3 Soft warning
- [ ] When `reliability_pct < soft_warning_threshold` (configurable secret, default 50%), send a "We've noticed..." email at most once per 30 days per user.

### 1.6 R-5 Consecutive no-show auto-suspend (hard block — resolved OQ-P16-3)
- [ ] Track consecutive no-shows in the rollup. If `consecutive_no_shows >= secret consecutive_no_show_cap` (default 3), the daily job inserts a `person_suspensions` row with `source='no_show_auto'`, `ends_us = now + suspend_days × 86400_000_000` (`suspend_days` secret, default 14). This is a **hard block**, not a soft warning (Mason: hard block, since enforcement matters for conduct cases too). Idempotent: don't stack a second auto-suspension while one is already active.

### 1.6b Admin conduct suspensions / bans (resolved OQ-P16-3)
- [ ] Admins can impose a manual suspension/ban (`source='admin_conduct'`, `reason` required, `ends_us` optional → NULL = **indefinite**) for conduct issues (e.g., sexually inappropriate behavior) that "really need to be enforced." Admins can also lift any active suspension (`lifted_us`/`lifted_by_person_id`).
- [ ] `SuspensionHelper` (`business_logic/scheduling/suspension_helper.h/.cpp/_test.cpp`): `SuspendPerson(tx, personId, source, reason, endsUs?, byPersonId?)`, `LiftSuspension(tx, suspensionId, byPersonId)`, `GetActiveSuspension(tx, personId, nowUs)` → optional row, `IsSuspended(tx, personId, nowUs)`. SQL lives in a `TableHelpers::PersonSuspensions` wrapper (no SQL in business logic).
- [ ] Endpoints (bespoke, permission `admin`): `POST /api/admin/person/<id>/suspend` `{ reason, ends_us? }`, `POST /api/admin/person/<id>/lift_suspension/<suspensionId>`, `GET /api/admin/person/<id>/suspensions`. Endpoint tests (403 / 400-missing-reason / 200+persist / lift).

### 1.6c Enforcement points (hard block — resolved OQ-P16-3)
- [ ] A hard block means an active suspension stops the user from participating, not just from template-claiming. At each of these, call `SuspensionHelper::IsSuspended(personId, now)` first and reject with `USER_SUSPENDED` (surfacing the reason where appropriate):
  - **Template-claim** new attendance-template entries (the original R-5 surface).
  - **Booking** — `BookingHelper::BookEvent` and the series-booking flow.
  - **Check-in** — `ClassCheckinHelper::CheckIn` (Phase 8), so a banned person can't be checked in either; staff sees the ban reason.
- [ ] "Just show up at the door" for membership-included classes is **not** an escape hatch for a conduct ban — that's exactly the case Mason wants enforced, so check-in is gated too. (A no-show auto-suspension is the milder, time-boxed case; same enforcement path, it just expires on `ends_us`.)

### 1.7 Frontend
- [ ] Admin user-detail page: reliability score badge + history sparkline.
- [ ] Admin user-detail page: a **Suspension** panel (resolved OQ-P16-3) — shows active suspension (source + reason + ends/indefinite), a "Suspend / ban" action (reason required, optional end date → blank = indefinite), and a "Lift" action; plus suspension history. Wired to the §1.6b endpoints.
- [ ] User homepage: gentle nudge banner if reliability drops below 60%; a suspended user sees a clear "your account is suspended" notice with the reason.
- [ ] Booking / check-in surfaces show the `USER_SUSPENDED` rejection cleanly (the user can't book; staff sees the ban reason at check-in).
- [ ] Specs for the suspension panel + the suspended-user booking/check-in block.

### 1.8 Scheduled job
- [ ] Daily 04:00 local: `POST /api/admin/compute_reliability_rollups`. Idempotent.

### 1.9 Tests
- [ ] Helper tests for the rollup math; **configurable window (resolved OQ-P16-1): changing `reliability_window_days` changes the counts and the stamped `window_days`**; mail-helper assertion for the once-per-30d soft warning.
- [ ] Suspension tests (resolved OQ-P16-3): R-5 auto-suspend inserts a time-boxed `no_show_auto` row at the cap and doesn't stack; admin conduct ban with NULL `ends_us` is indefinite until lifted; `IsSuspended` true within window / false after `ends_us` / false once lifted; **enforcement** — a suspended user is hard-blocked at template-claim, `BookEvent`, AND `CheckIn` with `USER_SUSPENDED`; lifting restores access. Endpoint tests for suspend / lift / list.

## 2. Specialty Instructor Payroll (PR-1..PR-4 + SI-5)

### 2.1 Database
- [ ] `payroll_periods` table — defines a payroll window (start_us, end_us, status). **Fully flexible: admin can create any window (resolved OQ-P16-2)** — weekly, bi-weekly, one-off — and **monthly is just the default** the create form pre-fills.
- [ ] `payroll_entries` table — one row per (instructor, payroll_period, event_session) with computed pay snapshot.

### 2.2 Business logic
- [ ] `PayrollHelper::ComputePeriod(periodId)` — iterates sessions in the window; calls `SpecialtyCostHelper::ComputeInstructorPayCents` for each; inserts `payroll_entries`.
- [ ] `PayrollHelper::ExportCsv(periodId)` — flattens to a CSV blob.

### 2.3 Endpoints
- [ ] `POST /api/admin/payroll/period` — create a period with an arbitrary `{ start_us, end_us }` (resolved OQ-P16-2); when omitted, default to the current calendar month. The admin payroll page (§2.4) pre-fills the current month but lets the admin pick any window.
- [ ] `POST /api/admin/payroll/period/<id>/compute` — run the computation.
- [ ] `GET /api/admin/payroll/period/<id>/csv` — returns CSV.

### 2.4 Frontend
- [ ] Admin payroll page: list of periods, a "New period" action with start/end date pickers **pre-filled to the current month but freely editable to any window** (resolved OQ-P16-2), plus "Compute" and "Download CSV" actions.

### 2.5 Tests
- [ ] Helper + endpoint + CSV golden-text comparison; incl. an **arbitrary (non-monthly) window** period computing only the sessions inside it (resolved OQ-P16-2).

## 3. AR-6 Series Min-Attendees Risk Dashboard

### 3.1 Business logic
- [ ] `SeriesRiskHelper::GetSeriesUnderMinByThreshold(asOfUs)` — returns series runs (`class_instances`) within their `class_series_instances.min_by_us` window whose confirmed count is below `class_series_instances.min_attendees`. Includes days-until-deadline. (Per the schedule redesign, series fields live on the `class_series_instances` augmentation, not on `class_schedules`.)

### 3.2 Endpoint
- [ ] `GET /api/admin/series_risk`. Permission `manage_class_schedule`.

### 3.3 Frontend
- [ ] Admin dashboard widget: list of at-risk series with progress bar (confirmed / min) and days remaining.

### 3.4 Tests
- [ ] Helper + endpoint + frontend spec.

## 4. Calendar ↔ live class data wiring (CAL-1)

### 4.1 Background
The public Calendar page (`ui/src/app/pages/calendar/`) is **entirely mock-driven**: `CalendarService._getCalendarEvents()` returns `mockCalendarResponse()` (hardcoded `EXAMPLE_CALENDAR_EVENTS`) behind a `// TODO replace with API call for production environment`, and `CalendarEvent` carries only `{ id, title, startTime, endTime, location, color? }` — no class linkage, price, membership, instructor, or tag data. Nothing on the calendar reflects real schedules. This single gap is why several phases deferred calendar-facing work; consolidate them here.

**Deferrals this unblocks (all currently note "calendar is mock-only"):**
- **Phase 2 §6.2** — session chip labels "Included / Tier-priced / Members only".
- **Phase 13 §2.5 + §6.5** — tag chip *color* on calendar events (the `color?` hook + `event-chip-tagged` style already exist on `CalendarEvent` / `calendar-event.component`; only the data wiring is missing). Closes the unchecked Phase 13 §5 acceptance boxes (lines ~204–205: "Calendar shows yellow chip for yoga…", "…lowest-sort_order tag wins").
- **Phase 10** — substitute/effective instructor display on the calendar.
- **Phase 11** — future-session "Sign-ups open on …" chip on the calendar.

### 4.2 Backend
- [ ] Reuse the existing derived-session machinery rather than inventing a new one. `GetDerivedSessionsForRange(... fromUs, toUs)` (Class Schedule Implementations Redesign / Phase 1) already walks dates, resolves the active instance + impl per day, expands slots, and left-joins persisted `event_sessions` for cancellations/subs/notes. Confirm/extend a calendar-range read endpoint (e.g. `GET /api/calendar/sessions?from_us=&to_us=[&facility_id=]`) that returns, per occurrence: class id/name, start/end, facility/room, effective instructor (+ `has_substitute`), price/membership flags (as `visible_event_sessions` already resolves), and the class's **`chip_color`** (first-tag color, OQ-P13-1) + tags.
- [ ] Permission/visibility: honor per-viewer visibility + signup-window state (so the calendar can render "Sign-ups open on …" like the catalog).

### 4.3 Frontend
- [ ] Replace `CalendarService._getCalendarEvents()` with a real `ServerAccess` call; map each returned occurrence into `CalendarEvent`, setting `color = occurrence.chip_color`. Extend `CalendarEvent` with the class/price/instructor fields the chips need.
- [ ] Render: tag color accent (already wired via `[style.border-left-color]`), the membership/price label (Phase 2 §6.2), substitute instructor (Phase 10), and the signup-window chip (Phase 11).
- [ ] Remove `mockCalendarResponse` from the production path (keep a mock for `-c local`).

### 4.4 Tests
- [ ] Backend: calendar-range endpoint (date expansion, cancellation/sub left-join, per-viewer price/visibility, `chip_color`). Frontend: `CalendarService` maps occurrences → events (incl. `color`), and `calendar-event` renders the color/label/instructor/signup chips. The Phase 13 §5 calendar acceptance assertions move here.

## 5. Promote individual stretch items as needed

When a stretch item is ready to ship, copy its sub-section into its own dedicated `Classes Phase ... .md` doc using the template, expand with more concrete schema / endpoint / UI sketches per the patterns in Phases 1-14, and remove it here.

## 6. Open Questions

All three resolved (Mason, 2026-06-09) and folded into the plan above (the cited item sections).

- **OQ-P16-1. — RESOLVED (Mason: "configurable, default 30 days").** The reliability window is a `reliability_window_days` secret, default 30; rollup counts are window-relative and stamp `window_days`. Folded into §1.3, §1.4, §1.9.
- **OQ-P16-2. — RESOLVED (Mason: "any window, monthly is a fine default").** `payroll_periods` accept arbitrary `{start_us, end_us}`; the create form pre-fills the current month but allows any window. Folded into §2.1, §2.3, §2.4, §2.5.
- **OQ-P16-3. — RESOLVED (Mason departs from the "defer" recommendation): enforced HARD BLOCK.** Mason wants suspensions enforced — including admin-imposed conduct bans (e.g., sexually inappropriate behavior), which must be indefinite-until-lifted. Added a `person_suspensions` table + `SuspensionHelper`, made R-5 auto-suspend a hard block, added admin suspend/lift endpoints + UI, and gate **template-claim, booking, AND check-in** on `IsSuspended` (so "just show up at the door" can't bypass a ban). Folded into §1.3, §1.6, §1.6b, §1.6c, §1.7, §1.9.

## 7. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 16.
- Builds on: [[Classes Phase 5 - Attendance Templates]] (indicated attendance source), [[Classes Phase 8 - Staff Check-in]] (actual attendance source), [[Classes Phase 12 - Specialty Instructor Cost]] (payroll cost snapshots), [[Classes Phase 7 - Class Series and Workshops]] (series-risk source).
- **CAL-1 (§4)** consolidates the calendar deferrals from [[Classes Phase 2 - Membership-Gated Drop-In]] §6.2, [[Classes Phase 13 - Tags Filters Favorites]] §2.5/§6.5, and the calendar-facing bits of [[Classes Phase 10 - Scheduling Exceptions and Shift Trades]] / [[Classes Phase 11 - Signup Windows and Reminders]]. Backend reuse: `GetDerivedSessionsForRange` from [[Class Schedule Implementations Redesign]].
