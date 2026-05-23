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

## 2. Database Schema

### 2.1 `skill_levels` table
- [ ] `db_schema/skill_levels.h/.cpp`:
  - `id BIGSERIAL PK`
  - `code TEXT NOT NULL UNIQUE`  (slug, e.g. `handstand_wall`)
  - `name TEXT NOT NULL`
  - `description TEXT NOT NULL DEFAULT ''`
  - `sort_order BIGINT NOT NULL DEFAULT 0`
  - `is_active BOOLEAN NOT NULL DEFAULT TRUE`
  - `created_us`, `updated_us`
- [ ] Index on (`is_active`, `sort_order`).

### 2.2 `skill_level_assignments` table
- [ ] `db_schema/skill_level_assignments.h/.cpp`:
  - `id BIGSERIAL PK`
  - `person_id BIGINT NOT NULL REFERENCES people(id)`
  - `skill_level_id BIGINT NOT NULL REFERENCES skill_levels(id)`
  - `assigned_by_person_id BIGINT NOT NULL REFERENCES people(id)`
  - `assigned_us BIGINT NOT NULL`
  - `note TEXT NOT NULL DEFAULT ''`
  - `removed_us BIGINT` NULL
  - `removed_by_person_id BIGINT` NULL
  - `removed_reason TEXT NOT NULL DEFAULT ''`
  - `created_us`, `updated_us`
- [ ] Partial unique index: `UNIQUE (person_id, skill_level_id) WHERE removed_us IS NULL` — one active assignment per (person, skill) at a time. Re-grant after revocation creates a new row.
- [ ] Index on `person_id` for `GetActiveAssignmentsForPerson`.

### 2.3 `class_skill_requirements` table
- [ ] `db_schema/class_skill_requirements.h/.cpp`:
  - `id BIGSERIAL PK`
  - `class_id BIGINT NOT NULL REFERENCES classes(id)`
  - `skill_level_id BIGINT NOT NULL REFERENCES skill_levels(id)`
  - `required_at_signup BOOLEAN NOT NULL DEFAULT TRUE` — vs attended-but-warn (Could Have, soft-warn at check-in)
  - `created_us`
- [ ] Unique on (`class_id`, `skill_level_id`).

### 2.4 Photo support
- [ ] In `create_database.cpp`, add `skill_levels` to `photo_support_tables` so the existing photo upload / scale endpoints accept skill-level photos.

### 2.5 Wire into DB init
- [ ] `make_database_info.cpp` adds three `Make*Table()` calls in FK order.
- [ ] `create_database.cpp` `CreateTables()` adds three `CreateTable()` calls.
- [ ] CMakeLists for `db_schema/` and `sql_util/table_helpers/`.

## 3. Table Helpers

### 3.1 `TableHelpers::SkillLevels`
- [ ] `skill_levels.h/.cpp/_test.cpp`:
  - `AddSkillLevel(Transaction&, const KeyValueTable&)`
  - `GetSkillLevel(Transaction&, int64_t id)` / `GetSkillLevelByCode(...)`
  - `GetActiveSkillLevels(Transaction&)` (ORDER BY sort_order)
  - `UpdateSkillLevel(Transaction&, int64_t id, const KeyValueTable& updates)`
  - `DeleteSkillLevel(Transaction&, int64_t id)` ← soft (set `is_active=false`)
- [ ] Tests covering CRUD, soft delete, unique-code violation.

### 3.2 `TableHelpers::SkillLevelAssignments`
- [ ] `skill_level_assignments.h/.cpp/_test.cpp`:
  - `AddAssignment(Transaction&, const KeyValueTable&)`
  - `GetAssignment(Transaction&, int64_t id)`
  - `GetActiveAssignmentsForPerson(Transaction&, int64_t personId)` — `WHERE removed_us IS NULL`
  - `GetAllAssignmentsForPerson(Transaction&, int64_t personId)` — historical including revocations (audit view)
  - `GetActiveAssignment(Transaction&, int64_t personId, int64_t skillLevelId)` — for idempotency check
  - `PersonHasSkill(Transaction&, int64_t personId, int64_t skillLevelId)` → bool
  - `SetRemoval(Transaction&, int64_t id, int64_t removedByPersonId, std::string_view removedReason)` — marks the row revoked
