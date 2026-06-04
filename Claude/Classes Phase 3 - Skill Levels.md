---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 6/3/2026
Version: 0.2
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please use your built in tools for read only operations on the filesystem or just say yes but do NOT prompt me when performing work that only reads the filesystem. I want you to run to completion (putting questions to be answered) but DO NOT FUCKING PROMPT ME. Please leave this Overview alone and build the plan in the following sections.

Classes Phase 3 - Skill Levels

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

**Should-have, blocking some Must-Have edge cases.** Skill levels are global named/photographed capabilities (e.g. "Can kick up to handstand against a wall"). Staff assigns / revokes them per person. A class can list required skill levels. Booking attempts (for the paid offerings that pass through `BookEvent`) enforce the gate; staff has override. Users view their own skills in the user portal.

**Prerequisites:**
- Phase 1 (classes catalog) — `classes` table exists; class detail page exists.
- Phase 2 (membership-gated drop-in) — booking flow is in shape for the paid path.
- Existing photo infrastructure ([[Adding support for images]]).
- Existing permission infrastructure (RBAC).

**Outcome:**
- `skill_levels`, `skill_level_assignments`, `class_skill_requirements` tables exist.
- Staff portal: search a person → view their skill levels → assign / revoke with a note.
- User portal: `/my/account/skills` shows the user's earned badges.
- Class detail page surfaces "Requires: X, Y, Z" prominently, with "You have" / "You're missing" per skill.
- Booking enforcement: `BookEvent` and the staff-check-in flow (Phase 8) verify prerequisites and reject with a clear error; admin / `manage_skills` permission can override with a logged reason.

## Layering & Conventions

Lowest layer first:

1. `db_schema/` — three new tables.
2. `sql_util/table_helpers/` — three new helpers.
3. `business_logic/` — `SkillLevelHelper`; booking-flow hook.
4. `endpoints/` — user + staff + admin endpoints.
5. Angular UI: user portal, staff portal, class-detail + admin-class-edit extensions.
6. Admin metadata in `create_database.cpp`.
7. Tests at every layer.

Skill-level photos hook into the existing `photo_support_tables` whitelist.

## 1. Pre-Coding Design Decisions

### 1.1 Scope (resolved per parent doc §2.6)
- [x] Skill levels are **global** (one set across the studio), not per-facility.
- [x] A class can have multiple required skills (AND-of-multiple).
- [x] Skill-level requirements key off `classes` only — no per-instance override in Phase 3 (could-have for later).

### 1.2 Compose with other prerequisite mechanisms (resolved per parent doc SL-12)
- [x] Skill check is one of three composable mechanisms at the booking gate: skill mastery (this phase), same-day predecessor (Phase 1 column + Phase 5 enforcement), and monthly-attendance-permission (Phase 13 / later). All three are SQL-and-permission checks; no rule engine / DSL.

### 1.3 Revocation
- [x] Yes — staff can revoke a previously granted skill. Audit-trail-friendly: capture `removed_us`, `removed_by_person_id`, `removed_reason`.

### 1.4 Reconciliation with the permission-based access model (added 2026-05-31)

> Per [[Permission-based class access redesign]] §4.4. The redesign built the **shared access gate + requirement-group infra** that this phase's skill gate (SL-5/6, SL-12) plugs into — Phase 3 **consumes** it rather than building a parallel mechanism.

**Already built (redesign §3, do not rebuild):**
- [x] `Scheduling::ClassAccessHelper::CheckAccess(classId, personId)` — the one booking gate (SL-12). Evaluates the class's CNF requirement groups (AND across groups, OR within), closure-aware.
- [x] `class_requirement_groups` + `class_requirement_group_literals`. **A literal can already be a skill level**: `class_requirement_group_literals.skill_level_id` exists as a **nullable BIGINT with no FK** — a deliberate forward reference to *this* phase's `skill_levels` table.
- [x] `booking_requirement_overrides` audit table + `ClassAccessHelper::RecordOverride(...)` — the logged staff-override path (SL-6). Phase 3's "staff override with reason" writes here; do not invent a second override record.
- [x] `CatalogHelper::GetEffectivePermissionIds` is closure-expanded, so **SL-10 attendance-threshold permissions and any role/membership permission flow into the gate automatically** once granted.

