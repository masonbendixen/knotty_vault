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

### 1.2 Filter semantics
- [x] `year` and `month` filter the session's date in **facility-local TZ** (NOT UTC). Justification: a 11pm Pacific session on March 31 reads as March, not April UTC.
- [x] `class_id` filters by `event_sessions.class_id` (denormalized convenience set when the occurrence row is ensured in Phase 1).
- [x] `instructor_id` filters via `event_session_staffing` rows with that `person_id` and role in (`'instructor'`, `'lead_instructor'`).

## 2. Business Logic — `AttendanceHistoryHelper`

Files: `business_logic/scheduling/attendance_history_helper.h/.cpp/_test.cpp`.

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

### 2.3 Methods
- [ ] `int64_t GetTotalCount(Transaction&, int64_t personId, const AttendanceHistoryFilter&)` — count query.
- [ ] `std::vector<AttendanceHistoryRow> GetHistory(Transaction&, int64_t personId, const AttendanceHistoryFilter&)` — the paginated read.
- [ ] `std::vector<int64_t> GetDistinctInstructorIdsForPerson(Transaction&, int64_t personId)` — feeds the filter dropdown.
- [ ] `std::vector<int64_t> GetDistinctClassIdsForPerson(Transaction&, int64_t personId)` — feeds the filter dropdown.

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
- [ ] Facility-TZ extraction handled in SQL with `AT TIME ZONE f.timezone`. Verify the Postgres TZ DB has the relevant zones (it does by default).
- [ ] Instructor names assembled in a separate batch query (one round-trip after the page query) to avoid N+1.

### 2.5 KeyValueTable conversion
- [ ] `AttendanceHistoryRowToKeyValueTable(...)` in `scheduling_key_value_table.h/.cpp`.

### 2.6 Tests
- [ ] `attendance_history_helper_test.cpp`:
  - Empty result for a person with no bookings.
  - Filter by year/month uses facility TZ (test with a session straddling midnight UTC).
  - Filter by class_id excludes other classes.
  - Filter by instructor_id excludes other instructors.
  - `includeNoShow=false` excludes no-shows by default.
  - Pagination correct (offset 0 / limit 10 returns most-recent 10; offset 10 returns the next 10).
  - Sort: most recent first.

## 3. Endpoints

### 3.1 User endpoint
- [ ] `endpoints/get_my_attendance_history.h/cpp` + test:
  - `GET /api/me/attendance_history?year=&month=&class_id=&instructor_id=&include_no_show=&include_cancelled=&offset=&limit=`.
  - Permission: logged-in session.
  - Returns `{ total_count, rows: [...], distinct_class_ids, distinct_instructor_ids }` — the distinct lists drive the filter dropdowns.
  - Use `crow::query_string` for query params (per memory).

## 4. Frontend

### 4.1 Attendance history page
- [ ] `ui/src/app/pages/account/attendance-history/attendance-history.component.*/.spec.ts`.
- [ ] Layout: filter row at top (year picker, month picker, class FK picker, instructor FK picker, "Show no-shows" toggle), then a Material table, then a Material paginator.
- [ ] Filter values feed query params; pagination state preserved on back-navigation.
- [ ] Empty state: "You haven't attended any classes yet. Browse the catalog at /classes."
- [ ] Mat-card border + back-nav + RouterTestingModule per memory `feedback_account_page_layout.md`.
- [ ] Use real date/time pickers per memory `feedback_date_time_pickers.md`.
- [ ] Spec covers filter combinations + pagination + empty state.

### 4.2 `ServerAccess` extension
- [ ] `getMyAttendanceHistory(filter): Observable<AttendanceHistoryResponse>`.
- [ ] Update `ServerAccess.mock.spec.ts`.

### 4.3 Types
- [ ] `attendance-history.types.ts`: `AttendanceHistoryFilter`, `AttendanceHistoryRow`, `AttendanceHistoryResponse`.

## 5. Tests-Required Summary

- [ ] Business logic test cases enumerated in 2.6.
- [ ] Endpoint test cases (filter combos, pagination, validation-errors).
- [ ] Component spec for filter UI + paginator + empty state + mock service.

## 6. Cross-Layer Acceptance Criteria

A member with 18 months of attendance:
- [ ] Opens `/my/account/attendance-history` and sees the most recent 25 attendances.
- [ ] Filters to "April 2026" — only April sessions appear, even if some were 11pm Pacific on March 31 (correct because TZ-aware).
- [ ] Filters to "instructor = Sara" — only Sara-led sessions appear.
- [ ] Clicks page 2 → next 25 rows; URL/state preserved on browser-back.
- [ ] Toggles "Show no-shows" → no-show rows appear with a red status chip.

## 7. Open Questions

- **OQ-P9-1.** Should we count distinct dates or distinct bookings? A member who attended 5 classes on the same day has 5 bookings. Recommended: distinct bookings — they're 5 separate experiences.
	- Mason- I'll go with your recommendation.
- **OQ-P9-2.** Year/month dropdowns: populate from the user's earliest attendance date to today, or fixed range (last 5 years)? Recommended: from earliest to today, computed server-side and returned as part of the metadata block alongside `distinct_class_ids`.

## 8. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 9.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 8 - Staff Check-in]] (creates the `bookings` rows we read here).
- Used by: future Reliability tracking ([[Classes Phase 16 - Stretch Items]]) — the same join + count infrastructure.
