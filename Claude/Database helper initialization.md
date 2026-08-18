---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/18/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

database_helper currently populates the database with a good amount of information when passed the flag --recreate_database. I'd like to add some things to this and then add another flag --recreate_database_test that does everything the --recreate_database does but then populates with some additional test data.

For these people, I'd like the following images to copied into the tree and added as their image associated with their account profile:

- Mason Bendixen (masonbendixen@gmail.com)
	- Image: D:\Pictures\Pics\Kauai.jpg
	- Copy into tree as Mason.jpg
- Caleb Ault
	- Image: D:\Pictures\Pics\Croc.jpg
	- Copy into tree as Caleb.jpg

In addition, I'd like to have Mason and Caleb both be configures as Instructors (with the same photo as their instructor photo for now) and also as service providers with the provider type as massage therapist.

Can you change the Knotty Yoga Monday 6pm slot so that the instructor is Mason Bendixen and the Wednesday one so that it is Caleb Ault. Can you add a Handstand class on Mondays at 7pm with Mason Bendixen as the instructor and require the person to attend the Monday 6pm Knotty Yoga class.

Can you add a class slot for Thursdays 6pm-7pm that is partner acrobatics taught by Mason Bendixen as well as Sunday 10am-11am.

Can you make these photos for these classes:
- Knotty Yoga
	- Image: D:\Acrobatics Photos\2025\Sep\Edited\20220407_033309094_iOS.jpg
	- Copy into tree as KnottyYoga.jpg
- Partner Acrobatics
	- Image: D:\Acrobatics Photos\2025\Oct\Oct Lorien Shoot\To Upload\20251028_054530759_iOS.png
	- Copy into tree as PartnerAcro.jpg
- Handstands
	- Image: D:\Acrobatics Photos\2025\Sep\Edited\20241201_201508414_iOS.jpg
	- Copy into tree as Handstand.jpg

Please update the photo for each class accordingly.

Up to this point, all of this stuff has been just the regular database creation (--recreate_database). The following should just be for --recreate_database_test.

Please create an Event Session on the two following Saturdays after the tool is run with from 10am-11am with the Product Intro workshop. Allow booking and show on home page 14 days in advance.

Please create a Product called Aerial Series with the code aerial-series and the Description Introduction to aerial acrobatics on rope and fabric with the Kind being Class Series. Make the price for everyone $30 per session and $10 per session for Knotty Yoga Gold members. Then under Class Schedules, make a class series class schedule that is Tue / Thu 6-7pm from the time the tool is run through the end of the FOLLOWING month. Please make Caleb Ault the instructor for those sessions.

For Caleb as a service provider, please create a Schedule Template with him for Mon-Fri 9am-5pm. Then populate this template starting from today through the end of the FOLLOWING month.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Current state (verified against the code, 8/18/2026)

What already exists, so the plan says what CHANGES rather than restating the file.

**`create_database.cpp` structure.** ~35 `Populate*` functions called in dependency order from `PopulateTables()`, each taking `(Transaction&, DatabaseHelper, DatabaseInfo)`. Rows are added through `MakeAddRowLambda(...)` or `DbCrud::AddRowToTableFetchInt64PrimaryKey` when the id is needed downstream.

**Seed images already work.** `SeedImageDirectory()` returns `<exe dir>/img`, overridable with `HONUWARE_SEED_IMAGE_DIR`; `ReadBinaryFile()` reads in **binary** mode (text mode corrupts JPEGs); `AttachSeedPhoto(txn, db, table, rowId, fileName, imageType)` uploads and associates, logging every outcome. A CMake step copies the image directory next to `knottyyoga_database_helper`. **So adding a photo to a row is a one-line call — the mechanism is done.**

**People.** `PopulatePeople` seeds four users. `masonbendixen@gmail.com` ("Mason Bendixen", pw `pass`) and `mr.calebault@gmail.com` ("Caleb Ault", pw `caleb`) already exist. There is also a `masonbendixen@hotmail.com` / "Mason2".

**Relevant tables** (all present):

