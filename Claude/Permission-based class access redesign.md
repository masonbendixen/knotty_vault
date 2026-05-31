---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/29/2026
Version: 0.1
tags:
---
# Overview

This is a corrective design + planning document. It supersedes the binary
"included with membership" modeling introduced in Phase 2 and replaces it with
the **permission-based access model** that the parent doc
[[Classes, schedules, and attendance]] already specifies (M-1, M-5, M-6, P-1,
SL-10/11/12). Use this document for planning; do not prompt — record decisions
inline and put genuine unknowns under Open Questions. Leave this Overview
intact.

# 1. The problem (what Phase 2 got wrong)

Phase 2 added a binary `products.is_membership_included BOOLEAN` and a derived
binary `price_included_in_membership` / `ClassPriceInfo.isIncludedInMembership`.
**This is the wrong abstraction.** "Included" is not a property of a product —
it is a *relationship between a viewer's permissions and a class*. Concretely:

- Memberships grant **permissions**, not a boolean. The studio will sell
  Silver, Silver Partner-Acro, Gold, and Platinum memberships granting
  `knotty_yoga_silver`, `knotty_yoga_silver_partner_acro`, `knotty_yoga_gold`,
  `knotty_yoga_platinum` (via `product_entitlement_rules.grants_permission_id`).
- A class is attendable by people who **hold a permission that the class
  accepts**. The same class can be *included* for Gold + Platinum, *not
  available* to Silver, etc. — a per-permission matrix, not one bit.
- Permissions also come from **attendance achievement** (SL-10): "attend ≥ N of
  the partner-acro classes last calendar month" grants a time-boxed permission
  (e.g. `acro_club`) that some classes require. And from **skill levels**
  (SL-5/6) and **roles**. All compose at one booking-permission gate (SL-12).

So "is it included?" is *derived* at read time from: which permissions the
viewer holds (membership + attendance + skill + role) ∩ which permissions the
class accepts. A binary product flag cannot express any of this and actively
conflicts with M-1 / M-5 / P-1.

# 2. The correct model (reconciled with the parent doc)

**Permissions are the single currency of access.** A person accumulates
permissions from four sources, all already in the schema or planned:

| Source | Mechanism | Status |
|--------|-----------|--------|
| Role | `role_permissions` → `role_assignments` | exists |
| Membership | `product_entitlement_rules.grants_permission_id` on an active entitlement | exists |
| Attendance threshold | `attendance_threshold_rules` + monthly job (SL-10) | planned (§2.6 / Phase 3) |
| Skill level | skill assignment (SL-2) gates via SL-5/6 | planned (Phase 3) |

`CatalogHelper::GetEffectivePermissionIds` already unions role + membership
permissions; attendance- and skill-derived permissions plug into the same set.

**A class/offering declares which permissions it accepts.** Two facets, kept
distinct:

