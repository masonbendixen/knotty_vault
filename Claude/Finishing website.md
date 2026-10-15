---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 10/15/2026
Version: 0.3
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

# Revision notes and working agreement

- **v0.3 (10/15/2026)** — the 10/15 production incident. The site went down at 19:08 UTC because the server never reconnects a dropped database connection. The investigation also found that the scheduled jobs had been refused since the 10/8 deploy. Fixing both, and making `/api/health` tell the truth, is a **new Phase 4**. The old Phases 4–14 move down one, to 5–15, and every reference in this doc was renumbered with them. Phases 2 and 3 are implemented.
- **v0.2 (10/14/2026)** rewritten against Mason's answers to the open questions. The changes (phase numbers as of v0.3):
  - Photos are now Phase 1.
  - Forgot password is a new phase (12).
  - Membership order is a real product sort column (Phase 3).
  - The rolling-availability job is gone (OQ-8).
  - The test-only seed data is retired (Phase 10).
  - Kit and the second Mason account stay in the base seed.
  - Knotty Yoga gets no approval skill (OQ-19).
  - The About page gets Mason's copy (Phase 8).
- **Who builds (OQ-18, answered 10/14).** The Overview's line "I will build the C++ server and run tests myself" is superseded. **Claude builds and runs the tests on both Windows and Linux, fixes what fails, and reports the result for each platform** ("Windows: N passed. Linux: M passed."). Claude announces before long or destructive runs. Git stays Mason's: Claude only reads history, and Mason commits, pushes and tags.

# Current state (verified against the code, 10/14/2026)

What exists today, so the plan says what CHANGES. All seed data lives in `src/database_helper/`: `seed_app_data.{h,cpp}` (the tested `namespace Seed` data seeders) and `create_database.cpp` (bootstrap ordering, admin metadata, and a few seeders that never moved).

| Area | Seeded today | Target |
|---|---|---|
| Classes | Knotty Yoga, Therapeutic Knotty Yoga, Partner Acrobatics, Tumbling, Handstands, Aerial Fabric | Beginner Knotty Yoga, Knotty Yoga, Handstands, Aerial Rope, Beginner Partner Acro, Intermediate Partner Acro |
| Slots | 5: KY Mon/Wed 18:00, Handstands Mon 19:00, Partner Acrobatics Thu 18:00 + Sun 10:00 | 14 (table below) |
| Class access | **No requirement groups on any class**, so every recurring class is "included" and free for everyone, members or not (`ClassCatalogHelper::GetClassDetail`, P-1) | Membership groups on all six, skill groups on three (Phase 6) |
| Skill levels | None seeded | Handstand, Aerial Rope, Intermediate Partner Acro |
| Products | intro-workshop, massage, gold ×3, **4 spa products**, class-dropin | Spa gone; Silver + Silver Couple's added |
| Product order | By **name** everywhere (`TableHelpers::Products`); there is no sort column | New `sort_order` column. Memberships list Silver, Silver Couple's, Gold, Gold Couple's, Gold Family |
| Permissions | instructor, gold_member, platinum_fitness, … | + `silver_member` |
| People | masonbendixen@hotmail.com ("Mason2", pw `mason`, admin), Caleb (`caleb`, admin), Kit Williams (`kit`), masonbendixen@gmail.com (`pass`, admin) | Same four (OQ-10), + Beth Cooper, Gifford Cheung, Tiana Cheung |
| Instructors | Mason, Caleb | + Beth, Gifford, Tiana, with real bios when they arrive |
| Massage providers | Mason **and** Caleb, no availability in the base seed | Caleb only, with his weekly hours |
| About page | Placeholder copy ("40 years … hootananny") | Mason's copy (Phase 8) |
| Test-only seed (`--recreate_database_test`) | Intro Workshop sessions, Aerial Series, Caleb's Mon–Fri 9–5 template | Retired (Phase 10). The local database matches production |
| Password recovery | None. A forgotten password cannot be recovered | **Forgot password?** on the Log In page (Phase 12) |
| Photos | Mason/Caleb profile + instructor; KnottyYoga, PartnerAcro, Handstand class photos; tier icons | All 13 subjects freshly shot (Phase 1), attached in Phase 9 |
| Database connection loss | The server never reconnects: one dropped connection takes the site down until a restart, while `/api/health` still says ok (the 10/15 incident) | Reconnects on its own; health probes the connection requests use (Phase 4) |
| Scheduled jobs | Every job refused with 403 by the CSRF guard since the 10/8 deploy | Jobs run (Phase 4) |

**Facts the plan depends on:**

1. ✅ **Fixed in Phase 2 (10/15/2026)**: every seeder now resolves by natural key. What it said before: **Seed ids are still hardcoded in four places.** `PopulateProductVariants`, `PopulateProductPrices` and `PopulateProductBookingWindows` (all still in `create_database.cpp`) address products 1–9 and variants 1–7 by literal id. `PopulateRoleAssignments` addresses people 1, 2 and 4. Removing the four spa products and adding two Silver ones renumbers everything after them, so **these have to move to natural keys before any catalog change.** That is the same failure mode as the Gold→instructor permission bug found in [[Database helper initialization]] Phase 3.2, and it is still listed there as an open follow-up.
2. **The access gate already exists and is the right tool.** A class's `class_requirement_groups` are AND-ed and each group's literals are OR-ed. A literal is a permission (closure-expanded) **or** a skill level (`ClassAccessHelper::CheckAccess`). "Silver or Gold" is one group with two literals. "Instructor approval" is a skill-level literal, which is exactly what skill levels are for: "Can kick up to handstand against a wall" is the schema's own example.
3. **"Requires attending the preceding class" already exists**: `class_schedule_slots.predecessor_class_schedule_slot_id` (SL-11), seeded today for Handstands. Four slots need it now.
4. **A slot has exactly ONE instructor** (`class_schedule_slots.instructor_person_id`). Tuesday's Beginner Knotty Yoga has two (Gifford **and** Tiana). Per-session `event_session_staffing` can carry several, but only for materialized sessions, and only substitutions and shift trades write it. **Two instructors on a recurring slot is not expressible today**, which is why Phase 11 exists.
5. **Nothing extends provider availability over time.** `ScheduleTemplateHelper::GenerateAvailability` is called only by the seed and by the admin **Generate Availability** button. The seed generates through the end of the following month. **Per OQ-8 that stays manual**: once a month someone presses Generate Availability on Caleb's template (15.4). The generator skips days that already exist, so overlapping ranges are harmless. It also ignores `effective_to_us`, so an open-ended template keeps working.
6. **Production does not pick up seed changes.** Prod was installed with `--install_schema` on 9/23, and every later deploy runs `--migrate`, which evolves the schema and never re-seeds. Phase 15 is how this content reaches the live site.
7. **The test-only seed data collides with the real schedule.** The Intro Workshop (Sat 10–11) sits inside Saturday Knotty Yoga, the Aerial Series (Tue/Thu 19:00) sits on Thursday Intermediate Partner Acro, and Caleb's Mon–Fri 9–5 template duplicates his real hours. **Per OQ-12 all of it goes** (Phase 10). Caleb's template is promoted to real data with his real hours (Phase 7). After that `PopulateTestData` is empty, so the flag goes too (D14).
8. **`ui/src/app/pages/public/services/homepage.service.ts` is dead code.** Only its own spec imports it, and it hardcodes the old six-class list. It gets deleted rather than updated.
9. **Customers never see a requirement group's `inline_description`.** The admin field is labelled "Description (shown to members)", but no customer template renders it. A visitor sees only **"Members only — Upgrade your membership to attend."** on the class page, or the **"Requires a membership"** badge on Our Classes. Nothing tells them *which* membership. The skill side does render: the class page's **"Required Skills"** section lists each skill's name and description, and the **"Skill required"** dialog shows the same. That is why the approval criteria go into the skill descriptions (D6). Phase 13 closes the membership half.
10. **The class page does not show "Requires attending".** The predecessor shows on the weekly **Our Classes** list (`Requires attending: {Class}`) and in the admin slot editor, not on the class detail page. No change planned; the hand-testing steps check it where it renders.
11. **Must-change-password is fully supported.** This answers the question in OQ-2.
    - The server side is honuware platform:
      - `people.must_change_password`
      - `People::SetMustChangePassword`
      - `/api/get_user_info` returns it
      - `/api/update_user_password` clears it
    - The front end is `@honuware/ui/auth`. After sign-in, `LoginComponent` sends a flagged user to the **Change Password** page (`/my/update-user-password`), and `AuthGuard` sends every signed-in page back there until they change it.
    - Today only staff-created quick accounts and the `force_password_change` test-helper command set the flag. Phase 5 adds a command that sets a temporary password **and** the flag together.
12. **All of sign-in lives in honuware, not in this repo.** That means the Log In, Register, Check Email and Verify pages, `/api/login`, `/api/register`, `/api/verify`, the `email_verifications` table, `PersonHelper`, `LoginGate` and the verification mail.
    - The back end is in `server_components` (`components/platform/…`), pinned by `GIT_TAG` in the top-level `CMakeLists.txt`.
    - The front end is in `honuware-web-components` (`@honuware/ui/auth`), exact-pinned in `ui/package.json`.
    - **Forgot password (Phase 12) is therefore built in both honuware repos and then pinned in here.**
    - Honuware has exactly one framework migration (`honuware/0001_site_assets`). This app's migrations ran through `app/0009_about_sections`; Phase 3 added `app/0010_products_sort_order`.
13. ✅ **Fixed in Phase 3 (10/15/2026)**: products now list by `sort_order`, then name. What it said before: **Product order is name order, everywhere.** All three `TableHelpers::Products` listings `ORDER BY name, id`, `CatalogHelper` keeps that order, and the membership hosts only filter `kind === 'subscription'`. Grouping Silver before Gold (OQ-13) therefore needs a column, not a front-end sort.

---

# Design decisions

- **D1 — This is SEED data, built on mechanisms that already exist.** Classes, slots, predecessors, requirement groups, skill levels, entitlement rules, provider templates and photos all have tested tables and helpers. There are three new *mechanisms*:
  - product sort order (Phase 3, OQ-13)
  - co-instructors (Phase 11, fact 4)
  - forgot password (Phase 12, OQ-2)

  Phase 4 (database reconnect, truthful health, scheduled jobs) is a fix to the framework, not a content mechanism.

  Each sits in a phase of its own or at the bottom of its phase, so the content can land and be reviewed without waiting on them. Rolling availability was considered and dropped (OQ-8).
- **D2 — Everything resolves by natural key:**
  - products and variants by `code`
  - price schedule by `name`
  - people by `email`
  - roles and permissions by `name`
  - classes by `name`
  - skills by `code`
  - rooms by `name`

  No literal ids anywhere in the seed when this is done. Phase 2 finishes D5 from [[Database helper initialization]].
- **D3 — Data-driven seed tables, not call sequences.** The fourteen slots, the six classes' access rules, the bios and the photo assignments are each one `constexpr` table of structs that a loop walks. The schedule then reads like the Overview, and a test can iterate the same table the seeder does, so the expected values are not re-typed in the test.
- **D4 — Tests assert through the production READER** ([[feedback_assert_through_the_reader]]):
  - access: `ClassAccessHelper::CheckAccess` for a Silver member, a Gold member, a non-member and an approved Gold member
  - prices and product order: `CatalogHelper`
  - availability: the booking-slot reader
  - instructor names on a session: `ClassCatalogHelper`
  - passwords: `PersonHelper::VerifyPassword`

  Re-reading the columns the seeder just wrote only proves the INSERT ran; the Phase 3.1 `show_on_upcoming` miss is the standing example.