- [ ] Tests for the partial-unique-index behavior (re-grant after revoke creates a new active row).

### 3.3 `TableHelpers::ClassSkillRequirements`
- [ ] `class_skill_requirements.h/.cpp/_test.cpp`:
  - `AddRequirement(Transaction&, ...)`
  - `RemoveRequirement(Transaction&, int64_t classId, int64_t skillLevelId)`
  - `GetRequirementsForClass(Transaction&, int64_t classId)` → list of (skillLevelId, required_at_signup)
- [ ] Tests.

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

  struct ClassRequirementsCheck { bool meetsAll; std::vector<int64_t> missingSkillLevelIds; std::vector<std::string> missingSkillNames; };
  ClassRequirementsCheck PersonMeetsClassRequirements(Transaction&, int64_t personId, int64_t classId);
  ```
- [ ] `AssignSkill` is idempotent — if person already has the active skill, returns ok with the existing id; does not insert duplicate. Auto-populates `assigned_us = now`.
- [ ] `RevokeSkill` flips `removed_us = now`, sets `removed_by_person_id`, `removed_reason`. No-op if not currently active.
- [ ] `PersonMeetsClassRequirements` joins `class_skill_requirements` against `skill_level_assignments WHERE removed_us IS NULL AND person_id = ?`. Returns `meetsAll=true` if every required skill is held; otherwise lists what's missing.
- [ ] All return-value structs surface enough info to render the UI without a follow-up call.

### 4.2 Booking-flow integration
- [ ] In `BookingHelper::BookEvent` (which Phase 2 already adjusted): after the membership-included reject check, look up `event_sessions.class_id` and call `SkillLevelHelper::PersonMeetsClassRequirements`.
- [ ] If not met:
  - Default path: return 400 `MISSING_SKILL_REQUIREMENTS` with the missing skill list in the response.
  - Override path: if request body has `staff_override = true` AND the caller has `manage_classes` permission, bypass with a logged reason (recorded in `bookings.notes`).
- [ ] For the staff-check-in flow (Phase 8) the same check runs — staff sees the warning + a "Yes, allow anyway with reason: ___" prompt.
- [ ] Tests cover: meets all, missing one, missing all, staff override accepted, non-staff override rejected.

### 4.3 KeyValueTable conversions
- [ ] In `business_logic/skills/skill_key_value_table.h/.cpp`:
  - `SkillLevelToKeyValueTable(const SkillLevelInfo&)`
  - `PersonSkillInfoToKeyValueTable(...)`
  - `ClassRequirementsCheckToKeyValueTable(...)`
- [ ] Tests.

## 5. Endpoints

### 5.1 Public / logged-in endpoints
- [ ] `endpoints/get_skill_levels.h/cpp` + test:
  - `GET /api/skill_levels` — list active skill levels with photo URLs.
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

### 5.3 Admin endpoints for class requirements
- [ ] `endpoints/admin_set_class_skill_requirement.h/cpp` + test:
  - `POST /api/admin/class/<id>/skill_requirement` body `{ skill_level_id, required_at_signup }`.
- [ ] `endpoints/admin_remove_class_skill_requirement.h/cpp` + test:
  - `DELETE /api/admin/class/<id>/skill_requirement/<skillId>`.

### 5.4 Routing + permission
- [ ] All registered in `web_app.cpp`.
- [ ] New permission `manage_skills` introduced. Add to `Studio Manager` and `Staff` roles; admin already has the master set.

## 6. Frontend

### 6.1 User portal — `/my/account/skills`
- [ ] `ui/src/app/pages/account/my-skills/my-skills.component.ts/.html/.scss/.spec.ts`.
- [ ] Grid of badge cards: photo, name, "Earned <date>", description.
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

### 6.4 Admin — class edit form extension
- [ ] In the Phase 1 class-schedules-edit component, add a "Required skill levels" multi-select using the skill-levels list.
- [ ] Spec.

### 6.5 `ServerAccess` extensions
- [ ] `getSkillLevels()`, `getSkillLevelDetail(id)`, `getMySkills()`, `getPersonSkills(personId)`, `assignSkill(personId, skillLevelId, note)`, `revokeSkill(personId, skillLevelId, reason)`, `setClassSkillRequirement(classId, skillLevelId, requiredAtSignup)`, `removeClassSkillRequirement(classId, skillLevelId)`.
- [ ] Update `ServerAccess.mock.spec.ts`.

### 6.6 Types
- [ ] `ui/src/app/shared/types/skill.types.ts`: `SkillLevel`, `PersonSkill`, `ClassRequirementsCheck`.

## 7. Admin Metadata

- [ ] `skill_levels` → `admin_top_level_tables`.
- [ ] `skill_level_assignments` → `admin_nested_tables` under `people` keyed by `person_id`.
- [ ] `class_skill_requirements` → `admin_nested_tables` under `classes` keyed by `class_id`.
- [ ] Permissions: all gated by `manage_skills`.
- [ ] Column data info, friendly names, table friendly names, display templates.
- [ ] Photo support in `photo_support_tables`.

## 8. Tests-Required Summary

- [ ] Table helpers: three new `*_test.cpp` files plus partial-unique-index regression.
- [ ] Business logic: `skill_level_helper_test.cpp` (assign/revoke/idempotency/requirements check), updated `booking_helper_test.cpp` for the requirements-gate path with override and reject branches.
- [ ] Endpoint tests for all eight new endpoints (success + permission-denied + validation-error per memory `error_response_status_codes.md`).
- [ ] Frontend specs for `my-skills`, class-detail skill section, person-skills staff page, class-edit multiselect.
- [ ] `ServerAccess.mock.spec.ts` updated.
- [ ] Seed data: two demo skill levels with photos pending upload, one demo requirement on the "Aerial 101" class.

## 9. Cross-Layer Acceptance Criteria

- [ ] Staff in the staff portal can search for a member, see they have zero skills, click "Assign", pick "Inversions (Wall Handstand)", add a note "Demonstrated cleanly on Mar 5", save, and see the row appear immediately.
- [ ] The member can log in to `/my/account/skills` and see the new badge with the assigned date.
- [ ] An attempt by that member to book the "Aerial 2 Workshop" (which requires the "Aerial Basics" skill they don't have) returns 400 `MISSING_SKILL_REQUIREMENTS`.
- [ ] Staff with `manage_classes` can submit the same booking with `staff_override=true` + reason "verified live in person" and have it accepted.
- [ ] If staff later revokes the original skill ("re-evaluated after months off"), the member's `/my/account/skills` page no longer shows it.

## 10. Open Questions

- **OQ-P3-1.** Should revoking a skill auto-cancel any future paid bookings the user has for classes that required it? Recommended: no — the user already has the booking; if there's a real safety concern, staff cancels manually with a voucher (BC-6). Less invasive.
- **OQ-P3-2.** Should the public `GET /api/skill_levels` endpoint return all skills, or filter to `is_active=true`? Recommended: filter to active.

## 11. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 3.
- Predecessors: [[Classes Phase 1 - Catalog and Schedule Authoring]], [[Classes Phase 2 - Membership-Gated Drop-In]].
- Feeds into: [[Classes Phase 5 - Attendance Templates]] (eligible-classes grid filters by skill), [[Classes Phase 8 - Staff Check-in]] (override at the door).
- Photo infrastructure: [[Adding support for images]].
