---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 10/17/2026
Version: 0.10
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

- **v0.10 (10/17/2026)** — **Phase 6 is implemented.** Windows 5259 passed, Linux 5259 passed, with the same test names on both.
  - The six real classes replace the old ones, the three approval skills are seeded, every class has its access rule, and the 14-slot weekly schedule is in, with Beth on Wednesday and Gifford on Tuesday.
  - One change from the plan: a missing permission or skill now **fails closed** (6.3). The plan's "skip the whole class" would have left that class open to everyone.
  - New **OQ-22**: signed in, **Our Classes** lists only the classes a person may already attend. So after Phase 6, non-members and the staff see an empty list. The default fix is new item 13.3.
  - Phase 5 was committed as `5ebb942a` and deliberately not tagged (see Phase 5).
- **v0.9 (10/16/2026, evening Pacific)** — **Phase 5 is implemented.** Windows 5252 passed, Linux 5252 passed, with the same test names on both.
  - Beth, Gifford and Tiana are seeded with random, unrecorded passwords, under OQ-1's placeholder emails. All five instructors hold the Instructor role and have instructor profiles with the default bios.
  - Instructors can now check people in and grant skills (`staff_access` + `manage_skills`).
  - `set_password` sets a temporary password that must be changed at first sign-in.
  - Also confirmed: sandbox.10's hourly jobs ran at 02:30 UTC, all eight succeeded with no 403s, so 4.6 is closed. Only tomorrow's first daily run (billing, about 6:31 pm Pacific on 10/17) is still to check.
- **v0.8 (10/16/2026, evening Pacific)** — **Phase 4 is in production.** Mason deployed `v1.0.0-sandbox.10` to the EC2 at 01:30 UTC on 10/17.
  - Health names the real tag. The helper signs in with both cookies (`cookies=2`), and the database sockets carry keepalives (4.6).
  - 4.8's re-test **passed**. Removing the `critical` drop-in restarted the host's networking, and the DHCP lease flapped as on 10/15. Sixteen minutes later every database page loaded first time, with no reconnect and no restart. The drop-in is gone, and the `--network host` fallback is not needed.
- **v0.7 (10/16/2026)** — `v1.0.0-sandbox.9` is live (deployed 17:58 UTC), but it was tagged on `71d78efd`, the **Phase 3** commit. The Phase 4 pin commits never reached GitLab: knottyyoga `7f38fbe` and CommunityFinder `e7f9b6f` are both still "ahead 1" locally. So production now has Phases 2–3 but **not** the reconnect, the honest health check or the scheduler fix. Phase 4 ships as **`v1.0.0-sandbox.10`** (4.6).
  - Later the same day, at Mason's request, the stale `HONUWARE_VERSION=v1.0.3` line was removed, so health now says `v1.0.0-sandbox.9`.
  - 4.8's netplan `critical` drop-in was applied too. Its test (a `systemd-networkd` restart) **still killed an idle database connection** and hung the site from 18:52 to 19:03 UTC, until Claude restarted the server. `critical` is not the fix; Phase 4's keepalives and reconnect are. 4.8 now waits on sandbox.10 for a re-test.
- **v0.6 (10/16/2026)** — Mason pushed `server_components` as `cddb9cb`. knottyyoga and CommunityFinder now pin it, and both are green on both platforms: knottyyoga 5244/5244, CommunityFinder 1840/1840 (4.5). Next are Mason's commits of the two pins plus `web_app.cpp`. Then the release (4.6) is tagged **`v1.0.0-sandbox.9`**, not v1.0.4: production runs `v1.0.0-sandbox.8`, and the "v1.0.3" in its health answer is a stale `server.env` override.
- **v0.5 (10/15/2026)** — Phase 4.1–4.5's gates are implemented in `server_components` and green on both platforms; each item records what changed in building it. The work also turned up a **second scheduler bug**: the helper's re-login after a 401 sent a garbled email (4.4), so after its first session expired it could not have recovered even without the CSRF refusals. Next is Mason's commit and push of `server_components`, then the pin bump (4.5) and the v1.0.4 release (4.6).
- **v0.4 (10/15/2026)** — Mason's answers to OQ-16, 19, 20 and 21. The changes:
  - The email sender becomes "Knotty Yoga".
  - About ▸ Our Location gets Mason's text with driving directions. Phase 8 now carries both.
  - OQ-19 and OQ-20 keep their defaults.
  - Phase 4.7 is built on the existing UptimeRobot monitor instead of a new Route 53 check.
  - Phase 4.0 records the incident's root cause: an unattended-upgrade restarted `systemd-networkd`.
  - New 4.8 stops the host from doing that again.
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

### About ▸ Our Location (OQ-16: Mason's text)

The studio facility's description (`facilities.description`, shown on **About ▸ Our Location**) replaces the placeholder that mentioned a climbing wall and free street parking after 6pm:

> Our home gym: a full acrobatics floor with crash mats, aerial rigging, 24 fabrics, 9 ropes, plus a private room for massage and bodywork. Parking is abundant. Conveniently off the 520. Heading Eastbound on 520 take the first 148th exit and then the right sub exit labelled 152. Go "through" the first roundabout and then go "left" at the second. We're in the last building (14) before the parking lot bends to the right. Space 2545 / Knotty Yoga.

One word differs from the answer as typed: **"We in the last building"** became **"We're in the last building"**, which reads as a dropped word. Everything else is verbatim, including the quoted "through" and "left". Say if it should go back exactly as typed.

### Email sender name (OQ-16)

Every email the site sends comes from **"Knotty Yoga"**, not "Knotty Yoga and Spa" (`kMailSenderNameValue`, a config-secret default). The sending address itself does not change.

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

### 3.12 Membership card layout (Mason's review of the five tiers, 10/15)
The five tiers made the cards' uneven heights visible: a name or description that wraps onto another line pushed that card's price and **Subscribe** button below its neighbours'.
- [x] **Prices and Subscribe buttons line up across a row.** `MembershipTierCardsComponent`'s content fills the card, and the price takes the spare space above itself (`margin-top: auto`), so the price and the button under it sit at the foot of every card. The grid already gives every card in a row the same height.
- [x] **Less white space above the crest and between it and the name.**
  - Most of it was inside the crest's box. The laurels are 222×160, and a 64 px *square* box drew them 46 px tall with about 9 px of empty box above and below. The box is now 64×48, the laurels' own shape.
  - The gap under the crest is 8 px (was 12).
  - The card's top padding is honestly 16 px now. The stylesheet said 24, but Material's `.mat-mdc-card-content:first-child` rule (16 px) always won, which a mutation run showed.
- [x] Specs, measured in a 1280 px-wide viewport (`inViewportOf`), because the three columns are a media query:
  - **Alignment:** three tiers, one with a name and description long enough to wrap. All price tops are equal, and all button tops are equal. Guards check that the three cards share one row and that the descriptions really differ in height. Mutation-checked: without the fix, the long tier's price sits at 337 px against 145 px for the others.
  - **Spacing:** the crest box is 48 px tall, at most 16 px below the card's inner edge, and at most 8 px above the name. Mutation-checked: the old values fail on the box (64) and the gap (12).
- [x] Angular: **3590 of 3590** specs pass, `ng lint` passes, and `ng build` succeeds.
- Found along the way, not fixed: `page-not-found.component.spec.ts:37` asserts inside a callback that runs after the test ends. It logs "Expected a spy, but got Function" on every run and tests nothing.

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
  - *Why `csrft` in particular (found while building 4.4):* Crow keeps response headers in an `unordered_multimap`. On Linux (libstdc++) a repeated header goes out **newest first**, so login's `session_token`-then-`csrft` reaches the wire as `csrft`, `session_token`, and last-wins kept `session_token`. A Windows server sends them in the other order, so the same helper would have lost the session cookie instead. Either way the job is refused.

All the code is in `server_components`; knottyyoga only moves its pin. During development knottyyoga builds against the local checkout (`FETCHCONTENT_SOURCE_DIR_HONUWARE`), as Phase 12 does. This phase ships to production **on its own release (4.6), ahead of everything after it**: it is a fix to what is live today.

### 4.0 Now, by hand
- [x] `sudo systemctl restart knottyyoga-server` on the EC2, then `curl -sS http://localhost/api/health` and a page load through the CloudFront URL. This is the only cure until 4.6 ships.
  - Mason restarted it at **20:08:20 UTC** on 10/15. Checked at about 21:50 UTC: health ok and no "Lost connection" since. The helper has still had **26 refusals and no successful jobs** since the restart, as expected until 4.6.
- [x] **Why did the connection drop?** Found 10/15:
  - **The connection did not die at 19:08.** That is when Mason's visit first *used* it. The last request that used the database successfully was **10/14 22:36 UTC**; the helper's refused jobs never reach the database, so the connection sat idle for 20½ hours.
  - **RDS events for the window:** only the nightly backup (09:06–09:09 UTC), which does not drop connections.
  - **The RDS Postgres log** (read in CloudShell): at **06:39:22 and 06:39:55 UTC** it logged `could not receive data from client: Connection timed out` for **two** connections from the EC2 (`172.31.2.31`, user `knottyyoga`). Postgres's keepalive probes went unanswered, so it hung up.
  - **The EC2's journal:** at **06:35:15** Ubuntu's `unattended-upgrade` ran. At **06:35:30** its new `systemd` package restarted **`systemd-networkd`**, which re-enumerated `ens5`, `docker0` and the server container's `veth`, and re-acquired the DHCP address.
  - **Why that breaks the database connections.** The server container's outbound connections go through Docker's MASQUERADE NAT, and the kernel drops MASQUERADE's connection-tracking entries when the host address goes away. RDS's keepalives then had nowhere to land, and RDS hung up four minutes later. The server process was never told and kept the dead socket (4.1).
  - **Not fully explained:** why the global connection behind `/api/health` survived. Probably something used it inside the 06:35–06:39 window, which re-created the NAT entry before RDS gave up. It does not change the conclusion or the fix.
  - **It will recur.** Unattended-upgrades runs daily; any update to `systemd` or `netplan` does this again. 4.1 makes the server shrug it off. 4.8 set out to stop the host from causing it, but `critical` could not. Its 10/17 re-test showed that 4.1's keepalives are enough on their own.