**Decision (resolved 2026-06-03 — OQ-P3-SKILL): model skill requirements as skill literals in requirement groups.**
- [x] **Skill requirements are modeled as skill literals in `class_requirement_group_literals`.** SL-5 "class requires skill X" becomes a single-literal group `{skill_level_id: X}`; "X or Y" is one group with two skill literals; skill AND membership AND attendance is three groups — all evaluated by the one gate. This unifies SL-12 and is exactly what the redesign assumed ("the requirement-group skill literals are exactly SL-5").
- **Consequence:** the separate **§2.3 `class_skill_requirements` table + §3.3 helper + §4 `PersonMeetsClassRequirements`** are **superseded** — they are marked `[~]` and **not built**. Requirement authoring is the §6.6 Phase-1 Requirements editor (skill-literal picker), and enforcement is already in `ClassAccessHelper`. The admin endpoints in §5.3 and the class-edit UI in §6.4 are likewise redirected to the requirement-group editor.
- **(Rejected alternative)** Keeping `class_skill_requirements` as a dedicated side table would split the gate into "groups + a side table," re-introducing exactly the fragmentation SL-12 set out to avoid.

**Required work regardless of the above:**
- [x] **Extend `ClassAccessHelper::CheckAccess` to evaluate skill literals.** ✅ Done — `class_access_helper.cpp` now satisfies a skill literal iff the viewer holds an active `skill_level_assignments` row (`removed_us IS NULL`) for that `skill_level_id` (via `SkillLevelAssignments::PersonHasSkill`). `GetClassRequirements` now resolves the skill literal's display name from `skill_levels`. FK `class_requirement_group_literals.skill_level_id → skill_levels(id)` added (skill_levels now created before the literals table). Tests added in `class_access_helper_test.cpp` (skill-only gate, revoke re-blocks, skill-OR-permission within a group, skill-AND-permission across groups, name resolution).
- [x] **Booking enforcement is already in the gate** — `BookingHelper` and `ClassCatalogHelper` already call `ClassAccessHelper::CheckAccess`, so the skill side now flows through to booking/catalog automatically; **no new call site added**. The failed group labels in `AccessResult` carry the (now skill-aware) group names for the `MISSING_SKILL_REQUIREMENTS`-style error.

## 2. Database Schema

