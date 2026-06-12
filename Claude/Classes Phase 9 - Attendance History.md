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

Classes Phase 9 - Attendance History

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

> **Access redesign note (2026-05-31, [[Permission-based class access redesign]] §4.5):** The attendance history this phase surfaces is the same data **SL-10** counts for the attendance-threshold permission grant (Phase 3's monthly job reads these facts). No access-model change; note the dependency.

## Phase Summary

**Must-have.** User can view paginated past attendances in the user portal, filterable by year / month / class / instructor. Each row shows class, date/time, facility, room, instructor, status. CSV export is Nice-to-Have (out of scope for the Must portion).

**Prerequisites:**
- Phase 1 (event_sessions has the denormalized `class_id` + `class_schedule_slot_id` + `occurrence_date_us`).
- Phase 8 (bookings populate with `checked_in_us` / `status='attended'` for membership-included classes).

### Class Schedule Redesign Impact (2026-05-28) — see [[Class Schedule Implementations Redesign]]
Minimal. Attendance history reads **only persisted `event_sessions` rows** — and under the lazy model, a row exists precisely when something was recorded against it (check-in, etc.), which is exactly the set of attended occurrences we want. No derived-session computation is needed here. Filters key off the denormalized `event_sessions.class_id` (not a `class_schedule_id`, which no longer lives on `event_sessions`).

**Outcome:**
- `GET /api/me/attendance_history` with year / month / class / instructor filters + pagination.
- `/my/account/attendance-history` Angular page with filter UI + paginator + table.
- All read-only; no schema work, no business-logic write paths.

## Layering & Conventions

Lowest layer first:

1. `business_logic/scheduling/` — `AttendanceHistoryHelper`.
2. `endpoints/` — one new endpoint.
3. Angular UI.
4. Tests.

No new DB tables; no new table helpers (uses existing `bookings`, `event_sessions`, `classes`, `event_session_staffing`, `people`, `location_rooms`, `facilities` reads).

## 1. Pre-Coding Design Decisions

### 1.1 Locked-in
- [x] Status filter defaults to `'attended'` only (members usually want to see what they've done; `no_show` and `cancelled` rows are noise by default; UI offers a "Show all" toggle).
- [x] Pagination defaults: 25 per page; client supplies `offset` and `limit`.
- [x] Sort: most recent first.
- [x] **Count distinct bookings, not distinct dates (resolved OQ-P9-1).** A member who attended 5 classes on one day sees 5 rows / counts as 5 — each is a separate experience. This is the natural per-`bookings`-row model already in §2 (one `AttendanceHistoryRow` per `bookings` row; `GetTotalCount` counts bookings), so no de-duplication by date anywhere.
- [x] **Year/month dropdown range = earliest attendance → today, computed server-side (resolved OQ-P9-2).** The endpoint returns the user's earliest attendance moment in the metadata block alongside `distinct_class_ids`; the UI builds the year list from that to the current year (no fixed "last 5 years" window).

### 1.2 Filter semantics
- [x] `year` and `month` filter the session's date in **facility-local TZ** (NOT UTC). Justification: a 11pm Pacific session on March 31 reads as March, not April UTC.
- [x] `class_id` filters by `event_sessions.class_id` (denormalized convenience set when the occurrence row is ensured in Phase 1).
- [x] `instructor_id` filters via `event_session_staffing` rows with that `person_id` and role in (`'instructor'`, `'lead_instructor'`).

## 2. Business Logic — `AttendanceHistoryHelper` ✅ DONE

Files: `business_logic/scheduling/attendance_history_helper.h/.cpp/_test.cpp` (registered in the scheduling CMakeLists).

**Reconciliations vs the plan's original prose (2026-06-11):**
- **Year/month extraction is `AT TIME ZONE 'UTC'`, NOT `AT TIME ZONE f.timezone`** (§2.4's sketch). Class-session `start_time_us` is the facility's **wall-clock encoded as UTC** (occurrence UTC-midnight + slot minutes — the Phase 8 timezone saga), so reading the encoded value as UTC *is* the §1.2 facility-local calendar. Applying the facility timezone again would double-convert and shift early-morning classes into the previous local day. The 11pm-March-31-Pacific acceptance case still reads as March — pinned by test.
- **Instructor roles are `('instructor','substitute')`** — the plan's `'lead_instructor'` does not exist in this schema (staffing_role enum: instructor / assistant / substitute; a substitute taught the session, an assistant didn't headline it).
- **Instructor matching/naming is staffing ∪ slot**: membership-class sessions created by `EnsureSessionExists` carry their instructor on `class_schedule_slots.instructor_person_id` and have NO staffing rows, so both paths are honored (UNION dedupes a person reachable via both).
- **History statuses are exactly `('attended'[, 'no_show'][, 'cancelled'])`** — the §2.4 sketch's NOT-exclusions would have leaked `'confirmed'` (future/unfinalized) bookings into history; they never appear, regardless of flags.
- `classPhotoUrl` is declared but empty, the `ClassCatalogEntry::photoUrl` convention (Phase 13 / image wiring surfaces it).
- Defensive paging: `limit < 1` → 25, capped at `kMaxPageSize = 200`; negative `offset` → 0; ties in the sort broken by `b.id DESC` so pagination is stable.

### 2.1 Filter struct
```cpp
struct AttendanceHistoryFilter {
    std::optional<int> year;            // e.g. 2026
    std::optional<int> month;           // 1..12
    std::optional<int64_t> classId;
    std::optional<int64_t> instructorPersonId;
    bool includeNoShow = false;
    bool includeCancelled = false;
    int64_t offset = 0;
    int64_t limit = 25;
};
```

### 2.2 Row struct
```cpp
struct AttendanceHistoryRow {
    int64_t bookingId;
    int64_t eventSessionId;
    int64_t classId;
    std::string className;
    std::string classPhotoUrl;
    int64_t startTimeUs;
    int64_t endTimeUs;
    int64_t facilityId;
    std::string facilityName;
    std::string facilityIanaTz;
    int64_t locationRoomId;
    std::string roomName;
    std::vector<std::string> instructorNames;
    std::string status; // 'attended' | 'no_show' | 'cancelled'
    bool isWalkin;
};
```

### 2.3 Methods ✅
- [x] `int64_t GetTotalCount(Transaction&, int64_t personId, const AttendanceHistoryFilter&)` — count query. Counts `bookings` rows (distinct bookings, **not** distinct dates — resolved OQ-P9-1); 5 same-day attendances count as 5.
- [x] `std::vector<AttendanceHistoryRow> GetHistory(Transaction&, int64_t personId, const AttendanceHistoryFilter&)` — the paginated read (one row per `bookings` row).
- [x] `std::vector<int64_t> GetDistinctInstructorIdsForPerson(Transaction&, int64_t personId)` — feeds the filter dropdown (slot ∪ staffing instructor/substitute, across all history statuses).
- [x] `std::vector<int64_t> GetDistinctClassIdsForPerson(Transaction&, int64_t personId)` — feeds the filter dropdown.
- [x] `std::optional<int64_t> GetEarliestAttendanceUs(Transaction&, int64_t personId)` — `MIN(event_sessions.start_time_us)` across the person's history bookings (all history statuses, so the year range covers what the toggles can reveal — resolved OQ-P9-2). `nullopt` when no history. NOTE for §4: the value is wall-clock-encoded, so the UI reads its year **as UTC** (no facility-TZ conversion).

### 2.4 SQL strategy
Single query with conditional WHERE clauses. Joins:
```
bookings b
  JOIN event_sessions es ON es.id = b.event_session_id
  JOIN classes c ON c.id = es.class_id
  JOIN facilities f ON f.id = es.facility_id
  LEFT JOIN location_rooms r ON r.id = es.location_room_id
WHERE b.person_id = :person_id
  AND (:include_cancelled OR b.status != 'cancelled')
  AND (:include_no_show OR b.status != 'no_show')
  AND (:year IS NULL OR EXTRACT(YEAR FROM session_start_at_facility_tz) = :year)
  AND (:month IS NULL OR EXTRACT(MONTH FROM session_start_at_facility_tz) = :month)
  AND (:class_id IS NULL OR es.class_id = :class_id)
  AND (:instructor_id IS NULL OR EXISTS (SELECT 1 FROM event_session_staffing s WHERE s.event_session_id = es.id AND s.person_id = :instructor_id AND s.role IN ('instructor','lead_instructor')))
ORDER BY es.start_time_us DESC
LIMIT :limit OFFSET :offset
```
- [x] ~~Facility-TZ extraction with `AT TIME ZONE f.timezone`~~ — **superseded**: extraction is `AT TIME ZONE 'UTC'` because session times are wall-clock-encoded (see the §2 reconciliation note). The §1.2 semantic (facility-local calendar) is what's delivered; the mechanism differs from the sketch.
- [x] Instructor names assembled in one batch UNION query (staffing ∪ slot, instructor-role-filtered, deduped) after the page query — no N+1.

### 2.5 KeyValueTable conversion ✅
- [x] `AttendanceHistoryRowToKeyValueTable` + `AttendanceHistoryRowsToKeyValueTableArray` in `scheduling_key_value_table.h/.cpp`. `instructor_names` is pipe-delimited (the `TodayClassEntry` convention); tz emitted as `facility_timezone` (the `EventSessionInfo` convention). 3 new KVT tests (full-field, empty-names + non-walk-in, array).

### 2.6 Tests ✅
- [x] `attendance_history_helper_test.cpp` — 9 tests, all plan cases plus the reconciliation behaviors:
  - `EmptyForPersonWithNoBookings` — history/count/distinct-lists/earliest all empty for a fresh person.
  - `RowsHydrateAllFields` — every row field incl. slot-instructor name, room/facility names, walk-in flag, empty photo URL; another person sees nothing.
  - `SortsMostRecentFirstAndPaginates` — 15 rows: page 1 (10, descending), page 2 (5, no overlap, ends at the oldest), count 15, and the defensive limit/offset clamps.
  - `YearMonthFilterUsesFacilityLocalCalendar` — 11pm March-31 Pacific class is **March** (month=3 matches, month=4 doesn't; year 2025 doesn't; month-only filter works) — the straddling-midnight-UTC case under the wall-clock encoding.
  - `ClassFilterExcludesOtherClasses` (+ distinct class ids).
  - `InstructorFilterMatchesSlotAndStaffing` — slot-instructor session and staffing-instructor session each match their teacher; an **assistant never matches**; names resolve per-path; distinct ids are the union.
  - `InstructorNamesDedupeSlotAndStaffing` — same person via both paths → named once.
  - `StatusFlagsControlNoShowAndCancelled` — default attended-only; flags add no_show then cancelled; **`confirmed` never appears** even with all flags on.
  - `FiveSameDayAttendancesCountAsFive` (OQ-P9-1).
  - `EarliestAttendanceIsMinStartAcrossHistory` (OQ-P9-2) — earliest is a no_show (full history statuses); a confirmed-only person → nullopt.

## 3. Endpoints ✅ DONE

### 3.1 User endpoint ✅
- [x] `endpoints/get_my_attendance_history.h/cpp` + test (registered in `web_app.cpp` include + reference var, and `endpoints/CMakeLists.txt`):
  - `GET /api/me/attendance_history?year=&month=&class_id=&instructor_id=&include_no_show=&include_cancelled=&offset=&limit=`.
  - Permission: logged-in session (401 anonymous).
  - Returns `{ total_count, rows: [...], distinct_class_ids, distinct_instructor_ids, earliest_attendance_us }` — `earliest_attendance_us` is JSON **null** when no history (drives the year dropdown, OQ-P9-2).
  - Validation (400 `ValidationError`): any non-integer numeric param; `month` outside 1–12. Bool flags accept `true`/`1`. `offset`/`limit` pass through to the helper's defensive clamps (§2).
  - Test (`get_my_attendance_history_test.cpp`, 8 cases): 401 anonymous; rows+metadata (sort order, instructor name on the row, both distinct lists, exact earliest); empty history → null earliest + empty arrays; year/month/class/instructor filters; include flags reveal no_show then cancelled; pagination across pages with most-recent-first; **invalid-param matrix** (month 13/0/abc, year `20x6`, class_id/instructor_id/offset/limit non-numeric → 400, sane request still 200 after); cross-user isolation (another member's bookings never leak). Query params set via `crow::query_string` per the memory rule; every request flushes `ThreadPool` before the next DB read.

## 4. Frontend ✅ DONE

**Reconciliations vs the plan's original prose (2026-06-11):**
- **No `MatTable`/`MatPaginator`** — neither module exists anywhere in the app (not in `MatImports`); the page uses the house pattern: a bordered CSS-grid table + a simple Previous/Next pager with an "X–Y of Z" label. Same §6 behaviors, established widgets.
- **Year/month are `mat-select` dropdowns, not date pickers** — the `feedback_date_time_pickers` rule covers picking calendar dates/times; a year/month *filter* over discrete options is a dropdown (matching the §4.1 "year picker / month picker" dropdown intent).
- **Field names are snake_case** matching the wire (`earliest_attendance_us`, `distinct_class_ids`), per the codebase convention — not the plan's camelCase prose names. Query type is `AttendanceHistoryQuery` (camelCase inputs) → network layer builds the snake_case query string.
- **URL is the single source of truth**: user actions `router.navigate` query params (`year/month/class_id/instructor_id/show_all/page`); the component fetches from the `queryParamMap` subscription, so browser back/forward restores filters AND page for free.
- Session times render with `timeZone: 'UTC'` (wall-clock encoding, the Phase 8 rule) — pinned by a timezone-proof spec test.

### 4.1 Attendance history page ✅
- [x] `pages/account/attendance-history/attendance-history.component.{ts,html,scss,spec.ts}` (standalone, SharedModule), route `/my/attendance-history` in `account.routes.ts`, + an "Attendance History" card on the account dashboard (`user.component` html/ts/spec updated — card count 11→12 + nav test).
- [x] Filter row: year + month + class + instructor `mat-select`s, "Show no-shows & cancellations" slide-toggle (the §1.1 "Show all" toggle: sets BOTH include flags). Class/instructor dropdowns resolve labels via `getClasses()`/`getInstructors()` against the metadata ids, with `Class #id` fallbacks for catalog-removed classes.
- [x] Year dropdown from `earliest_attendance_us` (UTC-read year, per §2.3's encoding note) up to the current year, descending; null ⇒ filters disabled + no-history empty state (OQ-P9-2).
- [x] Two empty states: no-history ("You haven't attended any classes yet" + catalog link) vs no-matches (filters active, with a clear-filters action).
- [x] Rows: date + time range (UTC-formatted), class (+walk-in tag), facility · room, instructors, status chip (green attended / red no-show / gray cancelled).
- [x] Pager: Previous/Next + range label, 25/page; page index round-trips through the URL.

### 4.2 `ServerAccess` extension ✅
- [x] `getMyAttendanceHistory(query?)` in interface + `ServerAccessNetwork` (query-string builder + row normalization: pipe-split `instructor_names`, string-bool `is_walkin`, `Number()`ed ids, null-preserved `earliest_attendance_us`) + proxy + mock.
- [x] Mock mirrors the server: encoded-as-UTC year/month matching, attended-only default, slot/staffing-style instructor ids, pagination, metadata unaffected by filters, month 13 → 400, 401 when logged out. 4 seeded rows (2 classes, 2 instructors, a no-show, a walk-in, a 2025 earliest).
- [x] `ServerAccess.mock.spec.ts` — 8 new cases incl. the **no-attendance → null earliest** case.

### 4.3 Types ✅
- [x] `shared/types/attendance-history.types.ts`: `AttendanceHistoryQuery`, `AttendanceHistoryRow`, `AttendanceHistoryStatus`, `AttendanceHistoryResponse` (carries `earliest_attendance_us: number | null` + the distinct id lists); re-exported from the ServerAccess barrel.

## 5. Tests-Required Summary

- [x] Business logic test cases enumerated in 2.6 (incl. distinct-bookings count + `GetEarliestAttendanceUs`) — done in §2.
- [x] Endpoint test cases (filter combos, pagination, validation-errors, and `earliest_attendance_us` present in metadata / null when no attendance) — done in §3.
- [x] Component spec for filter UI + paginator + empty state + mock service, incl. the year dropdown built from `earliest_attendance_us` (and disabled/empty when null) — done in §4: 12 component cases (init load + render, UTC time-format pin, year-range build, label resolution with fallbacks, both empty states, filter pass-through + page reset, show-all flags, pagination offsets/range label/guards, **URL-init for back-navigation**, no-show chip + walk-in tag, clear-filters) + 8 mock cases. Full Angular suite: 2569/2569.

## 6. Cross-Layer Acceptance Criteria

A member with 18 months of attendance:
- [ ] Opens `/my/account/attendance-history` and sees the most recent 25 attendances.
- [ ] Filters to "April 2026" — only April sessions appear, even if some were 11pm Pacific on March 31 (correct because TZ-aware).
- [ ] Filters to "instructor = Sara" — only Sara-led sessions appear.
- [ ] Clicks page 2 → next 25 rows; URL/state preserved on browser-back.
- [ ] Toggles "Show no-shows" → no-show rows appear with a red status chip.

## 7. Open Questions

Both resolved (Mason, 2026-06-09: "go with your recommendation") and folded into the plan above (§1.1 Locked-in + the cited sections).

- **OQ-P9-1. — RESOLVED.** Count **distinct bookings**, not distinct dates — 5 same-day attendances are 5 rows / count as 5. This is the existing per-`bookings`-row model; no date de-dup anywhere. Folded into §1.1, §2.3, §2.6, §5.
- **OQ-P9-2. — RESOLVED.** Year/month dropdown range runs from the user's **earliest attendance → today**, computed server-side: `GetEarliestAttendanceUs` → `earliest_attendance_us` in the endpoint metadata block → the UI builds the year list from it (null = no attendance → empty/disabled). Folded into §1.1, §2.3, §3.1, §4.1–4.3, §2.6, §5.

## 8. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 9.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 8 - Staff Check-in]] (creates the `bookings` rows we read here).
- Used by: future Reliability tracking ([[Classes Phase 16 - Stretch Items]]) — the same join + count infrastructure.
