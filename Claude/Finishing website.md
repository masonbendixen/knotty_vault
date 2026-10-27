---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 10/27/2026
Version: 0.35
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

- **v0.35 (10/27/2026)** — **13.13 is built**, and not committed. Ryan's edits replace the six class photos and Massage. Knotty Yoga's is a different photo, and Aerial Rope's is now upright.
  - Eight new photos fill two photo-only carousels: the home page's (`GalleryKnottyYoga1–4`), and a partner acro one (`GalleryPartnerAcro1–4`) that only **About** ▸ **Gallery** shows, after the home one. Neither has a heading, a blurb or captions.
  - **The class page** asks for a taller box, so the upright Aerial Rope comes back 884 px wide instead of 497 and isn't stretched.
  - **Found on the way:** two pages lost a day across the 11/1 clock change, and two Angular specs had started failing this week.
    - The weekly schedule's **Next week** dropped **Saturday 11/7**, its 10:00 AM Knotty Yoga included, and its heading read "Nov 1 – Nov 6". Visitors would see this all week.
    - The schedule grid's default **To** showed one day early.
    - Both now count calendar days, with clock-pinned specs.
  - **New open question, OQ-42:** does "no gallery name" include the Gallery page's own **Gallery** title? **Default:** keep it.
  - **Results:** knottyyoga Windows 5409, Linux 5409, the same names on both. Angular Windows 3995, Linux 3995. End to end, signed out, both carousels come back with no title, description or captions, and every photo is served.
  - **Mutations:** the seed's 9 of 9 caught. The UI's 4 of 5 caught; the fifth can't be, because its fix is redundant once the week's end counts calendar days (13.13).
