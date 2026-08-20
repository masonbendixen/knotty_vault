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

**Decisions from the answered open questions (8/18/2026):**

- **D6 — The prerequisite is `predecessor_class_schedule_slot_id`, which already exists.** ⚠️ *I got this wrong in v0.1* — I looked at `class_requirement_group_literals` (permissions and skill levels) and concluded the model could not express it. The right mechanism is on the SLOT: `class_schedule_slots.predecessor_class_schedule_slot_id`, documented as "SL-11 same-day sequencing", already referenced by `calendar_helper`, `attendance_template_helper`, the `class_schedule_slots` table helper and `create_database.cpp` itself. Requirement *groups* are for standing eligibility (a permission or a skill level); the predecessor column is for "you must have been in the class that finishes immediately before this one". Handstands Mon 19:00 points at the Knotty Yoga Mon 18:00 slot. **Nothing new is needed.**
- **D7 — Instructor photos are their own images.** `instructors` already has photo support and the admin "add new" flow takes a person reference, a bio and a photo. Seed `MasonInstructor.jpg` / `CalebInstructor.jpg` as separate files (same bytes as the profile photos for now). A person may reasonably want distinct profile / instructor / therapist pictures; service-provider photo support is a later change.
- **D8 — All new class slots bind to the existing `class-dropin` product.** No new products for Handstands or Partner Acrobatics.
- **D9 — Gold pricing keys on the `gold_member` permission (id 3, `pricingEligible = true`).** All three Gold membership products must grant it — today only some do, so this plan closes that gap.
- **D10 — Seed data gets its own test target.** `create_database.cpp` is not currently compiled by any test; the plan adds seeder coverage so the seeded rows themselves are verified, not just the helpers they call.
- **D11 — Pacific time for all absolute timestamps.** Hardcoded for seed data; making the studio timezone a config secret is a follow-up Mason is tracking separately.
- **D12 — "The two following Saturdays" excludes today.** A Saturday run seeds the next two Saturdays, not today plus one.

---

# Phase 1 — Foundations: dates, images, and the new flag ✅ **IMPLEMENTED 8/18/2026**

Everything later depends on these three. Lower layers first within the phase.

> ⚠️ **Phase 1 touches honuware** (`date_time_util`), which the plan did not anticipate — see 1.0. It needs a CI push and a pin bump in both consumers before Phase 2 builds cleanly.

### 1.0 `DateTimeUtil::LocalWallClockToUs` *(honuware foundation — NEW, not in the original plan)*
- [x] Add `LocalWallClockToUs(dayTimestampUs, minutesAfterMidnight, timezone)` to `components/foundation/util/date_time_util.h/cpp`.
- [x] Tests: resolves a wall-clock time; independent of the time-of-day passed in; **correct on a spring-forward day**; correct on a fall-back day; agrees with `GetMidnightUs` at zero minutes; correct in a no-DST zone (Phoenix).

- [x] **Fixed a pre-existing DST bug in `GetMidnightUs` that the new test exposed** — see below.

**⚠️ `GetMidnightUs` was wrong on DST transition days, and had been all along.** Writing the spring-forward test above produced a failure whose *shape* was the interesting part: the naive `GetMidnightUs + 10h` and the correct `LocalWallClockToUs` came out **identical**, when the whole point of the test was that they differ. Two errors were cancelling.

`GetMidnightUs` zeroed `tm_hour/min/sec` but never reset `tm_isdst`, so it inherited the flag from the **source** instant. Asked for midnight from an afternoon on a spring-forward day, that flag says "DST" — true of the afternoon, false of midnight — so `mktime` read 00:00 as PDT when it was really PST and returned an instant **one hour early**, i.e. 23:00 the previous day. The naive sum then missed the skipped hour on the way back up, landing on the right answer by accident.

Fixed with `tm_isdst = -1` plus a regression test. **The existing `GetMidnightUsPacificTimezone` test uses 14 March — *after* the transition, where both the source and midnight are PDT — which is exactly why this survived.** A date test that never crosses a boundary proves very little.

Blast radius: nothing else in honuware calls it; app-side the only production caller is `room_availability_helper`, so availability day-boundaries were an hour off two days a year. With the fix the original assertion holds on its own terms — midnight becomes 08:00 UTC, so naive gives 18:00 against the correct 17:00, differing by exactly the skipped hour.