### 4.1 Reconnect a dropped connection *(server_components: data/sql_util/database_access)*
- [x] `DatabaseHelperProd` keeps the `DatabaseHelperInit` it was built from and gains `ResetConnection()`, which opens a fresh `pqxx::connection`. Exposed through `DatabaseHelper`.
  - *As built:* the new connection is opened **before** the swap, so a failed reconnect leaves the old one in place. The old connection is kept alive one generation more (`retiredConnection_`), because a caller may still hold a reference to it from `GetConnection()`. Helpers that cannot reconnect (the test harness's, `MakeNoDatabaseHelper`) throw `std::logic_error` rather than pretend.
  - *As built:* the reconnect reasons are `connection_closed` (found closed before BEGIN) and `begin_failed` (BEGIN threw and the connection is no longer open). An ordinary SQL error with a live connection does **not** reconnect.
- [x] `ProductionTransactionProvider::RunInTransaction`, still under its mutex:
  - **Before BEGIN:** if the connection is closed, reconnect.
  - **If BEGIN itself throws `pqxx::broken_connection`:** reconnect once and begin again. This is safe because none of the caller's work has run yet.
  - **If the connection breaks mid-transaction:** reset it so the *next* transaction reconnects, and rethrow. The caller's work is **never re-run automatically**, because it may already have done something outside the database (a Square charge, an email).
- [x] TCP keepalives in the connection string (`keepalives=1 keepalives_idle=60 keepalives_interval=10 keepalives_count=3`), so a silently dead socket is noticed in about 90 seconds rather than by the next customer.
  - *As built:* also `tcp_user_timeout=30000`. Keepalives only probe an **idle** socket; a query sent into a dead NAT entry would otherwise sit in the kernel's retransmit queue for about 15 minutes. With it, the send fails after 30 seconds. The string is `kConnectionLivenessOptions` in `database_helper_init.h`.
- [x] One log line per reconnect: `[database] event=reconnected database=<name> reason=<…>`, so the journal says when it happened.
- [x] `PooledTransactionProvider` needs nothing of its own: each slot is a `ProductionTransactionProvider`.
- [x] Tests, against the test database, with a production provider pointed at it. A second connection kills the provider's backend with `pg_terminate_backend`, which is exactly what a dropped connection looks like to libpq:
  - after the kill, the next transaction **succeeds**
  - a transaction running when its backend dies throws, is **not** re-run (a counter in the callback proves it ran once), and the transaction after it succeeds
  - a pool of 1 recovers the same way
  - the keepalive parameters appear in `GetConnectionString()`
  - *As built:* also an ordinary SQL error does **not** reconnect, a healthy connection is kept, and the helpers that cannot reconnect refuse to. They are in `production_transaction_provider_test.cpp` and `database_helper_init_test.cpp`.

### 4.2 A health check that tells the truth *(server_components: platform/endpoints/health)*
- [x] `/api/health` probes the connections requests actually use:
  - the global provider
  - **every tenant pool** in the `TenantResourceRegistry`
  - the request's resolved tenant, if any (resolving creates its pool)

  It reports `db: ok` only if all of them pass. With 4.1 in place a probe on a dropped connection reconnects and passes, so health fails only when the database really cannot be reached. That is what it should mean.
- [x] On failure: still 503, and the body names what failed, e.g. `{"status":"fail","db":"fail","failed":["knottyyoga"]}`. The 200 body is unchanged, so the deploy script's check keeps working.
  - *As built:* the global connection is named `global`; each tenant pool by its database name. A failed probe also writes `[health] event=probe_failed` to the journal. With no probe targets at all the answer is 503, not a vacuous 200. `TenantResourceRegistry` gained `GetAll()` for this.
- [x] Tests:
  - a registry holding a tenant whose provider throws → 503, naming it
  - everything healthy → 200 and today's body
  - no registry at all (tenancy inert) → behaves as today
  - *As built:* plus the HTTP endpoint failing when the site's pool is dead though the global connection works, which is the 10/15 shape exactly (`health_test.cpp`).

### 4.3 Errors that say what they are *(server_components: foundation/util + platform/endpoints)*
- [x] `ErrorResponse::ServiceUnavailable(message)` → **503**. Today there is no 503 helper. Test.
- [x] `/api/login`:
  - A database failure (`pqxx::broken_connection`, `pqxx::sql_error`) is logged with context (`event=internal_error_context`) and answered **503** "Sign-in is temporarily unavailable. Please try again in a minute."
  - A wrong password still gets the same generic 401 as today. Nothing is revealed to a guesser.
- [x] `EndpointAuthHelper::Initialize`: the swallowed session-restore error is logged at ERROR, rate-limited to once a minute, so "everyone looks signed out" leaves a trace.
  - *As built:* the line is `[endpoint_auth] event=request_setup_failed path=… error="…" suppressed_since_last=N`. The once-a-minute limit is a small foundation class, `Util::LogThrottle` (with its own tests), and the `WebApp` owns the instance, so each server (and each test's `WebApp`) starts fresh instead of sharing a static. It also catches an empty `mail_app_password`, which fails in the same place.
  - *As built:* `/api/login` answers 503 for any `pqxx::failure` (broken connection, SQL error, …). Every other exception keeps the generic 401.
- [x] Tests: login over a provider that throws `broken_connection` → 503 and a logged context line; a wrong password → 401 as before; a session-restore failure logs once per minute, not per request.

### 4.4 The scheduled jobs get past the CSRF guard *(server_components: foundation/util/http + scheduler)*
- [x] `Http::HttpResponse` gains `std::vector<std::string> setCookies`, filled by the curl header callback with **every** `Set-Cookie`. The `headers` map keeps today's behaviour for everything else. Tests: two `Set-Cookie` lines both survive; the test HttpClient can return several.
- [x] `ApiClient::Login` stores every cookie. `CallEndpoint` sends them all, and on any non-GET call adds `X-CSRF-Token: <csrft>`, the double-submit the guard checks. The re-login-on-401 path refreshes both.
- [x] Not exempting the helper from CSRF instead: an exemption would mean trusting something the request says about itself. Doing the double-submit keeps one rule for every caller.
- [x] Tests:
  - `ApiClient` with a two-cookie login sends both cookies and the header
  - GET sends no header
  - re-login replaces both
  - **One end-to-end test:** `ApiClient` against the real web app's middleware (CsrfGuard included), through a test HttpClient that hands each request to `handle_full`. This is the test that would have caught the 10/8 regression; nothing exercised the helper against the guard.
    - *As built, and why it changed:* Crow's `handle_full` **skips the global middleware**, so through it neither `CookieParser` (which turns cookies into `Set-Cookie` headers) nor `CsrfGuard` ever runs. That is the very seam the bug lived in. So the test (`platform/endpoints/api_client_end_to_end_test.cpp`) starts the real `WebApp` on a loopback port and talks to it with the real libcurl client. The steps:
      - a bare POST is refused 403, which proves the guard is live
      - the helper logs in and holds both cookies
      - a mismatched token is refused
      - the helper's own POST to `/api/get_user_info` returns 200 with the user's email
    - It is the second live-server test in honuware; the first is `http_client_test.cpp`, whose comment asks each one to make its own case. This one's case is in its header.
  - *As built:* the live `http_client_test` now also sends two `Set-Cookie` headers and checks that both arrive. The per-line header parser is a public `Http::ParseResponseHeaderLine` with its own unit tests. The `headers` map keeps last-wins for every header, `Set-Cookie` included.
  - *As built:* a job's own `X-CSRF-Token` overrides the engine's, as every other per-job header does.
- [x] **Found while mutation-checking the end-to-end test: the helper's re-login was broken too.**
  - `CallEndpoint` re-logs in on a 401 by calling `Login(email_, password_)`, passing views into the client's own members. `Login` assigned the members first, which freed the buffers those views pointed into, and then built the request from freed memory.
  - Any address longer than 15 characters is affected, and `scheduler@knottyyoga.local` is 26. So once the helper's first session expired, every re-login sent a garbled email and failed, and every job after that would have failed. The 403s hid it.
  - Fixed: `Login` copies its arguments before touching the members.
  - Tests:
    - `ApiClientTest.ReloginSendsTheOriginalCredentials`, with the real service-account address.
    - A fifth step in the end-to-end test: log out (which leaves the helper holding a dead session), then the helper's next call gets 401, logs in again on its own, and its retry returns 200 with a new session cookie.
    - Both fail against the old code.
- [x] **Mutation checks (Windows):** each half of the fix was reverted on its own, and the end-to-end test failed every time:
  - every `Set-Cookie` → 401
  - the `X-CSRF-Token` header → 403
  - the re-login copy → 401, plus the unit test showing freed `0xDD` bytes sent as the email

### 4.5 Ship the server side
- [x] Gates on `server_components` (Linux + Windows) and on knottyyoga against the local checkout; Claude reports both platforms.
  - **server_components:** Windows 1819 passed, Linux 1819 passed.
  - **knottyyoga against the local checkout:** Windows 5244 passed, Linux 5244 passed, and the Linux database helper and server binaries build. Phase 3 had 5204. The last change after this run was test-only, a log assertion in honuware's `login_test`, and it passed in both honuware runs above.
  - **One knottyyoga change rides along:** `src/endpoints/web_app.cpp`'s link anchor named `&Endpoints::GetHealth`, which became ambiguous when 4.2 overloaded it. It now names the single-provider overload with a cast, which compiles against both the pinned and the new honuware. Honuware's own anchor list moved to `CollectHealthProbeTargets`.
  - **The Windows build tree is configured against the local checkout** (`x64-Debug-local-honuware` in `CMakeUserPresets.json`, which persists `FETCHCONTENT_SOURCE_DIR_HONUWARE` in the cache). That is right until the pin bump; the next item clears it.
- [x] **Mason** commits and pushes `server_components`. Pushed 10/15 as `cddb9cb` ("Phase 4 — Production reliability…"); its CI passed.
- [x] Claude bumps `GIT_TAG` in knottyyoga's `CMakeLists.txt`, removes the local override, and re-runs the knottyyoga gates against the pinned SHA.
  - `server/knottyyoga_server/CMakeLists.txt` now pins `cddb9cb56c317e4d6e8f769f1ea18b77c3dc48e3`.
  - **Windows 5244 passed, Linux 5244 passed.** The two runs executed the same 5244 test names, including the new suites. Both Linux binaries build (`knottyyoga_database_helper`, `knottyyoga_the_server`), and the Linux log says `honuware : pinned SHA`. Both build trees' `_deps/honuware-src` hold `cddb9cb`.
  - The Windows build tree is back on the pin: `FETCHCONTENT_SOURCE_DIR_HONUWARE` is empty in its cache. `CMakeUserPresets.json` and its `x64-Debug-local-honuware` preset are left in place for next time. In Visual Studio, pick the plain **x64-Debug** configuration, or the next reconfigure puts the override back. The local checkout is identical to `cddb9cb` today, so either one builds the same code.
- [x] CommunityFinder moves to the same pin (Mason's request, 10/15).
  - `server/communityfinder_server/CMakeLists.txt` now pins `cddb9cb`. No source changes were needed: it has no `GetHealth` link anchor.
  - **Windows 1840 passed, Linux 1840 passed**, with the same test names on both. Both Linux binaries build (`communityfinder_database_helper`, `communityfinder_server`); on Windows all four executables build.
  - What it gains: the reconnect (4.1), the truthful health check (4.2) and login's 503 (4.3). It has **no scheduler helper** (no `ApiClient` user, no helper executable), so 4.4's CSRF bug never affected it.
  - **Windows needed three runs.** Visual Studio, open on `communityfinder_server`, reconfigured the same build tree after the `CMakeLists.txt` edit:
    - The first build failed: FetchContent reported "Failed to unstash changes".
    - The second linked a test program from the old file list. It passed 1800 tests, 40 short of Linux, and was missing exactly the new Phase 4 suites.
    - The third, after VS had finished, ran all 1840.
  - From now on Claude compares the Windows and Linux test names, not just the counts.
- [x] **Mason** commits:
  - **knottyyoga:** `server/knottyyoga_server/CMakeLists.txt` (the pin) and `server/knottyyoga_server/src/endpoints/web_app.cpp` (the anchor). Committed 10/16 as `7f38fbe`.
  - **CommunityFinder:** `server/communityfinder_server/CMakeLists.txt` (the pin). Committed 10/16 as `e7f9b6f`.
- [x] **Mason** pushes both commits. **knottyyoga: pushed 10/16 19:25 UTC** (GitLab master = `7f38fbe`; master pipeline 2956756668 started). **CommunityFinder: pushed the evening of 10/16 Pacific**; GitLab master = `e7f9b6f`, and the local repo is level with `origin/master`. It is not part of this release. Earlier, as of 10/16 18:15 UTC, neither had reached GitLab: both repos' `git status` says `ahead 1`, and GitLab's knottyyoga master is still `71d78efd`. The commits were probably made with **Commit All** rather than **Commit All and Push**. In Visual Studio: **Git** → **Push**, once in each repo.

### 4.6 Release to production, ahead of the rest of the plan
- [x] **`v1.0.0-sandbox.9`: tagged and deployed by Mason on 10/16, but it holds Phase 3, not Phase 4.**
  - The tag points at `71d78efd` ("Phase 3 — Products, memberships and their display order"). GitLab's master was still there when the tag was made, because the Phase 4 commit had not been pushed (4.5). That commit pins honuware `e662f4d`, the code from before Phase 4.
  - Checked on the EC2 at 18:12 UTC:
    - both containers run `knottyyoga:v1.0.0-sandbox.9`, started 17:58
    - `/api/health` answers 200 `db: ok`
    - the site loads through CloudFront
  - What it shipped is Phases 2–3: the `app/0010_products_sort_order` migration, ordering in Manage Products, and 3.12's membership-card layout. The seed changes from Phases 2–3 still do not reach production (fact 6).
  - **How to tell it is not Phase 4:** the helper logs `event=login_success … cookies=1`. Phase 4's `ApiClient` keeps both login cookies and logs `cookies=2`. So sandbox.9's scheduled jobs are still refused with 403, and a dropped database connection still needs a manual restart (4.0).
  - Also seen: at deploy time the helper started before the server was listening. It logged `Scheduler initialization failed: CURL error` twice, then systemd's restart 10 seconds later logged in fine. This is harmless.
- [x] **`v1.0.0-sandbox.10` is the Phase 4 release.** Tagged, deployed 10/17 01:30 UTC and verified (below).
  - Once `7f38fbe` is on GitLab and master's pipeline for it is green (all six jobs), Claude tags it, per Mason's 10/16 request to tag the release after CI passes. If Mason tags it first, Claude sees the tag and skips.
  - **The tag must be on `7f38fbe`.** Don't put it on whatever master happens to be. If tagging by hand, confirm that master's newest commit on GitLab is "Phase 4 — Production reliability…" before **New tag**.
  - Naming: Deploying to AWS §7.1's convention is `v1.0.0-sandbox.N+1` until the switch to live Square, which becomes `v1.0.0`. "v1.0.4" was Claude's mistake. `/api/health` reports `v1.0.3` because `/etc/knottyyoga/server.env` still carries the hand-set `HONUWARE_VERSION=v1.0.3` from the 9/22 manual deploy, and an `--env-file` value overrides the image's own version. Claude read that number as the running build.
  - The tag pipeline runs `verify:green-commit`, `package:server`, `package:ui` and `release:gitlab`. `verify:green-commit` refuses a commit that has no green master pipeline.
  - **Nothing deploys until Mason presses ▶ `deploy-manual:ec2`.** `7f38fbe` changes only the server, so ▶ `deploy-manual:ui` is optional; it would redeploy sandbox.9's UI unchanged.
  - No migrations: Phase 4 adds none.
  - **Master pipeline 2956756668 (`7f38fbe`), 10/16:** `build:server`, `test:deploy-scripts`, `lint:frontend` and `build:frontend` passed. `test:frontend` failed after 174 s.
    - The commit touches no frontend code, so Claude reproduced the job in the same `node:22-bookworm` image with the same Chromium 155.
    - Run 1 froze the same way: "Chrome Headless … Disconnected … (ping timeout)" after 2781 of 3590 specs. It is not an assertion failure.
    - Run 2 passed 3590/3590, and Windows passed 3590/3590.
    - So it is a flaky browser freeze, not a broken test. Mason pressed **Retry**: `test:frontend` passed in 123 s and `test:backend` in 1126 s, so the pipeline is green.
    - **The hunt for the freezing spec found nothing.** Six more Linux runs, each logging every spec as it started and each in a different random order (seeds 18864, 61172, 99216, 34959, 45577, 14389), all passed 3590/3590. That makes 2 freezes in 9 Linux runs.
    - Both freezes happened while the C++ `test:backend` job was busy on the same Docker VM. The likely cause is CPU starvation, which stalls Chromium for the 7 seconds that Karma's ping and reconnect timeouts allow. It does not show up as a test failing.
    - **Follow-up, only if it recurs:** give the Karma run some slack. Angular's built-in Karma config can't be tuned, so this means a `karmaConfig` in `angular.json` with `browserDisconnectTolerance: 1` (Karma re-runs after a disconnect, but never after a failed assertion). Not done now: the commit is already tagged.
- [x] **Claude tagged `v1.0.0-sandbox.10`** on `7f38fbe` at 19:51 UTC, once master's pipeline 2956756668 was green. It is an annotated tag; GitLab confirms `refs/tags/v1.0.0-sandbox.10^{}` = `7f38fbe662dd…`.
  - Tag pipeline 2956830295 passed: `verify:green-commit` 4 s, `test:deploy-scripts` 4 s, `package:ui` 34 s, `package:server` 771 s, `release:gitlab` 6 s.
- [x] **Mason** presses ▶ `deploy-manual:ec2` on that pipeline. ▶ `deploy-manual:ui` is optional. **Deployed 10/17 01:30:38 UTC** (the evening of 10/16 Pacific); EC2 only.
- [x] **Optional, Mason's call (a production config change):** delete the `HONUWARE_VERSION=v1.0.3` line from `/etc/knottyyoga/server.env`, so `/api/health` names the tag that is actually running. Done 10/16 at Mason's request.
  - The file shrank by exactly that one line (24 bytes) and kept its `root:root 600` permissions.
  - `knottyyoga-server` was restarted at 18:36:22 UTC to pick up the change. Health now answers `{"db":"ok","status":"ok","version":"v1.0.0-sandbox.9"}`, the image's own tag.
  - Deploying to AWS 5.1 step 7's "add `HONUWARE_VERSION` to `server.env`" note now says not to re-add it.
- [x] On the EC2 after sandbox.10:
  - `sudo docker ps` shows both containers on `knottyyoga:v1.0.0-sandbox.10`.
  - `curl -sS http://localhost/api/health` → `{"db":"ok",…}`.
  - `sudo journalctl -u knottyyoga-helper -n 50 --no-pager` shows `login_success … cookies=2`, then `event=job_success` lines within the hour, and no `status=403`.
  - **Checked by Claude, 10/17 01:35 UTC:**
    - Both containers run `knottyyoga:v1.0.0-sandbox.10`, started 01:30.
    - `/api/health` answers `{"db":"ok","status":"ok","version":"v1.0.0-sandbox.10"}`. It names the real tag now that the `server.env` override is gone.
    - The helper logged `event=login_success … status=200 cookies=2` at 01:30:50, then `event=initialized jobs=19`. **The Phase 4 scheduler fix is live.**
    - As with sandbox.9, the helper started before the server was listening. It failed twice ("Could not connect to server", then "Failure when receiving data from the peer"), and systemd's restart 10 seconds later logged in. This is harmless.
    - The server's three database sockets show `timer:(keepalive,…)`, so 4.1's keepalives are on.
    - Through CloudFront, `/`, `/api/health`, `/api/classes`, `/api/catalog_products` and `/api/class_tags` all answer 200.
    - `[security_posture] prod=false` has been in every startup log since 10/15. That is deliberate: Deploying to AWS step 9 holds `production_mode_on` back until knottyyoga.com is live, because prod mode pins cookies and CORS to that domain.
  - **The scheduled jobs run again, 10/17 02:30:50 UTC.** These are the first successful jobs since 10/8. All eight hourly jobs logged `event=job_success … status=200` in 82–134 ms:
    - `send_event_reminders`
    - `process_waitlist_refunds`
    - `notify_admin_alerts`
    - `send_instructor_exception_digests`
    - `send_weekly_digests`
    - `finalize_class_attendance`
    - `send_signup_open_reminders`
    - `expire_stale_waitlist`

    No `status=403`, no `job_exception` and no `job_http_error` lines appeared. The server logged no `[database]` or `[health]` lines.
- [ ] **The missed jobs** (10/8 until sandbox.10). On the Square sandbox nothing real was lost. Billing and grace-period jobs pick up whatever is due on their next daily run. Reminders for classes that already happened are simply never sent.
  - **When they first run.** The helper starts each job's timer when it starts (`JobScheduler::ScheduleNext` waits one full interval before the first run), and it started at 01:30:50 UTC on 10/17. So:
    - The hourly jobs (reminders, waitlist refunds, digests, attendance) first run at about **02:31 UTC on 10/17**.
    - The daily jobs (billing, grace period, expiring entitlements and vouchers, the cleanups) first run at about **01:31 UTC on 10/18**, which is 6:31 pm Pacific on 10/17.
    - The card-expiry job runs every 30 days.
  - **To close this item:** `sudo journalctl -u knottyyoga-helper --since "2026-10-18 01:30" --no-pager -o cat | grep -E "job=(run_billing|expire_grace_periods) "` shows `event=job_success` for both. Any redeploy or helper restart moves these times.

### 4.7 Watch it from outside (OQ-21, answered)
Nothing new to build. The **UptimeRobot** monitor set up in Deploying to AWS (9/25) already checks `https://dv1tgxa9ok30f.cloudfront.net/api/health` every 5 minutes and emails Mason after 2 failures in a row. It counts any non-2xx answer as down. It stayed green through 10/15 only because health itself was wrong, and 4.2 makes health answer **503** when a connection requests use is dead, which the monitor already treats as an outage.
- [x] After 4.6 ships, confirm the monitor still shows **Up**: the 200 body is unchanged, so it should.
  - **Confirmed from the server's side, 10/17 02:10 UTC.** All 8 of UptimeRobot's checks since the sandbox.10 deploy (`HEAD /api/health`, every 5 minutes through CloudFront) got 200, including the ones during 4.8's network test. The server answered nothing with a 5xx, and logged no `[database]` or `[health]` lines. The monitor alerts only after 2 failures in a row, so it shows Up.
- [x] Correct the claim in Deploying to AWS's uptime section that `/api/health` "returns `db:"ok"` only if the database answered". Until 4.2 it only proved the *global* connection answered. A correction note now sits under that paragraph (10/15).
- **Not planned:** switching the monitor to a keyword match on `"db":"ok"`. Once 4.2 lands, a failing check already returns 503, so the keyword would add nothing.

### 4.8 Stop the host's network restarts from cutting live connections *(EC2, Deploying to AWS)*
Defence in depth for 4.0's cause. 4.1 alone already turns it into one reconnect and a log line.

**Result, 10/17 on sandbox.10: passed. The keepalives are the fix, and no host change is needed.** See the re-test item at the end of this section.

**Result, 10/16: `critical` did not stop it, and the test took the site down for 11 minutes.**
- Mason asked for 4.8 on 10/16. Production was still on sandbox.9 (Phase 3), so nothing could reconnect.
- The drop-in went on at 18:36:09. The server was restarted at 18:36:22 (for the version-line change), opened fresh database connections at 18:36:33, and `systemd-networkd` was restarted at 18:36:34. The three sockets showed the same ports before and after.
- After a 15-minute wait, the first request to use the site's connection (18:52:02, `/api/catalog_products`) **hung**:
  - It was sent into a dead socket: 13 retransmits, never acknowledged.
  - Sandbox.9 has no `tcp_user_timeout`, so that request held the server, and **every** request after it (`/api/health` included) got no answer.
  - CloudFront returned 504s, and UptimeRobot alerted Mason.
- Claude saved the socket state, then ran `sudo systemctl restart knottyyoga-server` at 19:03:09. Health, catalog and classes answered 200 through CloudFront straight away.
  - Outage: **18:52–19:03 UTC**, caused by this test.
  - The helper's 18:58 jobs were caught in the hang. They would have been refused with 403 on sandbox.9 anyway.
- **What the sockets show:**
  - The connection that died was idle from 18:36:33 to 18:52.
  - The global connection behind `/api/health` survived, because UptimeRobot's checks used it at 18:39, 18:44 and 18:49.
  - That is 10/15's pattern exactly, and it answers 4.0's "not fully explained" point.
  - The likely explanation: the NAT entries are still lost on a networkd restart, even with `KeepConfiguration=true`. A connection that sends something within a few minutes gets a fresh entry. An idle one never answers RDS's keepalives, and RDS hangs up.
- **What fixes it is 4.1, not netplan:**
  - Phase 4's keepalives (`keepalives_idle=60`) make every idle connection send a probe every minute, which re-creates its NAT entry well before RDS gives up.
  - `tcp_user_timeout=30000` turns today's hang into a 30-second error.
  - The reconnect brings the next request back.
- [x] Mark the EC2's interface **critical** in netplan, so a `systemd-networkd` restart keeps its DHCP address instead of dropping and re-acquiring it. *It was meant to stop the NAT state from being flushed. It did not (see above).*
  - **Applied 10/16 18:36 UTC.** `/etc/netplan/99-knottyyoga.yaml` (`root:root 600`) merges into cloud-init's `ens5` entry. `netplan get ethernets.ens5` shows the MAC match kept plus `critical: true`, and the generated `10-netplan-ens5.network` gains `KeepConfiguration=true`. A 10-minute automatic rollback timer was armed for the apply and cancelled once SSH and the address checked out.
  - `netplan apply` itself reconfigured `ens5` ("DHCP lease lost", re-acquired the same address); networkd now logs "DHCPv4 connection considered critical".
  - **Removed 10/17 01:37 UTC** as the re-test below. `/etc/netplan/` again holds only cloud-init's `50-cloud-init.yaml`, and the generated `.network` file no longer has `KeepConfiguration`. It was never added to Deploying to AWS's first-boot steps.
  - A `/etc/netplan/99-knottyyoga.yaml` drop-in: `network: {version: 2, ethernets: {ens5: {critical: true}}}`.
  - Checked 10/15: the interface is `ens5`. The only netplan file is cloud-init's `50-cloud-init.yaml`, which matches by MAC with `dhcp4: true`, `dhcp6: false` and sets nothing about `critical`. Netplan merges the drop-in on top by interface name, so cloud-init's file stays untouched.
  - Then `sudo netplan apply`.
- [x] **Verify before relying on it.** Run 10/16 on sandbox.9 against this item's own advice to wait for 4.6. **Failed**, with the outage above.
  - `sudo nsenter -t <server container pid> -n ss -tnoi "( dport = :5432 )"` lists the server's database connections (they live in the container's network namespace, so a plain `ss` on the host does not show them).
  - `sudo systemctl restart systemd-networkd` simulates the upgrade.
- [x] **Re-test once sandbox.10 is live** (4.6), with the same steps: wait at least 15 minutes without touching the site, then load a page that reads the database (Classes). **Passed 10/17, with no reconnect needed.**
  - Pass: it loads first time, with no server restart and no 504.
  - At most one `[database] event=reconnected` line in `sudo journalctl -u knottyyoga-server`; none is better, because it means the keepalives kept the NAT entry alive.
  - Do it by removing the drop-in: `sudo rm /etc/netplan/99-knottyyoga.yaml && sudo netplan apply`. That cleans up and runs the test in one step.
  - **What was run:**
    - A 10-minute automatic rollback timer was armed (to put the drop-in back).
    - The drop-in was removed and `netplan apply` ran at 01:37:35.
    - `systemctl restart systemd-networkd` ran at 01:37:39, the exact 10/15 trigger.
    - networkd logged "DHCP lease lost" and re-acquired `172.31.2.31`. SSH stayed up, and the timer was cancelled.
  - **Right after:** the server's three database sockets (`:38128`, `:38144`, `:38160`) kept their ports. Their keepalive probes were answered: `timer:(keepalive,…,0)`, where the trailing 0 counts unanswered probes. UptimeRobot's health check at 01:38:32, 53 seconds after the flap, got 200 in the same second.
  - **After 16 minutes (01:54:00), through CloudFront:**

    | Page | Status | Time |
    |---|---|---|
    | `/api/classes` | 200 | 0.26 s |
    | `/api/catalog_products` (the request that hung on 10/16) | 200 | 0.43 s |
    | `/api/class_tags` | 200 | 0.15 s |
    | `/api/health` | 200 | 0.17 s |

    - No `[database]` or `[health]` lines in the server journal, so no reconnect was needed.
    - No server restart since the deploy.
    - The same three sockets were still up.
  - **Why it held:** each connection now sends a keepalive within 60 seconds of going quiet. That re-creates its NAT entry long before RDS gives up, so no connection is ever idle long enough to die. Health also probes the site's pool now (4.2), so UptimeRobot touches it every 5 minutes as well. Had a socket died anyway, `tcp_user_timeout` would have failed it within 30 seconds and the next request would have reconnected (4.1).
- **Fallback, not needed (the re-test passed):** if a failure a visitor would see ever turns up again, run the server container with `--network host`, as the helper already does. Then no NAT sits in the database path at all. It is a deploy change to the systemd unit (`-p 80:80` goes away, the server binds port 80 on the host directly) and to `package/systemd/knottyyoga-server.service` in the repo.
- **Not planned:** disabling unattended-upgrades. Security patches matter more than avoiding a reconnect, and the re-test shows the keepalives absorb the network restarts they cause.

---

# Phase 5 — People, roles and instructors

**Implemented 10/16/2026 (evening Pacific).** Windows 5252 passed, Linux 5252 passed. The two runs executed the same 5252 test names; Phase 4's 5244 plus 8 new. The Linux database helper, server and test helper all build, and the log says `honuware : pinned SHA`. **Committed 10/17 as `5ebb942a`** ("Phase 5 — People, roles and instructors"), pushed, and master's pipeline is green.
- **Not tagged, on purpose.** Production re-runs only `--migrate`, which never re-seeds (fact 6), so a `sandbox.11` would change nothing on the live site. The only shipped code is `set_password`, and it has nobody to act on until the reinstall creates the three instructors. Phase 5 reaches production with the one Phase 15 release (its first step).

### 5.1 Temporary password *(business logic; answers OQ-2)*
- [x] New `business_logic/accounts/temporary_password.{h,cpp}` in `knotty_yoga_core`: `Accounts::SetTemporaryPassword(transaction, databaseHelper, secrets, email, password)`.
  - Calls honuware's `PersonHelper::UpdatePassword` with the production secrets, so the hash gets production Argon2 cost.
  - Then `People::SetMustChangePassword(true)`.
  - Returns false for an unknown email.
  - It lives in this repo rather than in honuware so Phase 5 does not wait on a honuware release.
  - **Also returns false, changing nothing, for an empty password and for an unverified account.** The second is `UpdatePassword`'s own rule, and the flag is set only after it succeeds, so a half-finished sign-up is neither activated nor left flagged.
  - New `business_logic/accounts/CMakeLists.txt`, added to `business_logic/CMakeLists.txt`.
- [x] Tests `temporary_password_test.cpp`, five of them:
  - the new password verifies and the old one no longer does, both through `PersonHelper::VerifyPassword`
  - `/api/get_user_info`'s reader reports `must_change_password`. This test calls the endpoint itself, signed in: `false` before, `true` after.
  - an unknown email returns false and changes nothing (a bystander's password and flag are untouched)
  - an empty password returns false and changes nothing
  - an unverified account returns false and is not flagged

### 5.2 `set_password` command *(test_helper)*
- [x] `knottyyoga_test_helper` command `set_password`, alias `sp`, group "Users & Permissions", with `--email` and `--password`. It is a thin call to 5.1. It prints "… must choose a new password at next sign-in", and a clear error for an unknown email. Works against prod the same way `set_secret` does.
  - It prints, for example, "Temporary password set. Beth Cooper (beth.cooper@knottyyoga.local) must choose a new password at next sign-in."
  - For an unknown email: "No verified account with email: nobody@example.com. Nothing was changed.", with exit code 1.
  - It never prints the password.
  - One-shot form: `knottyyoga_test_helper --command=set_password --email=… --password=…`.
- [x] `test_helper` is not linked into the test executable, which is why the logic lives in 5.1, where it is tested.

### 5.3 People *(seed)*
- [x] `PopulatePeople`: add Beth Cooper, Gifford Cheung and Tiana Cheung, emails per **OQ-1**, via `CreateFullyValidatedUser` with a **random, unrecorded** password (D9): 32 bytes from the existing secure-random utility, base64, never logged.
  - The utility is `Auth::AuthHelper::RandomBytes`, which uses libsodium. The password exists only long enough to be hashed.
  - The four development accounts keep their table and their committed passwords. The three instructors are a second table with no password column at all.
- [x] Email constants `kBethEmail`, `kGiffordEmail`, `kTianaEmail` in `seed_app_data.h`. **OQ-1 is still open, so these are its placeholders:** `beth.cooper@knottyyoga.local`, `gifford.cheung@knottyyoga.local` and `tiana.cheung@knottyyoga.local`. Each is one line to change when the real address arrives.
- [x] Kit Williams and `masonbendixen@hotmail.com` stay exactly as they are (OQ-10).

### 5.4 Roles *(seed)*
- [x] `PopulateRoleAssignments`: Instructor role for Mason, Caleb, Beth, Gifford and Tiana. Mason and Caleb keep Administrator.
- [x] `PopulateRolePermissions`: grant the Instructor role `staff_access` (door check-in) and `manage_skills` (granting the approval skills) (OQ-5, confirmed).
  - **Moved, with `PopulateRoles` and `PermissionIdByName`,** from `create_database.cpp`'s anonymous namespace into `seed_app_data.{h,cpp}`, the way Phase 2 moved the other seeders. Nothing could call them from a test before. The move is otherwise unchanged; the two new grants are the only additions.
  - **What `staff_access` opens, besides door check-in.** On the server it is the gate for every staff endpoint:
    - **Check-In** and **Class Check-In**
    - searching people
    - quick accounts for walk-ins
    - drop-in bookings
    - membership upgrades at the door
    - **Student Notes**

    So an instructor can do all of these at the front desk. Nothing in Manage Products, billing or Manage Data is reachable: those need `manage_products`, `manage_subscriptions` or Administrator.
  - **No front-end change was needed.** The **Staff** menu and the `/staff` pages already open for anyone holding the `instructor` permission (honuware's `hasStaffAccess`/`StaffGuard`). Until now, an Instructor-only person saw the portal but the server refused check-in and **Person Skills**.

### 5.5 Instructor profiles *(seed)*
- [x] `PopulateInstructors`: five rows, data-driven, with bios taken **verbatim** from the "Instructor bios" section above, or each name's default where nothing was pasted (OQ-15). The photo column moves to Phase 9's table.
  - No bios had been pasted, so all five are the defaults. `Seed::kSeedInstructors` in `seed_app_data.h` is the one table; the test walks it too (D3).
  - Mason and Caleb keep their headshots. Beth, Gifford and Tiana have an empty photo entry, which is skipped without a "seed_image_missing" warning. Phase 9 attaches their pictures and moves all five into its photo table.

### 5.6 Tests
- [x] All five instructors exist with the expected bios, read through `InstructorProfileHelper::GetInstructorProfile` (the public Instructors page's reader). `PopulateInstructorsSeedsAllFiveWithTheirBios`:
  - the photo flag matches the table
  - Kit and the hotmail account have no instructor profile
- [x] The three new people can **not** sign in with any password present in the source tree (`VerifyPassword` against each dev password returns false), so D9 is pinned. `PopulatePeopleGivesTheNewInstructorsNoKnownPassword`:
  - The passwords tried are `pass`, `mason`, `caleb`, `kit`, the test helper's default `Password123!`, the walkthrough's `Welcome-2026`, and empty.
  - It then shows `set_password`'s path works for all three, which also pins that the seed creates them verified.
- [x] All five hold the Instructor role, through `PersonHelper::GetRolesForUser` (`PopulateRoleAssignmentsResolvesPeopleByEmailNotSeedOrder`, updated).
- [x] An Instructor-only person holds `staff_access` and `manage_skills`, through `PersonHelper::GetPermissionsForUser`. `PopulateRolePermissionsLetsInstructorsCheckInAndGrantSkills`:
  - Beth holds exactly `instructor`, `staff_access` and `manage_skills`, with no admin permission.
  - Gifford and Tiana match her, and Kit holds none.
- [x] Kit and the hotmail account still exist in the base seed with their roles unchanged.
- [x] **Mutation-checked on Linux.** Six deliberate breaks each failed their own test, and nothing else:
  - a fixed `pass` instead of the random password
  - the Instructor's `staff_access` grant removed
  - Tiana's Instructor role removed
  - a wrong bio written for the photo-less instructors
  - the must-change flag not set
  - the empty-password check removed

  The sources were then restored byte-for-byte (hash-checked) and both suites re-run green.
- [x] **End to end, outside the test harness,** on a throwaway database on the local Postgres, dropped afterwards (never `knottyyoga`):
  - `knottyyoga_database_helper --install_schema` ran the real bootstrap, honuware's framework seed included.
  - `set_password` refused `nobody@example.com` and set Beth's.
  - The real server then answered:
    - Beth with `pass`: 401. With the temporary password: 200.
    - `/api/get_user_info` for Beth: `must_change_password: true`, roles `instructor`, permissions `instructor`, `staff_access` and `manage_skills`.
    - `/api/get_user_info` for Kit: no roles, no permissions.
    - `/api/get_instructors`: all five, with their bios; photos for Mason and Caleb only.

### 5.7 Hand-testing walkthrough
From a fresh local database:
1. Recreate it with `knottyyoga_database_helper --recreate_database`.
2. Start the server and the Angular app.

**The instructors**
1. **About** → **Instructors**. Five cards are shown: Mason Bendixen, Caleb Ault, Beth Cooper, Gifford Cheung and Tiana Cheung.
2. Check the three new cards:
   - Beth's reads "Teaches Beginner Knotty Yoga on Wednesdays."
   - Gifford's and Tiana's each say they co-teach Tuesdays with the other.
   - The three new cards have no photo yet (Phase 9).

**Nobody knows the new instructors' passwords**
1. **Sign In**: Email `beth.cooper@knottyyoga.local`, Password `pass`, then **Log In**. The sign-in is refused.

**Letting Beth in**
1. In `knottyyoga_test_helper`'s REPL, run `set_password --email=beth.cooper@knottyyoga.local --password=Welcome-2026`, or `sp` with the same arguments. It prints "Temporary password set. Beth Cooper (beth.cooper@knottyyoga.local) must choose a new password at next sign-in."
2. **Sign In**: Email `beth.cooper@knottyyoga.local`, Password `Welcome-2026`, then **Log In**. The site opens **Change Password**.
3. Click **Staff** in the top menu. The site sends you back to **Change Password**.
4. On **Change Password**, enter:
   - **Old Password** `Welcome-2026`
   - **Reenter Old Password** `Welcome-2026`
   - **New Password** `Beth-Studio-2026`

   Then click **Update**.

**What an instructor can do**
1. The top menu now has **Staff** and no **Admin**.
2. **Staff** → **Person Skills** card. In **Search for a person**, type `Kit` and click **Kit Williams**. It shows "This person has no skills assigned yet." and an **Assign new skill** button, with no error banner. The approval skills themselves arrive in Phase 6.
3. **Back to staff portal** → **Check-In** card. It opens without a "not authorized" error.

**A customer sees none of it**
1. Sign out: **Beth** (the name menu) → **Sign Out**, then **Log Out** in the dialog.
2. **Sign In** as `kiteagle@gmail.com` / `kit`. There is no **Staff** in the top menu.

---

# Phase 6 — Classes, approval skills, access rules and the weekly schedule

**Implemented 10/17/2026.** Windows 5259 passed, Linux 5259 passed. The two runs executed the same 5259 test names: Phase 5's 5252, less the three per-class slot tests, plus ten new. The Linux database helper, server and test helper all build, and the log says `honuware : pinned SHA`. Like the rest of the seed, it reaches production with the Phase 15 reinstall (fact 6).

All four tables are in `seed_app_data.h`, each walked by one seeder and by its tests (D3):
- `kSeedClasses`
- `kSeedSkillLevels`
- `kSeedClassAccess`
- `kSeedClassSlots`

### 6.1 Skill levels *(seed)*
- [x] New `Seed::PopulateSkillLevels`: the three skills from the table above, via `TableHelpers::SkillLevels::AddSkillLevel` (code, name, description), with `sort_order` 10/20/30. Codes `kHandstandSkillCode`, `kAerialRopeSkillCode` and `kIntermediatePartnerAcroSkillCode`.

### 6.2 Classes *(seed)*
- [x] `PopulateClasses`: the six classes with the Overview's descriptions, kind `recurring` (the column default). The four old classes are no longer created.
- [x] Copy is normalised only for spacing ("Beginner Knotty  Yoga" in the Overview has a double space). Otherwise verbatim; the studio can edit it in Manage. A script compared all six descriptions and the three skill texts with this document: identical.
- [x] Class-name constants in `seed_app_data.h`. The slot, access and photo tables all refer to these.
- [x] **Also changed: the three existing class photos.** `PopulateSeedPhotos` named "Partner Acrobatics", which no longer exists. Its picture now goes on **Beginner Partner Acro**, and Knotty Yoga and Handstands keep theirs. The other three classes have no photo until Phase 9.

### 6.3 Access rules *(seed)*
- [x] New `Seed::PopulateClassRequirements`, data-driven from the Class access table:
  - `ClassRequirementGroups::AddGroup` with label "Membership" or "Instructor approval", and `inline_description` as in the table. Customers see it once Phase 13 lands; until then only admins do (fact 9).
  - The membership descriptions are "Silver or Gold membership" (Beginner Knotty Yoga) and "Gold membership" (the other five). The approval groups read "Instructor approval: Handstand" and so on.
  - `ClassRequirementGroupLiterals::AddPermissionLiteral` / `AddSkillLiteral` for the literals.
- [x] Permissions and skills are resolved by name and code. **Changed from the plan: a missing one is logged and left out of its group, and the group is still written. It fails closed.** The plan said to skip the whole class's rules, and its reason was wrong: a class with **no** groups is open and free to everyone (D7), which is the most permissive outcome of all.
  - With a literal left out, a group can only admit fewer people. A group none of whose literals resolved admits nobody: `CheckAccess` treats an empty group as unsatisfied. It also shows up in the Requirements editor, where it can be fixed.
  - The log line is `event=class_requirement_literal_missing class=… skill=…` (or `permission=…`).
- [x] Runs after `PopulatePermissions`, `PopulateSkillLevels` and `PopulateClasses`. In `PopulateTables`: `PopulateSkillLevels` → `PopulateClassRequirements` → `PopulateClassSchedules`, after the instructors.

### 6.4 The weekly schedule *(seed)*
- [x] Rewrite `PopulateClassSchedules` around one `constexpr` slot table (D3) holding the 14 rows above. Each row is `{ className, dayOfWeek, startMinutes, durationMinutes, instructorEmail, predecessor { className, dayOfWeek } }`. The days are written as `Scheduling::kSunday` … `kSaturday`, so the table reads like the Overview.
- [x] One perpetual instance + default schedule per class (the existing `AddPerpetualSchedule` shape), all bound to `class-dropin` (unchanged decision D8 of the previous plan). Each is created with the class's first slot.
- [x] Predecessors are resolved through a `(className, dayOfWeek) → slotId` map, filled as slots are created.
  - The table is ordered so a predecessor always precedes its follower.
  - If one doesn't, the seeder logs `event=class_slot_skipped … reason=predecessor_missing` and skips the follower rather than writing a dangling id. The predecessor is checked before the class's schedule is created, so a class whose every slot is skipped is left with no empty schedule either.
  - A slot whose instructor is not seeded is still written, with no instructor (TBD), as before.
- [x] Tuesday Beginner Knotty Yoga gets Gifford as `instructor_person_id`. Tiana is added in Phase 11.
- [x] **Known, until Phase 10:** the test-only seed (`--recreate_database_test`) still adds the Aerial Series at Tue + Thu 19:00 in the Main Gym, which now double-books Thursday's Intermediate Partner Acro (fact 7). Phase 10 deletes it, so it was not moved. Plain `--recreate_database` is unaffected.

### 6.5 Tests
All in `seed_app_data_test.cpp`. Members are made the way a purchase makes them: a purchase, an entitlement built from the membership product's **own** entitlement rule, and a seat (`JoinMembership`). So the access tests cover the whole chain the hand tests walk (product → rule → permission → the class's group), not a permission granted by hand.
- [x] Replace `PopulateClassesSeedsTheSixStudioClasses` with the new six, plus "none of the four retired names exists". Read through `ClassCatalogHelper::GetActiveClasses`, the signed-out **Our Classes** reader: names, descriptions, and kind `recurring`.
- [x] `PopulateSkillLevelsSeedsTheThreeApprovalSkillsInOrder`: through `SkillLevels::GetActiveSkillLevels`, in sort order, with their descriptions.
- [x] **Access through `ClassAccessHelper::CheckAccess`** (D4), four seeded people (`PopulateClassRequirementsAdmitsEachKindOfMemberToItsClasses`):
  - Silver-only member: Beginner Knotty Yoga **allowed**, all five others **denied**.
  - Gold-only member: Beginner Knotty Yoga, Knotty Yoga and Beginner Partner Acro **allowed**. Handstands, Aerial Rope and Intermediate Partner Acro **denied**, on the approval group only.
  - Gold member + `handstand`: Handstands **allowed**, Aerial Rope still denied.
  - Non-member: everything denied. `Annotate` reports `members_only` for Beginner Knotty Yoga and Knotty Yoga, and `needs_skill` for Handstands (skill gates outrank membership gates). This is its own test, `PopulateClassRequirementsTellsAVisitorWhatEachClassNeeds`, covering all six classes. It also pins that Beginner Knotty Yoga's buy link names Silver, the cheaper tier.
- [x] **Added:** `EveryMembershipOpensTheClassesItsTierIncludes`. A member of each of the five membership products may attend Beginner Knotty Yoga, and only the three Gold ones may attend Knotty Yoga.
- [x] **Added:** `PopulateClassRequirementsReadsBackInTheRequirementsEditor`. Each class's groups through `GetClassRequirements`, the Requirements card's reader: labels, descriptions and literals, matched by label.
- [x] **Added:** `PopulateClassRequirementsFailsClosedWithoutTheSkills`. With no skills seeded, a Gold member is still refused Handstands, Aerial Rope and Intermediate Partner Acro, on the approval group (6.3).
- [x] Schedule: for each class, its slots match the slot table by **weekday and start** (never by array index), with the right instructor email. `PopulateClassSchedulesPutsTheWeeklyScheduleOnTheCalendar` reads it through `CalendarHelper::GetCalendar`, the feed behind the **Weekly schedule** page:
  - every one of the 14 slots, on its **local** weekday, at its local start and end (this pins the 0 = Sunday convention end to end)
  - the instructor's full name and the "Requires attending" class
  - each class's badge for a signed-out visitor
- [x] Predecessors: each of the four followers names the right slot, **and** `predecessor.start + predecessor.duration == follower.start`. `PopulateClassSchedulesLinksEachFollowerToTheClassJustBeforeIt` reads them through `GetSlotsByImplementationWithPredecessor`, the admin slot editor's reader.
- [x] **Added:** `PopulateClassSchedulesSkipsAFollowerWhosePredecessorIsMissing`. Without Beginner Partner Acro, both Intermediate Partner Acro slots are skipped and that class gets no schedule. The other 10 slots, and the other followers' links, are written.
- [x] Session instructors through `ClassCatalogHelper::GetClassDetail(...).upcomingSessions[].instructorNames`. For example, Wednesday Beginner Knotty Yoga reads "Beth Cooper". `PopulateClassSchedulesNamesTheInstructorOnEachClassPage` covers every slot, matched by slot id.
- [x] `NoTwoSlotsCollideInTheSameRoom` over the 14 slots, using half-open intervals so back-to-back classes are legal. It replaces `NoTwoSlotsCollideInTheSameRoomAfterTheFullTestSeed`, which included the test-only seed (6.4, known until Phase 10).
- [x] Update `PopulateClassSchedulesResolvesRowsByNaturalKeyNotSeedOrder` and `PopulateClassSchedulesSkipsWhenPrerequisitesAreMissing` to the new table, and delete the three per-class slot tests they replace. Both now check all six classes and all 14 slots.
- [x] Photo and class-name updates in the existing tests: `PopulateSeedPhotosAttachesProfileAndClassPictures` checks the three classes with a picture and the three without, and `AttachSeedPhotoSkipsAMissingFileWithoutThrowing` uses Aerial Rope instead of Tumbling.
- [x] **Mutation-checked on Linux.** Eight deliberate breaks, each caught by the test written for it:

  | Break | Failed |
  |---|---|
  | Silver dropped from Beginner Knotty Yoga | `…AdmitsEachKindOfMemberToItsClasses`, `EveryMembershipOpens…`, `…TellsAVisitorWhatEachClassNeeds` |
  | The approval group skipped when its skill is missing (fail open) | `…FailsClosedWithoutTheSkills` |
  | Sunday written as ISO 7 | `…PutsTheWeeklyScheduleOnTheCalendar`, `…NamesTheInstructorOnEachClassPage` |
  | Thursday Intermediate Partner Acro linked to Sunday's Beginner Partner Acro | `…LinksEachFollowerToTheClassJustBeforeIt` |
  | A follower written without its link when the predecessor is missing | `…SkipsAFollowerWhosePredecessorIsMissing` |
  | Skill sort order swapped | `…SeedsTheThreeApprovalSkillsInOrder` |
  | Wednesday Beginner Knotty Yoga given to Caleb | `…PutsTheWeeklyScheduleOnTheCalendar`, `…NamesTheInstructorOnEachClassPage` (the spelled-out "Beth Cooper" checks) |
  | Partner acro photo on Intermediate Partner Acro | `PopulateSeedPhotosAttachesProfileAndClassPictures` |

  - The follower-link break gave no result the first time, in the batch of eight, and that log was not kept. Run again on its own, it was caught.
  - The sources were restored byte for byte, with hashes checked inside the container and again from Windows. The 60 seed tests then passed on the restored files.
- [x] **End to end, outside the test harness,** on a throwaway database on the local Postgres (`phase6_check`, dropped afterwards; never `knottyyoga`):
  - `knottyyoga_database_helper --install_schema` ran the real bootstrap in its real order. The log has no `seed_lookup_missing`, `*_skipped`, `class_requirement_literal_missing` or `seed_image_*` line.
  - The real server's `/api/classes`, signed out: the six classes.
  - Its `/api/calendar`, signed out, over eight days: all 14 slots, each on its Pacific weekday and time, with its instructor, its badge (`members_only` on three classes, `needs_skill` on three) and the four "Requires attending" classes, exactly as in the schedule table.
  - `/api/classes` signed in as Mason: **zero** classes, which confirms OQ-22.

### 6.6 Hand-testing walkthrough
From a fresh local database:
1. Recreate it with `knottyyoga_database_helper --recreate_database`.
2. Start the server and the Angular app. Use a window at least 1280 px wide, so the top menu isn't collapsed.

**Signed out: the six classes**
1. Top nav **Our Classes**. The page is headed **All Classes** and shows six cards:
   - **Beginner Knotty Yoga**
   - **Knotty Yoga**
   - **Handstands**
   - **Aerial Rope**
   - **Beginner Partner Acro**
   - **Intermediate Partner Acro**

   There are no Therapeutic Knotty Yoga, Partner Acrobatics, Tumbling or Aerial Fabric cards. Knotty Yoga, Handstands and Beginner Partner Acro have a photo; the other three have none until Phase 9.
2. Click **Weekly schedule** (page headed **Our Classes**). The week has 14 classes matching the schedule table; Friday reads **No classes**. Each row shows its start time and length, then "with" the instructor. For example:
   - **Saturday**: **Knotty Yoga**, "10:00 AM · 120 min", "with Mason Bendixen"
   - **Wednesday**: **Beginner Knotty Yoga**, "5:00 PM · 60 min", "with Beth Cooper"
   - **Tuesday**: **Beginner Knotty Yoga**, "6:00 PM · 60 min", "with Gifford Cheung"

   If today's classes have already ended, click **Next week** to see a whole week.
3. These four rows also read **Requires attending: …**:
   - Monday 7:00 PM Handstands → **Knotty Yoga**
   - Wednesday 7:00 PM Aerial Rope → **Knotty Yoga**
   - Thursday 7:00 PM Intermediate Partner Acro → **Beginner Partner Acro**
   - Sunday 12:00 PM Intermediate Partner Acro → **Beginner Partner Acro**
4. Same page, badges:
   - **Beginner Knotty Yoga**, **Knotty Yoga** and **Beginner Partner Acro** rows carry **Requires a membership**.
   - **Handstands**, **Aerial Rope** and **Intermediate Partner Acro** carry **Prerequisite needed**.
5. Click **Prerequisite needed** on the Monday 7:00 PM **Handstands** row. A dialog titled **Skill required** lists **Handstand** with "Able to easily kick up to a wall and hold a belly to the wall handstand for thirty seconds." Click **Got it**.
6. **Our Classes** → **Handstands** card. **Required Skills** shows the **Handstand** chip, and **Pricing** reads **Members only**, "Upgrade your membership to attend."

**As the administrator: the rules behind it**
1. **Sign In**: `masonbendixen@gmail.com` / `pass`.
2. **Admin** → **Manage Products** → **Class Schedules** → **Class**: **Beginner Knotty Yoga**. The **Requirements** card holds one group, **Membership**, with the Silver and Gold permissions. **Description (shown to members)** reads "Silver or Gold membership".
3. Same page, **Class**: **Handstands**.
   - The **Requirements** card holds **Membership** (Gold, "Gold membership") and **Instructor approval** (Handstand).
   - Expand **Class Instances** → **Class Schedules** → **Class Schedule Slots**. There is one slot: **Monday**, **7:00 PM**, **60**, **Instructor** Mason Bendixen, **Requires attending** **Knotty Yoga · 6:00 PM–7:00 PM**.
4. **Admin** → **Manage Products** → **Skills**: three skills, **Handstand**, **Aerial Rope** and **Intermediate Partner Acro**.

**An instructor's approval is half of the rule**
1. Still as Mason: top nav **Staff** → **Person Skills** card → **Search for a person**: type `Kit` and click **Kit Williams**. Then **Assign new skill** → **Skill**: **Handstand**, **Note (optional)**: `Belly to the wall, 30 seconds` → **Assign**.
2. Sign out: **Mason** (the name menu) → **Sign Out**, then **Log Out** in the dialog. **Sign In** as `kiteagle@gmail.com` / `kit`.
3. **My Classes** → **Full Schedule**:
   - Monday 7:00 PM **Handstands** now reads **Requires a membership**, not **Prerequisite needed**. The approval is in; the Gold membership is still missing.
   - Wednesday 7:00 PM **Aerial Rope** still reads **Prerequisite needed**: approval for Handstand opens nothing else.
4. Known, not a fault of this phase: signed in, **Our Classes** → **All Classes** shows Kit no cards at all, because it lists only classes he may already attend (OQ-22, fixed in 13.3).

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

# Phase 8 — About page copy, the location's text and the sender name

### 8.1 Seed
- [ ] `PopulateAboutSections`: the three bodies from the About page table, **verbatim**. Titles, ordinals and links unchanged. The "40 years … hootananny" and "climbing wall" placeholder copy is gone.
- [ ] `PopulateFacilities`: the studio's `description` becomes the **About ▸ Our Location** text above (OQ-16). It is a named constant in `seed_app_data.h`, so the test reads the same value the seeder writes.
- [ ] `app_secret_values.cpp`: `kMailSenderNameValue` becomes **"Knotty Yoga"** (OQ-16). It is also the site's **studio name** (`Mail::LoadTenantBranding` sets `studioName = senderName`). So the same change fixes the footer's "© Copyright Knotty Yoga and Spa 2020" and email subjects such as "Welcome to Knotty Yoga and Spa - Your Account". The "2020" is hard-coded in `footer.component.html`, and nothing here changes it. Move it into `app_secret_values.h`, so the tests that check a message's From name compare against the constant rather than a copy of the string.
  - Eight tests assert the literal "Knotty Yoga and Spa" today: `app_secret_values_test`, `attendance_template_helper_test` (twice), `class_series_helper_test`, `favorite_instructor_helper_test`, `weekly_digest_helper_test`, `card_expiring_notification_test` and `signup_reminder_helper_test`. `class_checkin_helper_test` expects the subject "Welcome to Knotty Yoga and Spa - Your Account". It needs checking at implementation whether that subject is built from the sender name or from a separate site-name slot.
- [ ] Not changed: `ServerAccess.mock.ts`'s local-mode copy of the old location text. It is fixture data, as 13.2 notes for the mock catalog.

### 8.2 Tests
- [ ] Through the public About reader (the one behind `/api/about_sections`): three blocks, in ordinal order, with the new bodies, and no block containing "hootananny".
- [ ] Through the location reader the public page uses (`TableHelpers::Facilities::GetActiveFacilities`, as `PopulateFacilitiesGivesTheStudioADescriptionAndTimezone` does today): the studio's description is the new text, and contains neither "climbing wall" nor "Street parking".
- [ ] Sender name through the app's secret defaults (`app_secret_values_test`) and through one real message's From name. The eight existing assertions switch to the constant.

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

### 13.3 Signed-in visitors see every class (OQ-22, default)
- [ ] **Business logic.** `ClassCatalogHelper::GetClassesVisibleToPerson` keeps its "has an upcoming run" filter and drops the access-gate filter, so a signed-in visitor gets the same classes as a signed-out one. Update the `class_catalog_helper_test` cases that pin the hiding, and add one through the seeded rules: a signed-in non-member and a Silver member each see all six.
- [ ] **Endpoint.** `/api/classes` test: a signed-in non-member gets six classes.
- [ ] Check in the front end: the class page already shows **Members only** and **Required Skills** for a class the viewer can't attend. Whether an **All Classes** card should carry a badge too is checked at implementation; the weekly schedule's badges are the model.

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
   - **About** → **Our Location** shows the studio photo, and text beginning "Our home gym: a full acrobatics floor with crash mats, aerial rigging, 24 fabrics, 9 ropes…" with the 520 directions. No climbing wall, no "Street parking is free after 6pm".

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
26. In the masonbendixen@gmail.com inbox, open **Reset your Knotty Yoga password**. The sender reads **Knotty Yoga**, not "Knotty Yoga and Spa". Click the link. The **Reset your password** page opens.
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
- → 10/16: Phase 5 shipped with the placeholders, so nothing waits on this. Swap them in before Phase 15. Until then the instructor digest and, after Phase 12, password-reset emails to these three go nowhere.

**OQ-11 — Where are the photos?**
- Mason- Taking the photos IS a phase. Please list this as an early phase that the later phases will use. I'll try to take the photos tonight.
- → Plan: Phase 1 is the shoot, with a shot list, framing notes and checkboxes. **Default hand-off:** drop the originals into `C:\Users\mason\Documents\Obsidian\Knotty Yoga\Photos\` named as in 1.1 (any format or size); Claude converts and checks them (1.2). Say if you'd rather use another folder.

**OQ-15 — Instructor bios.**
- Mason- I asked my instructors to write bios. Leave a place for me to put them as they send them to me. Worst case, I manually add them once deployed but hopefully they get them to me before then.
- → Plan: the **Instructor bios** section above has a slot under each name. Whatever is there at Phase 5.5 is seeded verbatim, anything still empty gets its one-line default, and late ones go in through Manage after deploy (15.3).

**OQ-22 — Signed in, Our Classes hides every class you can't attend yet.** *(Found implementing Phase 6, 10/17.)*
- Signed out, **Our Classes** → **All Classes** lists all six classes. Signed in, it lists only the classes the person may already attend (`ClassCatalogHelper::GetClassesVisibleToPerson`, a deliberate rule from the access redesign, P-1/M-6). Until Phase 6 every class was open, so nobody noticed.
- Now that all six need a membership, a signed-in **non-member sees an empty page**, a Silver member sees only Beginner Knotty Yoga, and **so do the staff**: Mason's and the instructors' accounts hold no membership. The weekly schedule is unaffected: it lists all 14 rows with their badges, signed in or out.
- It also breaks 14.3 step 13, where a Silver Kit opens Knotty Yoga from **All Classes**.
- **Default:** signed-in visitors see every scheduled class, as signed-out ones do, and the class page and badges say what each one needs. New item 13.3. Say if a signed-in member should keep seeing only what they can book.
- Mason- Let's go with the default.

### Resolved

*Answered 10/15 first (OQ-16, 19, 20, 21), then the 10/14 answers.*

**OQ-16 — Leftover copy that no longer matches the studio.**
- Mason - Let's do: Who we are / How we teach / Come and find us *(full text moved into the About page table above, verbatim)*.
- → Plan: Phase 8, which now covers all three pieces of copy. The two follow-ups below were answered 10/15:
  - The mail sender name **"Knotty Yoga and Spa"** (`app_secret_values.cpp`). With the spa removed, would you like plain "Knotty Yoga"?
	  - Mason- Let's just do Knotty Yoga
	  - → Plan: 8.1. `kMailSenderNameValue` becomes "Knotty Yoga", and the eight tests that check the From name switch to the constant. Production picks it up with the Phase 15 reinstall.
  - The location's description on **About ▸ Our Location**, which still mentions "a climbing wall" and "Street parking is free after 6pm". Send replacement text, or say "drop the climbing wall".
	  - Mason- Let's say: "Our home gym: a full acrobatics floor with crash mats, aerial rigging, 24 fabrics, 9 ropes, plus a private room for massage and bodywork. Parking is abundant. Conveniently off the 520. Heading Eastbound on 520 take the first 148th exit and then the right sub exit labelled 152. Go "through" the first roundabout and then go "left" at the second. We in the last building (14) before the parking lot bends to the right. Space 2545 / Knotty Yoga."
	  - → Plan: the **About ▸ Our Location** section under *Target content*, seeded in 8.1. One word was changed, "We in" → "We're in", which is flagged there; say if it should stay exactly as typed.

**OQ-19 (new) — Knotty Yoga approval as a skill?** The Overview says Knotty Yoga "requires instructor approval to attend. Should be able to do a pull-up and five regular pushups". The OQ-4 answer lists skills only for handstand, aerial_rope and intermediate_partner_acro.
- **Default (written into the plan):** Knotty Yoga needs Gold only, and approval is checked at the door, as the class description says. Gold members can book it without an instructor first approving them in the system.
- **Alternative:** a fourth skill `knotty_yoga` ("Able to do a pull-up and five regular pushups."), gated like Handstands. Every Gold member would then need an instructor's approval before booking the main class.
- Mason- Let's go with the default.
- → Plan: no change; the plan was already written this way (D6, and the Class access table). Knotty Yoga is Gold-only, with approval checked at the door.

**OQ-20 (new, from Phase 3) — Five tiers on a three-column grid.** On a desktop-width screen the membership cards sit three to a row. So Silver, Silver Couple's and Gold share the first row, and Gold Couple's and Gold Family sit alone on the second. Gold is split from its own tiers, and the second row is two-thirds full. This shows on the Memberships page, the home page band and the shop.
- **Default:** leave it. The order is right; only the line break falls mid-Gold. You'll see it in the Phase 14 walkthrough.
- **Alternatives:** (a) two to a row on desktop: Silver pair, Gold pair, then Gold Family alone. (b) one row per tier family: Silver row and Gold row. That needs the cards to know which family a tier is in, which the data does not say today.
- Mason- Let's go with the default.
- → Plan: no change. The cards stay three to a row in tier order.

**OQ-21 (new, from the 10/15 incident) — Who tells you the site is down?** Today nothing does: you find out by visiting it.
- **Default (4.7):** a Route 53 health check plus a CloudWatch alarm that emails you. It stays inside AWS, needs no new account, costs about $1 a month, and checks every 30 seconds.
- **Alternative:** a free outside service (e.g. UptimeRobot) checking the same URL every 5 minutes. No AWS setup, but it is one more account, and an outage is noticed more slowly.
- **Not planned:** a watchdog that restarts the server whenever health fails. After 4.1 the server reconnects by itself, so a restart would only hide the next new kind of failure.
- Mason- We did UptimeRobot as part of the deploying to AWS guide and it contacts me if the health check fails. But, because of the bug, the health check wasn't failing.
- → Plan: 4.7 is rewritten around your existing UptimeRobot monitor, and the Route 53 default is dropped. The monitor already alerts on any non-2xx answer, and 4.2 makes health answer 503 in exactly this situation, so after 4.6 the monitor would catch a repeat of 10/15 with no change to it. Deploying to AWS's uptime section now carries a correction note saying the old "db ok means the database answered" claim was wrong.

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
