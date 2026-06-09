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

### 2.1 Database schema
- [ ] `db_schema/class_tags.h/.cpp`:
  - `id BIGSERIAL PK`
  - `code TEXT NOT NULL UNIQUE` (slug like `partner-acro`)
  - `name TEXT NOT NULL` (display name)
  - `color TEXT NOT NULL DEFAULT ''` (CSS color string for calendar chip — hex or named)
  - `sort_order BIGINT NOT NULL DEFAULT 0`
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us`, `updated_us`
- [ ] `db_schema/class_tag_assignments.h/.cpp`:
  - `id BIGSERIAL PK`
  - `class_id BIGINT NOT NULL REFERENCES classes(id)`
  - `class_tag_id BIGINT NOT NULL REFERENCES class_tags(id)`
  - `created_us`
  - `UNIQUE (class_id, class_tag_id)`

### 2.2 Table helpers
- [ ] `TableHelpers::ClassTags` + tests.
- [ ] `TableHelpers::ClassTagAssignments` + tests; method `GetTagsForClass(Transaction&, int64_t classId)` → list of `ClassTagInfo`, **ordered by `class_tags.sort_order ASC, id ASC`** so the first element is the chip-color tag (resolved OQ-P13-1). Test asserts the order.

### 2.3 Business logic
- [ ] Extend `ClassCatalogHelper` to surface tag list on every `ClassCatalogEntry` and `ClassDetail`.
- [ ] Add `FilterByTag(tagId)` to the catalog query.

### 2.4 Endpoints
- [ ] `GET /api/class_tags` → list active. Public.
- [ ] `GET /api/classes?tag_id=<id>` — extend existing endpoint with filter param.
- [ ] **Admin tag-vocabulary CRUD = bespoke, NOT Manage Data.** Managing the controlled vocabulary (create/edit/retire tags, set name/color/sort_order/is_active) is a real admin workflow, so it gets a dedicated "Manage Tags" UI (§2.5) — never the Manage Data generic table editor (memory `feedback_manage_data_is_debug_only.md`). Add:
  - `GET /api/admin/class_tags` → all tags incl. inactive. Permission `manage_class_schedule`. Endpoint test.
  - `POST /api/admin/class_tag` body `{ code, name, color, sort_order, is_active }` → create. Endpoint test (403 / 400-dup-code / 200+persist).
  - `PUT /api/admin/class_tag/<id>` → update. `DELETE /api/admin/class_tag/<id>` → soft-delete (`is_active=false`). Endpoint tests.
  - (Implementation note: these MAY be the generic CRUD REST endpoints called *from the bespoke §2.5 Manage Tags page* — the accepted Phase 1 `class-requirements-editor` pattern — but the **authoring surface must be the bespoke page, not the Manage Data editor**.)
  - Tag **assignment** to a class is already bespoke via the §2.5 class-edit multi-select (writes `class_tag_assignments`).

### 2.5 Frontend
- [ ] Class catalog gains a tag-filter chip row at the top.
- [ ] Calendar event chips use tag color.
- [ ] Class detail page lists tags.
- [ ] **New bespoke "Manage Tags" admin page** `manage/class-tags/class-tags-admin.component.*/.spec.ts`, modeled on `manage/skills/skills-admin.component`: a table of tags with inline create/edit/delete, a color picker for `color`, `sort_order` ordering, and an active/inactive toggle. Wired to the §2.4 admin tag endpoints. Reachable from the Manage dashboard. (This is the workflow that replaces "go to Manage Data → class_tags".)
- [ ] Admin class-edit form (`manage/class-schedules/dialogs/class-form-dialog`): multi-select of tags (writes `class_tag_assignments` for the class).
- [ ] `ServerAccess`: `getClassTags()`, `getAdminClassTags()`, `createClassTag(req)`, `updateClassTag(id, req)`, `deleteClassTag(id)`, `setClassTags(classId, tagIds)`, updated `getClasses(filter)`. Update `ServerAccess.mock.spec.ts` for every new method.

### 2.6 Admin metadata (debug-only fallback — NOT the authoring workflow)
Per `feedback_manage_data_is_debug_only.md`: the real tag-vocabulary workflow is the §2.5 Manage Tags page. The registration below is a debug/inspection escape hatch only.
- [ ] `class_tags` → top-level. Permission `manage_class_schedule`. (Inspection / debug only; authoring lives in Manage Tags.)
- [ ] `class_tag_assignments` → nested under `classes` keyed by `class_id`. (Inspection / debug only; assignment lives in the class-edit multi-select.)

### 2.7 Tests
- [ ] Helpers + endpoint tests (incl. the bespoke admin `class_tag` GET/POST/PUT/DELETE: 403 / dup-code / persist) + frontend specs (incl. the **Manage Tags page spec** and the class-edit tag multi-select spec) + `ServerAccess.mock.spec.ts` cases.

## 3. Favorite Instructors

### 3.1 Database schema
- [ ] `db_schema/user_favorite_instructors.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL REFERENCES people(id)`
  - `instructor_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `notify_on_schedule_change BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us`
  - `UNIQUE (person_id, instructor_person_id)`