**Why `LocalWallClockToUs` was needed.** Converting "10:00 local on this day" to an absolute timestamp cannot be `GetMidnightUs(...) + minutes`. On a US spring-forward day the clock jumps 02:00 → 03:00, so that local day is 23 hours and wall-clock 10:00 is only NINE elapsed hours after midnight — the naive sum lands an hour late. Every seeded event session and every generated availability row is a wall-clock time stored as an absolute `*_us`, so this is load-bearing for Phase 3, not a nicety. It belongs in foundation beside `GetMidnightUs`/`GetDayOfWeek` (which already own the timezone machinery and its mutex), not copied into the app. A test pins the naive form as wrong by exactly one hour so the implementation cannot quietly regress to it.

### 1.1 Date helper *(app — business_logic/scheduling, testable)*
- [x] Add `seed_date_util.h/cpp` in `business_logic/scheduling/`: `NextWeekdayAfter`, `NextWeekdaysAfter`, `EndOfFollowingMonthUs`, `WeekdayOccurrencesBetween`, plus `kSeedTimezone` and the day-of-week constants.
- [x] Tests (16): a Saturday run returns the NEXT Saturday not today; every weekday; results are local midnights; successive weeks hold their weekday **across a DST boundary**; month-end rollover December→January and January→February; stable across the source month; occurrences keep the same **local** wall clock across a DST transition; inverted range, no-matching-weekday, and start-inside-window cases.

**Two implementation notes.** Day stepping re-derives local midnight each step instead of adding 24 h, because a DST day is 23 or 25 hours and accumulated 24 h steps drift onto the wrong calendar day — that is why "three successive Saturdays across 1 November" is a test. And `WeekdayOccurrencesBetween` bounds on the **start** of a session, so a class beginning just inside the window is kept rather than dropped for finishing outside it.

### 1.2 Seed images into the tree
- [x] Copied into `src/database_helper/img/`: `Mason.jpg` (386 KB), `Caleb.jpg` (158 KB), `KnottyYoga.jpg` (1.2 MB), `Handstand.jpg` (697 KB).
- [x] **Converted** the Partner Acrobatics source from PNG to real JPEG as `PartnerAcro.jpg` — 9.9 MB PNG → 767 KB JPEG, magic bytes verified `ff d8 ff` (OQ-1).
- [x] `MasonInstructor.jpg` / `CalebInstructor.jpg` copied from the profile images (D7 — separate files, same bytes for now).
- [x] The CMake step copies the whole `img/` directory, so no build change was needed.

⚠️ **Trap worth recording:** `which convert` resolves to `C:\Windows\system32\convert` — the **FAT-to-NTFS filesystem converter**, not ImageMagick. Running it on an image path would have been a genuinely bad mistake. The conversion used `ffmpeg -q:v 3` instead.

*(Note for Phase 2: these are full-resolution photos — the largest is 1.2 MB. The uploader stores the source and derives scaled copies on demand, so that is workable, but if seeding feels slow that is where the time goes.)*

### 1.3 The `--recreate_database_test` flag *(database_helper)*
- [x] `ABSL_FLAG(bool, recreate_database_test, ...)` added to `database_helper/main.cpp`, wired into the mode count and the usage text, routed to `RunRecreate(/*includeTestData=*/true)`.
- [x] `RunRecreate` takes `includeTestData` and names its log events / error text after the mode that actually ran, so a failure says which flag produced it.
- [x] **Both** flags keep the `EnsureDestructiveAllowed()` guard — `--recreate_database_test` also drops the database and is not the "safe" one.
- [x] `bool includeTestData = false` threaded through `CreateAndPopulateDatabases` → `CreateSchemaAndPopulate`, ending in one `if (includeTestData) PopulateTestData(...)`. **The default is what keeps `Tenancy::ProvisionTenant` correct** — a real tenant must never be provisioned with sample sessions.
- [x] `PopulateTestData` added as an empty, logged entry point so the plumbing lands and can be exercised before Phase 3 fills it.
- [x] ⚠️ Build `knottyyoga_database_helper` explicitly — the standard gate does **not** build this target. **It caught a real break.**