- **v0.34 (10/23/2026)** — **13.12 is built**, and not committed. On the home page, **Types of classes** now shows `Class.jpg` (Mason's 10/22 shoot) and **Additional health services** shows `Massage.jpg`, the same file as the **Services** card. Both replace the design's stand-ins.
  - Neither is cropped: a home feature shows its picture whole, so these two blocks are 4:3 and a little taller than the design's 1.58:1. The two stand-ins are deleted from `img/`.
  - **Results:** knottyyoga Windows 5408 passed, Linux 5408 passed, the same names on both. Mutations: 4 of 4 caught. End to end, signed out, both pictures are served at 797 × 598 at the page's own request.
- **v0.33 (10/23/2026)** — **5.8, 13.11 and 17.8 are built**, and not committed.
  - **5.8:** the three instructors' real addresses are in the seed. A new test finds all three through the staff people search.
  - **13.11:** the five photos are converted and attached, so every class and the Massage card have a picture.
    - The **Services** card shows its picture at 4:3 (OQ-40's default).
    - The massage photo couldn't run from above Caleb's head to his elbow at 4:3. The crop keeps his face, his fist and the elbow on the client's back, and cuts the top of his hair.
  - **17.8:** **Active Subscriptions** and **Recent Purchases** list every row, five a page. Names and entitlements are looked up only for the rows on screen.
  - **Found on the way:**
    - The planned "two classes share a file" mutation wasn't caught by any test. The photo test now checks that each class shows a different picture.
    - `ClassCatalogHelperTest.GetClassDetailUpcomingSessionsExcludeCancelled` failed on Linux. It added seven days of hours to a "now"-relative class start, which is an hour off in the week before daylight saving time ends on 11/1. Only the test was wrong. Unfixed, it would have failed CI until then.
  - **Results:**
    - knottyyoga: Windows 5407 passed, Linux 5407 passed, the same names on both.
    - Angular: Windows 3992 passed, Linux 3992 passed.
    - Playwright layout specs: 378 passed.
    - Mutations: 11 of 11 caught (5.8: 1, 13.11: 6, 17.8: 4).
    - End to end on a throwaway database: all six class pictures and the Massage picture served signed out.
  - **New open question, OQ-41:** on a phone, a product page's subscription cards wrap the name and the billing period. It predates 17.8. **Default:** leave it.
  - **Left to build before Verification:** 12.7.
- **v0.32 (10/23/2026)** — **Mason answered OQ-1, OQ-38 and OQ-39, and sent five more photos (OQ-11).** Planning only; no code changed.
  - **OQ-1:** the three instructors' real addresses. New item **5.8** puts them in the seed.
    - It also makes the three findable under **Search for a person**. That search hides the placeholders' `@knottyyoga.local` domain, the scheduler account's.
  - **OQ-11:** five photos are in.
    - New: **Beginner Knotty Yoga**, **Aerial Rope** and **Massage**.
    - Replacements: **Handstands** and **Beginner Partner Acro**.
    - New item **13.11** converts and attaches them, so every class gets a picture.
    - Still to come: the studio picture for **About** ▸ **Our Location**. OQ-11 stays open for that alone.
  - **OQ-38:** the default. Each table's columns keep sizing to the rows on its page.
  - **OQ-39:** the alternative. New item **17.8** pages a product page's **Active Subscriptions** and **Recent Purchases** five at a time.
  - **New open question, OQ-40:** the massage photo is upright, and the **Services** card shows a wide strip. **Default:** crop the photo to 4:3 and show the card's picture at 4:3 (13.11).
  - **Left to build before Verification:** 5.8, 12.7, 13.11 and 17.8.
- **v0.31 (10/23/2026)** — **Phase 17 is built**, and not committed. All in `ui/`.
  - **The silent caps are gone (17.1).** Every list that asked for 100 to 10,000 rows and dropped the rest now asks for every row. Page size 0 already meant that on the server. More turned up than the plan listed, such as a session card's attendees and the product filters (17.1). **Subscription Revenue**'s totals now count every charge.
  - **The pager is on 51 more lists**, across 33 staff and Manage pages.
    - Not paged: **Announcements**, which are ordered with ↑ ↓, and the pricing views, which are a report.
  - **Screenshots:** 212 pictures, all read. Two things the pager made worse are fixed:
    - a request's notice crowding the pager on **Shift Requests**
    - the **Past** heading against the pager on **My Service Bookings**
  - **Results:** Windows 3786 → 3984 passed, Linux 3984 passed. Playwright: 376 passed (the pagination spec's 120 included). 8 of 8 mutations caught.
  - **New open questions:** OQ-38 (table columns that shift from page to page) and OQ-39 (a product page's latest-10 lists have no **View All**).
- **v0.30 (10/23/2026)** — **OQ-36 and OQ-37 answered, and Phase 16 checked on Linux.**
  - **OQ-36 and OQ-37:** Mason kept both defaults, and both are moved to Resolved. My Bookings keeps its two pagers, and the three-column grids keep ten a page.
  - **Linux:** `ng test` as CI runs it failed one test: 14.10's headline sweep at 320 px. That test is in the Phase 14 commit already. The runner has neither Barlow nor Arial, and in its wider font the clause doesn't fit the line. The sweep now checks first whether the clause can fit, and lets off only widths under 360 px. No change to the page (16.7).
  - **Results:** Windows 3786 passed, Linux 3786 passed.
- **v0.29 (10/22/2026)** — **Phase 16 is built**, and not committed. All in `ui/`.
  - **The pager:** one pager, "‹ Previous · 1–10 of 23 · Next ›", now on 48 lists across 27 pages. It replaces the three earlier ones: Blog, Attendance History (now 10 a page) and the admin Blog Posts' Material paginator.
  - **Many-rows mock:** `knottyyoga.mockManyRows` gives every one of those lists 23 rows.
  - **Screenshots:** `layout/pagination.spec.ts` photographs page 1 and the last page of each list at 390 and 1280 px, 192 pictures in all. Every one was read. Two problems found there are fixed:
    - the pager's buttons moved as its label widened
    - several booked sessions sat border to border
  - **Results:** Angular **3786** passed; Playwright: pagination 54 and the other layout specs 256 passed; 9 of 9 mutations caught.
  - **New open questions:** OQ-36 (two pagers close together on My Bookings) and OQ-37 (ten a page in three-column grids).
- **v0.28 (10/22/2026)** — **Phase 15 is built**, and not committed. All in `ui/`.
  - **15.1:** an empty state's calendar icon keeps its 24 px and sits wholly left of the sentence, on the home page band, My Schedule and every other `.empty-card`.
  - **15.2:** the month calendar fits the screen. Below an 800 px grid, a day shows short class names, three at most, then **+N more**. Tapping a date number opens **Day View** on that date.
  - **Results:** Angular 3669 passed; Playwright layout specs at all seven sizes, 256 passed; 9 of 9 mutations caught. No server change.
- **v0.27 (10/22/2026)** — **14.10: two fixes from Mason's first look at Phase 14.** No `server_components` change.
  - **"THE RIGHT WAY." could still land on a line of its own.** The red line was gone, but the headline wrapped wherever the window put it: at 1680 px it broke right after "YOUR LIFE,". Now the headline's lines are balanced, and "life, the right way." is never split, at any width.
  - **The desktop top picture was pixelated.** The server served it 800 px wide (honuware's cap on every scaled picture) and the page stretched it 2–3×. The seed now raises the cap to 2048, so both top pictures come back at their files' full size.
  - A top picture no longer draws its body, and Page Content no longer offers a **Body** field for one.
  - **Results:** knottyyoga Windows 5405 passed and Linux 5405 passed, with the same test names. Angular 3656; Playwright 29; 8 of 8 mutations caught; end to end PASS.
- **v0.26 (10/22/2026)** — **14.1 is pinned in both apps.** Mason pushed `server_components` `920f18a` (green in CI). knottyyoga and CommunityFinder now pin it.
  - knottyyoga: Windows 5404 passed, Linux 5404 passed.
  - CommunityFinder: Windows 1921 passed, Linux 1921 passed.
  - The test names are the same on both platforms, and the five new framework tests now run on Windows too.
  - Still to do: Mason's commit of the two pin lines and the knottyyoga Phase 14 changes, and his hand-test (14.9).
- **v0.25 (10/22/2026)** — **Phase 14 is built**, and not committed. Still waiting: Mason's `server_components` push, then the pin bump.
  - **`server_components` (14.1):** an upload labelled `svg` is accepted when its bytes really are an SVG document. The check is in `ImageHelper`, so no other path can store a raster under the vector label.
  - **knottyyoga:**
    - the three switches (migration 0012, admin metadata, the page-content bundle)
    - the seed: two top pictures, the intro row's own content, memberships off, the banner to `/shop`, and the studio's Facebook and Instagram
    - the editor, the home page and the footer icons
  - **Results:**
    - `server_components`: Windows 1900, Linux 1900.
    - knottyyoga: Linux 5404 against the local checkout; Windows 5399 on the pinned SHA, the same names less the five new framework tests.
    - Angular 3654; Playwright 28.
    - 27 of 27 mutations caught; both end-to-end passes PASS.
  - **Found on the way:**
    - The server caps every scaled picture at 800 px. The new mobile picture is served at 800 × 379, and the desktop one at 800 × 243, which is shown about one and a half times that size on a wide screen. Nothing is changed for that here (14.8).
    - A stale `ui/.angular/cache` had dropped every `xl:` class from the stylesheet, so the header couldn't switch to its desktop bar. Clearing the cache fixed it (14.8).
- **v0.24 (10/22/2026)** — **Mason answered OQ-26 to OQ-35.** Planning only; no code changed.
  - **Four answers change the plan:**
    - **OQ-27:** a Gold member who lacks only an instructor's approval reads **Included with your membership once an instructor approves you** under **Pricing**. New item **12.7**, a server and page change, not built yet.
    - **OQ-29:** "centred" means the top picture, not the photo strip.
      - Its headline becomes one line, "Get into the best shape of your life, the right way.", with no separate red line.
      - Today's picture becomes desktop-only, and is not centred.
      - A second top-picture row, mobile-only, uses Mason's new photo with **Centre the picture** on.
      - The photo strip is left as it is. The changes are in 14.2–14.5, 14.7–14.9 and 19.1.
    - **OQ-30:** the seed carries the studio's real Facebook and Instagram pages (14.3), so 19.3 no longer needs a step to set them.
    - **OQ-31:** the intro sentence moves to the body font, Roboto, the "New here?" line's lettering (14.5).
  - **Defaults confirmed, so nothing changes:** OQ-26, 28, 32, 33, 34 and 35.
  - All ten answered questions move to *Resolved*. OQ-1 and OQ-11 are still open.
- **v0.23 (10/21/2026)** — **Ryan's review becomes Phases 14–17. Verification moves to 18, and the production rollout to 19.** This is planning only; no code changed except one test comment that names the rollout's step (now 19.4).

  | Phase | What |
  |---|---|
  | **14** | Home page and footer: desktop-only and mobile-only sections, a centred photo strip, an editable intro strip (text and SVG badge), one font for the intro and "New here?", memberships off by default, "Come join the fun!" to Memberships, Facebook and Instagram only, brand icons |
  | **15** | Two phone layout fixes: the squeezed calendar icons, and a month calendar that fits the screen |
  | **16** | Paged lists, part 1: one shared pager, public pages and the user portal, checked with screenshots |
  | **17** | Paged lists, part 2: the staff portal and Manage, and the Manage lists that silently stop at 100 or 1,000 rows |

  - **Both layout bugs reproduced on the local site at 390 px**, signed in as Kit. The two icon reports are one bug in the shared `.empty-card` style. The month grid has a deliberate 800 px minimum, which an earlier mobile pass set and a Playwright test pins; 15.2 reverses it.
  - **The paging inventory found about sixty lists.** Only four page today, and about ten Manage pages silently drop rows past a fixed cap. **Subscription Revenue**'s totals are among them.
  - **One `server_components` change:** SVG uploads (14.1). The picker already offers SVGs and the server refuses them.
  - Every reference to the old 14 and 15 now uses 18 and 19. As in v0.14, the notes below keep the numbers they were written with.
  - New design decisions **D15** (lists page in the browser) and **D16** (desktop/mobile by window width, at 768 px). New open questions **OQ-29–35**, each with a default.
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
5. **Nothing extends provider availability over time.** `ScheduleTemplateHelper::GenerateAvailability` is called only by the seed and by the admin **Generate Availability** button. The seed generates through the end of the following month. **Per OQ-8 that stays manual**: once a month someone presses Generate Availability on Caleb's template (19.4). The generator skips days that already exist, so overlapping ranges are harmless. It also ignores `effective_to_us`, so an open-ended template keeps working.
6. **Production does not pick up seed changes.** Prod was installed with `--install_schema` on 9/23, and every later deploy runs `--migrate`, which evolves the schema and never re-seeds. Phase 19 is how this content reaches the live site.
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
- **D9 — New people never get a password from source control.** Beth, Gifford and Tiana are created with a random, unrecorded password, generated at seed time and never logged. To let one of them in, Mason sets a **temporary password with the must-change flag** (Phase 5), and they choose their own at first sign-in. Once Phase 11 ships, **Forgot password?** works too, given a real email (OQ-1). The four existing accounts keep their committed dev passwords; Mason, Caleb and Kit change theirs on production at deploy time (OQ-10, 19.3).
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
- **D15 — Lists page in the browser, over what the page already loads** (Phases 16–17, Ryan's review).
  - Nearly every list already arrives whole, and many pages filter it in the browser. Paging on the server would mean moving every filter there too, since `get_filtered_table_rows` only matches exact values.
  - The lists that already page on the server (**Blog**, **Attendance History**, **Blog Posts**) keep doing so, on the shared pager.
  - The Manage lists that ask for "the first N rows" ask for all of them instead (17.1), so paging can reach every row.
  - A person's own history (purchases, bookings) grows slowly, and dozens of rows load at once without trouble. Server paging is the follow-up if one ever gets slow.
- **D16 — Desktop-only and mobile-only are decided by window width, in the browser** (14.5, Ryan's review).
  - The line is **768 px** (OQ-35, confirmed): the width at which the rest of the site switches to its phone layout (the `md` breakpoint), and where the calendar already decides between Day View and Month View.
  - A small desktop window gets the phone sections, and a tablet held either way gets the desktop ones. An iPad mini held upright, at 744 px, gets the phone sections.
  - It is decided in the browser with a live media query, not with CSS that merely hides. A hidden section is never drawn, so it fetches nothing and downloads no picture. Resizing a window switches sections without a reload.

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

*→ 10/20 (Phase 13): all five bios below are in the seed (`kSeedInstructors` in `seed_app_data.h`), replacing the defaults. Six slips were corrected on the way in, and the text below is left as pasted. The corrections are listed in **OQ-28**; on 10/22 Mason kept all six.*

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

The later phases use these: Phase 13 attaches them and Phase 18's walkthrough looks for them. Nothing else blocks on them (D10), so the phases below can proceed while the shoot happens.

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
- [x] 6. Beginner Knotty Yoga class → `BeginnerKnottyYoga.jpg` *(in 10/23; attached in 13.11)*
- [x] 7. Knotty Yoga class → `KnottyYoga.jpg` (replaces today's, if reshot) *(not reshot; today's stays. → 10/27: replaced by a different photo in Ryan's edits, 13.13)*
- [x] 8. Aerial Rope class → `AerialRope.jpg` *(in 10/23; attached in 13.11)*
- [x] 9. Handstands class → `Handstand.jpg` (replaces today's, if reshot) *(reshot, in 10/23; replaced in 13.11)*
- [x] 10. Beginner Partner Acro class → `PartnerAcro.jpg` (replaces today's, if reshot) *(reshot, in 10/23; replaced in 13.11)*
- [x] 11. Intermediate Partner Acro class → `IntermediatePartnerAcro.jpg`
- [x] 12. Massage, shown on the Services card for Massage → `Massage.jpg` *(in 10/23; upright, so see OQ-40; attached in 13.11)*
- [ ] 13. General studio, shown on About ▸ Our Location → `Studio.jpg` *(not in yet)*

### 1.2 Hand-off and conversion
- [x] Mason drops the originals into **`C:\Users\mason\Documents\Obsidian\Knotty Yoga\Photos\`** (OQ-11 default), named as in 1.1. Any extension and any size is fine. That folder is outside the git repo, so phone-sized originals never get committed by accident.
  - → 10/20: they came in at `D:\Knotty Yoga\2026\Oct\JPG\` instead: six JPEGs, named as in 1.1. Only the six JPEGs were used; the five `.heic` files in the folder above it were left alone.
  - → 10/23: the second batch is in `D:\Knotty Yoga\2026\Oct\Oct 22 Website Shoot\JPG\`: six JPEGs, named as in 1.1.
    - `IntermediatePartnerAcro.jpg` there is byte for byte the 10/20 original, so it changes nothing.
    - The twelve `.heic` files in the folder above it are left alone, as before.
    - → 10/23, later: a seventh, `Class.jpg`, joined them for the home page's **Types of classes** (13.12). Upright, 4:3, no EXIF or GPS.
  - → 10/27: **Ryan's edits** are in `D:\Knotty Yoga\2026\Oct\Oct 27 Ryan Editied\`: 15 PNGs. Seven replace the six class photos and Massage (Knotty Yoga's is a different photo), and eight are new, for the home page's carousel and the Gallery. Converted and attached in **13.13**.
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
  - → 10/23, the second batch, checked on the originals (13.11 converts them): all five are upright, sharp and well lit. None carries GPS or a rotation flag. **No reshoots.** Two notes:
    - **The four class photos are 4:3**, the class page's own frame, so the class page shows each one whole. **All Classes**' square cards cut the sides:
      - **Aerial Rope:** both flyers lose a foot at the edges.
      - **Beginner Partner Acro:** the two outer flyers lose a hand.
      - **Handstands:** the person at the left edge is cut.

      Acceptable as is.
    - **Massage is upright** (3:4). 1.1 asked for landscape only for the class photos. The **Services** card shows its picture as a wide strip, which would keep only a third of this one: **OQ-40**.
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
- **Not tagged, on purpose.** Production re-runs only `--migrate`, which never re-seeds (fact 6), so a `sandbox.11` would change nothing on the live site. The only shipped code is `set_password`, and it has nobody to act on until the reinstall creates the three instructors. Phase 5 reaches production with the one Phase 19 release (its first step).
- → 10/23: **5.8** (the instructors' real addresses) built, not committed.

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

### 5.8 The instructors' real email addresses *(seed; added 10/23 from OQ-1; built 10/23, not committed)*

Mason sent them on 10/23:

| Instructor | Email |
|---|---|
| Beth Cooper | `bethismoving@gmail.com` |
| Gifford Cheung | `giffordcheung@gmail.com` |
| Tiana Cheung | `tianabun@hotmail.com` |

- [x] `kBethEmail`, `kGiffordEmail` and `kTianaEmail` in `seed_app_data.h` become these. Every seeder and test already goes through the three constants, so nothing else in the seed changes.
  - *As built:* the three lines and their comment, which now says these are real mailboxes. Nothing else in the repo named the placeholders.
- **One more effect: the three can be found by staff.** The staff people search, behind **Search for a person** on **Person Skills**, hides every address ending in `@knottyyoga.local`, the scheduler account's domain. So under the placeholders none of the three could be found there. With their real addresses they can.
- [x] Test: each of the three is found by the staff people search, through the endpoint **Person Skills** calls. Under the placeholders it finds none of them.
  - *As built:* `StaffSearchFindsTheNewInstructorsByName` seeds the real people, signs in as a staff member, and searches as the page does. "Cooper" offers exactly Beth's address; "Cheung" offers exactly Gifford's and Tiana's.
- [x] Mutation: one constant put back to its placeholder fails that test.
  - *As built:* Beth's put back: `StaffSearchFindsTheNewInstructorsByName` fails, the only test that does. The header was restored byte for byte.
- [x] Verification: knottyyoga, Windows and Linux, with the same test names on both.
  - *As built,* run with 13.11: **Windows: 5407 passed. Linux: 5407 passed.** That is 5405 plus the two new seed tests, and the two runs executed the same 5407 names. On Linux the database helper, server and test helper all build, against the pinned honuware.
  - Mason's server was running on Windows, so the Windows server program wasn't rebuilt there. `main.cpp` didn't change; Linux built it.
- **From now on a local database holds three real people's addresses.** A local run that mails an instructor reaches them, for example a **Forgot password?** asked for under one of their addresses. The walkthroughs send mail only to `masonbendixen@gmail.com`.

**Hand-testing walkthrough.** From a fresh database (`knottyyoga_database_helper --recreate_database`), with the server and the Angular app running:
1. **Sign In** as `masonbendixen@gmail.com` / `pass`.
2. **Staff** → **Person Skills** card → **Search for a person**: type `Cooper`. **Beth Cooper** is offered. Before 5.8 nothing was.
3. Clear the box and type `Cheung`. **Gifford Cheung** and **Tiana Cheung** are both offered.

---

# Phase 6 — Classes, approval skills, access rules and the weekly schedule

**Implemented 10/17/2026; committed as `32c9235c`, not tagged.** Windows 5259 passed, Linux 5259 passed. The two runs executed the same 5259 test names: Phase 5's 5252, less the three per-class slot tests, plus ten new. The Linux database helper, server and test helper all build, and the log says `honuware : pinned SHA`. Like the rest of the seed, it reaches production with the Phase 19 reinstall (fact 6).

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

**Implemented 10/17/2026; committed as `74af2aa8`, not tagged.** Windows 5267 passed, Linux 5267 passed. The two runs executed the same 5267 test names: Phase 6's 5259, less the two 9–5 template tests, plus ten new. The first attempt at the full runs was stopped part-way when the machine ran low on memory, so they were run again one at a time. Like the rest of the seed, it reaches production with the Phase 19 reinstall; 7.4's fix ships with the next release of the server.

The hours are one table, `kSeedMassageHours` in `seed_app_data.h`, walked by the seeder and by its tests (D3).

### 7.1 Provider assignment *(seed)*
- [x] `PopulateProviderTypeAssignments`: Caleb only, `massage_therapist`, accepting bookings. Mason's assignment is dropped (OQ-6, confirmed).

### 7.2 Weekly hours *(seed)*
- [x] **Promote** the test-only `PopulateProviderSchedule` into the base seed as `Seed::PopulateProviderSchedules(…, nowUs)`. It still resolves Caleb and the studio by natural key and generates through `GenerateAvailability`. What changed:
  - The template is named **"Massage hours"**.
  - It is **open-ended** (`effective_to_us` NULL). Checked first: nothing reads a provider template's dates. The generator ignores them, and the Templates page neither shows them nor sends real ones (it creates templates with 0 and 0). NULL only says "no end".
  - It carries the five ISO entries above instead of Mon–Fri 9–5.
  - **Changed from the plan:** the ISO constants (`kIsoMonday` … `kIsoSunday = 7`, with the warning comment) sit in `seed_app_data.h` beside the table rather than inside the function, because the tests walk the same table.
- [x] Availability is generated from today's local midnight through `Scheduling::EndOfFollowingMonthUs(nowUs)` in the studio facility. Beyond that it is extended by hand each month (19.4), which 7.4 makes safe to repeat.
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
Found while checking 19.4. `ScheduleTemplateHelper::GenerateAvailability` skipped only days with hand-entered or blocked hours. A second press over days the template had already generated added a second copy of them. 19.4's monthly press starts from today, so it always overlaps the last one: following it would have doubled every day already generated, and **Book Now** would have offered every time twice. 19.4 also said pressing twice was harmless.
- [x] A day **this** template already generated is now skipped and counted in `days_skipped`, so the Templates page's "Skipped N days (already existed)" is true. Hours from a different template on the same day are still added, since that is a split shift. The check reuses `ProviderAvailability::GetAvailabilityForProviderOnDate`, with no new SQL.
- [x] Tests:
  - `schedule_template_helper_test.cpp`: `GenerateAvailabilityTwiceAddsNothingTheSecondTime`, `…OverAnOverlappingRangeAddsOnlyTheNewDays` and `…StillAddsAnotherTemplatesHoursToTheSameDay`.
  - `admin_schedule_templates_test.cpp`: `GenerateAvailabilityPressedTwiceSkipsTheGeneratedDays`, through the endpoint the Generate button calls.
  - `seed_app_data_test.cpp`: `MonthlyGeneratePressExtendsCalebsHoursWithoutDuplicates`, which runs 19.4's press on the seeded template, from today through the end of the month after next.
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
- [x] **Found: Edit Template wipes the generated hours.** The Templates page's **Edit Template** saves by deleting the template and creating a new one. The delete cascades: the template's entries **and every availability row generated from it** go with it. In the run above, all 32 days were gone afterwards. So after editing Caleb's hours, **Book Now** offers no massage until **Generate Availability** is pressed for the new template. Bookings already made are separate rows and are not touched. This is how the page has always worked; Phase 7 only makes it matter. OQ-23 has the default: a note in 19.4, and no code change.

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

**Implemented 10/18/2026; validated by hand and committed as `f12a4c69`.** Windows 5268 passed, Linux 5268 passed. The two runs executed the same 5268 test names: Phase 7's 5267 plus one new. Like the rest of the seed, it reaches production with the Phase 19 reinstall. That includes the sender name: it is a config-secret default, written once when a database is created, so the live site keeps "Knotty Yoga and Spa" until then.

### 8.1 Seed
- [x] `PopulateAboutSections`: the three bodies from the About page table, **verbatim**. Titles, ordinals and links unchanged. The "40 years … hootananny" and "climbing wall" placeholder copy is gone.
  - **Changed from the plan:** the blocks moved out of the function into one table, `kSeedAboutBlocks` in `seed_app_data.h`, which the seeder and its test both walk (D3), like the massage hours.
- [x] `PopulateFacilities`: the studio's `description` becomes the **About ▸ Our Location** text above (OQ-16). It is a named constant, `kStudioDescription` in `seed_app_data.h`, so the test reads the same value the seeder writes.
- [x] `kMailSenderNameValue` becomes **"Knotty Yoga"** (OQ-16), and moved from `app_secret_values.cpp` into `app_secret_values.h`. It is also the site's **studio name** (`Mail::LoadTenantBranding` sets `studioName = senderName`). So the same change fixes the footer's "© Copyright Knotty Yoga and Spa 2020" and email subjects such as "Welcome to Knotty Yoga and Spa - Your Account". The "2020" is hard-coded in `footer.component.html`, and nothing here changes it.
  - Eight tests asserted the literal "Knotty Yoga and Spa": `app_secret_values_test`, `attendance_template_helper_test` (twice), `class_series_helper_test`, `favorite_instructor_helper_test`, `weekly_digest_helper_test`, `card_expiring_notification_test` and `signup_reminder_helper_test`.
  - **Checked:** `class_checkin_helper_test`'s subject is built from the sender name. honuware's `QuickAccountHelper` writes `"Welcome to " + branding.studioName + " - Your Account"`; there is no separate site-name slot. That test now builds the expected subject from the constant too.
- [x] Not changed:
  - `ServerAccess.mock.ts`'s local-mode copies of the old location text and the old About blocks. They are fixture data, as the note closing Phase 12 says of the mock catalog.
  - The `site_about_markdown` default in `app_secret_values.cpp`, which still carries the "40 years … hootananny" text. Nothing on this site has displayed it since Polish 7.3 replaced it with the About blocks. It still rides along unseen in the site-settings data the browser loads, and in theme bundles, which Phase 19 doesn't use (OQ-9). Say if you want it changed anyway.

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
2. The email sender is checked in Phase 18.3, step 26, which needs real mail.

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
- Production needs nothing beyond the Phase 19 reinstall. An upgraded database gets the table from `--migrate` (0011), which 10.8 checked.

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

Fact 12: this is built in the two honuware repos and then pinned here. **Nothing else in this plan depends on it.** The production rollout (Phase 19) goes ahead with or without it; until it ships, instructors get in with 5.2's temporary password. It follows the same build → push → pin cycle Phase 4 establishes, and touches some of the same files, so it starts after Phase 4's pin bump rather than in parallel.

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

**Implemented 10/19/2026 and committed.** It is all in knottyyoga, with no `server_components` change and no pin bump. **12.7 was added on 10/22 from OQ-27 and is not built yet.**
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
    - The page's chips, held/missing marks and "You're missing" box were built and tested, but never had data. 6.6 step 6 and the 18.3 walkthrough both expect a **Handstand** chip.
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
  - **A badge on the All Classes cards:** not added. That was **OQ-26**, and on 10/22 Mason kept the default of no badge.

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

### 12.7 Pricing when only the approval is missing *(added 10/22 from OQ-27; not built)*

**What changes.** Take a viewer who already holds the class's membership and lacks only an instructor's approval. Today **Pricing** tells them "Members only: Gold membership". It will read **Included with your membership once an instructor approves you** instead.
- The **Required Skills** box under it already names the missing skill and says "Talk to staff for evaluation."
- Everyone else reads what they read today: a visitor, a non-member, and a Silver member on a Gold class. 18.3 step 16 stays as written.

- [ ] **Business logic** (`ClassCatalogHelper::GetClassDetail`). A new `ClassPriceInfo.awaitsApproval` is set when all of these hold:
  - the access gate refuses the viewer
  - every group they fail is an approval, meaning a group with no permission in it (12.1's rule)
  - the class is recurring
  - **How:** compare the gate's `failedGroupIds` with the groups that `GetClassRequirements` already returned earlier in the same function. That needs no new query.
  - **Recurring only.** For a recurring class, passing the gate means "included". A workshop or series whose only gap is an approval still has a price to settle, so it keeps today's wording. No seeded class is one.
  - **Not `Annotate`.** It reports "needs a skill" whenever any skill group fails, even when the membership is missing too, so it can't tell that the approval is the only gap.
  - Tests in `class_catalog_helper_test`:
    - A viewer who holds the class's permission but not its skill: set.
    - A viewer lacking both: not set.
    - A viewer lacking only the permission: not set.
    - A failed "permission or skill" group: not set, because it counts as a membership.
    - A viewer who passes: not set, and the class still reads as included.
    - A workshop whose only failed group is a skill: not set.
  - Through the seeded rules, in `seed_app_data_test`:
    - A Gold member with no approvals: set on Handstands, Aerial Rope and Intermediate Partner Acro.
    - A Silver member on Handstands: not set.
- [ ] **Key-value table and endpoint.**
  - `ClassDetailToKeyValueTable` writes `price_awaits_approval` beside `price_is_available`, and `/api/classes/<id>` carries it as a boolean, always present.
  - Tests in `scheduling_key_value_table_test` and `get_class_detail_test`.
- [ ] **Frontend** (`class-detail`).
  - `ClassDetail.price_awaits_approval` is added. The network layer and the mock default it to false.
  - `pricingState` gains `'awaits_approval'`, checked before `'members_only'`. It shows the one line above, with no "Members only".
  - Specs:
    - the new line appears, and "Members only" does not
    - a members-only class without the flag is unchanged
    - the mock's default, in `ServerAccess.mock.spec.ts`
- [ ] **Mutation check**, planned:
  - the flag set when the membership is missing too
  - the flag set for a workshop
  - the key-value table or endpoint dropping it
  - the page ignoring it
- [ ] **Verification.**
  - knottyyoga on Windows and Linux, with the same test names on both.
  - Angular: `ng test`, `tsc` for the app and the specs, `ng lint` and `ng build`.

**Hand-testing walkthrough.** Recreate the database with `knottyyoga_database_helper --recreate_database`. Rebuild and start the server, then restart `ng serve`. Use a window at least 1280 px wide.
1. **Sign In** as `kiteagle@gmail.com` / `kit`. Kit holds no membership on a fresh database.
2. **Our Classes** → **All Classes** → **Handstands**. **Pricing** reads **Members only: Gold membership**, as before.
3. **Memberships** → **Knotty Yoga Gold Membership** → **Subscribe**.
   - Leave **Start month** as it is.
   - Enter the sandbox card `4111 1111 1111 1111`, any future expiry, CVV `111`.
   - Click **Subscribe & Pay $200.00**.
4. **Our Classes** → **All Classes** → **Handstands**.
   - **Pricing** reads **Included with your membership once an instructor approves you**, with no "Members only".
   - **Required Skills** shows the **Handstand** chip with a red mark, and "You're missing: Handstand".
5. **All Classes** → **Knotty Yoga**. It needs no approval, so **Pricing** reads **Included with your membership**, then "Just show up — staff will check you in."
6. Sign out: **Kit** (the name menu) → **Sign Out**, then **Log Out** in the dialog. Signed out, **Handstands** reads **Members only: Gold membership** again.

---

# Phase 13 — Attach the photos

*Moved here from Phase 9 on 10/18/2026. The photos aren't in yet, and nothing before Verification needs them, so the four phases that don't use them went first. The items are unchanged; only the numbers moved.*

Uses Phase 1's files.

**Status: implemented 10/20/2026, and committed as `13a7d74d`.** Six photos and all five bios are in. Four photos were still to come (1.1: Beginner Knotty Yoga, Aerial Rope, Massage, Studio); each joins later as file + row + test expectation (13.1). Results are in 13.4.
- → 10/23: Beginner Knotty Yoga, Aerial Rope and Massage are in, with new Handstands and Beginner Partner Acro photos. **13.11** attaches them: built 10/23, not committed. Only the studio picture is still to come (OQ-11).
- → 10/23, later: the home page's **Types of classes** and **Additional health services** show `Class.jpg` and `Massage.jpg` in place of the design's stand-ins. **13.12**, built 10/23, not committed.
- → 10/27: Ryan's edits replace the six class photos and Massage, and seed two photo-only carousels: the home page's, and a partner acro one on **About** ▸ **Gallery**. **13.13**, built 10/27, not committed.

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
  - [x] classes: all six. **Four are in:** Knotty Yoga, Handstands and Beginner Partner Acro keep their pictures, and Intermediate Partner Acro is new. Beginner Knotty Yoga and Aerial Rope wait for their photos. *(→ 10/23: all six, with new Handstands and Beginner Partner Acro pictures; 13.11.)*
  - [x] product `massage` → `Massage.jpg` *(→ 10/23: 13.11)*
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
- [ ] Massage and studio through their public readers: the catalog product's `has_photo`, and the facility's photo on the location reader. *(With their photos. → 10/23: massage's is done in 13.11, `ServicesShowsTheMassagePhoto`; the studio's waits for its photo.)*

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

### 13.11 The second batch of photos *(seed + ui; added 10/23 from OQ-11 and OQ-40; built 10/23, not committed)*

Five photos from Mason's 10/22 shoot, checked in 1.2:

| File | | Where it shows |
|---|---|---|
| `BeginnerKnottyYoga.jpg` | new | Beginner Knotty Yoga's card and class page |
| `AerialRope.jpg` | new | Aerial Rope's card and class page |
| `Handstand.jpg` | replaces today's | Handstands' card and class page |
| `PartnerAcro.jpg` | replaces today's | Beginner Partner Acro's card and class page |
| `Massage.jpg` | new | the Massage card on **Services** |

**Files**
- [x] Convert each into `src/database_helper/img/` as 1.2 did: ffmpeg, long edge 2000 px, `-q:v 3`, all metadata stripped. The four class photos come out 2000 × 1500.
  - *As built:* all four are 2000 × 1500: Beginner Knotty Yoga 469 KB, Aerial Rope 428 KB, Handstands 220 KB, Beginner Partner Acro 289 KB. All four were opened after conversion: upright, as framed in 1.2.
  - None carries EXIF, GPS or a colour profile. All that's left is ffmpeg's encoder note.
- [x] **Massage** is first cropped to 4:3 (OQ-40's default): the full width, from just above Caleb's head down to his elbow on the client's back. It also comes out 2000 × 1500.
  - *As built:* both ends don't fit. At the full width, 4:3 keeps 3213 of the original's 5712 rows, and his hair to his elbow spans about 3590. Three offsets were tried: 450, 625 and 800 px from the top. **800** won: his face and glasses, the raised fist, and the elbow pressing into the client's back, with the client's shoulders below it. Only the top of his hair is cut. 178 KB.
- [x] Copy the new files by hand into the `img/` folders beside the database helper and the test program, on both platforms, and hash-check them. Replacing a picture relinks nothing, so the build's own copy never runs (the trap in 1.2).
  - *As built:* Windows by hand, all five hashes matching the source. Linux needed no hand copy: this change edits the seed and its test, so both programs relink and the build's own copy runs. The hashes were checked there too (13.11 Verification).

**Seed**
- [x] `kSeedClassPhotos` gains Beginner Knotty Yoga and Aerial Rope, so all six classes have a picture. Handstands and Beginner Partner Acro keep their rows; only their files change.
  - *As built:* the table is in the weekly schedule's class order now, and its header no longer says two are missing.
- [x] `kSeedProductPhotos` gains `massage` → `Massage.jpg`. The stored type follows the extension (13.1), so it is stored as a JPEG.
  - *As built:* keyed by `kMassageProductCode`, like the rest of the seed's massage rows.

**Services card** *(ui: `shop/service-catalog`; OQ-40's default)*
- [x] The card's picture is 4:3 instead of a 160 px strip, still the card's full width: about 375 × 280 on a desktop.
  - *As built:* `aspect-ratio: 4 / 3` with `object-fit: cover`, so a picture of another shape uploaded later is cropped to the same frame.
- [x] It asks for its picture through `coverPhotoUrl`, like the class cards (13.7), so it stays sharp on a 2× screen. Today it asks for an 800 × 450 box.
  - *As built:* the widest the card gets is just under 576 px: one column, just before the grid splits in two. The square passed is 384, so the 3× box is 1152 px, twice that width. A desktop card (376 px) gets three times its width.

**Tests**
- [x] `PopulateSeedPhotosPutsEachListedFileOnItsRow`: no class is left without a picture, so its expected leftovers become none. It already compares each stored picture byte for byte with its listed file, which covers the two replaced files.
  - *As built:* every class is checked, and a class has a picture exactly when the table lists it.
  - **Added: each class's picture is a different one**, read back through the same reader. The byte comparison alone can't see one file listed against two classes: each row then matches its own listing. That is the planned "Aerial Rope given Beginner Knotty Yoga's file" mutation, which nothing caught until this check.
- [x] Massage through its public reader (13.2's open item): the **Services** feed's Massage product has `has_photo`, and its picture is `Massage.jpg` byte for byte.
  - *As built:* `ServicesShowsTheMassagePhoto` reads the public catalog, the feed the **Services** page reads. Massage is there as a bookable service, with `has_photo`.
  - Its stored picture is the file byte for byte, as a JPEG, and **4:3**. That last check is what pins the crop: an upright file fails it.
- [x] `EverySeedPhotoFileExists` covers the new files without a change.
- [x] The Services card's spec: the picture is 4:3 at the card's width, and its address is `coverPhotoUrl`'s.
  - *As built:* measured with the page 320 and 560 px wide: the picture fills the card inside its border, and its height is three quarters of its width. A fixed height can't pass at both.
  - A second spec checks that the box asked for is at least twice the widest card.

**Mutation check**
- [x] Planned:
  - Massage's row removed
  - Aerial Rope given Beginner Knotty Yoga's file
  - Aerial Rope's row removed
  - the card's picture back to a 160 px strip
  - the card asking for 800 × 450 again
  - *As built:* **6 of 6 caught**, the planned five and one more, each on its own run, and each file restored byte for byte. The seed ones ran on Linux:
    - Massage's row removed: `ServicesShowsTheMassagePhoto`
    - Aerial Rope given Beginner Knotty Yoga's file: `PopulateSeedPhotosPutsEachListedFileOnItsRow`, by its new "each class its own picture" check. Without that check nothing caught it (Tests, above).
    - Aerial Rope's row removed: the same test
    - **Added:** `Massage.jpg` uncropped, the upright original, in place of the 4:3 file: `ServicesShowsTheMassagePhoto` ("1500 x 2000 is not 4:3")
    - the card's picture a 160 px strip again: the card's 4:3 spec
    - the card asking for 800 × 450: the card's address spec

**Verification**
- [x] knottyyoga, Windows and Linux, with the same test names on both; Angular `ng test` on both, `tsc`, `ng lint`, `ng build`, and the Playwright layout specs. The desktop compare will report **Services** as taller, which is expected.
  - *As built,* knottyyoga: **Windows: 5407 passed. Linux: 5407 passed**, the same names on both (5.8 has the details).
    - The first Linux run failed one test, `ClassCatalogHelperTest.GetClassDetailUpcomingSessionsExcludeCancelled`, a date bug in the test and not in this change. It expected a class a week after one starting 10/25 at "the first start plus seven days of hours". Across the end of daylight saving time on 11/1 that lands at 9:00, while the page correctly shows 10:00. The test now finds the second start the way it found the first, as 10:00 at the studio on that date. Unfixed, it would have failed every run, CI included, until 11/1.
    - The new `img/` files were hash-checked beside both programs on both platforms.
  - *As built,* Angular and Playwright: see 17.8, which ran with this. **Windows: 3992 passed. Linux: 3992 passed.** Layout specs 378 passed, none failed.
  - The desktop audit of **Services**: 1280 × 800 → 1280 × 807, taller as expected. The mock's one pictured service now has a 376 × 282 frame where the strip was 160 px tall. The other pages the compare lists were already different from Phase 0's baseline before this change.
- [x] End to end: `knottyyoga_database_helper --install_schema` into a throwaway database, then the real server.
  - The log attaches six class pictures and the massage picture, with no `seed_image_missing`.
  - Signed out, all six classes serve a picture, and the Services feed has Massage with `has_photo`.
  - *As built,* on Linux, into a throwaway `knottyyoga_e2e_1311`, dropped afterwards:
    - The install log attaches all six class pictures and `Massage.jpg`. There is no `seed_image_missing`, `seed_image_failed` or `seed_lookup_missing` line.
    - Signed out, each class serves its picture at the **All Classes** card's request. The four new files come back 540 × 405. Intermediate Partner Acro (540 × 490) and Knotty Yoga (540 × 361) are as before.
    - `/api/catalog_products` lists Massage with `has_photo` true. The Services card's own request comes back **1152 × 863**: 4:3, and twice the widest card.
    - The server refuses to start with an empty mail password, so the install was given a dummy one. Nothing was mailed: the run only reads, signed out.

**Hand-testing walkthrough**

**Setup.** Recreate the database with `knottyyoga_database_helper --recreate_database`. Rebuild and start the server, and restart `ng serve`. Use a wide window.
1. Signed out: **Our Classes** (page headed **All Classes**). All six cards have a photo, and none shows the grey picture icon:
   - **Beginner Knotty Yoga:** a class standing in red aerial fabrics, arms out
   - **Aerial Rope:** two people hanging upside down on ropes
   - **Handstands:** handstands with spotters, against a white wall
   - **Beginner Partner Acro:** five people balanced on the feet of five lying on the floor
   - **Intermediate Partner Acro** and **Knotty Yoga:** the same pictures as before
2. Click the **Aerial Rope** card. The picture beside the description is the whole photo: both people, the ropes and the floor.
3. **Services** (page headed **Our Services**). The **Massage** card's picture shows both Caleb's face and his elbow on the client's back.
4. Make the window as narrow as it goes. The Massage picture still shows both.

### 13.12 The home page's two feature photos *(seed; added 10/23 from Mason's request; built 10/23, not committed)*

Mason, 10/23: "On the home page, for the Additional health services photo, please use Massage.jpg. For the Types of classes photo, please use `D:\Knotty Yoga\2026\Oct\Oct 22 Website Shoot\JPG\Class.jpg`."

| Home section | Was | Now |
|---|---|---|
| **Types of classes** | `home_class_types.jpg`, the design's stand-in | `Class.jpg`: a class stretching in the red aerial fabrics |
| **Additional health services** | `home_health_services.jpg`, the design's stand-in | `Massage.jpg`, the same file as the **Services** card (13.11) |

**Files**
- [x] Convert `Class.jpg` into `src/database_helper/img/` as 13.11 did: ffmpeg, long edge 2000 px, `-q:v 3`, all metadata stripped.
  - *As built:* 2000 × 1500, 397 KB. The original (5712 × 4284) carries no EXIF, GPS or rotation flag. It was opened after conversion: upright, the whole class in frame.
- [x] Neither picture is cropped. A home feature shows its picture whole, at the picture's own shape (`feature-section`: full column width, height auto). Both are 4:3, so these two blocks stand a little taller than the design's 1.58:1 stand-ins. "Why Knotty Yoga" keeps its picture.
- [x] The two stand-ins are deleted from `img/`: nothing else referenced them. Git still has them.

**Seed** *(create_database.cpp `PopulateHomeSectionsImpl`)*
- [x] **Types of classes** → `Class.jpg`; **Additional health services** → `Massage.jpg`. Both are still stored as JPEGs.
- Not changed: the page's request for a feature picture (a 944 × 598 box). A 4:3 picture fits it at 797 × 598, wider than the 708-pixel stand-ins it replaces.

**Tests** *(create_database_test.cpp)*
- [x] New `SeededHomeFeaturesShowTheStudiosOwnPhotos`: reads the rows through the home page's feed (`GetActiveHomeSections`) and each picture through the photo reader behind `/api/get_scaled_photo`. Each row's picture is its file byte for byte, a JPEG, and 4:3.
  - The expected file is named in the test, not read from the seed, so two rows given one file fail.
  - *As built:* it also fails if either row is missing from the feed.

**Mutation check**
- [x] Planned:
  - **Types of classes** back on its stand-in
  - **Additional health services** back on its stand-in
  - **Additional health services** given `Class.jpg`
  - `Class.jpg` replaced by a 1.58:1 picture beside the test program. The seed and the test read the same copy, so only the 4:3 check can see this one.
  - *As built:* **4 of 4 caught** by the new test, each on its own Linux run, and the source and the test's `img/` copy were restored byte for byte.
    - The two stand-ins: "does not show `Class.jpg`" / "does not show `Massage.jpg`", and "708 x 447 is not 4:3". The deleted stand-ins were still in the build's `img/` copy, which never deletes a file, so these rows really got the old pictures.
    - `Class.jpg` on both rows: "Additional health services does not show `Massage.jpg`".
    - The 1.58:1 `Class.jpg`: "708 x 447 is not 4:3", the only check that could see it.

**Verification**
- [x] knottyyoga on Windows and Linux, the same test names on both. No UI change, so no Angular run.
  - *As built:* **Windows: 5408 passed. Linux: 5408 passed**, the same 5408 names on both: 13.11's 5407 and the new test. On Linux the database helper, the server and the test helper all build. On Windows only the test program and the database helper were rebuilt, because Mason's server was running; the server's own code didn't change.
- [x] `Class.jpg` and `Massage.jpg` hash-checked beside the database helper and the test program on both platforms.
  - *As built:* all eight copies match. Both programs relinked, so the build's own copy ran; no hand copy was needed.
- [x] End to end: `--install_schema` into a throwaway database, then the real server. Signed out, the home page's feed lists both rows, and each picture comes back at the page's own request size.
  - *As built,* on Linux, into a throwaway `knottyyoga_e2e_1312`, dropped afterwards. The install log attaches `Class.jpg` to row 10 and `Massage.jpg` to row 11, and every other home picture as before, with no `seed_image_missing` or `seed_image_failed`.
  - Signed out, at the page's own 944 × 598 request: **Types of classes** and **Additional health services** both come back **797 × 598**, and **Why Knotty Yoga** 708 × 449 as before.
  - As in 13.11, the install was given a dummy mail password, since the server refuses to start without one. Nothing was mailed.

**Hand-testing walkthrough**

**Setup.** Recreate the database with `knottyyoga_database_helper --recreate_database`. Rebuild and start the server, and restart `ng serve`. Use a wide window, signed out.
1. **Home**. Scroll past **Why Knotty Yoga**. **Types of classes** shows a class stretching in red aerial fabrics, in an orange room, with the picture on the right.
2. Under it, **Additional health services** shows Caleb giving a massage, his elbow on the client's back, with the picture on the left.
3. Make the window as narrow as it goes. Each picture sits above its words, and is still whole.

### 13.13 Ryan's edited photos and the two galleries *(seed; added 10/27 from Mason's request; built 10/27, not committed)*

Mason, 10/27: "Ryan was able to edit the photos to make them nicer. I would like you to replace the existing photos with the edited ones at `D:\Knotty Yoga\2026\Oct\Oct 27 Ryan Editied`. Then please use the photos labelled GalleryKnottyYoga*.png for the home page gallery. Please don't show the name of the gallery on the page or the names of each photo. I just want the photos. I also want that gallery to show up on the gallery page under about but also create a second gallery for the about gallery page with the GalleryPartnerAcro*.png photos. Same thing, just raw photos. No gallery name or image name."

The folder holds 15 PNGs: Ryan's edits of the seven class and massage photos, and eight new gallery photos.

| File | | Where it shows |
|---|---|---|
| `BeginnerKnottyYoga.jpg` | Ryan's edit; recropped wider (1.59:1) | Beginner Knotty Yoga's card and class page |
| `KnottyYoga.jpg` | **a different photo**: a class in side planks, one foot in the red fabrics. The old one was a partner acro stack. | Knotty Yoga's card and class page |
| `AerialRope.jpg` | Ryan's edit; recropped **upright** to one of the two people | Aerial Rope's card and class page |
| `Handstand.jpg` | Ryan's edit; recropped closer (3:2) | Handstands' card and class page |
| `PartnerAcro.jpg` | Ryan's edit, same framing | Beginner Partner Acro's card and class page |
| `IntermediatePartnerAcro.jpg` | Ryan's edit; recropped wider (1.43:1) | Intermediate Partner Acro's card and class page |
| `Massage.jpg` | Ryan's edit (warmer), same framing | the Massage card on **Services**, and the home page's **Additional health services** (13.12) |
| `GalleryKnottyYoga1–3.jpg` | new (`4` was dropped the same day: see the follow-up below) | the home page's photo row, and first on **About** ▸ **Gallery** |
| `GalleryPartnerAcro1–4.jpg` | new | second on **About** ▸ **Gallery** only |

Not in the folder, so unchanged: `Class.jpg` (13.12), the five headshots and the home page's other pictures.

**Files**
- [x] Convert each into `src/database_helper/img/` as 1.2 did: ffmpeg, long edge 2000 px, `-q:v 3`, all metadata stripped.
  - *As built:* also 4:2:0 colour (ffmpeg keeps a PNG's full-colour 4:4:4 unless told), as every other seed JPEG is. 130–492 KB each.
  - Seven PNGs carried an sRGB colour profile. sRGB is what a browser assumes anyway, so removing it changes no colour; it was removed so these match the other seed photos. No PNG carried EXIF, GPS or text.
  - Two PNGs have a transparency channel. `GalleryPartnerAcro2` is fully opaque. `IntermediatePartnerAcro` has a few pixels at 97 % opacity, a difference no one can see; the JPEG drops the channel.
  - All 15 were opened after conversion, the seven replacements beside the pictures they replace: upright, sharp, and the same subjects.
- [x] **Massage** is cropped to 4:3 exactly as in 13.11 (OQ-40): Ryan's PNG is the full upright original, recoloured, so the same crop (full width, 800 px from the top) keeps the same face, fist and elbow. 2000 × 1500.
- [x] **Aerial Rope** is now upright (1474 × 2000). The class page shows a 4:3 cutout and the **All Classes** card a square, both from the middle. Checked both: the whole person is in each, the raised foot just inside the top of the 4:3 one.
- [x] The other five keep Ryan's own shapes, 1.40:1 to 1.59:1. The class page cuts a little off their sides to make 4:3.

**Seed** *(create_database.cpp `PopulateImageCarousels`)*
- [x] The class and massage photos need no seed change: same file names (13.1's tables).
- [x] The home carousel (`home`, "Home page photos") gets `GalleryKnottyYoga1`–`4`, in that order. *Since the follow-up below, `1`–`3`.*
- [x] A second carousel, `partner_acro` ("Partner acro photos"), gets `GalleryPartnerAcro1`–`4`. It is on the Gallery after the home one (gallery order 20 after 10), and no home page row shows it.
- [x] Both carousels: **Show Title** and **Show Description** off, so neither the home page nor the Gallery draws a heading or blurb. Each photo has no title or description, which is what a caption is made of. The display names and the home one's description are kept, because **Site Theme** ▸ **Image carousels** lists a carousel by them.
  - *As built:* a small `SeedCarousel` table and one `AddSeedCarousel` loop: the carousel, then its photos at ordinals 10, 20, 30, 40 (the editor's own steps), each with its file attached.
- No UI change for the galleries. The home page and the Gallery already hide a carousel's heading and blurb, and every caption, when those are off; their specs cover it. With no title, a photo's alt text is "Studio photo".
- Not changed: the page title **Gallery** at the top of **About** ▸ **Gallery**. It names the page, like every other page's title, not a carousel. See OQ-42.

**Class page** *(ui: `public/class-detail`; added 10/27, found checking the new photos end to end)*
- [x] The class page asks for its picture in a **900 × 1200** box instead of 900 × 675.
  - The server fits a picture inside the box, and the page shows it in a 4:3 frame, cropped to fill. In the old box, Ryan's upright Aerial Rope came back **497 px wide**, against 900 for every landscape class picture.
  - The frame is up to about 735 px wide (a tablet, where the picture spans the page), so Aerial Rope was stretched 1.5×, and about 2× on a phone's sharper screen.
  - In the new box it comes back 884 wide. A landscape picture still fits by width, so it is served exactly as before.
- [x] Spec: the fallback address, and a new one checking that a 3:4 and a 4:3 picture both come back 900 wide from the box.
- Not changed: the square class pictures (the **All Classes** cards, the weekly schedule's rows). Their boxes are square, which an upright picture fills.

**Found by the Angular run: two pages lose a day across the 11/1 clock change** *(ui; added 10/27)*

The Windows `ng test` failed two specs that 13.13 doesn't touch. Both fail only in a week that contains a daylight-saving change, so they were green last week and would have failed CI all this week. This is the same trap as 13.11's C++ test and the provider schedule's earlier fix. This time one of the two is a real bug visitors would see.
- [x] **Weekly schedule** (**Our Classes**, `public/our-classes`). It stepped through the week in 24-hour jumps from Sunday midnight. The week of Sunday 11/1 is 169 hours long, so every day after the change landed on 11 PM of the day before, and the week ended an hour early.
  - **Seen by visitors this week:** paging to next week showed Sunday to Friday only. **Saturday 11/7 was missing, with its 10:00 AM Knotty Yoga**, and the heading read "Nov 1 – Nov 6".
  - Fix: the week's end, the heading's last day and the day-by-day loop now count calendar days.
  - Spec `OurClassesComponent` "shows all seven days, Saturday's classes included, across a DST change": the clock is pinned to Tuesday 10/27. Next week has all seven days, a 10:00 AM and an 11:30 PM Saturday class (the week's last hour), and the heading "Nov 1 – Nov 7".
  - The existing spec ("renders all seven days for a future week") is unchanged; it failed only because of the bug.
- [x] **Schedule grid** (Manage, `manage/schedule-grid`). The default **To** date was "today + 7 × 24 hours". This week that's 11 PM on 11/2, so the date picker showed **Nov 2**, while the grid itself loaded through 11/3. Its request reads the date in UTC, which happened to round it up.
  - Fix: To is seven calendar days on, at midnight.
  - The existing spec now expects that.
  - The new spec pins the clock to 10/27 and checks To is 11/3 at midnight.
- Both clock-pinned specs only bite when the browser runs in a time zone with daylight saving, as the provider schedule's does: Mason's machine (Pacific) yes, the Linux CI runner (UTC) no. CI could only catch it in the week before a clock change, as it would have this week.

**Tests** *(create_database_test.cpp)*
- [x] New `SeededGalleriesShowRyansPhotosWithoutNames`, through the readers the pages use: the Gallery's list of carousels, each carousel's photos (both behind `/api/gallery_carousels`; the photos also behind `/api/image_carousel/<id>`, the home row's request), the home feed, and the photo reader behind `/api/get_scaled_photo`.
  - Exactly two carousels on the Gallery, home first. Neither shows its title or description.
  - Each has four photos (the home one three, since the follow-up below), none with a title or description, each in its own place. Each photo is its file byte for byte, as a JPEG. The files are named in the test in display order, so a photo swapped, repeated or out of order fails.
  - The home page has one carousel row, and it shows the home carousel.
- [x] The replaced files need no new test. `PopulateSeedPhotosPutsEachListedFileOnItsRow` compares every class picture with its file byte for byte, and checks each class has a different one. `ServicesShowsTheMassagePhoto` and `SeededHomeFeaturesShowTheStudiosOwnPhotos` check Massage is still 4:3.

**Mutation check**
- [x] Planned, each against the new test on its own Linux run:
  - g1: both carousels show their name
  - g2: both show their description
  - g3: each photo captioned with its file name
  - g4: the two galleries' first photos swapped
  - g5: home photos 2 and 3 out of order
  - g6: partner acro first on the Gallery
  - g7: the home row shows the partner acro carousel
  - g8: the partner acro carousel not seeded
  - g9: every photo in the same place (one ordinal)
  - u1: the class page's box back to 900 × 675
  - The replaced class photos can't be mutation-checked: the seed and the test read the same copy beside the test program. The hash check below stands in for it.
  - *As built:* **g1–g9, 9 of 9 caught** by `SeededGalleriesShowRyansPhotosWithoutNames`. The source was restored byte for byte, and the clean rerun passes.
    - g1 and g2: "home shows its name" / "home shows its description"
    - g3: "home has a photo with a title"
    - g4: "home photo 1 is not GalleryKnottyYoga1.jpg", and the same for partner acro
    - g5: "home photo 2 is not GalleryKnottyYoga2.jpg", and photo 3
    - g6: "the home page's photos come first on the Gallery"
    - g7: the home row points at carousel 8, not 7
    - g8: "both carousels belong on the Gallery" (1 of 2). Its first attempt was never applied: the pattern ended at a line end, which never matches before this file's Windows line endings. It was rerun without that anchor.
    - g9: one photo where four were expected
- [x] The two clock-change fixes, each reverted on its own on Windows (Pacific time), with that page's specs run and the file restored byte for byte. **u1 and these four:**
  - u1, the class page's box back to 900 × 675: both of the class page's address specs fail.
  - u3, the weekly schedule's heading ending 6 × 24 hours on: the new spec ("Nov 1 – Nov 6").
  - u5, the week ending 7 × 24 hours on: the new spec, which loses the 11:30 PM Saturday class.
  - u4, the grid's To back to today + 7 × 24 hours: both grid specs.
  - u2, the weekly schedule's loop back to 24-hour steps, is **not caught, and can't be**. Once the week's end is a calendar date, 24-hour steps from Sunday still give one entry per day, on the right dates. Either half of the fix alone brings Saturday back. The loop counts calendar days anyway, so no day logic on the page depends on 24-hour steps.

**Verification**
- [x] knottyyoga on Windows and Linux, the same test names on both.
  - *As built:* **Windows: 5409 passed. Linux: 5409 passed**, the same 5409 names on both: 13.12's 5408 and the new test.
  - On Linux the database helper, the server and the test helper all build.
  - On Windows only the test program and the database helper were rebuilt, because Mason's server was running; the server's own code didn't change.
- [x] Angular `ng test` on Windows and Linux; lint on the six changed files; `ng build`.
  - *As built:* **Windows: 3995 passed. Linux: 3995 passed**, 13.11's 3992 and the three new specs. Lint is clean.
  - `ng build` succeeds. Its size warnings are the existing ones; no stylesheet changed.
  - The first Windows run, before the clock-change fixes, had 3991 passed and 2 failed (above).
- [x] All 15 files hash-checked beside the database helper and the test program on both platforms.
  - *As built:* 15 of 15 beside each program, on both platforms. Both programs relinked, so the build's own copy ran; no hand copy was needed.
- [x] End to end: `--install_schema` into a throwaway database, then the real server, read signed out: the Gallery feed, the home page's carousel, and every class and the Massage picture at its page's request.
  - *As built,* on Linux, into a throwaway `knottyyoga_e2e_1313`, dropped afterwards. The install log attaches the eight gallery photos to `image_carousel_photos` rows 1–8, in order, and every other picture as before:
    - 6 classes, 8 home sections, 5 instructors, 2 people, 6 products
    - no `seed_image_missing`, `seed_image_failed` or `seed_lookup_missing` line
  - `/api/gallery_carousels`: `home`, then `partner_acro`, both with `show_title` and `show_description` false. Each has four photos at ordinals 10–40, every title and description empty. Each photo comes back 560 px tall at the carousel's own 900 × 560 request: home 636, 738, 775 and 460 wide; partner acro 567, then 839 three times.
  - The home feed's carousel row points at carousel 1, the `home` one, and `/api/image_carousel/1` returns the same four photos.
  - Class pictures at the **All Classes** card's request (540 box): Aerial Rope 397 × 540, upright, and the five others 540 wide. Massage's card request comes back 1152 × 863 (4:3), and both home features 797 × 598, as in 13.12.
  - As before, the install was given a dummy mail password, since the server refuses to start without one. Nothing was mailed.

**Follow-up, 10/27: the fourth home photo dropped** *(seed; built 10/27, not committed)*

Mason, 10/27: "Let's drop the last image from the knotty yoga gallery (GalleryKnottyYoga4)". It was the upright crop of the same aerial rope pose as `GalleryKnottyYoga3`.
- [x] The home carousel's seed lists `GalleryKnottyYoga1`–`3` only. The partner acro gallery is unchanged.
- [x] `GalleryKnottyYoga4.jpg` deleted from `src/database_helper/img/`. Ryan's PNG is still in the 10/27 folder.
  - Its copies beside the Windows and Linux programs were deleted by hand too: the build copies new files there but never removes old ones.
- [x] Test: `SeededGalleriesShowRyansPhotosWithoutNames` now expects three home photos, `1`–`3` in order.
- [ ] Mutation m1, the seed still listing the fourth photo while the file is gone (the regression this change could have):
- [ ] knottyyoga on Windows and Linux, the same test names on both. No UI change, so no Angular run.
- [ ] End to end, signed out:
- On a 280 px tall row, the three home photos come to about 1,110 px across, against 1,350 for four. So in most desktop windows the home row now sits still with no arrow. On the Gallery, whose content is at most 1,168 px wide, the home row fits too; the partner acro row (about 1,590 px) still scrolls.

**Hand-testing walkthrough**

**Setup.** Recreate the database with `knottyyoga_database_helper --recreate_database`. Rebuild and start the server, and restart `ng serve` (the class page changed). Use a wide window, signed out.
1. **Home**. Under the sentence and the **SAFE SPACE** badge, a row of three photos: two classes in the red aerial fabrics, then one of aerial rope, a woman upside down holding the rope. There is no heading above the row and no words on any photo. In a window wider than the three (about 1,110 px), the row ends short of the right edge and nothing moves.
2. Narrow the window until the third photo is cut off at the right edge. An arrow appears there; click it and the row moves on by one photo. Left alone, it moves along by itself every few seconds.
3. **About** → **Gallery**. Under the page's **Gallery** title, the same three photos, then a second row: partner acrobatics against a white wall, flyers balanced on their bases. Neither row has a heading, and no photo has words on it. In a wide window the first row fits the page and sits still with no arrow; the second is wider than the page and has its arrow.
4. **Our Classes** (page headed **All Classes**). **Knotty Yoga**'s card shows a new picture: a class in side planks, one foot each in the red fabrics. The other five are the same photos, recoloured; **Aerial Rope** now shows one person, upside down on the rope.
5. Click the **Aerial Rope** card. The picture beside the description shows the whole person, feet included, and is as sharp as the other classes' pictures. Make the window about tablet width, so the picture spans the page above the description: still sharp.
6. **Services** (page headed **Our Services**). The **Massage** card shows the same picture as before in warmer colour, Caleb's face and his elbow on the client's back.
7. Make the window as narrow as it goes. On **About** → **Gallery**, both rows are still photos only, a little shorter.
8. *(Only before Sunday 11/1: after it, next week has no clock change.)* Wide window again. **Our Classes** → **Weekly schedule** (page headed **Our Classes**) → **Next week**. The label beside the buttons reads **Nov 1 – Nov 7**, and the days run Sunday to **Saturday**, with Saturday's **Knotty Yoga** at 10:00 AM.
9. *(Same: only before 11/1.)* **Sign In** as `masonbendixen@gmail.com` / `pass`. **Admin** → **Manage Products** → **Schedule Grid** tile. **To** is a week from today: on Tuesday 10/27, **11/3/2026**, not 11/2.

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
| An option to centre the home page's banner images | 14.2–14.5 (OQ-29: the top picture, with a second, mobile-only one) |
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

**Planned 10/21/2026; updated 10/22** with Mason's answers to OQ-29, 30 and 31. One `server_components` change (14.1), with its own push and pin bump as in 11.11. The rest is knottyyoga.

**Implemented 10/22/2026, not committed.**
- `server_components`: Windows 1900 passed, Linux 1900 passed.
- knottyyoga: Linux 5404 passed against the local `server_components`. Windows 5399 passed on the pinned SHA, which is the same names less the five new `server_components` tests.
- Angular: 3654 passed. Playwright: 28 passed.
- All 27 mutations were caught. Both end-to-end passes PASS.

**Pinned 10/22/2026.** Mason pushed `server_components` `920f18a`, and both apps now pin it.
- knottyyoga: Windows 5404 passed, Linux 5404 passed.
- CommunityFinder: Windows 1921 passed, Linux 1921 passed.

**Reopened 10/22/2026 for 14.10**, after Mason's first look: the headline could still break before "THE RIGHT WAY.", and the desktop top picture was pixelated.

**Still to do:**
- Mason commits the two pin lines and the knottyyoga changes.
- Mason hand-tests 14.9, after recreating the database again for 14.10's picture limit.

### 14.1 Accept SVG uploads *(server_components: platform/endpoints/upload_photo)*
- [x] `/api/upload_photo/<table>/<id>/svg` is accepted alongside `jpeg` and `png` (`upload_photo.cpp:75`).
  - *As built:* the "is it really an SVG" check is `ImageHelper::IsSvgDocument`, and `UploadAndAssociatePhoto` applies it to every vector upload. It runs in the business layer rather than the endpoint, so no other path can store a raster labelled `svg` either; the endpoint turns its refusal into the 400.
  - It also skips a `<!DOCTYPE>`, including Illustrator's "with entities" form, whose internal subset carries `>` characters of its own. The seeded badge's own head, an XML declaration then an Illustrator comment, passes.
  - **Why.** The photo picker (`@honuware/ui/photos`) already offers SVG files and sends them as `svg`, and the server refuses them with "Unsupported image type. Must be 'jpeg' or 'png'". So today no admin can upload the intro badge (14.4), or replace the seeded Get Started artwork with another SVG.
  - **The body must be SVG text.** After an optional byte-order mark, whitespace, an XML declaration and comments, its first element must be `<svg`. Anything else sent as `svg` is refused with 400, so a mislabelled PNG can't be stored as a vector.
  - **Stored as it came, as a vector.** `ImageHelper::UploadAndAssociatePhoto` already accepts vectors, and `get_scaled_photo` already returns an SVG unscaled, with `nosniff`.
  - **Safe under the existing SVG rule** (Polish 11.1, `get_scaled_photo.cpp:175`):
    - every page draws these pictures with `<img>`, which runs nothing
    - opened directly, production's site-wide CSP (`script-src 'self'`) blocks any script inside
    - only someone who can already edit the row may upload (`RequireTableWriteAccess`)
- [x] Tests in `upload_photo_test.cpp`:
  - an SVG uploads and reads back byte for byte, as `image/svg+xml`, through the scaled-photo endpoint
  - PNG bytes labelled `svg` are refused, and nothing is stored
  - `gif` is still refused
  - *As built:* the first two are new (`UploadPhotoSvgReadsBackByteForByte`, `UploadPhotoPngLabelledSvgIsRefused`), and the existing `UploadPhotoUnsupportedType` already covers `gif`. Three more are in `image_helper_test.cpp`: what may come before the root, what is refused (a PNG, a JPEG, an HTML page with an inline SVG, `<svgfoo`, unterminated constructs), and the upload path refusing a PNG labelled `svg` with nothing stored.
- [ ] Ship as 11.11 did. Mason commits and pushes `server_components`. knottyyoga and CommunityFinder then pin the new SHA, and both apps' gates pass on both platforms.

### 14.2 Three new switches on a home section *(knottyyoga: db_schema, migration, admin metadata)*
- [x] `home_sections` gains three booleans. All default to FALSE, so every existing row behaves as today. They are named like the four `hidden_when_*` flags beside them.

  | Column | Editor label (14.4) | Meaning |
  |---|---|---|
  | `hidden_on_narrow_screens` | **Desktop only** | Hidden in windows narrower than 768 px (D16) |
  | `hidden_on_wide_screens` | **Mobile only** | Hidden in windows 768 px wide and wider |
  | `image_centered` | **Centre the picture** | A top-picture row is laid out around the middle: its headline goes under the picture, centred, at every width (14.5, OQ-29) |

- [x] Migration `app/0012_home_sections_screen_width_and_centering`: `ADD COLUMN IF NOT EXISTS` for each column.
  - Test, modelled on `HiddenWhenNotMemberAddsTheColumnAndIsIdempotent`: run it twice on a database without the columns, then once on a fresh one.
  - *As built:* `HomeSectionsScreenWidthAndCenteringAddsTheColumnsAndIsIdempotent` also checks that a row already there gets `f` for all three, and writes a row with all three on and reads it back through the home page's reader.
- [x] Admin metadata in `create_database.cpp`: `PopulateAdminColumnDataInfo` and `PopulateAdminColumnFriendlyNames` entries for all three.
  - `home_sections` is already a top-level admin table, so the foreign keys hold. `create_database_test`'s metadata check covers it.
- [x] Read path. `GetActiveHomeSections` is `SELECT *`, so the columns reach `/api/home_sections` with no code change. Tests:
  - in `home_sections_test.cpp`: a row written with each flag reads back with it
  - in the endpoint's test: the three fields arrive, each on its own row. Like the four flags beside them they cross the wire as `t`/`f`; the page's network layer turns them into booleans.
- [x] **Added: `HomeSectionExtras` carries the three, and the page-content bundle exports and imports them.** Without that, a theme export and re-import would have lost **Desktop only** from both seeded heroes. Test: `PageContentBundleSectionTest.ScreenWidthAndCenteringRoundTrip`.
- [x] **Added: `HomeSections::SetHomeSectionActive`**, which switches a row on or off without touching anything else on it. The seed needs it for the membership row (14.3), because `AddHomeSection` always creates an active row. Test: `SetHomeSectionActiveRetiresAndRestoresOnlyThatRow`.

### 14.3 Seed *(create_database.cpp `PopulateHomeSectionsImpl`; app_secret_values.cpp)*
- [x] **Membership tiers off by default.** The row is still seeded, with `active` false, so **Page Content** can turn it on with "Show this section on the home page".
- [x] **"Come join the fun!" links to Memberships.** The link becomes `/shop`, the page the top nav's **Memberships** opens, labelled **See memberships**. Today it is `/start`, labelled "Get started".
- [x] **Two top pictures, one for each width** (OQ-29).
  - **One headline.** "The right way." stops being a separate red line. The title is "Get into the best shape of your life, the right way." and the body is empty. Both rows take it from one constant, so they can't differ.
  - **Today's row becomes desktop only** (`hidden_on_narrow_screens`). It keeps `home_hero.jpg`, with the people on the left, and is not centred.
  - **A new row, mobile only,** at ordinal 25, so it sits in the same place on the page. It has `hidden_on_wide_screens` and `image_centered`.
    - Its picture is Mason's new photo, `C:\Users\mason\Downloads\bd1ac730-677f-4471-8cfd-0b1080081126.jpg`: the same two people, standing in the middle of a grey background.
    - It is copied in as `src/database_helper/img/home_hero_mobile.jpg`. At 2048 × 969 and 77 KB it needs no conversion.
    - The whole `img/` folder is already copied beside the executables, so there is no CMake change. The copy only happens on a relink, though, so copy the file there by hand too ([[gotcha_seed_img_copy_is_post_build]]).
    - *As built:* the copy is byte for byte, checked by hash. Both executables relinked in this phase, so their `img/` copies have it.
- [x] **The intro row carries its own content:**
  - **Body:** the sentence, from the same constant as the `site_hero_headline` default, so the two can't differ.
  - **Title:** "This studio is a designated safe space", the badge's description for screen readers. Today it is hardcoded in the template.
  - **Photo:** `SafeSpace_icon.svg`, copied from `ui/src/assets/svg/` into `src/database_helper/img/` and attached as `svg`.
  - *As built:* the seed file is named `safe_space_badge.svg`, after the img folder's snake_case names. The shared constant is `App::kSiteHeroHeadlineValue`, moved into `app_secret_values.h`.
- [x] **Social links: Facebook and Instagram only, at the studio's own pages** (`app_secret_values.cpp:61`, OQ-30):
  - `Facebook|https://www.facebook.com/KnottyYoga/`
  - `Instagram|https://www.instagram.com/knottyyoga`
- [x] Tests, through the readers the pages use (D4):
  - `create_database_test`, after `PopulateHomeSections`: `GetActiveHomeSections` has no `membership` row, and the full list has it, inactive.
  - The banner's link and label.
  - The intro row's body and title, and its photo is the SVG byte for byte, typed `svg`, read through the photo reader.
  - The two top-picture rows:
    - both carry the one-line headline and an empty body
    - one is desktop-only, with `home_hero.jpg`
    - the other is mobile-only and centred, with `home_hero_mobile.jpg` byte for byte
    - no other row is desktop-only, mobile-only or centred
  - The social-links value, read through the site-content reader: exactly the two lines above.
  - *As built:* the home rows are one test, `CreateDatabaseTest.SeededHomePageCarriesRyansReview`. The existing order test now expects two heroes in second and third place. The social links are `AppSecretValuesTest.SocialLinksAreTheStudiosFacebookAndInstagramPages`, read through `Branding::LoadSiteContent`.

### 14.4 Page Content editor *(ui: pages/manage/page-content)*
- [x] **Two more toggles** under the existing four. They are slide toggles like their neighbours, not checkboxes:
  - **Desktop only — hidden in windows narrower than 768 px**
  - **Mobile only — hidden in windows 768 px and wider**

  With both on, a warning says nobody will ever see the section, as the membership pair already does. The row list shows **Desktop only** and **Mobile only** badges, as it shows **Signed-in only**.
  - *As built:* the width pair has a warning of its own, which says why: every window is either narrower than 768 px or not.
- [x] **Hero band rows** get a **Centre the picture** toggle, with the hint "For a picture whose subject stands in the middle: the headline goes under it, centred."
  - *As built:* saving any other kind writes `image_centered` false, so the flag can't sit set on a row where nothing reads it.
- [x] **Intro rows** become editable like a content block:
  - an **Image** upload, labelled **Badge** (SVG, PNG or JPEG)
  - **Body**, labelled **Text**
  - **Title**, with the hint "Describes the badge for people using a screen reader."
  - *As built:* no link fields, since the strip has no button.
- [x] **Each built-in kind says what it is.** One line under **Kind** describes the chosen kind, for example "Intro strip: a sentence with a badge under it". This answers Ryan's question in the editor itself. It is one map of eleven short lines, and the same opacity applies to every built-in kind.
  - *As built:* twelve lines, one per kind, content blocks included.
- [x] **One place to edit the intro strip:**
  - **Site Theme** loses "Home intro sentence" (**Copy**) and "Home badge image URL" (**Brand basics**).
  - `SiteConfig` drops `heroHeadline`, `heroImageUrl` and `heroImageAsset`.
  - The framework's `site_hero_*` keys stay: CommunityFinder shares them. They are simply unused here.
  - *As built:* the two keys join `site_about_markdown` in Site Theme's `NOT_EDITABLE_HERE` list, which the slot-coverage spec reads.
  - `ui/src/assets/svg/SafeSpace_icon.svg` is left in place, though nothing reads it now.
- [x] Specs:
  - each new control shows for its kinds only, and saves its column
  - the both-on warning, and the two badges
  - the kind descriptions
  - the two Site Theme fields are gone

### 14.5 Home page *(ui: home-page and its sections)*
- [x] **Desktop only / Mobile only** (D16). `isVisible` drops `hidden_on_narrow_screens` rows below 768 px, and `hidden_on_wide_screens` rows from 768 px up.
  - One `matchMedia('(width >= 768px)')` with a change listener decides it, so narrowing or widening a window switches the sections live.
  - A hidden section is not drawn at all, so a desktop-only picture costs a phone nothing.
  - The width is the calendar's (`BELOW_MD_MEDIA_QUERY`, `calendar.types.ts:32`). It moves to `shared/utils/` so both read one constant.
  - *As built:* `shared/utils/breakpoints.ts`, which `calendar.types.ts` re-exports, so the calendar's imports are unchanged.
- [x] **A centred top picture** (`home-hero-section`, OQ-29). With `image_centered` on:
  - the picture still spans the band
  - the headline goes under it, centred, at every width
  - the picture is requested in a 2048 × 2048 box rather than the 2048 × 624 band, so a taller picture like the mobile one isn't shrunk to the band's height ([[gotcha_scaled_photo_box_size]])
  - *As built:* the server caps every scaled picture at 800 × 800 (`image_max_public_scaled_width`/`_height`), so the mobile picture comes back 800 × 379. Fitted to the band's box it would have been about 515 × 244.
  - *14.10 raised that limit to 2048, so the mobile picture now comes back at its full 2048 × 969.*

  With it off, nothing changes: from 768 px up the headline sits over the picture's right half, and below that it sits under it, aligned left.
  - **Why the headline moves.** A centred subject leaves no empty side for the words, so over the picture they would cover the people.
  - The photo strip is unchanged. OQ-29's default had guessed it was the picture meant.
- [x] **The intro strip** draws the row's **Text** and **Badge** instead of the two Site Theme settings. The badge's description is the row's **Title**. A row with no badge shows the sentence alone.
  - *As built:* the badge is asked for at three times its 142 px width. A badge that fails to load is removed, rather than left as a broken-image mark.
- [x] **One font** (OQ-31: Roboto).
  - The intro strip's sentence takes the "New here? …" line's lettering: the body font (`--font-body`, which is Roboto in the seed), at regular weight. Until now it used the display font, semibold.
  - One shared class sets both lines, so they can't drift apart again.
  - Each line keeps its own size. The intro sentence stays larger, as it is the strip's only text.
  - *As built:* the shared class is the existing global `.din`, which the "New here?" line already used.
- [x] **The local-mode mock's home rows match the seed**: the two top pictures, one desktop-only and one mobile-only.
  - *As built:* the rest of 14.3 too: the intro row's copy, the banner's link and the membership row switched off.
  - So the Playwright tap-target spec now finds one "View all" link on the home page, not two.
- [x] Specs:
  - visibility at each width, with `matchMedia` stubbed, including a live change
  - a centred top picture puts its headline under the picture at desktop width, and asks for the taller box; an uncentred one keeps the headline over it
  - the intro strip draws the row's text and badge, with the badge's description
  - both text lines compute to the same font family and weight, and the family is the body font
- [x] Playwright, new `e2e/mobile-audit/layout/home.spec.ts`, at 390 and 1280 px:
  - a desktop-only row is absent on the phone and present on the desktop, and a mobile-only row the reverse
  - exactly one top picture shows at each width: the desktop one at 1280 px, and the mobile one at 390 px
  - the screenshots are kept for 14.9
  - *As built:* at 390, 768 and 1280 px, plus a window resized across the line. 768 checks that "wide" starts at exactly 768.
    - The mock app has no pictures, so the spec serves the seed's own files.
    - Screenshots of the band and of the whole page are in `ui\e2e\mobile-audit\output\home\`.

### 14.6 Brand icons in the footer *(ui: shared/components/footer)*
- [x] The footer draws each social link as its brand's icon instead of the first two letters of its label (`footer.component.ts:27`).
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
- [x] **Where the icons come from:** five SVG paths from Simple Icons, which is public domain, in one file, `social-icons.ts`.
  - There is no new dependency: the package holds thousands of icons and this needs five.
  - Drawn inline in `currentColor`, so the theme's colour reaches them. They are fixed strings in the code, not studio content, so drawing them inline is safe.
  - Each link's `aria-label` stays its label, and the tap target stays 44 px.
  - *As built:* the paths are copied from Simple Icons 13.0.0 (CC0), fetched from the npm package's published icons.
    - A host matches exactly or as a subdomain, so `www.facebook.com` counts and `notfacebook.com` doesn't.
    - Labels compare as letters only, so "Tik Tok" is TikTok.
    - The icon is `aria-hidden`; the link's label already names it.
- [x] Twitter comes off too:
  - `site-config.service.ts`'s fallback list drops it, to match the seed.
  - Two code comments that already claim "the footer maps known labels to icons" become true.
  - *As built:* the fallback list carries the studio's two real addresses, like the seed.
- [x] Specs:
  - each of the five is recognised by its address, and by its label
  - an unknown network falls back to two letters
  - a recognised link draws its icon and shows no letters
  - these replace the two specs that pin "Fa In Tw"
  - *As built:* also, the address beats a label that names a different network.

### 14.7 Mutation check
- [x] **27 mutations, all caught**: 4 in `server_components`, 10 in knottyyoga's C++ and 13 in Angular. Every file was restored byte for byte, and a clean run of the same tests passes afterwards. The script is the scratchpad's `phase14_mutations.ps1`.
  - **`server_components`** (the Windows build, `ImageHelperTest` and `UploadPhotoTest`): an upload labelled `svg` stored whatever its bytes; the endpoint refusing a real SVG; `<svgfoo` taken for an `<svg>` root; a comment before the root not skipped, which refuses the studio's Illustrator badge.
  - **knottyyoga C++** (Windows):
    - migration 0012 not adding `image_centered`
    - the membership row seeded active
    - the banner linking to `/start`
    - the intro row seeded without its badge
    - the mobile hero seeded without **Mobile only**, or with the desktop photo
    - the headline cut back to "…of your life,"
    - Twitter back in the social links, or Facebook's front page
    - a bundle import dropping **Centre the picture**
  - **Angular**:
    - a desktop-only row on a phone, a mobile-only row on a desktop, and a resize that doesn't switch
    - a centred hero keeping its headline over the picture, or asking for the band's box
    - the intro strip ignoring its row
    - the two lines in different fonts
    - a known network falling back to letters, or the address ignored
    - **Mobile only** never saved
    - **Centre the picture** saved on a non-hero
    - the intro strip with no badge upload
    - Site Theme offering the old intro sentence
  - *Not mutated:* the network layer's `t`/`f` conversion of the new flags. That file has no spec in this repo, as 12.1 noted, so `tsc` is its only check.
- Planned:
  - the upload accepting non-SVG bytes as `svg`, or refusing a real SVG
  - the migration not adding a column
  - the membership row seeded active
  - the banner still linking to `/start`
  - the intro row seeded without its photo
  - the mobile top picture seeded without **Mobile only**, or without its picture
  - "The right way." seeded back as a separate line
  - Twitter back in the seed, or a front-page address in place of the studio's own
  - a desktop-only row shown on a phone, or a mobile-only row shown on a desktop
  - a centred top picture keeping its headline over the picture
  - the intro strip reading Site Theme again
  - the two fonts differing
  - a known network falling back to letters

### 14.8 Verification
- [x] `server_components` gates on both platforms, then knottyyoga and CommunityFinder against the new pin: Windows and Linux, same test names.
  - [x] **`server_components`: Windows 1900 passed. Linux 1900 passed.** That is 1895 plus the 5 new tests.
  - [x] **knottyyoga, before the pin bump.**
    - **Linux: 5404 passed**, built against the local `server_components` checkout, with both executables built.
    - **Windows: 5399 passed**, on the pinned SHA. The server executable wasn't rebuilt there because it was running; the tests, the database helper and the test helper were.
    - The names match except for the five new `server_components` tests, which only the local checkout has. Windows picks them up with the pin bump.
  - [x] After Mason pushes `server_components`: bump the pin in knottyyoga and CommunityFinder, and run both apps' pinned gates on both platforms.
    - Mason pushed `920f18a`. `GIT_TAG` now names it in `knottyyoga_server/CMakeLists.txt` and `communityfinder_server/CMakeLists.txt`.
    - **knottyyoga: Windows 5404 passed. Linux 5404 passed.**
    - **CommunityFinder: Windows 1921 passed. Linux 1921 passed.** That is 1916 plus the five new framework tests.
    - Both platforms ran the same test names, and each build's cloned copy is at `920f18a`.
    - On Linux both apps' executables were built. On Windows, `knottyyoga_the_server` wasn't rebuilt because it was running; every other target was.
- [x] Angular: `ng test`, `tsc` for the app and the specs, `ng lint` and `ng build`.
  - [x] **`ng test`: 3654 passed.** That is 3630 plus 24 new specs; the final run was on the restored sources, after the mutation check.
  - [x] `tsc` (app, specs and the Playwright specs), `ng lint` and `ng build` are clean, with the same six size-budget warnings as before.
  - [x] Playwright, on a dev server that had built cleanly: `home.spec.ts`, `public-tap-targets.spec.ts` and `header.spec.ts` at every size, **28 passed**. The home spec serves the seed's own pictures to the page, so "the headline is over the picture" is checked against a real picture.
  - *Found on the way: the build had dropped every `xl:` class.*
    - Without those classes the header can't switch to its desktop bar, so `header.spec.ts` failed and the screenshots showed the phone menu laid out open at every width.
    - `ng build` produced the classes only after `ui/.angular/cache` was cleared. One more dev server started by Playwright still failed; one started after that passed.
    - It is not Phase 14's code: Tailwind run directly on the same sources and config produces the classes.
    - If the header ever looks wrong under `ng serve`, clear that folder and restart.
- [x] **End to end:** `--install_schema` into a throwaway database, then the real server. *Both passes PASS (scratchpad `phase14_e2e.sh`).*
  - Signed out, the home feed has no membership row, and the banner points to `/shop`.
  - It has two hero rows. The mobile-only one's picture is served whole when asked for in the 2048 × 2048 box.
    - *As built:* it is served at 800 × 379: the server's 800 px cap, in the photo's own shape. The band's 2048 × 624 box would have served it at 513 × 243.
    - The desktop hero is served at 800 × 243 under the same cap, so on a wide screen it is shown at about one and a half times that width. That predates Phase 14 and is not changed here.
    - *Mason saw it pixelated on his screen, so 14.10 raised the cap to 2048. Both top pictures now come back at full size.*
  - The intro row's badge is served as an SVG.
  - Signed in as an admin, an SVG uploaded through the endpoint comes back byte for byte.
    - *As built:* through the real middleware, CSRF included. PNG bytes labelled `svg` get a 400, and the stored badge is unchanged.
  - `--migrate` on a database from before Phase 14 adds the three columns.
    - *As built:* the three columns are dropped from a fresh install, then `--migrate` reports success, adds all three back and records 0012 in the ledger. Every row in the feed then carries the three, all off.

### 14.9 Hand-testing walkthrough

**Setup.** Recreate the database with `knottyyoga_database_helper --recreate_database`. Rebuild and start the server, and restart `ng serve`. Use a window at least 1280 px wide, and start signed out.

**Uploading an SVG (step 10) is ready.** 14.1 is pinned (`920f18a`). Your running server predates the pin, so stop it, build `knottyyoga_the_server`, and start it again before you begin.

**After 14.10, do the setup again:** stop the server, recreate the database, then rebuild and start the server, and restart `ng serve`. The new picture size limit is written when the database is created. Then reload the home page with **Ctrl+F5**, so the browser fetches the pictures again.

**Signed out: the home page and footer**
1. Home page, at full width. The top picture is today's, with the two people on the left. Over its dark right half is one white headline: **GET INTO THE BEST SHAPE OF YOUR LIFE, THE RIGHT WAY.** There is no separate red "THE RIGHT WAY." line.
   - The picture is sharp, not blocky, even on your widest screen.
   - Slowly narrow the window from as wide as it goes down to about the width of a phone, watching the headline. Its lines are about even, and **LIFE, THE RIGHT WAY.** always stays together on the last line. "THE RIGHT WAY." never starts a line.
2. Narrow the window to about the width of a phone. Without a reload, the top picture changes to the new one: the same two people in the middle of a grey background. The same headline sits under the picture, centred. Widen the window again, and the first picture comes back.
3. Under the top picture is "Knotty Yoga is an inclusive, high-level acrobatic fitness studio.", with the rainbow **SAFE SPACE** badge under it, as before.
4. Scroll the whole page. There are no membership cards anywhere on it.
5. The **Come join the fun!** picture's button reads **See memberships**. Click it: the page is headed **Memberships**.
6. Back on the home page, look at the black **Get Started** band's "New here? The Get Started guide walks through…".
   - It is in the same lettering as the sentence in step 3: the same plain, not-bold typeface. The step 3 sentence is larger.
   - Neither uses the tall, narrow lettering of the top picture's headline.
7. Footer, under **Follow us on Social Media!**: exactly two icons, the Facebook "f" and the Instagram camera. There is no "Fa", "In" or "Tw", and no Twitter.
   - Click the Facebook icon. It opens Knotty Yoga's own Facebook page, not Facebook's front page.
   - Back on the site, the Instagram icon likewise opens Knotty Yoga's Instagram page.

**Signed in: editing the rows**

8. **Sign In** as `masonbendixen@gmail.com` / `pass`. Go to **Admin** → **Manage Products** → **Page Content** tile, **Home sections** tab.
   - The **Membership tiers** row carries a **Hidden** badge.
   - There are two rows whose kind badge reads **Hero band**. One carries a **Desktop only** badge, and the other a **Mobile only** badge.
9. Click the pencil on the **Mobile only** hero band. **Mobile only** and **Centre the picture** are both on, and **Image** shows the new photo. There is a **Title** field holding the whole headline, and no **Body** field. Click **Cancel**. On the **Desktop only** one, **Centre the picture** is off.
10. Click the pencil on the row whose kind badge reads **Intro strip**.
    - A line under **Kind** says what an intro strip is.
    - **Badge** shows the SAFE SPACE picture, and **Text** holds the sentence.
    - Change **Text** to `Knotty Yoga is an inclusive acrobatic fitness studio.`
    - Under **Badge**, **choose file**: `server\knottyyoga_server\src\database_helper\img\tier_icon_family.svg`.
    - Click **Save**.
11. Home page: the new sentence, with the family icon under it in place of the badge.
12. Back in **Page Content** → **Home sections**, edit **Why Knotty Yoga**: turn on **Desktop only** → **Save**. The row shows a **Desktop only** badge.
13. Home page: **Why Knotty Yoga** is there. Narrow the window to about the width of a phone. It disappears without a reload. Widen the window again and it comes back.
14. Edit **Types of classes**: turn on **Mobile only** → **Save**. On the home page it is gone at full width, and appears once the window is narrowed.
15. Edit **Additional health services**: turn on both **Desktop only** and **Mobile only**. A warning says nobody will ever see the section. Click **Cancel**.

### 14.10 Mason's first look: the headline and the picture *(knottyyoga: ui home-hero-section, page-content; database_helper seed)*

**Added and implemented 10/22/2026.** Mason, after recreating the database:
- "I said that I don't want GET INTO THE BEST SHAPE OF YOUR LIFE, and THE RIGHT WAY to be split."
- "The image for the GET INTO THE BEST SHAPE OF YOUR LIFE is very, very pixelated despite the source being very high res."

**What was wrong**, measured on his local site:
- **The break wandered with the window.** The red line was gone (14.3, 14.5), but the browser filled each line as far as it would go. At 1680 px that put the break right after "YOUR LIFE,", leaving **THE RIGHT WAY.** alone on the second line, which looks like the old separate line. At 1920 px it left **RIGHT WAY.** alone.
- **The desktop picture came back 800 × 243 at every width.** honuware caps every scaled picture at 800 × 800 (the stored settings `image_max_public_scaled_width`/`_height`), and the page stretched that 1.6× at 1280 px and 3.2× at 2560 px. More on a high-density screen. The file itself is 2048 × 624.

**The fixes:**
- [x] **Balanced lines** (`home-hero-section.component.scss`, `text-wrap: balance`). A headline that needs two lines splits into two about equal ones, "GET INTO THE BEST SHAPE OF / YOUR LIFE, THE RIGHT WAY.", rather than one long line and a short one.
- [x] **"life, the right way." is never split** (`keepClosingClauseTogether`, `home-hero-section.component.ts`).
  - Balancing alone isn't enough. On a screen narrow enough for three lines, the most even split can be "…THE BEST / SHAPE OF YOUR LIFE, / THE RIGHT WAY.".
  - So a short closing clause, at most three words after the headline's last comma, is joined to the word before the comma with no-break spaces.
  - A longer clause is left alone, since one unbreakable run that long could be wider than a phone. A screen reader reads the no-break spaces as ordinary spaces.
- [x] **A top picture has no body.** `home-hero-section` draws only the title, so a row that still carries the old "The right way." body can't bring the red line back. Page Content no longer offers **Body** on a hero row. The `db_schema/home_sections.h` comment that described the accent line now says it is gone.
- [x] **The picture size limit is 2048** (`Seed::PopulatePublicPhotoSizeLimit`, `seed_app_data.cpp`).
  - The seed writes 2048 over honuware's 800 for both settings, after the framework's defaults, the way `PopulateSiteAssets` writes the tagline slot.
  - 2048 is the box the top picture asks for and the width of both seeded top-picture files.
  - It reaches production with the Phase 19 reinstall, like the rest of the seed (D11).
  - A larger box is still capped, so nobody can make the server produce an arbitrarily large picture.
- *Not changed:* the seeded desktop file is 2048 × 624 (50 KB). On a screen wider than 2048 device pixels, a high-density laptop for instance, it is still stretched a little. A sharper result there needs a larger original. Uploading one under **Page Content** → the desktop hero's **Image** would also need the limit raised to match.

**Tests:**
- [x] Karma, `home-content-sections.component.spec.ts`:
  - the headline renders whole, with its clause held together
  - a hero row with a body never draws it
  - `keepClosingClauseTogether`: the seeded headline; only the last comma's clause; no comma, or one at the end; a four-word clause left alone
  - the lines are balanced
  - in real viewports from 320 to 2560 px, every 40 px: "life, the right way." always shares one line
    - *Fixed in 16.7 (10/22):* this failed on Linux, as CI runs it, at 320 px only. The runner has neither Barlow nor Arial, and in its wider fallback font the clause doesn't fit a 288 px line. Each width now checks first whether the clause fits in the font the browser has, and only a width under 360 px may be let off.
- [x] Karma, `page-content.component.spec.ts`: a hero row offers no **Body**, and a feature row still does.
- [x] Playwright, `layout/home.spec.ts`: with the real typeface, at 18 widths from 320 to 2560 px (both top pictures), "life, the right way." always shares one line. Screenshots `top-picture-360.png`, `-1680.png` and `-1920.png`.
- [x] C++, `SeedAppDataTest.PopulatePublicPhotoSizeLimitRaisesTheFrameworkCap`: starting from honuware's own defaults (taken from the framework, and checked to be 800), both settings read 2048 through `SecretsHelper`, the reader `/api/get_scaled_photo` clamps with.
- [x] End to end (scratchpad `phase14_e2e.sh install`), the real server on a fresh `--install_schema`: the desktop top picture comes back 2048 × 624, the mobile one 2048 × 969, and a 4096 box is still capped at 2048. *PASS, 17 of 17 checks.* The SVG upload checks pass against the pinned `920f18a` too.

**Results:**
- [x] Mutations: 8 of 8 caught.
  - UI, 5:
    - no balancing
    - the accent line drawn again
    - **Body** offered on a hero again
    - the raw title printed instead of the joined one
    - a four-word limit instead of three
  - Server, 3:
    - the seed writes 800
    - only the width is raised
    - the install never calls the step, which only the end-to-end run catches: its three picture checks fail
- [x] Angular: `ng test` **3656 passed**. `tsc` for the app, the specs and the Playwright specs, `ng lint` and `ng build` are clean, with the same six size-budget warnings.
- [x] Playwright: home, header and tap targets, **29 passed**.
  - The header spec failed again on a dev server Playwright started itself. It passed on one started by hand, as in 14.8.
- [x] **knottyyoga Linux: 5405 passed** (pinned `920f18a`; 5404 plus the new test), with both executables built.
- [x] **knottyyoga Windows: 5405 passed**, the same test names as Linux. `knottyyoga_the_server` wasn't rebuilt because it was running; every other target was.

---

# Phase 15 — Two layout fixes on phones

**Planned 10/21/2026.** All in `ui/`. Both were reproduced on the local site at 390 px on 10/21, signed in as Kit with no membership.

**Implemented 10/22/2026, not committed.** No server change, so no C++ build or gate was needed.
- Angular: `ng test` **3669 passed**, 13 more than after Phase 14. `tsc` for the app, the specs and the Playwright specs, `ng lint` and `ng build` are clean, with the same six size-budget warnings.
- Playwright: every layout spec at all seven audit sizes, **256 passed**. The 332 skipped are size-specific tests.
- Mutations: 9 of 9 caught by the specs, and both new Playwright specs fail against a broken build.

**Still to do:** Mason commits, then hand-tests 15.5.

### 15.1 Empty-state icons keep their size *(ui: assets/styles/_patterns.scss)*
- **Cause.** The two reports are one bug.
  - `.empty-card` is a flex row with no rule for its icon, so the long sentence beside it squeezes the 24 px `mat-icon` to a sliver.
  - The icon's own `overflow: hidden` then cuts the glyph off.
  - On the home page's **Upcoming classes** band it reads as clipped. On **My Schedule** the sentence sits over what is left of it.
- [x] **Fix:** `.empty-card mat-icon { @include icons.size(icons.$md); flex: none; }` in `_patterns.scss`. `.notice` already does this for its own icon (`_surfaces.scss:118`).
  - It also fixes every other `.empty-card` with an icon: Today's Classes, Attendance History, Class Check-In (twice), Student Notes and the style guide's demo.
- [x] Specs:
  - `style-guide.component.spec.ts`, for the global class: the `.empty-card` demo in a 260 px-wide host keeps a 24 × 24 icon, wholly to the left of the text.
  - `upcoming-classes` and `my-schedule` specs: the same check on their own empty states, at phone width.
  - *As built:* the two page checks are at 358 px, a 390 px phone less the page gutter.
- [x] Playwright, at 390 px, as a signed-in customer with no eligible classes: the home band and My Schedule, with the audit's overlap check. Screenshots kept for 15.5.
  - *As built:* new `layout/empty-states.spec.ts`, at all four phone sizes. Each checks:
    - a 24 × 24 icon wholly left of the text
    - no audit finding of text clipped, crammed or drawn over in that component
  - Screenshots are `output/empty-states/home-upcoming-classes.png` and `my-schedule.png`, at 390 px.
  - The mock had no member without classes, so a new mock switch, `knottyyoga.mockNoEligibleClasses` (`NO_ELIGIBLE_CLASSES_KEY` in `harness.ts`), empties the eligible classes and with them today's and upcoming ones. It has its own `ServerAccess.mock.spec` test, and the audit README lists it.

### 15.2 The month calendar fits the screen *(ui: pages/calendar/components/views/month-view)*
- **Today.** The month grid has a fixed 800 px minimum width (`month-view.component.html:31`, `min-w-[50rem]`) inside a box that scrolls sideways. On 10/21 it measured 808 px wide at both 390 and 768 px.
  - This was deliberate in the mobile pass (Mobile UI Updates 2.5, "Makeover OQ 19"), and `layout/calendar.spec.ts:21` asserts it. This item reverses it.
- [x] **No minimum width.** The seven columns share whatever width there is (`repeat(7, minmax(0, 1fr))`), at every size.
  - *As built:* the 800 px box and its sideways scroll are gone. The weekday row's `ml-2` became `pl-2`: on a full-width row it would have pushed Saturday 8 px past the edge.
- [x] **Narrow day cells**, decided by a container query on the grid's own width so a spec can size it:
  - each class chip shows the class name only, in small type, cut off with "…"
  - the time and the status icons drop out; both are on **Day View** and on the class's own page
  - cells are shorter
  - up to three chips a day, then **+N more**, which opens that day in **Day View**
  - *As built:*
    - **Threshold:** the cells turn compact when the grid is narrower than 800 px, the width it used to be held at. That covers phones (a 358 px grid at 390) and the 768 px tablet (704 px). A 1024 px window gets an 896 px grid and keeps today's cells.
    - **Type:** the small type is a new token, `--text-2xs` (11 px), since a phone's day is about 50 px wide. `design-tokens.spec.ts` lists it in the type scale.
    - **Height:** compact cells are 6.5 rem (104 px), against 128 px.
- [x] **Tapping a date number** opens that day in **Day View**. Today it only selects the day, for the next switch of view.
  - *As built:* `openDay` remembers the date, as a tap on the day already did, and switches to `?view=day`, which opens on it.
  - The number keeps its 24 px circle, with a 44 px touch area around it, the way Material sizes its own buttons. So it is easy to hit on a phone, and the audit doesn't flag it.
  - A tap elsewhere in the day still only selects it.
- Unchanged:
  - From 1024 px up nothing changes: cells there are already wider than the threshold.
  - Phones still open on **Day View**. This is for someone who picks **Month View**.
  - **Week View** still scrolls sideways on a phone (OQ-33; Mason kept that default).
- [x] Specs (Karma, month view):
  - with the host 358 px wide, the grid is no wider than the host, and chips show the name only
  - **+N more** appears after three chips, and opens **Day View** on that date
  - with the host 1000 px wide, the chips show the time and icons as today
  - *As built*, also:
    - at 1000 px every class is listed, with no **+N more**
    - a compact day is shorter than a wide one
    - the date number opens **Day View** on its date, while a tap elsewhere only selects the day
    - the date number's tap area is at least 44 × 44
  - The old test that pinned the 800 px scroll box now checks only that the header sits outside the grid.
- [x] Playwright `layout/calendar.spec.ts`:
  - the "scrolls sideways" test becomes "fits without scrolling", at every size
  - the chip-icon test applies only where cells are wide
  - screenshots kept for 15.5
  - *As built:*
    - **Fits:** Saturday's right edge is on screen, and neither the grid nor the page around it scrolls sideways.
    - **New, below 1024 px:** every visible chip is inside its day, ends in "…" and shows no time or icon.
    - **New, at 390 and 1280 px:** tapping the 20th opens **Day View** on October 20, 2026.
    - **Screenshots:** `output/calendar/month-phone.png`, `month-tablet.png` and `month-desktop.png`.

### 15.3 Mutation check
- [x] Planned:
  - the icon rule removed
  - `flex: none` dropped, keeping the size
  - the 800 px minimum restored
  - compact chips at every width
  - **+N more** never shown
  - **+N more** opening the wrong date
  - *As built: all six caught, plus three more:*
    - the date number only selecting the day
    - the touch area shrunk to the circle
    - the mock ignoring its new switch
  - *Each failed the spec written for it, and a clean run passed 744.*
  - *Against a live dev server, two of them were also tried on the new Playwright specs:*
    - the icon rule removed: 8 failures in `empty-states.spec.ts`
    - the 800 px minimum restored: 10 failures in `calendar.spec.ts`

### 15.4 Verification
- [x] Angular: `ng test`, `tsc` for the app and the specs, `ng lint` and `ng build`.
  - *As built:* `ng test` **3669 passed**, 3656 + 13. `tsc` for the Playwright specs too. All clean, with the same six size-budget warnings.
- [x] The Playwright layout specs at all seven audit sizes.
  - *As built:* **256 passed**, with 332 size-specific tests skipped. They ran against a dev server started by hand, since one Playwright starts itself has twice served a stylesheet with no `xl:` classes ([[gotcha_angular_cache_drops_tailwind]]).
- [x] No server change.

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

**Implemented 10/22/2026, not committed.** The pager is on all 48 lists in 16.3 and 16.4, plus "Your booked sessions" (see 16.4).
- Angular: **3786 passed**. `tsc` for the app, the specs and the Playwright specs, `ng lint` and `ng build` are clean, with the same six size-budget warnings.
- Playwright: `pagination.spec.ts` **54 passed**, 27 pages at two sizes. The other layout specs at all seven sizes, **256 passed**.
- Mutations: **9 of 9 caught**.
- Screenshots: all **192** read, with the findings in 16.5.

**Linux, as CI runs it:** **3786 passed**, after one fix to a 14.10 test (16.7). OQ-36 and OQ-37 kept their defaults, so nothing changed for them.

**Still to do:** Mason commits and hand-tests 16.8.

**What exists** (inventoried 10/21):
- Only four lists page today:
  - **Blog**: 5 a page, paged on the server
  - **Attendance History**: 25 a page, paged on the server
  - **Admin** ▸ **Blog Posts**: 10 a page, paged on the server
  - **Manage Data**: pages already
- There is no shared pager. The first two are hand-written, and the third is a Material paginator.
- About sixty lists show every row. Many Manage pages ask the server for "the first 100" (or 1,000, or 10,000) rows and silently drop the rest (Phase 17.1).

**The rules** (OQ-32, confirmed by Mason on 10/22):
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
- [x] `app-list-pager`: **‹ Previous** · "1–10 of 23" · **Next ›**, with 44 px buttons, hidden when the list fits on one page.
  - *As built:* `shared/components/list-pager/`.
    - **Two ways to drive it:** a list paged in the browser passes its `ListPaging` and its row count; a list paged on the server passes `pageIndex`, `pageSize` and the server's total, and fetches on `pageChange`.
    - **The label** has a fixed minimum width with even-width digits, so the buttons stay put from page to page (found in the 16.5 screenshots).
- [x] `pagedSlice(items, pageIndex, pageSize)` and a small `ListPaging` state:
  - it keeps the page in range when the list shrinks: deleting the last row of page 3 goes to page 2
  - it goes back to page 1 when a filter or search changes
  - it brings the list's top into view on a page change, so a phone user isn't left at the bottom
  - *As built:*
    - **Shrinking lists:** `ListPaging` stores the page asked for and clamps on every read, so a list that shrinks needs no code of its own, and nothing changes during change detection.
    - **Filters:** a filter calls `reset()`.
    - **Scrolling:** the list's top comes back into view only when it has scrolled above the screen.
- [x] `MAIN_LIST_PAGE_SIZE = 10` and `NESTED_LIST_PAGE_SIZE = 5`, in one place (`list-paging.ts`).
- [x] **Grouped lists**, such as days or a series' sessions, page over their items. A group cut by a page boundary repeats its heading on the next page.
  - *As built:* `ListPaging.groups()` and `groupedItemCount()`. A series inside **My Bookings** is one row of that list, with its own pager over its sessions.
- [x] **One look for every paged list.** The pager replaces the two hand-written ones (**Blog**, **Attendance History**) and **Blog Posts**' Material paginator.
  - All three keep paging on the server.
  - **Attendance History** asks for 10 a page instead of 25.
  - *As built:* Blog no longer shows two disabled buttons when everything fits on one page. Like every list, it shows no pager at all.
- [x] Specs:
  - the helper: slices, the last partial page, an empty list, and clamping
  - the component: its labels, the ends disabled, hidden on a single page, the event it emits
  - at phone width, one row with no wrapped labels (count the label's lines, per the audit harness's note)
  - *As built:* `list-paging.spec.ts` (10 tests) and `list-pager.component.spec.ts` (9). The pager's also cover:
    - a list that shrinks under it
    - server mode
    - bringing the list's top back into view
    - the buttons staying put
  - **Every paged list's own spec**, 48 lists, checks:
    - with 23 rows: a page of rows, "1–N of 23", Next and the last page
    - with a short list: no pager
    - where a list has them: a filter going back to page 1, and a removal on the last page showing the page before

### 16.2 Many-row mock data *(ui: ServerAccess.mock, e2e harness)*
- [x] `knottyyoga.mockManyRows=1` in local storage makes every list method the Phase 16 and 17 pages use return 23 rows. That is three pages at 10, and five at 5, so every page can be photographed mid-list. It is off by default, so no existing spec or audit changes. Tests in `ServerAccess.mock.spec.ts`.
  - *As built:* `MOCK_MANY_ROWS_STORAGE_KEY` (`MANY_ROWS_KEY` in `harness.ts`, documented in the audit README).
    - **How the rows are made:** `seedManyRows()` grows each list from its seeded rows, with numbered names that sort in order. Lists the mock builds on read (an instructor's classes, upcoming classes, sign-up offerings, shared memberships) check the flag in their own method.
    - **Audit data:** it is meant to be used without the audit data, whose records would add to some counts.
    - **Test:** one test counts all 41 list methods at 23 with the flag, and checks two stay short without it.
    - **Phase 17's lists** join it then.

### 16.3 Public pages
- [x] Each of these gets the pager:

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

  - *As built*, in each list's own spec:
    - **Shared components:** the "Series & workshops" card and the tier cards page inside their shared components, so `/classes`, `/shop/subscriptions` and the home page get the same pager.
    - **Series & workshops** goes back to page 1 only when its list really changes: its hosts pass a new array on every screen update.
    - **All Classes:** a tag filter goes back to page 1.
    - **A class page:** both lists go back to page 1 when the page moves to another class. The five-date cap and its "Showing the next N dates" note are gone: every date can be reached, five a page.
    - **A product's page:** a variant chosen on page 2 stays chosen.
  - *Found:* an instructor's page doesn't reload when the address changes from one instructor to another. That predates Phase 16 and isn't changed here.

### 16.4 The user portal
- [x] Each of these gets the pager:

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

  - *As built:*
    - **Added:** "Your booked sessions", the list above **Upcoming** on **My Schedule** and on the home band, also pages, at 5. With many bookings it was a long unpaged list on both. It is one of several sections, so it takes 5 (OQ-32).
    - **"Booked for Someone Else"** on **My Purchases** stays unpaged, as planned.
    - **Paging survives a reload:** each purchase's and each series' paging is kept by its id, so it survives the page reloading its data after a cancel or a seat change. A series' open or closed state survives paging away and back.
    - **Removing a row:** on **Saved Cards**, **My Bookings**, **Notifications** and **Favorite Instructors**, removing the only row on the last page shows the page before (each has a spec).
    - **New spec file:** **Saved Cards** had none, so `card-management.component.spec.ts` is new.
    - **Purchase dividers:** a purchase's entitlements used `:last-child` to drop the last divider. The pager is now the card's last child, so it is `:last-of-type`, with a test.

### 16.5 Screenshots (Ryan's check)
- [x] New `e2e/mobile-audit/layout/pagination.spec.ts` visits every page in 16.3 and 16.4 with many-row data, at 390 and 1280 px. On each it checks:
  - no more rows than the page size
  - the pager reads "1–10 of 23"
  - **Next** moves on to "11–20", and the last page holds 3
  - the audit's overlap and overflow checks pass
  - *As built:*
    - **What it covers:** 27 pages and 48 lists, plus **Blog Posts** ("of 25": its four seeded posts too).
    - **How it finds each list:** by its `data-paged-list` key, its rows by `data-paged-row`, and its pager as that element's own child, so a nested list is never confused with its parent.
    - **Each list from a fresh load**, since the one before it is left on its last page.
    - **The audit's checks** run from the top of the page, as the audit itself does.
- [x] Page 1 and the last page of each are saved to `ui/e2e/mobile-audit/output/pagination/`. Claude reads every one, lists anything that looks wrong here, and keeps the folder for Mason and Ryan.
  - *As built:* **192 pictures, all read.**
    - **Capture:** the first take was blank below the first screenful, because the app scrolls inside its own frame. Each picture is now taken with the window as tall as the page, the pointer off the rows, and a 16 px margin. Every pager shows, on one row, with its disabled button greyed.
  - **Fixed here**, each with a test:
    1. **The pager's buttons moved** as the label widened from "1–5 of 23" to "21–23 of 23". The label now keeps one width.
    2. **Several booked sessions sat border to border**, the first against its heading. They now have the 8 px gap a day's classes have.
  - **Left as they are** (Mason kept the defaults of OQ-36 and OQ-37):
    - two pagers close together on **My Bookings**
    - one card alone on the last row of the three-column grids
  - **Pager details**, cosmetic and left as they are:
    - **Centred under left-aligned content:** the pager is centred under the list, so under content that doesn't fill the width it sits off to the right: an instructor's class chips, the Gallery's album tiles (narrow because their photos don't load in the mock), and a product's variants.
    - **The gap above the pager** is about 16 px on most lists and 32–50 px where the rows carry their own bottom margin (**Our Providers**, **Upcoming classes**, **Workshops & Series**, Gallery, **Our Location**).
    - **Enabled looks quieter than disabled:** the enabled buttons use the site's grey "quiet" style and the disabled ones a plain outline (the global button design, unchanged).
  - **Seen on the pages, unrelated to paging** (they predate Phase 16):
    - **Blog Posts on a phone:** the table is wider than the screen, with the status badges cut off and the date and action columns off-screen. On a desktop its edit and delete icons stack, and its columns change width from page to page.
    - **Narrow text breaks on a phone:**
      - a number split from its unit ("90 / minutes")
      - "AM" alone on a line in entitlement dates
      - "Main / Room"
      - "VISA •••• 4002 |" ending in a lone bar
      - a series' title squeezed into two columns on **My Bookings**
    - **Faint expand chevrons** on **My Bookings** and **My Purchases**.
    - **Uneven card heights:** cards without a photo stretch to match one with a photo (**Services**, **Upcoming Events**), and **Our Providers**' "View profile" buttons sit at different heights.
    - **Saved Cards:** the default card's pencil sits where the others' star is.
    - **My Schedule:** day headings start at the band's edge, while the cards below are indented.
    - **Our Location on a desktop:** maps for locations without a photo are full-width and about 730 px tall.

### 16.6 Mutation check
- [x] Planned:
  - the pager showing every row
  - a page size of 25
  - the pager showing on a single page
  - a filter change not going back to page 1
  - deleting a page's last row leaving an empty page
  - a cut group not repeating its heading
  - **Attendance History** asking for 25
  - *As built:* **all seven caught**, plus two for the 16.5 fixes: the label's fixed width, and the booked sessions' gap. That is 9 of 9.
    - The tests that caught each: the helper's and the pager's specs, and the **Instructors**, **All Classes**, **Saved Cards**, **Upcoming classes** and **Attendance History** specs.
    - A clean run passed 112.

### 16.7 Verification
- [x] Angular: `ng test`, `tsc` for the app and the specs, `ng lint`, `ng build`.
  - *As built:* `ng test` **3786 passed** (3669 + 117). `tsc` for the Playwright specs too. All clean, with the same six size-budget warnings.
  - **One test changed** on the way: 14.10's headline sweep built 57 iframes and outran Jasmine's 5-second limit in the full suite. It now reads one iframe resized per width, through the new `acrossViewports()` in `shared/testing/in-viewport-of.ts`, which has its own spec.
- [x] **Linux, as CI runs it** (10/22, after OQ-36 and OQ-37): `ng test` in `node:22-bookworm` with Chromium.
  - *As built:* the first run failed **one** test, 14.10's headline sweep at 320 px.
    - **Why:** that version is already in the Phase 14 commit, so CI has probably been failing it since then. The runner has neither Barlow nor Arial, and in its wider fallback font "LIFE, THE RIGHT WAY." is wider than the 288 px line, so it split "WAY." mid-word.
    - **On a real phone:** in Barlow or Arial the clause is about 264 px and fits.
    - **The fix:** each width now first checks whether the clause fits on one line in the browser's font, and only widths under 360 px (the narrowest phone the audit checks) may be let off. No change to the page.
    - **Mutation checks:** a phone headline too large for the clause now fails at 360 px and up instead of being let off. With both fixes removed (no-break spaces and balanced lines), the sweep fails at 40 widths. Each fix alone keeps the clause together, so removing one is caught by its own one-line test.
  - **Windows: 3786 passed. Linux: 3786 passed.**
- [x] The pagination spec at both sizes.
  - *As built:* **54 passed**. The other layout specs at all seven sizes: **256 passed**.
  - Both ran against a dev server started by hand, as in 15.4.
- [x] No server change.

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

**Implemented 10/23/2026, not committed.** The pager is on 51 lists across 33 pages (17.2 and 17.3), and every silent cap in 17.1 is gone.
- Two lists in the tables below are not paged, each for a reason given in 17.3: **Announcements** and the pricing views.
- Results: see 17.6.
- Screenshots: all **212** read, with the findings in 17.4.
- **New open questions:** OQ-38 (table columns that shift from page to page) and OQ-39 (a product page's latest-10 lists have no **View All**).

**Still to do:** Mason commits and hand-tests 17.7 and 17.8. On 10/23 he kept OQ-38's default and chose OQ-39's alternative, which is new item **17.8**, built the same day and not committed.

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

- [x] Each asks for everything (page size 0, which the endpoint already treats as "all rows") and pages in the browser (D15).
  - Not changed: a product page's **Event Sessions**, **Active Subscriptions** and **Recent Purchases**, which deliberately show the latest 10 with a **View All** link.
  - *As built:*
    - **Page size 0 is "every row"** in the server's SQL (`LIMIT NULLIF($1, 0)`, honuware's `GetRowsByValues`). The mock now does the same, with a test.
    - **Every cap in the table is gone:** Products, Price Schedules, Event Sessions, My Sessions, Subscriptions, Entitlements and its two joins, Event Payment Info, a subscription's billing history, the pricing views and Subscription Revenue.
    - **Found on the way, same kind:**
      - a session card's attendees (100) and its price overrides (100)
      - the product filters of Event Sessions (100), Subscriptions (100) and Entitlements (200)
      - the pricing views' variants (1,000)
      - a product page's prices, schedules, variants and active entitlements (100 each)
    - **A second kind of request:** three lists read through `get_rows_by_column`, whose SQL is `LIMIT $1`, so 0 there means no rows at all. They now use `get_filtered_table_rows` with no filter:
      - the pricing views' prices (10,000) and permissions (100)
      - a product page's permissions, which its booking windows page over (100)
    - **Left alone:** one-row lookups, and the three latest-10 lists. Only an event product's **Event Sessions** has a **View All**; the other two have none (OQ-39).
      - → 10/23, Mason: page those two. That is **17.8**, now built.
- [x] Specs: each page's request asks for everything, and its eleventh row is reachable on page 2.
  - *As built:*
    - **Every changed request** has a spy checking page size 0.
    - **Products:** a 101st product appears.
    - **The pricing views:** a 201st product, a 101st schedule and a 10,001st price all reach the view.
    - **Subscription Revenue:** a 10,001st charge counts in the total, and a 10,001st subscription in the past-due count.

### 17.2 The staff portal
- [x] Each of these gets the pager:

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

  - *As built*, each list with its own specs:
    - **The session card** is one shared component, so **My Sessions** and **Manage** ▸ **Event Sessions** get the same three pagers.
    - **Filters:** on both session pages the year and month filters go back to page 1. On **Event Sessions**, so do status, product and instructor.
    - **Person Skills:** choosing another person puts both lists back on page 1. Both lists stay at 10 a page, as the table says.
    - **My Service Bookings:** the provider schedule links to one booking, and the list now opens on that booking's page, expanded. Without this, a booking on page 2 would open out of sight.
    - **Student Notes** is the page the dashboard tile of that name opens ("Notes from your students").
    - **Shift Requests:** the client rows' last divider used `:last-child`, which the pager now breaks, so it is `:last-of-type`, with a test.
    - **Not paged:** the booking lists that appear only after **Accept** or **Submit** on **Shift Requests** (staff) or **Review** (Manage). The plan named only the affected clients and bookings.

### 17.3 Manage
- [x] Each of these gets the pager:

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
- *As built*, each list with its own specs:
  - **Not paged, against the table:**
    - **Announcements** are ordered with ↑ ↓, like the Page Content rows, so OQ-32 leaves them unpaged.
    - **The pricing views** are a report (OQ-32). Only their caps went (17.1).
  - **Scheduling Exceptions** page over their cards, ten a page. Each card is one run of blocked dates, and the page shows no group headings, so there is no heading to repeat.
  - **Tables:** **Products** and **Price Schedules** are Material tables. Each is given one page of rows, cut once per change, so the table doesn't redraw on every check.
  - **Adding and editing:**
    - On **Instructors**, **Skills** and **Class Tags**, an edit or a cancelled **Add** comes back to the page it left.
    - On **Membership Tiers**, a toggle keeps the page.
    - On **Image carousels**, a new carousel's page is shown, with its photo panel open.
  - **Filters go back to page 1:**
    - **Subscriptions:** status, year and month
    - **Entitlements:** status, search and year
    - **Vouchers** and **Coupons:** search, and the date filters
    - **Class Schedules:** another class
    - **Room Schedules:** another room or product
    - **Attendance Templates:** another member
    - **Opening another record:** another subscription, or another voucher's or coupon's redemptions, starts at page 1.
  - **Removals:** on every list where a row can go (a delete, an approval or a denial, an unblock, a dismissed alert), removing the only row on the last page shows the page before.
  - **Room Occupancy** refreshes every 30 seconds and keeps its page.
  - **New spec file:** Room Occupancy had none.

### 17.4 Screenshots
- [x] 16.5's spec extends to every page in 17.2 and 17.3, with the same checks and the same folder. Claude reviews each picture.
  - *As built:*
    - **Many-row data:** the many-rows mock (16.2) now fills every Phase 17 list with 23 rows, one seeding method per group of pages. Each group has a test in `ServerAccess.mock.spec.ts` that counts its lists at 23. Those tests also check that every Phase 16 list is still 23.
    - **Totals other than 23:**
      - **Vouchers** read 46: the voucher list holds My Vouchers' 23 active and 23 spent.
      - **Image carousels** read 24: the three seeded and the Gallery's albums.
    - **What the spec covers:** 33 pages and 53 lists. The session card's attendees and staff are checked on both session pages.
    - **Steps before a list shows:** some lists need one first, such as choosing a person, opening a card or a tab, or showing inactive schedules. The spec does them, moving the pinned clock on after a typed search.
    - **All 212 pictures were read**, by four reviewers.
  - **Fixed here**, each with a test:
    1. **Shift Requests** (Manage): a request's notice, "Approving will reassign these bookings…", sat 8 px under the bookings' pager, which then read as part of the notice. It now keeps the pager's own 16 px when the pager shows.
    2. **My Service Bookings:** the **Past** heading sat against the Upcoming pager. The template's 24 px had never applied (the component's own margin beat it), so before paging it touched the last card.
  - **Left for OQ-38:** the columns of **Products**, **Membership Tiers** and a session card's **Staff** table shift by 25–175 px between pages. Each table sizes its columns to the rows it shows. **Blog Posts** has done the same since Phase 16.
    - → 10/23, Mason: leave it.
  - **Seen on the pages, unrelated to paging** (they predate Phase 17, and are not changed):
    - **Cut off on a phone:**
      - **Entitlements:** the rows' text is squeezed into narrow columns and clipped top and bottom ("60-/Minute/Massage").
      - **Class Schedules:** the instance, schedule and slot headers have a fixed height, so names and times lose their first and last lines, an icon is drawn over another, and a slot's delete icon is missing.
      - **A session card's Staff table** is wider than the card, with **Notes** cut off; unlike the other tables, it doesn't scroll sideways.
      - **My Service Bookings:** past bookings' titles are clipped by the panel header.
    - **Squeezed on a phone:**
      - the text on **Image carousels**, **Student Notes** and **Event Sessions** (where the date runs into the booked count: "6:00 PM0 / 20")
      - the buttons on **Vouchers** ("View Redemptions") and a provider's templates, which wrap and fill the button
      - **Scheduling Exceptions:** the badge breaks as "STUDIO- / WIDE"
      - the info and warning icons on both **Shift Requests** pages, and the calendar icon on the session cards
    - **Run together:** on **Shift Requests** (staff), a client's service and email ("Massageclient1@example.com").
    - **Alignment:**
      - the session cards touch each other (each card's `:last-child` rule matches itself)
      - availability blocks' times, and a session's price-override boxes, don't line up from row to row
      - the **Requires Attention** alerts sit centred in their cards (the page's `.alert-card` shares the global class's name)
      - in several tables only the first header cell is tinted, and on **Membership Tiers** the pinned first column paints over the green of an offered tier
      - dates in phone tables break mid-range ("Mar 1, 2025 – Mar / 31, 2025")
    - **Behaviour:**
      - **Event Payment Info** never finishes loading when none of a session's bookings has a purchase item.
      - **Room Schedules** shows the previous room's product overrides until the new ones arrive.
      - **Subscription Revenue**'s past-due and cancelled "revenue" fields hold counts, not cents.

### 17.5 Mutation check
- [x] Planned:
  - a cap restored (`page_size` 1000)
  - **Subscription Revenue** reading 10,000 again
  - a nested list at 10 instead of 5
  - a session card's attendees unpaged
  - the year and month filters not going back to page 1
  - *As built:* **all caught**, plus the year and month filters on both session pages and the two 17.4 fixes. That is **8 of 8**:
    - **My Sessions** asking for 1,000 again: its page-size-0 test.
    - **Subscription Revenue** reading 10,000 charges: two tests, including the 10,001st charge in the total.
    - A session card's staff at 10 a page: three tests.
    - Its attendees unpaged: two tests.
    - The year and month filters keeping the page: one test each on **My Sessions** and **Event Sessions**.
    - The notice's 16 px under the pager, and the **Past** heading's 24 px: one test each. Without the fix the heading measured 0 px from the pager.
    - Every file was restored byte for byte afterwards.

### 17.6 Verification
- [x] Angular: `ng test`, `tsc` for the app and the specs, `ng lint`, `ng build`.
  - *As built:* **Windows: 3984 passed. Linux: 3984 passed** (3786 + 198). Linux ran as CI does: `ng test` in `node:22-bookworm` with Chromium.
  - `tsc` for the app, the specs and the Playwright specs, and `ng lint`, are clean. `ng build` has the same six size-budget warnings.
- [x] The pagination spec.
  - *As built:* **120 passed**, Phase 16's 27 pages and Phase 17's 33, at both sizes.
  - With the other layout specs at all seven sizes: **376 passed**, 632 skipped, none failed. The other specs' 256 are the same as in Phase 16.
  - Run against a dev server started by hand, as in 15.4.
- [x] No server change.
  - *As built:* page size 0 already meant "every row" on the server (17.1), so nothing changed there.

### 17.7 Hand-testing walkthrough

**Setup.** A fresh database, the server and `ng serve` running, with a window at least 1280 px wide. On a fresh database only three of Phase 17's lists are long enough to page; the screenshots cover the rest.

1. **Sign In** as `masonbendixen@gmail.com` / `pass`. **Admin** → **Manage Products** → **Membership Tiers** tile.
   - The table shows ten permissions, then "‹ Previous · 1–10 of 13 · Next ›" under it.
   - Click **Next ›**: "11–13 of 13", with **silver_member**, **staff_access** and **view_admin_instructor_load**. **Next ›** is greyed.
   - Tick **Offered for pricing** on one of the three. The page stays on "11–13 of 13".
   - Untick it again, so the database is as it was.
2. **Admin** → **Manage Products** → **Service Providers** tile → **Availability** on **Caleb Ault**'s card.
   - **Provider Availability** shows ten blocks and "1–10 of N". N is about 28; it depends on the day the database was made, since the seed fills from that day to the end of next month.
   - **Next ›** moves on to "11–20 of N".
3. **Admin** → **Manage Products** → **Room Occupancy** tile. On the **Main Gym** card, **Projected occupancy:** shows five rows and "1–5 of 9".
   - Click **Next ›**: "6–9 of 9".
   - Wait 30 seconds for the page's refresh: it stays on "6–9 of 9".
4. **Admin** → **Manage Products** → **Instructors** tile: five instructors and no pager. A list that fits on one page looks as it did.
5. Open `ui\e2e\mobile-audit\output\pagination\`. The pictures whose names start `staff-` and `manage-` are one phone and one desktop picture of page 1 and of the last page, for every list in 17.2 and 17.3.

### 17.8 A product page's subscriptions and purchases, five at a time *(ui; added 10/23 from OQ-39; built 10/23, not committed)*

Today a product page's **Active Subscriptions** (a membership's page) and **Recent Purchases** (a one-time product's page) show the latest 10, with no way to reach the 11th. Mason chose to page them.
- [x] Both ask for every row (page size 0), newest first as today, and page **5** at a time on the shared pager, like the page's other nested lists (booking windows, the entitlement summary).
  - *As built:* the lists are keyed `product-subscriptions` and `product-purchases` for the specs. Each goes back to page 1 when the product loads.
  - The last subscription card kept a 12 px gap under it once the pager followed it: its "no margin on the last card" rule matched only the last child. It now matches the last card.
- [x] **Active Subscriptions** looks up each row's name, latest charge and entitlement only for the five rows shown, when their page is shown, and keeps what it found.
  - Today it does that for every row it loads, up to three requests a row. Over every subscription of a busy membership, that would be hundreds of requests at once.
  - *As built:* the page's spinner stays up until page 1 has its names, as before. On a later page a card shows "Loading…" for its name until the lookup answers.
  - A subscriber with several subscriptions is looked up once, and two subscriptions sharing an entitlement fetch it once.
  - A row whose lookups fail shows "Unknown", as before.
- Not changed:
  - An event product's **Event Sessions** keeps its latest 10 and **View All Event Sessions**.
  - **Active Subscriptions** still lists every status, each with its badge.
- [x] Specs:
  - Each request asks for everything.
  - The sixth row is on page 2.
  - With 23 subscriptions, page 1 looks up five names, **Next ›** looks up the next five, and **‹ Previous** looks up nothing.
  - *As built,* in the product page's spec, eight in all:
    - both requests ask for page size 0; the two latest-10 specs they replace said 10
    - each list: page 1, the sixth row first on page 2, the last three on page 5, and no pager at five rows
    - the lookups: the people asked for are exactly 101–105, then 101–110, and still 10 after **‹ Previous**; the charges likewise
    - a subscription on page 2 gets its entitlement ("6 seats"), and one on page 3 has been looked up for nothing
    - the last card has no margin under it, and the others do
- [x] Screenshots: the many-rows mock gives a membership and a one-time product 23 rows each. The pagination spec photographs both lists, with 17.4's checks.
  - *As built:* the mock's 23 subscriptions were already all product 3's, **Monthly Membership**. Product 2, **60-Minute Massage**, gets 23 purchases in a `purchase_items` table, which the mock never had. The mock's spec counts both.
  - The pagination spec photographs product 2's **Recent Purchases** with its other lists, and product 3's **Active Subscriptions** as a new page.
  - All eight pictures read: each list shows "1–5 of 23", then "21–23 of 23" with three rows.
    - The last page of subscriptions includes the cancelled one, with its badge.
    - On a phone, **Recent Purchases** scrolls sideways inside its own box, as the page's other tables do.
    - **Found:** on a phone, a subscription card's header wraps the name and the billing period onto two lines each. It predates 17.8: **OQ-41**.
- [x] Mutation check, planned:
  - the 10-row cap back
  - every row's name looked up at once
  - *As built:* **4 of 4 caught**, each on its own run, and each file restored byte for byte:
    - **Active Subscriptions** asking for 10: its page-size-0 spec
    - **Recent Purchases** asking for 10: its page-size-0 spec
    - every subscription enriched at load: the lookup spec and the page-3 spec
    - the last card's margin by `:last-child` again: the margin spec
- [x] Verification: Angular `ng test` on Windows and Linux, `tsc`, `ng lint`, `ng build`, and the pagination spec. No server change.
  - *As built,* run together with 13.11's card:
    - **Windows: 3992 passed. Linux: 3992 passed** (3984 + 8). Linux ran as CI does: `ng test` in `node:22-bookworm` with Chromium.
    - `tsc` for the app, the specs and the Playwright specs, and `ng lint`, are clean. `ng build` has the same six size-budget warnings.
    - Playwright: all the layout specs at all seven sizes, **378 passed**, 637 skipped, none failed. The pagination spec is 122 of them (120 + the new membership page at both sizes). Run against a dev server started by hand, as in 15.4.

**Hand-testing walkthrough.** On a fresh database neither list has more than one row, so the screenshots show the paging. This checks by hand that the names and entitlements still arrive. A fresh database, the server and `ng serve` running, a wide window:
1. **Sign In** as `kiteagle@gmail.com` / `kit`. **Memberships** → **Knotty Yoga Silver Membership** → **Subscribe**. Leave **Start month** as it is, enter the sandbox card `4111 1111 1111 1111` (any future expiry, CVV `111`), and click **Subscribe & Pay $160.00**.
2. Kit's first-name menu → **Sign Out** → **Log Out**. Sign in as `masonbendixen@gmail.com` / `pass`.
3. **Admin** → **Manage Products** → **Products** → **Knotty Yoga Silver Membership**. **Active Subscriptions** shows one card:
   - **Kit Williams**, marked **active**
   - **Next Billing** with a date
   - **Entitlement** "1 seat", **Valid** with two dates, and **Entitlement Status** **active**

   There is no pager.
4. Open `ui\e2e\mobile-audit\output\pagination\`. The pictures whose names start `manage-product-` include both lists, page 1 and the last page.

---

# Phase 18 — Verification

*Was Phase 14 until 10/21, when Ryan's review became Phases 14–17.*

### 18.1 Build and test (Claude, both platforms; reported per platform)
- [ ] knottyyoga: Linux gate (`server/docker/build_and_test.sh`) and Windows (`x64-Debug`). Also build `knottyyoga_database_helper` and `knottyyoga_test_helper` explicitly, since the gate builds neither `main.cpp`.
- [ ] Phases 4, 11 and 14 only: the `server_components` gates on both platforms. Phase 11 also: `honuware-web-components` `ng test` + build.
- [ ] Angular (`ui/`): `ng test`, `tsc --noEmit`, `ng build`, and the Playwright layout specs added in Phases 14–17.

### 18.2 Local run
- [ ] `knottyyoga_database_helper --recreate_database` against the local dev database. This is destructive, so it is announced before running. Confirm no `seed_lookup_missing`, `*_skipped` or `seed_image_failed` lines in the log, other than any photos still outstanding from Phase 1.

### 18.3 Hand-testing steps

Phases 14–17 each have their own walkthrough (14.9, 15.5, 16.8, 17.7), which these steps don't repeat.

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
   - **About** → **Our Location** shows the studio photo (once `Studio.jpg` is in; OQ-11), and text beginning "Our home gym: a full acrobatics floor with crash mats, aerial rigging, 24 fabrics, 9 ropes…" with the 520 directions. No climbing wall, no "Street parking is free after 6pm".

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

23. In `knottyyoga_test_helper` (REPL), run `set_password --email=bethismoving@gmail.com --password=Welcome-2026`, or `sp` with the same arguments. It reports that Beth must choose a new password at next sign-in. (It sends no mail.)
24. Sign out. **Sign In** as Beth: `bethismoving@gmail.com` with `Welcome-2026`. The site goes straight to **Change Password**, and clicking **My Classes** comes back to it. Fill in:
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

# Phase 19 — Production rollout

*Was Phase 15 until 10/21.*

Per D11 (OQ-9: wipe). Phase 4's fixes reach production earlier, in a release of their own (4.6); this phase is the content reinstall. Everything here runs on the EC2 host following the Deploying to AWS conventions:
- `--entrypoint` for helpers
- `-v /etc/knottyyoga:/etc/knottyyoga:ro`
- `--env-file /etc/knottyyoga/server.env`

### 19.1 Before
- [ ] Tag and deploy the release that contains the finished phases the normal way (CI tag → deploy script → `--migrate`). This proves the image, and confirms the new `img/` files are staged in it ([[gotcha_release_image_glibc_and_entrypoint]] #4), including Phase 14's `SafeSpace_icon.svg` and `home_hero_mobile.jpg`, and 13.11's `BeginnerKnottyYoga.jpg`, `AerialRope.jpg` and `Massage.jpg`.
- [ ] Manual RDS snapshot, named for this change. It is cheap insurance even though nothing is being kept.
- [ ] Have the `set_secret` block from Deploying to AWS 5.1 step 6 ready, with `website_address_login` = the cloudfront URL until go-live. Check that `mail_app_password` is in it: forgot password and the instructor digest need working mail.

### 19.2 Reinstall
- [ ] Stop `knottyyoga-helper` and `knottyyoga-server`.
- [ ] Run `knottyyoga_database_helper --install_schema --force` with `HONUWARE_ALLOW_DESTRUCTIVE=1` (one-shot, on that command only) and `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` from `server.env`.
- [ ] `--migrate` records the migration ledger; expect `applied=N skipped=0`, as on 9/23.
- [ ] Re-run the `set_secret` block.
- [ ] Start the server, then the helper. Expect `/api/health` → `db:"ok"`, and the helper log shows `login_success`.

### 19.3 After
- [ ] Mason, Caleb and Kit each sign in and change their password: first-name menu → **Profile** → **Change Password** (OQ-10). The seeded ones are in source control. This includes the `masonbendixen@hotmail.com` admin.
- [ ] For each instructor who should sign in, either:
  - `knottyyoga_test_helper --command=set_password` (5.2), with the temporary password handed over in person, or
  - once Phase 11 is live and their real email is seeded (5.8), they click **Forgot password?** themselves
- [ ] Paste any bios that arrived late under **Admin** → **Manage Products** → **Instructors** (OQ-15).
- [ ] The public half of the Phase 18.3 walkthrough (steps 1–9) against `https://dv1tgxa9ok30f.cloudfront.net/`.

### 19.4 Every month: extend Caleb's bookable hours (OQ-8)
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
- [[Mobile UI Updates]]: its 2.5 made the month calendar scroll sideways on phones on purpose; 15.2 reverses that at Ryan's request.
- [[Deploying to AWS]]: Phase 19's commands, and Phase 4.6's release. Go-live (the cert + alias records) remains separate.

---

# Open questions

*Mason's answers are kept as written. "→ Plan" says what changed. Anything still open has a default the plan is written against.*

### Still open

**OQ-11 — Where are the photos?**
- Mason- Taking the photos IS a phase. Please list this as an early phase that the later phases will use. I'll try to take the photos tonight.
- → Plan: Phase 1 is the shoot, with a shot list, framing notes and checkboxes. **Default hand-off:** drop the originals into `C:\Users\mason\Documents\Obsidian\Knotty Yoga\Photos\` named as in 1.1 (any format or size); Claude converts and checks them (1.2). Say if you'd rather use another folder.
- → 10/18: the photos aren't in yet, so attaching them moved from Phase 9 to **Phase 13**, just before Verification. Phases 9–12 don't use them and go first.
- → 10/20: six are in (all five headshots and Intermediate Partner Acro, from `D:\Knotty Yoga\2026\Oct\JPG\`) and attached in Phase 13. **Still to come:** Beginner Knotty Yoga, Aerial Rope, Massage and Studio (1.1 items 6, 8, 12, 13). Drop each in the same folder under its 1.1 name; it joins as file + row + test (13.1). Until then Phase 18.3's steps 1 and 9 will find those four without a picture.
- Mason- I put more of the missing / replacement photos at: D:\Knotty Yoga\2026\Oct\Oct 22 Website Shoot\JPG
- → 10/23: five photos there are new to the plan, and all five are usable (1.2):
  - new: **Beginner Knotty Yoga**, **Aerial Rope** and **Massage**
  - replacements: **Handstands** and **Beginner Partner Acro**
  - `IntermediatePartnerAcro.jpg` is byte for byte the one already in, so it changes nothing.
  - New item **13.11** converts and attaches them, not built yet. After it every class has its own picture.
  - **Still to come: the studio picture** (1.1 item 13, `Studio.jpg`), for **About** ▸ **Our Location**. It needs the facilities loop as well as the file (13.1). Until it arrives, 18.3's step 9 finds Our Location without a picture.
  - **Knotty Yoga** keeps today's picture.
  - The massage photo's framing is OQ-40.
- → 10/23, later: **13.11 is built.** All six classes and the Massage card have their pictures. Only `Studio.jpg` is still to come.
- → 10/27: Ryan's edits of those seven, and eight gallery photos, are in and attached (**13.13**). `Studio.jpg` is still the only one to come.

**OQ-40 — The massage photo is upright; the Services card shows a wide strip.** *(Found checking the new photos, 10/23.)*
- The **Services** card shows its picture across the top of the card, 160 px tall. That strip is about 2.3 times as wide as it is tall on a desktop (about 375 × 160), and 2.2 times on a phone.
- The massage photo is upright (3:4), so the strip keeps only a third of it:
  - **From the middle**, as the card crops today: Caleb's forearm and the client's back, without his face.
  - **From higher up:** his face and raised fist, without the massage.
- 1.1 asked for landscape only for the class photos, so this one wasn't covered.
- **Default:** two changes, both in 13.11.
  - The photo is cropped to landscape (4:3) when it is converted, from just above Caleb's head down to his elbow on the client's back.
  - The **Services** card shows its picture at 4:3, about 375 × 280 on a desktop, instead of the 160 px strip. A picture uploaded later for any service gets the same 4:3 frame.
- **Alternative:** keep the 160 px strip and crop the photo to Caleb's face and fist. Or a landscape reshoot.
- Say which.
- → 10/23: **13.11 is built with the default**, so the walkthrough there shows it. One difference from the plan: the photo can't run from above his head to his elbow at 4:3. The crop keeps his face, his fist and the elbow on the client's back, and cuts the top of his hair (13.11, Files). The alternative is still a small change: the card's CSS and a re-crop.

**OQ-41 — On a phone, a product page's subscription cards crowd their header.** *(17.8's screenshots, 10/23.)*
- Each card under **Active Subscriptions** puts three things on one row: the subscriber's name, the status badge and the billing period. At 390 px wide the name and the period each wrap onto two lines, with the badge between them:
  - "Member / 04"
  - "February 2025 - / March 2025"
- It was like this before 17.8. It shows now because the screenshots cover this list for the first time.
- Nothing overflows or overlaps, and a desktop shows each header on one line.
- **Default:** leave it. Manage is mostly used on a desktop.
- **Alternative:** on a phone, the period moves to its own line under the name and badge.
- Say which.

**OQ-42 — Does "no gallery name" include the Gallery page's own title?** *(13.13, 10/27.)*
- **About** ▸ **Gallery** opens with the page title **Gallery**, as every page opens with its own title. Under it, 13.13's two carousels have no heading, no blurb and no captions.
- **Default:** keep the page title. It names the page, not a carousel.
- **Alternative:** drop it, so the page starts with the photos. A one-line change to the page and its spec.
- Say which.

### Resolved

*Newest first by date: 10/23 (OQ-1, 38, 39), then 10/22 (OQ-26 to 37, in number order), then 10/20 (OQ-15), then 10/19 (OQ-25, 24, 23), then 10/17 (OQ-22), then 10/15 (OQ-16, 19, 20, 21), then the 10/14 answers.*

**OQ-1 — Emails for Beth Cooper, Gifford Cheung and Tiana Cheung.** `people.email` is required and unique, and the instructor digest and (after Phase 11) password resets go to it. **Default:** placeholders `beth.cooper@knottyyoga.local` (likewise for the other two), undeliverable like the scheduler account, so the code can land. Real ones are swapped in before Phase 19.
- Mason- I sent a message to all my instructors asking which email to use.
- → Plan: waiting. Paste the three addresses here when they arrive; they are one constant each in `seed_app_data.h`.
- → 10/16: Phase 5 shipped with the placeholders, so nothing waits on this. Swap them in before Phase 19. Until then the instructor digest and, after Phase 11, password-reset emails to these three go nowhere.
- Mason- Here are the remaining emails:
	- Beth Cooper - bethismoving@gmail.com
	- Gifford Cheung - giffordcheung@gmail.com
	- Tiana Cheung - tianabun@hotmail.com
- → Plan: new item **5.8**, not built yet. The three constants become these addresses.
  - It also makes the three findable under **Search for a person**. That search hides every `@knottyyoga.local` address, the scheduler account's domain, so it never found the placeholders.
  - 18.3's steps 23–24 now name Beth's address.

**OQ-38 — Table columns that shift from page to page.** *(17.4's screenshots, 10/23.)*
- A table sizes its columns to the rows it shows. Now that a page shows ten rows, a long value on one page widens its column there and not on the next. On a desktop, the columns move between pages by:
  - **Products:** 25–55 px
  - **Membership Tiers:** about 80 px
  - a session card's **Staff** table: up to 175 px
- **Blog Posts** has done the same since Phase 16. Nothing overflows or is cut off, and phones are hardly affected.
- **Default:** leave it.
- **Alternative:** fixed column widths on those four tables. The columns then stay put, but a long name wraps in its fixed column instead of widening it.
- Say which.
- Mason- We can go with the default
- → Plan: no change. Each table's columns keep sizing to the rows on its page.

**OQ-39 — A product page's latest-10 lists have no View All.** *(17.1, 10/23.)*
- The plan said a product page's **Event Sessions**, **Active Subscriptions** and **Recent Purchases** show the latest 10 with a **View All** link, so 17.1 left them alone. Only an event product's **Event Sessions** has that link.
- For other products, the 11th subscription or purchase can't be reached from the product page.
  - **Manage** ▸ **Subscriptions** with its **Product** filter lists every subscription of a product.
  - No page lists one product's purchases.
- **Default:** leave them as they are.
- **Alternative:** page **Active Subscriptions** and **Recent Purchases** five at a time, like the page's other lists, so every row can be reached there.
- Say which.
- Mason- Let's go with the alternative.
- → Plan: new item **17.8**, not built yet.
  - Both lists ask for every row and page five at a time, like the page's other nested lists.
  - **Active Subscriptions** looks up names and entitlements only for the five rows shown. Today it looks them up for every row it loads.

**OQ-26 — A badge on the All Classes cards?** *(12.3's front-end check, 10/19.)*
- After 12.3 everyone sees all six classes on **Our Classes** → **All Classes**. Each card shows the name, description, tags and session count, but not whether you can attend.
  - The weekly schedule marks each row "Requires a membership" or "Prerequisite needed".
  - Each class page now names what it needs (12.1).
- **Default:** no badge on the cards. Every class needs a membership, so for a visitor or a non-member all six cards would carry the same badge. The class page, one click away, names the membership and the skill.
- **Alternative:** the weekly schedule's badge on each card, worked out for the viewer. For example, a Silver member would see Beginner Knotty Yoga unmarked and the other five marked. The class list would then have to carry each viewer's access: a change to the server and to the page.
- Say which.
- Mason- The default is fine.
- → Plan: no change. The cards stay without a badge.

**OQ-27 — An unapproved Gold member reads "Members only: Gold membership".** *(Found implementing 12.1, 10/19.)*
- The class page's **Pricing** says "Members only" whenever the viewer can't attend, whatever the reason.
- So on Handstands, Aerial Rope and Intermediate Partner Acro, a Gold member whom no instructor has approved reads "Members only: Gold membership", though they already have Gold.
  - Right below it, **Required Skills** shows the skill with a red mark, then "You're missing: Handstand" and "Talk to staff for evaluation."
  - Before 12.1 the same member read "Members only" and "Upgrade your membership to attend.", so this isn't new.
- **Default:** leave it. The Required Skills box says what is actually missing.
- **Alternative:** when the viewer has the membership and lacks only the approval, **Pricing** says so instead, for example "Included with your membership once an instructor approves you". The server already tells the two apart (the schedule badge's "members only" versus "needs a skill"). It would have to pass that on, and the page would need a fourth wording.
- Say which.
- Mason- I like the alternative idea better.
- → Plan: new item **12.7**, not built yet.
  - The class page tells a viewer whose only gap is an approval: **Included with your membership once an instructor approves you**.
  - The server works it out from the groups the viewer fails. If every one is an approval (no permission in it), it says so with a new `price_awaits_approval` flag. The page then shows the fourth wording.
  - Recurring classes only: a workshop or series still has a price to settle.

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
- Mason- Keep the corrections.
- → Plan: no change. The seed keeps all six.

**OQ-29 — Which home page picture should centre?** *(Ryan's review, 10/21.)*
- Ryan asked for "an option to have the image be centered" for banner images. On a wide screen, three home page pictures could be meant:
  - The **"Come join the fun!" banner** is already centred, and fills its band.
  - The **top picture** (hero) fills the full width. Its people stand on the left because the photo is composed that way, with the words on the dark right.
  - The **photo carousel**: on 10/21, at 1400 and at 1920 px wide, its four photos sat at the left, ending at 1120 px, with the rest of the row empty.
- **Default:** the carousel, the one picture visibly not centred. 14.2–14.5 add **Centre the photos when they don't fill the width** to a carousel row, and the seed turns it on.
- **Alternative:** if Ryan meant the top picture, "centred" needs a description: the photo in the middle with the words over it, perhaps. Say what it should look like.
- Mason- We have a banner image with the text Get into the best shape of your life. Things we want changed:
	- I don't want THE RIGHT WAY. on a separate line / section. Just move that text appended to the GET INTO THE BEST SHAPE OF YOUR LIFE,
	- We want the current one to be only shown on desktop and then copy it but use this image centered on mobile: C:\Users\mason\Downloads\bd1ac730-677f-4471-8cfd-0b1080081126.jpg
	- Don't center the image on the desktop version
- → Plan: it is the top picture (the hero band), not the photo strip. The strip stays as it is.
  - **One line.** The headline is "Get into the best shape of your life, the right way.", and the separate red line is gone (14.3).
    - That puts the whole headline in white. The red belonged to the separate line.
  - **Desktop:** today's row becomes **Desktop only**, with its picture as now and not centred (14.3).
  - **Mobile:** a second top-picture row, **Mobile only**, uses your new photo, copied in as `home_hero_mobile.jpg`, with **Centre the picture** on (14.2–14.5).
  - **What "centred" does.** The new photo already has the people in the middle. So centring means the headline goes under the picture, centred, at any width, rather than over a side the photo doesn't leave empty. On a phone, the headline was already under the picture and now also sits in the middle.
  - Hand-test steps 1, 2, 8 and 9 in 14.9 cover it. The photo-strip steps are gone.

**OQ-30 — The studio's real Facebook and Instagram pages.**
- The seeded links are the two sites' front pages (`https://facebook.com`, `https://instagram.com`), so today's icons lead nowhere in particular.
- **Default:** keep them. After the reinstall, set the real addresses under **Site Theme** → **Brand basics** → **Social links** (19.3).
- **Alternative:** paste the two addresses here, and the seed carries them.
- Mason- https://www.facebook.com/KnottyYoga/ and https://www.instagram.com/knottyyoga
- → Plan: the seed carries both (14.3), and a test pins the two lines. 19.3's step to set them by hand is gone. 14.9 step 7 clicks each icon.

**OQ-31 — Which way should the two fonts match?** *(Ryan's review.)*
- The intro strip's sentence is in the display font, Barlow semibold, at 24 px. The Get Started band's "New here? …" line is in the body font, Roboto regular, at about 16 px.
- **Default:** the "New here?" line takes the intro strip's style (14.5).
- **Alternatives:** the intro sentence takes the plain style instead; or both use the display font, each keeping its own size.
- Say which.
- Mason- Roboto
- → Plan (14.5):
  - The intro sentence moves to the body font, which is Roboto in the seed, at regular weight, matching the "New here?" line.
  - It is set through the theme's body-font setting, so a studio that changes its body font changes both lines.
  - Each line keeps its own size, and the intro sentence stays the larger.

**OQ-32 — Paging: page sizes and the lists left alone.** *(Phases 16–17.)*
- **Default sizes:** a page's main list shows 10, and a list inside a card or one of several sections on a page shows 5. The pager hides when a list fits on one page.
- **Default exemptions**, with the reason for each, are in the table at the top of Phase 16. In short:
  - calendars and lists already paged by date
  - lists the studio orders with ↑ ↓
  - home page teasers that already link to the full list
  - cart and checkout
  - search results and drop-downs that are already capped
  - reports, and the check-in screens
  - lists of edit forms (**Fonts**, **Locations**)
  - **Manage Data**
  - a price schedule's embedded table, a library control that already pages 20 at a time
- **Alternative:** name any list above that should page anyway, or any list that should get 5 instead of 10.
- Mason- I'm fine with the default
- → Plan: no change. Phases 16–17 stand as written.

**OQ-33 — The week view still scrolls sideways on a phone.** *(15.2.)*
- Ryan asked about the month view, and 15.2 fixes only that one. **Week View**'s seven columns of hours also need about 800 px, and still scroll sideways on a phone.
- **Default:** leave it. Phones open on **Day View**, and a week of hourly columns can't be read at 50 px a day.
- **Alternative:** the same treatment as the month: seven narrow columns with short names.
- Mason- I'm okay with the default.
- → Plan: no change. Only the month view is reworked (15.2).

**OQ-34 — Uploading the badge as an SVG needs a server_components release.** *(14.1.)*
- An admin can't upload an SVG today: the framework's upload refuses it, although the picker offers it. So 14.1 changes `server_components`, with its own commit and push, then a pin bump and gates in both apps, as 11.11 did.
- **Default:** do it. The badge is an SVG, Ryan asked for the SVG to be editable, and the same fix lets the Get Started artwork be replaced with another SVG.
- **Alternative:** accept PNG and JPEG only for the badge. There is then no `server_components` change, and a new badge loses the sharpness a vector keeps at every size.
- Mason- Let's make the change.
- → Plan: no change. 14.1 stands, with its own `server_components` push and pin bump.

**OQ-35 — Where "desktop" starts: 768 px?** *(D16.)*
- **Default:** 768 px. That is the site's own phone line, and where the calendar already switches.
- **Alternatives:**
  - **1024 px.** Only big tablets get the desktop sections. Most tablets held upright (768–834 px) would get the phone ones.
  - **A setting in Site Theme,** so the studio can move the line. That is one more control to explain.
- Say which.
- Mason- Let's go with the default.
- → Plan: no change. D16 and 14.5 stay at 768 px.

**OQ-36 — Two pagers close together on My Bookings.** *(16.5's screenshots, 10/22.)*
- A series on **My Bookings** pages its own sessions, five at a time. When the series is the last row of the page, its pager ("1–5 of 23") sits about 40 px above the list's own pager ("21–23 of 23"). Both look the same, so it is easy to press the wrong one.
- This only happens with a series of more than five sessions on the last page of more than ten bookings.
- **Default:** leave it. Each pager is inside the box it moves.
- **Alternative:** a pager inside a card names what it counts, "Sessions 1–5 of 23". Every other pager stays as it is.
- Say which.
- Mason- Default is fine.
- → Plan: no change. Every pager keeps the same label, "1–5 of 23".

**OQ-37 — Ten a page in a three-column grid.** *(16.5's screenshots, 10/22.)*
- On a desktop, the card grids of **Memberships** (tiers, other products), **Upcoming Events**, **My Skills** and **Services** have three columns, so a full page of ten leaves one card alone on a fourth row.
- **Default:** keep ten everywhere (OQ-32). One rule is easier to explain than a size per layout.
- **Alternative:** nine or twelve a page for those grids only. Every page then ends on a full row on a desktop, but a phone (one column) sees a different count from the other lists.
- Say which.
- Mason- Default is fine.
- → Plan: no change. The grids keep ten a page, like every main list.

**OQ-15 — Instructor bios.**
- Mason- I asked my instructors to write bios. Leave a place for me to put them as they send them to me. Worst case, I manually add them once deployed but hopefully they get them to me before then.
- → Plan: the **Instructor bios** section above has a slot under each name. Whatever is there at Phase 5.5 is seeded verbatim, anything still empty gets its one-line default, and late ones go in through Manage after deploy (19.3).
- → 10/20: all five arrived and are in the seed (Phase 13), with OQ-28's corrections. 19.3's "paste late bios" step has nothing left to do unless one changes.

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
- **Default:** no code change. 19.4 now says to press **Generate Availability** right after any edit. You will rarely edit the hours, and the fix costs one press.
- **Alternative:** the Templates page generates for you after **Save Template**, from today through the end of next month. That is a front-end change, with tests.
- Say which.
- Mason- I'll go with the default.
- → Plan: no change. 19.4 keeps its "press **Generate Availability** straight away" step.

**OQ-22 — Signed in, Our Classes hides every class you can't attend yet.** *(Found implementing Phase 6, 10/17.)*
- Signed out, **Our Classes** → **All Classes** lists all six classes. Signed in, it lists only the classes the person may already attend (`ClassCatalogHelper::GetClassesVisibleToPerson`, a deliberate rule from the access redesign, P-1/M-6). Until Phase 6 every class was open, so nobody noticed.
- Now that all six need a membership, a signed-in **non-member sees an empty page**, a Silver member sees only Beginner Knotty Yoga, and **so do the staff**: Mason's and the instructors' accounts hold no membership. The weekly schedule is unaffected: it lists all 14 rows with their badges, signed in or out.
- It also breaks 18.3 step 13, where a Silver Kit opens Knotty Yoga from **All Classes**.
- **Default:** signed-in visitors see every scheduled class, as signed-out ones do, and the class page and badges say what each one needs. New item 12.3. Say if a signed-in member should keep seeing only what they can book.
- Mason- Let's go with the default.
- → Plan: 12.3 stands as written. Everyone sees all six classes on **All Classes**, signed in or out, and the class page says what each one needs. Until 12.3 lands, a signed-in non-member or staff account still sees an empty **All Classes** page; 6.6's last step notes it.

**OQ-16 — Leftover copy that no longer matches the studio.**
- Mason - Let's do: Who we are / How we teach / Come and find us *(full text moved into the About page table above, verbatim)*.
- → Plan: Phase 8, which now covers all three pieces of copy. The two follow-ups below were answered 10/15:
  - The mail sender name **"Knotty Yoga and Spa"** (`app_secret_values.cpp`). With the spa removed, would you like plain "Knotty Yoga"?
	  - Mason- Let's just do Knotty Yoga
	  - → Plan: 8.1. `kMailSenderNameValue` becomes "Knotty Yoga", and the eight tests that check the From name switch to the constant. Production picks it up with the Phase 19 reinstall.
  - The location's description on **About ▸ Our Location**, which still mentions "a climbing wall" and "Street parking is free after 6pm". Send replacement text, or say "drop the climbing wall".
	  - Mason- Let's say: "Our home gym: a full acrobatics floor with crash mats, aerial rigging, 24 fabrics, 9 ropes, plus a private room for massage and bodywork. Parking is abundant. Conveniently off the 520. Heading Eastbound on 520 take the first 148th exit and then the right sub exit labelled 152. Go "through" the first roundabout and then go "left" at the second. We in the last building (14) before the parking lot bends to the right. Space 2545 / Knotty Yoga."
	  - → Plan: the **About ▸ Our Location** section under *Target content*, seeded in 8.1. One word was changed, "We in" → "We're in", which is flagged there; say if it should stay exactly as typed.

**OQ-19 (new) — Knotty Yoga approval as a skill?** The Overview says Knotty Yoga "requires instructor approval to attend. Should be able to do a pull-up and five regular pushups". The OQ-4 answer lists skills only for handstand, aerial_rope and intermediate_partner_acro.
- **Default (written into the plan):** Knotty Yoga needs Gold only, and approval is checked at the door, as the class description says. Gold members can book it without an instructor first approving them in the system.
- **Alternative:** a fourth skill `knotty_yoga` ("Able to do a pull-up and five regular pushups."), gated like Handstands. Every Gold member would then need an instructor's approval before booking the main class.
- Mason- Let's go with the default.
- → Plan: no change; the plan was already written this way (D6, and the Class access table). Knotty Yoga is Gold-only, with approval checked at the door.

**OQ-20 (new, from Phase 3) — Five tiers on a three-column grid.** On a desktop-width screen the membership cards sit three to a row. So Silver, Silver Couple's and Gold share the first row, and Gold Couple's and Gold Family sit alone on the second. Gold is split from its own tiers, and the second row is two-thirds full. This shows on the Memberships page, the home page band and the shop.
- **Default:** leave it. The order is right; only the line break falls mid-Gold. You'll see it in the Phase 18 walkthrough.
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
- → Plan: the rolling-availability phase is deleted. Caleb's template is open-ended, the seed generates through the end of next month, and 19.4 is the monthly **Generate Availability** press with exact steps.

**OQ-9 — Getting this onto the live site.** Mason- No, I'm just in preview mode and still using the Square sandbox. Wipe away. → D11, Phase 19. Theme export/import is dropped; the `set_secret` block stays, because it carries the Square sandbox and mail settings.

**OQ-10 — Test accounts in the base seed.** Mason- I'll manually have the three of us change our passwords as we deploy. I'm not that concerned about this. → Kit and `masonbendixen@hotmail.com` stay in the base seed, unchanged (5.3). 19.3 is the password change for Mason (both accounts), Caleb and Kit.

**OQ-12 — Test-only data.** Mason- let's remove both of these. I'm getting ready to deploy and want the website to be as it will actually function. → Phase 9 and D14. With both gone and Caleb's template promoted to real data, nothing test-only remains, so `--recreate_database_test` is removed too.

**OQ-13 — Membership tier order.** Mason- I'd prefer the alternative. → D12, Phase 3: grouped by tier through a new `products.sort_order` column, editable as **Sort Order** in Manage Products.

**OQ-14 — Membership descriptions.** Mason- This sounds fine. → The Memberships table.

**OQ-17 — Tell visitors which membership a class needs.** Mason- let's make this change. → 12.1.

**OQ-18 — Who builds.**
- Mason- I trust you more now! Please update the note that you build and run the tests on windows and linux but report the results to me. When stuff would break, I'd just paste the output to you so I figure I trust you now to remove myself from this loop as your model has improved.
- → Plan: recorded in **Revision notes and working agreement** at the top of this doc and in Claude's standing memory. The Overview's "I will build…" line is left as you wrote it, since the Overview is yours, and is superseded by this answer.