### 3.2 Table helper
- [ ] `TableHelpers::UserFavoriteInstructors` + tests.

### 3.3 Business logic
- [ ] In `business_logic/scheduling/favorite_instructor_helper.h/.cpp`:
  - `AddFavorite(Transaction&, personId, instructorPersonId)` — idempotent.
  - `RemoveFavorite(Transaction&, personId, instructorPersonId)`.
  - `GetFavoriteInstructorIdsForPerson(Transaction&, personId)`.
  - `GetFollowersOfInstructor(Transaction&, instructorPersonId)` — used to fan out notifications.
- [ ] Hook into Phase 10's `InstructorSubstitutionHelper` and `ShiftChangeHelper`: after a substitution / shift-trade execution, fan out a notification email to each follower of the new instructor: "Maya is teaching Vinyasa Flow this Tuesday — a class you might love." Throttle: at most one email per (follower, instructor) per 24h.

### 3.4 Endpoints
- [ ] `POST /api/me/favorite_instructor/<personId>` — add. Endpoint test.
- [ ] `DELETE /api/me/favorite_instructor/<personId>` — remove.
- [ ] `GET /api/me/favorite_instructors` — list.

### 3.5 Frontend
- [ ] Add a "favorite" heart icon on instructor profile pages (3.6 / §4 below) + on class-detail instructor-list rows.
- [ ] `/my/account/favorite-instructors` page — list, manage.
- [ ] Per-user notification preference toggle: "Email me when a favorite instructor is teaching" (already lives in Phase 6's preferences page; extend).
- [ ] `ServerAccess`: `addFavoriteInstructor`, `removeFavoriteInstructor`, `getMyFavoriteInstructors`. Update mock.

### 3.6 Admin metadata (inspection only)
- [ ] `user_favorite_instructors` → nested under `people` keyed by `person_id`. Permission `admin`. This table is **user-generated** (rows created by the §3.5 favorite heart, removed by the user) — there is no admin authoring workflow to build, so registering it purely for inspection in Manage Data is appropriate (per `feedback_manage_data_is_debug_only.md`).

### 3.7 Tests
- [ ] Helper + endpoint + frontend specs + mail-helper assertion that fan-out queues exactly one email per follower per change with the 24h dedupe respected.

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
- [ ] Catalog filter "partner-acro" returns the two partner-acro classes.
- [ ] Phase 5's SL-10 monthly-attendance rule "≥ 4 partner-acro in last month → grants acro_club" now correctly identifies the partner-acro classes via tag membership.
- [ ] User favorites instructor "Sara"; when admin substitutes Sara into a new Wednesday class, follower receives a notification email next day; if Sara substitutes into a second class within 24h, NO second email (dedupe).
- [ ] Visiting `/instructors/<sara-id>` shows her bio, photo, list of classes she teaches, next 4 weeks of sessions.

## 6. Open Questions

- **OQ-P13-1.** When a class has multiple tags, which color drives the calendar chip? Recommended: the first (lowest sort_order) tag's color; alternatively split the chip visually (a tiny multi-color stripe) — probably overkill. Start with first-tag's-color.
	- Mason- I'll go with your recommendation.
- **OQ-P13-2.** Should favorite-instructor notifications also fire on the *first time* a favorite is on the upcoming schedule (not just substitutions)? Recommended: yes — extend Phase 13 fan-out to also fire when a new `class_schedule` is created with the instructor assigned. Daily job rather than per-event.
	- Mason- I'll go with your recommendation.

## 7. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 13.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 10 - Scheduling Exceptions and Shift Trades]].
- Tags feed [[Classes Phase 6 - Weekly Digest]] (color-coding in digest rows) and the SL-10 prerequisite rule from Phase 3.
- Existing instructor infrastructure: [[Adding an instructor page with photos]].
