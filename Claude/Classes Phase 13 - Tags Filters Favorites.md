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

Classes Phase 13 - Tags Filters Favorites

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

**Nice-to-Have.** Three discoverability features bundled because they're small individually but valuable together:

1. **Class category / tag taxonomy** (C-7) — controlled vocabulary in `class_tags` table; classes linked via `class_tag_assignments`. Drives catalog filter, calendar color-coding, AND the monthly-attendance-threshold prerequisite (SL-10).
2. **Favorite-instructor feature** (S-7 / N-11) — `user_favorite_instructors`; user marks favorites and gets notifications when a favorite appears on a new class / substitutes.
3. **Extended instructor profile pages** (C-8 / S-16) — extension of the existing instructor profile pages with class list + upcoming sessions.

**Prerequisites:**
- Phase 1 (classes catalog).
- Phase 10 (instructor substitution — favorite-instructor notification triggered here).
- Phase 5 (template / homepage feed reads tag info for filtering).
- Existing instructor profile infrastructure.

**Outcome:**
- `class_tags` controlled-vocabulary table + admin CRUD + assignment UI.
- Calendar / catalog filter by tag.
- Calendar color-coding by tag.
- `user_favorite_instructors` table + opt-in + notification on schedule changes for favorites.
- Existing `/instructors/<id>` profile page shows upcoming sessions + class list.

## Layering & Conventions

Lowest layer first per CLAUDE.md.

## 1. Pre-Coding Design Decisions

### 1.1 Locked-in
- [x] Controlled vocabulary (parent OQ-32) — `class_tags` table with admin CRUD, tag references via primary keys.
- [x] Tags applied to `classes` (NOT `class_schedules`) — color-coding and filtering happens at the class level.
- [x] **Multi-tag calendar chip color = first tag's color (resolved OQ-P13-1).** When a class has multiple tags, the calendar/catalog chip uses the color of the **lowest-`sort_order`** tag (a single solid color; no multi-color stripe). `GetTagsForClass` returns tags ordered by `sort_order ASC` so "first tag" is well-defined.
- [x] **Favorite-instructor notifications also fire on first appearance (resolved OQ-P13-2).** Beyond Phase 10 substitutions/shift-trades, a follower is also notified the **first time** a favorited instructor newly appears on the upcoming schedule (a new `class_schedules` impl / staffing assignment). This is driven by a **daily job**, not per-event, and fires once per (follower, instructor, class) appearance (see §3.3/§3.8).

## 2. Class Tags