**⚠️ The defaulted parameter broke `--create_tenant`, and every gate stayed green.** Adding `bool includeTestData = false` to `CreateSchemaAndPopulate` changed the function's **type**, and a default argument is not part of a function's type — so `&CreateSchemaAndPopulate`, passed to `Tenancy::ProvisionTenant` as a `std::function<void(DatabaseHelper, DatabaseInfo)>`, stopped converting:

```
error: invalid initialization of reference of type 'const Tenancy::CreatePopulateFn&'
       from expression of type 'void (*)(DatabaseHelper, DbSchema::DatabaseInfo, bool)'
```

**Tenant provisioning would not compile while honuware (1733), knottyyoga (5025) and CommunityFinder (1754) were all green.** That is the same structural hole the `Logging::Log()` typo went through, and the third time this target has caught something the gate cannot see.

Fixed by wrapping at the call site instead of restoring a two-argument overload — the lambda states the intent that the default only implied: *a provisioned tenant is a real studio and never gets sample data.*

---

# Phase 2 — Base seed data (`--recreate_database`) ✅ **COMPLETE 8/20/2026**

### 2.9 Extract the seed functions so they can be tested ✅ *(prerequisite for 2.8 and Phase 3's tests)*
- [x] Moved into `database_helper/seed_app_data.h/cpp` behind a public `namespace Seed`: the shared `MakeAddRowLambda` / `SeedImageDirectory` / `AttachSeedPhoto` / `Lookup*` helpers, the `kMasonEmail` / `kCalebEmail` constants, the **data** seeders (`PopulateClasses`, `PopulatePhotoSupportTables`, `PopulateProducts`, `PopulateFacilities`, `PopulateLocationRoomTypes`, `PopulateLocationRooms`, `PopulateProviderTypes`), all four Phase 2 seeders, and `PopulateTestData`.
- [x] `create_database.cpp` keeps the **bootstrap** — `CreateTables`, `PopulateTables`'s ordering, `CreateSchemaAndPopulate`, `CreateAndPopulateDatabases` — plus the **admin-metadata** seeders, and calls the rest through the header. One `using namespace Seed;` inside its anonymous namespace keeps ~200 call sites spelled the same.
- [x] `target_link_libraries(knotty_yoga_tests PUBLIC knotty_yoga_database_helper)`; `seed_app_data_test.cpp` registered into `knotty_yoga_tests`. `knotty_yoga_database_helper` stays OUT of the honuware DAG on purpose (comment updated in `cmake/honuware_layering.cmake`) — it sits above `knotty_yoga_core` with nothing below it that could link back up.
- [x] CMake copies `src/database_helper/img/` next to **`knottyyoga_tests`** as well, so the photo assertions exercise the real upload path instead of asserting "no photo".

**What only moved as far as it had to.** `create_database.cpp` is ~3500 lines and roughly 2/3 of it is `PopulateAdminColumnDataInfo` / `PopulateAdminColumnFriendlyNames` / friendly names / display templates / enums — *admin UI metadata*, not studio data, and nothing in Phase 2 or Phase 3 verifies it. Moving that too would have been a 3000-line relocation buying nothing. The split is "data the studio has" vs "how the generic admin editor renders it", which is a boundary that means something rather than a line count.

### ⚠️ 2.9 also closed the gate hole that motivated all of this
`build_and_test.sh` builds only `knottyyoga_tests`, so **`create_database.cpp` was compiled by nothing in the gate** — the standing warning next to the `Logging::Log()` typo. Now that `knotty_yoga_tests` links `knotty_yoga_database_helper`, that file compiles on every gate run. The separate `knottyyoga_database_helper` build is still needed for `main.cpp` (the flags/`absl` layer), but the 3500-line file behind it is covered.

### ⚠️ `DbPair` was a duplicate, and the anonymous namespace was hiding it
The move failed to compile with `call of overloaded 'DbPair(...)' is ambiguous` at every call site. honuware already defines `DbPair(std::string_view, std::string_view)` at **global scope** in `sql_util/database_common.h`; `create_database.cpp` carried a byte-identical copy in its anonymous namespace. That copy *hid* honuware's — unqualified lookup stops at the innermost scope with a match — so the two coexisted invisibly for as long as the file has existed. Moving the copy into a named namespace and re-exposing it with a using-directive put both at the same scope, and every one of the ~200 call sites went ambiguous at once. Deleted the copy; honuware's is the one definition now.

