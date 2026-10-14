---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 10/14/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

The website is getting close. I'd like to modify database helper to create the actual content for the studio. Here is my current class schedule:

* Sunday
	* 10am-11am Beginner Knotty Yoga (Caleb Ault)
	* 11am-12pm Beginner Partner Acro (Mason Bendixen)
	* 12pm-1pm Intermediate Partner Acro (Mason Bendixen) (requires attending preceding class)
* Monday
	* 5pm-6pm Beginner Knotty  Yoga (Caleb Ault)
	* 6pm-7pm Knotty Yoga (Mason Bendixen)
	* 7pm-8pm Handstands (Mason Bendixen) (requires attending preceding class)
* Tuesday
	* 6pm-7pm Beginner Knotty Yoga (Gifford Cheung, Tiana Cheung)
* Wednesday
	* 5pm-6pm Beginner Knotty Yoga (Beth Cooper)
	* 6pm-7pm Knotty Yoga (Mason Bendixen)
	* 7pm-8pm Aerial Rope (Mason Bendixen) (requires attending preceding class)
* Thursday
	* 5pm-6pm Beginner Knotty Yoga (Caleb Ault)
	* 6pm-7pm Beginner Partner Acro (Mason Bendixen)
	* 7pm-8pm Intermediate Partner Acro (Mason Bendixen) (requires attending preceding class)
* Saturday
	* 10am-12pm Knotty Yoga (Mason Bendixen)

Class Descriptions
- Beginner Knotty Yoga
	- Mixture of yoga, Thai massage, strength training, aerial conditioning, and acrobatics principles. Can just be a workout or a way to build the strength, flexibility, and body awareness to do many disciplines of circus acrobatics. This class is more accessible to lower levels of fitness than regular Knotty Yoga classes.
- Knotty Yoga
	- Mixture of yoga, Thai massage, strength training, aerial conditioning, and acrobatics principles. Can just be a workout or a way to build the strength, flexibility, and body awareness to do many disciplines of circus acrobatics. This class requires a reasonable baseline level of fitness and requires instructor approval to attend. Should be able to do a pull-up and five regular pushups to attend.
- Handstands
	- Solidly intermediate level class focused on building a handstand practice. People should be able to easily kick up to a wall to attend this class and hold a belly to the wall handstand for thirty seconds to attend. Requires instructor approval to attend.
- Aerial Rope
	- Teaches fundamentals and trick combinations on aerial rope. You should be able to do three pullups and be able to do two straddle climbs starting in the air as well as having instructor approval to attend.
- Beginner Partner Acro
	- Learn fundamental partner acrobatics combinations doing a mix of standing and L-basing moves. Even beginner partner acrobatics should be preceded by a strong fitness foundation. The beginner Knotty Yoga classes are specifically designed to get people ready for this. Primarily looking for couples or bases. Instructor permission required for solo flyers. These classes serve as the warmup and conditioning for the Intermediate Partner Acro class.
- Intermediate Partner Acro
	- Builds on Beginner Partner Acro class to add more challenging moves and longer, more technical sequences. Must have a solid foot to hand, free star, and thirty second shoulder stand as well as instructor permission to attend.

Remove the other classes currently in the system.

We currently don't have the spa yet so remove the spa products from the product catalog. Create Caleb Ault as a massage provider and please set his schedule as:
- Mon 11am-3pm
- Tue 11am-4pm
- Thu 11am-4pm
- Fri 11am-3pm
- Sun 1:30pm-4pm

For memberships, keep the existing memberships and add a Knotty Yoga Silver Membership that includes the Beginner Knotty Yoga classes for $160 and a Knotty Yoga Silver Couple's Membership for $240 that includes the Beginner Knotty Yoga classes for a couple. Please make a silver membership permission and have Beginner Knotty Yoga allow people with silver or gold permission to attend.

Leave Mason and Caleb but add instructor profiles for: Beth Cooper, Gifford Cheung, and Tiana Cheung.

We need the following photos:
- Headshots for:
	- Mason Bendixen
	- Caleb Ault
	- Beth Cooper
	- Tiana Cheung
	- Gifford Cheung
- Class photos for:
	- Beginner Knotty Yoga
	- Knotty Yoga
	- Aerial Rope
	- Handstands
	- Beginner Partner Acro
	- Intermediate Partner Acro
- Massage photo
- General studio photo


Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Current state (verified against the code, 10/14/2026)

What exists today, so the plan says what CHANGES. All seed data lives in `src/database_helper/`: `seed_app_data.{h,cpp}` (the tested `namespace Seed` data seeders) and `create_database.cpp` (bootstrap ordering, admin metadata, and a few seeders that never moved).

| Area | Seeded today | Target |
|---|---|---|
| Classes | Knotty Yoga, Therapeutic Knotty Yoga, Partner Acrobatics, Tumbling, Handstands, Aerial Fabric | Beginner Knotty Yoga, Knotty Yoga, Handstands, Aerial Rope, Beginner Partner Acro, Intermediate Partner Acro |
| Slots | 5: KY Mon/Wed 18:00, Handstands Mon 19:00, Partner Acrobatics Thu 18:00 + Sun 10:00 | 14 (table in Phase 4) |
| Class access | **No requirement groups on any class**, so every recurring class is "included" and free for everyone, members or not (`ClassCatalogHelper::GetClassDetail`, P-1) | Membership + instructor-approval groups (Phase 4) |
| Skill levels | None seeded | Four "instructor approval" skills |
| Products | intro-workshop, massage, gold ×3, **4 spa products**, class-dropin | Spa gone; Silver + Silver Couple's added |
| Permissions | instructor, gold_member, platinum_fitness, … | + `silver_member` |
| People | masonbendixen@hotmail.com ("Mason2", pw `mason`, **admin**), Caleb (`caleb`, admin), Kit Williams (`kit`), masonbendixen@gmail.com (`pass`, admin) | + Beth Cooper, Gifford Cheung, Tiana Cheung |
| Instructors | Mason, Caleb | + Beth, Gifford, Tiana |
| Massage providers | Mason **and** Caleb, no availability in the base seed | Caleb only, with his weekly hours |
| Photos | Mason/Caleb profile + instructor; KnottyYoga, PartnerAcro, Handstand class photos; tier icons | + 3 headshots, 3 class photos, massage, studio |

**Facts the plan depends on:**

