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

### 4.1 `SkillLevelHelper` (in `business_logic/auth/` or a new `business_logic/skills/`)
- [ ] Place in `business_logic/skills/skill_level_helper.h/.cpp` for clarity (mirrors `business_logic/auth/`, `business_logic/payment/`).
- [ ] Methods:
  ```cpp
  struct AssignSkillRequest { int64_t personId; int64_t skillLevelId; int64_t assignerPersonId; std::string note; };
  struct AssignSkillResult { bool ok; int64_t assignmentId; std::string errorCode; };
  AssignSkillResult AssignSkill(Transaction&, const AssignSkillRequest&);

  struct RevokeSkillRequest { int64_t personId; int64_t skillLevelId; int64_t revokerPersonId; std::string reason; };
  bool RevokeSkill(Transaction&, const RevokeSkillRequest&);

  struct PersonSkillInfo { int64_t skillLevelId; std::string code; std::string name; std::string photoUrl; int64_t assignedUs; std::string note; };
  std::vector<PersonSkillInfo> GetPersonSkills(Transaction&, int64_t personId);
  ```
- [ ] `AssignSkill` is idempotent — if person already has the active skill, returns ok with the existing id; does not insert duplicate. Auto-populates `assigned_us = now`.
- [ ] `RevokeSkill` flips `removed_us = now`, sets `removed_by_person_id`, `removed_reason`. No-op if not currently active.
- [ ] All return-value structs surface enough info to render the UI without a follow-up call.

> ❌ **`PersonMeetsClassRequirements` / `ClassRequirementsCheck` are SUPERSEDED per §1.4.** The class-requirements check is owned by the shared `Scheduling::ClassAccessHelper::CheckAccess`, extended in §1.4 required-work to evaluate skill literals. `SkillLevelHelper` exposes only assign / revoke / read-skills; it does **not** evaluate class gates.
- [ ] **Extend `ClassAccessHelper::CheckAccess` to evaluate skill literals** (the §1.4 required-work item — currently `class_access_helper.cpp:57` skips them): a skill literal is satisfied iff the viewer holds an active `skill_level_assignments` row (`removed_us IS NULL`) for that `skill_level_id`. Also populate `skillLevelName` in the view/read path (`class_access_helper.cpp:128`, currently left empty "until Phase 3"). Add tests to `class_access_helper_test.cpp` for the skill-literal branch (held / not-held / mixed with permission literals). **Sequence after §2 + §3.2.**

### 4.2 Booking-flow integration
> Per §1.4: the gate is the shared `Scheduling::ClassAccessHelper`, not a standalone skill check. This section now wires the **skill side** of the already-present gate rather than adding a new one.
- [ ] In `BookingHelper::BookEvent` (which Phase 2 already adjusted): the booking gate already calls `ClassAccessHelper::CheckAccess`. Once §4.1's skill-literal evaluation lands, skill requirements are enforced automatically — no new call site is added here.
- [ ] If not met:
  - Default path: return 400 `MISSING_SKILL_REQUIREMENTS`, surfacing the failed group labels from `AccessResult` (which now include skill-level names) in the response.
  - Override path: if request body has `staff_override = true` AND the caller has `manage_classes` permission, bypass and record the override via the existing `ClassAccessHelper::RecordOverride(...)` (writes `booking_requirement_overrides`) — do **not** invent a second override record (per §1.4).
- [ ] For the staff-check-in flow (Phase 8) the same gate runs — staff sees the warning + a "Yes, allow anyway with reason: ___" prompt, recorded through `RecordOverride`.
- [ ] Tests (extend `booking_helper_test.cpp` and `class_access_helper_test.cpp`) cover: meets all, missing one, missing all, staff override accepted + audit row written, non-staff override rejected.

### 4.3 KeyValueTable conversions
- [ ] In `business_logic/skills/skill_key_value_table.h/.cpp`:
  - `SkillLevelToKeyValueTable(const SkillLevelInfo&)`
  - `PersonSkillInfoToKeyValueTable(...)`
  - ~~`ClassRequirementsCheckToKeyValueTable(...)`~~ — superseded; the gate result is surfaced by `ClassAccessHelper`'s existing `AccessResult` conversion.