*The general shape is worth remembering: an anonymous namespace silently wins ties, so a duplicate inside one is invisible until the day it moves.*

### 2.x Also done, and needed by the tests
- [x] **`PopulateClassSchedules` now resolves the facility and room by natural key too** (`knotty-yoga-studio` / `Main Gym`), finishing what D5 started — `kFacilityId = 1` and `kRoomId = 1` were the last two seed-order constants in Phase 2 code.
- [x] **`PopulateLocationRooms` likewise** — it hardcoded `facilityId = 1` and room-type ids 1/2/4. Both were correct *only in a virgin database*: in a harness transaction the sequences have long since moved, so the old code would have inserted rooms against a facility that does not exist and failed on the FK. That is the concrete reason the extraction alone was not enough to make these functions testable.
- [x] Both now log and return when their prerequisite row is missing, rather than writing against id 0.

### 2.1 People photos ✅
- [x] Attach `Mason.jpg` to `masonbendixen@gmail.com` and `Caleb.jpg` to `mr.calebault@gmail.com` via `AttachSeedPhoto` on the `people` table, resolving `person_id` by email (`PopulateSeedPhotos`).
- [x] Verified `people` IS photo-supported — registered **framework-side** in honuware's `create_framework_tables.cpp`, not in the app's `PopulatePhotoSupportTables`. No change needed.

### 2.2 Instructors ✅
- [x] `PopulateInstructors()`: `instructors` rows for Mason and Caleb, each with a short bio.
- [x] Attach `MasonInstructor.jpg` / `CalebInstructor.jpg` to the `instructors` rows (D7 — its own image, not the profile photo).
- [x] Verified `instructors` is already in `PopulatePhotoSupportTables`. No change needed.

### 2.3 Service providers ✅
- [x] `PopulateProviderTypeAssignments()`: both assigned to the existing `massage_therapist` provider type with `is_accepting_bookings = true`, the type id looked up by code.
- [x] *(Noted, not built: service-provider photo support is a later change — D7.)*

### 2.4 Class photos ✅
- [x] Attach `KnottyYoga.jpg`, `PartnerAcro.jpg`, `Handstand.jpg` to the `classes` rows found **by name** (`PopulateSeedPhotos`).
- [x] Verified `classes` is already photo-supported.

### 2.5 Instructors on the existing slots ✅
- [x] `instructor_person_id` set to Mason on the Mon 18:00 slot and Caleb on the Wed 18:00 slot, both person ids looked up by email.

### 2.6 New class slots ✅
- [x] **Handstands, Mon 19:00–20:00**, instructor Mason: `class_instances` (perpetual) → `class_schedules` (default) → slot, bound to `class-dropin` (D8).
- [x] **Partner Acrobatics, Thu 18:00–19:00 and Sun 10:00–11:00**, instructor Mason — one instance/schedule with two slots, also `class-dropin`.
- [x] Both in the seeded facility (1) / Main Gym (1). A shared `AddPerpetualSchedule` lambda builds the instance+schedule pair so all three classes get the identical three-level shape.

### 2.7 Handstands prerequisite ✅ *(D6 — the mechanism already existed)*
- [x] `AddSlot` now RETURNS its id; the Knotty Yoga Monday slot id is captured and written to `predecessor_class_schedule_slot_id` on the Handstands Mon 19:00 slot.
- [x] The sequencing is genuinely back-to-back: Knotty Yoga is 18:00 + 60 min = 19:00, exactly when Handstands starts. That adjacency is what makes the column mean "you must have been in the class that just finished".
- [x] No `class_requirement_groups` row — that table is standing eligibility (permission / skill level), not same-day sequencing.

### 2.x Also done, not on the original checklist
- [x] **Replaced the hardcoded `kClassId = 1` and `kProductId = 10` with lookups** by class name and product code (D5). Those constants encoded seed ORDER, so appending a product above them would have silently rebound every class — and a photo on the wrong class is a bug nobody notices until a studio sees its home page.
- [x] Added `LookupIdByColumn` / `LookupPersonIdByEmail` / `LookupClassIdByName` / `LookupProductIdByCode`, each logging and returning 0 rather than throwing, so one missing seed row cannot abort database creation.