- **D5 — "Silver or Gold" is a two-literal group, not a permission implication.** `permission_implications` could make `gold_member` imply `silver_member`, so the group would need only Silver. But the Overview asks for exactly "silver or gold", and the two-literal group says that literally in the requirements editor. An implication would also silently change what Gold means for every future Silver-gated thing.
- **D6 — Instructor approval is one skill level per gated class, for Handstands, Aerial Rope and Intermediate Partner Acro** (OQ-4).
  - Each skill's description carries the Overview's physical criteria for that class.
  - An instructor grants it on the existing skill-assignment page, and the access gate does the rest.
  - **Knotty Yoga's approval** (OQ-19) and the **Beginner Partner Acro solo-flyer rule** stay in the class descriptions and are checked at the door. The system has no notion of "solo", and Knotty Yoga was left off the skill list in the OQ-4 answer.
- **D7 — Every non-beginner class requires `gold_member`** (OQ-3, confirmed). Without some membership group, a class is open and free to everyone (fact 2's corollary). Beginner Knotty Yoga takes Silver **or** Gold.
- **D8 — Spa removal keeps the room-type vocabulary.** These all go: the four spa products, their variants, prices and product schedule windows, the `Spa` room, and its room schedule template. The `spa` / `swim_spa` **room types** stay: they are reference vocabulary, harmless when unused, and "we don't have the spa *yet*" says it is coming back. This also retires the `late-night-spa` → nonexistent `gold_fitness` permission bug recorded in [[Database helper initialization]]'s follow-ups.
- **D9 — New people never get a password from source control.** Beth, Gifford and Tiana are created with a random, unrecorded password, generated at seed time and never logged. To let one of them in, Mason sets a **temporary password with the must-change flag** (Phase 5), and they choose their own at first sign-in. Once Phase 12 ships, **Forgot password?** works too, given a real email (OQ-1). The four existing accounts keep their committed dev passwords; Mason, Caleb and Kit change theirs on production at deploy time (OQ-10, 15.3).
- **D10 — Photos come first, and the code never waits on them.** Mason shoots them in Phase 1, and Claude converts them and attaches them in Phase 9. `AttachSeedPhoto` already logs and skips a missing file, so one late picture blocks nothing: it lands later as file + table row + test expectation in one change.
- **D11 — Production gets the content by reinstall, not by a data migration** (OQ-9: "Wipe away"). Prod is in preview on the Square sandbox. `--install_schema --force` plus the `set_secret` block is already verified end to end (Deploying to AWS 5.1 step 4). An in-place migration would have to delete spa products under purchases, rename classes under bookings and re-key slots, all for smoke-test data.
- **D12 — Product display order is a server-side `sort_order` column** (OQ-13). The tiers then group Silver, Silver Couple's, Gold, Gold Couple's, Gold Family.
  - Every listing (catalog, Services, Memberships, the home band, Manage Products) shares one order instead of each host sorting its own way.
  - The studio changes it in Manage Products with a **Sort Order** field.
  - The default is 0, with ties falling back to name, so every product nobody ordered keeps today's alphabetical place.
- **D13 — Forgot password is a honuware feature with the standard reset-link safeguards.** Fact 12 is why it lives in honuware.
  - The response is the same whether or not the account exists, and is not timed differently either.
  - The link is single-use and expires in 60 minutes.
  - The token is stored hashed.
  - Wrong guesses are capped per link, and requests are rate-limited per email and per IP.
  - A successful reset ends every session and remember-me token, clears must-change and the lockout counter, and marks the email verified, since clicking the link proves the inbox.
- **D14 — The test-only seed is retired along with its flag** (OQ-12). With the Intro Workshop sessions and Aerial Series deleted and Caleb's template promoted to real data, `PopulateTestData` has nothing left. `--recreate_database_test` would be a second name for `--recreate_database`, so it goes, and every local database is what production gets. Hand-testing walkthroughs use plain `--recreate_database`; Kit stays a base-seed member (OQ-10).

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

### Class access (requirement groups: AND across columns, OR inside a cell)

| Class | Membership group | Instructor-approval group (skill level) |
|---|---|---|
| Beginner Knotty Yoga | `silver_member` **or** `gold_member`: "Silver or Gold membership" | none |
| Knotty Yoga | `gold_member`: "Gold membership" | none. Approval is checked at the door (OQ-19) |
| Handstands | `gold_member` | `handstand` |
| Aerial Rope | `gold_member` | `aerial_rope` |
| Beginner Partner Acro | `gold_member` | none (solo-flyer rule is description text, D6) |
| Intermediate Partner Acro | `gold_member` | `intermediate_partner_acro` |

Skill levels (OQ-4: named for the skill, described with the Overview's criteria). The name and description are what members see in **Required Skills** and the **Skill required** dialog:

| Code | Name | Description |
|---|---|---|
| `handstand` | Handstand | Able to easily kick up to a wall and hold a belly to the wall handstand for thirty seconds. |
| `aerial_rope` | Aerial Rope | Able to do three pullups and two straddle climbs starting in the air. |
| `intermediate_partner_acro` | Intermediate Partner Acro | A solid foot to hand, free star, and thirty second shoulder stand. |

### Memberships (descriptions per OQ-14, confirmed)

| Sort | Code | Name | $/month | Seats | Grants | Tier icon | Description |
|---|---|---|---|---|---|---|---|
| 10 | `silver-membership` | Knotty Yoga Silver Membership | 160 | 1 | `silver_member` | `tier_icon_solo.svg` | Unlimited Beginner Knotty Yoga classes. |
| 20 | `silver-couples` | Knotty Yoga Silver Couple's Membership | 240 | 2 | `silver_member` | `tier_icon_couple.svg` | Unlimited Beginner Knotty Yoga classes for two. |
| 30 | `gold-membership` | Knotty Yoga Gold Membership | 200 | 1 | `gold_member` | `tier_icon_solo.svg` | Every class on the weekly schedule. Some classes also need instructor approval. |
| 40 | `gold-couples` | Knotty Yoga Gold Couple's Membership | 300 | 2 | `gold_member` | `tier_icon_couple.svg` | Every class on the weekly schedule, for two. Some classes also need instructor approval. |
| 50 | `gold-family` | Knotty Yoga Gold Family Membership | 400 | 3 | `gold_member` | `tier_icon_family.svg` | Every class on the weekly schedule, for three. Some classes also need instructor approval. |

Every other product keeps `sort_order` 0 and therefore its alphabetical place (D12).

### Caleb's massage hours (`schedule_template_entries`, **ISO: Mon = 1 … Sun = 7**)

| Day | ISO | Hours | Minutes |
|---|---|---|---|
| Mon | 1 | 11:00–15:00 | 660–900 |
| Tue | 2 | 11:00–16:00 | 660–960 |
| Thu | 4 | 11:00–16:00 | 660–960 |
| Fri | 5 | 11:00–15:00 | 660–900 |
| Sun | **7** | 13:30–16:00 | 810–960 |

None of these overlaps a class Caleb teaches (Sun 10–11, Mon 17–18, Thu 17–18); Phase 7.3 pins that with a test. ⚠️ Sunday is **7** here and **0** in `class_schedule_slots` ([[gotcha_two_day_of_week_conventions]]). A literal `0` in a template entry silently generates nothing.

### About page (OQ-16: Mason's copy, seeded verbatim)

The three blocks keep their ordinals and their links. Only the body text changes.

| # | Title | Body | Link (unchanged) |
|---|---|---|---|
| 10 | Who we are | Mason started teaching circus acrobatics in 2004 and quickly realized that few people had the strength needed to do aerial acrobatics. He began putting together a conditioning program to solve this and took multiple aerial teacher trainings, the Ana Forrest Yoga instructor training, TRX instructor training, massage school, and multiple Thai massage schools in Thailand. Teaching students for over two decades, he has heavily refined the curriculum to a general purpose workout routine that supports most athletic activities. | **Get started** |
| 20 | How we teach | There is no magic to fitness: it's just consistency with intensity with technique coaching. Classes present a set of progressions to constantly move to the next level so you see continual improvement over time. You don't need any experience or background in acrobatics. You just need to attend consistently and be willing to do the work! | **See the class schedule** |
| 30 | Come and find us | We have inch thick padded floors throughout the whole studio so yoga mats are not needed. We also have a grid of 24 aerial fabrics so people generally have their own equipment so there is no time wasted waiting in line for a turn on the equipment. Please come to our next intro workshop. | **Our location** |

---

# Instructor bios (Mason: paste them here)

*Whatever is under each name when Phase 5.5 is implemented is seeded **verbatim**. A name with nothing pasted gets the one-line default shown, so the Instructors page never shows an empty card. Bios can always be changed later under **Admin** → **Manage Products** → **Instructors**. If a bio arrives after the production rollout, edit it there rather than re-seeding.*

### Mason Bendixen
Default (today's seed): "Teaches Knotty Yoga, handstands and partner acrobatics."

*(paste bio here)*

### Caleb Ault
Default (today's seed): "Teaches Knotty Yoga and aerial work."

*(paste bio here)*

### Beth Cooper
Default: "Teaches Beginner Knotty Yoga on Wednesdays."

*(paste bio here)*

### Gifford Cheung
Default: "Co-teaches Beginner Knotty Yoga on Tuesdays with Tiana Cheung."

*(paste bio here)*

### Tiana Cheung
Default: "Co-teaches Beginner Knotty Yoga on Tuesdays with Gifford Cheung."

*(paste bio here)*

---

# Phase 1 — Take the photos (Mason)

The later phases use these: Phase 9 attaches them and Phase 14's walkthrough looks for them. Nothing else blocks on them (D10), so the phases below can proceed while the shoot happens.

### 1.1 Shot list

How the site crops each kind, so the subject survives the crop:
- **Headshots:** head and shoulders, centred, with room on every side. Instructor cards crop to fill; Our Providers crops square.
- **Class photos:** landscape, with the action in the middle. The class page crops to 4:3.
- **Studio:** wide landscape. About ▸ Our Location crops to 16:10.
- **Phone setting:** JPEG, not HEIC. On an iPhone that is Settings ▸ Camera ▸ Formats ▸ **Most Compatible**.

- [ ] 1. Mason Bendixen headshot → `MasonInstructor.jpg` (replaces today's)
- [ ] 2. Caleb Ault headshot → `CalebInstructor.jpg` (replaces today's)
- [ ] 3. Beth Cooper headshot → `BethInstructor.jpg`
- [ ] 4. Tiana Cheung headshot → `TianaInstructor.jpg`
- [ ] 5. Gifford Cheung headshot → `GiffordInstructor.jpg`
- [ ] 6. Beginner Knotty Yoga class → `BeginnerKnottyYoga.jpg`
- [ ] 7. Knotty Yoga class → `KnottyYoga.jpg` (replaces today's, if reshot)
- [ ] 8. Aerial Rope class → `AerialRope.jpg`
- [ ] 9. Handstands class → `Handstand.jpg` (replaces today's, if reshot)
- [ ] 10. Beginner Partner Acro class → `PartnerAcro.jpg` (replaces today's, if reshot)
- [ ] 11. Intermediate Partner Acro class → `IntermediatePartnerAcro.jpg`
- [ ] 12. Massage, shown on the Services card for Massage → `Massage.jpg`
- [ ] 13. General studio, shown on About ▸ Our Location → `Studio.jpg`

### 1.2 Hand-off and conversion
- [ ] Mason drops the originals into **`C:\Users\mason\Documents\Obsidian\Knotty Yoga\Photos\`** (OQ-11 default), named as in 1.1. Any extension and any size is fine. That folder is outside the git repo, so phone-sized originals never get committed by accident.
- [ ] Claude converts each into `server/knottyyoga_server/src/database_helper/img/` with **ffmpeg**:
  - JPEG, long edge 2000 px, `-q:v 3`, under about 1.5 MB.
  - Phone rotation applied.
  - **All metadata stripped** (`-map_metadata -1`), because phone photos carry GPS coordinates.
  - Never `convert`: on Windows that resolves to the FAT-to-NTFS disk converter.
- [ ] Claude opens every converted file to check orientation and framing against 1.1, and lists anything that needs a reshoot in this phase before Phase 9 wires it.

---

# Phase 2 — Seed plumbing: natural keys everywhere

Prerequisite for every catalog and people change (fact 1). No content changes in this phase: a fresh database must come out **identical** to today's, which is what makes it safe to land first.

### 2.1 Move the remaining id-addressed seeders into `seed_app_data` *(seed)*
- [x] Move `PopulatePeople`, `PopulateRoleAssignments`, `PopulateProductVariants`, `PopulateProductPrices` and `PopulateProductBookingWindows` from `create_database.cpp`'s anonymous namespace into `seed_app_data.{h,cpp}` (`namespace Seed`). Keep the same `(Transaction&, DatabaseHelper, DatabaseInfo)` signature, so a test can call them.
  - Each is now a `constexpr`-style table of structs walked by a loop (D3), in the same insertion order as before, so a fresh database numbers its rows exactly as it always has.
  - New constant `kMassageProductCode`, used by every massage reference in the seed.
- [x] Variants: resolve the product by code (`massage`, the four spa codes).
- [x] Prices: resolve the product by code, the variant by `product_variants.code`, and the price schedule by `kSeedPriceScheduleName`. Add a `LookupProductVariantIdByCode` helper beside the existing `Lookup*` ones (same contract: logs and returns 0).
  - Variant codes are unique across all products, so the code alone names the row.
  - A missing variant **skips** that price rather than writing it at product level, which would charge every massage length the same.
  - A missing price schedule logs `event=product_prices_seed_skipped` and writes nothing.
- [x] Booking windows: resolve `massage` by code.
- [x] Role assignments: resolve people by email. `RoleIdByName` moved alongside, into `seed_app_data.h`. It still throws on a missing role, because roles are created earlier in the same bootstrap. `PermissionIdByName` stays in `create_database.cpp`, where all of its callers are.
  - **Correction to v0.2:** this item said "a new `kMasonLiveEmail` for the gmail account", but `kMasonEmail` already *is* the gmail account (`masonbendixen@gmail.com`). The one without a constant was the **hotmail** account (old person 1, "Mason2"). It is now `kMasonHotmailEmail`. Kit got `kKitEmail` so `PopulatePeople` spells every address once.
- [x] Move the trailing `UPDATE products … WHERE code = 'massage'` (provider type + required room type) into a named `Seed::PopulateServiceProductLinks`, resolved by code rather than by a subquery inside a string. It writes through `DbCrud::UpdateRow`, and if the product, provider type or room type is missing it logs and writes neither column.
- [x] The bootstrap order in `PopulateTables` is unchanged. `PopulateServiceProductLinks` sits exactly where the `UPDATE` was, after the provider and room types it needs.

### 2.2 Tests *(seed_app_data_test.cpp)*
- [x] Prices: `PopulateProductPricesResolvesRowsByNaturalKeyNotSeedOrder`.
  - A decoy product, variant and **inactive** price schedule are inserted first.
  - The 60/90/120-minute massage variants price at 16000/22000/30000 through `CatalogHelper::GetVariantListPrice`. `ResolveBestPriceForPerson` is product-level only and cannot see a variant price, so it is the wrong reader for those three.
  - The intro workshop (900) and the three Gold memberships (20000/30000/40000) price through `CatalogHelper::ResolveBestPriceForPerson`, as an anonymous visitor.
  - No price landed on any decoy.
- [x] `PopulateProductPricesNeverPricesAVariantServiceAtProductLevel`: with no variants seeded, massage has no price at all, while the intro workshop still prices at 900.
- [x] `PopulateProductPricesSkipsWithoutThePriceSchedule`.
- [x] Variants: `PopulateProductVariantsGivesMassageItsThreeLengths`, through `CatalogHelper::GetVariantsForProduct`, with a decoy product first.
- [x] Role assignments: `PopulateRoleAssignmentsResolvesPeopleByEmailNotSeedOrder`, through `PersonHelper::GetRolesForUser`.
  - A decoy person is created first.
  - Both Mason accounts are `admin`, Caleb is `admin` + `instructor`, Kit has no roles, and the decoy has none.
- [x] People: `PopulatePeopleSeedsTheAccountsTheHandTestsSignInWith`. All four accounts sign in with their committed dev passwords (`PersonHelper::VerifyPassword`), with the right first names. The hand-testing walkthroughs depend on exactly these.
  - Both people tests store libsodium's minimum Argon2 cost in `config_secrets` first. Otherwise `PopulatePeople`'s production cost made them take 15 s and 8 s on Windows Debug; now they take under a second.
- [x] Booking window: `PopulateProductBookingWindowsLetsAnyoneBookMassageAWeekAhead`, through `ProductBookingWindows::ResolveAdvanceDaysForUser`. Massage gives 7 days and the decoy product gives 0.
- [x] `PopulateServiceProductLinksPointsMassageAtItsProviderAndRoom`.
  - Decoy provider and room types are inserted first.
  - It reads through `TableHelpers::Products::GetProduct`, the same read `ServiceAvailabilityHelper::ComputeAvailableSlots` starts with.
  - The intro workshop gets no provider type.
- [x] `PopulateServiceProductLinksSkipsWithoutTheProviderType`: neither column is written.
- [x] `LookupsReturnZeroWhenTheRowIsAbsent` also covers `LookupProductVariantIdByCode`.
- [x] `IntroWorkshopSessionsAppearOnBothTheHomePageAndEventsFeeds` hand-inserted its own price, because `PopulateProductPrices` "still addresses products by hardcoded ids". It now calls the real `PopulateProductVariants` + `PopulateProductPrices`.

### 2.3 Verification
- [x] **"Identical to today's", checked end to end.** No unit test can run the whole bootstrap, so the new `knottyyoga_database_helper --install_schema` seeded a throwaway database (`HONUWARE_DB_NAME=knottyyoga_phase2_check`). Its people, role assignments, products (including the provider and room links and the spa settings), variants, prices, booking windows and price schedule were then diffed against the local `knottyyoga` database built by the old code. The result: **identical, ids included.** The throwaway database was dropped afterwards, and the local `knottyyoga` database was only read.
- [x] Build and full test suite on both platforms (10/15/2026):
  - **Windows** (`x64-Debug`, all targets): **5193 passed.**
  - **Linux** (container gate, plus `knottyyoga_database_helper` and `knottyyoga_the_server` built): **5193 passed.** It ran against the local honuware checkout, which is clean and identical to the pinned SHA `e662f4db`.

---

# Phase 3 — Products, memberships and their display order

### 3.1 Schema: `products.sort_order` *(db_schema)*
- [x] `kProductsSortOrder = "sort_order"`, `NOT NULL DEFAULT 0`, in `MakeProductsTable`. It is **`BIGINT`**, not the `INT` written here before, to match `product_variants.sort_order`.
- [x] Admin metadata in `create_database.cpp`: column data info (`number` edit, "Display order wherever products are listed…") and friendly name **Sort Order**.
- [x] Idempotent migration **`app/0010_products_sort_order`** (`ADD COLUMN IF NOT EXISTS sort_order BIGINT NOT NULL DEFAULT 0`) ([[gotcha_new_table_needs_migration]]). Prod gets the column through the reinstall anyway, but every other database needs it.
- [x] Migration test `AppMigrationsTest.ProductsSortOrderAddsTheColumnAndIsIdempotent`. It drops the column with a product already in the table, applies the migration twice, and checks two things: the existing product reads back 0, and the subscription listing then orders by the new column.

### 3.2 Table helper *(sql_util/table_helpers)*
- [x] `TableHelpers::Products`: all three listing queries order by `sort_order ASC, name ASC, id ASC`.
- [x] New `Products::SetSortOrder(transaction, id, sortOrder)`, which the duplicate action and the tests use. It stamps `updated_us` like the other setters.
- [x] Tests: `SetSortOrderBasic`, plus one per listing query: `GetProductsListsBySortOrderBeforeName`, `GetActiveProductsListsBySortOrderBeforeName`, `GetProductsByKindListsBySortOrderBeforeName`. Each uses names whose alphabetical order is the *reverse* of the sort order, so a query still sorting by name fails. `GetProductsBreaksSortOrderTiesByName` covers the fallback: unordered products keep today's alphabetical order.

### 3.3 Business logic *(business_logic/payment)*
- [x] `CatalogHelper` keeps the table helper's order. As expected, no code change. Verified by `SeedAppDataTest.MembershipsListInTierOrderAtTheirMonthlyPrices` (3.10), which reads `CatalogHelper::GetCatalog`. Here the order **is** the contract, so asserting positions is right; CLAUDE.md's "don't assume order" is about unordered results.
- [x] `ProductHelper::DuplicateProduct` copies column by column, so it now carries `sort_order` across via `SetSortOrder`. Test `DuplicateProductKeepsTheSourcesSortOrder`: a copy of Gold (30) lists right after Gold, ahead of a 50.

### 3.4 Silver permission *(seed)*
- [x] `PopulatePermissions`: add `silver_member`, "Knotty Yoga Silver Membership.", `is_pricing_eligible = true`, right after `gold_member`. Add the name constant `kSilverMemberPermissionName` beside `kGoldMemberPermissionName`.

### 3.5 Product list *(seed)*
- [x] `PopulateProducts`: **remove** `early-bird-spa`, `non-peak-spa`, `peak-spa`, `late-night-spa`.
- [x] **Add** `silver-membership` and `silver-couples` (kind `subscription`), named as in the Memberships table.
- [x] Set all five memberships' descriptions and `sort_order` from the Memberships table. `PopulateProducts` is now a table of structs (D3). The other three products keep `sort_order` 0.

### 3.6 Variants, prices, entitlements *(seed)*
- [x] `PopulateProductVariants`: massage 60/90/120 only.
- [x] `PopulateProductPrices`: drop the spa rows; add Silver 16000 and Silver Couple's 24000 (monthly, no variant, like Gold).
- [x] `PopulateProductEntitlementRules`: `silver-membership` → 1 seat, `calendar_month`, `silver_member`; `silver-couples` → 2 seats, same.

### 3.7 Remove the spa infrastructure *(create_database.cpp bootstrap tail)*
- [x] Delete the spa `UPDATE`s, the `Spa` room schedule template and its seven entries, and every `product_schedule_windows` row (all four belonged to spa products). This also retires the `late-night-spa` → nonexistent `gold_fitness` visibility bug.
- [x] `PopulateLocationRooms`: drop the `Spa` room. Room types stay (D8).

### 3.8 Tier icons *(seed)*
- [x] `PopulateSeedPhotos`' product table: `silver-membership` → `tier_icon_solo.svg`, `silver-couples` → `tier_icon_couple.svg`.

### 3.9 Frontend: Manage Products
- [x] `ProductDetailComponent`: a **Sort Order** number field in the **Product Information** card, right after **Kind**, hinted "Lower numbers list first. Ties list by name." **Save Changes** saves it with the rest through the generic `update_item`, so no new endpoint.
  - A cleared field saves as `0`, because the column is `NOT NULL`.
  - Decimals are dropped, because it is an integer.
  - A product row without the column reads as `0`.
- [x] **The product list** (Manage ▸ Products) uses the same order, as D12 asks. The generic table query sorts by one column, so the server still sorts by name, and `sortProductRows` stable-sorts by `sort_order` on top. Ties keep the name order. `ProductRow` gains `sort_order`.
- [x] **Changed from v0.2:** `CatalogProduct` does **not** gain `sort_order`. The catalog's order *is* the contract and nothing displays the number, so adding it would only invite a host to re-sort.
- [x] Specs (108 pass in the three files):
  - `product-detail`: the field loads the product's value; **Save Changes** sends the edited value; a cleared field sends `'0'`; a product without the column reads as 0.
  - `product-list`: a name-ordered server answer renders as sort order with name ties. Checked on both the array and the rendered table rows.
  - `MembershipTierCardsComponent` renders cards in **input** order, which pins that no card re-sorts what the server ordered. The three hosts (Memberships page, home band, shop) only filter `kind === 'subscription'`; none of them sorts.

### 3.10 Seed tests
- [x] `PopulateProductsSeedsExactlyTheStudiosProducts`: exactly the eight product codes through `Products::GetActiveProducts`, and **no product whose code contains `spa`**. I kept `PopulateProductsSeedsTheDropInClassProduct` alongside it rather than rewriting it, because it pins the drop-in's kind and the workshop's capacity.
- [x] `SilverProductsGrantTheSilverMemberPermission`, modelled on the Gold test: seats 1/2, `calendar_month`, "the Silver and Gold permission ids differ", and the permission is pricing-eligible.
- [x] Membership prices and order through `CatalogHelper::GetCatalog`, both in `MembershipsListInTierOrderAtTheirMonthlyPrices`:
  - order: Silver, Silver Couple's, Gold, Gold Couple's, Gold Family
  - prices: 16000 / 24000 / 20000 / 30000 / 40000
- [x] `PopulateSeedPhotosGivesEachMembershipTierItsIcon`: five tiers now.
- [x] Spa removal:
  - **No `Spa` room:** `PopulateLocationRoomsSeedsTheGymAndMassageRoomOnly`, through the facility-filtered rows query that Manage ▸ Locations uses. It also checks that the `spa` room *type* survives.
  - **No product schedule windows and no room schedule templates:** these lived in the bootstrap tail, which no unit test can call, so they were checked end to end instead (3.11).

### 3.11 Verification
- [x] **Fresh seed, end to end.** `knottyyoga_database_helper --install_schema` was run into a throwaway database (`knottyyoga_phase3_check`), which was dropped afterwards. The local `knottyyoga` database was not touched. The seed came out as follows:
  - **Products:** eight, listing Class Drop-In, Intro Workshop, Massage (all 0, by name), then the five tiers at 10–50 with the OQ-14 descriptions.
  - **Variants and prices:** three massage variants. Prices are 900 / 16000 / 22000 / 30000 / 16000 / 24000 / 20000 / 30000 / 40000.
  - **Entitlements:** Silver ×1/×2 grants `silver_member`; Gold ×1/×2/×3 grants `gold_member`.
  - **Rooms:** Main Gym and Massage Room 1. All six room types are still there, `spa` included.
  - **Spa leftovers:** **0** room schedule templates, **0** template entries, **0** product schedule windows.
  - **Tier icons:** on all five tiers.
  - **Massage:** still linked to `massage_therapist` / `massage_room`.
  - **Admin metadata:** `products.sort_order` is labelled **Sort Order**.
- [x] Full suites (10/15/2026):
  - **Windows** (`x64-Debug`, all targets): **5204 passed.**
  - **Linux** (container gate, plus `knottyyoga_database_helper` and `knottyyoga_the_server` built): **5204 passed.**
  - **Angular:** **3588 of 3588** specs pass. `ng build` (production) succeeds, and `ng lint` reports all files passing. The build's CSS budget warnings were already there; this phase changed no stylesheet.

---

# Phase 4 — Production reliability: reconnect, a truthful health check, and the scheduled jobs *(honuware)*

**The 10/15 incident (19:08 UTC).** The site looked hung. What the server's journal showed:
- **The site's database connection dropped, and the server never replaced it.**
  - Every request runs on the site's own pooled connection (tenancy's pool of 1). `DatabaseHelperProd` opens that connection once, at startup (`database_helper.cpp:38`), and has no way to reopen it.
  - From 19:08:46 on, every request on it failed with *"Lost connection to the database server"*. The site stayed down until `systemctl restart`.
- **`/api/health` kept answering `db: ok` the whole time.**
  - It does make a database call (`SELECT 1`), but through `webApp->GetTransactionProvider()`, the deployment's *global* connection. That is a different socket from the one requests use.
  - So the answer to "should the health check call the database?" is that it already does, on the wrong connection.
- **The outage looked like a wrong password.**
  - `/api/login` answers 401 "The email or password you entered is incorrect" for *any* exception (`login.cpp:33-34`), without logging why. Mason's 16 sign-in attempts at 19:09 all failed that way. His password was fine, and nothing could record the failures, so no lockout either.
  - Session restore swallows every error (`EndpointAuthHelper::Initialize`), so signed-in people silently appeared signed out.
- **Unrelated but found alongside it: no scheduled job had run since the 10/8 deploy.**
  - Every helper call got 403 from `CsrfGuard`: *"csrft cookie missing"*. That covers billing, reminders, waitlist refunds, digests and attendance finalisation, about 200 refusals a day for a week.
  - `Http::HttpResponse::headers` is a `std::map`. The login reply sets two cookies (`session_token` and `csrft`), and the second `Set-Cookie` overwrites the first, so the helper keeps one. It also never sends the `X-CSRF-Token` header the guard compares against (`csrf_guard.cpp:107-113`).

All the code is in `server_components`; knottyyoga only moves its pin. During development knottyyoga builds against the local checkout (`FETCHCONTENT_SOURCE_DIR_HONUWARE`), as Phase 12 does. This phase ships to production **on its own release (4.6), ahead of everything after it**: it is a fix to what is live today.

### 4.0 Now, by hand
- [ ] `sudo systemctl restart knottyyoga-server` on the EC2, then `curl -sS http://localhost/api/health` and a page load through the CloudFront URL. This is the only cure until 4.6 ships.
- [x] **Why did the connection drop?** Found 10/15:
  - **The connection did not die at 19:08.** That is when Mason's visit first *used* it. The last request that used the database successfully was **10/14 22:36 UTC**; the helper's refused jobs never reach the database, so the connection sat idle for 20½ hours.
  - **RDS events for the window:** only the nightly backup (09:06–09:09 UTC), which does not drop connections.
  - **The RDS Postgres log** (read in CloudShell): at **06:39:22 and 06:39:55 UTC** it logged `could not receive data from client: Connection timed out` for **two** connections from the EC2 (`172.31.2.31`, user `knottyyoga`). Postgres's keepalive probes went unanswered, so it hung up.
  - **The EC2's journal:** at **06:35:15** Ubuntu's `unattended-upgrade` ran. At **06:35:30** its new `systemd` package restarted **`systemd-networkd`**, which re-enumerated `ens5`, `docker0` and the server container's `veth`, and re-acquired the DHCP address.
  - **Why that breaks the database connections.** The server container's outbound connections go through Docker's MASQUERADE NAT, and the kernel drops MASQUERADE's connection-tracking entries when the host address goes away. RDS's keepalives then had nowhere to land, and RDS hung up four minutes later. The server process was never told and kept the dead socket (4.1).
  - **Not fully explained:** why the global connection behind `/api/health` survived. Probably something used it inside the 06:35–06:39 window, which re-created the NAT entry before RDS gave up. It does not change the conclusion or the fix.
  - **It will recur.** Unattended-upgrades runs daily; any update to `systemd` or `netplan` does this again. 4.1 makes the server shrug it off, and 4.8 stops the host from causing it.

### 4.1 Reconnect a dropped connection *(server_components: data/sql_util/database_access)*
- [ ] `DatabaseHelperProd` keeps the `DatabaseHelperInit` it was built from and gains `ResetConnection()`, which opens a fresh `pqxx::connection`. Exposed through `DatabaseHelper`.
- [ ] `ProductionTransactionProvider::RunInTransaction`, still under its mutex:
  - **Before BEGIN:** if the connection is closed, reconnect.
  - **If BEGIN itself throws `pqxx::broken_connection`:** reconnect once and begin again. This is safe because none of the caller's work has run yet.
  - **If the connection breaks mid-transaction:** reset it so the *next* transaction reconnects, and rethrow. The caller's work is **never re-run automatically**, because it may already have done something outside the database (a Square charge, an email).
- [ ] TCP keepalives in the connection string (`keepalives=1 keepalives_idle=60 keepalives_interval=10 keepalives_count=3`), so a silently dead socket is noticed in about 90 seconds rather than by the next customer.
- [ ] One log line per reconnect: `[database] event=reconnected database=<name> reason=<…>`, so the journal says when it happened.
- [ ] `PooledTransactionProvider` needs nothing of its own: each slot is a `ProductionTransactionProvider`.
- [ ] Tests, against the test database, with a production provider pointed at it. A second connection kills the provider's backend with `pg_terminate_backend`, which is exactly what a dropped connection looks like to libpq:
  - after the kill, the next transaction **succeeds**
  - a transaction running when its backend dies throws, is **not** re-run (a counter in the callback proves it ran once), and the transaction after it succeeds
  - a pool of 1 recovers the same way
  - the keepalive parameters appear in `GetConnectionString()`

### 4.2 A health check that tells the truth *(server_components: platform/endpoints/health)*
- [ ] `/api/health` probes the connections requests actually use:
  - the global provider
  - **every tenant pool** in the `TenantResourceRegistry`
  - the request's resolved tenant, if any (resolving creates its pool)

  It reports `db: ok` only if all of them pass. With 4.1 in place a probe on a dropped connection reconnects and passes, so health fails only when the database really cannot be reached. That is what it should mean.
- [ ] On failure: still 503, and the body names what failed, e.g. `{"status":"fail","db":"fail","failed":["knottyyoga"]}`. The 200 body is unchanged, so the deploy script's check keeps working.
- [ ] Tests:
  - a registry holding a tenant whose provider throws → 503, naming it
  - everything healthy → 200 and today's body
  - no registry at all (tenancy inert) → behaves as today

### 4.3 Errors that say what they are *(server_components: foundation/util + platform/endpoints)*
- [ ] `ErrorResponse::ServiceUnavailable(message)` → **503**. Today there is no 503 helper. Test.
- [ ] `/api/login`:
  - A database failure (`pqxx::broken_connection`, `pqxx::sql_error`) is logged with context (`event=internal_error_context`) and answered **503** "Sign-in is temporarily unavailable. Please try again in a minute."
  - A wrong password still gets the same generic 401 as today. Nothing is revealed to a guesser.
- [ ] `EndpointAuthHelper::Initialize`: the swallowed session-restore error is logged at ERROR, rate-limited to once a minute, so "everyone looks signed out" leaves a trace.
- [ ] Tests: login over a provider that throws `broken_connection` → 503 and a logged context line; a wrong password → 401 as before; a session-restore failure logs once per minute, not per request.

### 4.4 The scheduled jobs get past the CSRF guard *(server_components: foundation/util/http + scheduler)*
- [ ] `Http::HttpResponse` gains `std::vector<std::string> setCookies`, filled by the curl header callback with **every** `Set-Cookie`. The `headers` map keeps today's behaviour for everything else. Tests: two `Set-Cookie` lines both survive; the test HttpClient can return several.
- [ ] `ApiClient::Login` stores every cookie. `CallEndpoint` sends them all, and on any non-GET call adds `X-CSRF-Token: <csrft>`, the double-submit the guard checks. The re-login-on-401 path refreshes both.
- [ ] Not exempting the helper from CSRF instead: an exemption would mean trusting something the request says about itself. Doing the double-submit keeps one rule for every caller.
- [ ] Tests:
  - `ApiClient` with a two-cookie login sends both cookies and the header
  - GET sends no header
  - re-login replaces both
  - **One end-to-end test:** `ApiClient` against the real web app's middleware (CsrfGuard included), through a test HttpClient that hands each request to `handle_full`. This is the test that would have caught the 10/8 regression; nothing exercised the helper against the guard.

### 4.5 Ship the server side
- [ ] Gates on `server_components` (Linux + Windows) and on knottyyoga against the local checkout; Claude reports both platforms.
- [ ] **Mason** commits and pushes `server_components`.
- [ ] Claude bumps `GIT_TAG` in knottyyoga's `CMakeLists.txt`, removes the local override, and re-runs the knottyyoga gates against the pinned SHA.
- [ ] CommunityFinder has the same three bugs. It moves when Mason bumps its pin. Its helper is refused by the same guard if it is deployed.

### 4.6 Release to production, ahead of the rest of the plan
- [ ] **Mason** tags the release (v1.0.4) the normal way. The deploy's `--migrate` also applies Phase 3's `app/0010_products_sort_order` (adds the column; no data changes). The seed changes from Phases 2–3 do not reach production (fact 6).
- [ ] On the EC2 afterwards:
  - `curl -sS http://localhost/api/health` → `{"db":"ok",…}`.
  - `sudo journalctl -u knottyyoga-helper -n 50 --no-pager` shows `event=job_success` lines within the hour, and no `status=403`.
- [ ] **The week of missed jobs.** On the Square sandbox nothing real was lost. Billing and grace-period jobs pick up whatever is due on their next daily run. Reminders for classes that already happened are simply never sent.

### 4.7 Watch it from outside (OQ-21)
- [ ] **Default:** a Route 53 health check on `https://dv1tgxa9ok30f.cloudfront.net/api/health`.
  - HTTPS, every 30 seconds, passing only if the body contains `"db":"ok"`.
  - A CloudWatch alarm on it emails `masonbendixen@gmail.com` through SNS after 2 failures in a row.
  - About $1 a month.
  - The exact console clicks get written into Deploying to AWS when this is done.
- [ ] Only useful once 4.2 lands, since today's health check would have stayed green through this incident.

---

# Phase 5 — People, roles and instructors

### 5.1 Temporary password *(business logic; answers OQ-2)*
- [ ] New `business_logic/accounts/temporary_password.{h,cpp}` in `knotty_yoga_core`: `Accounts::SetTemporaryPassword(transaction, databaseHelper, secrets, email, password)`.
  - Calls honuware's `PersonHelper::UpdatePassword` with the production secrets, so the hash gets production Argon2 cost.
  - Then `People::SetMustChangePassword(true)`.
  - Returns false for an unknown email.
  - It lives in this repo rather than in honuware so Phase 5 does not wait on a honuware release.
- [ ] Tests `temporary_password_test.cpp`:
  - the new password verifies and the old one no longer does, both through `PersonHelper::VerifyPassword`
  - `/api/get_user_info`'s reader reports `must_change_password`
  - an unknown email returns false and changes nothing

### 5.2 `set_password` command *(test_helper)*
- [ ] `knottyyoga_test_helper` command `set_password`, alias `sp`, group "Users & Permissions", with `--email` and `--password`. It is a thin call to 5.1. It prints "… must choose a new password at next sign-in", and a clear error for an unknown email. Works against prod the same way `set_secret` does.
- [ ] `test_helper` is not linked into the test executable, which is why the logic lives in 5.1, where it is tested.

### 5.3 People *(seed)*
- [ ] `PopulatePeople`: add Beth Cooper, Gifford Cheung and Tiana Cheung, emails per **OQ-1**, via `CreateFullyValidatedUser` with a **random, unrecorded** password (D9): 32 bytes from the existing secure-random utility, base64, never logged.
- [ ] Email constants `kBethEmail`, `kGiffordEmail`, `kTianaEmail` in `seed_app_data.h`.
- [ ] Kit Williams and `masonbendixen@hotmail.com` stay exactly as they are (OQ-10).

### 5.4 Roles *(seed)*
- [ ] `PopulateRoleAssignments`: Instructor role for Mason, Caleb, Beth, Gifford and Tiana. Mason and Caleb keep Administrator.
- [ ] `PopulateRolePermissions`: grant the Instructor role `staff_access` (door check-in) and `manage_skills` (granting the approval skills) (OQ-5, confirmed).

### 5.5 Instructor profiles *(seed)*
- [ ] `PopulateInstructors`: five rows, data-driven, with bios taken **verbatim** from the "Instructor bios" section above, or each name's default where nothing was pasted (OQ-15). The photo column moves to Phase 9's table.

### 5.6 Tests
- [ ] All five instructors exist with the expected bios, read through `InstructorProfileHelper::GetInstructorProfile` (the public Instructors page's reader).
- [ ] The three new people can **not** sign in with any password present in the source tree (`VerifyPassword` against each dev password returns false), so D9 is pinned.
- [ ] All five hold the Instructor role, through `PersonHelper::GetRolesForUser`.
- [ ] An Instructor-only person holds `staff_access` and `manage_skills`, through `PersonHelper::GetPermissionsForUser`.
- [ ] Kit and the hotmail account still exist in the base seed with their roles unchanged.

---

# Phase 6 — Classes, approval skills, access rules and the weekly schedule

### 6.1 Skill levels *(seed)*
- [ ] New `Seed::PopulateSkillLevels`: the three skills from the table above, via `TableHelpers::SkillLevels::AddSkillLevel` (code, name, description), with `sort_order` 10/20/30.

### 6.2 Classes *(seed)*
- [ ] `PopulateClasses`: the six classes with the Overview's descriptions, kind `recurring` (the column default). The four old classes are no longer created.
- [ ] Copy is normalised only for spacing ("Beginner Knotty  Yoga" in the Overview has a double space). Otherwise verbatim; the studio can edit it in Manage.
- [ ] Class-name constants in `seed_app_data.h`. The slot, access and photo tables all refer to these.

### 6.3 Access rules *(seed)*
- [ ] New `Seed::PopulateClassRequirements`, data-driven from the Class access table:
  - `ClassRequirementGroups::AddGroup` with label "Membership" or "Instructor approval", and `inline_description` as in the table. Customers see it once Phase 13 lands; until then only admins do (fact 9).
  - `ClassRequirementGroupLiterals::AddPermissionLiteral` / `AddSkillLiteral` for the literals.
- [ ] Permissions and skills are resolved by name and code. A missing one logs and skips that **whole class's** rules rather than writing half of them, because a half-written CNF is *more* permissive than intended.
- [ ] Runs after `PopulatePermissions`, `PopulateSkillLevels` and `PopulateClasses`.

### 6.4 The weekly schedule *(seed)*
- [ ] Rewrite `PopulateClassSchedules` around one `constexpr` slot table (D3) holding the 14 rows above. Each row is `{ className, dayOfWeek, startMinutes, durationMinutes, instructorEmail, predecessor { className, dayOfWeek } }`.
- [ ] One perpetual instance + default schedule per class (the existing `AddPerpetualSchedule` shape), all bound to `class-dropin` (unchanged decision D8 of the previous plan).
- [ ] Predecessors are resolved through a `(className, dayOfWeek) → slotId` map, filled as slots are created.
  - The table is ordered so a predecessor always precedes its follower.
  - If one doesn't, the seeder logs and skips the follower rather than writing a dangling id.
- [ ] Tuesday Beginner Knotty Yoga gets Gifford as `instructor_person_id`. Tiana is added in Phase 11.

### 6.5 Tests
- [ ] Replace `PopulateClassesSeedsTheSixStudioClasses` with the new six, plus "none of the four retired names exists".
- [ ] **Access through `ClassAccessHelper::CheckAccess`** (D4), four seeded people:
  - Silver-only member: Beginner Knotty Yoga **allowed**, all five others **denied**.
  - Gold-only member: Beginner Knotty Yoga, Knotty Yoga and Beginner Partner Acro **allowed**. Handstands, Aerial Rope and Intermediate Partner Acro **denied**, on the approval group only.
  - Gold member + `handstand`: Handstands **allowed**, Aerial Rope still denied.
  - Non-member: everything denied. `Annotate` reports `members_only` for Beginner Knotty Yoga and Knotty Yoga, and `needs_skill` for Handstands (skill gates outrank membership gates).
- [ ] Schedule: for each class, its slots match the slot table by **weekday and start** (never by array index), with the right instructor email.
- [ ] Predecessors: each of the four followers names the right slot, **and** `predecessor.start + predecessor.duration == follower.start`.
- [ ] Session instructors through `ClassCatalogHelper::GetClassDetail(...).upcomingSessions[].instructorNames`. For example, Wednesday Beginner Knotty Yoga reads "Beth Cooper".
- [ ] `NoTwoSlotsCollideInTheSameRoom` over the 14 slots, using half-open intervals so back-to-back classes are legal.
- [ ] Update `PopulateClassSchedulesResolvesRowsByNaturalKeyNotSeedOrder` and `PopulateClassSchedulesSkipsWhenPrerequisitesAreMissing` to the new table, and delete the three per-class slot tests they replace.

---

# Phase 7 — Caleb as the massage provider

### 7.1 Provider assignment *(seed)*
- [ ] `PopulateProviderTypeAssignments`: Caleb only, `massage_therapist`, accepting bookings. Mason's assignment is dropped (OQ-6, confirmed).

### 7.2 Weekly hours *(seed)*
- [ ] **Promote** the test-only `PopulateProviderSchedule` into the base seed as `Seed::PopulateProviderSchedules(…, nowUs)`. It already resolves Caleb and the studio by natural key and generates through `GenerateAvailability`. What changes:
  - The template is named **"Massage hours"**.
  - It is **open-ended** (`effective_to_us` NULL, so future Generate presses keep working).
  - It carries the five ISO entries above instead of Mon–Fri 9–5. The ISO constants are named locally, keeping the existing warning comment and adding `kIsoSunday = 7`.
- [ ] Availability is generated from today's local midnight through `Scheduling::EndOfFollowingMonthUs(nowUs)` in the studio facility. Beyond that it is extended by hand each month (15.4).
- [ ] `PopulateTables` reads `now_us()` once and passes it in, as `PopulateTestData` does today (D2 of the previous plan).

### 7.3 Tests
- [ ] Through the **booking-slot reader** (the one behind the massage picker and `list_available_slots`):
  - on a fixed Monday, Caleb offers slots inside 11:00–15:00 and none outside
  - Wednesday and Saturday offer none
  - a Sunday offers 13:30–16:00, which pins ISO 7
- [ ] Every generated row is Caleb's, `source = template`, and on a Mon/Tue/Thu/Fri/Sun local date.
- [ ] **Caleb's massage hours never overlap a class slot he teaches.** This cross-checks Phase 6's slot table against Phase 7's entries, so a future schedule edit that double-books him fails the gate.
- [ ] Mason holds no provider assignment.
- [ ] The test-data tests for the old 9–5 template move here and change to the new hours.

---

# Phase 8 — About page copy

### 8.1 Seed
- [ ] `PopulateAboutSections`: the three bodies from the About page table, **verbatim**. Titles, ordinals and links unchanged. The "40 years … hootananny" and "climbing wall" placeholder copy is gone.

### 8.2 Tests
- [ ] Through the public About reader (the one behind `/api/about_sections`): three blocks, in ordinal order, with the new bodies, and no block containing "hootananny".

---

# Phase 9 — Attach the photos

Uses Phase 1's files.

### 9.1 Table-driven photo attachment *(seed)*
- [ ] Extend `PopulateSeedPhotos` to one table per target, with one loop each, every lookup by natural key, and every outcome logged (the existing `AttachSeedPhoto` contract):
  - `instructors`, by person email
  - `classes`, by name
  - `products`, by code
  - `facilities`, by code
- [ ] Instructor headshots move off `PopulateInstructors` into this table, so every picture is listed in one place.
- [ ] Rows (files from 1.1):
  - instructors: Mason, Caleb, Beth, Tiana, Gifford
  - classes: all six
  - product `massage` → `Massage.jpg`
  - facility `knotty-yoga-studio` → `Studio.jpg`
- [ ] Mason's and Caleb's **profile** photos (`Mason.jpg`, `Caleb.jpg` on `people`) are unchanged. New instructors upload their own profile pictures.
- [ ] Any photo still missing when this phase lands is left out of the table. It joins later as file + row + test expectation in one change (D10), and stays listed here unchecked until then.

### 9.2 Tests
- [ ] `EverySeedPhotoFileExists`: walks the same tables the seeder walks and asserts each file is in `SeedImageDirectory()`. This catches a renamed or deleted file.
- [ ] Attachment by target type: each listed row has exactly one photo, and **classes not in the table have none** (the D5 wrong-row guard from the previous plan, kept).
- [ ] Massage and studio through their public readers: the catalog product's `has_photo`, and the facility's photo on the location reader.

---

# Phase 10 — Retire the test-only seed data (OQ-12, D14)

### 10.1 Seed
- [ ] Delete `PopulateIntroWorkshopSessions`, `PopulateAerialSeriesProduct`, `PopulateAerialSeriesRun` and `PopulateTestData` from `seed_app_data.{h,cpp}`. `PopulateProviderSchedule` already moved in 7.2. The `intro-workshop` **product** stays in the base catalog: staff schedule real workshop sessions through the UI, and the About copy invites people to the next one.

### 10.2 `knottyyoga_database_helper`
- [ ] Remove the `--recreate_database_test` flag and its help text (`main.cpp`), and the `includeTestData` parameter of `CreateSchemaAndPopulate` (`create_database.{h,cpp}`).
- [ ] Fix the stale reference in `ServerAccess.mock.ts`'s comment, and any doc that names the flag.
- [ ] Update the hand-testing memory ([[feedback_test_instructions_format]]): the flag is gone and Kit is a base-seed member.

### 10.3 Tests
- [ ] Delete the test-data tests, including `OnlyPopulateTestDataAddsTheSampleData`.
- [ ] A base-seed test asserts nothing sample-only exists: no `aerial-series` product, no `Aerial Series` class, and no event sessions at all.

---

# Phase 11 — Co-instructors on a recurring slot (Gifford + Tiana)

Fact 4: a slot cannot carry two instructors today. OQ-7: "we should support multiple instructors."

Scope: an occurrence's **displayed** instructors become `staffing override if present, else slot lead + co-instructors`. These stay keyed on the **lead**:
- pay and specialty cost
- the instructor-load grid
- the exception-notes digest
- favorite-instructor alerts

That is the conservative reading, and each can widen later on its own.

### 11.1 Schema *(db_schema)*
- [ ] New nested table `class_schedule_slot_co_instructors`:
  - `id`
  - `class_schedule_slot_id` (FK)
  - `person_id` (FK → people)
  - `created_us`, `updated_us`
  - UNIQUE(slot, person), and an index on slot
- [ ] The full new-table checklist (CLAUDE.md):
  - `make_app_tables`
  - `CreateTables`
  - the allowed, top-level and nested admin tables
  - column data info, friendly names and display template
  - table permission (`manage_class_schedule`)
  - CMakeLists
- [ ] **Idempotent migration `app/0011_class_schedule_slot_co_instructors`** ([[gotcha_new_table_needs_migration]]), with a migration test.

### 11.2 Table helper *(sql_util/table_helpers)*
- [ ] `TableHelpers::ClassScheduleSlotCoInstructors`: `Add`, `GetForSlot`, `GetForSlots` (batch, for the catalog's N slots) and `Delete`. Tests for each, including the unique constraint.

### 11.3 Business logic *(business_logic/scheduling)*
- [ ] One shared `EffectiveInstructorIds`: the staffing override if present, else lead + co-instructors, lead first. The instructor-profile helper already has a private version. Callers:
  - `ClassCatalogHelper` (class detail sessions)
  - `CalendarHelper`
  - `InstructorProfileHelper`, both "classes I teach" (whose SQL `UNION` gains the co-instructor table) and upcoming sessions
- [ ] `ClassScheduleHelper`: add or remove a co-instructor on a slot, refusing the lead themself and a person who is not an instructor.
- [ ] Tests per caller:
  - a co-taught occurrence lists both names
  - a substitution still **replaces** both
  - Tiana's instructor profile lists Beginner Knotty Yoga
  - the instructor-load grid is unchanged, which pins the scope line above

### 11.4 Endpoints
- [ ] `POST /api/admin/class_schedule_slots/{id}/co_instructors` and `DELETE …/{personId}`, gated on `manage_class_schedule`, thin over 11.3.
- [ ] Endpoint tests: auth (401/403), happy path, duplicate, and lead-as-co-instructor refused.

### 11.5 Seed
- [ ] Tuesday Beginner Knotty Yoga gets Tiana as co-instructor. The slot table gains an optional `coInstructorEmail`.
- [ ] Test through `ClassCatalogHelper`: Tuesday sessions read "Gifford Cheung, Tiana Cheung".

### 11.6 Frontend
- [ ] `ServerAccess` + network + mock + proxy methods for add and remove; `ServerAccess.mock.spec.ts` cases.
- [ ] Slot form dialog (Manage ▸ class schedules): a **Co-instructors** chip picker beneath Instructor, reusing the instructor person picker. Manage Data is not a workflow ([[feedback_manage_data_is_debug_only]]).
- [ ] Wherever a session renders one instructor name, render the list joined with ", ": class detail, calendar event and instructor profile.
- [ ] Specs:
  - the dialog adds and removes a co-instructor
  - class detail renders two names
  - the lead cannot be picked as their own co-instructor

---

# Phase 12 — Forgot password (OQ-2)

Fact 12: this is built in the two honuware repos and then pinned here. **Nothing else in this plan depends on it.** The production rollout (Phase 15) goes ahead with or without it; until it ships, instructors get in with 5.2's temporary password. It follows the same build → push → pin cycle Phase 4 establishes, and touches some of the same files, so it starts after Phase 4's pin bump rather than in parallel.

During development, knottyyoga builds against the local `server_components` checkout via `FETCHCONTENT_SOURCE_DIR_HONUWARE` in the gitignored `CMakeUserPresets.json`, so both repos build together before anything is pushed.

### 12.1 Schema *(server_components: platform/db_schema)*
- [ ] New framework table `password_reset_tokens`:
  - `id`
  - `person_id` (FK → people, **UNIQUE**)
  - `token_hash`
  - `expires_at` (µs)
  - `attempts` (INT default 0)
  - `created_at`

  It has the same shape as `email_verifications` but is a separate table, because a pending verification and a pending reset must be able to coexist. Registered like `email_verifications`: schema only, **never** in the admin CRUD allow-lists, so no generic endpoint can read a token row.
- [ ] Idempotent framework migration **`honuware/0002_password_reset_tokens`**, with a migration test.
- [ ] `login_attempts` kind `password_reset`; `auth_events` kinds `password_reset_requested` and `password_reset_completed`.

### 12.2 Table helper *(server_components: platform/sql_util/table_helpers)*
- [ ] `TableHelpers::PasswordResetTokens`: `ReplaceForPerson` (delete + insert, so a new request kills the old link), `GetForPerson`, `IncrementAttempts`, `DeleteForPerson`. Tests for each.

### 12.3 Business logic *(server_components: platform/business_logic/auth)*
- [ ] `Auth::PasswordResetHelper::RequestReset(transaction, secrets, mail, email)`:
  - Unknown email: does nothing, and sends no mail.
  - Known email:
    - a 32-byte secret, stored only as its hash (the same hashing `email_verifications` uses)
    - expires in 60 minutes (secret `auth_password_reset_ttl_minutes`, default 60)
    - sends the reset mail
- [ ] Reset mail `password_reset_mail.{h,cpp}`, beside `person_verify_mail`:
  - a `FormatString` template, wrapped in `NormalizeCrLf`
  - sender from `::Mail::LoadSenderAddress`
  - tenant branding
  - a link to the SPA's `reset-password` page with `email` and `secret` URL-encoded, built from `website_address_login` exactly as the verification link is
  - Subject: "Reset your {site name} password"
  - Body: who asked, the link, "expires in 60 minutes", and "If you didn't ask for this, ignore this email. Your password has not changed."
- [ ] `Auth::PasswordResetHelper::CompleteReset(transaction, secrets, email, secret, newPassword)`:
  - **Failure.** Missing, wrong or expired all collapse to one generic failure. Each wrong secret counts, and the fifth deletes the token.
  - **Password rules.** The same rules registration enforces.
  - **Success:**
    - `UpdatePassword` with production Argon2 cost
    - clear `must_change_password`
    - clear `failed_login_attempts` / `locked_until`
    - set `email_verified_at` if empty, which also rescues an account whose verification link expired
    - delete the token
    - `LogoutPerson`, which ends every session and remember-me device token
- [ ] `LoginGate::CheckBeforePasswordReset(transaction, secrets, email, ip)`: per-email and per-IP request caps (defaults 5 per hour per email and 20 per hour per IP, as secrets), so the button cannot be used to flood someone's inbox.
- [ ] Tests:
  - an unknown email sends nothing
  - a known one sends exactly one mail whose link carries the secret, and the stored value is not the secret
  - a second request replaces the first, so the old link fails
  - an expired link fails
  - a wrong secret fails and counts, and the fifth wrong secret deletes the token
  - success changes the password (the old fails and the new verifies, via `VerifyPassword`)
  - success clears must-change and the lockout, verifies the email, ends sessions and device tokens, and the link cannot be reused
  - the gate caps per email and per IP

### 12.4 Endpoints *(server_components: platform/endpoints)*
- [ ] `POST /api/forgot_password` `{email}`:
  - **Always `200`** with the same body, whether or not the account exists.
  - Synchronous SQL first, then the mail send is queued on `ThreadPool` ([[feedback_sync_sql_before_threadpool_queue]]), so response time does not reveal whether the account exists either.
  - Gate denial → the generic 429.
- [ ] `POST /api/reset_password` `{email, secret, new_password}`:
  - `200` on success, or one generic `400` "reset link is invalid or expired"
  - per-IP gate as `/api/verify` has
  - attempt recorded async, last
  - `auth_events` row on success
- [ ] Both are added to the auth-endpoint list in the honuware `ErrorInterceptor`, so a 401 there never bounces to the login page.
- [ ] Endpoint tests:
  - an unknown email gets 200 and no mail
  - a known email gets 200 and one mail
  - 429 when gated
  - reset happy path, then the same link a second time fails
  - a bad secret gets the generic 400
  - `ThreadPool::Shutdown()` right after each `handle_full`

### 12.5 Ship the server side
- [ ] Gates on `server_components` (Linux `docker/build_and_test.sh` + Windows) and on knottyyoga against the local checkout; Claude reports both platforms.
- [ ] **Mason** commits and pushes `server_components`.
- [ ] Claude bumps `GIT_TAG` in knottyyoga's top-level `CMakeLists.txt` to that SHA, removes the local `FETCHCONTENT_SOURCE_DIR_HONUWARE` override, and re-runs the knottyyoga gates. CommunityFinder pins its own SHA and is untouched until it bumps.

### 12.6 Frontend library *(honuware-web-components: `@honuware/ui/auth`)*
- [ ] `HonuwareAuthAccess` gains `forgotPassword(email)` and `resetPassword(email, secret, newPassword)`. `AUTH_ROUTES` gains `forgotPasswordPath: '/forgot-password'` and `resetPasswordPath: '/reset-password'`.
- [ ] `LoginComponent`: a **Forgot password?** link beneath the password field.
- [ ] `ForgotPasswordComponent`:
  - title **Reset your password**
  - **Email** field
  - **Send reset link** button
  - afterwards, always: "If an account exists for that email, we've sent a link to reset your password. Check your spam folder too." with **Back to log in**
- [ ] `ResetPasswordComponent`:
  - reads `email` and `secret` from the query string, as `VerifyComponent` does
  - **New Password** and **Confirm New Password**, with show/hide toggles
  - a **Set Password** button
  - success: toast "Your password has been changed. Please log in.", then the Log In page
  - failure: "This reset link is invalid or has expired." with a **Send a new link** button back to the forgot page
- [ ] Specs:
  - the link exists on Log In
  - forgot shows the same message for success and for any error
  - reset rejects mismatched passwords without calling the server
  - reset's success and failure paths
- [ ] Minor version bump in `projects/honuware-ui/package.json`. **Mason** commits, pushes, creates the tag and pushes the tag ([[reference_honuware_ui_release]]).

### 12.7 Knottyyoga wiring *(ui/)*
- [ ] `npm install @honuware/ui@X.Y.0 --save-exact`.
- [ ] `ServerAccess` (+ network + mock + proxy) implements `forgotPassword` / `resetPassword`; `ServerAccess.mock.spec.ts` cases.
- [ ] `auth.routes.ts`: `forgot-password` → `ForgotPasswordComponent`, `reset-password` → `ResetPasswordComponent`.
- [ ] `ng test`, `tsc --noEmit`, `ng build`.

---

# Phase 13 — The class page names the membership; cleanup

### 13.1 Say which membership a class needs (OQ-17, confirmed)
- [ ] **Business logic.** `ClassCatalogHelper::GetClassDetail` adds the class's non-empty membership-group `inline_description`s to the detail, reusing `ClassAccessHelper::GetClassRequirements`, which already resolves them. Test: Beginner Knotty Yoga's detail carries "Silver or Gold membership", and Knotty Yoga's carries "Gold membership".
- [ ] **Endpoint.** The class-detail endpoint serialises it; `scheduling_key_value_table.cpp` already writes `inline_description` for the admin view. Endpoint test.
- [ ] **Frontend.** The class page's "Members only" pricing state reads **"Members only: Silver or Gold membership"**, falling back to today's "Upgrade your membership to attend." when there is no description. Type + mock + spec, covering both the described and the undescribed state.

### 13.2 Dead code
- [ ] Delete `pages/public/services/homepage.service.ts` and its spec (fact 8).

*(Not in scope: `ServerAccess.mock.ts`'s local-mode catalog still has the old classes. It is fixture data for `ng serve -c local` and for dozens of specs; changing it is churn with no user-visible effect.)*

---

# Phase 14 — Verification

### 14.1 Build and test (Claude, both platforms; reported per platform)
- [ ] knottyyoga: Linux gate (`server/docker/build_and_test.sh`) and Windows (`x64-Debug`). Also build `knottyyoga_database_helper` and `knottyyoga_test_helper` explicitly, since the gate builds neither `main.cpp`.
- [ ] Phases 4 and 12 only: the `server_components` gates on both platforms. Phase 12 also: `honuware-web-components` `ng test` + build.
- [ ] Angular (`ui/`): `ng test`, `tsc --noEmit`, `ng build`.

### 14.2 Local run
- [ ] `knottyyoga_database_helper --recreate_database` against the local dev database. This is destructive, so it is announced before running. Confirm no `seed_lookup_missing`, `*_skipped` or `seed_image_failed` lines in the log, other than any photos still outstanding from Phase 1.

### 14.3 Hand-testing steps

Menu labels were checked against the templates on 10/14. `mockHeaderResponse.ts` **is** the production menu: `header.service.ts` always builds from it. Below 1280 px the top nav collapses into the hamburger, so use a wide window.

**Setup.** Run `knottyyoga_database_helper --recreate_database` with these set:
- `HONUWARE_ALLOW_DESTRUCTIVE=1`
- `SCHEDULER_SERVICE_ACCOUNT_PASSWORD`
- `HONUWARE_MAIL_APP_PASSWORD`, so steps 23–26 can send mail

Then start the server and the Angular app. Accounts:
- Admin: `masonbendixen@gmail.com` / `pass`
- Member: **Kit Williams**, `kiteagle@gmail.com` / `kit`

**Signed out: the catalog**
1. Top nav **Our Classes**. The page is headed **All Classes** and shows exactly six cards, each with its photograph:
   - **Beginner Knotty Yoga**
   - **Knotty Yoga**
   - **Handstands**
   - **Aerial Rope**
   - **Beginner Partner Acro**
   - **Intermediate Partner Acro**

   There are no Therapeutic Knotty Yoga, Tumbling, Aerial Fabric or Partner Acrobatics cards.
2. Click **Weekly schedule** (page headed **Our Classes**). Check the 14 rows against the schedule table, including **Saturday 10:00 AM – 12:00 PM Knotty Yoga with Mason Bendixen** and **Wednesday 5:00 PM Beginner Knotty Yoga with Beth Cooper**. These four rows also read **Requires attending: …**:
   - Monday 7:00 PM Handstands → **Knotty Yoga**
   - Wednesday 7:00 PM Aerial Rope → **Knotty Yoga**
   - Thursday 7:00 PM Intermediate Partner Acro → **Beginner Partner Acro**
   - Sunday 12:00 PM Intermediate Partner Acro → **Beginner Partner Acro**
3. Same page, badges:
   - **Beginner Knotty Yoga**, **Knotty Yoga** and **Beginner Partner Acro** rows carry **Requires a membership**.
   - **Handstands**, **Aerial Rope** and **Intermediate Partner Acro** carry **Prerequisite needed**, because a skill gate outranks a membership gate.
4. Click **Prerequisite needed** on the Monday 7:00 PM **Handstands** row. A dialog titled **Skill required** lists **Handstand** with "Able to easily kick up to a wall…". Click **Got it**.
5. **Our Classes** → **Beginner Knotty Yoga** card. Under **Pricing** it reads **Members only: Silver or Gold membership**. Back via **All Classes** → **Handstands**: **Required Skills** shows the **Handstand** chip.
6. **About** → **Instructors**: five cards, each with a headshot and a bio:
   - **Mason Bendixen**
   - **Caleb Ault**
   - **Beth Cooper**
   - **Gifford Cheung**
   - **Tiana Cheung**
7. **About** → **About Knotty Yoga**: three blocks, **Who we are**, **How we teach** and **Come and find us**, carrying the copy in the About page table. Nothing mentions forty years, a hootananny or a climbing wall.
8. **Memberships**: five cards in this order, each with a laurel icon and a **Subscribe** button:
   1. Knotty Yoga Silver Membership **$160.00/mo**
   2. Knotty Yoga Silver Couple's Membership **$240.00/mo**
   3. Knotty Yoga Gold Membership **$200.00/mo**
   4. Knotty Yoga Gold Couple's Membership **$300.00/mo**
   5. Knotty Yoga Gold Family Membership **$400.00/mo**
9. **Services** (page headed **Our Services**):
   - **Massage** is listed with its photo.
   - **Early Bird Spa, Non-Peak Spa, Peak Spa and Late Night Spa are gone.**
   - **About** → **Our Providers** shows **Caleb Ault** only (not Mason).
   - **About** → **Our Location** shows the studio photo.

**Silver member: Silver opens Beginner Knotty Yoga and nothing else**

10. **Sign In** as `kiteagle@gmail.com` / `kit`.
11. **Memberships** → **Knotty Yoga Silver Membership** → **Subscribe**. Leave **Start month** as it is, enter the sandbox card `4111 1111 1111 1111` (any future expiry, CVV `111`), and click **Subscribe & Pay $160.00**.
12. **My Classes** → **Full Schedule**:
    - Beginner Knotty Yoga rows no longer carry a badge.
    - Knotty Yoga and Beginner Partner Acro still read **Requires a membership**.
    - Handstands still reads **Prerequisite needed**.
13. **Our Classes** → **All Classes**:
    - **Beginner Knotty Yoga**: **Pricing** reads **Included with your membership**, "Just show up — staff will check you in."
    - Back → **Knotty Yoga**: **Members only: Gold membership**.

**Instructor approval: AND across groups**

14. Kit's first-name menu → **Sign Out** → **Log Out**. Sign in as `masonbendixen@gmail.com` / `pass`.
15. Top nav **Staff** (page headed **Staff Portal**) → **Person Skills** tile → **Search for a person**: type `Kit` and click **Kit Williams**. Then **Assign new skill** → **Skill**: **Handstand**, **Note (optional)**: `Belly to the wall, 30 seconds` → **Assign**.
16. Sign out, then sign back in as Kit. **Our Classes** → **All Classes** → **Handstands**: **Required Skills** now reads **You meet the prerequisites**, but **Pricing** still reads **Members only: Gold membership**. Approval alone does not admit a Silver member, because Handstands needs Gold **and** approval.

**Admin views of the same data**

17. As Mason: **Admin** → **Manage Products** → **Class Schedules** → **Class**: **Beginner Knotty Yoga**. The **Requirements** card holds one group, **Membership**, with the Silver and Gold permissions. **Description (shown to members)** reads "Silver or Gold membership".
18. Same page, **Class**: **Handstands**.
    - The **Requirements** card holds **Membership** (Gold) and **Instructor approval** (Handstand).
    - Expand **Class Instances** → **Class Schedules** → **Class Schedule Slots**. There is one slot: **Monday**, **7:00 PM**, **60**, **Instructor** Mason Bendixen, **Requires attending** **Knotty Yoga · 6:00 PM–7:00 PM**.
19. **Admin** → **Manage Products** → **Skills**: three skills, **Handstand**, **Aerial Rope** and **Intermediate Partner Acro**.
20. **Admin** → **Manage Products** → **Products**:
    - No product named "Spa" anything, even with **Show Inactive** on.
    - Click **Knotty Yoga Gold Family Membership**. In **Product Information**, **Sort Order** reads **50**.
    - Change it to `5` and click **Save Changes**. **Memberships** now lists Gold Family first.
    - Set it back to `50` and click **Save Changes**.

**Massage: Caleb's hours**

21. **Admin** → **Manage Products** → **Service Providers**: one card, **Caleb Ault**, with **Accepting bookings** on. Click **Templates**: one template, **Massage hours**, with chips for Mon, Tue, Thu, Fri and Sun at the hours in the massage table.
22. **Services** → **Massage** → **Book Now**.
    - In the week strip, click the next **Mon** (use **Next week** if needed). There is one provider card, **Caleb Ault**. The first hour header is **11:00 AM**, and no session starts so late that it would end after 3:00 PM.
    - Click the next **Wed**: no provider card.
    - Click the next **Sun**: Caleb's earliest time is **1:30 PM**.

**Temporary password (Phase 5)**

23. In `knottyyoga_test_helper` (REPL), run `set_password --email=<Beth's email from OQ-1> --password=Welcome-2026`, or `sp` with the same arguments. It reports that Beth must choose a new password at next sign-in.
24. Sign out. **Sign In** as Beth with `Welcome-2026`. The site goes straight to **Change Password**, and clicking **My Classes** comes back to it. Fill in:
    - **Old Password**: `Welcome-2026`
    - **Reenter Old Password**: `Welcome-2026`
    - **New Password**: one of your choosing

    Click **Update**. **My Classes** now opens normally.

**Forgot password (Phase 12)**

25. Sign out. **Sign In** → **Forgot password?**
    - **Email**: `masonbendixen@gmail.com` → **Send reset link**. The page says a link was sent if the account exists.
    - Repeat with `nobody@example.com`. The page shows the identical message.
26. In the masonbendixen@gmail.com inbox, open **Reset your Knotty Yoga password** and click the link. The **Reset your password** page opens.
    - **New Password** and **Confirm New Password**: `Reset-2026` → **Set Password**. The Log In page says the password has changed.
    - Signing in with `pass` now fails; `Reset-2026` works.
    - Clicking the email's link a second time shows **This reset link is invalid or has expired.**

**Co-instructors (Phase 11)**

27. **My Classes** → **Full Schedule** → Tuesday 6:00 PM **Beginner Knotty Yoga** reads **with Gifford Cheung, Tiana Cheung**. **About** → **Instructors** → **Tiana Cheung** lists Beginner Knotty Yoga among her classes.

**Nothing sample-only (Phase 10)**

28. **Upcoming Events** lists nothing. **Our Classes** has no **Aerial Series**. This base seed is exactly what production gets.

**Surviving a database restart (Phase 4)**

29. With the server and the site open, restart the local database container: in `server_components\database_server`, stop the `load_container.cmd` window (or its daemon) and run `load_container.cmd` again. Wait about ten seconds, then reload the home page **without restarting the server**. It renders normally, and **Sign In** with `masonbendixen@gmail.com` / `pass` works. Before Phase 4, every page answered "An unexpected error occurred" and sign-in said "The email or password you entered is incorrect" until the server was restarted.
30. Start `knottyyoga_helper` locally with `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` set and the extra argument `--event_reminders_interval=60`, so one job runs every minute instead of hourly. Within two minutes its window prints `event=job_success job=send_event_reminders`, and never `event=job_http_error … status=403`. Repeat step 29's database restart while it runs: the next minute's job still succeeds.

---

# Phase 15 — Production rollout

Per D11 (OQ-9: wipe). Phase 4's fixes reach production earlier, in a release of their own (4.6); this phase is the content reinstall. Everything here runs on the EC2 host following the Deploying to AWS conventions:
- `--entrypoint` for helpers
- `-v /etc/knottyyoga:/etc/knottyyoga:ro`
- `--env-file /etc/knottyyoga/server.env`

### 15.1 Before
- [ ] Tag and deploy the release that contains the finished phases the normal way (CI tag → deploy script → `--migrate`). This proves the image, and confirms the new `img/` files are staged in it ([[gotcha_release_image_glibc_and_entrypoint]] #4).
- [ ] Manual RDS snapshot, named for this change. It is cheap insurance even though nothing is being kept.
- [ ] Have the `set_secret` block from Deploying to AWS 5.1 step 6 ready, with `website_address_login` = the cloudfront URL until go-live. Check that `mail_app_password` is in it: forgot password and the instructor digest need working mail.

### 15.2 Reinstall
- [ ] Stop `knottyyoga-helper` and `knottyyoga-server`.
- [ ] Run `knottyyoga_database_helper --install_schema --force` with `HONUWARE_ALLOW_DESTRUCTIVE=1` (one-shot, on that command only) and `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` from `server.env`.
- [ ] `--migrate` records the migration ledger; expect `applied=N skipped=0`, as on 9/23.
- [ ] Re-run the `set_secret` block.
- [ ] Start the server, then the helper. Expect `/api/health` → `db:"ok"`, and the helper log shows `login_success`.

### 15.3 After
- [ ] Mason, Caleb and Kit each sign in and change their password: first-name menu → **Profile** → **Change Password** (OQ-10). The seeded ones are in source control. This includes the `masonbendixen@hotmail.com` admin.
- [ ] For each instructor who should sign in, either:
  - `knottyyoga_test_helper --command=set_password` (5.2), with the temporary password handed over in person, or
  - once Phase 12 is live and their real email is seeded, they click **Forgot password?** themselves
- [ ] Paste any bios that arrived late under **Admin** → **Manage Products** → **Instructors** (OQ-15).
- [ ] The public half of the Phase 14.3 walkthrough (steps 1–9) against `https://dv1tgxa9ok30f.cloudfront.net/`.

### 15.4 Every month: extend Caleb's bookable hours (OQ-8)
- [ ] **Admin** → **Manage Products** → **Service Providers** → **Caleb Ault** → **Templates** → **Massage hours** → **Generate Availability**. Then:
  - **From Date**: today
  - **To Date**: the last day of next month
  - **Facility**: Knotty Yoga Studio

  Click **Generate**. It reports "Generated N availability blocks", and "Skipped M days (already existed)" for days already generated, so pressing it early or twice is harmless. If this ever gets forgotten, the massage picker simply shows no times after the last generated day. A scheduled job is the follow-up if it becomes a chore.

---

# Cross-references

- [[Database helper initialization]]: this plan closes two of its follow-ups, both ticked there now:
  - "`PopulateProductVariants` / `PopulateProductPrices` / `PopulateProductBookingWindows` still hardcode ids" (Phase 2) ✅
  - "`late-night-spa` is gated on a permission that does not exist" (Phase 3, by removal) ✅
- [[Photo plan for studio]]: the earlier shot list; Phase 1 supersedes it for these thirteen.
- [[Permission-based class access redesign]]: the requirement-group model Phase 6 seeds.
- [[Deploying to AWS]]: Phase 15's commands, and Phase 4.6's release. Go-live (the cert + alias records) remains separate.

---

# Open questions

*Mason's answers are kept as written. "→ Plan" says what changed. Anything still open has a default the plan is written against.*

### Still open

**OQ-1 — Emails for Beth Cooper, Gifford Cheung and Tiana Cheung.** `people.email` is required and unique, and the instructor digest and (after Phase 12) password resets go to it. **Default:** placeholders `beth.cooper@knottyyoga.local` (likewise for the other two), undeliverable like the scheduler account, so the code can land. Real ones are swapped in before Phase 15.
- Mason- I sent a message to all my instructors asking which email to use.
- → Plan: waiting. Paste the three addresses here when they arrive; they are one constant each in `seed_app_data.h`.

**OQ-11 — Where are the photos?**
- Mason- Taking the photos IS a phase. Please list this as an early phase that the later phases will use. I'll try to take the photos tonight.
- → Plan: Phase 1 is the shoot, with a shot list, framing notes and checkboxes. **Default hand-off:** drop the originals into `C:\Users\mason\Documents\Obsidian\Knotty Yoga\Photos\` named as in 1.1 (any format or size); Claude converts and checks them (1.2). Say if you'd rather use another folder.

**OQ-15 — Instructor bios.**
- Mason- I asked my instructors to write bios. Leave a place for me to put them as they send them to me. Worst case, I manually add them once deployed but hopefully they get them to me before then.
- → Plan: the **Instructor bios** section above has a slot under each name. Whatever is there at Phase 5.5 is seeded verbatim, anything still empty gets its one-line default, and late ones go in through Manage after deploy (15.3).

**OQ-16 — Leftover copy that no longer matches the studio.**
- Mason - Let's do: Who we are / How we teach / Come and find us *(full text moved into the About page table above, verbatim)*.
- → Plan: Phase 8. **Two items remain unanswered, and the default is to leave both unchanged:**
  - The mail sender name **"Knotty Yoga and Spa"** (`app_secret_values.cpp`). With the spa removed, would you like plain "Knotty Yoga"?
	  - Mason- Let's just do Knotty Yoga
  - The location's description on **About ▸ Our Location**, which still mentions "a climbing wall" and "Street parking is free after 6pm". Send replacement text, or say "drop the climbing wall".

**OQ-19 (new) — Knotty Yoga approval as a skill?** The Overview says Knotty Yoga "requires instructor approval to attend. Should be able to do a pull-up and five regular pushups". The OQ-4 answer lists skills only for handstand, aerial_rope and intermediate_partner_acro.
- **Default (written into the plan):** Knotty Yoga needs Gold only, and approval is checked at the door, as the class description says. Gold members can book it without an instructor first approving them in the system.
- **Alternative:** a fourth skill `knotty_yoga` ("Able to do a pull-up and five regular pushups."), gated like Handstands. Every Gold member would then need an instructor's approval before booking the main class.

**OQ-20 (new, from Phase 3) — Five tiers on a three-column grid.** On a desktop-width screen the membership cards sit three to a row. So Silver, Silver Couple's and Gold share the first row, and Gold Couple's and Gold Family sit alone on the second. Gold is split from its own tiers, and the second row is two-thirds full. This shows on the Memberships page, the home page band and the shop.
- **Default:** leave it. The order is right; only the line break falls mid-Gold. You'll see it in the Phase 14 walkthrough.
- **Alternatives:** (a) two to a row on desktop: Silver pair, Gold pair, then Gold Family alone. (b) one row per tier family: Silver row and Gold row. That needs the cards to know which family a tier is in, which the data does not say today.

**OQ-21 (new, from the 10/15 incident) — Who tells you the site is down?** Today nothing does: you find out by visiting it.
- **Default (4.7):** a Route 53 health check plus a CloudWatch alarm that emails you. It stays inside AWS, needs no new account, costs about $1 a month, and checks every 30 seconds.
- **Alternative:** a free outside service (e.g. UptimeRobot) checking the same URL every 5 minutes. No AWS setup, but it is one more account, and an outage is noticed more slowly.
- **Not planned:** a watchdog that restarts the server whenever health fails. After 4.1 the server reconnects by itself, so a restart would only hide the next new kind of failure.

### Resolved

**OQ-2 — Do they need to sign in now, and how do they get a password?**
- Mason- Hrm... the forgot password flow seems like a bad problem. Can we add a phase to correct that with a forgot email button that sends an email with a create new password flow? But for this, I figure I will give them a temporary password that they create here. Do we have must change password on next login support?
- → Plan: **Yes, must-change-password is fully supported** (fact 11). After signing in with a flagged account, the site goes straight to Change Password and keeps returning there until it is changed.
  - Phase 5.1–5.2 adds `set_password`, which sets a temporary password **and** the flag in one step, so you choose the temporary password and hand it over.
  - Phase 12 is the forgot-password flow: a **Forgot password?** link on Log In, an email with a single-use link that expires in an hour, and a page to set a new password.

**OQ-3 — Gold for the non-beginner classes.** Mason- that's correct. → D7.

**OQ-4 — Approval skills.** Mason- Let's make skills for handstand, aerial_rope, intermediate_partner_acro. Say the things in there that I listed for each in the description. → D6 and the skill table: codes `handstand`, `aerial_rope`, `intermediate_partner_acro`, named for the skill, with your criteria as the descriptions. Knotty Yoga: see OQ-19.

**OQ-5 — Instructor role can check people in and grant approvals.** Mason- Yes, this sounds correct. → 5.4.

**OQ-6 — Mason as a massage provider.** Mason- Let's just do Caleb for now. → 7.1.

**OQ-7 — Co-instructors.** Mason- we should support multiple instructors. → Phase 11.

**OQ-8 — Rolling availability.**
- Mason- no no no... let's not make this a new feature. We can just add slots with the template. The generate availability feature is pretty simple. Could also do this as a scheduled job pretty easily if need be. Let's not complicate the webserver more than necessary.
- → Plan: the rolling-availability phase is deleted. Caleb's template is open-ended, the seed generates through the end of next month, and 15.4 is the monthly **Generate Availability** press with exact steps.

**OQ-9 — Getting this onto the live site.** Mason- No, I'm just in preview mode and still using the Square sandbox. Wipe away. → D11, Phase 15. Theme export/import is dropped; the `set_secret` block stays, because it carries the Square sandbox and mail settings.

**OQ-10 — Test accounts in the base seed.** Mason- I'll manually have the three of us change our passwords as we deploy. I'm not that concerned about this. → Kit and `masonbendixen@hotmail.com` stay in the base seed, unchanged (5.3). 15.3 is the password change for Mason (both accounts), Caleb and Kit.

**OQ-12 — Test-only data.** Mason- let's remove both of these. I'm getting ready to deploy and want the website to be as it will actually function. → Phase 10 and D14. With both gone and Caleb's template promoted to real data, nothing test-only remains, so `--recreate_database_test` is removed too.

**OQ-13 — Membership tier order.** Mason- I'd prefer the alternative. → D12, Phase 3: grouped by tier through a new `products.sort_order` column, editable as **Sort Order** in Manage Products.

**OQ-14 — Membership descriptions.** Mason- This sounds fine. → The Memberships table.

**OQ-17 — Tell visitors which membership a class needs.** Mason- let's make this change. → 13.1.

**OQ-18 — Who builds.**
- Mason- I trust you more now! Please update the note that you build and run the tests on windows and linux but report the results to me. When stuff would break, I'd just paste the output to you so I figure I trust you now to remove myself from this loop as your model has improved.
- → Plan: recorded in **Revision notes and working agreement** at the top of this doc and in Claude's standing memory. The Overview's "I will build…" line is left as you wrote it, since the Overview is yours, and is superseded by this answer.