| Concern | Tables |
|---|---|
| Instructor | `instructors` (`person_id`, `bio`) |
| Service provider | `provider_types` (**`massage_therapist` already seeded**), `provider_type_assignments` (`person_id`, `provider_type_id`, `is_accepting_bookings`, buffers…) |
| Class scheduling | `class_instances` → `class_schedules` → `class_schedule_slots` (`day_of_week`, `start_time_minutes`, `duration_minutes`, `facility_id`, `location_room_id`, **`instructor_person_id`**) |
| Prerequisites | `class_requirement_groups` (`class_id`, `label`, `inline_description`) + `class_requirement_group_literals` (`group_id`, **`permission_id`**, **`skill_level_id`**) |
| Series | `class_series_instances` (`class_instance_id`, `min_attendees`, `min_by_us`, `min_not_met_policy`, `prorated_signups_allowed`) |
| Events | `event_sessions` (`product_id`, `start_time_us`, `end_time_us`, `capacity`, `status`, **`show_on_home_page`**, **`home_page_visible_days_before`**, `show_on_upcoming`, `upcoming_visible_days_before`) |
| Pricing | `product_prices` (`product_id`, `price_schedule_id`, **`permission_id`**, `amount_cents`, `price_kind`) |
| Provider schedule | `schedule_templates` (`provider_person_id`, `name`, `effective_from_us`, `effective_to_us`, `is_active`) + `schedule_template_entries` (`day_of_week`, `start_time_minutes`, `end_time_minutes`) → `provider_availability` |

**Two facts that shape the plan:**

1. **`ScheduleTemplateHelper::GenerateAvailability(transaction, request)`** already exists in `business_logic/scheduling/` and is tested. Populating a template into `provider_availability` is a CALL, not new logic.
2. **`kProductKindClassSeries = "class_series"`** already exists as a product kind, alongside `event`, `class`, `workshop`, `bookable_service`, `subscription`, `one_time`.

**Existing seeded class schedule.** One perpetual `class_instances` row for class 1 (Knotty Yoga) → one "Default schedule" → **two slots: Mon(1) and Wed(3) at 18:00 for 60 min**, `instructor_person_id` deliberately omitted. This is exactly what §2 below edits.

---

# Design decisions