- [ ] Tests.

## 5. Endpoints

### 5.1 Public / logged-in endpoints
- [ ] `endpoints/get_skill_levels.h/cpp` + test:
  - `GET /api/skill_levels` — list **active only** (`is_active=true`) skill levels with photo URLs (resolved OQ-P3-2). Test the active-filter explicitly.
- [ ] `endpoints/get_skill_level_detail.h/cpp` + test:
  - `GET /api/skill_levels/<id>` — single detail.
- [ ] `endpoints/get_my_skills.h/cpp` + test:
  - `GET /api/me/skills` — logged-in user's active skill assignments + assigned dates.

### 5.2 Staff endpoints
- [ ] `endpoints/staff_get_person_skills.h/cpp` + test:
  - `GET /api/staff/person/<id>/skills` — gated by `manage_skills` or `staff` role.
- [ ] `endpoints/staff_assign_skill.h/cpp` + test:
  - `POST /api/staff/person/<id>/skill/<skillId>` body `{ note?: string }`.
- [ ] `endpoints/staff_revoke_skill.h/cpp` + test:
  - `DELETE /api/staff/person/<id>/skill/<skillId>` body `{ reason: string }`.

### 5.3 Admin endpoints for class requirements — `[~]` SUPERSEDED (do not build)
> ❌ **Superseded per §1.4.** Class skill requirements are authored through the existing **requirement-group editor endpoints** (from the permission-based-access redesign), adding/removing skill literals on a group. No skill-specific admin endpoints are added.
- [~] ~~`endpoints/admin_set_class_skill_requirement.h/cpp`~~
- [~] ~~`endpoints/admin_remove_class_skill_requirement.h/cpp`~~
- [ ] **If the requirement-group editor endpoints do not yet expose a skill-literal field**, extend them (and their tests) to accept `skill_level_id` on a literal. Verify against the redesign's existing endpoint before adding anything new.

### 5.4 Routing + permission
- [ ] All registered in `web_app.cpp`.
- [ ] New permission `manage_skills` introduced. Add to `Studio Manager` and `Staff` roles; admin already has the master set.

## 6. Frontend

### 6.1 User portal — `/my/account/skills`
- [ ] `ui/src/app/pages/account/my-skills/my-skills.component.ts/.html/.scss/.spec.ts`.
- [ ] Grid of badge cards: photo, name, "Earned {date}", description.
- [ ] Empty state: "You don't have any skill levels yet. Talk to a staff member to get evaluated."
- [ ] Use the standard back-nav / title-font / RouterTestingModule pattern per memory `feedback_account_page_layout.md`.

### 6.2 Class detail page — skill section
- [ ] In the existing Phase 1 class-detail component, add a "Required skills" section. Each requirement is a chip with a `you-have` / `you-don't-have` icon if logged in.
- [ ] If user has all → show a green "You meet the prerequisites" banner.
- [ ] If missing → show a red "You're missing: X, Y" banner with a "Talk to staff for evaluation" CTA.
- [ ] Spec.