### 2.8 Tests ✅
- [x] No new table-helper methods were needed — Phase 2 uses `DbCrud::GetRow`, `AddRowToTableFetchInt64PrimaryKey` and the existing `AttachSeedPhoto`, all already covered.
- [x] **Seed verification (D10 / OQ-5) — `src/database_helper/seed_app_data_test.cpp`, 18 tests, green on Linux (3.5 s).** Each seeds its own prerequisites, calls the **real** seeder (not a copy), and asserts the rows:

| Area | What is pinned |
|---|---|
| Artwork | `SeedImageDirectory()` really is `<exe dir>/img` **and exists** — a guard, because every photo assertion below is vacuous otherwise, and "no photo attached" would send the reader hunting in the wrong place. A missing file logs and carries on rather than aborting the bootstrap. |
| Lookups | All three `Lookup*` return 0 (not throw) on an absent row. |
| Catalog | All six class names exist — the three that `PopulateSeedPhotos` / `PopulateClassSchedules` resolve by hand are load-bearing, so a rename breaks the test rather than the seed. `class-dropin` is kind `class`; `intro-workshop` is kind `event` with capacity 20 (Phase 3.1 reads that). |
| Rooms | Main Gym binds to the facility found by **code** and the room type found by **code**; with no facility seeded, nothing is written. |
| 2.1 / 2.4 | Both people and exactly the three named classes get pictures — **and Tumbling does not.** A photo on the wrong class is the bug D5 exists to prevent, so "the others stayed empty" is half the assertion. Class artwork still lands when the people are absent. |
| 2.2 | An `instructors` row per person, non-empty bio, and its **own** photo on the instructors row (D7). |
| 2.3 | Both assigned to `massage_therapist`, `is_accepting_bookings` true; nothing written when the provider type is missing. |
| 2.5 | Knotty Yoga has exactly 2 slots — Mon 1080/60 Mason, Wed 1080/60 Caleb. |
| 2.7 | Handstands Mon 1140 names the Knotty Yoga Monday slot as predecessor, **and** `predecessor.start + predecessor.duration == handstands.start`. The adjacency is what makes the column mean "the class that just ended"; asserting only the id would let a future edit move one of them and keep the test green. |
| 2.6 | Partner Acrobatics Thu 1080 + Sun 600, both Mason. |
| D5 | **Decoy row test.** A decoy class, product, facility, room type and room are inserted FIRST, so `kClassId = 1` / `kProductId = 10` / `kFacilityId = 1` / `kRoomId = 1` would each now name a decoy. Asserts the instance binds to the real drop-in product, the slot to the real facility + Main Gym, and that the decoy class got no schedule at all. |
| Skip path | With no product/facility/room, `PopulateClassSchedules` writes **nothing** — a half-built three-level hierarchy is worse than none. |
| Phase 3 seam | `PopulateTestData` is reachable and currently adds nothing, so the day it starts adding rows this test has to be updated deliberately. |

- [x] No fixtures (CLAUDE.md): a `SeedPrerequisites()` free function does the setup and each test calls it. Slots are found **by weekday**, never by array index.
- [x] The `people` photo-support registration is framework-side (`create_framework_tables.cpp`) and the harness creates tables without running the framework seed, so the test inserts that one row itself — noted inline so it does not read as an oversight.
- [x] The photo tests really do decode and store the full-resolution JPEGs (386 KB … 1.2 MB), which is where the suite's 3.5 s goes. Worth it: it is the only thing that proves the upload path works on the actual files, not on a stub.

**The photos were never RUN before this.** Phase 2 shipped as "written and compiles". The first execution of `PopulateSeedPhotos` happened in this test run, and it is the first evidence any of the artwork decodes and stores — including `PartnerAcro.jpg`, the one converted from PNG in Phase 1.2. `--recreate_database` against a live database (Phase 4.1) is still outstanding, but the seed *logic* is now exercised.

---

# Phase 3 — Test-only seed data (`--recreate_database_test`)

