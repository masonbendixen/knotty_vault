---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 10/19/2026
Version: 0.21
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

- **v0.22 (10/20/2026)** — **Phase 13 is built**, with the photos and bios Mason supplied. It is all in knottyyoga and not committed.
  - Six photos are converted and in the seed: all five instructor headshots (Mason's and Caleb's replace the old ones) and Intermediate Partner Acro. No reshoots are needed (1.2). Four are still to come: Beginner Knotty Yoga, Aerial Rope, Massage and Studio (OQ-11).
  - All five bios replace the one-line defaults (OQ-15, now resolved). Six slips in them were corrected; **OQ-28** lists each, default keep.
  - Every seed picture now sits in one of four tables in `seed_app_data.h`, and the tests walk those same tables. The headshots moved off `PopulateInstructors`, and the bootstrap now attaches photos after the instructor rows exist.
  - Added **13.3**: the new headshots are waist-up, and every page cut its square from the middle, taking off the top of the head. The four places a headshot appears now crop from the top.
  - Results: knottyyoga Windows 5389 and Linux 5389, with the same names on both; Angular 3622; all 11 mutations caught. End to end, a fresh install attaches every listed picture and serves every bio intact (13.4). Hand-test steps are in 13.6.
  - **Later the same day — 13.7, sharp pictures.** Mason's hand-test found the headshots blurry and blocky. The cards were asking for pictures smaller than they show them, so the browser enlarged a crude server-side shrink. Every square headshot and class card now asks for three times its size. That is a page-only change; the server and seed are untouched.
  - **13.8, Our Providers.** Signed out, Caleb's provider card showed a broken image. It asked for his private account photo; it now shows his public instructor headshot, as its bio already did.
  - After both: knottyyoga Windows 5392 and Linux 5392, same names; Angular 3627; all 8 new mutations caught (13.9).
  - **Ryan's edited headshots** replace Mason's, Caleb's, Gifford's and Tiana's, both in the seed and on the live local site, through the admin photo upload (1.2).
  - **13.10, Site Theme speed.** The admin Site Theme page took 22 s to open, and a keystroke could freeze it for 4 s. Its colour pickers built a new canvas for every colour on every screen update. Now it opens in 2.4 s and typing is instant (Angular 3630).
- **v0.21 (10/19/2026)** — **Phase 12 is built.** It is all in knottyyoga and not committed.
  - Each class page names the membership it needs ("Members only: Silver or Gold membership").
  - Signed in, **All Classes** lists all six classes for everyone.
  - The dead homepage service is gone.
  - Found while checking 12.3: **Required Skills** had never appeared on any class page, because the server never filled it in. 12.1 fixes it with the same lookup.
  - Two new open questions: OQ-26 (a badge on the All Classes cards?) and OQ-27 (what an unapproved Gold member reads under **Pricing**).
  - Results: knottyyoga Windows 5389 and Linux 5389, Angular 3618, and all 12 mutations caught. Hand-test steps are in 12.6.
- **v0.20 (10/19/2026)** — **Phase 11 is complete.**
  - Mason hand-tested forgot password and committed everything.
  - The checks the low-memory stop had interrupted are now run against the committed code:
    - **11.8:** all 22 C++ mutations and all 7 `@honuware/ui` mutations are caught.
    - **11.9:** the end-to-end install pass passes 19 of 19 checks, and the migrate pass rebuilds the table on a pre-Phase-11 database.
  - No code changed. The C++ mutation script's `+` case now points at `util/types.cpp`, where 11.11 moved the encoder. One library mutation was rewritten so it compiles.
- **v0.19 (10/19/2026)** — **11.11 is pinned in both apps.** Mason pushed the activation-link fix as `server_components` `4b59f5b` (green in CI). Both apps now pin it: knottyyoga Windows 5382 and Linux 5382; CommunityFinder Windows 1916 and Linux 1916; same names on both platforms. Phase 11's code is complete. Still to do: Mason's commit of the two pin lines and the knottyyoga UI changes, his hand-test (11.10, now 15 steps), and the 11.8/11.9 runs stopped earlier by low memory.
- **v0.18 (10/19/2026)** — **Mason answered OQ-23, 24 and 25; Phase 11 gains the activation-link fix.**
  - OQ-23 and OQ-24: the defaults stand, with no code change. Both moved to *Resolved*.
  - OQ-25: **fix it now.** New **11.11**: the activation link encodes the address the way the reset link does, so `mason+test@…` can activate. The encoder moved into foundation (`URLEncodeQueryValue` in `util/types.h`) and both mails use it. That is a second `server_components` change, so 11.11 ends with another commit, pin bump and gates in both apps.
  - The Phase 11 release so far: `server_components` `4f72341` is pinned in knottyyoga and CommunityFinder (Windows 5376 and Linux 5376; Windows 1910 and Linux 1910), and `@honuware/ui` 0.3.0 is on npm and installed (3614 Angular specs).
- **v0.17 (10/18/2026)** — **Phase 11 is built, in all three repos; some checks wait on memory.**
  - `server_components` has the table, the migration, the reset helper and mail, the gate checks, and the two endpoints. Windows 1889 passed, Linux 1889 passed, with the same names on both. knottyyoga built against it passes 5376 on Linux, and both of its programs build.
  - `@honuware/ui` 0.3.0 has the **Forgot password?** link and the two new pages: 514 specs, lint and build clean. knottyyoga is wired to them and passes 3614 Angular specs against a local build of 0.3.0.
  - Found while building: both endpoints had to be exempt from the CSRF guard, or no signed-out visitor could use them (11.4).
  - **Stopped by a low-memory reap and not restarted:**
    - knottyyoga's Windows build against the local checkout
    - the mutation checks (11.8)
    - the end-to-end run (11.9)

    They are ready to run when Mason says.
  - New **OQ-25**: the *activation* link has always broken for plus-addressed emails. Default: fix it next time the component is opened.
- **v0.16 (10/18/2026)** — **Phase 10 is implemented.** Windows 5306 passed, Linux 5306 passed, with the same test names on both; Angular 3608 passed.
  - A slot can now carry co-instructors, and Tuesday's Beginner Knotty Yoga is Gifford's and Tiana's on the weekly schedule, the class page, both instructors' profiles and My Classes. A substitution still replaces both.
  - Pay, the load grid, the exception digest and favorite alerts stay keyed on the lead, as planned.
  - Admins add and remove co-instructors in the slot dialog (**Co-instructors** chips).
  - All 14 C++ and 3 Angular mutations were caught. End to end, a fresh install and an upgraded database both behaved.
  - Small additions beyond the plan, each noted in 10.3: assistants no longer headline a session; My Classes uses the same rule; "copy slots forward" carries co-instructors; "Substituting for" names the whole team.
  - New **OQ-24**: a shift trade on a co-taught date drops the co-instructor from that date. Default: leave it.
- **v0.15 (10/18/2026)** — **Phase 9 is implemented.** Windows 5260 passed, Linux 5260 passed, with the same test names on both.
  - The test-only seed is gone: no Intro Workshop sessions, no Aerial Series, and no `--recreate_database_test` flag. Every local database is now what production gets. The `intro-workshop` product stays.
  - All four mutations were caught by the new `TheBaseSeedHasNoSampleData`. End to end, the old flag is refused as unknown, and a fresh install shows no sample data on any page.
  - Left in place, noted in 9.1: `NextWeekdaysAfter`, a tested date utility whose only caller was the deleted workshop seeder.
- **v0.14 (10/18/2026)** — **The photos phase moves to just before Verification**, because the photos aren't in yet (OQ-11). The four phases that don't use them each move up one:

  | Was | Now | Phase |
  |---|---|---|
  | 10 | **9** | Retire the test-only seed data |
  | 11 | **10** | Co-instructors on a recurring slot |
  | 12 | **11** | Forgot password |
  | 13 | **12** | The class page names the membership; cleanup |
  | 9 | **13** | Attach the photos |

  - Every reference in the plan follows the new numbers, except in the revision notes below, which keep the numbers they were written with. Nine code comments in `src/database_helper/` that name these phases were renumbered too.
  - Verification (14) and the production rollout (15) stay last: Verification's walkthrough checks the photographs.
  - Phase 8 was validated by hand and committed as `f12a4c69`.
- **v0.13 (10/18/2026)** — **Phase 8 is implemented.** Windows 5268 passed, Linux 5268 passed, with the same test names on both.
  - About carries your three blocks, the location your text with the directions, and the site's name is "Knotty Yoga". That name is also the email sender, the footer's name and the name in email subjects.
  - All eight mutations were caught. The end-to-end run compared the real server's answers with this document, character for character, and they match.
  - Left as it was, noted in 8.1: an unused site setting still holds the old "40 years … hootananny" text. Nothing on the site has shown it since the About page became blocks.
  - Phase 7 was committed as `74af2aa8`, not tagged.
- **v0.12 (10/17/2026)** — **Phase 7 is implemented.** Windows 5267 passed, Linux 5267 passed, with the same test names on both.
  - Caleb is the only massage therapist. His real hours are a base-seed template, "Massage hours", with no end date, generated through the end of next month.
  - New **7.4**: **Generate Availability** pressed twice used to double the hours. 15.4's monthly press always overlaps the last one, so the plan as written would have doubled every day. It now skips days the template already generated.
  - All nine mutations were caught, and the end-to-end run on a throwaway database matched the hours.
  - New **OQ-23**: **Edit Template** removes the hours its template generated, so after an edit **Book Now** offers no massage until Generate is pressed. The default is a note in 15.4 and no code change.
  - Phase 6 was committed as `32c9235c`.
- **v0.11 (10/17/2026)** — Mason's answer to **OQ-22**: the default. 13.3 stands as written, so signed-in visitors will see every class, as signed-out ones do. OQ-22 moves to Resolved.
  - OQ-1 (instructor emails), OQ-11 (photos) and OQ-15 (bios) stay open, each still waiting on the instructors or the photo shoot. Their defaults are unchanged. Of these, only the photos hold up a phase: Phase 9 attaches whatever exists when it lands, and any photo still missing is added later (9.1). The emails are needed before Phase 15; bios can even come after it, through Manage.
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
| Test-only seed (`--recreate_database_test`) | Intro Workshop sessions, Aerial Series, Caleb's Mon–Fri 9–5 template | Retired (Phase 9). The local database matches production |
| Password recovery | None. A forgotten password cannot be recovered | **Forgot password?** on the Log In page (Phase 11) |
| Photos | Mason/Caleb profile + instructor; KnottyYoga, PartnerAcro, Handstand class photos; tier icons | All 13 subjects freshly shot (Phase 1), attached in Phase 13 |
| Database connection loss | The server never reconnects: one dropped connection takes the site down until a restart, while `/api/health` still says ok (the 10/15 incident) | Reconnects on its own; health probes the connection requests use (Phase 4) |
| Scheduled jobs | Every job refused with 403 by the CSRF guard since the 10/8 deploy | Jobs run (Phase 4) |

**Facts the plan depends on:**

1. ✅ **Fixed in Phase 2 (10/15/2026)**: every seeder now resolves by natural key. What it said before: **Seed ids are still hardcoded in four places.** `PopulateProductVariants`, `PopulateProductPrices` and `PopulateProductBookingWindows` (all still in `create_database.cpp`) address products 1–9 and variants 1–7 by literal id. `PopulateRoleAssignments` addresses people 1, 2 and 4. Removing the four spa products and adding two Silver ones renumbers everything after them, so **these have to move to natural keys before any catalog change.** That is the same failure mode as the Gold→instructor permission bug found in [[Database helper initialization]] Phase 3.2, and it is still listed there as an open follow-up.
2. **The access gate already exists and is the right tool.** A class's `class_requirement_groups` are AND-ed and each group's literals are OR-ed. A literal is a permission (closure-expanded) **or** a skill level (`ClassAccessHelper::CheckAccess`). "Silver or Gold" is one group with two literals. "Instructor approval" is a skill-level literal, which is exactly what skill levels are for: "Can kick up to handstand against a wall" is the schema's own example.
3. **"Requires attending the preceding class" already exists**: `class_schedule_slots.predecessor_class_schedule_slot_id` (SL-11), seeded today for Handstands. Four slots need it now.
4. ✅ **Fixed in Phase 10 (10/18/2026)**: a slot now carries co-instructors beside its lead, and Tuesday is Gifford's and Tiana's. What it said before: **A slot has exactly ONE instructor** (`class_schedule_slots.instructor_person_id`). Tuesday's Beginner Knotty Yoga has two (Gifford **and** Tiana). Per-session `event_session_staffing` can carry several, but only for materialized sessions, and only substitutions and shift trades write it. **Two instructors on a recurring slot is not expressible today**, which is why Phase 10 exists.
5. **Nothing extends provider availability over time.** `ScheduleTemplateHelper::GenerateAvailability` is called only by the seed and by the admin **Generate Availability** button. The seed generates through the end of the following month. **Per OQ-8 that stays manual**: once a month someone presses Generate Availability on Caleb's template (15.4). The generator skips days that already exist, so overlapping ranges are harmless. It also ignores `effective_to_us`, so an open-ended template keeps working.
6. **Production does not pick up seed changes.** Prod was installed with `--install_schema` on 9/23, and every later deploy runs `--migrate`, which evolves the schema and never re-seeds. Phase 15 is how this content reaches the live site.
7. **The test-only seed data collides with the real schedule.** The Intro Workshop (Sat 10–11) sits inside Saturday Knotty Yoga, the Aerial Series (Tue/Thu 19:00) sits on Thursday Intermediate Partner Acro, and Caleb's Mon–Fri 9–5 template duplicates his real hours. **Per OQ-12 all of it goes** (Phase 9). Caleb's template is promoted to real data with his real hours (Phase 7). After that `PopulateTestData` is empty, so the flag goes too (D14).
8. **`ui/src/app/pages/public/services/homepage.service.ts` is dead code.** Only its own spec imports it, and it hardcodes the old six-class list. It gets deleted rather than updated.
9. **Customers never see a requirement group's `inline_description`.** The admin field is labelled "Description (shown to members)", but no customer template renders it. A visitor sees only **"Members only — Upgrade your membership to attend."** on the class page, or the **"Requires a membership"** badge on Our Classes. Nothing tells them *which* membership. The skill side does render: the class page's **"Required Skills"** section lists each skill's name and description, and the **"Skill required"** dialog shows the same. That is why the approval criteria go into the skill descriptions (D6). Phase 12 closes the membership half.
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
    - **Forgot password (Phase 11) is therefore built in both honuware repos and then pinned in here.**
    - Honuware has exactly one framework migration (`honuware/0001_site_assets`). This app's migrations ran through `app/0009_about_sections`; Phase 3 added `app/0010_products_sort_order`.
13. ✅ **Fixed in Phase 3 (10/15/2026)**: products now list by `sort_order`, then name. What it said before: **Product order is name order, everywhere.** All three `TableHelpers::Products` listings `ORDER BY name, id`, `CatalogHelper` keeps that order, and the membership hosts only filter `kind === 'subscription'`. Grouping Silver before Gold (OQ-13) therefore needs a column, not a front-end sort.

---

# Design decisions

- **D1 — This is SEED data, built on mechanisms that already exist.** Classes, slots, predecessors, requirement groups, skill levels, entitlement rules, provider templates and photos all have tested tables and helpers. There are three new *mechanisms*:
  - product sort order (Phase 3, OQ-13)
  - co-instructors (Phase 10, fact 4)
  - forgot password (Phase 11, OQ-2)

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
- **D9 — New people never get a password from source control.** Beth, Gifford and Tiana are created with a random, unrecorded password, generated at seed time and never logged. To let one of them in, Mason sets a **temporary password with the must-change flag** (Phase 5), and they choose their own at first sign-in. Once Phase 11 ships, **Forgot password?** works too, given a real email (OQ-1). The four existing accounts keep their committed dev passwords; Mason, Caleb and Kit change theirs on production at deploy time (OQ-10, 15.3).
- **D10 — Photos come first, and the code never waits on them.** Mason shoots them in Phase 1, and Claude converts them and attaches them in Phase 13, just before Verification. `AttachSeedPhoto` already logs and skips a missing file, so one late picture blocks nothing: it lands later as file + table row + test expectation in one change.
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

*→ 10/20 (Phase 13): all five bios below are in the seed (`kSeedInstructors` in `seed_app_data.h`), replacing the defaults. Six slips were corrected on the way in, and the text below is left as pasted. The corrections are listed in **OQ-28**; say if any should go back.*