1. **Seed ids are still hardcoded in four places.** `PopulateProductVariants`, `PopulateProductPrices` and `PopulateProductBookingWindows` (all still in `create_database.cpp`) address products 1–9 and variants 1–7 by literal id, and `PopulateRoleAssignments` addresses people 1, 2, 4. Removing the four spa products and adding two Silver ones renumbers everything after them, so **these have to move to natural keys before any catalog change.** That is the same failure mode as the Gold→instructor permission bug found in [[Database helper initialization]] Phase 3.2, and it is still listed there as an open follow-up.
2. **The access gate already exists and is the right tool.** A class's `class_requirement_groups` are AND-ed and each group's literals are OR-ed; a literal is a permission (closure-expanded) **or** a skill level (`ClassAccessHelper::CheckAccess`). "Silver or Gold" is one group with two literals. "Instructor approval" is a skill-level literal, which is exactly what skill levels are for ("Can kick up to handstand against a wall" is the schema's own example).
3. **"Requires attending the preceding class" already exists**: `class_schedule_slots.predecessor_class_schedule_slot_id` (SL-11), seeded today for Handstands. Four slots need it now.
4. **A slot has exactly ONE instructor** (`class_schedule_slots.instructor_person_id`). Tuesday's Beginner Knotty Yoga has two (Gifford **and** Tiana). Per-session `event_session_staffing` can carry several, but only for materialized sessions, and only substitutions and shift trades write it. **Two instructors on a recurring slot is not expressible today**, which is why Phase 8 exists.
5. **Nothing extends provider availability over time.** `ScheduleTemplateHelper::GenerateAvailability` is called only by the seed and by the admin "generate" endpoint, and no scheduler job runs it. Seeded massage availability simply runs out after the generated window, which is why Phase 9 exists.
6. **Production does not pick up seed changes.** Prod was installed with `--install_schema` on 9/23 and every later deploy runs `--migrate`, which evolves the schema and never re-seeds. Phase 12 is how this content reaches the live site.
7. **Seed test data collides with the real schedule.** The `--recreate_database_test` Intro Workshop (Sat 10–11) overlaps the new Saturday Knotty Yoga (10–12) in Main Gym. The test-only Aerial Series run (Tue/Thu 19:00) overlaps Thursday Intermediate Partner Acro, and Caleb's test-only Mon–Fri 9–5 template duplicates his new real hours. Phase 7 fixes all three.
8. **`ui/src/app/pages/public/services/homepage.service.ts` is dead code.** Only its own spec imports it, and it hardcodes the old six-class list. It gets deleted rather than updated.
9. **Customers never see a requirement group's `inline_description`.** The admin field is labelled "Description (shown to members)", but no customer template renders it. A visitor sees only **"Members only — Upgrade your membership to attend."** on the class page, or the **"Requires a membership"** badge on Our Classes. Nothing tells them *which* membership, so nothing on the site says Silver covers Beginner Knotty Yoga except the Silver product's own description. The skill side does render: the class page's **"Required Skills"** section lists each skill's name and description, and the **"Skill required"** dialog shows the same. That is why the approval criteria go into the skill descriptions (D6). Phase 10.3 closes the membership half.
10. **The class page does not show "Requires attending".** The predecessor shows on the weekly **Our Classes** list (`Requires attending: {Class}`) and in the admin slot editor, not on the class detail page. No change planned; the hand-testing steps check it where it renders.

---

# Design decisions

- **D1 — This is SEED data, built on mechanisms that already exist.** Classes, slots, predecessors, requirement groups, skill levels, entitlement rules, provider templates and photos all have tested tables and helpers. The only new *mechanisms* are co-instructors (Phase 8, forced by fact 4) and rolling availability (Phase 9, forced by fact 5). Both are kept out of the content phases so the content can land, and be reviewed, without waiting on them.
- **D2 — Everything resolves by natural key.** Products by `code`, variants by `code`, price schedule by `name`, people by `email`, roles and permissions by `name`, classes by `name`, skills by `code`, rooms by `name`. No literal ids anywhere in the seed when this is done. Phase 1 finishes D5 from [[Database helper initialization]].
- **D3 — Data-driven seed tables, not call sequences.** The fourteen slots, the six classes' access rules and the photo assignments are each one `constexpr` table of structs that a loop walks. The schedule then reads like the Overview, and a test can iterate the same table the seeder does, so the expected values are not re-typed in the test.
- **D4 — Tests assert through the production READER** ([[feedback_assert_through_the_reader]]). Access is checked with `ClassAccessHelper::CheckAccess` for a Silver member, a Gold member, a non-member and an approved Gold member. Prices go through `CatalogHelper`, availability through the booking-slot reader, and instructor names on a session through `ClassCatalogHelper`. Re-reading the columns the seeder just wrote only proves the INSERT ran; the Phase 3.1 `show_on_upcoming` miss is the standing example.
- **D5 — "Silver or Gold" is a two-literal group, not a permission implication.** `permission_implications` could make `gold_member` imply `silver_member` so the group needs only Silver. But the Overview asks for exactly "silver or gold", the two-literal group says that literally in the requirements editor, and an implication would silently change what Gold means for every future Silver-gated thing. It is easy to switch later if Silver becomes a true sub-tier.
- **D6 — Instructor approval is one skill level per gated class.** Knotty Yoga, Handstands, Aerial Rope and Intermediate Partner Acro each get a skill (code `<class>_approved`) whose description carries the Overview's physical criteria. An instructor grants it on the existing skill-assignment page; the access gate does the rest. The **Beginner Partner Acro solo-flyer rule** cannot be enforced (the system has no notion of "solo"), so it lives in the class description only.
- **D7 — Membership requirement on every non-beginner class is `gold_member`** *(default for OQ-3)*. Without some membership group, a class is open and free to everyone (fact 2's corollary), which would make Gold buy nothing. Beginner Knotty Yoga takes Silver **or** Gold, as the Overview says.
- **D8 — Spa removal keeps the room-type vocabulary.** The four spa products, their variants, prices, product schedule windows, the `Spa` room and its room schedule template all go. The `spa` / `swim_spa` **room types** stay: they are reference vocabulary, harmless when unused, and "we don't have the spa *yet*" says it is coming back. This also retires the `late-night-spa` → nonexistent `gold_fitness` permission bug recorded in [[Database helper initialization]]'s follow-ups.
- **D9 — Real people never get a password from source control.** Beth, Gifford and Tiana are created with a random, unrecorded password (generated at seed time, never logged). The committed dev passwords stay only on the accounts that already have them (Mason, Caleb) and on the test-only accounts (OQ-10).
- **D10 — Photos land code-first.** `AttachSeedPhoto` already logs and skips a missing file, so the seed works before the pictures exist. Each photo is added as **file + one table row + one test expectation** in the same change (Phase 6.2), so the gate is never red waiting on a photographer.
- **D11 — Production gets the content by reinstall, not by a data migration** *(default for OQ-9)*. The site is not live (knottyyoga.com still points nowhere). An in-place migration would have to delete spa products that purchases may reference, rename classes under existing bookings, and re-key slots, all to preserve data that is only smoke tests. `--install_schema --force` plus the existing theme export/import and `set_secret` block is shorter and already verified end to end (Deploying to AWS 5.1 step 4).

---

# Target content at a glance

### Weekly class schedule (Main Gym, Knotty Yoga Studio)

`day_of_week` is `class_schedule_slots`' convention: **0 = Sunday … 6 = Saturday**. Minutes are after local midnight.

| Day | Time | Class | Instructor(s) | Start / duration | Requires attending |
|---|---|---|---|---|---|
| Sun (0) | 10:00–11:00 | Beginner Knotty Yoga | Caleb Ault | 600 / 60 | |
| Sun (0) | 11:00–12:00 | Beginner Partner Acro | Mason Bendixen | 660 / 60 | |
| Sun (0) | 12:00–13:00 | Intermediate Partner Acro | Mason Bendixen | 720 / 60 | Sun Beginner Partner Acro |
| Mon (1) | 17:00–18:00 | Beginner Knotty Yoga | Caleb Ault | 1020 / 60 | |
| Mon (1) | 18:00–19:00 | Knotty Yoga | Mason Bendixen | 1080 / 60 | |
| Mon (1) | 19:00–20:00 | Handstands | Mason Bendixen | 1140 / 60 | Mon Knotty Yoga |
| Tue (2) | 18:00–19:00 | Beginner Knotty Yoga | Gifford Cheung + Tiana Cheung | 1080 / 60 | |
| Wed (3) | 17:00–18:00 | Beginner Knotty Yoga | Beth Cooper | 1020 / 60 | |
| Wed (3) | 18:00–19:00 | Knotty Yoga | Mason Bendixen | 1080 / 60 | |
| Wed (3) | 19:00–20:00 | Aerial Rope | Mason Bendixen | 1140 / 60 | Wed Knotty Yoga |
| Thu (4) | 17:00–18:00 | Beginner Knotty Yoga | Caleb Ault | 1020 / 60 | |
| Thu (4) | 18:00–19:00 | Beginner Partner Acro | Mason Bendixen | 1080 / 60 | |
| Thu (4) | 19:00–20:00 | Intermediate Partner Acro | Mason Bendixen | 1140 / 60 | Thu Beginner Partner Acro |
| Sat (6) | 10:00–12:00 | Knotty Yoga | Mason Bendixen | 600 / **120** | |

Every predecessor ends exactly when its follower starts, which is what makes the predecessor link mean "the class that just ended". The Wednesday Knotty Yoga instructor changes from Caleb (today's seed) to Mason.

### Class access (requirement groups: AND across rows, OR inside a cell)

| Class | Membership group | Approval group (skill level) |
|---|---|---|
| Beginner Knotty Yoga | `silver_member` **or** `gold_member`: "Silver or Gold membership" | none |
| Knotty Yoga | `gold_member`: "Gold membership" | `knotty_yoga_approved` |
| Handstands | `gold_member` | `handstands_approved` |
| Aerial Rope | `gold_member` | `aerial_rope_approved` |
| Beginner Partner Acro | `gold_member` | none (solo-flyer rule is description text, D6) |
| Intermediate Partner Acro | `gold_member` | `intermediate_partner_acro_approved` |

Skill levels (name / description shown to members):

| Code | Name | Description |
|---|---|---|
| `knotty_yoga_approved` | Knotty Yoga approval | Can do a pull-up and five regular push-ups, and an instructor has approved you for Knotty Yoga. |
| `handstands_approved` | Handstands approval | Kicks up to a wall easily and holds a belly-to-the-wall handstand for thirty seconds, and an instructor has approved you for Handstands. |
| `aerial_rope_approved` | Aerial Rope approval | Can do three pull-ups and two straddle climbs starting in the air, and an instructor has approved you for Aerial Rope. |
| `intermediate_partner_acro_approved` | Intermediate Partner Acro approval | Solid foot-to-hand, free star and thirty-second shoulder stand, and an instructor has approved you for Intermediate Partner Acro. |

### Memberships

| Code | Name | $/month | Seats | Grants | Tier icon |
|---|---|---|---|---|---|
| `silver-membership` | Knotty Yoga Silver Membership | 160 | 1 | `silver_member` | `tier_icon_solo.svg` |
| `silver-couples` | Knotty Yoga Silver Couple's Membership | 240 | 2 | `silver_member` | `tier_icon_couple.svg` |
| `gold-membership` | Knotty Yoga Gold Membership | 200 | 1 | `gold_member` | `tier_icon_solo.svg` |
| `gold-couples` | Knotty Yoga Gold Couple's Membership | 300 | 2 | `gold_member` | `tier_icon_couple.svg` |
| `gold-family` | Knotty Yoga Gold Family Membership | 400 | 3 | `gold_member` | `tier_icon_family.svg` |

### Caleb's massage hours (`schedule_template_entries`, **ISO: Mon = 1 … Sun = 7**)

| Day | ISO | Hours | Minutes |
|---|---|---|---|
| Mon | 1 | 11:00–15:00 | 660–900 |
| Tue | 2 | 11:00–16:00 | 660–960 |
| Thu | 4 | 11:00–16:00 | 660–960 |
| Fri | 5 | 11:00–15:00 | 660–900 |
| Sun | **7** | 13:30–16:00 | 810–960 |

None of these overlaps a class Caleb teaches (Sun 10–11, Mon 17–18, Thu 17–18); Phase 5.3 pins that with a test. ⚠️ Sunday is **7** here and **0** in `class_schedule_slots` ([[gotcha_two_day_of_week_conventions]]). A literal `0` in a template entry silently generates nothing.

---

# Phase 1 — Seed plumbing: natural keys everywhere

Prerequisite for every catalog and people change (fact 1). No content changes in this phase: a fresh database must come out **identical** to today's, which is what makes it safe to land first.

### 1.1 Move the remaining id-addressed seeders into `seed_app_data` *(seed)*
- [ ] Move `PopulatePeople`, `PopulateRoleAssignments`, `PopulateProductVariants`, `PopulateProductPrices` and `PopulateProductBookingWindows` from `create_database.cpp`'s anonymous namespace into `seed_app_data.{h,cpp}` (`namespace Seed`), same `(Transaction&, DatabaseHelper, DatabaseInfo)` signature, so a test can call them.
- [ ] Variants: resolve the product by code (`massage`, …).
- [ ] Prices: resolve the product by code, the variant by `product_variants.code`, and the price schedule by `kSeedPriceScheduleName`. Add a `LookupProductVariantIdByCode` helper beside the existing `Lookup*` (logs and returns 0, same contract).
- [ ] Booking windows: resolve `massage` by code.
- [ ] Role assignments: resolve people by email (`kMasonEmail`, `kCalebEmail`, plus a new `kMasonLiveEmail`). `RoleIdByName` already exists; move it alongside.
- [ ] Move the trailing `UPDATE products … WHERE code = 'massage'` (provider type + required room type) into a named `Seed::PopulateServiceProductLinks`, resolved by code rather than by subquery-in-a-string.

### 1.2 Tests *(seed_app_data_test.cpp)*
- [ ] Prices through `CatalogHelper::ResolveBestPriceForPerson`: the 60/90/120-minute massage variants price at 16000/22000/30000 and the intro workshop at 900, **with a decoy product, variant and price schedule inserted first** so a literal id would bind to the decoy (same shape as the existing `PopulateClassSchedulesResolvesRowsByNaturalKeyNotSeedOrder`).
- [ ] Role assignments through `PersonHelper::GetRolesForUser`: Mason and Caleb are Administrators and Caleb is an Instructor, again behind a decoy person seeded first.
- [ ] `PopulateServiceProductLinks`: the massage product resolves to the `massage_therapist` provider type and the `massage_room` room type.
- [ ] Build `knottyyoga_database_helper` explicitly as well; the gate does not build its `main.cpp`.

---

# Phase 2 — Products and memberships

### 2.1 Silver permission *(seed)*
- [ ] `PopulatePermissions`: add `silver_member`, "Knotty Yoga Silver Membership.", `is_pricing_eligible = true`. Name constant `kSilverMemberPermissionName` beside `kGoldMemberPermissionName`.

### 2.2 Product list *(seed)*
- [ ] `PopulateProducts`: **remove** `early-bird-spa`, `non-peak-spa`, `peak-spa`, `late-night-spa`.
- [ ] **Add** `silver-membership` and `silver-couples` (kind `subscription`), named as in the Memberships table above.
- [ ] Membership descriptions say what each tier includes (copy in OQ-14). Today's "Monthly membership with Gold Member benefits" tells a visitor nothing.

### 2.3 Variants, prices, entitlements *(seed)*
- [ ] `PopulateProductVariants`: massage 60/90/120 only.
- [ ] `PopulateProductPrices`: drop the spa rows; add Silver 16000 and Silver Couple's 24000 (monthly, no variant, like Gold).
- [ ] `PopulateProductEntitlementRules`: `silver-membership` → 1 seat, `calendar_month`, `silver_member`; `silver-couples` → 2 seats, same.

### 2.4 Remove the spa infrastructure *(create_database.cpp bootstrap tail)*
- [ ] Delete the spa `UPDATE`s, the `Spa` room schedule template + its seven entries, and every `product_schedule_windows` row (all four belonged to spa products).
- [ ] `PopulateLocationRooms`: drop the `Spa` room. Room types stay (D8).

### 2.5 Tier icons *(seed)*
- [ ] `PopulateSeedPhotos`' product table: `silver-membership` → `tier_icon_solo.svg`, `silver-couples` → `tier_icon_couple.svg`.

### 2.6 Tests
- [ ] Rewrite `PopulateProductsSeedsTheDropInClassProduct` into a whole-catalog assertion: exactly the eight product codes, and **no product whose code contains `spa`**.
- [ ] `SilverProductsGrantTheSilverMemberPermission`, modelled on the Gold test, including "the Silver and Gold permission ids differ".
- [ ] Membership prices through `CatalogHelper`: 16000 / 24000 / 20000 / 30000 / 40000.
- [ ] `PopulateSeedPhotosGivesEachMembershipTierItsIcon`: five tiers now.
- [ ] Spa removal through the readers: no `Spa` room in the facility's rooms, no product schedule windows, no room schedule templates.

---

# Phase 3 — People, roles and instructors

### 3.1 People *(seed)*
- [ ] `PopulatePeople`: add Beth Cooper, Gifford Cheung and Tiana Cheung, emails per **OQ-1**, via `CreateFullyValidatedUser` with a **random, unrecorded** password (D9): 32 bytes from the existing secure-random utility, base64, never logged.
- [ ] Email constants `kBethEmail`, `kGiffordEmail`, `kTianaEmail` in `seed_app_data.h`.
- [ ] *(Per OQ-10)* Move Kit Williams and the `masonbendixen@hotmail.com` "Mason2" admin into `PopulateTestData`. Neither belongs on a production studio, and Mason2 is an **admin with password `mason`**.

### 3.2 Roles *(seed)*
- [ ] `PopulateRoleAssignments`: Instructor role for Mason, Caleb, Beth, Gifford and Tiana (Mason and Caleb keep Administrator).
- [ ] *(Per OQ-5)* `PopulateRolePermissions`: grant the Instructor role `staff_access` (door check-in) and `manage_skills` (granting the approval skills from Phase 4). Without `manage_skills`, only an admin can approve anyone for Knotty Yoga.

### 3.3 Instructor profiles *(seed)*
- [ ] `PopulateInstructors`: five rows, data-driven. Bios per **OQ-15**: Mason's and Caleb's updated to the classes they teach now, and new ones for Beth, Gifford and Tiana.

### 3.4 A way to hand an instructor a password *(test_helper; per OQ-2)*
- [ ] `knottyyoga_test_helper` command `set_password` (alias `sp`, "Users & Permissions"): `--email`, `--password`. Calls `PersonHelper::UpdatePassword` with the production secrets, then sets `must_change_password` (as `force_password_change` already does) so the instructor picks their own at first sign-in. Works against prod the same way `set_secret` does.
- [ ] Tests: changes the hash (the old password no longer verifies, the new one does), sets the flag, and refuses an unknown email with a clear message.

### 3.5 Tests
- [ ] All five instructors exist with non-empty bios, read through `InstructorProfileHelper::GetInstructorProfile` (the public Instructors page's reader).
- [ ] The three new people can **not** sign in with any password present in the source tree (`VerifyPassword` against the dev passwords returns false), so D9 is pinned.
- [ ] All five hold the Instructor role, through `PersonHelper::GetRolesForUser`.
- [ ] *(OQ-5)* An Instructor-only person holds `staff_access` and `manage_skills` through `PersonHelper::GetPermissionsForUser`.

---

# Phase 4 — Classes, approval skills, access rules and the weekly schedule

### 4.1 Skill levels *(seed)*
- [ ] New `Seed::PopulateSkillLevels`: the four approval skills from the table above, via `TableHelpers::SkillLevels::AddSkillLevel`, `sort_order` 10/20/30/40.

### 4.2 Classes *(seed)*
- [ ] `PopulateClasses`: the six classes with the Overview's descriptions, kind `recurring` (the column default). The four old classes are no longer created.
- [ ] Copy normalised only for spacing ("Beginner Knotty  Yoga" in the Overview has a double space). Otherwise verbatim; the studio can edit it in Manage.
- [ ] Class-name constants in `seed_app_data.h`; the slot, access and photo tables all refer to these.

### 4.3 Access rules *(seed)*
- [ ] New `Seed::PopulateClassRequirements`, data-driven from the Class access table: `ClassRequirementGroups::AddGroup` (label "Membership" / "Instructor approval", `inline_description` as in the table; admin-only until 10.3, fact 9) plus `ClassRequirementGroupLiterals::AddPermissionLiteral` / `AddSkillLiteral`. Permissions and skills resolved by name and code; a missing one logs and skips that **whole class's** rules rather than writing half of them, because a half-written CNF is *more* permissive than intended.
- [ ] Runs after `PopulatePermissions`, `PopulateSkillLevels` and `PopulateClasses`.

### 4.4 The weekly schedule *(seed)*
- [ ] Rewrite `PopulateClassSchedules` around one `constexpr` slot table (D3): `{ className, dayOfWeek, startMinutes, durationMinutes, instructorEmail, predecessor { className, dayOfWeek } }`, the 14 rows above.
- [ ] One perpetual instance + default schedule per class (the existing `AddPerpetualSchedule` shape), all bound to `class-dropin` (unchanged decision D8 of the previous plan).
- [ ] Predecessors resolved through a `(className, dayOfWeek) → slotId` map filled as slots are created; the table is ordered so a predecessor always precedes its follower, **and** the seeder refuses (logs and skips the follower) if not, rather than writing a dangling id.
- [ ] Tuesday Beginner Knotty Yoga gets Gifford as `instructor_person_id`; Tiana is added in Phase 8.

### 4.5 Tests
- [ ] Replace `PopulateClassesSeedsTheSixStudioClasses` with the new six, plus "none of the four retired names exists".
- [ ] **Access through `ClassAccessHelper::CheckAccess`** (D4), four seeded people:
  - Silver-only member: Beginner Knotty Yoga **allowed**, all five others **denied**.
  - Gold-only member: Beginner Knotty Yoga and Beginner Partner Acro **allowed**; Knotty Yoga, Handstands, Aerial Rope and Intermediate Partner Acro **denied** on the approval group only.
  - Gold member + `knotty_yoga_approved`: Knotty Yoga **allowed**, Handstands still denied.
  - Non-member: everything denied; `Annotate` reports `members_only` for Beginner Knotty Yoga and `needs_skill` for Knotty Yoga (skill gates outrank membership gates).
- [ ] Schedule: for each class, its slots match the slot table by **weekday and start** (never by array index), with the right instructor email.
- [ ] Predecessors: each of the four followers names the right slot, **and** `predecessor.start + predecessor.duration == follower.start`.
- [ ] Session instructors through `ClassCatalogHelper::GetClassDetail(...).upcomingSessions[].instructorNames`, e.g. Wednesday Beginner Knotty Yoga reads "Beth Cooper".
- [ ] `NoTwoSlotsCollideInTheSameRoom`: run against the **base** seed's 14 slots (half-open intervals, so back-to-back is legal).
- [ ] Update `PopulateClassSchedulesResolvesRowsByNaturalKeyNotSeedOrder` and `PopulateClassSchedulesSkipsWhenPrerequisitesAreMissing` to the new table; delete the three per-class slot tests they replace.

---

# Phase 5 — Caleb as the massage provider

### 5.1 Provider assignment *(seed)*
- [ ] `PopulateProviderTypeAssignments`: Caleb only, `massage_therapist`, accepting bookings (per OQ-6, Mason's assignment is dropped).

### 5.2 Weekly hours *(seed)*
- [ ] New `Seed::PopulateProviderSchedules(…, nowUs)`: one open-ended `schedule_templates` row for Caleb ("Massage hours", `effective_to_us` NULL), with the five ISO entries above. The ISO constants are named locally exactly as the test-data seeder does today, with the same warning comment.
- [ ] Generate `provider_availability` from today's local midnight through the horizon (Phase 9's constant if it lands, otherwise `EndOfFollowingMonthUs`) via `ScheduleTemplateHelper::GenerateAvailability`, in the studio facility.
- [ ] `PopulateTables` reads `now_us()` once for this, as `PopulateTestData` does (D2 of the previous plan).

### 5.3 Tests
- [ ] Through the **booking-slot reader** (the one behind the massage picker and `list_available_slots`): on a fixed Monday Caleb offers slots inside 11:00–15:00 and none outside; Wednesday and Saturday offer none; a Sunday offers 13:30–16:00 (pins ISO 7).
- [ ] Every generated row is Caleb's, `source = template`, and on a Mon/Tue/Thu/Fri/Sun local date.
- [ ] **Caleb's massage hours never overlap a class slot he teaches.** Cross-checks Phase 4's slot table against Phase 5's entries, so a future schedule edit that double-books him fails the gate.
- [ ] Mason holds no provider assignment.

---

# Phase 6 — Photos

### 6.1 Table-driven photo attachment *(seed)*
- [ ] Extend `PopulateSeedPhotos` to one table per target: `instructors` (by person email), `classes` (by name), `products` (by code) and `facilities` (by code). One loop each, every lookup by natural key, every outcome logged (the existing `AttachSeedPhoto` contract).
- [ ] Instructor headshots move off the hardcoded list in `PopulateInstructors` into this table, so every picture is listed in one place.

### 6.2 Photo intake: file + table row + test expectation, together (D10)

Files go in `server/knottyyoga_server/src/database_helper/img/`. JPEG, long edge about 2000 px, under about 1.5 MB. Convert with `ffmpeg -q:v 3`, **never** `convert`, which on Windows resolves to the FAT-to-NTFS disk converter.

| # | Subject | Attached to | File | Status |
|---|---|---|---|---|
| 1 | Mason headshot | instructors (Mason) | `MasonInstructor.jpg` | ✅ exists; replace if a new headshot is provided |
| 2 | Caleb headshot | instructors (Caleb) | `CalebInstructor.jpg` | ✅ exists; replace if new |
| 3 | Beth Cooper headshot | instructors (Beth) | `BethInstructor.jpg` | ⬜ needed |
| 4 | Tiana Cheung headshot | instructors (Tiana) | `TianaInstructor.jpg` | ⬜ needed |
| 5 | Gifford Cheung headshot | instructors (Gifford) | `GiffordInstructor.jpg` | ⬜ needed |
| 6 | Beginner Knotty Yoga | classes | `BeginnerKnottyYoga.jpg` | ⬜ needed |
| 7 | Knotty Yoga | classes | `KnottyYoga.jpg` | ✅ exists |
| 8 | Aerial Rope | classes | `AerialRope.jpg` | ⬜ needed |
| 9 | Handstands | classes | `Handstand.jpg` | ✅ exists |
| 10 | Beginner Partner Acro | classes | `PartnerAcro.jpg` | ✅ exists (today's Partner Acrobatics photo) |
| 11 | Intermediate Partner Acro | classes | `IntermediatePartnerAcro.jpg` | ⬜ needed |
| 12 | Massage | products (`massage`), the Services card | `Massage.jpg` | ⬜ needed |
| 13 | General studio | facilities (`knotty-yoga-studio`), About ▸ Our Location | `Studio.jpg` | ⬜ needed |

- [ ] Rows 1, 2, 7, 9, 10 wired in 6.1 (files exist).
- [ ] Rows 3–6, 8, 11–13: each lands as file + table row + test expectation in one change, as the picture arrives (OQ-11).
- [ ] Mason's and Caleb's **profile** photos (`Mason.jpg`, `Caleb.jpg` on `people`) are unchanged. New instructors upload their own profile pictures.

### 6.3 Tests
- [ ] `EverySeedPhotoFileExists`: walks the same tables the seeder walks and asserts each file is in `SeedImageDirectory()`. Because entries are only added with their file (D10), this is green by construction and catches a renamed or deleted file.
- [ ] Attachment by target type: each listed row has exactly one photo, and **classes not in the table have none** (the D5 wrong-row guard from the previous plan, kept).
- [ ] Massage and studio photos through their public readers (catalog product `has_photo`; the facility's photo on the location reader) once rows 12 and 13 land.

---

# Phase 7 — Test-only seed data (`--recreate_database_test`) reconciled with the real schedule

### 7.1 Move the colliding sessions *(seed; per OQ-12)*
- [ ] Intro Workshop sessions: Saturday **13:00–14:00** (was 10:00–11:00, which sits inside Saturday Knotty Yoga).
- [ ] Aerial Series run: **Tuesday + Friday** 19:00–20:00 (was Tue/Thu; Thursday 19:00 is Intermediate Partner Acro).

### 7.2 Remove what the base seed now owns *(seed)*
- [ ] Delete `PopulateProviderSchedule` (Caleb's Mon–Fri 9–5 test template): Phase 5 makes his real hours base data, and two templates would generate overlapping availability.
- [ ] *(OQ-10)* Kit Williams and Mason2 now arrive here (from 3.1).

### 7.3 Tests
- [ ] Update the Phase 3 test-data tests for the new times and days. `IntroWorkshopSessionsAppearOnBothTheHomePageAndEventsFeeds` stays as-is: it asserts through the readers.
- [ ] Extend the collision test to **event sessions vs. class slots** in the same room after the full test seed. That is the check that would have caught 7.1 automatically.
- [ ] `OnlyPopulateTestDataAddsTheSampleData`: now also asserts Kit and Mason2 are absent from the base seed.

---

# Phase 8 — Co-instructors on a recurring slot (Gifford + Tiana)

Fact 4: a slot cannot carry two instructors today. *(Default for OQ-7: build it. Until it lands, Tuesday shows Gifford only.)*

Scope: an occurrence's **displayed** instructors become `staffing override if present, else slot lead + co-instructors`. Pay and specialty cost, the instructor-load grid, the exception-notes digest and favorite-instructor alerts stay keyed on the **lead**. That is the conservative reading, and each can widen later on its own.

### 8.1 Schema *(db_schema)*
- [ ] New nested table `class_schedule_slot_co_instructors` (`id`, `class_schedule_slot_id` FK, `person_id` FK → people, `created_us`, `updated_us`, UNIQUE(slot, person), index on slot).
- [ ] Full new-table checklist (CLAUDE.md): `make_app_tables`, `CreateTables`, allowed + top-level + nested admin tables, column data info, friendly names, display template, table permission (`manage_class_schedule`), CMakeLists.
- [ ] **Idempotent migration `app/0010_class_schedule_slot_co_instructors`** ([[gotcha_new_table_needs_migration]]). Prod gets the table through the reinstall anyway, but every other database needs it.

### 8.2 Table helper *(sql_util/table_helpers)*
- [ ] `TableHelpers::ClassScheduleSlotCoInstructors`: `Add`, `GetForSlot`, `GetForSlots` (batch, for the catalog's N slots), `Delete`. Tests for each, including the unique constraint.

### 8.3 Business logic *(business_logic/scheduling)*
- [ ] One shared `EffectiveInstructorIds` (the instructor-profile helper already has a private one): staffing override → else lead + co-instructors, lead first. Callers: `ClassCatalogHelper` (class detail sessions), `CalendarHelper`, `InstructorProfileHelper` (both "classes I teach", whose SQL `UNION` gains the co-instructor table, and upcoming sessions).
- [ ] `ClassScheduleHelper`: add/remove co-instructor on a slot, refusing the lead themself and a non-instructor person.
- [ ] Tests per caller: a co-taught occurrence lists both names; a substitution still **replaces** both; Tiana's instructor profile lists Beginner Knotty Yoga; the instructor-load grid is unchanged (pins the scope line above).

### 8.4 Endpoints
- [ ] `POST /api/admin/class_schedule_slots/{id}/co_instructors` and `DELETE …/{personId}`, gated on `manage_class_schedule`, thin over 8.3. Endpoint tests: auth (401/403), happy path, duplicate, lead-as-co-instructor refused.

### 8.5 Seed
- [ ] Tuesday Beginner Knotty Yoga: Tiana as co-instructor. The slot table gains an optional `coInstructorEmail`.
- [ ] Test through `ClassCatalogHelper`: Tuesday sessions read "Gifford Cheung, Tiana Cheung".

### 8.6 Frontend
- [ ] `ServerAccess` + network + mock + proxy methods for add/remove; `ServerAccess.mock.spec.ts` cases.
- [ ] Slot form dialog (Manage ▸ class schedules): a **Co-instructors** chip picker beneath Instructor, reusing the instructor person picker. Manage Data is not a workflow ([[feedback_manage_data_is_debug_only]]).
- [ ] Wherever a session renders one instructor name, render the list joined with ", " (class detail, calendar event, instructor profile).
- [ ] Specs: the dialog adds and removes a co-instructor; class detail renders two names; the lead cannot be picked as their own co-instructor.

---

# Phase 9 — Keep provider availability rolling forward

Fact 5: generated availability runs out. *(Default for OQ-8: build it, 60-day horizon.)*

### 9.1 Table helper *(sql_util/table_helpers)*
- [ ] `ProviderAvailability::GetLatestTemplateRow(templateId)`: the most recent generated row (`date_us`, `facility_id`). Custom SQL justified by `MAX`/`ORDER BY … LIMIT 1`. Tests: none, one, many.

### 9.2 Business logic *(business_logic/scheduling)*
- [ ] `ScheduleTemplateHelper::ExtendActiveTemplates(transaction, nowUs, horizonDays)`: for every active template whose window includes now, generate from the day after its latest generated row (or today) through `min(today + horizon, effective_to)`, in **that row's facility**. A template that has never been generated is skipped and logged, because its facility is only chosen by a first manual or seeded generation.
- [ ] **Idempotent**: a second run the same day generates nothing. The existing manual/blocked/exception skips still apply.
- [ ] Tests: extends to exactly the horizon; a second run is a no-op; respects `effective_to`; skips inactive and never-generated templates; walks across a DST change without drifting off local midnight.

### 9.3 Endpoint + scheduled job
- [ ] `POST /api/admin/schedule_templates/extend_availability`, gated like the existing generate endpoint, thin over 9.2. Endpoint tests.
- [ ] `JobIntervals::extendProviderAvailabilitySeconds = 86400` + the job entry in `BuildKnottyYogaJobs`. Job-catalog test that it is present, targets the endpoint, and is removed at interval 0.

### 9.4 Seed
- [ ] Phase 5.2 generates through the same horizon constant, so the job's first run extends by one day rather than a month.

---

# Phase 10 — Frontend polish

### 10.1 Membership tier order *(per OQ-13)*
- [ ] `MembershipTierCardsComponent` sorts by monthly price ascending: Silver 160, Gold 200, Silver Couple's 240, Gold Couple's 300, Gold Family 400. Today the catalog orders by **name**, which would show Gold Couple's first and Silver last. Both hosts (home band, Memberships page) share the component, so one change covers both.
- [ ] Spec: given products in name order, the cards render in price order; ties keep input order.

### 10.2 Dead code
- [ ] Delete `pages/public/services/homepage.service.ts` and its spec (fact 8).

### 10.3 Say which membership a class needs *(per OQ-17; business logic → endpoint → frontend)*
- [ ] `ClassCatalogHelper::GetClassDetail`: add the class's non-empty membership-group `inline_description`s to the detail (the requirements view already resolves them; reuse `ClassAccessHelper::GetClassRequirements`). Test: Beginner Knotty Yoga's detail carries "Silver or Gold membership", and Knotty Yoga's carries "Gold membership".
- [ ] Class-detail endpoint serialises it (`scheduling_key_value_table.cpp` already writes `inline_description` for the admin view). Endpoint test.
- [ ] Class page: the "Members only" pricing state reads **"Members only: Silver or Gold membership"** (the description), falling back to today's text when there is none. Type + mock + spec: both the described and the undescribed states render.

*(Not in scope: `ServerAccess.mock.ts`'s local-mode catalog still has the old classes. It is fixture data for `ng serve -c local` and for dozens of specs; changing it is churn with no user-visible effect.)*

---

# Phase 11 — Verification

### 11.1 Build and test
- [ ] Linux gate (`knottyyoga/server/docker_project/build_and_test.sh`) and Windows (`x64-Debug`), reported per platform. Plus the explicit `knottyyoga_database_helper` and `knottyyoga_test_helper` builds, which the gate does not build.
- [ ] Angular: `ng test`, `tsc --noEmit`, `ng build`.

### 11.2 Local run
- [ ] `knottyyoga_database_helper --recreate_database` against the local dev database (destructive: announced before running), then `--recreate_database_test`. Confirm no `seed_lookup_missing`, `*_skipped` or `seed_image_failed` lines in the log beyond the photos still pending in 6.2.

### 11.3 Hand-testing steps

Menu labels were checked against the templates on 10/14 (`mockHeaderResponse.ts` **is** the production menu; `header.service.ts` always builds from it). Below 1280 px the top nav collapses into the hamburger, so use a wide window.

**Setup.** Run `knottyyoga_database_helper --recreate_database_test` (needs `HONUWARE_ALLOW_DESTRUCTIVE=1` and `SCHEDULER_SERVICE_ACCOUNT_PASSWORD`), start the server and the Angular app. The test flag is used because steps 9–14 need the member account **Kit Williams** (`kiteagle@gmail.com` / `kit`), which after OQ-10 exists only in test data. Admin is `masonbendixen@gmail.com` / `pass`.

**Signed out: the catalog**
1. Top nav **Our Classes**. The page is headed **All Classes** and shows exactly six cards: **Beginner Knotty Yoga, Knotty Yoga, Handstands, Aerial Rope, Beginner Partner Acro, Intermediate Partner Acro**. There are no Therapeutic Knotty Yoga, Tumbling, Aerial Fabric or Partner Acrobatics cards. Knotty Yoga, Handstands and Beginner Partner Acro have photographs; the others gain theirs as 6.2's files arrive.
2. Click **Weekly schedule** (page headed **Our Classes**). Check the 14 rows against the schedule table above, including **Saturday 10:00 AM – 12:00 PM Knotty Yoga with Mason Bendixen** and **Wednesday 5:00 PM Beginner Knotty Yoga with Beth Cooper**. These four rows also read **Requires attending: …**:
   - Monday 7:00 PM Handstands → **Knotty Yoga**
   - Wednesday 7:00 PM Aerial Rope → **Knotty Yoga**
   - Thursday 7:00 PM Intermediate Partner Acro → **Beginner Partner Acro**
   - Sunday 12:00 PM Intermediate Partner Acro → **Beginner Partner Acro**
3. Same page: **Beginner Knotty Yoga** and **Beginner Partner Acro** rows carry the badge **Requires a membership**. **Knotty Yoga**, **Handstands**, **Aerial Rope** and **Intermediate Partner Acro** carry **Prerequisite needed**: a skill gate outranks a membership gate.
4. Click **Prerequisite needed** on a Monday **Knotty Yoga** row. A dialog titled **Skill required** lists **Knotty Yoga approval** with "Can do a pull-up and five regular push-ups…". Click **Got it**.
5. **Our Classes** → **Beginner Knotty Yoga** card. Under **Pricing**: **Members only**, followed by "Silver or Gold membership" once 10.3 lands, and "Upgrade your membership to attend." before it. Back via **All Classes** → **Knotty Yoga**: **Required Skills** shows the **Knotty Yoga approval** chip.
6. **About** → **Instructors**: five cards (**Mason Bendixen, Caleb Ault, Beth Cooper, Gifford Cheung, Tiana Cheung**), each with a bio.
7. **Memberships**: five cards, in this order after 10.1:
   - Knotty Yoga Silver Membership **$160.00/mo**
   - Knotty Yoga Gold Membership **$200.00/mo**
   - Knotty Yoga Silver Couple's Membership **$240.00/mo**
   - Knotty Yoga Gold Couple's Membership **$300.00/mo**
   - Knotty Yoga Gold Family Membership **$400.00/mo**

   Each has a laurel icon and a **Subscribe** button.
8. **Services** (page headed **Our Services**): **Massage** is listed. **Early Bird Spa, Non-Peak Spa, Peak Spa and Late Night Spa are gone.** **About** → **Our Providers** shows **Caleb Ault** only (not Mason).

**Silver member: Silver opens Beginner Knotty Yoga and nothing else**

9. **Sign In** as `kiteagle@gmail.com` / `kit`.
10. **Memberships** → **Knotty Yoga Silver Membership** → **Subscribe**. Leave **Start month** as it is, enter sandbox card `4111 1111 1111 1111` (any future expiry, CVV `111`), and click **Subscribe & Pay $160.00**.
11. **My Classes** → **Full Schedule**. Beginner Knotty Yoga rows no longer carry a badge; Beginner Partner Acro still reads **Requires a membership**; Knotty Yoga still reads **Prerequisite needed**.
12. **Our Classes** → **All Classes** → **Beginner Knotty Yoga**: **Pricing** reads **Included with your membership**, "Just show up — staff will check you in." Back → **Beginner Partner Acro**: **Members only**.

**Instructor approval: AND across groups**

13. Kit's first-name menu → **Sign Out** → **Log Out**. Sign in as `masonbendixen@gmail.com` / `pass`. Top nav **Staff** (page headed **Staff Portal**) → **Person Skills** tile → **Search for a person**: type `Kit`, click **Kit Williams** → **Assign new skill** → **Skill**: **Knotty Yoga approval**, **Note (optional)**: `Pull-up and five push-ups` → **Assign**.
14. Sign out, sign back in as Kit. **Our Classes** → **All Classes** → **Knotty Yoga**: **Required Skills** now reads **You meet the prerequisites**, but **Pricing** still reads **Members only**. Approval alone does not admit a Silver member; Knotty Yoga needs Gold **and** approval.

**Admin views of the same data**

15. As Mason: **Admin** → **Manage Products** → **Class Schedules** → **Class**: **Beginner Knotty Yoga**. The **Requirements** card holds one group, **Membership**, with the Silver and Gold permissions, and **Description (shown to members)** reads "Silver or Gold membership".
16. Same page, **Class**: **Handstands** → expand **Class Instances** → **Class Schedules** → **Class Schedule Slots**. One slot: **Monday**, **7:00 PM**, **60**, **Instructor** Mason Bendixen, **Requires attending** **Knotty Yoga · 6:00 PM–7:00 PM**.
17. **Admin** → **Manage Products** → **Skills**: four skills, **Knotty Yoga approval, Handstands approval, Aerial Rope approval, Intermediate Partner Acro approval**.
18. **Admin** → **Manage Products** → **Products**: no product named "Spa" anything, with **Show Inactive** on as well. Both Silver memberships are listed.

**Massage: Caleb's hours**

19. **Admin** → **Manage Products** → **Service Providers**: one card, **Caleb Ault**, **Accepting bookings** on. Click **Templates**: one template, **Massage hours**, with Monday, Tuesday, Thursday, Friday and Sunday ticked at the hours in the table above.
20. **Services** → **Massage** → **Book Now**. In the week strip, click the next **Mon** (use **Next week** if needed). One provider card, **Caleb Ault**. The first hour header is **11:00 AM +** and no session starts so late it would end after 3:00 PM.
21. Click the next **Wed**: no provider card. Click the next **Sun**: Caleb's earliest time is **1:30 PM**.

**Co-instructors (after Phase 8)**

22. **My Classes** → **Full Schedule** → Tuesday 6:00 PM **Beginner Knotty Yoga** reads **with Gifford Cheung, Tiana Cheung**. **About** → **Instructors** → **Tiana Cheung** lists Beginner Knotty Yoga among her classes.

**The flag boundary**

23. Re-run with plain `--recreate_database`. Signing in as `kiteagle@gmail.com` now fails (the account does not exist). **Upcoming Events** has no Intro Workshop sessions, and **Our Classes** has no **Aerial Series**. Steps 1–8 and 15–21 are unchanged: that is the base seed, and it is what production gets.

---

# Phase 12 — Production rollout

Per D11 / OQ-9 (reinstall). Everything here runs on the EC2 host following the Deploying to AWS conventions: `--entrypoint` for helpers, `-v /etc/knottyyoga:/etc/knottyyoga:ro`, `--env-file /etc/knottyyoga/server.env`.

### 12.1 Before
- [ ] Tag + deploy the release that contains Phases 1–10 the normal way (CI tag → deploy script → `--migrate`). This proves the image, and confirms the new `img/` files are staged in it ([[gotcha_release_image_glibc_and_entrypoint]] #4).
- [ ] Manual RDS snapshot, named for this change.
- [ ] `--export_theme /etc/knottyyoga/theme-backup` *(only matters if anything was edited on prod; OQ-9)*.
- [ ] Have the `set_secret` block from Deploying to AWS 5.1 step 6 ready, with `website_address_login` = the cloudfront URL until go-live.

### 12.2 Reinstall
- [ ] Stop `knottyyoga-helper` and `knottyyoga-server`.
- [ ] `knottyyoga_database_helper --install_schema --force` with `HONUWARE_ALLOW_DESTRUCTIVE=1` (one-shot, on that command only) and `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` from `server.env`.
- [ ] `--migrate` (records the migration ledger; expected `applied=N skipped=0` as on 9/23).
- [ ] Re-run the `set_secret` block; `--import_theme` the backup if one was taken.
- [ ] Start server then helper; `/api/health` → `db:"ok"`; helper log shows `login_success`.

### 12.3 After
- [ ] Mason and Caleb change their passwords (Profile → change password). The seed's are committed to source control.
- [ ] `knottyyoga_test_helper --command=set_password` for each instructor who should sign in (3.4), with a temporary password handed over in person.
- [ ] The Phase 11.3 walkthrough's public half against `https://dv1tgxa9ok30f.cloudfront.net/`.

---

# Cross-references

- [[Database helper initialization]]: this plan closes two of its follow-ups: "`PopulateProductVariants` / `PopulateProductPrices` / `PopulateProductBookingWindows` still hardcode ids" (Phase 1) and "`late-night-spa` is gated on a permission that does not exist" (Phase 2, by removal). Tick them there as each lands.
- [[Photo plan for studio]]: the source list for 6.2.
- [[Permission-based class access redesign]]: the requirement-group model Phase 4 seeds.
- [[Deploying to AWS]]: Phase 12's commands, and go-live (the cert + alias records) remains separate.

---

# Open questions

*Each has a default the plan is written against. Answer inline and I'll adjust; anything unanswered goes ahead on its default.*

**OQ-1 — Emails for Beth Cooper, Gifford Cheung and Tiana Cheung.** `people.email` is required and unique, and the instructor digest emails them. **Default:** placeholders `beth.cooper@knottyyoga.local` (likewise for the other two), undeliverable like the scheduler account, so the code can land. 12.3 swaps in real ones before rollout.
- Mason- I sent a message to all my instructors asking which email to use.

**OQ-2 — Do they need to sign in now, and how do they get a password?** Knottyyoga has no forgot-password flow. If one of them later tries to **Register** with their real email, it will fail because the seeded row already exists. **Default:** the random seed password (D9) plus a new `knottyyoga_test_helper set_password` command (3.4) that you run to give each a temporary password, which they must change at first sign-in. A real self-service reset is a separate feature.
- Mason- Hrm... the forgot password flow seems like a bad problem. Can we add a phase to correct that with a forgot email button that sends an email with a create new password flow? But for this, I figure I will give them a temporary password that they create here. Do we have must change password on next login support?

**OQ-3 — Which memberships include the non-beginner classes?** The Overview only says Beginner Knotty Yoga is Silver-or-Gold. With no membership group, a class is open **and free to everyone**. **Default:** Knotty Yoga, Handstands, Aerial Rope and both Partner Acro classes require Gold (D7).
- Mason- that's correct.

**OQ-4 — "Instructor approval" as a skill level per class.** **Default:** four skills (table above), granted by an instructor on the skill-assignment page; Beginner Partner Acro's solo-flyer rule is description text only (D6). Say if any approval should instead be a single studio-wide "approved" skill.
- Mason- Let's make skills for handstand, aerial_rope, intermediate_partner_acro. Say the things in there that I listed for each in the description.

**OQ-5 — Should the Instructor role be able to check people in and grant approvals?** Check-in needs `staff_access`, and granting a skill needs `manage_skills`. Today only admins and Studio Managers have them. **Default:** grant both to the Instructor role, so Beth, Gifford and Tiana can run their own classes.
- Mason- Yes, this sounds correct.

**OQ-6 — Is Mason still a massage provider?** He is assigned today, and the photo plan mentions a "Mason massage" photo, but the Overview gives hours only for Caleb. A provider with no hours appears in the picker with nothing to book. **Default:** Caleb only; re-adding Mason later is one admin row.
- Mason- Let's just do Caleb for now.

**OQ-7 — Co-instructors (Phase 8).** It is a real feature: a new table, readers in three helpers, two endpoints and a dialog control. **Default:** build it. **Alternative:** Gifford alone on the slot, with Tiana named in nothing. Say which.
- Mason- we should support multiple instructors.

**OQ-8 — Rolling availability (Phase 9).** **Default:** build it, with a 60-day horizon. **Alternative:** skip it and press **Generate** on Caleb's template in Manage every month. If that is forgotten, massage booking quietly shows nothing.
- Mason- no no no... let's not make this a new feature. We can just add slots with the template. The generate availability feature is pretty simple. Could also do this as a scheduled job pretty easily if need be. Let's not complicate the webserver more than necessary.

**OQ-9 — Getting this onto the live site.** **Default:** reinstall prod (D11, Phase 12). That **erases** every account registered there, every booking and purchase, and anything edited through Manage (page content, uploaded photos, carousel). Theme settings survive via export/import. **Has anything on prod been entered by hand that you want kept?** If so, I'll list it in 12.1 and carry it across, or switch to an in-place migration for just those parts.
- Mason- No, I'm just in preview mode and still using the Square sandbox. Wipe away.

**OQ-10 — Test accounts in the base seed.** `masonbendixen@hotmail.com` (an **admin**, password `mason`) and Kit Williams (`kit`) are created by `--install_schema`, so prod has them today. **Default:** move both to `--recreate_database_test` only (3.1). Hand-testing walkthroughs that use Kit as "the member" will say to run `--recreate_database_test`. Separately: **Mason's and Caleb's prod passwords are the committed `pass` / `caleb` unless you've changed them**, and the cloudfront URL is public. Worth changing now, regardless of this plan.
- Mason- I'll manually have the three of us change our passwords as we deploy. I'm not that concerned about this.

**OQ-11 — Where are the photos?** None of the eight "needed" files in 6.2 is in the tree. **Default:** you drop them into `src/database_helper/img/` under the names in the table, or give me paths as in the previous plan and I'll copy/convert them. New headshots for Mason and Caleb replace the existing `MasonInstructor.jpg` / `CalebInstructor.jpg`.
- Mason- Taking the photos IS a phase. Please list this as an early phase that the later phases will use. I'll try to take the photos tonight.

**OQ-12 — Test-only data times.** **Default:** Intro Workshop to Saturday 13:00–14:00; Aerial Series to Tuesday + Friday 19:00–20:00 (7.1).
- Mason- let's remove both of these. I'm getting ready to deploy and want the website to be as it will actually function.

**OQ-13 — Membership tier order.** **Default:** price ascending (Silver, Gold, Silver Couple's, Gold Couple's, Gold Family). **Alternative:** grouped by tier (Silver, Silver Couple's, Gold, Gold Couple's, Gold Family), which needs an explicit sort column on products.
- Mason- I'd prefer the alternative.

**OQ-14 — Membership descriptions.** **Default:**
- Silver: "Unlimited Beginner Knotty Yoga classes."
- Silver Couple's: "Unlimited Beginner Knotty Yoga classes for two."
- Gold: "Every class on the weekly schedule. Some classes also need instructor approval."
- Gold Couple's: the Gold text, for two. Gold Family: the Gold text, for three.
- Mason- This sounds fine.

**OQ-15 — Instructor bios.** Shown publicly on the Instructors page. **Default:** one factual line each from the schedule (e.g. Beth: "Teaches Beginner Knotty Yoga on Wednesdays."), editable later in Manage. Send real bios if you have them.
- Mason- I asked my instructors to write bios. Leave a place for me to put them as they send them to me. Worst case, I manually add them once deployed but hopefully they get them to me before then.

**OQ-16 — Leftover copy that no longer matches the studio** *(flagged, not changed by default)*:
- Mail sender name **"Knotty Yoga and Spa"** (`app_secret_values.cpp`).
- About ▸ "Who we are": placeholder text ("making you and your family yoga for over 40 years … hootananny").
- Facility description: "a climbing wall", "Street parking is free after 6pm".

Want real copy for any of these seeded now?
- Mason - Let's do:
	- Who we are
		- Mason started teaching circus acrobatics in 2004 and quickly realized that few people had the strength needed to do aerial acrobatics. He began putting together a conditioning program to solve this and took multiple aerial teacher trainings, the Ana Forrest Yoga instructor training, TRX instructor training, massage school, and multiple Thai massage schools in Thailand. Teaching students for over two decades, he has heavily refined the curriculum to a general purpose workout routine that supports most athletic activities.
	- How we teach
		- There is no magic to fitness: it's just consistency with intensity with technique coaching. Classes present a set of progressions to constantly move to the next level so you see continual improvement over time. You don't need any experience or background in acrobatics. You just need to attend consistently and be willing to do the work!
	- Come and find us
		- We have inch thick padded floors throughout the whole studio so yoga mats are not needed. We also have a grid of 24 aerial fabrics so people generally have their own equipment so there is no time wasted waiting in line for a turn on the equipment. Please come to our next intro workshop.

**OQ-17 — Tell visitors which membership a class needs (Phase 10.3).** Today the class page says only "Members only — Upgrade your membership to attend." (fact 9), so nothing on Beginner Knotty Yoga says Silver is enough. **Default:** build 10.3 (small: one detail field, one endpoint field, one template branch).
- Mason- let's make this change.

**OQ-18 — Who builds.** The Overview says you build and run the tests; the standing 9/11 agreement is that I build Windows and Linux and report each. **Default:** I build both and report, announcing before long or destructive runs. Say if you'd rather I don't for this one.
- Mason- I trust you more now! Please update the note that you build and run the tests on windows and linux but report the results to me. When stuff would break, I'd just paste the output to you so I figure I trust you now to remove myself from this loop as your model has improved.