1. **Access set (inclusion)** — the set of permissions for which the offering is
   attendable. For a recurring class this is "the memberships that include it"
   (M-1's X, Y, Z). For a workshop/series this is "which tiers may buy it" and
   pairs with per-tier pricing.
2. **Requirements (prerequisites)** — additional gates that must *also* be
   satisfied: skill levels (SL-5/6), attendance-threshold permission (SL-10),
   same-day predecessor (SL-11). These compose at the booking gate (SL-12).

**Pricing follows P-1/M-2:**
- Recurring classes: **no drop-in price.** Access is binary by permission — your
  tier includes it or you upgrade. "Included" = you hold an access-set
  permission. No `product_prices` rows needed for recurring access.
- Workshops / series / intro: **per-permission pricing** via the existing
  `product_prices` (permission_id + amount_cents), best (lowest) qualifying tier
  wins (M-5), optional non-member tier (M-6).

**"Included with your membership" is a derived label, computed per viewer:** the
offering is a recurring class AND the viewer holds one of its access-set
permissions. There is no stored boolean.

## 2.1 Central representation decision (the key design question)

How does a class declare its **access set** of permissions? The single
`products.booking_permission_id` cannot express M-1's *set* (e.g. Partner Acro
included for `knotty_yoga_silver_partner_acro` **and** `knotty_yoga_gold` **and**
`knotty_yoga_platinum` — and `partner_acro` is orthogonal to the
silver→gold→platinum ladder, so a pure tier hierarchy doesn't cover it).

Options (see **OQ-PA-1**):

- **(A) Access-permission join table (recommended).** A new
  `class_access_permissions (class_id /* or class_instances.product_id */,
  permission_id)` — "holding any of these permissions grants access." Clean,
  matches M-1 directly, handles orthogonal permissions (partner-acro), and reuses
  the standard permission gate. `booking_permission_id` becomes a legacy
  single-permission shortcut (or is dropped for classes).
- **(B) `$0 product_prices` rows per including permission.** Express "included for
  Gold" as a `product_prices {permission_id: gold, amount_cents: 0}` row. Reuses
  existing machinery; `ResolveBestPriceForPerson` already returns 0 for a held
  permission. But it overloads "price" to also mean "access," muddies P-1 (which
  says recurring classes have *no* price), and a `$0` public row would wrongly
  read as "free for everyone."
- **(C) Permission hierarchy.** Make Platinum grant {platinum, gold, silver, …}
  so one `booking_permission_id` (lowest including tier) suffices. Elegant for a
  strict ladder but breaks on orthogonal permissions (partner-acro, attendance
  grants). Could complement (A) for the tier ladder but not replace it.

Recommendation: **(A)**, with attendance-/skill-derived permissions flowing
through the same `GetEffectivePermissionIds` union. Confirm via OQ-PA-1.

Mason- I'm leaning towards A but I'm not sure that is enough. For instance, I might require for intermediate partner acro that people have achieved skill intermediate_acro, have attended 6 beginner level partner acro classes the previous month (or the current month), AND be a gold or platinum member. I probably want some kind of admin override for this too.

## 2.2 Revised access model — hierarchy + requirement groups (per Mason's notes, 2026-05-30)

Mason's two notes refine the model in two ways that fit together neatly.

**(a) Membership tiers nest — adopt a permission-implication hierarchy (resolves OQ-PA-3 = yes).** silver ⊂ silver_partner_acro ⊂ gold ⊂ platinum, and platinum adds extra benefit permissions. Rather than hand-granting bundles, model implication as data:
- New table `permission_implications (permission_id, implies_permission_id)` — a **DAG**, not just a chain (`platinum→gold`, `platinum→benefit_x`, `gold→silver_partner_acro`, `silver_partner_acro→silver`, …).
- A membership entitlement still grants ONE permission (`product_entitlement_rules.grants_permission_id` = the tier); `GetEffectivePermissionIds` expands the **transitive closure**. Holding `platinum` ⇒ effectively holds `gold`, `silver_partner_acro`, `silver`, and platinum's benefit permissions.
- **Consequence:** "gold **or** platinum" collapses to "requires `gold`" — the hierarchy supplies the tier-OR, so per-class requirements rarely need to enumerate tiers. This makes a flat option-A set workable for the *membership* dimension, but not across requirement *types* — see (b).

**(b) Access is a conjunction of requirement types, not one flat set.** Mason's intermediate-partner-acro example requires ALL of: skill `intermediate_acro` + the attendance achievement (6 beginner partner-acro last/this month → an SL-10-granted permission) + gold-or-higher membership. So we need **AND across types**, with **OR available inside a type**.

**Recommended: requirement groups (CNF), evaluated as pure SQL — no rule-engine DSL (honors SL-12).**
- A class/offering has **0..N requirement groups**.
- Each group lists one or more **literals**; a literal is a held **permission** (role / membership / attendance-granted, closure-expanded) or a **skill level**.
- A group is **satisfied** if the viewer holds ≥1 of its literals (**OR** within a group).
- The class is **accessible** iff **every** group is satisfied (**AND** across groups) — or a logged **staff override** is present.

Mason's example → three single-literal groups:

| Group | Literal | Source |
|-------|---------|--------|
| Skill | `intermediate_acro` (skill) | SL-5/6 (Phase 3) |
| Attendance | `acro_6_beginner_recent` (permission) | SL-10 monthly job |
| Membership | `gold` (permission; platinum satisfies via the hierarchy) | tier grant + closure |

Because the hierarchy collapses tier-OR, most groups are single-literal hard requirements; multi-literal groups stay available for genuine cross-type disjunction ("`gold` **or** `staff_comp`") without a DSL. Suggested tables: `class_requirement_groups (id, class_id /* or product_id */, label)` + `class_requirement_group_literals (group_id, permission_id NULL, skill_level_id NULL)`.

**Admin override (Mason's note).** Staff with `manage_classes` can book a person past any failing group, recording who / when / why (matches SL-6 / SL-12). Per-attempt and logged — never a standing bypass. Likely a `booking_requirement_overrides` audit row (or a reason field on the booking).

**Decision (OQ-PA-5): CNF chosen now** for maximum flexibility. Groups + literals are keyed on the **class** (OQ-PA-2). (A flat AND-set was the discarded fallback.)

**Pricing is unchanged by all this.** Recurring-class access is still binary-by-requirements (no drop-in price, P-1); workshops/series keep per-permission `product_prices` (M-5). "Included for this viewer" = recurring class whose requirement groups the viewer satisfies.

# 3. Corrective work (undo the Phase 2 binary flag)

Lowest layer first.

### 3.1 Database ✅ DONE (2026-05-30) — extensively tested; C++ for Mason to build
- [x] **Remove `products.is_membership_included`** — done in §3.2 (atomic with the consumer rewiring): constant + DDL gone from `products.{h,cpp}`, admin metadata removed from `create_database.cpp`.
- [x] **Permission-implication hierarchy** — `db_schema/permission_implications.{h,cpp}` (FK×2 to permissions + unique edge + `permission_id` index), wired into `make_database_info` + `create_database` (CreateTables + indexes) + top-level admin metadata. Table helper `TableHelpers::PermissionImplications` with CRUD **and `ExpandClosure`** (recursive-CTE transitive closure; bigint-cast seeds; cycle-safe via `UNION`). Tests: chain walk, DAG branch, leaf, empty, no-edges, cycle termination. *(Seed edges = data Mason supplies.)*
- [x] **Requirement groups (CNF)** — `class_requirement_groups.{h,cpp}` (FK class + label + timestamps) + `class_requirement_group_literals.{h,cpp}` (FK group + nullable FK permission + nullable `skill_level_id` BIGINT, **no FK** — Phase-3 `skill_levels` forward ref). Wired into init + nested admin metadata (groups under `classes`, literals under groups) + `manage_class_schedule` gate. Table helpers `ClassRequirementGroups` + `ClassRequirementGroupLiterals` (literals split AddPermissionLiteral / AddSkillLiteral). Tested.
- [x] **Override audit (OQ-PA-6)** — `booking_requirement_overrides.{h,cpp}` (`booking_id` made **nullable** FK — audit subject is person/class/who/why; gate sets the booking when known) + helper `BookingRequirementOverrides`. Tested. *(Not generic-CRUD registered — written by the §3.2 gate.)*

### 3.2 Business logic ✅ DONE (2026-05-31) — extensively tested; C++ for Mason to build
- [x] **`GetEffectivePermissionIds` — closure-expanded** over `permission_implications` (now **public** so the gate consumes the same set). Makes tier-OR / "platinum ⇒ gold" automatic for both pricing and the gate. Covered by `catalog_helper_test::ResolveBestPriceClosureGrantsTierViaHierarchy` (platinum holder qualifies for the gold price via closure) plus the `permission_implications` closure tests in §3.1.
- [x] **`ClassAccessHelper`** (`business_logic/scheduling/class_access_helper.{h,cpp}`) — `CheckAccess(classId, personId)` evaluates the CNF groups (AND across groups, OR within), closure-aware; no groups ⇒ open. **Skill literals are not yet evaluated** (Phase-3 `skill_levels`) — documented; permission literals fully work. `RecordOverride(...)` writes the audit row (joins failed-group labels). New `class_access_helper_test.cpp`: open class, single-group gate, closure-satisfies, CNF AND, OR-within-group, override audit.
- [x] **`CatalogHelper::ResolveBestPriceForPerson` / `PersonalizedPrice`** — `is_membership_included` + `isIncluded` + `productIsMembershipIncluded` removed; now pure lowest-qualifying-tier pricing (closure-aware). `PersonHoldsPermission` removed (dead after the flag).
- [x] **`ClassCatalogHelper::GetClassDetail` / `GetClassesVisibleToPerson`** — rewired to the access gate + `classes.kind`: recurring + passes gate ⇒ included (no drop-in price, P-1); workshop/series + passes gate ⇒ tier price via `ResolveBestPriceForPerson`; fails gate ⇒ members-only/hidden (M-6). Phase-2 flag tests rewritten to use requirement groups; Phase-1 no-group fixtures stay visible (open).
- [x] **`BookingHelper::BookEvent` `NO_ADVANCE_BOOKING_REQUIRED` guard** — now fires on *recurring `classes.kind` + viewer passes the access gate* (via a `ClassAccessHelper` member), not the flag. Tests rewritten (recurring+gated rejects; series still books).

### 3.3 Frontend
- [ ] Rename/redefine `ClassDetail.price_included_in_membership` → keep the derived semantic ("included for this viewer") but ensure it is computed server-side from the access model, not a product flag. `price_is_available` stays (members-only when the viewer holds no access permission and there's no purchasable tier). Update `class-detail` copy + specs accordingly.

### 3.4 Tests
- [ ] Update `catalog_helper_test`, `class_catalog_helper_test`, `booking_helper_test`, `scheduling_key_value_table_test`, and the frontend class-detail / mock specs to the permission-set model (replace the `is_membership_included`-flag fixtures with access-permission fixtures).

# 4. Per-document impact

### 4.1 Parent — [[Classes, schedules, and attendance]]
- [ ] **C-1**: drop "included-with-membership flag" wording; a class declares an **access permission set** (+ skill requirements). Keep "list of allowed booking permissions."
- [ ] **M-1**: reaffirm as a *set* of membership permissions ("included in memberships X, Y, Z"), realized via the access set — explicitly not a boolean.
- [ ] Add a cross-reference to this redesign in §2.4 and the §1 "what exists" list.

### 4.2 [[Classes Phase 1 - Catalog and Schedule Authoring]]
- [ ] The `class_requirement_groups` + `class_requirement_group_literals` + `permission_implications` tables are Phase-1-style schema additions (class-level, OQ-PA-2). The class-authoring UI gains a "Requirements" editor: add groups, add permission/skill literals to each group. Mark as a Phase 1 follow-up / shared dependency of Phase 2/3.

### 4.3 [[Classes Phase 2 - Membership-Gated Drop-In]]
- [ ] Primary cleanup site. Update the Phase 2 status block + §1.1/§2.2/§4.1/§4.3/§6.1 to the permission-set model; remove the `is_membership_included` deliverables and replace with the access-set resolution. The already-built tier-pricing path (M-5 via `product_prices`) stays — only the binary inclusion modeling changes.

### 4.4 [[Classes Phase 3 - Skill Levels]]
- [ ] Owns SL-10 (`attendance_threshold_rules` + monthly grant job — Mason's "attend N classes last month → permission"; note the "current OR previous month" wording in Mason's example → the rule/job needs a configurable window, see OQ-PA-7), SL-5/6 (skill gate), SL-11 (predecessor), SL-12 (one gate). These attendance-/skill-derived permissions enter the closure-expanded `GetEffectivePermissionIds` and the shared access gate (§3.2). The requirement-group model + access gate likely *lands here* (or is shared infra between Phase 2/3) since it's the home of prerequisites — the `class_requirement_groups` skill literals are exactly SL-5.

### 4.5 [[Classes Phase 5 - Attendance Templates]] / [[Classes Phase 8 - Staff Check-in]] / [[Classes Phase 9 - Attendance History]]
- [ ] These produce the *attendance facts* SL-10 counts (P-5: staff-only attribution; CI-4: check-in creates the booking for membership-included recurring classes). No model change, but the SL-10 job depends on their data — note the dependency.

### 4.6 [[Classes Phase 7 - Class Series and Workshops]]
- [ ] Confirms the per-permission *pricing* path (M-2/M-5) is unaffected by this redesign — paid offerings keep `product_prices`. Only recurring-class inclusion changes.

# 5. Open Questions

- **OQ-PA-1.** ~~Access-set representation (flat set A / $0 prices B / hierarchy C).~~ **Superseded by §2.2** per Mason's notes: adopt **C (permission-implication hierarchy) + requirement groups** rather than a single flat set. (B is rejected — overloads price with access.) Remaining sub-question is OQ-PA-5.
- **OQ-PA-2.** ✅ **Resolved (Mason): class-level.** Requirement groups + the access gate key on `class_id`; pricing stays on the `class_instances` product. (So the requirement-group tables below use `class_id`.)
	- Mason- This is class level. Confirmed :)
- **OQ-PA-3.** ✅ **Resolved (Mason):** tiers nest — silver ⊂ silver_partner_acro ⊂ gold ⊂ platinum, platinum adds extra benefits. Model via `permission_implications` (DAG) + closure expansion (§2.2a). Need the full grant/implication seed list from Mason (which permissions each tier implies, incl. platinum's "random benefits").
- **OQ-PA-4.** ✅ **Resolved (Mason): remove `is_membership_included` now.** §3 corrective work proceeds; the hierarchy + gate are shared infra consumed by Phase 2 (pricing/inclusion) and Phase 3 (prerequisites).
	- Mason- I'd like to remove it now.
- **OQ-PA-5.** ✅ **Resolved (Mason): CNF requirement groups now.** Tables: `class_requirement_groups (id, class_id, label)` + `class_requirement_group_literals (group_id, permission_id NULL, skill_level_id NULL)`. Access = every group has ≥1 satisfied literal (closure-aware) — pure SQL `NOT EXISTS(... AND NOT EXISTS(...))`. (The "AND-sets" fallback is dropped.)
	- Mason- I'd like the CNF model now for maximum flexibility.
- **OQ-PA-6.** ✅ **Resolved (Mason → recommendation): dedicated `booking_requirement_overrides` audit table, gated by `manage_classes`.** (a) over (b).
	- Mason- Can you explain what you are asking in more detail? Maybe use an example.
	- **Clarification (what the override is + the actual question).** The access gate (the CNF groups) will *block* a booking when a person doesn't satisfy every group. The override is the "but staff can let them in anyway" escape hatch (SL-6 / SL-12). Two things to decide:
		- **(1) How do we record an override?** *Example:* Intermediate Partner Acro needs `intermediate_acro` (skill) **AND** `acro_6_recent` (attendance) **AND** `gold` (membership). Dana attended only 4 beginner classes last month (no `acro_6_recent`) and is `silver_partner_acro` (not `gold`), so the gate blocks them. Head coach Pat judges Dana ready and books them anyway. We want a durable record that this bypass happened. Options:
			- **(a) Dedicated audit table** `booking_requirement_overrides (booking_id, person_id, class_id, overridden_by_person_id, reason, failed_groups, created_us)` — e.g. `{booking 812, Dana, Intermediate Partner Acro, Pat, "ready — coach approved", "attendance, membership", …}`. Queryable: "all overrides last month," "everyone Pat let in," "which gates get bypassed most."
			- **(b) Two columns on `bookings`** — `requirement_override_by` + `requirement_override_reason`. Simpler, no new table, but you lose the structured "which groups were skipped" and it clutters the bookings row.
		- **(2) Who may override?** Which permission must the staff member hold — `manage_classes` (the class-management permission)? Or something else?
		- *Why it matters:* these gates exist partly to prevent gaming (P-5), so an auditable trail of who bypassed what and why is valuable — hence the recommendation of (a) + `manage_classes`. **Decision needed: (a) audit table or (b) bookings columns; and the gating permission.**
		- Mason- I'll go with your recommendation.
- **OQ-PA-7.** ✅ **Resolved (Mason):** an earned grant is valid for the **current month + the next month**; a person earns it by hitting the threshold on **either last-month or current-month** attendance (community-participation incentive). *Implementation note (Phase 3 / SL-10):* "reap benefits immediately" means the count can't only be evaluated at month-end — re-evaluate on each staff check-in (the attendance event, P-5) and grant the permission through end of next month the moment the running current-month count crosses the threshold; a month-rollover job still handles people who qualified last month but haven't attended yet this month. The grant is refreshed each month they keep qualifying and lapses after a month without hitting the threshold.
	- Mason- I would like it to cover the current month plus the next month. Partner acro very much is dependent on community participation. Hence wanting to let people into the intermediate program who attended more than a certain number of classes last month. However, if they have participated that much this current month, I'd like them to start reaping the benefits immediately.

# 6. Status & build order (2026-05-30)

**All open questions are resolved** (PA-2 class-level · PA-3 nested hierarchy · PA-4 remove flag now · PA-5 CNF groups · PA-6 audit table + `manage_classes` · PA-7 current+next-month grant). The plan is implementation-ready. The only outstanding item is **seed data, not design**: the exact `permission_implications` edges + which classes require which permissions/skills (Mason to supply; can be filled in as the studio finalizes tiers).

Suggested build order (bottom-up, shared infra first; this redesign owns the **infra**, while Phase 3 owns the SL-10 attendance job and skill literals):
1. **Schema** — `permission_implications`; `class_requirement_groups` + `class_requirement_group_literals`; `booking_requirement_overrides`. Wire `make_database_info` + `create_database` (CreateTables + admin metadata, nested under `classes`). Remove `products.is_membership_included` + its admin metadata.
2. **Table helpers** — CRUD for the new tables + tests.
3. **`GetEffectivePermissionIds` closure expansion** over `permission_implications` (recursive CTE) + tests — the linchpin.
4. **`ClassAccessHelper` access gate** (CNF evaluation, closure-aware, staff-override path writing the audit row) + tests.
5. **Rewire consumers** to the gate — `CatalogHelper::ResolveBestPriceForPerson` (drop the flag), `ClassCatalogHelper` visibility/inclusion, `BookingHelper::BookEvent` guard — + update their tests.
6. **Frontend** — derive inclusion/members-only from the gate result; update `class-detail` + specs. (Admin "Requirements" editor UI can be a Phase-1 follow-up.)
7. **Doc reconciliation** — apply §4 edits to the parent doc + Phase 1/2/3 docs.

SL-10's attendance-threshold job, the skill-level tables (SL-5/6), and the "Requirements" authoring UI are **Phase 3 / Phase 1** deliverables that plug into this infra — not part of the immediate cleanup.

# 7. Cross-references
- Parent: [[Classes, schedules, and attendance]] (M-1, M-5, M-6, P-1, SL-10/11/12, C-1).
- Corrects: [[Classes Phase 2 - Membership-Gated Drop-In]].
- Depends on / feeds: [[Classes Phase 3 - Skill Levels]] (attendance + skill permissions), [[Classes Phase 1 - Catalog and Schedule Authoring]] (schema home for the access set).
- Pricing context: [[Payment Design Document]], [[Product browsing and quoting endpoints]].