- **D1 — `--recreate_database_test` composes, it does not fork.** The new flag runs the entire existing `--recreate_database` path and then calls one additional `PopulateTestData()` entry point. No branching inside the existing `Populate*` functions: a fork would let the two paths drift, and the test data is defined as "everything real, plus extra".
- **D2 — Relative dates are computed from ONE captured `now`.** Every "next two Saturdays" / "through the end of the following month" is derived from a single `nowUs` read at the start of the run, threaded through. Reading the clock per-function makes a run that straddles midnight produce inconsistent data.
- **D3 — Date arithmetic goes in a testable helper, not inline in the seeder.** `create_database.cpp` is **not** compiled by the standard test gate (it is in `knottyyoga_database_helper`, not `knotty_yoga_tests`), so any logic written inline there is untestable and unverified until a manual run. All "next Saturday", "end of following month", "every Tue/Thu between X and Y" logic goes into a **new tested helper in `business_logic/scheduling/`**, and the seeder only calls it. *This is the single most important structural decision in this plan.*
- **D4 — Seed images are copied into the tree, not referenced from `D:\`.** The absolute paths in the Overview are Mason's machine. The plan copies each file into the repo's seed-image directory under the given name, and the seeder refers to it by filename only — same as the existing home-page images.
- **D5 — Photos attach by looking the row up by natural key, never by hardcoded id.** Existing code uses hardcoded ids (`kClassId = 1`, `kProductId = 10`) with a comment explaining the seed order. That is fragile and already load-bearing; new code looks up `classes` by name and `people` by email so inserting a row above does not silently repoint a photo.

---

# Phase 1 — Foundations: dates, images, and the new flag

Everything later depends on these three. Lower layers first within the phase.

### 1.1 Date helper *(business_logic/scheduling — testable)*
- [ ] Add `seed_date_util.h/cpp` in `business_logic/scheduling/`: `NextWeekdayOnOrAfter(nowUs, dayOfWeek, tz)`, `EndOfFollowingMonthUs(nowUs, tz)`, `WeekdayOccurrencesBetween(fromUs, toUs, dayOfWeek, startMinutes, durationMinutes, tz)`.
- [ ] Tests: a Saturday run returns the NEXT Saturday not today; month-end rollover across December→January; DST boundaries; an empty range returns no occurrences.

### 1.2 Seed images into the tree
- [ ] Copy the five source images into the repo seed-image directory as `Mason.jpg`, `Caleb.jpg`, `KnottyYoga.jpg`, `PartnerAcro.jpg`, `Handstand.jpg`. *(See **OQ-1** — one source is a `.png` to be stored as `.jpg`.)*
- [ ] Confirm the CMake copy step picks them up (it copies the whole directory today).

### 1.3 The `--recreate_database_test` flag *(database_helper)*
- [ ] Add `ABSL_FLAG(bool, recreate_database_test, ...)` in `database_helper/main.cpp`, add it to the mode count and the usage text, and route it to `RunRecreate(/*withTestData=*/true)`.
- [ ] Thread a `bool includeTestData` through `CreateSchemaAndPopulate` → `PopulateTables`, ending in a single `if (includeTestData) PopulateTestData(...)` call at the end.
- [ ] ⚠️ Build `knottyyoga_database_helper` explicitly — the standard gate does **not** build this target.

---

# Phase 2 — Base seed data (`--recreate_database`)

### 2.1 People photos
- [ ] Attach `Mason.jpg` to `masonbendixen@gmail.com` and `Caleb.jpg` to `mr.calebault@gmail.com` via `AttachSeedPhoto` on the `people` table, resolving `person_id` by email.
- [ ] Verify `people` is in `photo_support_tables` (`PopulatePhotoSupportTables`); add it if not.

### 2.2 Instructors
- [ ] `PopulateInstructors()`: add `instructors` rows for Mason and Caleb.
- [ ] Attach the same photo to each instructor row *(see **OQ-2** — whether `instructors` is photo-supported or the profile photo is reused)*.

### 2.3 Service providers
- [ ] `PopulateProviderTypeAssignments()`: assign both to the existing `massage_therapist` provider type with `is_accepting_bookings = true`.

### 2.4 Class photos
- [ ] Attach `KnottyYoga.jpg`, `PartnerAcro.jpg`, `Handstand.jpg` to the `classes` rows found by name.
- [ ] Verify `classes` is photo-supported.

### 2.5 Instructors on the existing slots
- [ ] Set `instructor_person_id` on the seeded Mon 18:00 slot to Mason, and the Wed 18:00 slot to Caleb.

### 2.6 New class slots
- [ ] **Handstands, Mon 19:00–20:00**, instructor Mason. Needs its own `class_instances` + `class_schedules` + slot for the Handstands class *(see **OQ-3** — which product it binds to)*.
- [ ] **Partner Acrobatics, Thu 18:00–19:00 and Sun 10:00–11:00**, instructor Mason. Same structure.

### 2.7 Handstands prerequisite
- [ ] Add a `class_requirement_groups` row on Handstands with an inline description.
- [ ] ⚠️ **Blocked on OQ-4.** `class_requirement_group_literals` expresses a requirement as a **permission** or a **skill level** — there is no "attended class X" literal. The Overview asks for "require the person to attend the Monday 6pm Knotty Yoga class", which the current model cannot express directly.

### 2.8 Tests
- [ ] Extend the seeder's own test coverage where it exists; add tests for any new **table-helper** methods introduced.
- [ ] ⚠️ `create_database.cpp` itself has no test target — see **OQ-5**.

---

# Phase 3 — Test-only seed data (`--recreate_database_test`)

All of this lives in `PopulateTestData()` and runs only under the new flag.

### 3.1 Intro Workshop event sessions
- [ ] Create two `event_sessions` for the existing `intro-workshop` product on the **next two Saturdays**, 10:00–11:00, using the Phase 1.1 helper.
- [ ] Set `show_on_home_page = true` and `home_page_visible_days_before = 14`. *(See **OQ-6** re: `show_on_upcoming`.)*

### 3.2 Aerial Series product
- [ ] Add product `aerial-series` — "Aerial Series", description "Introduction to aerial acrobatics on rope and fabric", kind `class_series`.
- [ ] `product_prices`: $30/session general (`permission_id` NULL) and $10/session for the Gold permission, both on the seeded 2026 price schedule. *(See **OQ-7** for the exact Gold permission and `price_kind`.)*

### 3.3 Aerial Series class schedule
- [ ] Create the `class_instances` → `class_series_instances` → `class_schedules` chain bound to the `aerial-series` product, valid from now through **end of the following month**.
- [ ] Slots **Tue(2) and Thu(4) 18:00–19:00**, `instructor_person_id` = Caleb.
- [ ] ⚠️ Thursday 18:00 now has **both** Partner Acrobatics (2.6) and Aerial Series — see **OQ-8**.

### 3.4 Caleb's provider schedule
- [ ] Create a `schedule_templates` row for Caleb, Mon–Fri, with five `schedule_template_entries` at 09:00–17:00 (540–1020 minutes).
- [ ] Call `ScheduleTemplateHelper::GenerateAvailability` for **today through end of the following month** to populate `provider_availability`.

### 3.5 Tests
- [ ] Tests for every new business-logic helper (the date helper is the main one).
- [ ] A test asserting `PopulateTestData` is **not** reachable from the plain `--recreate_database` path.

---

# Phase 4 — Verification

### 4.1 Build and run
- [ ] Build `knotty_yoga_tests` **and** `knottyyoga_database_helper` (the latter is not in the standard gate).
- [ ] Run `--recreate_database` on a scratch database; confirm photos, instructors, providers and the six slots.
- [ ] Run `--recreate_database_test`; confirm the events, series, prices and availability.

### 4.2 Hand-testing steps
- [ ] Write live-server steps per the precise-instructions rule: exact menu → submenu → dashboard item → field labels, for each thing seeded.

---

# Open questions

**OQ-1 — One image is a PNG stored under a `.jpg` name.** The Partner Acrobatics source is `...20251028_054530759_iOS.png`, to be copied in as `PartnerAcro.jpg`. `AttachSeedPhoto` takes the image type as an explicit argument and the decoder goes by content, so a PNG named `.jpg` will work but is misleading. Do you want it (a) converted to real JPEG, (b) kept as PNG and named `PartnerAcro.png`, or (c) left exactly as written?
- Mason- Let's convert to JPG

**OQ-2 — Instructor photo storage.** You asked for "the same photo as their instructor photo for now". Is the `instructors` row meant to carry its own photo (I would add `instructors` to `photo_support_tables` and attach a second copy of the bytes), or should the UI just fall back to the person's profile photo? The second stores the image once; the first lets an instructor headshot diverge from a profile picture later.
- Mason- In instructors, when I do add new from the admin portal for Instructors, I auto complete to reference an existing person and add a bio and a photo. There is already photo support for instructors so please use that. I want these to be separate photos. In a later change, we will add photo support for service providers. I could easily see someone wanting a different personal photo as well as distinct instructor and massage therapist pictures. So copy the same photo for now but use MasonInstructor.jpg and CalebInstructor.jpg.

**OQ-3 — Product binding for the new classes.** The existing Knotty Yoga slots bind to the `class-dropin` product. Should Handstands and Partner Acrobatics reuse `class-dropin`, or get their own products (which is what per-class pricing would eventually need)?
- Mason- They should also use class-dropin

**OQ-4 — The Handstands prerequisite cannot be expressed as written.** ⚠️ This is the one that blocks work. `class_requirement_group_literals` supports a **permission** or a **skill level** — not "has attended class X". Options: (a) create a skill level like "Knotty Yoga Attendee" and require that, granting it manually/by staff; (b) create a permission and require that; (c) treat it as descriptive only (`inline_description` text, no enforced literal); (d) extend the requirement model with an attendance literal — a real feature, not seed data. Which?
- Mason- You are sniffing glue. Each slot currently has a Requires Attending option that must be a class that wraps up right before this class finishes. The support is already there. So please look it up and use it.

**OQ-5 — Testing the seeder.** `create_database.cpp` lives in `knottyyoga_database_helper`, which has **no test target**, so nothing in it is covered — this is how a `Logging::Log()` typo shipped past a green run before. Your instruction is to test everything testable. I have planned to push all logic into tested helpers (D3), leaving the seeder as flat data. Do you also want a **new test target for the seeder** (build it into `knotty_yoga_tests` and assert the seeded row counts/values against a scratch database)? That is the only way the seed data itself is ever verified automatically.
- Mason- Sure, that sounds good.

**OQ-6 — Event visibility.** You specified "allow booking and show on home page 14 days in advance". `event_sessions` has a separate `show_on_upcoming` / `upcoming_visible_days_before` pair. Should those be set to 14 as well, or left off? Also — "allow booking" maps to `status`; I plan `scheduled`, confirm.
- Mason- Let's leave those two blank.

**OQ-7 — Gold pricing.** The $10 Gold price is a `product_prices` row keyed on `permission_id`. Which permission is "Knotty Yoga Gold" for pricing purposes? There are three Gold products (single/couple/family) and the memory note says class access is permission-based. Also: `price_kind` — is a class-series price `per_session`, and is the $30 likewise?
- Mason- Please use the permission gold_member. All three memberships should grant that permission.

**OQ-8 — Thursday 18:00 collision.** Phase 2.6 puts Partner Acrobatics at Thu 18:00–19:00 in the seeded room, and Phase 3.3 puts Aerial Series at Tue/Thu 18:00–19:00. In `--recreate_database_test` both exist at once in "Main Gym". Is that intentional (they are different rooms in reality), should Aerial Series go in a different room, or should one move?
- Mason- Good catch. Let's do 7-8pm for the aerial series.

**OQ-9 — Timezone.** All the times you gave ("6pm", "10am") are wall-clock. Slots store `start_time_minutes` (wall clock, fine), but `event_sessions` and `provider_availability` store absolute `*_us` timestamps, so they need a zone. Is the studio timezone available as a config secret I should read, or should I hardcode `America/Los_Angeles` for seed data?
- Mason- Let's just use Pacific time for now and I'll add this as a follow up item.

**OQ-10 — "Two following Saturdays" when run ON a Saturday.** If the tool runs on a Saturday, do you want that same day plus the next (today counts), or the next two strictly after today? I have assumed strictly after.
- Mason- Let's not count the current day and do the two next Saturdays if the tool is run on a Saturday.