All of this lives in `PopulateTestData()` and runs only under the new flag.

### 3.1 Intro Workshop event sessions
- [ ] Create two `event_sessions` for the existing `intro-workshop` product on the **next two Saturdays**, 10:00–11:00, using the Phase 1.1 helper.
- [ ] Set `show_on_home_page = true` and `home_page_visible_days_before = 14`; **leave `show_on_upcoming` and `upcoming_visible_days_before` unset** (OQ-6).
- [ ] `status = scheduled` so the sessions are bookable; capacity from the product's `default_capacity` (20).

### 3.2 Aerial Series product
- [ ] Add product `aerial-series` — "Aerial Series", description "Introduction to aerial acrobatics on rope and fabric", kind `class_series` (the constant already exists).
- [ ] `product_prices` on the seeded 2026 price schedule: **$30/session** general (`permission_id` NULL) and **$10/session** keyed on the **`gold_member`** permission (id 3, already `pricingEligible`) — D9.
- [ ] **Gap to close:** confirm all three Gold products (`gold-membership`, `gold-couples`, `gold-family`) grant `gold_member` via `product_entitlement_rules`; add the missing rules. `PopulateProductEntitlementRules` has a standing comment that the gold membership permission "would be added separately", so at least one is likely missing today.

### 3.3 Aerial Series class schedule
- [ ] Create the `class_instances` → `class_series_instances` → `class_schedules` chain bound to the `aerial-series` product, valid from now through **end of the following month**.
- [ ] Slots **Tue(2) and Thu(4) 19:00–20:00**, `instructor_person_id` = Caleb — moved off 18:00 to clear the Partner Acrobatics Thursday slot (D/OQ-8).

### 3.4 Caleb's provider schedule
- [ ] Create a `schedule_templates` row for Caleb, Mon–Fri, with five `schedule_template_entries` at 09:00–17:00 (540–1020 minutes).
- [ ] Call `ScheduleTemplateHelper::GenerateAvailability` for **today through end of the following month** to populate `provider_availability`.

### 3.5 Tests
- [ ] Tests for every new business-logic helper (the date helper is the main one).
- [ ] A test asserting `PopulateTestData` is **not** reachable from the plain `--recreate_database` path.
- [ ] Seed verification for the test data (D10): two Saturday event sessions in the future with `home_page_visible_days_before = 14` and the upcoming pair unset; the `aerial-series` product with both prices and the Gold one keyed on permission 3; Tue/Thu 19:00 slots with Caleb; Caleb's template with five entries and generated availability rows.
- [ ] A test that **no two slots collide** on (facility, room, day, time) after the full test seed — the Thursday clash in v0.1 of this plan is exactly what that catches.

---

# Phase 4 — Verification

### 4.1 Build and run
- [ ] Build `knotty_yoga_tests` **and** `knottyyoga_database_helper` (the latter is not in the standard gate).
- [ ] Run `--recreate_database` on a scratch database; confirm photos, instructors, providers and the six slots.
- [ ] Run `--recreate_database_test`; confirm the events, series, prices and availability.

### 4.2 Hand-testing steps
- [ ] Write live-server steps per the precise-instructions rule: exact menu → submenu → dashboard item → field labels, for each thing seeded.

---

# Follow-ups (recorded, not in scope)

- [ ] **Studio timezone as configuration.** Seed data hardcodes Pacific (D11). Mason is tracking making this a config secret so a tenant in another zone gets correct absolute timestamps — this matters beyond seeding, since `event_sessions` and `provider_availability` both store absolute `*_us`.
- [ ] **Photo support for service providers.** So a person can have distinct profile / instructor / massage-therapist images (D7).

---

# Open questions — ALL RESOLVED 8/18/2026

*Mason's answers are inline below. The plan above has been updated; these are kept for the reasoning.*

**Summary of what changed in the plan:** OQ-4 was wrong on my part and unblocked the phase (see D6) — I proposed extending the requirement model when the mechanism already existed. OQ-8 moved Aerial Series to 19:00–20:00. OQ-2 and OQ-5 each *added* work (separate instructor images; a seed-verification test target). OQ-7 surfaced a gap: all three Gold products must grant `gold_member`, and at least one probably does not today.

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