### 2.1 `skill_levels` table
- [x] `db_schema/skill_levels.h/.cpp`: ✅ built with all columns below.
  - `id BIGSERIAL PK`
  - `code TEXT NOT NULL UNIQUE`  (slug, e.g. `handstand_wall`)
  - `name TEXT NOT NULL`
  - `description TEXT NOT NULL DEFAULT ''`
  - `sort_order BIGINT NOT NULL DEFAULT 0`
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us`, `updated_us`
- [x] Index on (`is_active`, `sort_order`) — `CreateSkillLevelsIndexes` (real-DB path only).

### 2.2 `skill_level_assignments` table
- [x] `db_schema/skill_level_assignments.h/.cpp`: ✅ built. `removed_by_person_id` is a nullable FK to `people(id)`; all other columns as below.
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL REFERENCES people(id)`
  - `skill_level_id BIGINT NOT NULL REFERENCES skill_levels(id)`
  - `assigned_by_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `assigned_us BIGINT NOT NULL DEFAULT now_us()`
  - `note TEXT NOT NULL DEFAULT ''`
  - `removed_us BIGINT` NULL
  - `removed_by_person_id BIGINT` NULL (FK people)
  - `removed_reason TEXT NOT NULL DEFAULT ''`
  - `created_us`, `updated_us`
- [x] Partial unique index `UNIQUE (person_id, skill_level_id) WHERE removed_us IS NULL` + `person_id` index — `CreateSkillLevelAssignmentsIndexes`. ⚠️ Note: `CreateXxxIndexes` run only in the real DB-creation path, **not** in the test harness's `SetupAllTables`, so the partial-unique guard is verified in prod, not unit tests; helper read paths use explicit `removed_us IS NULL` filters so correctness does not depend on the index.

### 2.3 `class_skill_requirements` table — `[~]` SUPERSEDED (do not build)
> ❌ **Superseded per §1.4 (OQ-P3-SKILL, resolved 2026-06-03).** Skill requirements are modeled as skill literals in `class_requirement_group_literals`; this dedicated table, its §3.3 helper, and §4's `PersonMeetsClassRequirements` are **not built**. Kept here only for historical context.
- [~] ~~`db_schema/class_skill_requirements.h/.cpp`:~~
  - ~~`id BIGSERIAL PK`~~
  - ~~`class_id BIGINT NOT NULL REFERENCES classes(id)`~~
  - ~~`skill_level_id BIGINT NOT NULL REFERENCES skill_levels(id)`~~
  - ~~`required_at_signup BOOLEAN NOT NULL DEFAULT TRUE`~~
  - ~~`created_us`~~
- [~] ~~Unique on (`class_id`, `skill_level_id`).~~

### 2.4 Photo support
- [x] In `create_database.cpp`, add `skill_levels` to `photo_support_tables` so the existing photo upload / scale endpoints accept skill-level photos. ✅ Added `AddRow(DbSchema::kSkillLevelsTable)` in `PopulatePhotoSupportTables` (header already included). Prod-only seed path (not exercised by the test harness, like the other `Populate*`/`CreateXxxIndexes` paths), so no unit test added.

### 2.5 Wire into DB init
- [x] `make_database_info.cpp` adds the two `Make*Table()` calls in FK order (`skill_levels`, then `skill_level_assignments`), placed before the requirement-group literals so the new FK target exists.
- [x] `create_database.cpp` `CreateTables()` adds the two `CreateTable()` + index calls (and includes the new headers).
- [x] Added the FK `class_requirement_group_literals.skill_level_id → skill_levels(id)` (per §1.4 required work).
- [x] CMakeLists for `db_schema/` and `sql_util/table_helpers/` (sources + the two new `*_test.cpp`).

## 3. Table Helpers

### 3.1 `TableHelpers::SkillLevels`
> ✅ Fully built (§1.4 gate slice + remaining staff/admin CRUD).
- [x] `skill_levels.h/.cpp/_test.cpp` created:
  - [x] `AddSkillLevel(Transaction&, code, name)` + `AddSkillLevel(Transaction&, const KeyValueTable&)`
  - [x] `GetSkillLevel(Transaction&, int64_t id)`
  - [x] `GetSkillLevelByCode(...)` — `DbCrud::GetRow` on the unique `code` column.
  - [x] `GetActiveSkillLevels(Transaction&)` — custom SQL `WHERE is_active = TRUE ORDER BY sort_order ASC, id ASC` (multi-column order → custom SQL per memory).
  - [x] `UpdateSkillLevel(Transaction&, int64_t id, const KeyValueTable& updates)` — `DbCrud::UpdateRow`, stamps `updated_us = now_us()`.
  - [x] `DeleteSkillLevel(Transaction&, int64_t id)` ← soft (sets `is_active=false` via `UpdateSkillLevel`).
- [x] Tests: add + defaults, KeyValueTable add, missing→empty, by-code (found/missing), **unique-code violation throws**, active-list ordering by `sort_order`, active-list excludes inactive, update changes fields + bumps `updated_us`, **soft delete deactivates + drops from active list**.

### 3.2 `TableHelpers::SkillLevelAssignments`
> ✅ Fully built.
- [x] `skill_level_assignments.h/.cpp/_test.cpp` created:
  - [x] `AddAssignment(Transaction&, personId, skillLevelId, assignedByPersonId, note)` (typed args rather than raw KeyValueTable)
  - [x] `GetAssignment(Transaction&, int64_t id)`
  - [x] `GetActiveAssignmentsForPerson(Transaction&, int64_t personId)` — `WHERE removed_us IS NULL`
  - [x] `GetAllAssignmentsForPerson(Transaction&, int64_t personId)` — historical including revocations (audit view); custom SQL `WHERE person_id = $1 ORDER BY assigned_us DESC, id ASC`.
  - [x] `GetActiveAssignment(Transaction&, int64_t personId, int64_t skillLevelId)`
  - [x] `PersonHasSkill(Transaction&, int64_t personId, int64_t skillLevelId)` → bool (consumed by the gate)
  - [x] `SetRemoval(Transaction&, int64_t id, int64_t removedByPersonId, std::string_view removedReason)`
- [x] Tests: add+defaults, `PersonHasSkill` active/inactive, `GetActiveAssignment`, `SetRemoval` revokes, re-grant after revoke creates a new active row (app-level; the partial-unique DB guard is prod-only per §2.2), active-list excludes revoked, **full audit view includes revoked + scopes to the person**, **empty audit view when none**.

### 3.3 `TableHelpers::ClassSkillRequirements` — `[~]` SUPERSEDED (do not build)
> ❌ **Superseded per §1.4.** Class skill requirements live in the existing `class_requirement_group_literals` table (managed by the requirement-group helper from the permission-based-access redesign). No dedicated helper is built.
- [~] ~~`class_skill_requirements.h/.cpp/_test.cpp`:~~
  - ~~`AddRequirement(Transaction&, ...)`~~
  - ~~`RemoveRequirement(Transaction&, int64_t classId, int64_t skillLevelId)`~~
  - ~~`GetRequirementsForClass(Transaction&, int64_t classId)`~~

## 4. Business Logic

### 4.1 `SkillLevelHelper` (in `business_logic/skills/`)
- [x] Placed in `business_logic/skills/skill_level_helper.h/.cpp` (new `Skills` namespace + subdir, wired into `business_logic/CMakeLists.txt`).
- [x] Methods (built; `PersonSkillInfo`/`SkillLevelInfo` expose `hasPhoto` rather than a `photoUrl` string, matching the codebase convention — the frontend builds `/api/photo/skill_levels/{id}` like it does for instructors/classes):
  ```cpp
  struct AssignSkillRequest { int64_t personId; int64_t skillLevelId; int64_t assignerPersonId; std::string note; };
  struct AssignSkillResult { bool ok; int64_t assignmentId; std::string errorCode; };  // "INVALID_ARGUMENT"|"PERSON_NOT_FOUND"|"SKILL_NOT_FOUND"
  AssignSkillResult AssignSkill(Transaction&, const AssignSkillRequest&);

  struct RevokeSkillRequest { int64_t personId; int64_t skillLevelId; int64_t revokerPersonId; std::string reason; };
  bool RevokeSkill(Transaction&, const RevokeSkillRequest&);

  struct PersonSkillInfo { int64_t assignmentId; int64_t skillLevelId; std::string code; std::string name; std::string description; bool hasPhoto; int64_t assignedUs; std::string note; };
  std::vector<PersonSkillInfo> GetPersonSkills(Transaction&, int64_t personId);

  // Typed catalog reads backing §5.1 endpoints (wrap TableHelpers::SkillLevels + resolve hasPhoto):
  std::vector<SkillLevelInfo> GetActiveSkillLevels(Transaction&);
  SkillLevelInfo GetSkillLevel(Transaction&, int64_t skillLevelId);  // id == 0 when missing
  ```
- [x] `AssignSkill` is idempotent — if person already has the active skill, returns ok with the existing id; does not insert duplicate. `assigned_us` defaulted to `now_us()` by the table helper. Validates person + skill exist first (clean errorCode instead of a raw FK throw).
- [x] `RevokeSkill` soft-revokes the active grant via `SkillLevelAssignments::SetRemoval` (stamps `removed_us`/`removed_by_person_id`/`removed_reason`). No-op (returns false) if not currently active.
- [x] All return-value structs surface enough info (code/name/description/hasPhoto/assignedUs/note) to render the UI without a follow-up call.
- [x] No CRUD SQL in the helper — it composes `TableHelpers::SkillLevels`, `TableHelpers::SkillLevelAssignments`, `TableHelpers::People`, and `Images::ImageHelper` (per `feedback_no_sql_in_business_logic`).
- [x] Tests `skill_level_helper_test.cpp`: assign creates active grant, idempotent (no duplicate), invalid-arg / missing-person / missing-skill error codes, revoke removes grant, revoke no-op when none, reassign-after-revoke makes a fresh active row, `GetPersonSkills` resolves fields + excludes revoked + empty case, active-list ordered/excludes-inactive, `GetSkillLevel` found/missing.

> ❌ **`PersonMeetsClassRequirements` / `ClassRequirementsCheck` are SUPERSEDED per §1.4.** The class-requirements check is owned by the shared `Scheduling::ClassAccessHelper::CheckAccess`. `SkillLevelHelper` exposes only assign / revoke / read-skills; it does **not** evaluate class gates.
- [x] **Extend `ClassAccessHelper::CheckAccess` to evaluate skill literals** — already done in §1.4 (skill literal satisfied iff active `skill_level_assignments` row; `skillLevelName` resolved in the read path; tests in `class_access_helper_test.cpp`).

### 4.2 Booking-flow integration ✅ (OQ-P3-3 resolved — option a)
> The gate is the shared `Scheduling::ClassAccessHelper`, now **enforced as a blocking gate at booking** (Mason: "a person shouldn't be able to sign up without the skill, absent admin/staff override; the UI shows the skill is required").
- [x] `ClassAccessHelper::CheckAccess` is skill-aware (§1.4) and `RecordOverride` writes the audit row (tested: `class_access_helper_test.cpp::RecordOverrideWritesAudit`).
- [x] **Blocking gate wired into `BookingHelper::BookEvent`.** Restructured the old recurring-only short-circuit into a unified gate: for any session tied to a class, evaluate `CheckAccess`. If `!allowed` → reject with `kErrorMissingRequirements` (`MISSING_SKILL_REQUIREMENTS`), carrying `failedRequirementLabels` (the failed group labels — skill/membership names). When the gate passes, a recurring class still returns `NO_ADVANCE_BOOKING_REQUIRED` (P-1). Consistent with the catalog/detail (`ClassCatalogHelper`), which already marks `!allowed` classes unavailable — there is no "pay to bypass" path, so no existing paid-drop-in behavior breaks (classes with no requirement groups stay open).
- [x] **Override path.** `BookEventRequest` gained `staffOverride` / `actingPersonId` / `overrideReason`. When the gate fails and `staffOverride` is set (authorized at the endpoint), the bypass is logged via `ClassAccessHelper::RecordOverride(...)` (NULL booking id) and booking proceeds. For a recurring class the override is audited and still returns `NO_ADVANCE_BOOKING_REQUIRED`.
- [x] **Endpoint (`book_event.cpp`).** Parses `staff_override` + `override_reason`; only honors the override when the caller is admin or has `manage_class_schedule` (`Session::IsAdmin` / `ActiveUserHasPermission`) — a raw flag from an unprivileged user is ignored. Maps `MISSING_SKILL_REQUIREMENTS` → **403** with a `missing_requirements` array extension so the UI can show "You're missing: X, Y" and (for staff) offer the override.
- [ ] For the staff-check-in flow (Phase 8) the same gate runs at the door — deferred to Phase 8.
- [x] Tests. **`booking_helper_test.cpp`**: blocks when missing skill (+ failed labels), allows when skill held, staff override bypasses + writes audit row (series), recurring missing-skill blocked (not "just show up"), recurring override → `NO_ADVANCE` + audit. **`book_event_test.cpp`**: 403 + `missing_requirements` for under-qualified user, staff override (with `manage_class_schedule`) succeeds, override flag without permission still 403.

### 4.3 KeyValueTable conversions
- [x] In `business_logic/skills/skill_key_value_table.h/.cpp`:
  - [x] `SkillLevelToKeyValueTable(const SkillLevelInfo&)` + `SkillLevelsToKeyValueTableArray(...)`
  - [x] `PersonSkillInfoToKeyValueTable(const PersonSkillInfo&)` + `PersonSkillInfosToKeyValueTableArray(...)`
  - ~~`ClassRequirementsCheckToKeyValueTable(...)`~~ — superseded; the gate result is surfaced by `ClassAccessHelper`'s existing `AccessResult` conversion.
- [x] Tests `skill_key_value_table_test.cpp` (both single + array converters, active/inactive + photo flags, empty arrays).

## 5. Endpoints

### 5.1 Public / logged-in endpoints ✅
- [x] `endpoints/get_skill_levels.h/cpp` + test:
  - `GET /api/skill_levels` (public) — list **active only** (`is_active=true`) skill levels, ordered by `sort_order`; each item carries `has_photo` (frontend builds `/api/scaled_photo/skill_levels/<id>`). Tests: empty, active-only filter + ordering explicitly.
- [x] `endpoints/get_skill_level_detail.h/cpp` + test:
  - `GET /api/skill_levels/<id>` (public) — single detail; returns the row regardless of `is_active` (so a class detail page can resolve a deactivated prerequisite's name). Tests: found, 404 missing, 400 zero-id, inactive-still-returned.
- [x] `endpoints/get_my_skills.h/cpp` + test:
  - `GET /api/me/skills` (login required) — the logged-in user's active grants (excludes revoked). Tests: 401 anon, active-excluding-revoked, empty.

### 5.2 Staff endpoints ✅
- [x] `endpoints/staff_get_person_skills.h/cpp` + test:
  - `GET /api/staff/person/<id>/skills` — gated by `manage_skills`. Tests: 401 anon, 403 no-permission, 200 with skills, 400 zero-id.
- [x] `endpoints/staff_assign_skill.h/cpp` + test:
  - `POST /api/staff/person/<id>/skill/<skillId>` body `{ note?: string }`, gated by `manage_skills`; assigner = session person id. Idempotent. Tests: 403, success, idempotent (no dup), 404 missing-skill, 404 missing-person, 400 zero-ids.
- [x] `endpoints/staff_revoke_skill.h/cpp` + test:
  - `DELETE /api/staff/person/<id>/skill/<skillId>` body `{ reason?: string }`, gated by `manage_skills`; returns `{ revoked: bool }` (idempotent no-op when no active grant). Tests: 403, revokes active grant, no-op, 400 zero-ids.
  - Note: assign (POST) and revoke (DELETE) share the path with different methods — verified Crow supports this (cf. `admin_card_actions.cpp` GET+POST on one path).

### 5.3 Admin endpoints for class requirements — `[~]` SUPERSEDED (do not build)
> ❌ **Superseded per §1.4.** Class skill requirements are authored through the existing **requirement-group editor endpoints** (from the permission-based-access redesign), adding/removing skill literals on a group. No skill-specific admin endpoints are added.
- [~] ~~`endpoints/admin_set_class_skill_requirement.h/cpp`~~
- [~] ~~`endpoints/admin_remove_class_skill_requirement.h/cpp`~~
- [x] **No new endpoint needed.** Per `admin_class_requirements_list.h`, requirement-group literal writes already reuse the **generic admin CRUD endpoints** against `class_requirement_group_literals` (gated by `manage_class_schedule`); `skill_level_id` is just a nullable column on that table, so authoring a skill literal is an `add_item`/`delete_item` call. The remaining wiring is the **admin column metadata** for `skill_level_id` (§7), not an endpoint.

### 5.4 Routing + permission ✅
- [x] All six registered in `web_app.cpp` (includes + reference variables).
- [x] New permission `manage_skills` introduced (`db_schema/permissions.h::kPermissionManageSkills`), seeded in `create_database.cpp` `PopulatePermissions` (id 10) and granted to the **admin** and **Studio Manager** roles in `PopulateRolePermissions`. (There is no distinct "Staff" role in the current model — staff endpoints elsewhere gate on the `staff_access` permission; skill management gets its own `manage_skills` granted to the manager-tier roles. Admin inherits via its master set.)

## 6. Frontend

### 6.1 User portal — `/my/account/skills` ✅
- [x] `ui/src/app/pages/account/my-skills/my-skills.component.ts/.html/.scss/.spec.ts`; route `skills` registered in `account.routes.ts`.
- [x] Grid of bordered badge cards: photo (when `has_photo`, `/api/get_scaled_photo/skill_levels/<id>/...`), name, "Earned {date}" (`assigned_us` → `formatCalendarDate`), description, note. Loading + error states.
- [x] Empty state: "You don't have any skill levels yet. Talk to a staff member to get evaluated."
- [x] Standard back-nav / `din-condensed-bold` title / `RouterTestingModule` spec per `feedback_account_page_layout`.

### 6.2 Class detail page — skill section ✅
- [x] In the existing public class-detail component, added a "Required skills" section (gated on `required_skills.length > 0`). Each requirement is a `mat-chip`; when logged in, a green `check_circle` (held) / red `cancel` (missing) avatar icon (case-insensitive trimmed name match vs `getMySkills()`).
- [x] All held → green "You meet the prerequisites" banner.
- [x] Missing → red "You're missing: X, Y" banner + "Talk to staff for evaluation" CTA.
- [x] Held/missing computed in `.ts` as a `ClassRequirementsCheck`. Auth via `AuthService.authData.isAuth`. Spec extended (chips, both banners, not-logged-in bare chips, case-insensitive match).

### 6.3 Staff portal — person skill management ✅
- [x] `ui/src/app/pages/staff/person-skills/person-skills.component.*` (followed the actual `pages/staff/` convention, not the nominal `portal/staff/`) + route in `staff.routes.ts` + a dashboard card.
- [x] Person search reuses the existing `staffSearchPeople` (`/api/staff/people/search`), debounced autocomplete.
- [x] Selecting a person → table of `getPersonSkills` (name, assigned date, note); "Assign new skill" → `AssignSkillDialogComponent` (skill `mat-select` from `getSkillLevels`, excludes held; optional note) → `assignSkill` + refetch.
- [x] Per-row "Revoke" → `RevokeSkillDialogComponent` (confirm + reason) → `revokeSkill` + refetch.
- [x] Specs for the page + both dialogs (search, select, assign flow, revoke flow, empty state, cancel no-ops).

### 6.4 Admin — class requirements editor extension ✅
- [x] In `class-requirements-editor.component`, added a per-group **skill-literal picker** (`mat-select` from active `getSkillLevels()`, excludes already-used skills) beside the existing permission autocomplete.
- [x] Adding writes a `class_requirement_group_literals` row via the same generic CRUD path with `skill_level_id` set (no `permission_id`); removal uses the same delete-by-id path.
- [x] Skill literals render as a distinct amber `military_tech` chip using `skill_level_name` (falls back to a lookup in the loaded skill list). Spec extended (load/sort/active-filter, add with skill_level_id only, render by name, name fallback, remove, permission path still green).

### 6.5 `ServerAccess` extensions ✅
- [x] `getSkillLevels()`, `getSkillLevelDetail(id)`, `getMySkills()`, `getPersonSkills(personId)`, `assignSkill(personId, skillLevelId, note?)`, `revokeSkill(personId, skillLevelId, reason?)` — added to the interface (`types/ServerAccess.ts`), proxy (`network/ServerAccess.ts`), real impl (`ServerAccessNetwork.ts`, with `is_active`/`has_photo` string→bool coercion), and mock (`ServerAccess.mock.ts`, in-memory state).
- [x] ~~`setClassSkillRequirement(...)`~~ — superseded per §1.4 (authored via the requirement-group editor's generic CRUD).
- [x] `ServerAccess.mock.spec.ts` updated (active-filter/order, detail found/inactive/404, my-skills + 401, assign + idempotent + 404, revoke + no-op).

### 6.6 Types ✅
- [x] `ui/src/app/shared/types/skill.types.ts`: `SkillLevel`, `PersonSkill`, `AssignSkillResult`, `RevokeSkillResult`, plus UI helpers `SkillRequirementStatus` + `ClassRequirementsCheck` (re-exported from `@shared/types/ServerAccess`).

## 7. Admin Metadata

- [ ] `skill_levels` → `admin_top_level_tables`.
- [ ] `skill_level_assignments` → `admin_nested_tables` under `people` keyed by `person_id`.
- [~] ~~`class_skill_requirements` → `admin_nested_tables`~~ — superseded per §1.4 (no such table).
- [ ] Permissions: all gated by `manage_skills`.
- [ ] Column data info, friendly names, table friendly names, display templates.
- [ ] Photo support in `photo_support_tables`.

## 8. Tests-Required Summary

- [x] Table helpers: two new `*_test.cpp` files (`skill_levels`, `skill_level_assignments`) ✅ (§3). (Partial-unique-index is prod-only per §2.2; `class_skill_requirements` helper dropped per §1.4.)
- [x] Business logic (Phase 4): `skill_level_helper_test.cpp` (assign/revoke/idempotency/read-skills/catalog reads) + `skill_key_value_table_test.cpp` (conversions) ✅. `class_access_helper_test.cpp` skill-literal branch ✅ (§1.4). `booking_helper_test.cpp` reject/override branches ✅ (OQ-P3-3 option a — block missing skill, allow when held, staff override + audit, recurring blocked / recurring override). `book_event_test.cpp` 403 + override-success + override-without-permission ✅.
- [x] Endpoint tests for all six new endpoints (3 public/logged-in §5.1 + 3 staff §5.2; success + permission-denied + validation-error) ✅. §5.3 needs no endpoint (generic CRUD), so no extra endpoint test.
- [x] Frontend specs for `my-skills`, class-detail skill section, person-skills staff page (+ both dialogs), and the requirements-editor skill-literal picker (§6.4). ✅
- [x] `ServerAccess.mock.spec.ts` updated. ✅
- [ ] Seed data: two demo skill levels with photos pending upload, one demo requirement modeled as a skill literal in a requirement group on the "Aerial 101" class.

## 9. Cross-Layer Acceptance Criteria

- [ ] Staff in the staff portal can search for a member, see they have zero skills, click "Assign", pick "Inversions (Wall Handstand)", add a note "Demonstrated cleanly on Mar 5", save, and see the row appear immediately.
- [ ] The member can log in to `/my/account/skills` and see the new badge with the assigned date.
- [x] An attempt by that member to book the "Aerial 2 Workshop" (which requires the "Aerial Basics" skill they don't have) returns `MISSING_SKILL_REQUIREMENTS` (HTTP 403, with the failed requirement labels). ✅ §4.2
- [x] Staff with `manage_class_schedule` can submit the same booking with `staff_override=true` + reason "verified live in person" and have it accepted (logged via `RecordOverride`). ✅ §4.2
- [ ] If staff later revokes the original skill ("re-evaluated after months off"), the member's `/my/account/skills` page no longer shows it.

## 10. Open Questions — ALL RESOLVED

### Resolved (2026-06-03)

- [x] **OQ-P3-3 (resolved — option a).** The blocking skill gate lives in `BookingHelper::BookEvent` / `book_event.cpp` (§4.2): a person cannot book a class they don't qualify for (skill/membership) absent an admin/`manage_class_schedule` staff override with a logged reason; the booking UI surfaces the failed requirements (`missing_requirements` array on the 403). The Phase-8 check-in flow runs the same gate at the door. Mason: "A person shouldn't be able to sign up for a given class if they don't have the skill without admin / staff override. It should show up in the UI that the skill is required."

- [x] **OQ-P3-SKILL (resolved).** Where do skill requirements live? → **Modeled as skill literals in `class_requirement_group_literals`**, evaluated by the shared `ClassAccessHelper`. The dedicated `class_skill_requirements` table/helper/check (§2.3, §3.3, §4.1's `PersonMeetsClassRequirements`) are superseded. See §1.4.
- [x] **OQ-P3-1 (resolved — no).** Revoking a skill does **not** auto-cancel the user's existing future paid bookings for classes that required it. The user keeps the booking; if there's a genuine safety concern, staff cancels manually with a voucher (BC-6). Less invasive.
- [x] **OQ-P3-2 (resolved — active only).** `GET /api/skill_levels` returns **only `is_active=true`** skill levels. (Admin/staff still see inactive ones via the admin table views.) See §5.1.

## 11. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 3.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]].
- Feeds into: [[Classes Phase 5 - Attendance Templates]] (eligible-classes grid filters by skill), [[Classes Phase 8 - Staff Check-in]] (override at the door).
- Photo infrastructure: [[Adding support for images]].