### Mason Bendixen
Default (today's seed): "Teaches Knotty Yoga, handstands and partner acrobatics."

Mason founded Knotty Yoga in 2011 and had been teaching acrobatics since 2004. Mason completed the Ana Forrest yoga teacher training in 2006 and attended Bellevue Massage School in 2011 before training at three massage schools in Thailand. Mason has completed a TRX instructor training and many acrobatics teacher training. Mason has a degree in Computer Science from the Univesrity of Utah and has worked as a software engineer at Microsoft, Tableau, Google, and Meta.

### Caleb Ault
Default (today's seed): "Teaches Knotty Yoga and aerial work."

Caleb has attended Knotty Yoga as a student since 2016 before assisting and teaching classes. Caleb became a licensed massage therapist at Bellevue Massage School in 2023 and has extensive training in Thai massage, cupping, and trigger point therapy. He is passionate about helping people eliminate pain and achieve their fitness goals.

### Beth Cooper
Default: "Teaches Beginner Knotty Yoga on Wednesdays."

Beth Cooper has 30 years of experience as a licensed physical therapist and is the owner of a solo practice: The Body Creative, PLLC.  She is a certified teacher of the Feldenkrais Method® of Movement Education. She also has an extensive background as a performing artist and a student of many things movement-related. Beth came to the studio for intro workshop over a dozen years ago and it was love at first class! Beth’s classes emphasize increasing your awareness of unconscious movement habits and improving your self-organization to maximize success and decrease risk of injury. 

### Gifford Cheung
Default: "Co-teaches Beginner Knotty Yoga on Tuesdays with Tiana Cheung."

Tiana has been attending Knotty Yoga since 2012 and has been teaching since 2014. Gifford has a PhD from the University of Washington in computer science and works at Nintendo by day.

### Tiana Cheung
Default: "Co-teaches Beginner Knotty Yoga on Tuesdays with Gifford Cheung."

Tiana has been attending Knotty Yoga since 2008 and has been teaching since 2014.

---

# Phase 1 — Take the photos (Mason)

The later phases use these: Phase 13 attaches them and Phase 14's walkthrough looks for them. Nothing else blocks on them (D10), so the phases below can proceed while the shoot happens.

### 1.1 Shot list

How the site crops each kind, so the subject survives the crop:
- **Headshots:** head and shoulders, centred, with room on every side. Instructor cards crop to fill; Our Providers crops square.
- **Class photos:** landscape, with the action in the middle. The class page crops to 4:3.
- **Studio:** wide landscape. About ▸ Our Location crops to 16:10.
- **Phone setting:** JPEG, not HEIC. On an iPhone that is Settings ▸ Camera ▸ Formats ▸ **Most Compatible**.

- [x] 1. Mason Bendixen headshot → `MasonInstructor.jpg` (replaces today's)
- [x] 2. Caleb Ault headshot → `CalebInstructor.jpg` (replaces today's)
- [x] 3. Beth Cooper headshot → `BethInstructor.jpg`
- [x] 4. Tiana Cheung headshot → `TianaInstructor.jpg`
- [x] 5. Gifford Cheung headshot → `GiffordInstructor.jpg`
- [ ] 6. Beginner Knotty Yoga class → `BeginnerKnottyYoga.jpg` *(not in yet)*
- [ ] 7. Knotty Yoga class → `KnottyYoga.jpg` (replaces today's, if reshot) *(not reshot; today's stays)*
- [ ] 8. Aerial Rope class → `AerialRope.jpg` *(not in yet)*
- [ ] 9. Handstands class → `Handstand.jpg` (replaces today's, if reshot) *(not reshot; today's stays)*
- [ ] 10. Beginner Partner Acro class → `PartnerAcro.jpg` (replaces today's, if reshot) *(not reshot; today's stays)*
- [x] 11. Intermediate Partner Acro class → `IntermediatePartnerAcro.jpg`
- [ ] 12. Massage, shown on the Services card for Massage → `Massage.jpg` *(not in yet)*
- [ ] 13. General studio, shown on About ▸ Our Location → `Studio.jpg` *(not in yet)*

### 1.2 Hand-off and conversion
- [x] Mason drops the originals into **`C:\Users\mason\Documents\Obsidian\Knotty Yoga\Photos\`** (OQ-11 default), named as in 1.1. Any extension and any size is fine. That folder is outside the git repo, so phone-sized originals never get committed by accident.
  - → 10/20: they came in at `D:\Knotty Yoga\2026\Oct\JPG\` instead: six JPEGs, named as in 1.1. Only the six JPEGs were used; the five `.heic` files in the folder above it were left alone.
- [x] Claude converts each into `server/knottyyoga_server/src/database_helper/img/` with **ffmpeg**:
  - JPEG, long edge 2000 px, `-q:v 3`, under about 1.5 MB.
  - Phone rotation applied.
  - **All metadata stripped** (`-map_metadata -1`), because phone photos carry GPS coordinates.
  - Never `convert`: on Windows that resolves to the FAT-to-NTFS disk converter.
  - → 10/20: done, and none of the six carried GPS or a rotation flag to begin with. The results: the four phone headshots are 1500 × 2000 (110–175 KB), Beth's is 576 × 720 (127 KB; it came small and was not enlarged), and Intermediate Partner Acro is 2000 × 1818 (165 KB). What is left in each file is ffmpeg's encoder note and, in Beth's, her photographer's colour profile.
- [x] Claude opens every converted file to check orientation and framing against 1.1, and lists anything that needs a reshoot in this phase before Phase 13 wires it.
  - → 10/20: all six are upright, sharp and well lit. **No reshoots.** Two framing notes:
    - **The four phone headshots are waist-up, not head-and-shoulders.** Every instructor picture is shown in a square cropped from the middle, which cut the top of the hair off Beth, Caleb, Gifford and Tiana. Rather than reshoot, those squares now crop from the top (13.3), which frames all five from the head down.
    - **Intermediate Partner Acro is nearly square.** The class page frames pictures 4:3, so it trims a little off the top and bottom: the raised foot's toes and the bases' heels. Acceptable as is; a wider shot would avoid it if this one is ever retaken.
  - → 10/20, evening: **Ryan's edited headshots** for Mason, Caleb, Gifford and Tiana (`D:\Knotty Yoga\2026\Oct\JPG\Edited\`) replace those four. They are tighter crops on a cleaner grey, 1401–1536 × 1771–1864 px and 111–175 KB each. They already carry no EXIF or GPS, so they were copied **byte for byte**, not re-encoded, which would only have lost quality. They are progressive JPEGs, which the server reads fine: it took all four and served them back at their true sizes. Beth's is unchanged.
    - The live local site was updated as an admin would do it: the server's own photo upload, onto the four instructor records, with no database recreate. Each was then fetched back signed out at the Instructors card's size.
    - The seed photo tests (9) pass against the new files on Windows and Linux.
    - **Trap found on the way:** the `img/` copies beside the database helper and the test program are made by a CMake POST_BUILD step, which runs only when that program relinks. Replacing a picture relinks nothing, so both copies kept the old pictures, and a Windows `--recreate_database` would have seeded those. Both copies were refreshed by hand on both platforms. Production is unaffected: the release script copies straight from the source folder.

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

All the code is in `server_components`; knottyyoga only moves its pin. During development knottyyoga builds against the local checkout (`FETCHCONTENT_SOURCE_DIR_HONUWARE`), as Phase 11 does. This phase ships to production **on its own release (4.6), ahead of everything after it**: it is a fix to what is live today.

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
- [x] `PopulateInstructors`: five rows, data-driven, with bios taken **verbatim** from the "Instructor bios" section above, or each name's default where nothing was pasted (OQ-15). The photo column moves to Phase 13's table.
  - No bios had been pasted, so all five are the defaults. `Seed::kSeedInstructors` in `seed_app_data.h` is the one table; the test walks it too (D3).
  - Mason and Caleb keep their headshots. Beth, Gifford and Tiana have an empty photo entry, which is skipped without a "seed_image_missing" warning. Phase 13 attaches their pictures and moves all five into its photo table.

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
   - The three new cards have no photo yet (Phase 13).

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

**Implemented 10/17/2026; committed as `32c9235c`, not tagged.** Windows 5259 passed, Linux 5259 passed. The two runs executed the same 5259 test names: Phase 5's 5252, less the three per-class slot tests, plus ten new. The Linux database helper, server and test helper all build, and the log says `honuware : pinned SHA`. Like the rest of the seed, it reaches production with the Phase 15 reinstall (fact 6).

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
- [x] **Also changed: the three existing class photos.** `PopulateSeedPhotos` named "Partner Acrobatics", which no longer exists. Its picture now goes on **Beginner Partner Acro**, and Knotty Yoga and Handstands keep theirs. The other three classes have no photo until Phase 13.

### 6.3 Access rules *(seed)*
- [x] New `Seed::PopulateClassRequirements`, data-driven from the Class access table:
  - `ClassRequirementGroups::AddGroup` with label "Membership" or "Instructor approval", and `inline_description` as in the table. Customers see it once Phase 12 lands; until then only admins do (fact 9).
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
- [x] Tuesday Beginner Knotty Yoga gets Gifford as `instructor_person_id`. Tiana is added in Phase 10.
- [x] **Known, until Phase 9:** the test-only seed (`--recreate_database_test`) still adds the Aerial Series at Tue + Thu 19:00 in the Main Gym, which now double-books Thursday's Intermediate Partner Acro (fact 7). Phase 9 deletes it, so it was not moved. Plain `--recreate_database` is unaffected.

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
- [x] `NoTwoSlotsCollideInTheSameRoom` over the 14 slots, using half-open intervals so back-to-back classes are legal. It replaces `NoTwoSlotsCollideInTheSameRoomAfterTheFullTestSeed`, which included the test-only seed (6.4, known until Phase 9).
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

   There are no Therapeutic Knotty Yoga, Partner Acrobatics, Tumbling or Aerial Fabric cards. Knotty Yoga, Handstands and Beginner Partner Acro have a photo; the other three have none until Phase 13.
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
4. Known, not a fault of this phase: signed in, **Our Classes** → **All Classes** shows Kit no cards at all, because it lists only classes he may already attend (OQ-22, fixed in 12.3).

---

# Phase 7 — Caleb as the massage provider

**Implemented 10/17/2026; committed as `74af2aa8`, not tagged.** Windows 5267 passed, Linux 5267 passed. The two runs executed the same 5267 test names: Phase 6's 5259, less the two 9–5 template tests, plus ten new. The first attempt at the full runs was stopped part-way when the machine ran low on memory, so they were run again one at a time. Like the rest of the seed, it reaches production with the Phase 15 reinstall; 7.4's fix ships with the next release of the server.

The hours are one table, `kSeedMassageHours` in `seed_app_data.h`, walked by the seeder and by its tests (D3).

### 7.1 Provider assignment *(seed)*
- [x] `PopulateProviderTypeAssignments`: Caleb only, `massage_therapist`, accepting bookings. Mason's assignment is dropped (OQ-6, confirmed).

### 7.2 Weekly hours *(seed)*
- [x] **Promote** the test-only `PopulateProviderSchedule` into the base seed as `Seed::PopulateProviderSchedules(…, nowUs)`. It still resolves Caleb and the studio by natural key and generates through `GenerateAvailability`. What changed:
  - The template is named **"Massage hours"**.
  - It is **open-ended** (`effective_to_us` NULL). Checked first: nothing reads a provider template's dates. The generator ignores them, and the Templates page neither shows them nor sends real ones (it creates templates with 0 and 0). NULL only says "no end".
  - It carries the five ISO entries above instead of Mon–Fri 9–5.
  - **Changed from the plan:** the ISO constants (`kIsoMonday` … `kIsoSunday = 7`, with the warning comment) sit in `seed_app_data.h` beside the table rather than inside the function, because the tests walk the same table.
- [x] Availability is generated from today's local midnight through `Scheduling::EndOfFollowingMonthUs(nowUs)` in the studio facility. Beyond that it is extended by hand each month (15.4), which 7.4 makes safe to repeat.
- [x] `PopulateTables` reads `now_us()` once and passes it in. It runs last, after `PopulateServiceProductLinks`, so the massage product is bookable before its provider has hours.
- [x] `PopulateTestData` no longer adds a template; the old 9–5 one and `PopulateProviderSchedule` are gone.

### 7.3 Tests
All in `seed_app_data_test.cpp` unless named otherwise.
- [x] `PopulateProviderTypeAssignmentsMakesCalebTheOnlyMassageTherapist` (replaces `…MakesBothMassageTherapists`), read through **About ▸ Our Providers**' reader (`ProviderDirectoryHelper`) and the assignment readers booking uses: Caleb alone, and **Mason holds no provider assignment**. `…SkipsWithoutTheProviderType` now checks Caleb.
- [x] `PopulateProviderSchedulesWritesTheOpenEndedMassageHoursTemplate` (replaces the 9–5 template test), read the way **Templates** reads it: one template, "Massage hours", open-ended, its five entries matched by day. Spelled out as well: ISO days {1, 2, 4, 5, 7}, and Sunday 13:30–16:00.
- [x] **Through the booking-slot reader** (`ServiceAvailabilityHelper::ComputeAvailableSlots`, behind **Book Now** and `list_available_slots`), `…OffersMassageOnlyInCalebsHours`. The plan said "a fixed Monday"; the reader drops anything already started by the database clock, so the test takes the seven days from tomorrow instead.
  - Every slot is Caleb's, in the studio, inside that day's hours, and the first starts when he opens.
  - Spelled out: Monday's first massage is 11:00 and Sunday's 13:30 (which pins ISO 7).
  - Wednesday and Saturday offer none.
- [x] `…GeneratesCalebsDaysThroughTheEndOfNextMonth` (replaces `ProviderAvailabilityIsGeneratedOnWeekdaysOnly`): every generated row is Caleb's, `source = template`, in the studio, at local midnight, on one of his days at that day's hours. The set of dates equals every such local date from today through the end of next month, worked out independently of the generator.
- [x] **Caleb's massage hours never overlap a class he teaches** (`…NeverOverlapsAClassCalebTeaches`). His classes come from the calendar reader, his hours from the availability reader, across the whole generated window. It also checks he teaches Sunday, Monday and Thursday and has hours on each of those days, so the comparison is never empty.
- [x] `PopulateProviderSchedulesSkipsWithoutCaleb`.
- [x] `OnlyPopulateTestDataAddsTheSampleData`: Caleb's template now exists after the base seed, and the test-only seed adds no second one.
- [x] **Mutation-checked on Linux.** Seven deliberate breaks, each caught:

  | Break | Failed |
  |---|---|
  | Mason added back alongside Caleb | `…MakesCalebTheOnlyMassageTherapist` |
  | Sunday written the class-slot way (0) | `…WritesTheOpenEnded…`, `…OffersMassageOnly…`, `…NeverOverlaps…` |
  | Monday runs to 17:30, into his 17:00 class | `…NeverOverlapsAClassCalebTeaches` |
  | The template given an end date again | `…WritesTheOpenEndedMassageHoursTemplate` |
  | Only two weeks generated | `…GeneratesCalebsDaysThroughTheEndOfNextMonth` |
  | Sunday opens at 13:00 | `…WritesTheOpenEnded…`, `…OffersMassageOnly…` |
  | Thursday dropped | `…WritesTheOpenEnded…`, `…OffersMassageOnly…`, `…NeverOverlaps…` |

  The sources were restored byte for byte, with hashes checked in the container and again from Windows. Leaving the call out of `PopulateTables` is the one break no unit test can see, since nothing can run `PopulateTables`; the end-to-end run covers it.

### 7.4 Generate Availability is safe to press twice *(business_logic/scheduling)* — added
Found while checking 15.4. `ScheduleTemplateHelper::GenerateAvailability` skipped only days with hand-entered or blocked hours. A second press over days the template had already generated added a second copy of them. 15.4's monthly press starts from today, so it always overlaps the last one: following it would have doubled every day already generated, and **Book Now** would have offered every time twice. 15.4 also said pressing twice was harmless.
- [x] A day **this** template already generated is now skipped and counted in `days_skipped`, so the Templates page's "Skipped N days (already existed)" is true. Hours from a different template on the same day are still added, since that is a split shift. The check reuses `ProviderAvailability::GetAvailabilityForProviderOnDate`, with no new SQL.
- [x] Tests:
  - `schedule_template_helper_test.cpp`: `GenerateAvailabilityTwiceAddsNothingTheSecondTime`, `…OverAnOverlappingRangeAddsOnlyTheNewDays` and `…StillAddsAnotherTemplatesHoursToTheSameDay`.
  - `admin_schedule_templates_test.cpp`: `GenerateAvailabilityPressedTwiceSkipsTheGeneratedDays`, through the endpoint the Generate button calls.
  - `seed_app_data_test.cpp`: `MonthlyGeneratePressExtendsCalebsHoursWithoutDuplicates`, which runs 15.4's press on the seeded template, from today through the end of the month after next.
- [x] **Mutation-checked on Linux:**
  - With the new check removed (the old behaviour), four tests fail: the two "twice" tests, the overlapping-range test and the monthly-press test.
  - With the check counting **any** template's hours, `…StillAddsAnotherTemplatesHoursToTheSameDay` fails.
  - The source was restored and its hash matches.

### 7.5 End to end
- [x] On a throwaway database on the local Postgres (`phase7_check`, dropped afterwards; never `knottyyoga`), `knottyyoga_database_helper --install_schema` ran the real bootstrap and the real server answered:
  - The install log has no seed warning, and reads `provider_availability_generated days=32 skipped=0`. This also shows `PopulateTables` calls the new seeder, which no unit test can see.
  - **Our Providers**: Caleb Ault, Massage Therapist, alone.
  - Caleb's templates: one, "Massage hours", no end date, Mon/Tue/Thu/Fri 11:00 to 15:00 or 16:00 and Sun 13:30–16:00.
  - **Book Now**'s slots for the next eight days (Sun 10/18 – Sun 10/25): only Caleb, only on Sunday, Monday, Tuesday, Thursday and Friday. Sunday's first start is 1:30 PM and the others' 11:00 AM. Monday and Friday's last massage ends 2:50 PM, Tuesday and Thursday's 4:00 PM, Sunday's 3:40 PM.
  - Generate pressed twice through the real endpoint, from now through 70 days out: the first press `days_generated 18, days_skipped 32`, the second `0` and `50`. The booking slots before and after: 60 each, none offered twice.
- [x] **Found: Edit Template wipes the generated hours.** The Templates page's **Edit Template** saves by deleting the template and creating a new one. The delete cascades: the template's entries **and every availability row generated from it** go with it. In the run above, all 32 days were gone afterwards. So after editing Caleb's hours, **Book Now** offers no massage until **Generate Availability** is pressed for the new template. Bookings already made are separate rows and are not touched. This is how the page has always worked; Phase 7 only makes it matter. OQ-23 has the default: a note in 15.4, and no code change.

### 7.6 Hand-testing walkthrough
From a fresh local database:
1. Recreate it with `knottyyoga_database_helper --recreate_database`.
2. Start the server and the Angular app. Use a window at least 1280 px wide, so the top menu isn't collapsed.

**Signed out: one massage therapist**
1. Top nav **About** → **Our Providers**: one card, **Caleb Ault**, "Massage Therapist". There is no Mason Bendixen card.

**A customer books: only Caleb's hours**
1. **Sign In** as `kiteagle@gmail.com` / `kit`.
2. Top nav **Services** (page headed **Our Services**) → **Massage** → **Book Now**. The page is headed **Massage**, with a strip of days under it. If a day below isn't in the strip, use the **›** arrow (Next week).
3. **Wed** and **Sat** are greyed out and can't be clicked: Caleb has no hours on either day.
4. Click the next **Mon**. One card, **Caleb Ault**, with three closed hour headers: **11:00 AM**, **12:00 PM** and **1:00 PM**. Click **11:00 AM** to open it: **11:00 AM** offers **60-Minute Massage**, **90-Minute Massage** and **120-Minute Massage**. Open **1:00 PM**: the last time is **1:20 PM**, so nothing runs past 3:00 PM.
5. Click the next **Tue**. The last hour header is **2:00 PM**. Open it: **2:40 PM** offers only the **60-Minute Massage**, because the day ends at 4:00 PM.
6. Click the next **Sun**. The hour headers are **1:00 PM** and **2:00 PM**. Open **1:00 PM**: its first time is **1:30 PM**. Open **2:00 PM**: **2:40 PM** offers only the **60-Minute Massage**.

**As the administrator: Caleb's template**
1. Sign out: **Kit** (the name menu) → **Sign Out**, then **Log Out** in the dialog. **Sign In** as `masonbendixen@gmail.com` / `pass`.
2. **Admin** → **Manage Products** → **Service Providers**: one card, **Caleb Ault**, with the **Massage Therapist** badge and **Accepting bookings** on. There is no Mason Bendixen card.
3. Click **Templates** (page headed **Schedule Templates**). One card, **Massage hours**, with five chips:
   - "Monday: 11:00 AM – 3:00 PM"
   - "Tuesday: 11:00 AM – 4:00 PM"
   - "Thursday: 11:00 AM – 4:00 PM"
   - "Friday: 11:00 AM – 3:00 PM"
   - "Sunday: 1:30 PM – 4:00 PM"

**Generate is safe to press twice (7.4)**
1. Same card → **Generate Availability**. Set:
   - **From Date**: today
   - **To Date**: the last day of next month
   - **Facility**: Knotty Yoga Studio
2. Click **Generate**. It reads "Generated 0 availability blocks. Skipped N days (already existed)." The seed already generated those days.
3. Change **To Date** to the last day of the month after next and click **Generate** again. This time it generates some blocks and skips the rest.
4. Sign back in as Kit and open **Services** → **Massage** → **Book Now** → the next **Mon**. Open each hour header: every time is listed once, as in step 4 of the customer section.

---

# Phase 8 — About page copy, the location's text and the sender name

**Implemented 10/18/2026; validated by hand and committed as `f12a4c69`.** Windows 5268 passed, Linux 5268 passed. The two runs executed the same 5268 test names: Phase 7's 5267 plus one new. Like the rest of the seed, it reaches production with the Phase 15 reinstall. That includes the sender name: it is a config-secret default, written once when a database is created, so the live site keeps "Knotty Yoga and Spa" until then.

### 8.1 Seed
- [x] `PopulateAboutSections`: the three bodies from the About page table, **verbatim**. Titles, ordinals and links unchanged. The "40 years … hootananny" and "climbing wall" placeholder copy is gone.
  - **Changed from the plan:** the blocks moved out of the function into one table, `kSeedAboutBlocks` in `seed_app_data.h`, which the seeder and its test both walk (D3), like the massage hours.
- [x] `PopulateFacilities`: the studio's `description` becomes the **About ▸ Our Location** text above (OQ-16). It is a named constant, `kStudioDescription` in `seed_app_data.h`, so the test reads the same value the seeder writes.
- [x] `kMailSenderNameValue` becomes **"Knotty Yoga"** (OQ-16), and moved from `app_secret_values.cpp` into `app_secret_values.h`. It is also the site's **studio name** (`Mail::LoadTenantBranding` sets `studioName = senderName`). So the same change fixes the footer's "© Copyright Knotty Yoga and Spa 2020" and email subjects such as "Welcome to Knotty Yoga and Spa - Your Account". The "2020" is hard-coded in `footer.component.html`, and nothing here changes it.
  - Eight tests asserted the literal "Knotty Yoga and Spa": `app_secret_values_test`, `attendance_template_helper_test` (twice), `class_series_helper_test`, `favorite_instructor_helper_test`, `weekly_digest_helper_test`, `card_expiring_notification_test` and `signup_reminder_helper_test`.
  - **Checked:** `class_checkin_helper_test`'s subject is built from the sender name. honuware's `QuickAccountHelper` writes `"Welcome to " + branding.studioName + " - Your Account"`; there is no separate site-name slot. That test now builds the expected subject from the constant too.
- [x] Not changed:
  - `ServerAccess.mock.ts`'s local-mode copies of the old location text and the old About blocks. They are fixture data, as the note closing Phase 12 says of the mock catalog.
  - The `site_about_markdown` default in `app_secret_values.cpp`, which still carries the "40 years … hootananny" text. Nothing on this site has displayed it since Polish 7.3 replaced it with the About blocks. It still rides along unseen in the site-settings data the browser loads, and in theme bundles, which Phase 15 doesn't use (OQ-9). Say if you want it changed anyway.

### 8.2 Tests
- [x] `PopulateAboutSectionsSeedsMasonsCopyInPageOrder` (new, `seed_app_data_test.cpp`), through the public About reader (the one behind `/api/about_sections`):
  - three blocks, in the page's order (Who we are, How we teach, Come and find us);
  - every field equal to its `kSeedAboutBlocks` row;
  - spelled out as well, so a wrong table can't pass by agreeing with itself: how each body opens, the **Get started** label and the `/location` link;
  - no block containing "hootananny", "40 years" or "climbing wall".
  
  The older `…SeedsBlocksTheAboutPageCanRender` keeps its general checks. Its "40 years" line is gone.
- [x] `PopulateFacilitiesGivesTheStudioADescriptionAndTimezone`, through the location reader the public page uses (`TableHelpers::Facilities::GetActiveFacilities`): the description equals `kStudioDescription`. Spelled out: it opens "Our home gym: a full acrobatics floor with crash mats", contains "We're in the last building (14)" and ends "Space 2545 / Knotty Yoga.". It contains neither "climbing wall" nor "Street parking".
- [x] Sender name: `AppSecretValuesTest.RegistersBrandDefaultsForFrameworkKeys` spells out "Knotty Yoga", the one place the value itself is pinned. The eight From-name checks in real messages, and the welcome-email subject, compare against `App::kMailSenderNameValue`.
- [x] **Mutation-checked on Linux.** Eight deliberate breaks, each caught:

  | Break | Failed |
  |---|---|
  | Who we are carries the old "40 years … hootananny" copy | `…SeedsMasonsCopyInPageOrder` |
  | The seeder writes the old location text instead of the constant | `PopulateFacilitiesGivesTheStudio…` |
  | The location text says "We in", as typed | `PopulateFacilitiesGivesTheStudio…` |
  | The sender name keeps the spa | `RegistersBrandDefaultsForFrameworkKeys` |
  | The defaults register "Knotty Yoga and Spa" instead of the constant | `RegistersBrandDefaultsForFrameworkKeys`, plus all eight message tests (the welcome subject among them) |
  | How we teach moves to the end of the page | `…SeedsMasonsCopyInPageOrder` |
  | The seeder skips Come and find us | `…SeedsMasonsCopyInPageOrder` |
  | The first block's button reads "Start here" | `…SeedsMasonsCopyInPageOrder` |

  The fourth row is why the value is spelled out once: the message tests follow the constant, so only that test notices the constant itself is wrong. The sources were restored byte for byte, with hashes checked in the container and the broken lines searched for from Windows.

### 8.3 End to end
- [x] On a throwaway database on the local Postgres (`phase8_check`, dropped afterwards; never `knottyyoga`), `knottyyoga_database_helper --install_schema` ran the real bootstrap with no seed warnings. The real server then answered, and a script compared its answers with **this document**, character for character:
  - **About**: three blocks in ordinal order. Each title, body and button label matches its row in the About page table.
  - **Our Location**: one location, whose text matches the **About ▸ Our Location** paragraph.
  - None of the old copy appears: no hootananny, "40 years", climbing wall or "Street parking".
  - The site's name, as the footer and the email subjects read it, is **Knotty Yoga**.

  The first attempt stopped at the install: `--install_schema` expects the database to exist already, so it was created first and the check rerun.

### 8.4 Hand-testing walkthrough
From a fresh local database:
1. Recreate it with `knottyyoga_database_helper --recreate_database`.
2. Start the server and the Angular app. Use a window at least 1280 px wide, so the top menu isn't collapsed. Nothing here needs signing in.

**The About page**
1. Top nav **About** → **About Knotty Yoga**. Three blocks, top to bottom:
   - **Who we are**, beginning "Mason started teaching circus acrobatics in 2004", with a **Get started** button.
   - **How we teach**, beginning "There is no magic to fitness: it's just consistency with intensity with technique coaching.", with a **See the class schedule** button.
   - **Come and find us**, beginning "We have inch thick padded floors throughout the whole studio", with an **Our location** button.
2. Nothing on the page mentions forty years, a hootananny or a climbing wall.

**Our Location**
1. Click **Our location** under **Come and find us**. The page is headed **Our Location**, with one card, **Knotty Yoga Studio**, at 2545 152nd Ave NE, Redmond.
2. Its text begins "Our home gym: a full acrobatics floor with crash mats, aerial rigging, 24 fabrics, 9 ropes" and ends "We're in the last building (14) before the parking lot bends to the right. Space 2545 / Knotty Yoga." No climbing wall, no "Street parking is free after 6pm".

**The site's name**
1. Scroll to the bottom of any page. The footer reads "© Copyright Knotty Yoga 2020 all rights reserved", without "and Spa". The "2020" is written into the page and is unchanged.
2. The email sender is checked in Phase 14.3, step 26, which needs real mail.

---

# Phase 9 — Retire the test-only seed data (OQ-12, D14)

**Implemented 10/18/2026.** Windows 5260 passed, Linux 5260 passed. The two runs executed the same 5260 test names: Phase 8's 5268, less the nine test-data tests, plus one new. Production never had the test-only seed (it was installed with `--install_schema`), so nothing changes there.

### 9.1 Seed
- [x] Delete `PopulateIntroWorkshopSessions`, `PopulateAerialSeriesProduct`, `PopulateAerialSeriesRun` and `PopulateTestData` from `seed_app_data.{h,cpp}`. `PopulateProviderSchedule` already moved in 7.2. The `intro-workshop` **product** stays in the base catalog: staff schedule real workshop sessions through the UI, and the About copy invites people to the next one.
  - Their constants went with them: the Aerial Series' code, class name and run name in the header, and the workshop and series times, prices and capacity in the `.cpp`. So did the two includes only they used (`class_series_helper.h`, `db_schema/event_sessions.h`).
  - **Left in place:** `Scheduling::NextWeekdaysAfter` in `business_logic/scheduling/seed_date_util`. The workshop seeder was its only caller, so nothing in the app uses it now. It is a tested date utility, and removing it wasn't in this phase's list.

### 9.2 `knottyyoga_database_helper`
- [x] Remove the `--recreate_database_test` flag and its help text (`main.cpp`), and the `includeTestData` parameter of `CreateSchemaAndPopulate` (`create_database.{h,cpp}`).
  - `CreateAndPopulateDatabases` lost the same parameter, and `RunRecreate` no longer takes one.
  - `--create_tenant` passes `CreateSchemaAndPopulate` to the tenant provisioner directly again. It had been wrapped in a lambda only because the extra parameter stopped the function matching the provisioner's type.
- [x] Fix the stale reference in `ServerAccess.mock.ts`'s comment, and any doc that names the flag.
  - The comment now says the mock's two providers are fixture data from before Phase 7, not what a fresh database seeds. The data itself is unchanged.
  - No document in the repository names the flag. The CLAUDE.md files, `server/*.md` and the launch settings were all checked.
- [x] Update the hand-testing memory ([[feedback_test_instructions_format]]): the flag is gone and Kit is a base-seed member. It also now points at `seed_app_data.{h,cpp}` for the seed, not `create_database.cpp`.

### 9.3 Tests
- [x] Delete the test-data tests, including `OnlyPopulateTestDataAddsTheSampleData`. Nine went: four for the workshop sessions, two for the Aerial Series product, two for its run, and that one.
- [x] `TheBaseSeedHasNoSampleData` (new): runs the whole base seed in `PopulateTables`' order, then checks:
  - no event sessions at all, while the `intro-workshop` product is still there;
  - no `aerial-series` product, no `Aerial Series` class and no class series run;
  - exactly the six real classes;
  - one provider template, Caleb's.

  The names are spelled out as literals, since the constants went with the seeders.
- [x] **Mutation-checked on Linux.** Four deliberate breaks, each putting a piece of the sample data back into the base seed. `TheBaseSeedHasNoSampleData` caught every one:

  | Break | Failed |
  |---|---|
  | `PopulateClasses` adds an "Aerial Series" class | `TheBaseSeedHasNoSampleData`, `PopulateClassesSeedsTheSixStudioClasses` |
  | `PopulateProducts` adds an `aerial-series` product | `TheBaseSeedHasNoSampleData`, `PopulateProductsSeedsExactlyTheStudiosProducts` |
  | `PopulateClassSchedules` adds an event session | `TheBaseSeedHasNoSampleData`, `PopulateClassSchedulesSkipsWhenPrerequisitesAreMissing` |
  | `PopulateProviderSchedules` adds a Mon–Fri 9–5 template beside Caleb's | `TheBaseSeedHasNoSampleData`, plus two Phase 7 template tests |

  The source was restored byte for byte, with its hash checked in the container and the broken lines searched for from Windows.

### 9.4 End to end
- [x] With the Linux database helper built from these sources:
  - **The old flag is refused.** `--recreate_database_test` exits with "ERROR: Unknown command line flag 'recreate_database_test'". It was run with destructive operations not allowed and the database host pointed at a name that cannot resolve, so even a helper that still had the flag could not have touched a database.
  - **The usage text** (the helper run with no mode) lists `--recreate_database` and no longer lists `--recreate_database_test`.
- [x] `--install_schema` into a throwaway database on the local Postgres (`phase9_check`, dropped afterwards; never `knottyyoga`): no seed warnings, and no sample-data lines in the log. The real server then answered:
  - **Upcoming Events** and the home page's events band: no sessions.
  - **Our Classes**: the six real classes, no Aerial Series.
  - **The catalog**, signed in as Mason: no `aerial-series`; `intro-workshop` is there.

### 9.5 Hand-testing walkthrough
1. Run `knottyyoga_database_helper` with no options. The list of modes starts with `--recreate_database` and has no `--recreate_database_test`.
2. Recreate the database with `knottyyoga_database_helper --recreate_database`.
3. Start the server and the Angular app. Use a window at least 1280 px wide, so the top menu isn't collapsed. Stay signed out.

**No sample workshops or series**
1. On the home page, the **Upcoming events** section reads "No upcoming events at this time."
2. Top nav **Upcoming Events**. The page is headed **Upcoming Events** and reads "No upcoming events at this time." No Aerial Series appears anywhere on it.
3. Top nav **Our Classes**. **All Classes** shows the six real classes and no **Aerial Series** card.
4. Click **Weekly schedule**. Tuesday has one class, **6:00 PM Beginner Knotty Yoga**. Thursday's 7:00 PM row is **Intermediate Partner Acro** only, with no Aerial Series beside it.

---

# Phase 10 — Co-instructors on a recurring slot (Gifford + Tiana)

**Implemented 10/18/2026.** Windows 5306 passed, Linux 5306 passed. The two runs executed the same 5306 test names: Phase 9's 5260 plus 46 new. Angular: 3608 specs passed, `tsc --noEmit` (app and specs) and `ng build` clean.
- On Windows, the server binary itself didn't relink: a running `knottyyoga_the_server.exe` held the file open. Everything else built, the tests included. The Linux run built and ran the server (10.8).
- Production needs nothing beyond the Phase 15 reinstall. An upgraded database gets the table from `--migrate` (0011), which 10.8 checked.

Fact 4: a slot cannot carry two instructors today. OQ-7: "we should support multiple instructors."

Scope: an occurrence's **displayed** instructors become `staffing override if present, else slot lead + co-instructors`. These stay keyed on the **lead**:
- pay and specialty cost
- the instructor-load grid
- the exception-notes digest
- favorite-instructor alerts

That is the conservative reading, and each can widen later on its own.

### 10.1 Schema *(db_schema)*
- [x] New nested table `class_schedule_slot_co_instructors`:
  - `id`
  - `class_schedule_slot_id` (FK)
  - `person_id` (FK → people)
  - `created_us`, `updated_us`
  - UNIQUE(slot, person), and an index on slot
  - Both keys cascade, as every generated key does, so deleting a slot or a person takes their rows with it. Without that, **Delete slot** would fail on any co-taught slot.
  - The unique constraint's index leads with the slot, so it *is* the per-slot index; there is no second one.
- [x] The full new-table checklist (CLAUDE.md):
  - `make_app_tables`
  - `CreateTables`
  - the allowed, top-level and nested admin tables
  - column data info, friendly names and display template
  - table permission (`manage_class_schedule`)
  - CMakeLists

  The existing `create_database_test` checks (the admin metadata's own keys, and every table with column metadata on the allow list) now run over the new rows too.
- [x] Schema test, 5 cases: a valid row with its defaults, both keys refusing unknown ids, the same person twice on one slot refused (another person, or the same person on another slot, allowed), and deleting the slot removing only its own rows.
- [x] **Idempotent migration `app/0011_class_schedule_slot_co_instructors`** ([[gotcha_new_table_needs_migration]]), with a migration test.
  - It is guarded on the table list, like 0006 and 0009, so the DDL lives only in `db_schema`.
  - Unlike those tables, this one has foreign keys, and a key can only be described against a parent the schema object knows. So the migration builds the whole composed schema and creates just this table from it.
  - Test: drop the table, apply, write and read a row through the table helper, replay with the row intact, then delete the slot and see the row go (so the keys came with it).

### 10.2 Table helper *(sql_util/table_helpers)*
- [x] `TableHelpers::ClassScheduleSlotCoInstructors`: `Add`, `GetForSlot`, `GetForSlots` (batch) and `Delete`. Tests for each, including the unique constraint (6 cases).
  - Every read returns rows in the order they were added, each with the person's first and last name.
  - `GetForSlots` serves the admin slot list: one query for all of a schedule's slots. The pages that name instructors ask per occurrence, the way they already read staffing. That is fine at this studio's size.

### 10.3 Business logic *(business_logic/scheduling)*
- [x] One shared `EffectiveInstructorIds`: the staffing override if present, else lead + co-instructors, lead first. The instructor-profile helper already has a private version. Callers:
  - `ClassCatalogHelper` (class detail sessions)
  - `CalendarHelper`
  - `InstructorProfileHelper`, both "classes I teach" (whose SQL `UNION` gains the co-instructor table) and upcoming sessions

  It is `Scheduling::EffectiveInstructorHelper` (new file), with `SlotInstructorIds` (the slot's own team) beside it. The profile helper's private copy is gone. Two things differ from the sketch above:
  - **Only instructor and substitute staffing rows count.** An assistant row no longer replaces the team. That is the rule the substitution and shift-trade code already state ("assistants never headline a session"); before, three of these readers let an assistant row hide the instructor.
  - **A fourth caller, `AttendanceTemplateHelper`.** It feeds My Classes' today and upcoming lists, the weekly digest email and an instructor's teaching list. Without it, My Classes would have said "with Gifford Cheung" while the calendar said both. It also puts the Tuesday class on Tiana's teaching list.
- [x] `ClassScheduleHelper`: add or remove a co-instructor on a slot, refusing the lead themself and a person who is not an instructor.
  - Error codes: `INVALID_SLOT`, `NOT_AN_INSTRUCTOR`, `CO_INSTRUCTOR_IS_LEAD` and `DUPLICATE_CO_INSTRUCTOR`; remove adds `CO_INSTRUCTOR_NOT_FOUND`.
  - A duplicate is refused *before* the insert. A unique-key violation would abort the caller's whole transaction.
  - **Copy slots forward** (a new class schedule copied from another) now copies each slot's co-instructors. Without that, Tiana would silently drop off a copied schedule.
- [x] Substitutions name the whole team. "Substituting for" on the weekly schedule and My Classes now reads "Gifford Cheung, Tiana Cheung" rather than the lead alone.
- [x] Tests per caller:
  - a co-taught occurrence lists both names
  - a substitution still **replaces** both
  - Tiana's instructor profile lists Beginner Knotty Yoga
  - the instructor-load grid is unchanged, which pins the scope line above

  Where: the shared helper (7 cases, on real derived occurrences), the catalog (2), the calendar (2), the profile (2), My Classes and the teaching list (2), `ClassScheduleHelper` (8, including copy forward) and the load grid (1).
- **Not changed, deliberately:**
  - **Past attendance history** (My Classes ▸ history) still names the lead and any staffing, not co-instructors. It is its own SQL report and wasn't on this phase's list.
  - **Shift trades on a co-taught class.** See OQ-24: Mason kept the default (10/19), so this stays as is. An admin substitution is the workaround.

### 10.4 Endpoints
- [x] `POST /api/admin/class_schedule_slots/{id}/co_instructors` and `DELETE …/{personId}`, gated on `manage_class_schedule`, thin over 10.3.
  - As built: `POST /api/admin/class_schedule_slot/{id}/co_instructors` with `{ "person_id": N }`, and `DELETE /api/admin/class_schedule_slot/{id}/co_instructors/{personId}`. That is the singular `class_schedule_slot`, matching the slot's other routes.
  - A missing slot, or a person who isn't on it, is 404; the other refusals are 400. Both answer `{ ok, error_code }` like the other slot edits.
- [x] The slot list (`GET /api/admin/class_schedule/{id}/slots`) also returns `co_instructors`, every listed slot's rows with names, so the slot rows and the dialog can show them.
- [x] Endpoint tests: auth (401/403), happy path, duplicate, and lead-as-co-instructor refused. Also: not an instructor, a missing slot or co-instructor (404), no `person_id` (400), and the slot list carrying the right rows for the right slots.

### 10.5 Seed
- [x] Tuesday Beginner Knotty Yoga gets Tiana as co-instructor. The slot table gains an optional `coInstructorEmail`.
- [x] Test through `ClassCatalogHelper`: Tuesday sessions read "Gifford Cheung, Tiana Cheung".
  - The two per-slot seed tests (weekly schedule and class page) now expect each row's lead and co-instructor, both read from the seed table.
  - New: Tiana's instructor profile, through its reader, teaches Beginner Knotty Yoga and nothing else, and every session on it is a Tuesday naming both.

### 10.6 Frontend
- [x] `ServerAccess` + network + mock + proxy methods for add and remove; `ServerAccess.mock.spec.ts` cases (4). The slot list carries `co_instructors` in all three layers.
- [x] Slot form dialog (Manage ▸ class schedules): a **Co-instructors** chip picker beneath Instructor, reusing the instructor person picker. Manage Data is not a workflow ([[feedback_manage_data_is_debug_only]]).
  - The dialog only collects the list. After the slot itself saves, the page brings the co-instructors in line, removals first and then additions. So swapping Gifford and Tiana works in one save.
  - The instructor is never offered as a co-instructor. Making a listed co-instructor the instructor takes their chip away.
  - The slot's row on the page shows **Co-instructors** under **Instructor**.
- [x] Wherever a session renders one instructor name, render the list joined with ", ": class detail, calendar event and instructor profile.
  - The class page already listed every name, and the calendar gets its joined string from the server, so neither needed a change.
  - The instructor profile showed no names on its sessions at all. A co-taught session there now reads "With Gifford Cheung, Tiana Cheung"; one taught alone stays as it was.
- [x] Specs:
  - the dialog adds and removes a co-instructor
  - class detail renders two names
  - the lead cannot be picked as their own co-instructor

  The dialog has 5 new cases plus a rendered one (the chip's remove button), the manage page 6 (including the order of the calls and a rendered row), and the instructor profile 2. The class-detail case already existed.

### 10.7 Mutation check
- [x] **C++, on Linux.** Fourteen deliberate breaks, each rebuilt and run against the affected suites (289 tests). Every one was caught:

  | Break | Caught by |
  |---|---|
  | The seed leaves Tiana off Tuesday | the three seed tests: weekly schedule, class page, Tiana's profile |
  | The shared reader forgets co-instructors | 20 tests across every caller, the seed included |
  | A substitution no longer replaces the team | 10 tests, among them the older substitution tests |
  | An assistant headlines the session | `AnAssistantDoesNotReplaceTheTeam` |
  | The lead can be their own co-instructor | the helper test and the endpoint test |
  | Someone who isn't an instructor can be added | the helper test and the endpoint test |
  | Copying slots forward drops co-instructors | `CreateImplementationCopiesCoInstructorsForward` |
  | Removing a co-instructor removes nothing | the helper test and the endpoint test |
  | "Classes I teach" ignores co-taught slots | two profile tests and the seed's Tiana test |
  | The calendar names only the lead as substituted for | `SubstitutionReplacesTheLeadAndCoInstructor` |
  | The migration builds nothing | the migration test |
  | The slot list stops carrying co-instructors | three endpoint tests |
  | A lead also listed as a co-instructor is named twice | `ALeadAlsoListedAsCoInstructorIsNamedOnce` |
  | One slot's co-instructors come back newest first | the table helper's and the shared reader's order tests |

  The sources were restored byte for byte, with their hashes checked in the container and the broken lines searched for from Windows.
- [x] **Angular.** Three breaks, each run against its own spec and restored by hash:
  - the dialog offers the instructor as a co-instructor (3 dialog cases failed)
  - the page adds before it removes (the "removals first" case)
  - the profile shows a team line on a session taught alone (its case)

### 10.8 End to end
- [x] **A fresh install.** `--install_schema` into a throwaway database on the local Postgres (`phase10_check`, dropped afterwards; never `knottyyoga`), with no seed warnings. Then, from the real server:
  - **Weekly schedule:** every Tuesday Beginner Knotty Yoga reads "Gifford Cheung, Tiana Cheung"; Wednesday's reads "Beth Cooper".
  - **The class page:** every Tuesday session names both.
  - **Tiana's profile:** Beginner Knotty Yoga and nothing else; every session names both.
  - **The slot list (admin):** Tuesday's slot carries Tiana, Wednesday's nobody.
  - **The admin endpoints, signed in as Mason with a real CSRF token:**
    - Gifford on his own slot is refused with 400 `CO_INSTRUCTOR_IS_LEAD`, and Tiana twice with 400 `DUPLICATE_CO_INSTRUCTOR`.
    - Adding Tiana to Wednesday shows in the list, and removing her empties it again.
    - Signed out, the add is refused.
- [x] **An existing site.** The new table dropped from that database, as a site from before Phase 10 would be. `--migrate` then reported `applied=12 skipped=0`, 0011 among them, and the same add, list and remove worked through the endpoints.

### 10.9 Hand-testing walkthrough
1. Recreate the database with `knottyyoga_database_helper --recreate_database`.
2. Start the server and the Angular app. Use a window at least 1280 px wide, so the top menu isn't collapsed. Stay signed out.

**Tuesday is Gifford's and Tiana's**
1. Top nav **Our Classes** → **Weekly schedule**. Tuesday's **6:00 PM Beginner Knotty Yoga** reads **with Gifford Cheung, Tiana Cheung**. Wednesday's 5:00 PM Beginner Knotty Yoga still reads **with Beth Cooper**.
2. Click **Beginner Knotty Yoga**. Each Tuesday session in the list reads **With: Gifford Cheung, Tiana Cheung**.
3. **About** → **Instructors** → **Tiana Cheung**. **Classes** shows one chip, **Beginner Knotty Yoga**. Each session under **Upcoming sessions** is a Tuesday at 6:00 PM and reads **With Gifford Cheung, Tiana Cheung**.
4. Back to **Instructors** → **Gifford Cheung**: the same sessions, with the same line.

**Editing co-instructors**

5. **Sign In** as `masonbendixen@gmail.com` / `pass`. **Admin** → **Manage Products** → **Class Schedules** → **Class**: **Beginner Knotty Yoga**.
6. Expand **Perpetual**, then **Default schedule**, then the **Tue · 6:00 PM** slot. It reads **Instructor** Gifford Cheung and **Co-instructors** Tiana Cheung.
7. Click the slot's pencil (**Edit slot**). The **Edit class schedule slot** dialog shows **Instructor (optional)**: Gifford Cheung, and a **Tiana Cheung** chip above **Co-instructors (optional)**.
8. Click in **Co-instructors (optional)**. The list offers Mason Bendixen, Caleb Ault and Beth Cooper; Gifford and Tiana are not in it. Pick **Beth Cooper**: a Beth Cooper chip appears beside Tiana's. Click **Save**. The slot reads **Co-instructors** Tiana Cheung, Beth Cooper.
9. Top nav **Our Classes** → **Weekly schedule**: Tuesday reads **with Gifford Cheung, Tiana Cheung, Beth Cooper**.
10. Back on **Class Schedules** (steps 5–6), **Edit slot** again and click the ⊗ on the **Beth Cooper** chip, then **Save**. The slot is back to **Co-instructors** Tiana Cheung.
11. **Edit slot** once more and change **Instructor (optional)** to **Tiana Cheung**: her chip disappears, because nobody co-teaches their own class. In **Co-instructors (optional)** pick **Gifford Cheung**, then **Save**. The slot reads **Instructor** Tiana Cheung, **Co-instructors** Gifford Cheung, and the weekly schedule's Tuesday reads **with Tiana Cheung, Gifford Cheung**.
12. Swap them back the same way: **Instructor** Gifford Cheung, co-instructor Tiana Cheung, **Save**.

**A substitution replaces both**

13. On the **Tue · 6:00 PM** slot click ⇄ (**Substitute instructor for one date**). **Occurrence date**: next Tuesday; **Substitute instructor**: **Mason Bendixen**; **Reason**: `Both away` → **Substitute**.
14. **Weekly schedule** (use **Next week** if needed): that Tuesday reads **with Mason Bendixen (Substituting for Gifford Cheung, Tiana Cheung)**. The Tuesday after it still reads **with Gifford Cheung, Tiana Cheung**.

---

# Phase 11 — Forgot password (OQ-2)

Fact 12: this is built in the two honuware repos and then pinned here. **Nothing else in this plan depends on it.** The production rollout (Phase 15) goes ahead with or without it; until it ships, instructors get in with 5.2's temporary password. It follows the same build → push → pin cycle Phase 4 establishes, and touches some of the same files, so it starts after Phase 4's pin bump rather than in parallel.

During development, knottyyoga builds against the local `server_components` checkout via `FETCHCONTENT_SOURCE_DIR_HONUWARE` in the gitignored `CMakeUserPresets.json`, so both repos build together before anything is pushed.

### 11.1 Schema *(server_components: platform/db_schema)*
- [x] New framework table `password_reset_tokens`:
  - `id`
  - `person_id` (FK → people, **UNIQUE**)
  - `token_hash`
  - `expires_at` (µs)
  - `attempts` (INT default 0)
  - `created_at`

  It has the same shape as `email_verifications` but is a separate table, because a pending verification and a pending reset must be able to coexist. Registered like `email_verifications`: schema only, **never** in the admin CRUD allow-lists, so no generic endpoint can read a token row.
  - *As built:* every new database gets it from `CreateFrameworkTables`; the existing guard test checks that list against the schema. Its `token_hash` is also redacted for admins, as `email_verifications`' is. `CreateFrameworkTablesTest.PasswordResetTokensIsBuiltAndKeptOutOfTheAdminLists` checks all three lists the CRUD endpoints read (top-level, nested, allowed).
- [x] Idempotent framework migration **`honuware/0002_password_reset_tokens`**, with a migration test.
  - *As built:* the table has a foreign key onto `people`, so the migration describes every framework table (`MakeFrameworkTables`) and creates just this one. That is the framework version of memory trap #4: a schema holding only the one table can't type the key. The test drops the table, applies the migration, uses the table through its helper, checks the cascade on deleting the person, and applies it again.
- [x] `login_attempts` kind `password_reset`; `auth_events` kinds `password_reset_requested` and `password_reset_completed`.
  - *As built:* **two** `login_attempts` kinds, because the two caps count differently. `password_reset_request` counts every "send me a link" request, success or not, since a request always "succeeds". `password_reset` counts uses of a link, and only failures, as verify does. With one kind, failed resets would have eaten into the request cap. `password_reset_requested` is written only when the address has an account; there is no person to record otherwise.

### 11.2 Table helper *(server_components: platform/sql_util/table_helpers)*
- [x] `TableHelpers::PasswordResetTokens`: `ReplaceForPerson` (delete + insert, so a new request kills the old link), `GetForPerson`, `IncrementAttempts`, `DeleteForPerson`. Tests for each.
  - *As built:* `IncrementAttempts` is one conditional `UPDATE … RETURNING`, the same claim `email_verifications` uses. It counts the attempt only on a link that hasn't expired and has tries left, and returns the row for the caller to compare, so two concurrent guesses can't both slip under the limit. 10 tests.
  - *Also added, each tested:* `LoginAttempts::RecentAttemptCountForEmail` / `…ForIp`, which count every attempt and not just failures, for the request cap. And `People::ClearLoginLockout`, which zeroes the failure count and lifts `locked_until`, as a successful sign-in does.

### 11.3 Business logic *(server_components: platform/business_logic/auth)*
- [x] `Auth::PasswordResetHelper::RequestReset(transaction, secrets, mail, email)`:
  - Unknown email: does nothing, and sends no mail.
  - Known email:
    - a 32-byte secret, stored only as its hash (the same hashing `email_verifications` uses)
    - expires in 60 minutes (secret `auth_password_reset_ttl_minutes`, default 60)
    - sends the reset mail
  - *As built:*
    - It returns the account's person id (0 for an unknown address) so the endpoint can write the audit event.
    - All the SQL runs inside the call; the mail itself is **queued on the ThreadPool** by the helper, keeping email in business logic. The queued task never touches the database, so the endpoint can still write afterwards. A failed send is logged as `[password_reset] event=mail_failed`.
    - The mail goes to the address as **stored**: email is case-insensitive, so `ALICE@Example.com` finds the account but the mail is addressed `alice@example.com`.
- [x] Reset mail `password_reset_mail.{h,cpp}`, beside `person_verify_mail`:
  - a `FormatString` template, wrapped in `NormalizeCrLf`
  - sender from `::Mail::LoadSenderAddress`
  - tenant branding
  - a link to the SPA's `reset-password` page with `email` and `secret` URL-encoded, built from `website_address_login` exactly as the verification link is
  - Subject: "Reset your {site name} password"
  - Body: who asked, the link, "expires in 60 minutes", and "If you didn't ask for this, ignore this email. Your password has not changed."
  - *As built:*
    - The page the link opens is a new secret, `password_reset_link` (default `reset-password`), appended to `website_address_login` exactly as `activation_link` is for verification.
    - The body reads: "Hello {first name}, Someone asked to reset the password for the {studio} account {email}." Then a **Reset your password** link, "This link expires in 60 minutes and works once.", and the plan's closing line.
    - **The link percent-encodes everything but letters, digits and `- . _ ~`.** The verification link's encoder leaves `+` alone. The SPA's router reads an unencoded `+` as a space, so a plus-addressed email (`mason+test@…`) would arrive as `mason test@…` and the reset would fail. This was checked against Angular itself. The verification link had this bug (OQ-25); 11.11 fixes it with the same encoder.
    - 7 tests.
- [x] `Auth::PasswordResetHelper::CompleteReset(transaction, secrets, email, secret, newPassword)`:
  - **Failure.** Missing, wrong or expired all collapse to one generic failure. Each wrong secret counts, and the fifth deletes the token.
  - **Password rules.** The same rules registration enforces.
  - **Success:**
    - `UpdatePassword` with production Argon2 cost
    - clear `must_change_password`
    - clear `failed_login_attempts` / `locked_until`
    - set `email_verified_at` if empty, which also rescues an account whose verification link expired
    - delete the token
    - `LogoutPerson`, which ends every session and remember-me device token
  - *As built:*
    - It returns `Success`, `InvalidLink` (every bad case) or `PasswordRejected`.
    - The password check (registration's only rule is "not empty") comes first, so it costs no try.
    - An attempt is claimed **before** the secret is compared, so a correct secret also counts, and so does one that isn't even valid base64. The fifth wrong one deletes the link.
    - Verifying an unverified email has to come **before** the password change, because `PersonHelper` only changes a verified account's password.
- [x] `LoginGate::CheckBeforePasswordReset(transaction, secrets, email, ip)`: per-email and per-IP request caps (defaults 5 per hour per email and 20 per hour per IP, as secrets), so the button cannot be used to flood someone's inbox.
  - *As built:* named **`CheckBeforeForgotPassword`**, beside a second check, **`CheckBeforeResetPassword(ip)`**, for 11.4's reset endpoint (30 failures per IP per 15 minutes, like verify). Six new framework secrets, all defaulted in `secret_values.cpp`:
    - `auth_password_reset_ttl_minutes` = 60
    - `…_max_requests_per_email` = 5
    - `…_max_requests_per_ip` = 20
    - `…_request_window_in_micros` = 1 hour
    - `…_max_failures_per_ip` = 30
    - `…_failure_window_in_micros` = 15 minutes

    A database that predates them has no rows, and the code falls back to the same numbers.
- [x] Tests:
  - an unknown email sends nothing
  - a known one sends exactly one mail whose link carries the secret, and the stored value is not the secret
  - a second request replaces the first, so the old link fails
  - an expired link fails
  - a wrong secret fails and counts, and the fifth wrong secret deletes the token
  - success changes the password (the old fails and the new verifies, via `VerifyPassword`)
  - success clears must-change and the lockout, verifies the email, ends sessions and device tokens, and the link cannot be reused
  - the gate caps per email and per IP
  - *As built:* 19 helper tests plus 9 new gate tests. Every outcome is read back through the code production uses:
    - the password through `VerifyPassword` and `LoginPerson`
    - the lockout through `LoginGate::CheckBeforeLogin`
    - sign-outs through session lookup and remember-me
    - verification through `IsPerson`
    - whether a link still works through `CompleteReset` itself

    The session checks first prove the session and the remember-me token work *before* the reset, so they can't pass vacuously. Also checked:
    - the production Argon2 cost, from the hash's own parameters
    - the mail goes to the stored address
    - a pending verification and a reset coexist

### 11.4 Endpoints *(server_components: platform/endpoints)*
- [x] `POST /api/forgot_password` `{email}`:
  - **Always `200`** with the same body, whether or not the account exists.
  - Synchronous SQL first, then the mail send is queued on `ThreadPool` ([[feedback_sync_sql_before_threadpool_queue]]), so response time does not reveal whether the account exists either.
  - Gate denial → the generic 429.
  - *As built:* the body is empty, as `/api/verify`'s is. Every request is recorded (write-behind, last) toward the caps. An unexpected error answers the generic 500 and logs the detail; the endpoint never echoes it, because an error that only happened for a real account would give the account away.
- [x] `POST /api/reset_password` `{email, secret, new_password}`:
  - `200` on success, or one generic `400` "reset link is invalid or expired"
  - per-IP gate as `/api/verify` has
  - attempt recorded async, last
  - `auth_events` row on success
  - *As built:* a missing field is a 400 naming it (`new_password is required`), as register's are.
- [x] **Found while building, not in the plan: both endpoints had to join the CSRF guard's exempt list.** The guard refuses any POST without a `csrft` cookie, and the cookie is only set at sign-in, so someone who can't sign in has none. Without the exemption, every forgot-password request from the SPA would have been refused 403. The endpoint tests can't see this, because `handle_full` skips the guard. It is now exempt the same way `/api/register` and `/api/verify` are, and that's safe for the same reasons:
  - a forged `forgot_password` can only mail a link to that address's own inbox, and is capped
  - a forged `reset_password` would need the secret from that mail

  Two `CsrfGuardTest` cases pin it, and the end-to-end run (11.9) goes through the real guard.
- [x] Both are added to the auth-endpoint list in the honuware `ErrorInterceptor`, so a 401 there never bounces to the login page. *(Done in 11.6, where the interceptor lives.)*
- [x] Endpoint tests:
  - an unknown email gets 200 and no mail
  - a known email gets 200 and one mail
  - 429 when gated
  - reset happy path, then the same link a second time fails
  - a bad secret gets the generic 400
  - `ThreadPool::Shutdown()` right after each `handle_full`
  - *As built:* 8 forgot + 7 reset tests. The reset tests get their link from a real `/api/forgot_password` request, read out of the mail it sent. They also cover:
    - known and unknown addresses get byte-identical answers
    - the per-IP cap on requests
    - every request is recorded
    - failures are recorded and successes aren't counted
    - both auth events
    - sign-out on success
    - missing fields

### 11.5 Ship the server side
- [x] Gates on `server_components` (Linux `docker/build_and_test.sh` + Windows) and on knottyyoga against the local checkout; Claude reports both platforms.
  - **server_components: Windows 1889 passed, Linux 1889 passed**, with the same test names on both (1819 at the Phase 4 pin, so 70 new).
  - **knottyyoga against the local checkout, Linux: 5376 passed** (5306 + the 70), and both `knottyyoga_database_helper` and `knottyyoga_the_server` build. The log confirms `honuware : /honuware (local override)`. No knottyyoga server source changed.
  - **knottyyoga against the local checkout, Windows: not finished.** The build was stopped at 1161 of 2517 steps, with no errors so far, when Claude Code reaped background runs for low memory (10/18 evening). Superseded by the pinned Windows run below, which builds the same code.
- [x] **Mason** commits and pushes `server_components`: **`4f72341`** ("Phase 11 — Forgot password"), green in CI.
- [x] Claude bumps `GIT_TAG` in knottyyoga's top-level `CMakeLists.txt` to that SHA, removes the local `FETCHCONTENT_SOURCE_DIR_HONUWARE` override, and re-runs the knottyyoga gates.
  - *Done:* `GIT_TAG` is `4f723413ada4ac823ea71e280de0a40013fd5b1b` in **both** knottyyoga and CommunityFinder (Mason asked for both, 10/18). Nothing else changed in either repo.
  - The Windows build tree is back on the pin: `FETCHCONTENT_SOURCE_DIR_HONUWARE` is empty in its cache, and `_deps/honuware-src` is a clean clone at `4f72341`. Visual Studio's configuration is the plain **x64-Debug**, so it won't put the override back.
  - **knottyyoga, pinned, Linux: 5376 passed**, the same count as against the local checkout, and both executables build. The log says `pinned SHA (FetchContent will clone it)`.
  - **knottyyoga, pinned, Windows: 5376 passed**, with the same 5376 test names as Linux. `knottyyoga_database_helper` and `knottyyoga_test_helper` build. `knottyyoga_the_server` wasn't relinked because Mason's copy was running; the Linux run builds it from the same sources.
  - **CommunityFinder, pinned, Linux: 1910 passed** (1840 at `cddb9cb`, plus the 70 new honuware tests), and both `communityfinder_database_helper` and `communityfinder_server` build. Its server now has both endpoints and the new table, but its UI stays on `@honuware/ui` 0.1.2, with no **Forgot password?** link, until it moves to 0.3.0.
  - **CommunityFinder, pinned, Windows: 1910 passed**, with the same 1910 test names as Linux. Its server, database helper and test helper all build.
  - **Mason:** commit the one-line `GIT_TAG` change in each repo. knottyyoga's waits for the `@honuware/ui` 0.3.0 install (11.7), so it goes in with the UI changes; CommunityFinder's can go now.

### 11.6 Frontend library *(honuware-web-components: `@honuware/ui/auth`)*
- [x] `HonuwareAuthAccess` gains `forgotPassword(email)` and `resetPassword(email, secret, newPassword)`. `AUTH_ROUTES` gains `forgotPasswordPath: '/forgot-password'` and `resetPasswordPath: '/reset-password'`.
  - *As built:* the interface is `AuthAccess`. Both methods are on it, on `AuthHttpAccess`, on the serializing `HonuwareAccessProxy`, on `AuthService`, and on the testing `MockAuthAccess`; the mock accepts any link for a known account and signs it out. Both route fields are **optional**, as `checkEmailPath` is, so an app's existing `AuthRoutes` object still compiles. With no `forgotPasswordPath`, the login page simply shows no link.
- [x] `LoginComponent`: a **Forgot password?** link beneath the password field.
  - *As built:* right-aligned under the field, above **Remember Me**, with a 44px tap target like the page's other links.
- [x] `ForgotPasswordComponent`:
  - title **Reset your password**
  - **Email** field
  - **Send reset link** button
  - afterwards, always: "If an account exists for that email, we've sent a link to reset your password. Check your spam folder too." with **Back to log in**
  - *As built:* a line above the field says what happens ("Enter the email address you sign in with and we'll send you a link to choose a new password."). The button is disabled while the request is out, and the form is replaced by the message so nobody sends twice by accident. **Back to log in** is there before and after.
- [x] `ResetPasswordComponent`:
  - reads `email` and `secret` from the query string, as `VerifyComponent` does
  - **New Password** and **Confirm New Password**, with show/hide toggles
  - a **Set Password** button
  - success: toast "Your password has been changed. Please log in.", then the Log In page
  - failure: "This reset link is invalid or has expired." with a **Send a new link** button back to the forgot page
  - *As built:*
    - Its title is **Choose a new password**.
    - Like Verify, it scrubs the secret from the address bar the moment it opens, leaving `/reset-password`. The values stay in memory, so the secret isn't in browser history, or in a Referer, while the person types.
    - A link missing its email or secret shows the failure state straight away, without calling the server.
    - One eye button shows or hides both fields.
    - "Passwords do not match." appears under the second field.
- [x] Specs:
  - the link exists on Log In
  - forgot shows the same message for success and for any error
  - reset rejects mismatched passwords without calling the server
  - reset's success and failure paths
  - *As built:* the library has **514** specs, up from 482 (32 new).
    - Log In: 4 (the link, a remapped path, no link when unrouted, its tap target).
    - Forgot: 8 (the message after 429, 500 and network failures as well as success; nothing sent without a valid email).
    - Reset: 12 (URL scrub, missing parts, toggles, tap targets).
    - Also `AuthHttpAccess` 2, the proxy 1, `AuthService` 1, the `ErrorInterceptor` 1 (no bounce to login on a 401 from either endpoint), and `MockAuthAccess` 3.

    `npm run lint` and `npm run build` are clean.
- [x] Minor version bump in `projects/honuware-ui/package.json`. **Mason** commits, pushes, creates the tag and pushes the tag ([[reference_honuware_ui_release]]).
  - *Done by Claude:* `0.2.0` → **`0.3.0`**, and the package README lists the two pages and how to route them.
  - *Released by Mason:* commit `b7d77d4`, tag `v0.3.0`. **`@honuware/ui@0.3.0` is on npm** (published 10/18 21:49 Pacific) and is now `latest`.
  - The published code matches Claude's local build. The only differences are in a few component templates: Claude's Windows checkout has CRLF line endings there, CI's has LF. Like 0.2.0, the package has a single top-level `package.json`.
  - *CommunityFinder* pins `@honuware/ui` 0.1.2 and has its own `AuthAccess` implementation. When it moves to 0.3.0 it needs the two new methods; nothing changes for it until then.

### 11.7 Knottyyoga wiring *(ui/)*
- [x] `npm install @honuware/ui@X.Y.0 --save-exact`.
  - *Done 10/19:* `npm install @honuware/ui@0.3.0 --save-exact`. `ui/package.json` and the lock file now say **0.3.0**, and the published package replaced Claude's local build in `node_modules`.
- [x] `ServerAccess` (+ network + mock + proxy) implements `forgotPassword` / `resetPassword`; `ServerAccess.mock.spec.ts` cases.
  - *As built:* in local mode the mock accepts any request, and any reset link that has all its parts; a reset signs the mock session out. 4 mock spec cases.
- [x] `auth.routes.ts`: `forgot-password` → `ForgotPasswordComponent`, `reset-password` → `ResetPasswordComponent`.
  - *As built:* plus a new `auth.routes.spec.ts` (2 cases) pinning both routes to `DEFAULT_AUTH_ROUTES`' paths, which are where the login link and the reset mail send people. The mobile-audit route list gained both pages; its Karma guard fails on any unlisted route.
- [x] `ng test`, `tsc --noEmit`, `ng build`.
  - Against the local 0.3.0 build: **Angular 3614 passed** (3608 + 6). `tsc` is clean for both app and specs, `ng lint` is clean, and `ng build` is OK with the same six old budget warnings.
  - **Re-run against the published 0.3.0 (10/19):** 3614 passed; `tsc` clean for app and specs; `ng build` OK with the same six budget warnings.

### 11.8 Mutation check

- [x] **C++ (honuware, Windows):** 22 mutations are scripted, one per safeguard. Among them:
  - storing the secret rather than its hash
  - a new request not replacing the old link
  - the fifth wrong secret not deleting it
  - a used link surviving
  - sessions, lockout or must-change surviving a reset
  - no email verification
  - the test-cost hash
  - expired links working
  - the request cap counting only failures
  - requests not recorded
  - no gate
  - failures recorded as successes
  - the CSRF exemption missing
  - the migration never creating the table
  - `+` left unencoded
  - `ClearLoginLockout` leaving the lock
  - the attempt count skipping successes
  - a new database without the table
  - mail to the typed address
  - the TTL secret ignored

  **Run 10/19 against the committed `4b59f5b`: all 22 caught.** Each one failed at least one test written for that safeguard. A few examples:
  - Storing the secret instead of its hash fails `OnlyAHashOfTheSecretIsStored` and 8 others.
  - A used link surviving fails `TheLinkWorksOnce` and the endpoint's `TheLinkSetsANewPasswordAndThenStopsWorking`.
  - The missing CSRF exemption fails `CsrfGuardTest.ForgotPasswordPathBypasses`.
  - A new database without the table fails both `CreateFrameworkTablesTest` cases.

  The `+` mutation now targets the encoder in `util/types.cpp`, where 11.11 moved it. It fails `TypesTest.URLEncodeQueryValueEncodesPlusAndAt` and `PasswordResetMailTest.LinkEncodesAPlusAddressedEmail`. Every file was restored byte for byte, and `server_components` shows no changes. The rebuilt suites pass clean: 149 tests in 12 suites.
- [x] **@honuware/ui:** 7 mutations are scripted:
  - a failed request shows nothing
  - the secret stays in the address bar
  - mismatched passwords reach the server
  - a failed reset offers no new link
  - **Forgot password?** points elsewhere
  - a 401 from the reset endpoints bounces to Log In
  - the wrong JSON key

  **Run 10/19 against the committed `b7d77d4` (0.3.0): all 7 caught by a spec.** Some examples:
  - The address-bar mutation fails "scrubs the secret from the URL as soon as the page opens".
  - The mismatch mutation fails "rejects mismatched passwords without calling the server".
  - The wrong-key mutation fails "resetPassword POSTs a snake_cased body to /api/reset_password".
  - A wrong **Forgot password?** target fails all 3 of the link's specs.

  The first version of the address-bar mutation didn't compile, so it proved nothing about the spec. It was rewritten to compile, and re-run. Every file was restored and checked by hash, and the library shows no changes.

### 11.9 End to end

- [x] It runs against a throwaway `phase11_check` database through the real server, middleware included, with outgoing mail pointed at a closed local port so nothing leaves the container. It covers:
  - a signed-out request getting through the CSRF guard
  - identical answers for known and unknown addresses
  - one hashed link stored
  - both requests counted, and the audit events
  - a wrong secret giving the generic 400 and counting
  - Beth signing in with a Phase 5 temporary password, then resetting it, with her old session ended and the new password working
  - must-change cleared
  - the link refused a second time
  - the sixth request in an hour answering 429

  A second mode drops the table and checks that `--migrate` brings it back.

  **Run 10/19 on Linux, knottyyoga built at the pinned `4b59f5b`:**
  - **Install pass: 19 of 19 checks passed** (a fresh `phase11_check`, then `--install_schema`).
    - Every send failed at the closed port (`mail_failed … Server connecting failed.`), so no mail left the container.
    - The server log's only errors are the six the checks asked for: two 400s for a bad link, a 401 for the ended session, a 401 for the old password, the 429, and the missing `new_password`.
  - **Migrate pass: all checks passed.** This is a fresh install with `password_reset_tokens` dropped, like a site installed before Phase 11.
    - `--migrate` reported `applied=13 skipped=0`, the table is back, and the ledger records `0002_password_reset_tokens`.
    - A request then stored exactly one link.
    - `phase11_check` was dropped afterwards.

### 11.10 Hand-testing walkthrough

*Mason verified the forgot-password flow by hand on 10/19.*

**Before you start.** Use a freshly recreated local database, the local server built from Visual Studio's **x64-Debug** configuration at 11.11's pin (`4b59f5b`; a server started before 10/19 is older, so stop it and rebuild it first: steps 13–15 fail on the old one), and the site from `ng serve`, restarted since `@honuware/ui` 0.3.0 was installed (11.7). Local mail must be set up (`HONUWARE_MAIL_APP_PASSWORD` at recreate time), because the steps read a real reset mail in the masonbendixen@gmail.com inbox. Start signed out.

**Asking for a link**

1. Open **Log In**. Under the **Password** field, at its right, is **Forgot password?**. Click it.
2. The page is **Reset your password**. Type `masonbendixen@gmail.com` in **Email** and click **Send reset link**. The form is replaced by: "If an account exists for that email, we've sent a link to reset your password. Check your spam folder too."
3. Click **Back to log in**, then **Forgot password?** again. Send `nobody@example.com`. The page says exactly the same sentence.
4. In the masonbendixen@gmail.com inbox, one new mail from **Knotty Yoga**, subject **Reset your Knotty Yoga password**. It reads "Hello Mason," and "This link expires in 60 minutes and works once."

**Using it**

5. Click **Reset your password** in the mail. The page **Choose a new password** opens, and the address bar shows only the page, with no email or secret.
6. Type `newpass1` in **New Password** and `newpass2` in **Confirm New Password**. "Passwords do not match." appears, and **Set Password** does nothing.
7. Make the second `newpass1`, and click the eye on either field: both passwords show. Click **Set Password**. A message says "Your password has been changed. Please log in." and the **Log In** page opens.
8. Log in with password `pass`: refused, "The email or password you entered is incorrect". Log in with `newpass1`: you're in.
9. Click the link in the same mail again, type `newpass2` twice, **Set Password**. The page says "This reset link is invalid or has expired." Click **Send a new link**: it opens **Reset your password**.

**It signs you out everywhere**

10. In a second browser (or a private window), log in as `masonbendixen@gmail.com` / `newpass1`.
11. In the first browser, send yourself another link (steps 1–2), open the new mail's link, and set the password back to `pass`.
12. In the second browser, reload the page: you're signed out, and logging in with `newpass1` is refused. Log in with `pass` to finish where you started.

**A plus-addressed account activates (11.11)**

13. Sign out. On **Log In**, click **Create an account**. Enter **Email** `masonbendixen+oq25@gmail.com`, **First name** `Plus`, **Last name** `Test`, and `pass` in both **Password** and **Reenter password**, then click **Register**. **Check your email** says "We've sent an activation link to **masonbendixen+oq25@gmail.com**."
14. Gmail delivers `+` addresses to the main inbox, so the mail lands in masonbendixen@gmail.com: **Knotty Yoga Account Activation**, "Greetings Plus,". Click **Click here to activate your account.** The site says "Email verified — please log in." and opens **Log In**. Before the fix it said "Verification failed or expired. Please try again."
15. Log in as `masonbendixen+oq25@gmail.com` / `pass`: you're in.

### 11.11 Plus-addressed activation links (OQ-25, Mason: fix now)

- [x] **Foundation** (`util/types.h`): `URLEncodeQueryValue(value)` percent-encodes everything but letters, digits and `- . _ ~`. It is the reset link's encoder (11.3), moved out of `password_reset_mail.cpp` so both mails share it. The reset link is unchanged.
  - 4 tests in `types_test.cpp`: `+` and `@`; the unreserved set left alone; everything else, including UTF-8, as upper-case `%XX`; a round trip through `URLDecode`.
- [x] **Business logic** (`business_logic/auth`): the activation-mail builder encodes the address itself with `URLEncodeQueryValue`, and `PersonHelper::PreliminaryRegisterPerson` now passes the stored address instead of `URLEncode`'s output. `mason+test@example.com` now goes out as `?email=mason%2Btest%40example.com`.
  - `person_verify_mail_test`: the expected link now has `%40` for `@`. New `PlusAddressedEmailIsEncodedInTheLink`.
  - `person_test`: new `PlusAddressedEmailActivatesFromItsLink`. It registers `mason+test@example.com`, reads the email and secret back out of the sent mail the way Angular does (a bare `+` becomes a space, then `%XX` is decoded), passes them to `VerifyPersonEmail`, and checks that the account is now verified.
  - No change to `/api/verify`, the SPA or either app. The test utility that computes the expected activation mail goes through the same builder, so the existing activation tests in all three repos follow the change by themselves.
- [x] Gates: `server_components` on both platforms.
  - **Windows: 1895 passed. Linux: 1895 passed** (1889 + the 6 new), with the same test names on both.
  - **knottyyoga against the local checkout, Linux: 5382 passed** (5376 + the 6), and both executables build. Its own activation tests needed no change.
- [x] Mutation check, 2 mutations, on the Windows honuware build. **Both caught**:
  - The builder going back to `URLEncode`: 4 tests fail. They are both of the plus-address tests and the two exact-body tests.
  - The encoder leaving `+` unencoded: 4 tests fail, including the reset mail's plus-address test, which now goes through the shared encoder.

  The files were restored byte for byte, and a clean run of the same suites passes.
  - *Not mutated:* `PreliminaryRegisterPerson` pre-encoding again with `URLEncode`. That would encode twice, but `URLEncode` leaves every character an address normally has (`+`, `@`, `.`, letters, digits) as it is. So the output is the same for any real address, and no test can tell the two apart.
- [x] **Mason** commits and pushes `server_components`: **`4b59f5b`** ("Phase 11 cleanup", the 9 files above), green in CI.
- [x] Claude bumps `GIT_TAG` in knottyyoga and CommunityFinder to `4b59f5b` and runs both apps' pinned gates on both platforms. Every build cloned `4b59f5b` with no local override.
  - **knottyyoga: Windows 5382 passed. Linux 5382 passed**, with the same test names on both. Linux built both executables. Windows built the tests, `knottyyoga_database_helper` and `knottyyoga_test_helper` but not `knottyyoga_the_server`, because Mason's server was running from that file.
  - **CommunityFinder: Windows 1916 passed. Linux 1916 passed**, with the same test names. Both platforms built every executable.
  - **Mason:** commit the one-line `GIT_TAG` change in each app. In knottyyoga it goes in with the Phase 11 UI changes that are still uncommitted.

---

# Phase 12 — The class page names the membership; cleanup

**Implemented 10/19/2026, not committed.** It is all in knottyyoga, with no `server_components` change and no pin bump.
- **knottyyoga: Windows 5389 passed. Linux 5389 passed.** That is Phase 11's 5382 plus 7 new tests, with the same names on both platforms.
- **Angular: 3618 passed.** That is 3614, less the deleted service's one spec, plus 5 new.
- All 12 mutations were caught.

### 12.1 Say which membership a class needs (OQ-17, confirmed)
- [x] **Business logic.** `ClassCatalogHelper::GetClassDetail` adds the class's non-empty membership-group `inline_description`s to the detail, reusing `ClassAccessHelper::GetClassRequirements`, which already resolves them. Test: Beginner Knotty Yoga's detail carries "Silver or Gold membership", and Knotty Yoga's carries "Gold membership".
  - *As built:* a new `ClassDetail.membershipRequirements` list.
    - A group counts as a membership when it has a permission in it. That is `Annotate`'s own rule, the one behind the schedule's badges, so the class page and the badges can't disagree.
    - The list belongs to the class, so every viewer gets the same one.
  - **Found at 12.3's check and fixed here: Required Skills never appeared on any class page.**
    - `ClassDetail.requiredSkills` was declared "for Phase 3", and nothing ever filled it.
    - The page's chips, held/missing marks and "You're missing" box were built and tested, but never had data. 6.6 step 6 and the 14.3 walkthrough both expect a **Handstand** chip.
    - The same loop now fills the list with the skill names from every group that has no permission in it.
  - Tests in `class_catalog_helper_test`:
    - `GetClassDetailNamesTheMembershipAndTheSkillItNeeds`: a visitor and a Gold member get the same two lists.
    - `GetClassDetailCountsAGroupWithAPermissionAsAMembership`: a "permission or skill" group is a membership, and its skill is not listed as required.
    - `GetClassDetailLeavesOutAMembershipWithNoDescription`: an undescribed membership adds nothing, and the class is still members-only. A class with no groups needs nothing.
  - Through the seeded rules, in `seed_app_data_test`: `PopulateClassRequirementsNamesWhatEachClassNeedsOnItsPage`.
    - All six classes are checked against the table the seeder walks, including the plan's two examples.
    - Handstands, Aerial Rope and Intermediate Partner Acro each carry their skill.
- [x] **Endpoint.** The class-detail endpoint serialises it; `scheduling_key_value_table.cpp` already writes `inline_description` for the admin view. Endpoint test.
  - *As built:* `/api/classes/<id>` sends `membership_requirements` as a JSON array beside `required_skills`, and both are always present.
    - The endpoint builds them from the struct, as it already did for `required_skills`, rather than through the key-value table, where they would be one joined string.
  - Tests:
    - New `GetClassDetailTest.NamesTheMembershipAndTheSkillTheClassNeeds`.
    - The existing detail test now checks that both lists are present and empty.
- [x] **Frontend.** The class page's "Members only" pricing state reads **"Members only: Silver or Gold membership"**, falling back to today's "Upgrade your membership to attend." when there is no description. Type + mock + spec, covering both the described and the undescribed state.
  - *As built:*
    - When the class names its membership, that one line replaces both of today's lines.
    - When it names none, or only blank text, the page keeps "Members only" and "Upgrade your membership to attend."
    - Two required memberships join with "and".
    - A viewer who can attend sees no membership text.
    - The network layer turns a missing list into an empty one.
  - Specs:
    - 4 in `class-detail.component.spec.ts`, checking the notice's exact lines.
    - 1 in `ServerAccess.mock.spec.ts`; the mock's default is also asserted.
    - The network layer has no spec file in this repo, so its one-line default is checked by `tsc` only.

### 12.2 Dead code
- [x] Delete `pages/public/services/homepage.service.ts` and its spec (fact 8).
  - Only its own spec imported it. The emptied `pages/public/services/` folder went too.

### 12.3 Signed-in visitors see every class (OQ-22, confirmed)
- [x] **Business logic.** `ClassCatalogHelper::GetClassesVisibleToPerson` keeps its "has an upcoming run" filter and drops the access-gate filter, so a signed-in visitor gets the same classes as a signed-out one. Update the `class_catalog_helper_test` cases that pin the hiding, and add one through the seeded rules: a signed-in non-member and a Silver member each see all six.
  - *As built:* `personId` stays in the signature, unused, because the endpoint still calls a separate function for signed-in visitors.
  - Tests:
    - `GetClassesVisibleToPersonHidesMembersOnlyFromNonMember` is replaced by `…ShowsClassesTheViewerCannotAttend`. A non-member and a member each see a members-only class and an approval-gated one.
      - It first checks that the gate really refuses them, so the test can't pass vacuously.
    - Through the seeded rules: `EverySignedInViewerSeesAllSixClasses`. A non-member, a Silver member and Mason each see exactly the six; before this they saw 0, 1 and 0.
  - The same list also feeds **Close Classes**' class picker and **Attendance History**'s class filter. So a staff account without a membership now gets every class there too.
- [x] **Endpoint.** `/api/classes` test: a signed-in non-member gets six classes.
  - *As built:* `GetClassesTest.SignedInNonMemberSeesMembersOnlyClasses` signs in through a real session cookie and sees two members-only classes of its own.
    - The "six" is pinned through the seeded rules above, in the reader this endpoint calls unchanged.
- [x] Check in the front end: the class page already shows **Members only** and **Required Skills** for a class the viewer can't attend. Whether an **All Classes** card should carry a badge too is checked at implementation; the weekly schedule's badges are the model.
  - **Members only:** yes.
  - **Required Skills:** no. It never had data, and is fixed in 12.1.
  - **A badge on the All Classes cards:** not added; that is **OQ-26**, with "no badge" as the default.

*(Not in scope: `ServerAccess.mock.ts`'s local-mode catalog still has the old classes. It is fixture data for `ng serve -c local` and for dozens of specs; changing it is churn with no user-visible effect.)*

### 12.4 Verification
- [x] knottyyoga C++:
  - **Windows: 5389 passed. Linux: 5389 passed.** Both platforms ran the same test names.
  - Every executable built on Windows, the server included.
  - On Linux the gate built both `knottyyoga_database_helper` and `knottyyoga_the_server`.
  - The final runs are on the final code. Removing an untested de-duplication of skill names, after the first runs, is the only change since.
- [x] Angular (`ui/`):
  - **3618 passed.**
  - `tsc --noEmit` is clean for the app and for the specs, and `ng lint` passes.
  - `ng build` is OK, with the same six budget warnings as before.

### 12.5 Mutation check
- [x] **C++, on the knottyyoga Windows build: 7 mutations, all caught.**
  - The access gate filtering **All Classes** again fails 3 tests: the helper test, the seeded six-class test and the endpoint test.
  - The membership text never reaching the detail fails 4 tests.
  - Counting every group as a membership, approvals included, fails 3 tests.
  - Only a group's first literal deciding whether it is a membership fails the mixed-group test.
  - The skill never reaching **Required Skills** fails 3 tests.
  - An undescribed membership adding an empty requirement fails the no-description test.
  - The endpoint dropping the list fails the endpoint test.

  Every file was restored byte for byte, and a clean run of the same suites passes: 39 tests.
- [x] **Angular: 5 mutations, all caught.**
  - Joining with a comma.
  - The notice not naming the membership.
  - A blank description counting as a membership.
  - An undescribed class losing the generic wording.
  - Local mode dropping the list.

  Every file was restored, and the two specs pass clean: 50 and 630 tests.

### 12.6 Hand-testing walkthrough

**Before you start.** Recreate the local database with `knottyyoga_database_helper --recreate_database`. Start the server from Visual Studio's **x64-Debug** configuration and the site from `ng serve`, both rebuilt or restarted since this phase. Use a window at least 1280 px wide, and start signed out.

**Signed out: each class page says what it needs**
1. Top nav **Our Classes**. The page is headed **All Classes**. Click **Beginner Knotty Yoga**.
   - **Pricing** reads **Members only: Silver or Gold membership** on one line, with no "Upgrade your membership to attend." under it.
   - There is no **Required Skills** section.
2. Click **All Classes** at the top, then **Knotty Yoga**. **Pricing** reads **Members only: Gold membership**.
3. **All Classes** → **Handstands**.
   - **Pricing** reads **Members only: Gold membership**.
   - **Required Skills** shows one chip, **Handstand**.
4. Likewise:
   - **Aerial Rope**: **Members only: Gold membership**, and an **Aerial Rope** chip.
   - **Intermediate Partner Acro**: **Members only: Gold membership**, and an **Intermediate Partner Acro** chip.
   - **Beginner Partner Acro**: **Members only: Gold membership**, and no **Required Skills**.

**Signed in without a membership: every class is still listed**
5. **Sign In**: Email `kiteagle@gmail.com`, Password `kit`, then **Log In**. Kit holds no membership.
6. Top nav **My Classes**, then **Browse all classes**. **All Classes** shows the same six cards. Before this phase it showed Kit none.
7. Click **Handstands**.
   - **Pricing** reads **Members only: Gold membership**.
   - **Required Skills** shows the **Handstand** chip with a red mark, and a box: "You're missing: Handstand" and "Talk to staff for evaluation."
8. Sign out: **Kit** (the name menu) → **Sign Out**, then **Log Out** in the dialog.
9. **Sign In** as `masonbendixen@gmail.com` / `pass`. **My Classes** → **Browse all classes** shows all six cards. Mason's account holds no membership either, and before this phase it showed none.

**The weekly schedule is unchanged**
10. Still as Mason, top nav **My Classes**. The page is headed **Our Classes**.
    - **Beginner Knotty Yoga**, **Knotty Yoga** and **Beginner Partner Acro** rows still carry **Requires a membership**.
    - **Handstands**, **Aerial Rope** and **Intermediate Partner Acro** still carry **Prerequisite needed**.

    This phase changed the class list, not the schedule.

---

# Phase 13 — Attach the photos

*Moved here from Phase 9 on 10/18/2026. The photos aren't in yet, and nothing before Verification needs them, so the four phases that don't use them went first. The items are unchanged; only the numbers moved.*

Uses Phase 1's files.

**Status: implemented 10/20/2026, not committed.** Six photos and all five bios are in. Four photos are still to come (1.1: Beginner Knotty Yoga, Aerial Rope, Massage, Studio); each joins later as file + row + test expectation (13.1). Results are in 13.4.

### 13.1 Table-driven photo attachment *(seed)*
- [x] Extend `PopulateSeedPhotos` to one table per target, with one loop each, every lookup by natural key, and every outcome logged (the existing `AttachSeedPhoto` contract):
  - `instructors`, by person email
  - `classes`, by name
  - `products`, by code
  - `facilities`, by code
  - → As built: the tables are public in `seed_app_data.h`, so the tests walk them (D3): `kSeedPersonPhotos`, `kSeedInstructorPhotos`, `kSeedClassPhotos` and `kSeedProductPhotos`, one `SeedPhoto { key, file }` per row.
  - → The stored type now follows the file's extension (`.svg` is a vector, anything else a JPEG). The product loop used to pass "svg" for every row, which would have mis-stored `Massage.jpg` when it joins.
  - → **No facilities table yet.** Its only row is `Studio.jpg`, which isn't in. It arrives with that photo as a fifth table and loop; the header says so where the product table is.
- [x] Instructor headshots move off `PopulateInstructors` into this table, so every picture is listed in one place.
  - → `PopulateInstructors` now writes the rows and bios only. The bootstrap runs `PopulateSeedPhotos` **after** it (`create_database.cpp`); it used to run before, when the headshots didn't need instructor rows. A headshot's instructor row is found from the person's email.
- [ ] Rows (files from 1.1):
  - [x] instructors: Mason, Caleb, Beth, Tiana, Gifford
  - [ ] classes: all six. **Four are in:** Knotty Yoga, Handstands and Beginner Partner Acro keep their pictures, and Intermediate Partner Acro is new. Beginner Knotty Yoga and Aerial Rope wait for their photos.
  - [ ] product `massage` → `Massage.jpg` *(waiting for the photo)*
  - [ ] facility `knotty-yoga-studio` → `Studio.jpg` *(waiting for the photo)*
- [x] Mason's and Caleb's **profile** photos (`Mason.jpg`, `Caleb.jpg` on `people`) are unchanged. New instructors upload their own profile pictures.
  - → `MasonInstructor.jpg` and `CalebInstructor.jpg` are the new headshots, so each instructor picture now differs from that person's profile photo. Before, they were copies of each other.
- [x] Any photo still missing when this phase lands is left out of the table. It joins later as file + row + test expectation in one change (D10), and stays listed here unchecked until then.
  - → To add one: put the converted file in `src/database_helper/img/`, add its row to the table, and change the expected leftovers in `PopulateSeedPhotosPutsEachListedFileOnItsRow` (classes only). Massage also needs no new code; Studio needs the facilities loop.
- [x] **Also: the instructors' bios** (OQ-15). All five replace the one-line defaults in `kSeedInstructors`, with OQ-28's corrections.

### 13.2 Tests
- [x] `EverySeedPhotoFileExists`: walks the same tables the seeder walks and asserts each file is in `SeedImageDirectory()`. This catches a renamed or deleted file.
  - → It also requires each file to be non-empty: `AttachSeedPhoto` treats an empty file as a missing one.
- [x] Attachment by target type: each listed row has exactly one photo, and **classes not in the table have none** (the D5 wrong-row guard from the previous plan, kept).
  - → `PopulateSeedPhotosPutsEachListedFileOnItsRow`. "Exactly one" is guaranteed by the upload itself, which replaces a row's earlier photo. So the test checks something stronger: each listed row's stored picture is **byte for byte the listed file**, read back through the reader the photo endpoints serve from, with the right type. That is what catches two rows swapped.
  - → The same test reads all five instructors through the Instructors page's own reader (`hasPhoto`). It also checks that the classes left out are exactly Beginner Knotty Yoga and Aerial Rope, and that neither has a picture.
  - → Replaced: `PopulateSeedPhotosAttachesProfileAndClassPictures` and `PopulateInstructorsCreatesBothWithTheirOwnHeadshots`. `PopulateInstructorsSeedsAllFiveWithTheirBios` no longer checks photos; instead it checks that each bio names its instructor, which the one-line defaults didn't.
- [ ] Massage and studio through their public readers: the catalog product's `has_photo`, and the facility's photo on the location reader. *(With their photos.)*

### 13.3 Headshots crop from the top *(ui; added 10/20)*
- [x] Every instructor picture is shown in a square, `object-fit: cover`. The server only shrinks a picture to fit, so a 3:4 portrait reaches the page as 192 × 256, and the page cut the square from its **middle**. On the new waist-up headshots that cut the top of the hair off Beth, Caleb, Gifford and Tiana. 1.1 asked for head-and-shoulders; the cheaper fix than a reshoot is to crop from the **top**.
  - → `object-position: top` on all four places a headshot appears: the **Instructors** cards, an instructor's own page, **Favorite Instructors** and the **Manage Instructors** thumbnails. A landscape or square picture is unaffected, and a portrait an instructor uploads later gets the same framing.
  - → Each component's spec checks the computed style (`50% 0%`).

### 13.4 Verification
- [x] knottyyoga: **Windows: 5389 passed. Linux: 5389 passed.** Same test names on both, and both programs build. The count is unchanged because two tests were added and two retired (13.2).
- [x] Angular (`ui/`): **3622 passed** (3618 plus the four crop specs). `tsc` for the app and the specs, `ng lint` and `ng build` are clean; the build's one warning is the existing size budget on the class check-in page.
- [x] **End to end** (the gap no unit test covers is the bootstrap's own call order): `knottyyoga_database_helper --install_schema` into a throwaway database, then the real server.
  - The install log shows 2 profile photos, 5 headshots, 4 class pictures and 5 tier icons attached, and no `seed_image_missing`, `seed_image_failed` or `seed_lookup_missing` line.
  - Signed out, the Instructors feed returns all five, each with `has_photo` and its bio character for character, "®" and "’" included. Each card's photo URL serves a JPEG 256 px tall.
  - The four pictured classes serve their picture (Intermediate Partner Acro's at the new picture's proportions); Beginner Knotty Yoga and Aerial Rope answer 404.
  - **Run again with the old order** (photos before instructors): all five headshots go missing, with five `seed_lookup_missing` lines, and the check fails 10 times. The file was then restored byte for byte.

### 13.5 Mutation check
- [x] Each change broken on purpose, one at a time, and the suite run against it. Every one must fail a test. **All 11 caught**, plus the bootstrap-order swap, which only the end-to-end run can see (13.4). Each file was restored and the clean runs pass.

  | # | Broken on purpose | Caught by |
  |---|---|---|
  | m1 | No instructor gets a headshot | `PopulateSeedPhotosPutsEachListedFileOnItsRow` |
  | m2 | Every instructor gets Mason's headshot | `PopulateSeedPhotosPutsEachListedFileOnItsRow` |
  | m3 | The tier icons are stored as JPEGs | that test and `PopulateSeedPhotosGivesEachMembershipTierItsIcon` |
  | m4 | The photographs are stored as vectors | `PopulateSeedPhotosPutsEachListedFileOnItsRow` |
  | m5 | Every class picture lands on Aerial Rope | that test and `PopulateSeedPhotosSkipsPeopleThatDoNotExist` |
  | m6 | A listed file that isn't in `img/` | `EverySeedPhotoFileExists` (and the attachment test) |
  | m7 | Tiana's bio goes back to the one-line default | `PopulateInstructorsSeedsAllFiveWithTheirBios` |
  | n1–n4 | Each of the four headshot squares crops from the middle again | that component's "crops … from the top" spec |

### 13.6 Hand-testing walkthrough

**Setup.** Recreate the database with `knottyyoga_database_helper --recreate_database`. Then rebuild and start the server, and restart `ng serve` (13.3 changed four style sheets). Use a wide window.

1. Signed out: **About** → **Instructors**. Five cards, each with a headshot, and each head has room above it:
   - **Mason Bendixen**: "Mason founded Knotty Yoga in 2011…"
   - **Caleb Ault**: "Caleb has attended Knotty Yoga as a student since 2016…"
   - **Beth Cooper**: "Beth Cooper has 30 years of experience…", with "Feldenkrais Method®" and "Beth’s classes" spelled as written
   - **Gifford Cheung**: "Gifford has been attending Knotty Yoga since 2012…"
   - **Tiana Cheung**: "Tiana has been attending Knotty Yoga since 2008…"
2. Click **Tiana Cheung**. Her page shows the same headshot, top of the head included, and the same bio.
3. **Our Classes** (page headed **All Classes**): **Intermediate Partner Acro** shows the partner acrobatics picture with four people. **Knotty Yoga**, **Handstands** and **Beginner Partner Acro** show their pictures as before. **Beginner Knotty Yoga** and **Aerial Rope** show the grey picture icon instead.
4. Click the **Intermediate Partner Acro** card. The picture beside the description is the same one.
5. **Sign In** as `masonbendixen@gmail.com` / `pass`. The small round picture beside your first name, top right, is your old profile photo, not the new headshot.
6. **Admin** → **Manage Products** → **Instructors** tile: five rows, each with a round thumbnail showing the face, not the shirt.
7. *(13.7)* Back on **About** → **Instructors**, every headshot is as sharp as the photo itself: crisp edges and a readable "KNOTTY YOGA" on the shirts, with no blockiness. The same goes for the pictures on **Our Classes**.
8. *(13.8)* Sign out. **About** → **Our Providers**: **Caleb Ault**'s card shows his headshot, head at the top, the same picture as on Instructors.

### 13.7 Sharp pictures *(ui; added 10/20 after Mason's hand-test)*

Mason's review of 13.6, step 1: the headshots "look like they have been converted to a low res image and then blown up." **They had been.**
- The card asked the server for a picture fitting a **256 × 256** box. The server only shrinks a picture to fit, so a 3:4 headshot came back **192 × 256**. The card then stretched it to 256 wide, and a 125% display stretched it again, to 320.
- The server's resampler reaches 192 × 256 from 1500 × 2000 with plain bilinear sampling. At an 8× reduction that reads a few source pixels and skips the rest, which is what made the edges and the shirt lettering jagged.
- The old headshots hid both: they were smaller and closer to square.
- Ruled out: every served JPEG is a whole multiple of 1 KB, which looked like truncation. It is Boost GIL padding its last output block after the JPEG's end marker, and each file decodes cleanly.

Tried on the live local server before changing anything: asking it for a **768** box and letting the browser shrink the result to the card is indistinguishable from a careful resize of the original. So the fix is on the page alone, with nothing needed in `server_components`:
- [x] New `shared/utils/cover-photo-url.ts`: `coverPhotoUrl(table, id, squarePx)` asks for a box **three times** the square the picture fills. That leaves the short side of anything up to 3:2 at twice the square, which is enough for a 2× screen, and the browser does the rest of the shrinking. Its spec checks the factor against a 3:4 and a 3:2 picture.
- [x] Used everywhere a picture fills a square with `object-fit: cover`:

  | Where | Shown at | Was | Now |
  |---|---|---|---|
  | **Instructors** card | 256 | 256 box | 768 box |
  | An instructor's page | 192 | 256 box | 768 box (the card's own URL, so its download is reused) |
  | **Favorite Instructors** | 96 | 256 box | 288 box |
  | **Manage Instructors** thumbnail | 56 | 64 box | 168 box |
  | **Our Classes** (All Classes) cards | 180 | 256 box (256 × 171 for a 3:2 class photo) | 540 box |

- [x] Each component's spec now checks the new URL; Favorite Instructors gained one.
- Size: a 576 × 768 headshot is about 190 KB, so the Instructors page now downloads about 1 MB of pictures. The server keeps each size after it is first made, so only the first visitor waits for the resize.
- Not changed: the server's resampler (`ImageResize` in `server_components`). With the bigger boxes its shrinks are about 2.6×, where bilinear is fine. The header's 32 px profile avatar still asks for a 32 box; it is the old profile photo and too small to notice.
- No database change, so no recreate: restarting `ng serve` is enough. The new URLs are new files to the browser, so nothing stale is shown.

### 13.8 Our Providers shows the headshot *(server + ui; added 10/20 after Mason's hand-test)*

Mason: "The photo on the provider page isn't showing up." Signed out, **About** → **Our Providers** showed Caleb's card with a broken image.
- The card asked for Caleb's **account** photo (`people`). Account photos are deliberately not on the server's public photo list (`PublicPhotoTables` in `main.cpp`), because anyone could then fetch any member's photo by number. So a signed-out visitor got **401 Login required**.
- The page's own comment claimed `people` was public; it never was. Not caused by Phase 13: Caleb's account photo is the file it always was.
- The card's bio already came from Caleb's staff profile (`instructors`), which *is* public. The fix takes the picture from the same place: **his instructor headshot**.

- [x] Server: the directory query also returns the profile's id (`instructor_id`, 0 without one). `PublicProviderInfo.hasPhoto` now means that profile has a headshot, and the feed carries `instructor_id`. A provider with no staff profile shows the placeholder.
- [x] Page: `providerPhotoUrl` builds the headshot's URL at the Instructors card's size (13.7), so it is the file a visitor may already have. The card's square crops from the top (13.3).
- [x] Tests:
  - `ProviderDirectoryHelperTest.ThePictureIsTheInstructorHeadshotNotTheAccountPhoto`: an account photo alone shows nothing; a profile gives its id; its headshot is what shows.
  - `SeedAppDataTest.OurProvidersShowsCalebsHeadshot`, through the page's reader on the real seed.
  - The feed conversion and the endpoint carry `instructor_id`.
  - The page's spec checks the URL and the crop, and the local-mode mock carries the id.
- Account photos stay private: the end-to-end run checks the old URL still answers 401.

### 13.9 Verification of 13.7 and 13.8
- [x] knottyyoga: **Windows: 5392 passed. Linux: 5392 passed.** Same names on both; both programs build on Linux. On Windows every target but the server was built, because Mason's server was running; Linux builds the server.
- [x] Angular: **3627 passed** (3622, plus 2 for `coverPhotoUrl`, 1 for Favorite Instructors' URL, 1 for the provider card's crop and 1 for the mock's provider id). `tsc`, `ng lint` and `ng build` are clean.
- [x] End to end, signed out on a fresh install: Our Providers lists Caleb with his profile id and `has_photo`, and his headshot is served at the card's URL. His account photo's URL still answers 401.
- [x] Every one of the 8 new mutations is caught:

  | # | Broken on purpose | Caught by |
  |---|---|---|
  | n5 | Pictures requested at the size they are shown | `coverPhotoUrl` spec |
  | n6 | The Instructors card asks for a 256 box again | `InstructorsComponent` spec |
  | n7 | The provider card asks for the account photo again | `ProvidersComponent` spec |
  | n8 | The provider card crops from the middle | `ProvidersComponent` spec |
  | n9 | Local mode loses the provider's profile id | `ServerAccessMock` spec |
  | p1 | The directory checks the account photo again | `ThePictureIsTheInstructorHeadshotNotTheAccountPhoto` |
  | p2 | The feed drops `instructor_id` | `PublicProviderInfoToKeyValueTable` |
  | p3 | The query never finds the profile | that test and `OurProvidersShowsCalebsHeadshot` |

  Plus n1–n4 again (the crops), still caught. Every file was restored and the clean runs pass.

### 13.10 Site Theme was frozen by its colour pickers *(ui; added 10/20 from Mason's report)*

Mason: "The site theme page is really, really slow. Particularly the image carousels." (**Admin** → **Manage Products** → **Site Theme**, signed in as an admin.) Not caused by Phase 13, but found during it, so fixed here.

**Measured** in a real browser on the local site, signed in as the admin:

| Step | Before | After |
|---|---|---|
| Open Site Theme | 21.9 s, 17.5 s of it a frozen page | 2.4 s, mostly font downloads; 0.1 s frozen |
| Open **Image carousels** | 12.5 s | 0.25 s |
| Type 12 letters into a carousel's **Name** | 27.7 s; one keystroke froze 3.9 s | 0.76 s; worst keystroke 16 ms |
| Type 12 letters into a **Brand basics** field | 12.2 s | 0.76 s |

The server was never the problem: its requests total about 1.5 s either way.

**Cause**, from a CPU profile of three keystrokes (10.7 s):
- 5.9 s was the colour picker's `hexFor` → `resolveCssColor`, which created a **new canvas for every call**; 3.8 s more was garbage-collecting those canvases.
- Every picker builds its named-colour menu (148 chips, each with a tooltip) up front, even closed, and every chip's colour is a template binding re-evaluated on each change detection.
- The Colors tab holds dozens of pickers, and Material tabs build all their content at once. So one keystroke anywhere on the page, the carousels tab included, created thousands of canvases. Development mode then checks everything a second time, doubling it.

**Fix**, both in `controls/color-picker`:
- [x] `resolveCssColor` remembers each answer and reuses one canvas. A colour name's meaning never changes, so an answer once found stays right.
- [x] The palette is wrapped in `<ng-template matMenuContent>`, so its chips are built when the menu opens, not with the picker.
- Not changed: the other tabs still build up front. With the pickers cheap, they cost little, and lazy tabs would have meant rewriting the Site Theme specs that read those tabs without opening them.

**Tests** (`color-picker.component.spec.ts`):
- [x] A colour already resolved is answered without touching the canvas again.
- [x] Resolving 50 new colours creates at most one canvas.
- [x] With the menu closed no chip colour is computed; opening it builds them.

**Mutation check:** all three caught: answers not remembered (c1), a canvas per lookup (c2), and the palette built eagerly again (c3, by hand). Every file was restored.

**Results:** Angular **3630 passed**; `tsc`, `ng lint` and `ng build` clean. No server change.

---

Mason- Please add new phases and push the existing stuff down. I met with Ryan to get the photos and look at the website and he had some suggestions he would like to see. Please organize these into a phase or phases with related work.
- In the social links:
	- Get rid of Twitter from the set of options (just Facebook and Instagram)
	- Can you convert Facebook/Instagram/Twitter/Bluesky/tiktok can you convert them to the icons for each site instead of just showing Fa In Tw?
- Can you systemically go through the site, including the user, staff, and admin portals and make sure anytime you build a data driven list, it is paginated? For instance, I know a lot of things in the user portal show purchases, booked sessions, or even the classes for each instructor card and a lot of these are unbounded lists. I really don't want more than ten items to show on a given page. For some of those where it is a nice to have, five items might be good. Please validate by doing screen captures in a test browser.
- For the home page, for banner images, I'd like to have an option to have the image be centered.
- For each home page item, I'd like to have checkboxes to show on desktop only and another to show on mobile only. If neither is checked, they show up anywhere (current behavior). I really want this to be based on a width though instead of mobile desktop and have desktop be above a given window width and mobile to be less than that so that a small window on desktop gets the same benefits (and a big tablet gets the desktop experience)
- The upcoming classes on mobile with a logged in user has No upcoming classes but the calendar icon is clipped
- Let's get turn off showing memberships on the home page by default (leave the option to manually turn them off later in the admin UI -- this is just the default population of the database)
- Let's change the Come join the fun section on the home page to link to the memberships page instead of getting started (since we are removing memberships from the homepage by default)
- What is intro strip? Right now it is just an opaque option to choose for a home page section that shows text and the pride dumbbell. I'd like to have the SVG and text for this editable.
- It’d like the font for the intro strip text and the New Here? The Get Started… to be the same.
- Can we have monthly calendar view shrink things down so that it fits without side scrolling?
- If you click on my schedule in the user portal but don’t have a membership, No eligible classes overlaps with the calendar icon.

*→ Plan (10/21/2026): Ryan's eleven points are Phases 14–17, grouped by the code they touch. Verification and the production rollout move down to **18** and **19**.*

| Ryan's point | Where |
|---|---|
| Twitter out of the social links | 14.3 |
| Brand icons instead of "Fa In Tw" | 14.6 |
| Every data-driven list paged: at most 10 a page, 5 where that is enough, checked with screenshots | Phases 16 and 17 (D15, OQ-32) |
| An option to centre the home page's banner images | 14.2, 14.4, 14.5 (OQ-29) |
| Desktop-only and mobile-only home sections, decided by window width | 14.2, 14.4, 14.5 (D16, OQ-35) |
| Upcoming classes on a phone: the calendar icon is clipped | 15.1 |
| Memberships off the home page by default | 14.3 |
| "Come join the fun" links to Memberships | 14.3 |
| What is the intro strip? Make its picture and text editable | 14.1, 14.3–14.5 (OQ-34) |
| One font for the intro strip and "New here?" | 14.5 (OQ-31) |
| The month calendar fits without scrolling sideways | 15.2 (OQ-33) |
| My Schedule without a membership: the text overlaps the calendar icon | 15.1 |

**What the intro strip is.** It is the sentence under the top picture ("Knotty Yoga is an inclusive, high-level acrobatic fitness studio.") with the rainbow **SAFE SPACE** dumbbell badge under it.
- Both can be edited today, just not where Ryan looked:
  - the sentence is **Site Theme** ▸ **Copy** ▸ "Home intro sentence"
  - the badge is a web address typed into **Site Theme** ▸ **Brand basics** ▸ "Home badge image URL"
- The Page Content row only says where on the page the strip goes. Phase 14 moves both onto the row.

---

# Phase 14 — Home page and footer (Ryan's review)

**Planned 10/21/2026.** One `server_components` change (14.1), with its own push and pin bump as in 11.11. The rest is knottyyoga.

### 14.1 Accept SVG uploads *(server_components: platform/endpoints/upload_photo)*
- [ ] `/api/upload_photo/<table>/<id>/svg` is accepted alongside `jpeg` and `png` (`upload_photo.cpp:75`).
  - **Why.** The photo picker (`@honuware/ui/photos`) already offers SVG files and sends them as `svg`, and the server refuses them with "Unsupported image type. Must be 'jpeg' or 'png'". So today no admin can upload the intro badge (14.4), or replace the seeded Get Started artwork with another SVG.
  - **The body must be SVG text.** After an optional byte-order mark, whitespace, an XML declaration and comments, its first element must be `<svg`. Anything else sent as `svg` is refused with 400, so a mislabelled PNG can't be stored as a vector.
  - **Stored as it came, as a vector.** `ImageHelper::UploadAndAssociatePhoto` already accepts vectors, and `get_scaled_photo` already returns an SVG unscaled, with `nosniff`.
  - **Safe under the existing SVG rule** (Polish 11.1, `get_scaled_photo.cpp:175`):
    - every page draws these pictures with `<img>`, which runs nothing
    - opened directly, production's site-wide CSP (`script-src 'self'`) blocks any script inside
    - only someone who can already edit the row may upload (`RequireTableWriteAccess`)
- [ ] Tests in `upload_photo_test.cpp`:
  - an SVG uploads and reads back byte for byte, as `image/svg+xml`, through the scaled-photo endpoint
  - PNG bytes labelled `svg` are refused, and nothing is stored
  - `gif` is still refused
- [ ] Ship as 11.11 did. Mason commits and pushes `server_components`. knottyyoga and CommunityFinder then pin the new SHA, and both apps' gates pass on both platforms.

### 14.2 Three new switches on a home section *(knottyyoga: db_schema, migration, admin metadata)*
- [ ] `home_sections` gains three booleans. All default to FALSE, so every existing row behaves as today. They are named like the four `hidden_when_*` flags beside them.

  | Column | Editor label (14.4) | Meaning |
  |---|---|---|
  | `hidden_on_narrow_screens` | **Desktop only** | Hidden in windows narrower than 768 px (D16) |
  | `hidden_on_wide_screens` | **Mobile only** | Hidden in windows 768 px wide and wider |
  | `images_centered` | **Centre the photos when they don't fill the width** | A carousel row's photos sit in the middle (14.5) |

- [ ] Migration `app/0012_home_sections_screen_width_and_centering`: `ADD COLUMN IF NOT EXISTS` for each column.
  - Test, modelled on `HiddenWhenNotMemberAddsTheColumnAndIsIdempotent`: run it twice on a database without the columns, then once on a fresh one.
- [ ] Admin metadata in `create_database.cpp`: `PopulateAdminColumnDataInfo` and `PopulateAdminColumnFriendlyNames` entries for all three.
  - `home_sections` is already a top-level admin table, so the foreign keys hold. `create_database_test`'s metadata check covers it.
- [ ] Read path. `GetActiveHomeSections` is `SELECT *`, so the columns reach `/api/home_sections` with no code change. Tests:
  - in `home_sections_test.cpp`: a row written with each flag reads back with it
  - in the endpoint's test: the three fields arrive as booleans

### 14.3 Seed *(create_database.cpp `PopulateHomeSectionsImpl`; app_secret_values.cpp)*
- [ ] **Membership tiers off by default.** The row is still seeded, with `active` false, so **Page Content** can turn it on with "Show this section on the home page".
- [ ] **"Come join the fun!" links to Memberships.** The link becomes `/shop`, the page the top nav's **Memberships** opens, labelled **See memberships**. Today it is `/start`, labelled "Get started".
- [ ] **The intro row carries its own content:**
  - **Body:** the sentence, from the same constant as the `site_hero_headline` default, so the two can't differ.
  - **Title:** "This studio is a designated safe space", the badge's description for screen readers. Today it is hardcoded in the template.
  - **Photo:** `SafeSpace_icon.svg`, copied from `ui/src/assets/svg/` into `src/database_helper/img/` and attached as `svg`.
- [ ] **The photo carousel row** is seeded with `images_centered` true (OQ-29). Its carousel is seeded empty, so this shows only once photos are added.
- [ ] **Social links: Facebook and Instagram only** (`app_secret_values.cpp:61`). Their addresses are still the two sites' home pages (OQ-30).
- [ ] Tests, through the readers the pages use (D4):
  - `create_database_test`, after `PopulateHomeSections`: `GetActiveHomeSections` has no `membership` row, and the full list has it, inactive.
  - The banner's link and label.
  - The intro row's body and title, and its photo is the SVG byte for byte, typed `svg`, read through the photo reader.
  - The carousel row is centred, and no other row is.
  - The social-links value, read through the site-content reader: exactly two lines, Facebook and Instagram.

### 14.4 Page Content editor *(ui: pages/manage/page-content)*
- [ ] **Two more toggles** under the existing four. They are slide toggles like their neighbours, not checkboxes:
  - **Desktop only — hidden in windows narrower than 768 px**
  - **Mobile only — hidden in windows 768 px and wider**

  With both on, a warning says nobody will ever see the section, as the membership pair already does. The row list shows **Desktop only** and **Mobile only** badges, as it shows **Signed-in only**.
- [ ] **Carousel rows** get **Centre the photos when they don't fill the width**, under **Carousel**.
- [ ] **Intro rows** become editable like a content block:
  - an **Image** upload, labelled **Badge** (SVG, PNG or JPEG)
  - **Body**, labelled **Text**
  - **Title**, with the hint "Describes the badge for people using a screen reader."
- [ ] **Each built-in kind says what it is.** One line under **Kind** describes the chosen kind, for example "Intro strip: a sentence with a badge under it". This answers Ryan's question in the editor itself. It is one map of eleven short lines, and the same opacity applies to every built-in kind.
- [ ] **One place to edit the intro strip:**
  - **Site Theme** loses "Home intro sentence" (**Copy**) and "Home badge image URL" (**Brand basics**).
  - `SiteConfig` drops `heroHeadline`, `heroImageUrl` and `heroImageAsset`.
  - The framework's `site_hero_*` keys stay: CommunityFinder shares them. They are simply unused here.
- [ ] Specs:
  - each new control shows for its kinds only, and saves its column
  - the both-on warning, and the two badges
  - the kind descriptions
  - the two Site Theme fields are gone

### 14.5 Home page *(ui: home-page and its sections)*
- [ ] **Desktop only / Mobile only** (D16). `isVisible` drops `hidden_on_narrow_screens` rows below 768 px, and `hidden_on_wide_screens` rows from 768 px up.
  - One `matchMedia('(width >= 768px)')` with a change listener decides it, so narrowing or widening a window switches the sections live.
  - A hidden section is not drawn at all, so a desktop-only picture costs a phone nothing.
  - The width is the calendar's (`BELOW_MD_MEDIA_QUERY`, `calendar.types.ts:32`). It moves to `shared/utils/` so both read one constant.
- [ ] **Centred photos.** `app-image-carousel` gains a `centered` input.
  - When every photo fits, the strip sits in the middle.
  - When they don't fit, it scrolls from the left as today.
  - The Gallery page doesn't set it, so it is unchanged.
- [ ] **The intro strip** draws the row's **Text** and **Badge** instead of the two Site Theme settings. The badge's description is the row's **Title**. A row with no badge shows the sentence alone.
- [ ] **One font** (OQ-31). The Get Started band's "New here? …" line takes the intro strip's style: the display font, semibold, at the intro's size. Both use one shared class, so they can't drift apart again.
- [ ] Specs:
  - visibility at each width, with `matchMedia` stubbed, including a live change
  - the carousel centres only when nothing scrolls
  - the intro strip draws the row's text and badge, with the badge's description
  - both text lines compute to the same font family, weight and size
- [ ] Playwright, new `e2e/mobile-audit/layout/home.spec.ts`, at 390 and 1280 px:
  - a desktop-only row is absent on the phone and present on the desktop, and a mobile-only row the reverse
  - the screenshots are kept for 14.9

### 14.6 Brand icons in the footer *(ui: shared/components/footer)*
- [ ] The footer draws each social link as its brand's icon instead of the first two letters of its label (`footer.component.ts:27`).
  - Five networks are recognised by the link's address:

    | Network | Addresses |
    |---|---|
    | Facebook | `facebook.com`, `fb.com` |
    | Instagram | `instagram.com` |
    | X (Twitter) | `x.com`, `twitter.com` |
    | Bluesky | `bsky.app` |
    | TikTok | `tiktok.com` |

    Failing the address, the label decides.
  - Anything else keeps its two letters, so an unknown network still shows something.
  - A Twitter link shows X's mark, Twitter's current logo.
- [ ] **Where the icons come from:** five SVG paths from Simple Icons, which is public domain, in one file, `social-icons.ts`.
  - There is no new dependency: the package holds thousands of icons and this needs five.
  - Drawn inline in `currentColor`, so the theme's colour reaches them. They are fixed strings in the code, not studio content, so drawing them inline is safe.
  - Each link's `aria-label` stays its label, and the tap target stays 44 px.
- [ ] Twitter comes off too:
  - `site-config.service.ts`'s fallback list drops it, to match the seed.
  - Two code comments that already claim "the footer maps known labels to icons" become true.
- [ ] Specs:
  - each of the five is recognised by its address, and by its label
  - an unknown network falls back to two letters
  - a recognised link draws its icon and shows no letters
  - these replace the two specs that pin "Fa In Tw"

### 14.7 Mutation check
- [ ] Each change is broken on purpose, one at a time, and every one must fail a test. Planned:
  - the upload accepting non-SVG bytes as `svg`, or refusing a real SVG
  - the migration not adding a column
  - the membership row seeded active
  - the banner still linking to `/start`
  - the intro row seeded without its photo
  - Twitter back in the seed
  - a desktop-only row shown on a phone, or a mobile-only row shown on a desktop
  - the carousel centring when it should scroll
  - the intro strip reading Site Theme again
  - the two fonts differing
  - a known network falling back to letters

### 14.8 Verification
- [ ] `server_components` gates on both platforms, then knottyyoga and CommunityFinder against the new pin: Windows and Linux, same test names.
- [ ] Angular: `ng test`, `tsc` for the app and the specs, `ng lint` and `ng build`.
- [ ] **End to end:** `--install_schema` into a throwaway database, then the real server.
  - Signed out, the home feed has no membership row, and the banner points to `/shop`.
  - The intro row's badge is served as an SVG.
  - Signed in as an admin, an SVG uploaded through the endpoint comes back byte for byte.
  - `--migrate` on a database from before Phase 14 adds the three columns.

### 14.9 Hand-testing walkthrough

**Setup.** Recreate the database with `knottyyoga_database_helper --recreate_database`. Rebuild and start the server, and restart `ng serve`. Use a window at least 1280 px wide, and start signed out.

**Signed out: the home page and footer**
1. Home page. Under the top picture is "Knotty Yoga is an inclusive, high-level acrobatic fitness studio.", with the rainbow **SAFE SPACE** badge under it, as before.
2. Scroll the whole page. There are no membership cards anywhere on it.
3. The **Come join the fun!** picture's button reads **See memberships**. Click it: the page is headed **Memberships**.
4. Back on the home page, the black **Get Started** band reads "New here? The Get Started guide walks through…" in the same lettering, weight and size as the sentence in step 1.
5. Footer, under **Follow us on Social Media!**: exactly two icons, the Facebook "f" and the Instagram camera. There is no "Fa", "In" or "Tw", and no Twitter.

**Signed in: editing the rows**

6. **Sign In** as `masonbendixen@gmail.com` / `pass`. Go to **Admin** → **Manage Products** → **Page Content** tile, **Home sections** tab. The **Membership tiers** row carries a **Hidden** badge.
7. Click the pencil on the row whose kind badge reads **Intro strip**.
   - A line under **Kind** says what an intro strip is.
   - **Badge** shows the SAFE SPACE picture, and **Text** holds the sentence.
   - Change **Text** to `Knotty Yoga is an inclusive acrobatic fitness studio.`
   - Under **Badge**, **choose file**: `server\knottyyoga_server\src\database_helper\img\tier_icon_family.svg`.
   - Click **Save**.
8. Home page: the new sentence, with the family icon under it in place of the badge.
9. **Site Theme** tile → **Image carousels** tab → expand **Home page photos** → **Add a photo**. Choose any picture, for example from `D:\Knotty Yoga\2026\Oct\JPG\Edited`, then click **Add photo**. Repeat once more.
10. Home page: the two photos sit in the middle of the page, not at its left edge.
11. Back in **Page Content** → **Home sections**, edit **Why Knotty Yoga**: turn on **Desktop only** → **Save**. The row shows a **Desktop only** badge.
12. Home page: **Why Knotty Yoga** is there. Narrow the window to about the width of a phone. It disappears without a reload. Widen the window again and it comes back.
13. Edit **Types of classes**: turn on **Mobile only** → **Save**. On the home page it is gone at full width, and appears once the window is narrowed.
14. Edit **Additional health services**: turn on both **Desktop only** and **Mobile only**. A warning says nobody will ever see the section. Click **Cancel**.

---

# Phase 15 — Two layout fixes on phones

**Planned 10/21/2026.** All in `ui/`. Both were reproduced on the local site at 390 px on 10/21, signed in as Kit with no membership.

### 15.1 Empty-state icons keep their size *(ui: assets/styles/_patterns.scss)*
- **Cause.** The two reports are one bug.
  - `.empty-card` is a flex row with no rule for its icon, so the long sentence beside it squeezes the 24 px `mat-icon` to a sliver.
  - The icon's own `overflow: hidden` then cuts the glyph off.
  - On the home page's **Upcoming classes** band it reads as clipped. On **My Schedule** the sentence sits over what is left of it.
- [ ] **Fix:** `.empty-card mat-icon { @include icons.size(icons.$md); flex: none; }` in `_patterns.scss`. `.notice` already does this for its own icon (`_surfaces.scss:118`).
  - It also fixes every other `.empty-card` with an icon: Today's Classes, Attendance History, Class Check-In (twice), Student Notes and the style guide's demo.
- [ ] Specs:
  - `style-guide.component.spec.ts`, for the global class: the `.empty-card` demo in a 260 px-wide host keeps a 24 × 24 icon, wholly to the left of the text.
  - `upcoming-classes` and `my-schedule` specs: the same check on their own empty states, at phone width.
- [ ] Playwright, at 390 px, as a signed-in customer with no eligible classes: the home band and My Schedule, with the audit's overlap check. Screenshots kept for 15.5.

### 15.2 The month calendar fits the screen *(ui: pages/calendar/components/views/month-view)*
- **Today.** The month grid has a fixed 800 px minimum width (`month-view.component.html:31`, `min-w-[50rem]`) inside a box that scrolls sideways. On 10/21 it measured 808 px wide at both 390 and 768 px.
  - This was deliberate in the mobile pass (Mobile UI Updates 2.5, "Makeover OQ 19"), and `layout/calendar.spec.ts:21` asserts it. This item reverses it.
- [ ] **No minimum width.** The seven columns share whatever width there is (`repeat(7, minmax(0, 1fr))`), at every size.
- [ ] **Narrow day cells**, decided by a container query on the grid's own width so a spec can size it:
  - each class chip shows the class name only, in small type, cut off with "…"
  - the time and the status icons drop out; both are on **Day View** and on the class's own page
  - cells are shorter
  - up to three chips a day, then **+N more**, which opens that day in **Day View**
- [ ] **Tapping a date number** opens that day in **Day View**. Today it only selects the day, for the next switch of view.
- Unchanged:
  - From 1024 px up nothing changes: cells there are already wider than the threshold.
  - Phones still open on **Day View**. This is for someone who picks **Month View**.
  - **Week View** still scrolls sideways on a phone (OQ-33).
- [ ] Specs (Karma, month view):
  - with the host 358 px wide, the grid is no wider than the host, and chips show the name only
  - **+N more** appears after three chips, and opens **Day View** on that date
  - with the host 1000 px wide, the chips show the time and icons as today
- [ ] Playwright `layout/calendar.spec.ts`:
  - the "scrolls sideways" test becomes "fits without scrolling", at every size
  - the chip-icon test applies only where cells are wide
  - screenshots kept for 15.5

### 15.3 Mutation check
- [ ] Planned:
  - the icon rule removed
  - `flex: none` dropped, keeping the size
  - the 800 px minimum restored
  - compact chips at every width
  - **+N more** never shown
  - **+N more** opening the wrong date

### 15.4 Verification
- [ ] Angular: `ng test`, `tsc` for the app and the specs, `ng lint` and `ng build`.
- [ ] The Playwright layout specs at all seven audit sizes.
- [ ] No server change.

### 15.5 Hand-testing walkthrough

**Setup.** A fresh database, the server and `ng serve` running. Use a phone, or narrow the browser window as far as it goes.

1. **Sign In** as `kiteagle@gmail.com` / `kit`. Kit holds no membership on a fresh database.
2. Home page, the **Upcoming classes** band: "No upcoming classes you're eligible for. Add some from "My Schedule" first.", with a whole calendar icon to its left, not a sliver.
3. Click **Plan your weekly schedule →**. On **My Schedule**, **My Weekly Plan** reads "No eligible classes yet — talk to staff about a skill evaluation or upgrade your membership." The calendar icon is whole, and the text starts to its right without touching it.
4. Open the menu (☰) → **Your Calendar**. It opens on **Day View**. Click **Full Schedule**, then the **Day View** menu → **Month View**.
   - The whole month fits across the screen: Sunday through Saturday, with nothing to scroll sideways.
   - Each class shows as a short name. No day in the real schedule has more than three classes, so **+N more** doesn't appear; the specs cover it.
5. Tap a date number. **Day View** opens on that date.
6. Widen the window to full desktop width. **Month View** looks as it did before this phase, with each class's time and icons.

---

# Phase 16 — Paged lists, part 1: the pager, public pages and the user portal

**Planned 10/21/2026.** All in `ui/`, no server change (D15).

**What exists** (inventoried 10/21):
- Only four lists page today:
  - **Blog**: 5 a page, paged on the server
  - **Attendance History**: 25 a page, paged on the server
  - **Admin** ▸ **Blog Posts**: 10 a page, paged on the server
  - **Manage Data**: pages already
- There is no shared pager. The first two are hand-written, and the third is a Material paginator.
- About sixty lists show every row. Many Manage pages ask the server for "the first 100" (or 1,000, or 10,000) rows and silently drop the rest (Phase 17.1).

**The rules** (OQ-32):
- **Page size.** A page's main list shows **10** a page. A list inside a card, or one of several sections on a page, shows **5**.
- **The pager hides when everything fits on one page**, so a short list looks exactly as it does today.
- **Not paged**, each for a reason:

  | Kind of list | Examples | Why |
  |---|---|---|
  | Calendars, and lists already paged by date | The calendar views; the weekly **Our Classes** list; provider and room week grids; **Who's Teaching What**; **Today's Classes**; Check-In's 90-minute window | Already paged by date |
  | Ordered lists the studio arranges with ↑ ↓ | Page Content rows; Getting Started steps; carousel photos; product variants | Moving a row across a page boundary breaks, and these are short by nature |
  | Home-page teasers that already show a few and link to the rest | "View All Events"; "All my bookings" | Already short, with a link to the rest |
  | Content, transactions and pick lists | The About blocks; the cart, checkout and seat assignment; search results and drop-downs the server already caps (people search 10, attendance templates 25) | Not data lists to browse |
  | Reports and working screens | **Schedule Grid**; **Enrollment Trends**; **Open-Seat Heatmap**; **Refund Effectiveness**; the pricing matrix; the class check-in roster | Paging gets in the way |
  | Lists of edit forms | **Fonts**; **Locations** | Each row is a form |
  | Already paged elsewhere | **Manage Data** | Already pages |

### 16.1 The pager *(ui: shared/components/list-pager)*
- [ ] `app-list-pager`: **‹ Previous** · "1–10 of 23" · **Next ›**, with 44 px buttons, hidden when the list fits on one page.
- [ ] `pagedSlice(items, pageIndex, pageSize)` and a small `ListPaging` state:
  - it keeps the page in range when the list shrinks: deleting the last row of page 3 goes to page 2
  - it goes back to page 1 when a filter or search changes
  - it brings the list's top into view on a page change, so a phone user isn't left at the bottom
- [ ] `MAIN_LIST_PAGE_SIZE = 10` and `NESTED_LIST_PAGE_SIZE = 5`, in one place.
- [ ] **Grouped lists**, such as days or a series' sessions, page over their items. A group cut by a page boundary repeats its heading on the next page.
- [ ] **One look for every paged list.** The pager replaces the two hand-written ones (**Blog**, **Attendance History**) and **Blog Posts**' Material paginator.
  - All three keep paging on the server.
  - **Attendance History** asks for 10 a page instead of 25.
- [ ] Specs:
  - the helper: slices, the last partial page, an empty list, and clamping
  - the component: its labels, the ends disabled, hidden on a single page, the event it emits
  - at phone width, one row with no wrapped labels (count the label's lines, per the audit harness's note)

### 16.2 Many-row mock data *(ui: ServerAccess.mock, e2e harness)*
- [ ] `knottyyoga.mockManyRows=1` in local storage makes every list method the Phase 16 and 17 pages use return 23 rows. That is three pages at 10, and five at 5, so every page can be photographed mid-list. It is off by default, so no existing spec or audit changes. Tests in `ServerAccess.mock.spec.ts`.

### 16.3 Public pages
- [ ] Each of these gets the pager:

  | Page | List | A page |
  |---|---|---|
  | **About** ▸ **Instructors** | Instructor cards | 10 |
  | An instructor's page | **Classes**; **Upcoming sessions** | 5; 5 |
  | **About** ▸ **Gallery** | Carousels | 10 |
  | **About** ▸ **Our Location** | Locations | 10 |
  | **About** ▸ **Our Providers** | Provider cards | 10 |
  | **Our Classes** ▸ **All Classes** | Class cards (a tag filter goes back to page 1) | 10 |
  | A class page | **Series Runs**; **Upcoming Sessions** (the pager replaces "Showing the next N dates") | 5; 5 |
  | **Services** | Service cards | 10 |
  | **Upcoming Events** | Event cards; the series & workshops list above them | 10; 5 |
  | **Memberships** | Tier cards; other products | 10; 10 |
  | A product's page | Variants | 10 |
  | **Blog** | Posts (server-paged, new look) | 5 |
  | Home page | **Series & workshops**; signed in, **Upcoming classes** | 5; 5 |

### 16.4 The user portal
- [ ] Each of these gets the pager:

  | Page | List | A page |
  |---|---|---|
  | **My Purchases** | Purchases; inside each, items, entitlements and payments | 10; 5 |
  | A purchase | Items; entitlements; payments | 5 each |
  | **My Bookings** | **Upcoming**; **Past**; a series' sessions | 10; 10; 5 |
  | **My Subscriptions** | Subscriptions; **Shared With Me** | 10; 5 |
  | **Saved Cards** | Cards | 10 |
  | **Purchase Sharing** | Each of its five sections | 5 |
  | **Vouchers & Credits** | **Active**; **Past** | 10; 5 |
  | **My Skills** | Skills | 10 |
  | **My Schedule** ▸ **Upcoming** | Classes, grouped by day | 10 |
  | **My Schedule** ▸ **My Weekly Plan** | "Kept on your schedule" (the week grid itself is a calendar) | 5 |
  | **Attendance History** | Classes attended (server-paged) | 10 |
  | **Workshops & Series** | Offerings | 10 |
  | **Notifications** | Pending sign-up reminders | 5 |
  | **Favorite Instructors** | Instructors | 10 |

### 16.5 Screenshots (Ryan's check)
- [ ] New `e2e/mobile-audit/layout/pagination.spec.ts` visits every page in 16.3 and 16.4 with many-row data, at 390 and 1280 px. On each it checks:
  - no more rows than the page size
  - the pager reads "1–10 of 23"
  - **Next** moves on to "11–20", and the last page holds 3
  - the audit's overlap and overflow checks pass
- [ ] Page 1 and the last page of each are saved to `ui/e2e/mobile-audit/output/pagination/`. Claude reads every one, lists anything that looks wrong here, and keeps the folder for Mason and Ryan.

### 16.6 Mutation check
- [ ] Planned:
  - the pager showing every row
  - a page size of 25
  - the pager showing on a single page
  - a filter change not going back to page 1
  - deleting a page's last row leaving an empty page
  - a cut group not repeating its heading
  - **Attendance History** asking for 25

### 16.7 Verification
- [ ] Angular: `ng test`, `tsc` for the app and the specs, `ng lint`, `ng build`.
- [ ] The pagination spec at both sizes.
- [ ] No server change.

### 16.8 Hand-testing walkthrough

**Setup.** A fresh database, the server and `ng serve` running, with a window at least 1280 px wide.

1. Signed out: **About** → **Instructors** shows five cards and no pager. **Our Classes** → **All Classes** shows six cards and no pager. A list that fits on one page looks as it did.
2. **Sign In** as `kiteagle@gmail.com` / `kit`. Go to **Memberships** → **Knotty Yoga Silver Membership** → **Subscribe**.
   - Leave **Start month** as it is.
   - Enter the sandbox card `4111 1111 1111 1111`, any future expiry, CVV `111`.
   - Click **Subscribe & Pay $160.00**.
3. Home page → **Plan your weekly schedule →**. On **My Weekly Plan**, tick all five **Beginner Knotty Yoga** classes.
4. **Upcoming** tab: ten classes, grouped by day, then the pager "1–10 of N", where N is about 20. Click **Next ›**: the rest, starting from the eleventh. A day split across the two pages shows its heading on both.
5. Open `ui\e2e\mobile-audit\output\pagination\`: one phone and one desktop picture of page 1 and of the last page, for every list in 16.3 and 16.4.

---

# Phase 17 — Paged lists, part 2: the staff portal and Manage

**Planned 10/21/2026.** All in `ui/`, on 16's pager, rules and many-row mock.

### 17.1 The silent caps go *(ui)*
- **Today**, each list below asks `get_filtered_table_rows` for one page of a fixed size and never reads the total, so anything past the cap never appears. For example, a 101st product is missing from **Products**, and the 1,001st event session from **Event Sessions**.

  | Page | Cap |
  |---|---|
  | **Products** | 100 |
  | **Price Schedules** | 100 |
  | **Event Sessions** | 1,000 |
  | **My Sessions** | 1,000 |
  | **Subscriptions** | 1,000 |
  | **Entitlements** | 500, and its two 5,000-row joins |
  | **Event Payment Info** | 200 |
  | A subscription's billing history | 200 |
  | The pricing views | 100, 200 and 10,000 |
  | **Subscription Revenue** | 10,000. It totals what it reads, so past the cap the totals are silently wrong |

- [ ] Each asks for everything (page size 0, which the endpoint already treats as "all rows") and pages in the browser (D15).
  - Not changed: a product page's **Event Sessions**, **Active Subscriptions** and **Recent Purchases**, which deliberately show the latest 10 with a **View All** link.
- [ ] Specs: each page's request asks for everything, and its eleventh row is reachable on page 2.

### 17.2 The staff portal
- [ ] Each of these gets the pager:

  | Page | List | A page |
  |---|---|---|
  | **Event Sessions** (**My Sessions**) | Session cards (the year and month filters go back to page 1) | 10 |
  | Inside a session card | **Attendees**; **Staff**; price overrides | 10; 5; 5 |
  | **Person Skills** | Current skills; history | 10; 10 |
  | **Student Notes** | Notes | 10 |
  | **My Service Bookings** | **Upcoming**; **Past** | 10; 10 |
  | **Time Off Requests** | Requests | 10 |
  | **Shift Requests** | Requests; each one's affected clients | 10; 5 |

  **Check-In** and **Class Check-In** stay unpaged: they are working screens with their own search (OQ-32).

### 17.3 Manage
- [ ] Each of these gets the pager:

  | Page | List | A page |
  |---|---|---|
  | **Manage Products** dashboard | **Requires Attention** alerts | 5 |
  | **Products** | Products | 10 |
  | A product | Booking windows; entitlement summary | 5; 5 |
  | **Price Schedules** | Schedules | 10 |
  | **Event Sessions** | Session cards (nested lists as in 17.2) | 10 |
  | **Event Payment Info** | Bookings | 10 |
  | **Subscriptions** | Subscriptions | 10 |
  | A subscription | Billing history; Entitlement History | 10; 10 |
  | **Entitlements** | Entitlements; each one's seat assignments | 10; 5 |
  | **Service Providers** | Providers; a provider's availability blocks; schedule templates | 10; 10; 10 |
  | **Time Off Requests** | Pending requests | 10 |
  | **Scheduling Exceptions** | Exceptions, grouped | 10 |
  | **Shift Requests** | Pending requests; affected bookings | 10; 5 |
  | **Vouchers** | Vouchers; each one's redemptions | 10; 5 |
  | **Coupons** | Coupons; redemptions | 10; 5 |
  | **Bundles** | Bundles | 10 |
  | **Class Schedules** | Instances; each one's schedules; each schedule's slots; specialty costs | 10; 5; 5; 5 |
  | **Instructors** | Instructors | 10 |
  | **Membership Tiers** | Permissions | 10 |
  | **Skills** | Skills | 10 |
  | **Class Tags** | Tags | 10 |
  | **Attendance Templates** | A member's entries; exceptions | 10; 10 |
  | **Room Schedules** | Date overrides; product overrides | 10; 10 |
  | **Room Occupancy** | Rooms; each room's projections | 10; 5 |
  | **Site Theme** ▸ **Image carousels** | Carousels (their photos are ordered, so not paged) | 10 |
  | **Page Content** ▸ **Announcements** | Announcements | 10 |
  | **Admin** ▸ **Blog Posts** | Posts (server-paged; 16.1 swapped its paginator) | 10 |

- **Left as they are:**
  - A price schedule's embedded prices table. It already pages, 20 a page, but its buttons jump to **Manage Data**. That is a library control, so it is noted here rather than changed (OQ-32).
  - The two `/admin/event-session/…` pages. No link reaches them.

### 17.4 Screenshots
- [ ] 16.5's spec extends to every page in 17.2 and 17.3, with the same checks and the same folder. Claude reviews each picture.

### 17.5 Mutation check
- [ ] Planned:
  - a cap restored (`page_size` 1000)
  - **Subscription Revenue** reading 10,000 again
  - a nested list at 10 instead of 5
  - a session card's attendees unpaged
  - the year and month filters not going back to page 1

### 17.6 Verification
- [ ] Angular: `ng test`, `tsc` for the app and the specs, `ng lint`, `ng build`.
- [ ] The pagination spec.
- [ ] No server change.

### 17.7 Hand-testing walkthrough
- Written when 17 is built. It will use the Manage lists that have more than ten rows on a fresh database; the screenshots (17.4) cover the rest.

---

# Phase 14 — Verification

### 14.1 Build and test (Claude, both platforms; reported per platform)
- [ ] knottyyoga: Linux gate (`server/docker/build_and_test.sh`) and Windows (`x64-Debug`). Also build `knottyyoga_database_helper` and `knottyyoga_test_helper` explicitly, since the gate builds neither `main.cpp`.
- [ ] Phases 4 and 11 only: the `server_components` gates on both platforms. Phase 11 also: `honuware-web-components` `ng test` + build.
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

21. **Admin** → **Manage Products** → **Service Providers**: one card, **Caleb Ault**, with **Accepting bookings** on. Click **Templates**: one template, **Massage hours**, with five chips, "Monday: 11:00 AM – 3:00 PM" through "Sunday: 1:30 PM – 4:00 PM", at the hours in the massage table.
22. **Services** → **Massage** → **Book Now**.
    - In the week strip, click the next **Mon** (use **Next week** if needed). There is one provider card, **Caleb Ault**. The first hour header is **11:00 AM**, and no session starts so late that it would end after 3:00 PM.
    - The next **Wed** is greyed out and can't be clicked: no hours that day.
    - Click the next **Sun**: Caleb's earliest time is **1:30 PM**.

**Temporary password (Phase 5)**

23. In `knottyyoga_test_helper` (REPL), run `set_password --email=<Beth's email from OQ-1> --password=Welcome-2026`, or `sp` with the same arguments. It reports that Beth must choose a new password at next sign-in.
24. Sign out. **Sign In** as Beth with `Welcome-2026`. The site goes straight to **Change Password**, and clicking **My Classes** comes back to it. Fill in:
    - **Old Password**: `Welcome-2026`
    - **Reenter Old Password**: `Welcome-2026`
    - **New Password**: one of your choosing

    Click **Update**. **My Classes** now opens normally.

**Forgot password (Phase 11)**

25. Sign out. **Sign In** → **Forgot password?**
    - **Email**: `masonbendixen@gmail.com` → **Send reset link**. The page says a link was sent if the account exists.
    - Repeat with `nobody@example.com`. The page shows the identical message.
26. In the masonbendixen@gmail.com inbox, open **Reset your Knotty Yoga password**. The sender reads **Knotty Yoga**, not "Knotty Yoga and Spa". Click the link. The **Reset your password** page opens.
    - **New Password** and **Confirm New Password**: `Reset-2026` → **Set Password**. The Log In page says the password has changed.
    - Signing in with `pass` now fails; `Reset-2026` works.
    - Clicking the email's link a second time shows **This reset link is invalid or has expired.**

**Co-instructors (Phase 10)**

27. **My Classes** → **Full Schedule** → Tuesday 6:00 PM **Beginner Knotty Yoga** reads **with Gifford Cheung, Tiana Cheung**. **About** → **Instructors** → **Tiana Cheung** lists Beginner Knotty Yoga among her classes.

**Nothing sample-only (Phase 9)**

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
  - once Phase 11 is live and their real email is seeded, they click **Forgot password?** themselves
- [ ] Paste any bios that arrived late under **Admin** → **Manage Products** → **Instructors** (OQ-15).
- [ ] The public half of the Phase 14.3 walkthrough (steps 1–9) against `https://dv1tgxa9ok30f.cloudfront.net/`.

### 15.4 Every month: extend Caleb's bookable hours (OQ-8)
- [ ] **Admin** → **Manage Products** → **Service Providers** → **Caleb Ault** → **Templates** → **Massage hours** → **Generate Availability**. Then:
  - **From Date**: today
  - **To Date**: the last day of next month
  - **Facility**: Knotty Yoga Studio

  Click **Generate**. It reports "Generated N availability blocks", and "Skipped M days (already existed)" for days already generated, so pressing it early or twice is harmless (7.4; before that fix a second press doubled the hours). If this ever gets forgotten, the massage picker simply shows no times after the last generated day. A scheduled job is the follow-up if it becomes a chore.
- [ ] **After changing Caleb's hours with Edit Template, press Generate Availability straight away**, with the same three fields. Saving an edit replaces the template, and every hour generated from the old one goes with it, so until you press it **Book Now** offers no massage at all (OQ-23).

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

**OQ-1 — Emails for Beth Cooper, Gifford Cheung and Tiana Cheung.** `people.email` is required and unique, and the instructor digest and (after Phase 11) password resets go to it. **Default:** placeholders `beth.cooper@knottyyoga.local` (likewise for the other two), undeliverable like the scheduler account, so the code can land. Real ones are swapped in before Phase 15.
- Mason- I sent a message to all my instructors asking which email to use.
- → Plan: waiting. Paste the three addresses here when they arrive; they are one constant each in `seed_app_data.h`.
- → 10/16: Phase 5 shipped with the placeholders, so nothing waits on this. Swap them in before Phase 15. Until then the instructor digest and, after Phase 11, password-reset emails to these three go nowhere.

**OQ-11 — Where are the photos?**
- Mason- Taking the photos IS a phase. Please list this as an early phase that the later phases will use. I'll try to take the photos tonight.
- → Plan: Phase 1 is the shoot, with a shot list, framing notes and checkboxes. **Default hand-off:** drop the originals into `C:\Users\mason\Documents\Obsidian\Knotty Yoga\Photos\` named as in 1.1 (any format or size); Claude converts and checks them (1.2). Say if you'd rather use another folder.
- → 10/18: the photos aren't in yet, so attaching them moved from Phase 9 to **Phase 13**, just before Verification. Phases 9–12 don't use them and go first.
- → 10/20: six are in (all five headshots and Intermediate Partner Acro, from `D:\Knotty Yoga\2026\Oct\JPG\`) and attached in Phase 13. **Still to come:** Beginner Knotty Yoga, Aerial Rope, Massage and Studio (1.1 items 6, 8, 12, 13). Drop each in the same folder under its 1.1 name; it joins as file + row + test (13.1). Until then Phase 14.3's steps 1 and 9 will find those four without a picture.

**OQ-26 — A badge on the All Classes cards?** *(12.3's front-end check, 10/19.)*
- After 12.3 everyone sees all six classes on **Our Classes** → **All Classes**. Each card shows the name, description, tags and session count, but not whether you can attend.
  - The weekly schedule marks each row "Requires a membership" or "Prerequisite needed".
  - Each class page now names what it needs (12.1).
- **Default:** no badge on the cards. Every class needs a membership, so for a visitor or a non-member all six cards would carry the same badge. The class page, one click away, names the membership and the skill.
- **Alternative:** the weekly schedule's badge on each card, worked out for the viewer. For example, a Silver member would see Beginner Knotty Yoga unmarked and the other five marked. The class list would then have to carry each viewer's access: a change to the server and to the page.
- Say which.

**OQ-27 — An unapproved Gold member reads "Members only: Gold membership".** *(Found implementing 12.1, 10/19.)*
- The class page's **Pricing** says "Members only" whenever the viewer can't attend, whatever the reason.
- So on Handstands, Aerial Rope and Intermediate Partner Acro, a Gold member whom no instructor has approved reads "Members only: Gold membership", though they already have Gold.
  - Right below it, **Required Skills** shows the skill with a red mark, then "You're missing: Handstand" and "Talk to staff for evaluation."
  - Before 12.1 the same member read "Members only" and "Upgrade your membership to attend.", so this isn't new.
- **Default:** leave it. The Required Skills box says what is actually missing.
- **Alternative:** when the viewer has the membership and lacks only the approval, **Pricing** says so instead, for example "Included with your membership once an instructor approves you". The server already tells the two apart (the schedule badge's "members only" versus "needs a skill"). It would have to pass that on, and the page would need a fourth wording.
- Say which.

**OQ-28 — Six slips corrected in the bios.** *(Phase 13, 10/20.)*
- The plan says bios are seeded verbatim. Six things in the pasted text looked like slips, so the seed corrects them and the pasted text above is left as it was:
  1. Mason: "Univesrity of Utah" → "**University** of Utah".
  2. Mason: "many acrobatics teacher training" → "many acrobatics teacher **trainings**".
  3. Beth: "is the owner a solo practice" → "is the owner **of** a solo practice".
  4. Beth: "came to the studio for intro workshop" → "came to the studio for **an** intro workshop".
  5. Beth: the double space after "PLLC." and the trailing space are removed.
  6. Gifford: his bio began "**Tiana** has been attending Knotty Yoga since 2012…". Tiana's own bio says 2008, so this reads as Gifford's own line with the wrong name, and it now begins "**Gifford** has been attending Knotty Yoga since 2012…".
- Left as written: everything else, including "had been teaching acrobatics since 2004", "Feldenkrais Method®" and the curly apostrophe in "Beth’s".
- **Default:** keep the corrections.
- **Alternative:** seed any of them back exactly as pasted. Each is one string in `kSeedInstructors`, or an edit under **Admin** → **Manage Products** → **Instructors** once deployed.
- Say which, and especially whether 6 is right.

### Resolved

*Newest first: 10/20 (OQ-15), then 10/19 (OQ-25, 24, 23), then 10/17 (OQ-22), then 10/15 (OQ-16, 19, 20, 21), then the 10/14 answers.*

**OQ-15 — Instructor bios.**
- Mason- I asked my instructors to write bios. Leave a place for me to put them as they send them to me. Worst case, I manually add them once deployed but hopefully they get them to me before then.
- → Plan: the **Instructor bios** section above has a slot under each name. Whatever is there at Phase 5.5 is seeded verbatim, anything still empty gets its one-line default, and late ones go in through Manage after deploy (15.3).
- → 10/20: all five arrived and are in the seed (Phase 13), with OQ-28's corrections. 15.3's "paste late bios" step has nothing left to do unless one changes.

**OQ-25 — A plus-addressed email can't activate its account.** *(Found implementing Phase 11, 10/18.)*
- The activation mail's link writes the address with honuware's `URLEncode`, which leaves `+` as it is, so `mason+test@gmail.com` goes out as `?email=mason+test@gmail.com`.
- The SPA's router reads a `+` in a query string as a space. Checked against Angular itself: that link arrives as `mason test@gmail.com`, `/api/verify` finds no such account, and the person sees "Verification failed or expired".
- Plain addresses, which is nearly everyone, are unaffected. The new reset link (11.3) encodes `+` as `%2B` and works, so **Forgot password?** is the workaround: following its link verifies the account too.
- **Default:** fix it in the honuware component the next time it is opened, by building the activation link with the reset mail's encoder, with a test for a `+` address. It is not part of Phase 11.
- **Alternative:** fix it now, in the same `server_components` change as Phase 11 (one line plus a test), before Mason commits it.
- Say which.
- Mason- Let's fix it now.
- → Plan: **11.11**. Phase 11's first `server_components` change was already pushed (`4f72341`), so this is a second one with its own pin bump. The reset link's encoder moved into foundation as `URLEncodeQueryValue`, the activation mail uses it, and a test registers `mason+test@example.com` and activates it from the link the way the SPA reads it. Hand-test steps 13–15 in 11.10.

**OQ-24 — A shift trade on a co-taught class drops the co-instructor from that date.** *(Found implementing Phase 10, 10/18.)*
- A shift trade works by writing who teaches one date onto that date, the same record a substitution writes. Phase 10's rule is that such a record replaces the slot's whole team, which is right for a substitution ("both are away, Mason covers").
- For a trade it is not quite right. When Gifford asks to trade one Tuesday, the system records Gifford alone as teaching it, so from the moment he asks, that Tuesday shows only Gifford and Tiana disappears from it. Once someone takes the shift, it shows only them.
- Tiana herself can't trade a Tuesday: the request is refused with "You can only trade your own teaching shift", because only the lead holds the shift.
- **Default:** leave it. Trades on the one co-taught class should be rare, and an admin substitution (which names whoever covers) is the workaround.
- **Alternative:** when the lead asks to trade a co-taught date, record the co-instructors on that date too, so the trade replaces only the lead. That is a change to the shift-trade code, with tests.
- Say which.
- Mason- Let's go with the default.
- → Plan: no change. 10.3's "Not changed, deliberately" list says so.

**OQ-23 — Editing Caleb's hours removes his bookable times until Generate is pressed.** *(Found implementing Phase 7, 10/17.)*
- On **Service Providers** → **Templates**, **Edit Template** saves by deleting the template and creating a new one. The delete takes every availability row generated from the old template with it (7.5). Right after an edit, **Book Now** offers no massage at all until **Generate Availability** is pressed on the new template. Bookings already made are untouched.
- **Default:** no code change. 15.4 now says to press **Generate Availability** right after any edit. You will rarely edit the hours, and the fix costs one press.
- **Alternative:** the Templates page generates for you after **Save Template**, from today through the end of next month. That is a front-end change, with tests.
- Say which.
- Mason- I'll go with the default.
- → Plan: no change. 15.4 keeps its "press **Generate Availability** straight away" step.

**OQ-22 — Signed in, Our Classes hides every class you can't attend yet.** *(Found implementing Phase 6, 10/17.)*
- Signed out, **Our Classes** → **All Classes** lists all six classes. Signed in, it lists only the classes the person may already attend (`ClassCatalogHelper::GetClassesVisibleToPerson`, a deliberate rule from the access redesign, P-1/M-6). Until Phase 6 every class was open, so nobody noticed.
- Now that all six need a membership, a signed-in **non-member sees an empty page**, a Silver member sees only Beginner Knotty Yoga, and **so do the staff**: Mason's and the instructors' accounts hold no membership. The weekly schedule is unaffected: it lists all 14 rows with their badges, signed in or out.
- It also breaks 14.3 step 13, where a Silver Kit opens Knotty Yoga from **All Classes**.
- **Default:** signed-in visitors see every scheduled class, as signed-out ones do, and the class page and badges say what each one needs. New item 12.3. Say if a signed-in member should keep seeing only what they can book.
- Mason- Let's go with the default.
- → Plan: 12.3 stands as written. Everyone sees all six classes on **All Classes**, signed in or out, and the class page says what each one needs. Until 12.3 lands, a signed-in non-member or staff account still sees an empty **All Classes** page; 6.6's last step notes it.

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
  - Phase 11 is the forgot-password flow: a **Forgot password?** link on Log In, an email with a single-use link that expires in an hour, and a page to set a new password.

**OQ-3 — Gold for the non-beginner classes.** Mason- that's correct. → D7.

**OQ-4 — Approval skills.** Mason- Let's make skills for handstand, aerial_rope, intermediate_partner_acro. Say the things in there that I listed for each in the description. → D6 and the skill table: codes `handstand`, `aerial_rope`, `intermediate_partner_acro`, named for the skill, with your criteria as the descriptions. Knotty Yoga: see OQ-19.

**OQ-5 — Instructor role can check people in and grant approvals.** Mason- Yes, this sounds correct. → 5.4.

**OQ-6 — Mason as a massage provider.** Mason- Let's just do Caleb for now. → 7.1.

**OQ-7 — Co-instructors.** Mason- we should support multiple instructors. → Phase 10.

**OQ-8 — Rolling availability.**
- Mason- no no no... let's not make this a new feature. We can just add slots with the template. The generate availability feature is pretty simple. Could also do this as a scheduled job pretty easily if need be. Let's not complicate the webserver more than necessary.
- → Plan: the rolling-availability phase is deleted. Caleb's template is open-ended, the seed generates through the end of next month, and 15.4 is the monthly **Generate Availability** press with exact steps.

**OQ-9 — Getting this onto the live site.** Mason- No, I'm just in preview mode and still using the Square sandbox. Wipe away. → D11, Phase 15. Theme export/import is dropped; the `set_secret` block stays, because it carries the Square sandbox and mail settings.

**OQ-10 — Test accounts in the base seed.** Mason- I'll manually have the three of us change our passwords as we deploy. I'm not that concerned about this. → Kit and `masonbendixen@hotmail.com` stay in the base seed, unchanged (5.3). 15.3 is the password change for Mason (both accounts), Caleb and Kit.

**OQ-12 — Test-only data.** Mason- let's remove both of these. I'm getting ready to deploy and want the website to be as it will actually function. → Phase 9 and D14. With both gone and Caleb's template promoted to real data, nothing test-only remains, so `--recreate_database_test` is removed too.

**OQ-13 — Membership tier order.** Mason- I'd prefer the alternative. → D12, Phase 3: grouped by tier through a new `products.sort_order` column, editable as **Sort Order** in Manage Products.

**OQ-14 — Membership descriptions.** Mason- This sounds fine. → The Memberships table.

**OQ-17 — Tell visitors which membership a class needs.** Mason- let's make this change. → 12.1.

**OQ-18 — Who builds.**
- Mason- I trust you more now! Please update the note that you build and run the tests on windows and linux but report the results to me. When stuff would break, I'd just paste the output to you so I figure I trust you now to remove myself from this loop as your model has improved.
- → Plan: recorded in **Revision notes and working agreement** at the top of this doc and in Claude's standing memory. The Overview's "I will build…" line is left as you wrote it, since the Overview is yours, and is superseded by this answer.
