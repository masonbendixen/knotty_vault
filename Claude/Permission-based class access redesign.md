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

# 3. Corrective work (undo the Phase 2 binary flag)

Lowest layer first.

### 3.1 Database
- [ ] **Remove `products.is_membership_included`** — constant in `db_schema/products.h`, the DDL column in `products.cpp`, and the admin metadata in `create_database.cpp` (`PopulateAdminColumnDataInfo` + `PopulateAdminColumnFriendlyNames`).
- [ ] **Add the access-set representation** chosen in OQ-PA-1 (option A → new `class_access_permissions` table: schema header + DDL + `make_database_info` + `create_database` CreateTables + admin metadata + table-helper). Defer if OQ-PA-1 lands on (B).

### 3.2 Business logic
- [ ] **`CatalogHelper::ResolveBestPriceForPerson` / `PersonalizedPrice`** — drop the `is_membership_included` read and the `productIsMembershipIncluded` field. Recompute `isIncluded` as: *the offering is a recurring class AND the viewer holds an access-set permission* (option A), i.e. inclusion is derived from access + class kind, not a flag. For workshops/series keep the lowest-qualifying-tier price path unchanged.
- [ ] **`GetEffectivePermissionIds`** — confirm it will also union attendance-threshold (SL-10) and skill-derived permissions once those exist (Phase 3). No change now beyond a documented seam.
- [ ] **`ClassCatalogHelper::GetClassDetail` / `GetClassesVisibleToPerson`** — replace the `productIsMembershipIncluded`-based filter with the access-set check (hold an access permission ⇒ visible/included; recurring + no access permission ⇒ hidden per M-6).
- [ ] **`BookingHelper::BookEvent` `NO_ADVANCE_BOOKING_REQUIRED` guard** — fire on *recurring class kind + access via a covering permission*, not on the flag.

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
- [ ] If OQ-PA-1 = (A), the `class_access_permissions` table + admin metadata is a Phase-1-style schema addition; note it here (the class-authoring UI should let an admin pick the access permission set per class). Mark as a Phase 1 follow-up or a shared dependency of Phase 2/3.

### 4.3 [[Classes Phase 2 - Membership-Gated Drop-In]]
- [ ] Primary cleanup site. Update the Phase 2 status block + §1.1/§2.2/§4.1/§4.3/§6.1 to the permission-set model; remove the `is_membership_included` deliverables and replace with the access-set resolution. The already-built tier-pricing path (M-5 via `product_prices`) stays — only the binary inclusion modeling changes.

### 4.4 [[Classes Phase 3 - Skill Levels]]
- [ ] Owns SL-10 (`attendance_threshold_rules` + monthly grant job — the user's "attend N classes last month → permission"), SL-5/6 (skill gate), SL-11 (predecessor), SL-12 (compose at one gate). This is where attendance-/skill-derived permissions enter `GetEffectivePermissionIds`. Add a note that these feed the same access gate this redesign defines.

### 4.5 [[Classes Phase 5 - Attendance Templates]] / [[Classes Phase 8 - Staff Check-in]] / [[Classes Phase 9 - Attendance History]]
- [ ] These produce the *attendance facts* SL-10 counts (P-5: staff-only attribution; CI-4: check-in creates the booking for membership-included recurring classes). No model change, but the SL-10 job depends on their data — note the dependency.

### 4.6 [[Classes Phase 7 - Class Series and Workshops]]
- [ ] Confirms the per-permission *pricing* path (M-2/M-5) is unaffected by this redesign — paid offerings keep `product_prices`. Only recurring-class inclusion changes.

# 5. Open Questions

- **OQ-PA-1.** Access-set representation: (A) `class_access_permissions` join table [recommended], (B) `$0 product_prices` rows, or (C) permission hierarchy? (A) handles orthogonal permissions (partner-acro) and attendance grants cleanly.
- **OQ-PA-2.** Is the access set on the **class** or the **class_instances product**? Pricing lives on the product (per the Phase-1 redesign); access could too, for consistency — but membership inclusion feels class-level. Recommend product-level for one resolution path, with the class as the catalog identity.
- **OQ-PA-3.** Do membership tiers nest (Platinum ⊇ Gold ⊇ Silver)? If yes, seed the access set / grants accordingly; partner-acro stays orthogonal regardless. Affects how many permissions each membership grants.
- **OQ-PA-4.** Should Phase 2 be reopened to do this cleanup now, or should the `is_membership_included` column be left dormant (unused) until Phase 3 lands the full permission model? Recommend cleaning it up now (small) so no code depends on the wrong abstraction.

# 6. Cross-references
- Parent: [[Classes, schedules, and attendance]] (M-1, M-5, M-6, P-1, SL-10/11/12, C-1).
- Corrects: [[Classes Phase 2 - Membership-Gated Drop-In]].
- Depends on / feeds: [[Classes Phase 3 - Skill Levels]] (attendance + skill permissions), [[Classes Phase 1 - Catalog and Schedule Authoring]] (schema home for the access set).
- Pricing context: [[Payment Design Document]], [[Product browsing and quoting endpoints]].