### 2.1 Database schema ✅ (2026-06-23)
- [x] `db_schema/class_tags.h/.cpp`: `id BIGSERIAL PK`; `code TEXT NOT NULL UNIQUE` (`AddColumnSimple` + `AddUniqueConstraint` — **note:** `AddColumnUnique` produces a *nullable* unique column, caught by the `RequiresCode` test, so code uses NOT NULL + a single-column unique constraint like `instructors.person_id`); `name TEXT NOT NULL`; `color TEXT NOT NULL DEFAULT ''`; `sort_order BIGINT NOT NULL DEFAULT 0`; `is_active BOOLEAN NOT NULL DEFAULT TRUE`; `created_us`/`updated_us`. Plus `CreateClassTagsIndexes` → `(is_active, sort_order)` index for the active-list browse (mirrors `skill_levels`).
- [x] `db_schema/class_tag_assignments.h/.cpp`: `id BIGSERIAL PK`; `class_id` FK→classes; `class_tag_id` FK→class_tags; `created_us`; named UNIQUE `(class_id, class_tag_id)` (`uq_class_tag_assignments_class_tag`) — its leading `class_id` column also serves the §2.2 `GetTagsForClass` lookup, so no separate index needed.
- [x] Wired into DB init: `make_database_info.cpp` (includes + `MakeClassTagsTable`/`MakeClassTagAssignmentsTable` after the skill block — classes already made, class_tags before assignments), `create_database.cpp::CreateTables` (includes + `CreateTable` calls + index, same FK order), and `db_schema/CMakeLists.txt` (4 sources + 2 test files).
- [x] Schema tests: `class_tags_test.cpp` (defaults, color/sort/inactive, dup-code rejected, requires-code, requires-name, CreateIndexes adds + idempotent) and `class_tag_assignments_test.cpp` (valid insert, class-FK + tag-FK rejection, unique-blocks-duplicate-pair / allows-different-tag).
- *(Admin metadata registration is §2.6, deferred — not part of the schema layer; consistent with how Phase 12's new tables were added.)*

### 2.2 Table helpers ✅ (2026-06-24)
- [x] `TableHelpers::ClassTags` (`class_tags.h/.cpp`) — CRUD mirroring `SkillLevels`: `AddClassTag` (code/name + KVT overloads), `GetClassTag(id)`, `GetClassTagByCode(code)`, `GetActiveClassTags` (active, `ORDER BY sort_order, id`), `GetAllClassTags` (incl. inactive, ordered), `UpdateClassTag` (bumps `updated_us`), `DeleteClassTag` (soft-delete `is_active=false`). Tests `class_tags_test.cpp`: add/get + defaults, KVT color/sort, by-code (+unknown→empty), active-order (sort then id tiebreak, excludes inactive), all-incl-inactive ordered, update, soft-delete.
- [x] `TableHelpers::ClassTagAssignments` (`class_tag_assignments.h/.cpp`) — `AddAssignment(classId, tagId)`, **`GetTagsForClass(classId)`** → joined `class_tags` rows **ordered by `class_tags.sort_order ASC, id ASC`** (first = chip-color tag, resolved OQ-P13-1), `GetTagIdsForClass` (raw ids), `DeleteAssignment(classId, tagId)`, `DeleteAssignmentsForClass(classId)`, `SetTagsForClass(classId, ids)` (replace-all + dedupe, backs the §2.5 multi-select). Tests `class_tag_assignments_test.cpp`: add/get, **GetTagsForClass orders by sort_order regardless of assignment order**, tag-ids, class-scoping, delete-one, delete-all, set-replaces-and-dedupes (incl. set-to-empty).
- [x] Registered both helpers (4 sources) + 2 test files in `sql_util/table_helpers/CMakeLists.txt`.
- *(`GetTagsForClass` returns the joined `class_tags` rows as a `KeyValueTableArray` per the table-helper convention — the `ClassTagInfo` domain struct lives in §2.3 business logic, which maps these rows. No active-filter at this layer: a class lists all its assigned tags (§2.5); active-filtering is a §2.3 concern.)*

### 2.3 Business logic ✅ (2026-06-24)
- [x] New `struct ClassTagInfo { id; code; name; color; sortOrder; }`; upgraded `ClassCatalogEntry.tags` from `vector<string>` → `vector<ClassTagInfo>` (the old field was a placeholder "empty until Phase 13"). `BuildEntry` populates it from `ClassTagAssignments::GetTagsForClass` (ordered by `sort_order ASC`, so `tags[0]` is the chip-color tag — resolved OQ-P13-1). Added a `classTagAssignments_` member. Surfaces on every catalog entry AND `ClassDetail.summary`.
- [x] `FilterByTag`: optional `std::optional<int64_t> filterTagId` on `GetActiveClasses` and `GetClassesVisibleToPerson` — when set, only classes carrying that tag are returned (post-build `HasTag` filter; the catalog is small). `/api/classes?tag_id=` (§2.4) passes it through.
- [x] KVT converters (`scheduling_key_value_table.*`): `ClassCatalogEntryToKeyValueTable` now emits `chip_color` (first tag's color, "" when none) + `tag_count` instead of the old comma-joined `tags` string; added `ClassTagInfoToKeyValueTable` + `ClassTagInfosToKeyValueTableArray` so §2.4 endpoints can nest the full ordered tag array.
- [x] Tests: `class_catalog_helper_test.cpp` (tags populated in sort_order incl. empty-list, filter-by-tag returns only tagged / orphan-tag→empty / no-filter→all, visible-to-person filter, class-detail surfaces tags) + `scheduling_key_value_table_test.cpp` (updated the catalog converter test to ClassTagInfos asserting `chip_color`/`tag_count`, empty-tags chip_color, `ClassTagInfo` converter + array). Only the catalog helper, its converters, and their tests reference `.tags` — no other consumers affected.
- *(Endpoint JSON nesting of the `tags` array + the `tag_id` query param parse are §2.4; the converters above are the building blocks.)*

### 2.4 Endpoints ✅ (2026-06-24)
- [x] `GET /api/class_tags` (`get_class_tags.{h,cpp}`) → `{ items: [...] }` from `GetActiveClassTags` (active only, `sort_order` order). Public. Test `get_class_tags_test.cpp` (active-in-order excl. inactive, empty).
- [x] `GET /api/classes?tag_id=<id>` (`get_classes.cpp`): parses the optional `tag_id` query param → `GetActiveClasses/GetClassesVisibleToPerson(filterTagId)`; and now **nests the ordered `tags` array** per entry (each entry's flat KVT carries `chip_color` + `tag_count`; the array is spliced in as JSON, mirroring `upcoming_sessions`). `get_class_detail` likewise nests `tags`. Tests appended to `get_classes_test.cpp` (nests tags + chip_color from first/lowest-sort_order tag, empty when none; filter-by-tag returns only tagged — query param set via `crow::query_string` per `feedback_crow_query_params_test`).
- [x] **Admin tag CRUD (bespoke, gated `manage_class_schedule`):**
  - `GET /api/admin/class_tags` (`admin_class_tags_list`) → all incl. inactive. Test: 403, 200 (incl. inactive).
  - `POST /api/admin/class_tag` (`admin_class_tag_create`) body `{ code, name, color?, sort_order?, is_active? }`. Pre-checks `GetClassTagByCode` to return a clean **400 `DUPLICATE_CODE`** instead of a constraint 500. Test: 403, 400-missing-field, 400-dup-code, 200+persist.
  - `PUT /api/admin/class_tag/<id>` (`admin_class_tag_update`) — partial updates; 404 unknown; 400 `DUPLICATE_CODE` when renaming onto another tag's code (own code allowed). Test: 403, 404, 400-dup, 200.
  - `DELETE /api/admin/class_tag/<id>` (`admin_class_tag_delete`) — soft-delete (`is_active=false`); 404 unknown. Test: 403, 404, 200 (row kept inactive, drops from active list).
  - Registered all 5 endpoints in `web_app.cpp` + `endpoints/CMakeLists.txt`.
- *(Tag **assignment** to a class — the §2.5 multi-select — uses the `ClassTagAssignments::SetTagsForClass` helper from §2.2; the write endpoint vs. generic-CRUD path is decided in §2.5, so no set-tags endpoint added here, faithful to this §2.4 list.)*

### 2.5 Frontend
- [x] Class catalog (`public/class-info`) gains a tag-filter chip row at the top: an "All" chip plus a chip per active tag (`getClassTags()`); selecting one re-fetches `getClasses(tagId)`. Each class card now also shows its tags as colored chips. Spec covers the chip row, filter re-fetch (incl. "re-select active = no re-fetch"), colored chips, and graceful degradation when the vocabulary load fails.
- [x] Calendar event chips: added optional `color?` to `CalendarEvent`; `calendar-event.component` renders it as the chip's left-edge accent (`event-chip-tagged`), defaulting when absent. Spec asserts the color is applied. **NOTE (deferral):** the calendar is still a disconnected mock (`mockCalendarResponse`, a `// TODO replace with API call` stub) — it carries no class linkage, so the color *plumbing* exists but the data wiring (deriving each event's color from its class's first tag) waits until the calendar reads real class data. Tracked as **[[Classes Phase 16 - Stretch Items]] §4 (CAL-1)**; not blocking §5 acceptance which is exercised through the catalog chips.
- [x] Class detail page (`public/class-detail`) lists all of a class's tags as colored chips, in `sort_order`. Spec covers ordered colored chips + the no-tags case.
- [x] **New bespoke "Manage Tags" admin page** `manage/class-tags/class-tags-admin.component.*` (+ `.spec.ts`), modeled on `skills-admin`: a table of tags (color swatch + name + code + sort order + active state, incl. inactive), inline add/edit panels with a native `<input type="color">` picker, `sort_order` field, and an active toggle; create/edit/delete wired to the §2.4 admin endpoints (dup-code → friendly message; delete error → "set inactive instead"). Route `/manage/class-tags` + a "Class Tags" Manage-dashboard card (with a dashboard spec). 13 component specs.
- [x] Admin class-edit form (`class-form-dialog`): multi-select (`mat-select multiple`) of tags; inactive tags flagged in the option label. Parent (`class-schedule-manage`) loads the vocabulary (`getAdminClassTags`) + the class's current tags (`getClassDetail().tags`) before opening, and on save writes the class row then `setClassTags(classId, tagIds)` (non-fatal if the tag write fails). Specs on both the dialog (selection in/out) and the parent (vocabulary seeded, current tags pre-selected, setClassTags applied, vocabulary-load-failure still creates the class).
- [x] `ServerAccess`: added `getClassTags()`, `getAdminClassTags()`, `createClassTag(req)→{id}`, `updateClassTag(id, req)→{ok}`, `deleteClassTag(id)→{ok}`, `setClassTags(classId, tagIds)→{ok}`, and changed `getClasses(tagId?)`. New `shared/types/class-tag.types.ts` (`ClassTag`, `ClassTagRef`, `CreateClassTagRequest`, `UpdateClassTagRequest`); `ClassCatalogEntry.tags` upgraded `string[]`→`ClassTagRef[]` + added `chip_color`/`tag_count`. Implemented across interface/network (with `normalizeClassTag` is_active coercion)/proxy/mock; `ServerAccess.mock.spec.ts` gained 14 cases (vocabulary, admin list + 401, create dup/missing, update rename/dup/404, soft-delete, setClassTags + 404, plus catalog tag-nesting + `tag_id` filter).

### 2.6 Admin metadata (debug-only fallback — NOT the authoring workflow)
Per `feedback_manage_data_is_debug_only.md`: the real tag-vocabulary workflow is the §2.5 Manage Tags page. The registration below is a debug/inspection escape hatch only.
- [x] `class_tags` → top-level. Permission `manage_class_schedule`. (Done in `create_database.cpp` Populate* functions; inspection/debug only — authoring lives in Manage Tags.)
- [x] `class_tag_assignments` → nested under `classes` keyed by `class_id`. (Done; inspection/debug only — assignment lives in the class-edit multi-select.)

### 2.7 Tests
- [x] Helpers + endpoint tests (incl. the bespoke admin `class_tag` GET/POST/PUT/DELETE: 403 / dup-code / persist, and the new `PUT /api/admin/class/<id>/tags` set-tags endpoint: 403/404/400/200-replaces-then-clears) done in §2.2–2.4. Frontend specs: Manage Tags page spec, class-edit multi-select dialog + parent specs, catalog filter/chip specs, class-detail tag specs, calendar-event color spec, and `ServerAccess.mock.spec.ts` cases. **Verified: `ng build` clean + full Karma suite green (2759/2759).**

## 3. Favorite Instructors

### 3.1 Database schema
- [ ] `db_schema/user_favorite_instructors.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL REFERENCES people(id)`
  - `instructor_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `notify_on_schedule_change BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us`
  - `UNIQUE (person_id, instructor_person_id)`
- [ ] `db_schema/favorite_instructor_notifications.h/.cpp` — **sent-log so the daily first-appearance job fires once per appearance (resolved OQ-P13-2)** and stays idempotent across reruns:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL REFERENCES people(id)` — the follower
  - `instructor_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `class_id BIGINT NOT NULL REFERENCES classes(id)` — the class the instructor newly appeared on
  - `notified_us BIGINT NOT NULL`
  - `UNIQUE (person_id, instructor_person_id, class_id)` — one "new appearance" email per follower per (instructor, class).

### 3.2 Table helper
- [ ] `TableHelpers::UserFavoriteInstructors` + tests.
- [ ] `TableHelpers::FavoriteInstructorNotifications` + tests — `HasNotified(Transaction&, personId, instructorPersonId, classId)` and `RecordNotified(...)` (idempotent on the UNIQUE constraint), backing the §3.3 first-appearance dedupe.

### 3.3 Business logic
- [ ] In `business_logic/scheduling/favorite_instructor_helper.h/.cpp`:
  - `AddFavorite(Transaction&, personId, instructorPersonId)` — idempotent.
  - `RemoveFavorite(Transaction&, personId, instructorPersonId)`.
  - `GetFavoriteInstructorIdsForPerson(Transaction&, personId)`.
  - `GetFollowersOfInstructor(Transaction&, instructorPersonId)` — used to fan out notifications.
- [ ] Hook into Phase 10's `InstructorSubstitutionHelper` and `ShiftChangeHelper`: after a substitution / shift-trade execution, fan out a notification email to each follower of the new instructor: "Maya is teaching Vinyasa Flow this Tuesday — a class you might love." Throttle: at most one email per (follower, instructor) per 24h.
- [ ] **First-appearance fan-out (resolved OQ-P13-2):** `int NotifyNewScheduleAppearances(Transaction&, MailHelper*, int64_t nowUs)` — the daily-job entry point:
  1. Find instructor↔class pairs that newly appear on the **upcoming** schedule: classes whose active `class_instances`/`class_schedules` impls (or `event_session_staffing` rows for future sessions) assign an instructor and were created/changed since the prior run (scan a "since last run" window; the sent-log makes exact windowing non-critical).
  2. For each such (instructor, class): for each follower from `GetFollowersOfInstructor` with `notify_on_schedule_change=true`, skip if `FavoriteInstructorNotifications::HasNotified(follower, instructor, class)`; else queue "{Instructor} is now teaching {Class} — a class you might love", `RecordNotified(...)`, and respect the same at-most-one-per-(follower,instructor)-per-24h throttle as the substitution path.
  3. Return the count sent. Idempotent: a second run the same day sends nothing new (sent-log + throttle).

### 3.4 Endpoints
- [ ] `POST /api/me/favorite_instructor/<personId>` — add. Endpoint test.
- [ ] `DELETE /api/me/favorite_instructor/<personId>` — remove.
- [ ] `GET /api/me/favorite_instructors` — list.
- [ ] `POST /api/admin/send_favorite_instructor_schedule_alerts` — cron-callable, idempotent; runs `NotifyNewScheduleAppearances(now)`. Permission `admin`. Endpoint test (403 + 200 sends-once-then-no-op).

### 3.5 Frontend
- [ ] Add a "favorite" heart icon on instructor profile pages (3.6 / §4 below) + on class-detail instructor-list rows.
- [ ] `/my/account/favorite-instructors` page — list, manage.
- [ ] Per-user notification preference toggle: "Email me when a favorite instructor is teaching" (already lives in Phase 6's preferences page; extend).
- [ ] `ServerAccess`: `addFavoriteInstructor`, `removeFavoriteInstructor`, `getMyFavoriteInstructors`. Update mock.

### 3.6 Admin metadata (inspection only)
- [ ] `user_favorite_instructors` → nested under `people` keyed by `person_id`. Permission `admin`. This table is **user-generated** (rows created by the §3.5 favorite heart, removed by the user) — there is no admin authoring workflow to build, so registering it purely for inspection in Manage Data is appropriate (per `feedback_manage_data_is_debug_only.md`).
- [ ] `favorite_instructor_notifications` → nested under `people` keyed by `person_id`. Permission `admin`. **System-generated** sent-log (written by the §3.8 daily job); inspection only — never hand-authored.

### 3.7 Tests
- [ ] Helper + endpoint + frontend specs + mail-helper assertion that the substitution/shift-trade fan-out queues exactly one email per follower per change with the 24h dedupe respected.
- [ ] **First-appearance (resolved OQ-P13-2):** `NotifyNewScheduleAppearances` queues one email per follower when a favorited instructor newly appears on a class's upcoming schedule; a second same-day run sends nothing (sent-log + 24h throttle); a follower with `notify_on_schedule_change=false` gets none; `FavoriteInstructorNotifications::HasNotified`/`RecordNotified` idempotency.

### 3.8 Scheduled job (resolved OQ-P13-2)
- [ ] Add a **daily** job to `knottyyoga_helper`: `POST /api/admin/send_favorite_instructor_schedule_alerts`. Idempotent; wired in the three standard places (`scheduler/scheduled_job.cpp` `BuildStandardJobs` via `AppendIfEnabled`, a `JobIntervals::favoriteInstructorAlertSeconds` default 86400s, and a `--favorite_instructor_alert_interval` flag in `scheduler/main.cpp`). Per the existing interval-cron pattern, the endpoint self-gates / is idempotent rather than relying on an exact time. Update `scheduled_job_test` (job count + disable/propagate cases for the new job).

## 4. Extended Instructor Profile Pages

### 4.1 Business logic
- [ ] In `business_logic/scheduling/instructor_profile_helper.h/.cpp` (new):
  - `struct InstructorProfile { int64_t personId; std::string firstName; std::string lastName; std::string bio; std::string photoUrl; std::vector<ClassSummary> classesTaught; std::vector<UpcomingSessionInfo> upcomingSessions; }`.
  - `InstructorProfile GetInstructorProfile(Transaction&, int64_t personId)` — joins `people` → `event_session_staffing` → distinct `class_id`; pulls upcoming sessions in the next 4 weeks.

### 4.2 Endpoints
- [ ] `GET /api/instructors/<id>` — public. Endpoint test.
- [ ] `GET /api/instructors` — listing of active instructors. Public.

### 4.3 Frontend
- [ ] `ui/src/app/pages/instructors/instructor-detail/instructor-detail.component.*/.spec.ts` (extend if exists).
- [ ] Hero photo + bio + tag chips (their primary tags) + upcoming sessions list + favorite heart.
- [ ] Linked from class detail page (each instructor's name is a hyperlink).

### 4.4 `ServerAccess`
- [ ] `getInstructor(id)`, `getInstructors()`. Update mock.

### 4.5 Tests
- [ ] Helper + endpoint + frontend specs.

## 5. Cross-Layer Acceptance Criteria

- [ ] Admin creates tags "yoga", "aerial", "partner-acro" with distinct colors. Assigns "yoga" to Vinyasa Flow, "aerial" to Aerial 101, "partner-acro" to Partner Acro - All Levels and Partner Acro - Intermediate.
- [ ] Calendar shows yellow chip for yoga, purple for aerial, teal for partner-acro.
- [ ] A class tagged both "yoga" (sort_order 1) and "partner-acro" (sort_order 3) shows the **yoga** color on its calendar chip — the lowest-sort_order tag wins (resolved OQ-P13-1).
- [ ] Catalog filter "partner-acro" returns the two partner-acro classes.
- [ ] Phase 5's SL-10 monthly-attendance rule "≥ 4 partner-acro in last month → grants acro_club" now correctly identifies the partner-acro classes via tag membership.
- [ ] User favorites instructor "Sara"; when admin substitutes Sara into a new Wednesday class, follower receives a notification email next day; if Sara substitutes into a second class within 24h, NO second email (dedupe).
- [ ] Admin newly schedules Sara to teach a brand-new Friday class; the next daily run emails her followers "Sara is now teaching {Class}" once; subsequent daily runs send nothing for that same appearance (resolved OQ-P13-2).
- [ ] Visiting `/instructors/<sara-id>` shows her bio, photo, list of classes she teaches, next 4 weeks of sessions.

## 6. Open Questions

Both resolved (Mason, 2026-06-09: "go with your recommendation") and folded into the plan above (§1.1 Locked-in + the cited sections).

- **OQ-P13-1. — RESOLVED.** Multi-tag calendar chip uses the lowest-`sort_order` tag's single solid color (no multi-color stripe); `GetTagsForClass` returns `sort_order`-ordered tags. Folded into §1.1, §2.2, §2.5, §5.
- **OQ-P13-2. — RESOLVED.** Favorite-instructor notifications also fire on the **first appearance** of a favorited instructor on the upcoming schedule, via a **daily job** (`NotifyNewScheduleAppearances` → `POST /api/admin/send_favorite_instructor_schedule_alerts`), deduped once per (follower, instructor, class) by a new `favorite_instructor_notifications` sent-log plus the existing 24h throttle. Folded into §1.1, §3.1–3.4, §3.6–3.8, §5.

## 6.5 Manual Test Guide — §2 Tags/Filters (frontend)

**Two ways to exercise this. Mock mode needs no backend; full-stack needs the C++ server + a reset DB.**

### Path A — Fast path (mock mode, no backend)
The mock seeds three tags (Yoga sort 1 / Aerial sort 2 / Partner Acro sort 3 + an inactive "Retired") and assigns Class 1 = Yoga+Partner Acro, Class 2 = Partner Acro.
1. Terminal: `cd ui` → `ng serve -c local` → open `http://localhost:4200`.
2. **Catalog filter** — top menu **Classes** (`/classes`). You'll see a chip row: **All · Yoga · Aerial · Partner Acro** (Retired is hidden — it's inactive). "All" is dark (selected). "Knotty Yoga" shows amber **Yoga** + teal **Partner Acro** chips; "Partner Acrobatics" shows teal **Partner Acro**.
   - Click **Partner Acro** → both classes remain. Click **Yoga** → only "Knotty Yoga". Click **All** → both return.
3. **Class detail** — click "Knotty Yoga" → under the title you see **Yoga** then **Partner Acro** chips (Yoga first = lowest sort_order). This is the OQ-P13-1 rule.
4. **Manage Tags** (mock is always "logged in") — top menu **Manage** → **Class Tags** card → `/manage/class-tags`. Add/edit/delete tags with the color picker; changes reflect in the list.

### Path B — Full stack (real backend)
**Prereqs:** reset the DB with `knottyyoga_database_helper`, start the C++ server, `cd ui && ng serve`, and log in as a user holding **manage_class_schedule** (an admin/manager). The DB starts with **no** tags.

**Step 1 — Create the tag vocabulary (Manage Tags page).**
- Top nav **Manage** → on the dashboard click the **Class Tags** card (icon `label`) → `/manage/class-tags`.
- Click **Add tag**. Enter exactly:
  - **Code:** `yoga`  **Name:** `Yoga`  **Color:** pick amber (`#f59e0b`)  **Sort order:** `1`  **Active:** on → **Create**.
- **Add tag** again: **Code** `aerial` · **Name** `Aerial` · **Color** purple (`#8b5cf6`) · **Sort order** `2` · **Active** on → **Create**.
- **Add tag** again: **Code** `partner-acro` · **Name** `Partner Acro` · **Color** teal (`#14b8a6`) · **Sort order** `3` · **Active** on → **Create**.
- You now have three rows, each with a colored swatch, name, code, **Active** chip, and `#sort`.
- **Duplicate-code check:** Add tag → **Code** `yoga` · **Name** `Dup` → **Create** ⇒ red message "That code is already in use." Cancel.
- **Edit check:** click the pencil on **Yoga** → change **Name** to `Yoga Flow`, **Save** ⇒ row updates. (Re-open and set it back to `Yoga` if you like.)

**Step 2 — Assign tags to a class (Class Schedules → edit class).**
- **Manage** dashboard → **Class Schedules** card → `/manage/class-schedules`.
- Create a class if none exists: **Add class** → **Name** `Knotty Yoga`, **Description** anything, **Default capacity** `16`, **Kind** `Recurring`, **Active** checked. In the **Tags** dropdown (multi-select) tick **Yoga** and **Partner Acro** → **Save**.
  - (To test editing: select the class in the left list, click **Edit class** — the **Tags** dropdown is pre-checked with its current tags. Add/remove and **Save**.)
- Create/edit a second class `Partner Acrobatics` and tag it **Partner Acro** only.

**Step 3 — Verify the public catalog filter + chips.**
- Top nav **Classes** (`/classes`, public — log out or use another browser if you want the logged-out view; the filter works either way).
- The chip row reads **All · Yoga · Aerial · Partner Acro**. Each class card shows its colored tag chips.
- Click **Partner Acro** ⇒ both tagged classes show. Click **Yoga** ⇒ only "Knotty Yoga". Click **Aerial** ⇒ empty/"No classes" (nothing tagged aerial yet). Click **All** ⇒ everything.

**Step 4 — Verify class detail tag list + the chip-color rule.**
- From the catalog click **Knotty Yoga** → under the title, tag chips render **in sort_order**: **Yoga** (amber) then **Partner Acro** (teal). The lower-sort_order tag (Yoga) is first — this is the chip/accent color (OQ-P13-1).
- A class with no tags shows no chip row.

**Step 5 — Inactive + delete behavior.**
- Manage Tags → edit **Aerial** → toggle **Active** off → **Save**. Back on `/classes`, **Aerial** disappears from the filter row (public vocabulary is active-only). In the class-edit **Tags** dropdown it shows as `Aerial (inactive)`.
- Manage Tags → delete a tag that is **not** assigned to any class → it's removed (soft-deleted). Deleting a tag still referenced surfaces "This tag is in use — set it inactive instead."

**Step 6 — (Optional) debug inspection.** `Manage Data` (admin generic editor) now lists **Class Tags** (top-level) and **Class Tag Assignments** (nested under classes) for read-only inspection — but authoring is the Manage Tags page above, not this editor.

**Known gap:** the Calendar view (`/calendar`) is still a disconnected demo (hardcoded sample events); its chips have the color hook wired but won't show real tag colors until the calendar is connected to live class data. **Tracked as [[Classes Phase 16 - Stretch Items]] §4 (CAL-1)** — the calendar↔live-data wiring that also unblocks the deferred Phase 2 §6.2 / Phase 10 / Phase 11 calendar work.

## 7. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 13.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 10 - Scheduling Exceptions and Shift Trades]].
- Tags feed [[Classes Phase 6 - Weekly Digest]] (color-coding in digest rows) and the SL-10 prerequisite rule from Phase 3.
- Existing instructor infrastructure: [[Adding an instructor page with photos]].