### 6.3 Staff portal — person skill management
- [ ] `ui/src/app/pages/portal/staff/person-skills/person-skills.component.*` + spec.
- [ ] Person search at the top (autocomplete) — reuse the existing `/api/staff/people/search` if it exists; if not, create one (it's also needed for Phase 8 check-in).
- [ ] Selecting a person populates a table of their current skills + a button "Assign new skill" → dialog with skill picker + note field.
- [ ] Each row has a "Revoke" button → confirm + reason field.
- [ ] Spec.

### 6.4 Admin — class requirements editor extension
> Per §1.4: skill requirements are skill literals in requirement groups. Authoring happens in the **§6.6 Phase-1 Requirements editor**, not a standalone multi-select.
- [ ] In the existing requirement-group editor (Phase-1 / permission-based-access redesign), add a **skill-literal picker** so a literal can reference a skill level (populated from `getSkillLevels()`) alongside the existing permission literals.
- [ ] Render skill literals with the skill name (and photo if available) in the group display.
- [ ] Spec.

### 6.5 `ServerAccess` extensions
- [ ] `getSkillLevels()`, `getSkillLevelDetail(id)`, `getMySkills()`, `getPersonSkills(personId)`, `assignSkill(personId, skillLevelId, note)`, `revokeSkill(personId, skillLevelId, reason)`.
- [ ] ~~`setClassSkillRequirement(...)` / `removeClassSkillRequirement(...)`~~ — superseded per §1.4; class skill requirements are authored via the existing requirement-group editor's `ServerAccess` methods (extend those for a skill-literal field if needed).
- [ ] Update `ServerAccess.mock.spec.ts`.

### 6.6 Types
- [ ] `ui/src/app/shared/types/skill.types.ts`: `SkillLevel`, `PersonSkill`, `ClassRequirementsCheck`.

## 7. Admin Metadata

- [ ] `skill_levels` → `admin_top_level_tables`.
- [ ] `skill_level_assignments` → `admin_nested_tables` under `people` keyed by `person_id`.
- [~] ~~`class_skill_requirements` → `admin_nested_tables`~~ — superseded per §1.4 (no such table).
- [ ] Permissions: all gated by `manage_skills`.
- [ ] Column data info, friendly names, table friendly names, display templates.
- [ ] Photo support in `photo_support_tables`.

## 8. Tests-Required Summary

- [ ] Table helpers: two new `*_test.cpp` files (`skill_levels`, `skill_level_assignments`) plus partial-unique-index regression. (`class_skill_requirements` helper dropped per §1.4.)
- [ ] Business logic: `skill_level_helper_test.cpp` (assign/revoke/idempotency/read-skills), extended `class_access_helper_test.cpp` for the skill-literal branch, updated `booking_helper_test.cpp` for the gate path with override (audit row) and reject branches.
- [ ] Endpoint tests for all six new endpoints (3 public/logged-in §5.1 + 3 staff §5.2; success + permission-denied + validation-error per memory `error_response_status_codes.md`). Plus tests for any skill-literal extension to the existing requirement-group editor endpoint (§5.3).
- [ ] Frontend specs for `my-skills`, class-detail skill section, person-skills staff page, and the requirements-editor skill-literal picker (§6.4).
- [ ] `ServerAccess.mock.spec.ts` updated.
- [ ] Seed data: two demo skill levels with photos pending upload, one demo requirement modeled as a skill literal in a requirement group on the "Aerial 101" class.

## 9. Cross-Layer Acceptance Criteria

- [ ] Staff in the staff portal can search for a member, see they have zero skills, click "Assign", pick "Inversions (Wall Handstand)", add a note "Demonstrated cleanly on Mar 5", save, and see the row appear immediately.
- [ ] The member can log in to `/my/account/skills` and see the new badge with the assigned date.
- [ ] An attempt by that member to book the "Aerial 2 Workshop" (which requires the "Aerial Basics" skill they don't have) returns 400 `MISSING_SKILL_REQUIREMENTS`.
- [ ] Staff with `manage_classes` can submit the same booking with `staff_override=true` + reason "verified live in person" and have it accepted.
- [ ] If staff later revokes the original skill ("re-evaluated after months off"), the member's `/my/account/skills` page no longer shows it.

## 10. Open Questions — ALL RESOLVED (2026-06-03)

- [x] **OQ-P3-SKILL (resolved).** Where do skill requirements live? → **Modeled as skill literals in `class_requirement_group_literals`**, evaluated by the shared `ClassAccessHelper`. The dedicated `class_skill_requirements` table/helper/check (§2.3, §3.3, §4.1's `PersonMeetsClassRequirements`) are superseded. See §1.4.
- [x] **OQ-P3-1 (resolved — no).** Revoking a skill does **not** auto-cancel the user's existing future paid bookings for classes that required it. The user keeps the booking; if there's a genuine safety concern, staff cancels manually with a voucher (BC-6). Less invasive.
- [x] **OQ-P3-2 (resolved — active only).** `GET /api/skill_levels` returns **only `is_active=true`** skill levels. (Admin/staff still see inactive ones via the admin table views.) See §5.1.

## 11. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 3.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]].
- Feeds into: [[Classes Phase 5 - Attendance Templates]] (eligible-classes grid filters by skill), [[Classes Phase 8 - Staff Check-in]] (override at the door).
- Photo infrastructure: [[Adding support for images]].
