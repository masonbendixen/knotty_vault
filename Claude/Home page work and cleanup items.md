---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/29/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I would like the home page of the website to show upcoming events and essentially marketing material for when a user is not logged in. Once they have created an account, I'd like to show any upcoming events they have signed up for as well as upcoming events they could sign up for. I'd like to show the upcoming class schedule for the next four days as items on the home page with options like being able to make themselves as attending or not attending. If they don't have a membership, show the membership tiers on the home page. We should have a link to setup their class schedule template.

Let's do the following cleanup items:
- Let’s simplify the menu structure. Let’s have Our Classes just have All Classes and Our schedule. Let’s not have entries for separate classes.
- Let’s have the classes page have a link to the calendar
- Let’s have the Calendar have a link to the schedule template
- Let’s have the schedule template page have a link to the notification settings
- Get rid of Upcoming Events from the Services menu and make the Services menu just go to the current Browse Services page (so not be a dropdown menu anymore)
- Get rid of the all classes under Our schedule
- Turn Our Classes go straight to the current Our Classes / Our schedule so it is a menu item not a drop down menu
	- Have the attend this class buttons like the calendar
	- Have a link to the schedule template
	- If someone isn’t logged in, have all the classes just up with their time slots
		- Have the classes that require a membership be a link to buy a membership for people who don’t have a membership
- Change shop to Memberships
- We need a getting started page with detailed instructions
- Let’s redo the Our schedule page
- For a class, the Upcoming Sessions has too many entries. Link to calendar instead of showing upcoming sessions.
- Note that series (and presumably workshops), don’t really have a place to sign up for them anymore. This should be easy to discover and sign up. They should be able to sign up from the series and workshops tab in the profile, from the calendar, and from the class page.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here

> Plan drafted 7/29/2026 from a full read of the nav/home/calendar/schedule/class/shop/portal frontend and the scheduling + products C++ API surface. Checkboxes get checked off as each item is implemented. Open questions are collected in the **Open Questions** section at the bottom — please answer there.
>
> **Update 7/30/2026:** round-1 answers are folded into the phases below. Three follow-ups still need input — see the "Still open after round 1" box at the top of the Open Questions section.
>
> **Update 7/30/2026 (round 2): all open questions are resolved — the plan is execution-ready.** Final calls: the account-dropdown "Memberships" entry is removed (Phase 1.1), the day-grouped schedule layout is confirmed (Phase 2.4), and OQ-9 landed as option (c) — Phase 2.5 removes the weekly-schedule surface end to end (frontend seam + backend endpoint/helper/tests).

## Current State (grounding, 2026-07-29)

What the code does today, so the phases below are uncontroversial:

- **Menu** is built entirely in `ui/src/app/shared/services/header/mockHeaderResponse.ts` (despite the name, it is the live menu builder; `HeaderService.refreshHeaderData` feeds it the class catalog fetched from `GET /api/classes` just to populate the per-class dropdown entries). Current top-level: Get Started (`/start`), About ▾, Our Classes ▾ (Our Schedule, All Classes, + one entry per class), Services ▾ (Browse Services, Upcoming Events), Upcoming Events, Shop, [auth] Your Calendar, [staff/admin items], user dropdown / Sign In. The user dropdown contains two **dead links** ("My Goals" → `/goals`, "My Classes" → `/myclasses` — no such routes).
- **Home page** (`pages/public/home-page/`) = photo carousel + hardcoded hero copy + a single "Next Upcoming Event" card (`GET /api/visible_event_sessions?placement=home_page`, takes `sessions[0]`) + "View All Events" link. **No logged-in personalization at all.** `/start` maps to the same `HomePageComponent` — there is no real Getting Started page. The About ▸ Gallery menu item points at `/#gallery` but the home template has **no `id="gallery"` anchor**.
- **Our Schedule** (`/schedule`, `pages/public/our-schedule/`) = read-only Sun–Sat weekly timetable from `GET /api/schedule/week`. That payload has **no `class_schedule_slot_id`** and no per-viewer access state, so it cannot drive attend toggles.
- **Calendar** (`/calendar`, public route) is the rich surface: `GET /api/calendar` returns `CalendarItem`s carrying `kind` (`class`/`workshop`/`series`/`event`/`service_booking`), `class_id`, `class_instance_id`, `class_schedule_slot_id`, `occurrence_date_us`, per-viewer `access` (`eligible`/`members_only`/`needs_skill`), `required_permission_id/name`, `missing_skill_ids`, `signup_window_open`/`signup_opens_at_us`, capacity, cancellation. The chip (`calendar-event.component`) renders Booked/Waitlisted/Sold out/Locked/Cancelled/Remind-me/attend-toggle states; the attend toggle posts `POST /api/me/template/exception` via `attendance-dialog`; click routing lives in `CalendarNavigationService`. **Anonymous viewers get no access/lock state** — `CalendarHelper` skips the gate when `personId == 0`.
- **Class detail** (`/classes/:id`) renders an **unbounded** `upcoming_sessions` list. For workshops that list doubles as the booking surface (`POST /api/me/materialize_class_occurrence` → `/shop/event/:sessionId`); series get "Buy full series / Join prorated" buttons → `/shop/series/:classInstanceId` — but the series booking page **only works when a `SeriesRun` arrives via router state** (`history.state.run`), which is why the calendar can't link to it (its series clicks bail to the class page).
- **Workshops & Series portal tab** (`/my/upcoming-offerings`) lists offerings from `GET /api/me/upcoming_signup_offerings` but renders **inert text "Sign-ups are open"** with no Book button; the payload has **no `class_instance_id`**, so a direct series-booking link isn't possible today.
- **Memberships storefront** exists at `/shop/subscriptions` (`SubscriptionCatalogComponent`, filters `kind === 'subscription'`, inline tier cards — no shared card component). `/shop` (`CatalogComponent`) shows `one_time` + `subscription` products. Membership state is derived ad-hoc from `GET /api/subscriptions` (active subscription, else `shared_entitlements[0]`) — that's what the profile hub does.
- **Per-user feeds already built** (Classes Phase 5/11): `GET /api/me/upcoming_classes?from&to` (eligible occurrences + `on_template` + exception state — the attend/skip model), `GET /api/me/today_classes`, `GET /api/my_bookings?status=upcoming`, `GET /api/me/upcoming_signup_offerings?from&to`. `UpcomingClassesComponent` (`pages/account/upcoming-classes/`) already renders a day-grouped feed with "I'll be there" / "I can't make it" controls and supports `[embedded]="true"`.
- **Anonymous-safe endpoints**: `/api/calendar`, `/api/visible_event_sessions`, `/api/schedule/week`, `/api/classes`, `/api/classes/:id`, `/api/class_series_runs/:classId`, `/api/catalog_products`, `/api/home_page_photos/:count`. Everything under `/api/me/*`, `/api/my_*`, `/api/book_*` requires login.
- **Seed data** (fresh DB via `knottyyoga_database_helper`): one recurring class **"Knotty Yoga"** (Mon + Wed 18:00, 60 min, Main Gym at Knotty Yoga Studio, product "Class Drop-In"); products include **"Intro Workshop"** (event kind), **"Knotty Yoga Gold Membership"**, **"Knotty Yoga Gold Couple's Membership"**, **"Knotty Yoga Gold Family Membership"** (subscriptions), massage/spa services.

## Design Decisions

1. **Final menu shape** (desktop and mobile share one model, so this is one edit in `mockHeaderResponse.ts`):
   `Get Started` → `/start` (real page, Phase 6) · `About ▾` (unchanged) · `Our Classes` → `/schedule` (direct item, no dropdown) · `Services` → `/shop/services` (direct item, no dropdown) · `Upcoming Events` → `/events` (kept top-level — OQ-2 ✅) · `Memberships` → `/shop` (label-only rename of Shop, same target — OQ-1 ✅; the shop lists nothing but the membership products today) · [auth] `Your Calendar` · staff/admin unchanged · user dropdown minus the two dead links and minus its redundant "Memberships" entry (OQ-1 follow-up ✅ — "make it as simple as possible"). With the per-class dropdown entries gone, `HeaderService` no longer needs to fetch `/api/classes` at all — the fetch, per-identity cache, and the `classes` parameter of `mockHeaderResponse` are removed.
2. **The redone Our Schedule page is rebuilt on the calendar feed**, not `GET /api/schedule/week`. Rationale: attend toggles need `class_schedule_slot_id` + `occurrence_date_us` + per-viewer `access` + the attendance overlay — exactly what `/api/calendar` + `/api/me/upcoming_classes` provide and what `/api/schedule/week` lacks. The calendar's chip mechanics ("like the calendar", per the Overview) are extracted to `shared/` and reused. Two small backend additions make one data source serve both auth states: anonymous access states, and the predecessor-class note (parity with the old page). Per OQ-3 ✅ the page adopts the calendar's mode behavior: logged-in defaults to **eligible classes only** with a "Full Schedule" toggle; logged-out always sees the full schedule as an advertisement (the `/calendar` page already works exactly this way — `CalendarService` defaults to `'my'` when logged in, `'full'` otherwise — so no calendar-side change is needed). Per OQ-9 ✅ (option c — no dead code) the **entire weekly-schedule surface is removed** once the redone page lands: the frontend `getWeeklySchedule` seam method + its types AND the backend endpoint + `WeeklyScheduleHelper` + their tests (Phase 2.5).
3. **Class detail keeps a booking surface for paid kinds.** "Link to calendar instead of showing upcoming sessions" is applied literally to recurring classes (their derived list is the one that explodes — 90 days × slots). Workshops keep a **capped** occurrence list (it is the only place a workshop occurrence can be materialized + booked from the class page) plus the calendar link; series keep the runs/buy section. (Cap of 5 confirmed — OQ-4 ✅.)
4. **Series booking becomes deep-linkable.** New public endpoint `GET /api/class_series_run/<classInstanceId>` returns the single run; `SeriesBookingComponent` self-loads when router state is missing. `class_instance_id` is added to `UpcomingSignupOffering`. Then the Workshops & Series tab gets real Book CTAs and the calendar routes eligible open-window series/workshop clicks straight into the booking flow.
5. **Home page composition** (one `HomePageComponent`, sections toggled on `authData$`):
   - Both states: photo carousel (gains `id="gallery"`) + hero marketing + "Get Started" CTA banner.
   - Logged out: featured events strip (`placement=home_page`, up to 4 cards — today's curated visitor-marketing surface, kept) + membership tier cards (shared component extracted from the subscriptions catalog — OQ-8 ✅).
   - Logged in: "Your upcoming events" (`my_bookings` upcoming, first few) and "Events you could sign up for" — per OQ-5 ✅ simply the existing Upcoming Events list (`placement=upcoming`) minus already-booked sessions, capped at the **4 soonest** (no offerings merge); "Next 4 days at the studio" (embedded `UpcomingClassesComponent` with a new `daysAhead` input — attend/not-attend toggles come for free); tier cards **only when no membership** (same `getSubscriptions()` derivation the profile hub uses — OQ-6 ✅); and a "Plan your weekly schedule" link → `/my/my-schedule`.
6. **Getting Started** becomes a real public page at `/start` with step-by-step instructions (draft copy in Phase 6 ships as-is; wording gets revised together after it renders — OQ-7 ✅).
7. **Testing discipline**: every backend change lands with helper + endpoint tests; every frontend change lands with component specs, and any `ServerAccess` change updates interface/network/proxy/mock + `ServerAccess.mock.spec.ts`. Each phase ends with live-server hand-testing steps against the seeded database.

---

## Phase 1 — Menu & cross-link cleanup (frontend only)

> No backend work in this phase. All items are small, independent, and unblock the later phases' navigation story.

> **Implemented 7/30/2026.** Angular gate green: the 7 affected spec files pass (79 specs), the full suite is **2648 SUCCESS / 0 failures**, `ng build` is clean (only the pre-existing bundle/scss budget warnings), and `npx eslint` on the touched files reports no new problems.

### 1.1 Menu restructure
- [x] `mockHeaderResponse.ts`: replace the "Our Classes" dropdown with a direct `InternalLink` **Our Classes** → `/schedule`; delete the per-class entries and the "All Classes" dropdown entry (the All Classes page stays routed and gets linked from the redone schedule page in Phase 2).
- [x] `mockHeaderResponse.ts`: replace the "Services" dropdown with a direct `InternalLink` **Services** → `/shop/services` (drops the duplicate "Upcoming Events" entry; the top-level Upcoming Events item stays — OQ-2 ✅).
- [x] `mockHeaderResponse.ts`: rename **Shop** → **Memberships**, target unchanged (`/shop`) — label-only rename per OQ-1 ✅.
- [x] `mockHeaderResponse.ts`: remove the now-redundant "Memberships" entry (→ `/shop/subscriptions`) from the account dropdown (OQ-1 follow-up ✅ — the top-level item covers the storefront; "My Subscriptions" covers what you own; the `/shop/subscriptions` route itself stays).
- [x] `mockHeaderResponse.ts`: remove the dead user-dropdown entries "My Goals" (`/goals`) and "My Classes" (`/myclasses`).
- [x] Drop the now-unneeded `classes` parameter from `mockHeaderResponse(...)`; simplify `HeaderService.refreshHeaderData` to stop fetching `/api/classes` and delete the per-identity class cache. *(The `SERVER_ACCESS_TOKEN` injection came out of `HeaderService` entirely — nothing else in the service used it.)*
- [x] Specs: update `header.service.spec.ts` (no class fetch), `mockHeaderResponse` assertions wherever they live (header/component specs), and mobile-menu spec if it asserts menu contents. Assert: Our Classes is a direct link to `/schedule`; Services is a direct link to `/shop/services`; a top-level "Memberships" item targeting `/shop` present, "Shop" label absent; the account dropdown has no "Memberships" entry; no `/goals` / `/myclasses` entries; Upcoming Events appears exactly once. *(`header.component.spec.ts` / `header-mobile-menu.component.spec.ts` assert no menu contents — nothing to change there. `table-crud.integration.spec.ts` already called `mockHeaderResponse(authData)` with one argument and still passes.)*

### 1.2 Classes page → calendar link (and keep All Classes reachable)
- [x] `class-info.component.html`: add a clearly visible "View the calendar" link/button (→ `/calendar`) near the page title, styled consistently with existing page-header actions.
- [x] `our-schedule.component.html`: add a "Browse all classes" link (→ `/classes`) in the page header **now** — with the menu entry gone, this is the classes page's entry point until the Phase 2 redo (which keeps the link).
- [x] Specs: `class-info.component.spec.ts` renders the calendar link; `our-schedule.component.spec.ts` renders the browse-all-classes link.

### 1.3 Calendar → schedule template link
- [x] `calendar-home.component.html` toolbar: add a "My weekly plan" link (→ `/my/my-schedule`), shown only when logged in (next to the My Schedule/Full Schedule mode toggle).
- [x] `calendar-home.component.spec.ts`: link renders when authed, absent when anonymous.

### 1.4 Schedule template page → notification settings link
- [x] `my-schedule.component.html`: add a "Notification settings" link (→ `/my/notification-preferences`) in the page header area.
- [x] `my-schedule.component.spec.ts`: link renders.

### 1.5 Gallery anchor fix
- [x] `home-page.component.html`: give the photo-carousel section `id="gallery"` so the existing About ▸ Gallery `/#gallery` fragment link actually lands somewhere.
- [x] `home-page.component.spec.ts`: anchor present.

### 1.6 Live hand-testing (Phase 1)
Fresh database via `knottyyoga_database_helper`, server + `ng serve` running, no extra data needed.
1. Logged out: the top menu reads **Get Started, About, Our Classes, Services, Upcoming Events, Memberships, Sign In**. **Our Classes** navigates straight to the weekly schedule page (no dropdown). **Services** navigates straight to the Browse Services page (no dropdown). **Memberships** opens the shop catalog, which lists exactly **Knotty Yoga Gold Membership**, **Knotty Yoga Gold Couple's Membership**, and **Knotty Yoga Gold Family Membership**.
2. Open **About** → **Gallery**: the page scrolls to the photo section of the home page.
3. Register/log in (menu **Sign In** → **Create one** flow), then open the account menu (your first name, far right): it lists **Profile, My Purchases, My Bookings, My Subscriptions, Sign Out** — no "Memberships", no "My Goals", no "My Classes".
4. Open **Your Calendar**: a **My weekly plan** link is visible and opens the My Schedule page; on that page a **Notification settings** link opens Notification Preferences.
5. Open **Our Classes**, then use the schedule page's **Browse all classes** link: the All Classes page opens and shows the **View the calendar** link, which opens the calendar.

---

## Phase 2 — Our Schedule page redo (the "Our Classes" landing page)

> Backend first (data layer additions to the calendar feed), then the shared-component extraction, then the page rebuild.

> **2.1 + 2.2 implemented 7/30/2026.** Linux docker gate green: full app suite **4620 tests, all passed** (floor 3500), plus a filtered run of `CalendarHelperTest` / `GetCalendarTest` / `SchedulingKeyValueTableTest` (149 tests) while iterating. No honuware files touched.

### 2.1 Backend — anonymous access states in the calendar feed
`business_logic/scheduling/calendar_helper.cpp` currently skips the access gate for `personId == 0`. Change: evaluate class access for anonymous viewers against an **empty** effective-permission/skill set, so anonymous items carry real `access` states and the membership deep-link fields.
- [x] For anonymous viewers, run the same `ClassAccessHelper`-based evaluation used for logged-in viewers with empty permission + skill sets. Classification: any unsatisfied requirement group containing a permission literal → `members_only` (populate `required_permission_id`/`required_permission_name` from the group's permission literal exactly as the logged-in path does); unsatisfied groups with only skill literals → `needs_skill` + `missing_skill_ids`. *(Implemented by deleting the `if (personId <= 0) return;` early-out in `AnnotateAccess` — the gate already degrades correctly for person 0: `CatalogHelper::GetEffectivePermissionIds` returns an empty vector and no `skill_level_assignments` row can name person 0. Anonymous therefore gets **byte-identical** classification logic to a logged-in non-member, including the existing skill-over-membership precedence, rather than a parallel code path.)*
- [x] Leave bookings/service-booking overlays and attendance state absent for anonymous (unchanged). *(`BuildBookedEventMap` and `AppendServiceBookingItems` keep their own `personId <= 0` guards; covered by a new `AnonymousStillCarriesNoBookingOverlay` test.)*
- [x] Tests (`calendar_helper_test.cpp`): anonymous viewer sees `members_only` + required-permission fields on a permission-gated class; `eligible` on an open class; `needs_skill` on a skill-gated class; logged-in behavior unchanged (regression case — the existing `EligibleWhenViewerPassesTheGate` / `MembersOnlyForNonMemberCarriesTheGatingPermission` / `NeedsSkillForViewerLackingThePrerequisiteSkill` / `SkillGatePrecedesMembershipGate` tests are untouched and still pass).
- [x] Endpoint-level coverage added in `get_calendar_test.cpp` (not in the original plan, but the old `AnonymousSeesDerivedOccurrencesWithoutAccessState` test asserted the behavior being replaced): renamed to `AnonymousSeesDerivedOccurrencesAsEligible`, plus a new `AnonymousSeesMembersOnlyAccessWithPermission` asserting the gating permission reaches an anonymous caller over the wire.

> ⚠️ **Live side effect from 2.1, already active:** the existing `/calendar` page changes for logged-out visitors as soon as this ships — gated class chips now render **locked** (`month-view.isLocked` / `calendar-event`), and clicking a `members_only` chip routes to `/shop?membership=<permission>` via `CalendarNavigationService`. That is the Phase-2 behavior we want ("Requires a membership" → Memberships page), just arriving on the calendar before the schedule page is rebuilt. Nothing is hidden from anonymous visitors: `CalendarService` defaults them to `full` mode, and the mode filter is the only place `access` removes rows.

### 2.2 Backend — predecessor note on `CalendarItem`
The old schedule page showed "Requires attending: {class}" from `/api/schedule/week`; the calendar feed lacks it. Parity addition:
- [x] Add `predecessor_class_name` (empty when none) to `CalendarItem`: during class-occurrence derivation, when the slot has `predecessor_class_schedule_slot_id`, resolve that slot's class name. *(New `CalendarHelper::PredecessorClassName` walks predecessor slot → impl → instance → class — same chain as `WeeklyScheduleHelper::ResolvePredecessor` — so cross-class prerequisites (SL-11) resolve. `AppendClassItems` memoizes it per slot id, so the four-table walk runs once per slot rather than once per derived occurrence. Added `TableHelpers::ClassSchedules` + `ClassScheduleSlots` members to `CalendarHelper`.)*
- [x] KVT converter + `scheduling_key_value_table_test.cpp` case.
- [x] `calendar_helper_test.cpp`: slot with a predecessor surfaces the name; slot without stays empty. *(Plus: the name repeats on **every** occurrence of the slot — the cache regression — and events carry an empty note.)*
- [x] `get_calendar_test.cpp` `ItemCarriesPredecessorClassName` — the field reaches the wire (`CalendarFixture` now returns its `slotId`).

### 2.3 Frontend — extract the calendar occurrence mechanics to `shared/`
> **2.3 – 2.5 implemented 7/30/2026.** Gates green: Angular full suite **2675 SUCCESS**, `ng build` clean, `npx tsc --noEmit` clean, `eslint` on the new/moved files reports no new problems; Linux docker C++ suite **4601 tests passed** (507 suites — down 19 tests / 2 suites, exactly the deleted `schedule_week` + `WeeklyScheduleHelper` + `ScheduleSlotView` KVT cases).

So the schedule page (and later the home feed / offerings tab) can render occurrences "like the calendar" without cross-page imports:
- [x] Move `pages/calendar/components/views/calendar-event/` → `shared/components/calendar-event/`, `pages/calendar/components/attendance-dialog/` → `shared/components/attendance-dialog/`, `pages/calendar/components/skill-requirement-dialog/` → `shared/components/skill-requirement-dialog/`, and `pages/calendar/calendar-navigation.service.ts` → `shared/services/calendar-navigation.service.ts` (selectors/class names unchanged — pure relocation; update all imports).
- [x] Move their `.spec.ts` files alongside and keep them green.
- [x] Extract the attendance-map builder (keyed `${slotId}-${occurrenceDateUs}`, currently inside `CalendarService`) into a small shared util so the schedule page can reuse it; `CalendarService` calls the util. Unit test for the util. *(→ `shared/utils/attendance-map.ts`: `attendanceKey`, `buildAttendanceMap`, `isAttending`.)*
- [x] `ui/src/app/shared/types/calendar-item.types.ts`: add `predecessor_class_name`; `ServerAccessNetwork.getCalendar` normalization updated if needed; `ServerAccess.mock.spec.ts` updated for the new field.

**Three deviations from "pure relocation", each forced by the move:**
1. **`CalendarNavigationService` no longer depends on `CalendarService`.** It used to call `calendarService.updateAttendance(...)` after a successful `setException` — an import from `shared/` into `pages/`, i.e. the wrong direction. It now publishes an `AttendanceChange` on a new `attendanceChanged$` observable; `CalendarService` subscribes in its constructor (third ctor param) and the redone schedule page subscribes in `ngOnInit`. One change now updates *every* surface showing that occurrence, which is what Phase 5's home feed will want too.
2. **`CalendarEvent` / `CalendarAttendance` / `CalendarEventTag` moved to `shared/types/calendar-event.types.ts`**, because the moved chip imports them. `pages/calendar/calendar.types.ts` re-exports all three (the app's established shim pattern), so its ~20 existing importers are untouched; only the grid model (`CalendarView`, `CalendarDay`, `CalendarData`, …) still lives there.
3. **New `shared/utils/calendar-event-mapper.ts`** (`calendarItemToEvent`, `itemDate`, `usToWallClockLocalDate`) — the feed-item → chip mapping and its two timezone rules were private to `CalendarService`; the schedule page needs the identical mapping, so it was extracted rather than copied. `CalendarService._toEvent` and its two public statics now delegate. Has its own spec.

### 2.4 Frontend — rebuild `OurScheduleComponent`
- [x] Data: `GET /api/calendar` for the selected week window (from `max(now, week start)` to week end) filtered to class kinds (`class`, `workshop`, `series` — standalone `event`s stay on Upcoming Events/home) + facility filter dropdown reused from the calendar toolbar pattern; when logged in, also `GET /api/me/upcoming_classes` for the same window to build the attendance map. *(The fetch window is padded ±1 day and the exact week bound applied client-side: class occurrence times are wall-clock-encoded-as-UTC, so an exact local-midnight window could clip an edge day in the viewer's timezone.)*
- [x] Layout: day-grouped list for the week (keeps the current page's day-section feel — OQ-3 follow-up ✅), each occurrence row shows class photo, name (→ `/classes/:id`), time + duration, room, instructor (+ substitute note), `predecessor_class_name` note, and a **calendar-style chip** (`app-calendar-event` states: attend toggle for plannable recurring items, Booked/Waitlisted, Sold out, "Sign-ups open {date}" + Remind me, Cancelled, lock for `members_only`/`needs_skill`).
- [x] Eligibility mode (OQ-3 ✅): logged-in defaults to **eligible classes only** with a "Full Schedule" toggle (same `members_only`/`needs_skill` hide-filter and labels as the calendar's mode toggle); logged-out always renders the full schedule as an advertisement.
- [x] Week navigation: **This week / Next week / following weeks** (past days of the current week are omitted — the feed clamps to now). *(Previous is disabled at "this week" — there is nothing behind now to show.)*
- [x] Logged-in extras: a "Plan your weekly schedule" link → `/my/my-schedule` in the page header.
- [x] Logged-out behavior: same list with time slots, no attend toggles; `members_only` rows show "Requires a membership" linking to the Memberships page (via the shared `CalendarNavigationService` behavior — same as the calendar's lock handling); page header gets "New here? Get started" (→ `/start`) and "Sign in" CTAs.
- [x] Carry the Phase-1 "Browse all classes" link (→ `/classes`) through into the redone page header — it is where All Classes lives now that it left the menu.
- [x] Remove the `getWeeklySchedule` usage from the page (the full surface removal is 2.5).
- [x] Specs: rewrite `our-schedule.component.spec.ts` — logged-out rows without toggles + membership link on gated class + full schedule always shown; logged-in defaults to eligible-only and the Full Schedule toggle reveals gated rows; attend toggle calls `setException` optimistically; week navigation; browse-all-classes + weekly-plan links; facility filter. *(23 cases. The attend path is asserted as "the chip delegates to the shared `CalendarNavigationService`" plus "an `attendanceChanged$` broadcast flips the row to *I'll be there*" — `setException` itself is now owned and tested by the shared service, so asserting it here would test the same call twice.)*

**One addition to the moved chip:** `app-calendar-event` gained an `@Input() compact = false`. The schedule row draws the class identity itself (photo, name link, time, room, instructor, predecessor), so repeating title/date/location inside the chip would be duplicated noise — compact mode renders only the state/action part (status badge, lock, sign-ups row, attendance row) and keeps the same click routing. It also spells the lock out as **"Requires a membership"** / **"Prerequisite needed"**, because on this page the lock IS the call to action rather than a decoration. Default `false` leaves the calendar's card byte-identical.

### 2.5 Remove the dead weekly-schedule surface (OQ-9 ✅ — option c)
Lands with (or immediately after) 2.4, so the endpoint never sits caller-less. Delete top-down: frontend consumer first, then the seam, then the backend.
- [x] Pre-check: confirm the redone page (2.4) is the only caller — search for any other reference to `getWeeklySchedule` / `/api/schedule/week` (app code, `ServerAccess` mock state, manual-testing-helper commands, scheduler jobs) before deleting. *(Repo-wide search: only the four seam files, the page, and the backend half. No test-helper command, no scheduler job.)*
- [x] Frontend: delete `getWeeklySchedule` from the `ServerAccess` interface / `ServerAccessNetwork` / proxy / mock, its `ServerAccess.mock.spec.ts` cases, and the now-unused `WeeklySchedule` / `ScheduleDay` / `ScheduleSlot` types in `shared/types/class.types.ts`.
- [x] Backend: delete the `/api/schedule/week` endpoint (`endpoints/schedule_week.{h,cpp}` + its endpoint test), its `web_app.cpp` registration and `endpoints/CMakeLists.txt` entries; delete `Scheduling::WeeklyScheduleHelper` (`business_logic/scheduling/`) + its test + CMakeLists entries; remove any KVT converters (and their test cases) used only by this surface. *(`ScheduleSlotViewToKeyValueTable` + its 2 test cases, and the `weekly_schedule_helper.h` include in `scheduling_key_value_table.h`.)*

### 2.6 Live hand-testing (Phase 2)
- [x] Steps written (below) and updated to match what shipped — awaiting your run against a live server.

Fresh database; use the test-helper app to add a second, membership-gated class so both states are visible (per its documented commands — create a class with a requirement group referencing the Gold Member permission and a Tue/Thu slot).
1. Logged out, open **Our Classes**: the week list shows **Knotty Yoga** on Monday and Wednesday at 6:00 PM with its facility and duration, with no attend controls and no **My Schedule / Full Schedule** toggle; the membership-gated class shows a **Requires a membership** badge that lands on the Memberships page when clicked; the header shows **New here? Get started** and **Sign in**, plus **Browse all classes** opening the All Classes page.
2. Register a fresh account (no membership) and reopen **Our Classes**: the toggle now appears, defaulted to **My Schedule**, and the list shows only **Knotty Yoga** (the open class); flip to **Full Schedule** and the membership-gated class appears with its **Requires a membership** badge. The header's Get started / Sign in pair is replaced by **Plan your weekly schedule**, which opens My Schedule.
3. Log in as a member who passes the gate (buy **Knotty Yoga Gold Membership** with the Square sandbox card via **Memberships**, or grant the permission via the test-helper), reopen **Our Classes**: both classes now show under **My Schedule**, and the Monday 6:00 PM **Knotty Yoga** row shows the **Tap to plan attendance** chip; tapping it and confirming **I'll be there** flips it to the checked "I'll be there" state; the same state is visible on **Your Calendar** for that Monday.
4. Use **Next week** and confirm the same weekly pattern renders for the following week; **Previous** is greyed out while you are on the current week. On the current week, only today and later days are listed.
5. Give the gated class's slot a predecessor via the test-helper (its Tue/Thu slot requires attending **Knotty Yoga**) and reload: that row shows **Requires attending: Knotty Yoga** under the instructor line.
6. With the gated class in a second facility, the **Facility** dropdown appears; picking one narrows the week to that facility's classes, and **All facilities** restores both.

---

## Phase 3 — Class detail page cleanup

> Frontend only. Preserves the workshop booking surface (Design Decision 3).

> **Implemented 8/1/2026.** Angular gate green: `class-detail.component.spec.ts` **46 SUCCESS** (+7 new cases), full suite **2715 SUCCESS**, `ng build` clean, `tsc --noEmit` clean, `eslint` on the page clean. No backend work, as planned.

### 3.1 Upcoming Sessions → calendar link (recurring classes)
- [x] `class-detail.component.html/.ts`: for `kind === 'recurring'`, replace the entire Upcoming Sessions list with a "See this class on the calendar" card/CTA → `/calendar` (plain unfiltered link — OQ-10 ✅). *(The whole section header goes with it: a recurring class shows a "When it runs" card explaining that recurring classes are just-show-up, so the "No upcoming sessions scheduled" empty state no longer appears for them either.)*
- [x] Weekly-pattern summary (day/time slots) stays if already shown elsewhere on the page; nothing else about the recurring layout changes. *(There is no weekly-pattern summary on this page today — nothing to preserve.)*

### 3.2 Workshops — capped list + calendar link
- [x] For `kind === 'workshop'`, cap the rendered occurrence list to the next **5** (OQ-4 ✅) and add "See all dates on the calendar" → `/calendar` below it. The cap is a template slice — the bookable click-through (`materializeClassOccurrence` → `/shop/event/:sessionId`) is unchanged. *(A "Showing the next 5 dates." note appears beside the link only when the cap actually hid something.)*

### 3.3 Series — unchanged runs section + calendar link
- [x] Series keep the runs/buy section as-is; add the same "See this class on the calendar" link for consistency. *(A series' occurrence list is naturally bounded by its run, so it is left uncapped.)*

### 3.4 Specs
- [x] `class-detail.component.spec.ts`: recurring shows the calendar link and no session cards; workshop shows ≤5 session cards + the link and still books an occurrence on click; series shows runs + link. *(Plus: a workshop with fewer than 5 dates shows them all with no cap note; a class with no `kind` keeps its list and gets the generic link; a recurring class with zero occurrences still offers the calendar; and the capped list's **5th** card books the **5th** occurrence — proving the slice is display-only.)*

### 3.5 Live hand-testing (Phase 3)
- [x] Steps written (below) — awaiting your run against a live server.

1. From **Our Classes**, open **Knotty Yoga**: instead of a long Upcoming Sessions list the page ends with a **When it runs** card explaining the class is just-show-up, and its **See this class on the calendar** button opens the calendar.
2. Create a workshop class with more than five future occurrences via the test-helper app; open its class page from **Our Classes** → **Browse all classes**: exactly 5 dated session cards render, followed by **See all dates on the calendar** and the note **Showing the next 5 dates.** Clicking the *fifth* card leads to the event booking page for that fifth date (not the first).
3. Give that workshop only two future occurrences: both cards render, and the cap note is gone while the calendar link stays.
4. Open a series class page: the **Series Runs** section is unchanged (with its Buy full series / Join prorated buttons) and **See this class on the calendar** sits below the session list.

### 3.6 Rename the landing page "Our Schedule" → "Our Classes" — **DONE (8/2/2026)**
The top-level menu item said **Our Classes** but opened a page titled **Our Schedule**. Renamed the page to match the menu.
- [x] `pages/public/our-schedule/` → `pages/public/our-classes/`, files `our-schedule.component.*` → `our-classes.component.*`, class `OurScheduleComponent` → `OurClassesComponent`, selector `app-our-schedule` → `app-our-classes`, and the page `<h1>` → **Our Classes**.
- [x] **Second collision resolved:** the full catalog at `/classes` was *also* titled "Our Classes". Its `<h1>` is now **All Classes**, which is what both links into it already say (the schedule page's "Browse all classes" button and the class-detail back-link).
- [x] ~~Route path **stays `/schedule`**~~ — **superseded by OQ-13 (answered 8/2/2026, see 3.7).** The URLs were reshuffled so the menu item and the URL agree.
- [x] Stale `"Our Schedule"` references in code comments across `shared/`, `pages/calendar/` and `pages/manage/` updated to the new name.
- [x] Gate: full suite **2715 SUCCESS**, `ng build` clean, `tsc --noEmit` clean, `eslint` clean on the touched files.

### 3.7 URL reshuffle so the menu item and the URL agree (OQ-13) — **DONE (8/2/2026)**
> **Mason (OQ-13):** *"This sounds like a good change."*
>
> (You flagged OQ-12 and OQ-13 as "shouldn't involve any work". OQ-12 indeed needed none — the defaults were already shipped. **OQ-13 did**: it is a route change plus every link that pointed at the old paths. Done below.)

After 3.6 the weekly landing page was titled **Our Classes** but lived at `/schedule`, while the catalog titled **All Classes** owned `/classes`. Now the paths match the titles.
- [x] `pages/public/public.routes.ts`: `/classes` → `OurClassesComponent`; `/classes/all` → `ClassInfoComponent`; `/classes/:id` unchanged. **`classes/all` is declared before `classes/:id`** — the other order parses the catalog as a class id.
- [x] `/schedule` → `redirectTo: 'classes'` (`pathMatch: 'full'`), so existing links and bookmarks still resolve.
- [x] Menu: `mockHeaderResponse.ts` **Our Classes** now points at `/classes`.
- [x] Every link into the catalog re-pointed at `/classes/all`: the Our Classes header's **Browse all classes** button, the class-detail **All Classes** back-link, the series-booking "Browse classes" fallback, and the attendance-history empty-state "Browse the catalog" link.
- [x] Specs updated: `mockHeaderResponse.spec.ts` (menu target + the "no stray /classes entries" assertion, now "exactly one"), `header.service.spec.ts`, `our-classes.component.spec.ts` (browse-link href), `class-detail.component.spec.ts` (back-link href).

---

## Phase 3B — Series (and workshop) discoverability

> **The problem (Mason, 8/2/2026):** a series is effectively unreachable. Today you either scroll the **Our Classes** week list to a week where an instance of the series happens and click that occurrence, or you go **Our Classes → Browse all classes → the series class → Series Runs**. Nothing surfaces "these series are starting soon" anywhere a visitor or member would look.
>
> The fix is one new public feed plus one shared, expandable card reused on three surfaces: **Upcoming Events**, the top of **Our Classes**, and the **home page**.
>
> **Sequencing note — this depends on Phase 4.** The card's Book CTA routes to `/shop/series/:classInstanceId`, which today only works when a `SeriesRun` arrives via `history.state`. That is fixed by **4.1** (`GET /api/class_series_run/<classInstanceId>`) + **4.4** (self-loading `SeriesBookingComponent`). **Recommended order: do 4.1 and 4.4 first**, then 3B. If you'd rather ship 3B first, the card's CTA has to point at the class page instead and get re-pointed later — more work overall, so the plan assumes 4.1/4.4 land first.

> **Implemented 8/2/2026.** Gates green: Linux docker C++ suite **4646 tests passed** (up 15 from 4631), Angular full suite **2750 SUCCESS**, `ng build` clean, `tsc --noEmit` clean, no new lint.
>
> **OQ-11 follow-on (same day).** Gates green again after widening the feed to workshops (3B.8) and the OQ-13 URL reshuffle (3.7): Linux docker C++ suite **4664 tests passed**, Angular full suite **2765 SUCCESS**, `ng build` clean, `tsc --noEmit` clean on both `tsconfig.app.json` and `tsconfig.spec.json`, `ng lint` unchanged from its pre-existing baseline (no finding in any file touched here).
>
> **Phase 4.1 and 4.4 were implemented as part of this**, as the sequencing note called for. Without them the card's Book button dead-ends, which would have missed the whole point of the request — so they are done and checked off in Phase 4 below.
>
> **Open questions — Mason answered 8/2/2026.** **OQ-11** → *"Let's do workshops too."* → the feed was widened to series **+ workshops**; see **3B.8** for what changed (the endpoint, the seam method and the card were all renamed off "series"). **OQ-12** → *"These defaults are fine."* → no work; expanded on Upcoming Events, collapsed on Our Classes, as shipped. **OQ-13** → *"This sounds like a good change."* → the URLs were reshuffled; see **3.7**.

### 3B.1 Backend — a public "upcoming series" feed *(became the offerings feed in 3B.8)*
`ClassSeriesHelper` can only answer "runs for THIS class" (`GetSeriesRunsForClass`). Every surface below needs "runs across all classes, soonest first", and `SeriesRun` doesn't currently carry enough to render a cross-class card (no class name/photo).
- [x] `Scheduling::SeriesInfo` (or a new `UpcomingSeriesInfo`) gains `className`, `classHasPhoto`, and the signup-window state (`signupWindowOpen` / `signupOpensAtUs`), resolved the same way `CalendarHelper::ComputeSignupWindow` does — the card must not offer Book before sign-ups open. *(Also `inProgress`, so the card can say "Already under way" and offer the prorated CTA. The window is measured against the run's START, mirroring how the calendar measures an occurrence's.)*
- [x] `ClassSeriesHelper::GetUpcomingSeriesRuns(...)` — every **active** `kind='series'` class's runs that haven't ended, whether they start later or are already under way. Ordered by `valid_from_us`, then class name. Personalized pricing when `personId > 0`, public otherwise. *(Named `GetUpcomingSeriesRunsAcrossClasses` — the per-class `GetUpcomingSeriesRuns` already existed. Both, plus the new single-run read, now share one private `BuildSeriesInfo`, so pricing, occurrence counting and the annotation resolve one way.)*

> 🐞 **Bug found on the first live run (8/2/2026) and fixed.** I originally excluded in-progress runs unless `prorated_signups_allowed` — reasoning that a started run has no way in otherwise. **That was wrong:** `BookFullSeries` has no start-date guard, and the class-detail page happily sells a started run ("Buy full series" is offered whenever a price resolves; `prorated_signups_allowed` only gates the *extra* prorated CTA). Mason's first real series was exactly that shape — started, prorated off — so both feeds came back empty while the class page showed it.
>
> The filter is gone: the feed now lists every not-yet-ended run. `in_progress` + `prorated_signups_allowed` still ride along so the card picks the CTA — **"Join from today — prorated"** only when the studio allows it, **"Book the series"** otherwise, with an "Already under way" note either way. The backend test that encoded the wrong rule was rewritten, and an explicit *ended runs are excluded* test added on both sides.
>
> **Worth a decision later:** buying a started run "in full" charges for sessions that already happened. That is pre-existing class-page behavior, not something 3B introduced — but if you'd rather started runs were prorated-or-nothing, the clean fix is in `BookFullSeries` (reject a started run) rather than in the feed, so the class page and the card stay consistent.
- [x] Access-gate the list the same way the calendar does: a series the viewer can't take still appears (it's marketing) but carries the `members_only` / `needs_skill` annotation. **Anonymous viewers get real states.** *(This forced a good cleanup: the classification was private to `CalendarHelper::AnnotateAccess`. It moved to `ClassAccessHelper::Annotate` + a `ClassAccessAnnotation` struct, so the calendar and the series feed classify through one path; `kCalendarAccess*` are now aliases of the shared constants, so no call site churned.)*
- [x] New thin endpoint `GET /api/upcoming_series_runs?to_us&limit` (public / anonymous-safe) → `{ items: [...] }`; registered in `web_app.cpp` + `endpoints/CMakeLists.txt`. *(Superseded by **3B.8**: renamed to `GET /api/upcoming_offerings` when workshops joined the feed; the old file was deleted.)*
- [x] KVT converter + `scheduling_key_value_table_test.cpp` case (2 cases: populated + defaults).
- [x] Tests: helper (`GetRunByClassInstanceId` found/missing/wrong-kind, plus ended-run and inactive-class exclusion) + endpoint `get_upcoming_series_runs_test.cpp` (empty, cross-class ordering, in-progress only when prorated is allowed, non-series ignored, gated-but-listed, limit + to_us). A shared `series_run_test_fixture.h` builds a run in one call. *(The endpoint test became `get_upcoming_offerings_test.cpp` in **3B.8**; the fixture gained a `BuildWorkshop` builder.)*

### 3B.2 Frontend — `ServerAccess` seam
- [x] `getUpcomingSeriesRuns(toUs?, limit?)` *(renamed to `getUpcomingOfferings` in **3B.8**)* **and** `getSeriesRunByInstance(id)` across interface / network / proxy / mock; `SeriesRun` gains `class_name`, `has_photo`, `in_progress`, `access`, `required_permission_id/name`, `missing_skill_ids`, `signup_window_open`, `signup_opens_at_us`. One `normalizeSeriesRun` in the network layer coerces the string booleans + comma-joined skill ids for all three series reads.
- [x] `ServerAccess.mock.spec.ts`: 6 new cases (single-run found/404, feed ordering, under-way run dropped when prorated is forbidden, limit + to_us, gate carried). The mock is seeded with a second, gated run so local mode exercises the lock path.

### 3B.3 Frontend — the shared expandable series card
- [x] New `shared/components/series-highlight/` *(renamed `offering-highlight/` in **3B.8**)* — one component, three hosts. Collapsed it is a single summary row; expanded it lists each run with the class photo, class name + run name, date window, session count, per-session and total price, and the right CTA:
  - eligible + sign-ups open → **Book the series** → `/shop/series/:classInstanceId`;
  - eligible + in progress → **Join from today — prorated**;
  - sign-ups not open yet → "Sign-ups open {date}";
  - `members_only` / `needs_skill` → the calendar's lock treatment (→ Memberships page / skill dialog).
- [x] Inputs: `runs` *(now `offerings`)*, `heading`, `collapsedByDefault`. Hidden entirely when `runs` is empty. *(Once the viewer toggles it, their choice sticks — a re-fetch won't slam it shut on them.)*
- [x] Spec: 16 cases — collapse/expand, every CTA state, hidden-when-empty, UTC window formatting, and lock-beats-closed-window precedence. *(One trap worth remembering: `MatDialogModule` provides `MatDialog` at **module** scope, so a standalone component importing `SharedModule` gets it from its own injector — the spec must spy on the component's instance, not `TestBed.inject(MatDialog)`.)*
- [x] Deviation from the plan: **no "Remind me" button** on a not-yet-open run. That action is slot+occurrence keyed (`requestSignupReminder(slotId, occurrenceDateUs)`) and a series run has neither — it is a date window, not an occurrence. The card states the open date instead; wiring reminders for whole runs would need a new backend key and is worth its own item if you want it.

### 3B.4 Upcoming Events page — series section
- [x] `upcoming-events.component`: fetches `getUpcomingSeriesRuns()` alongside the event list and renders `app-series-highlight` **above** the one-off events, headed "Series starting soon", expanded by default. *(**3B.8**: now `getUpcomingOfferings()` → `app-offering-highlight`, headed "Series & workshops coming up", filtered through `visibleOfferings`.)*
- [x] Spec: renders above the events list (asserted by document order), absent when empty, and the two fetches are independent in **both** directions — a series failure leaves the events list intact, and an events failure still shows the series card.

### 3B.5 Our Classes page — series card at the top
- [x] `our-classes.component`: renders `app-series-highlight` between the page header and the week list, **collapsed by default**, headed "Series & multi-week programs". *(**3B.8**: now `app-offering-highlight`, headed "Series & workshops".)*
- [x] The card is week-independent — fetched once in `ngOnInit` (not in `load()`), so paging weeks never refetches it, and it is untouched by the facility filter and the eligible-only toggle.
- [x] Spec: above the week list, expands on demand, hidden when empty, survives week navigation without a refetch, survives filters that hide every class, and the week list still renders when the series fetch fails.

### 3B.6 Home page — series card
> Lands **with Phase 5** (the home page redesign) rather than before it, so the section is placed once rather than built and then moved.
- [ ] Logged out: `app-offering-highlight` collapsed, under the membership tiers — a series is a bigger commitment than a drop-in, so it sits below the tier pitch.
- [ ] Logged in: collapsed, alongside "Events you could sign up for"; suppress offerings the viewer has already booked (same exclusion the events strip uses — workshop dates can reuse the `event_session_id` key **3B.8** added).
- [ ] Add to the Phase 5.4 / 5.5 spec + hand-test lists rather than duplicating them here.

### 3B.7 Live hand-testing (Phase 3B)
- [x] Steps written (below) — awaiting your run against a live server.

Fresh database; via the test-helper app create two series classes — one starting next month, one already in progress with `prorated_signups_allowed` — plus a third gated behind the Gold Member permission.
1. Logged out, open **Upcoming Events**: a **Series starting soon** section sits above the one-off events, expanded, listing both open series with their date windows, session counts and prices; the gated one shows **Requires a membership** and lands on the Memberships page.
2. Click **Book the series** on the future one: it goes straight to the series booking page with the run's dates and price — *without* having visited the class page first.
3. The in-progress one offers **Join from today — prorated**; that CTA opens the same page in prorated mode.
4. Open **Our Classes**: a collapsed **Series & multi-week programs** card sits under the page header; expanding it shows the same three runs. Page to **Next week** — the card is unchanged and no refetch happens. Flip **Full Schedule** / change the **Facility** filter — the card is unaffected.
5. Log in as a member who passes the gate: the third series now shows a Book CTA instead of the membership lock.
6. With no series in the system at all, neither page shows the card (no empty heading, no stray whitespace).

### 3B.8 OQ-11 — widen the feed to workshops too — **DONE (8/2/2026)**
> **Mason (OQ-11):** *"Let's do workshops too."*

A recurring class is just-show-up and belongs in the weekly grid. The two kinds that need **buying** — a series run and a workshop occurrence — were equally hard to find, so the series-only feed became one **offerings** feed carrying both. Everything named "series" on this path was renamed; the series behavior is unchanged.

**Backend**
- [x] New `business_logic/scheduling/upcoming_offerings_helper.{h,cpp}` — `UpcomingOffering` + `UpcomingOfferingsHelper::GetUpcomingOfferings(txn, personId, asOfUs, toUs = 0, limit = 0)`. `kind` is `"series"` or `"workshop"` and decides which half of the struct is meaningful; **every field is emitted either way** (zeroed for the other half) so the client sees one stable shape.
  - *Series half:* `class_instance_id`, `run_name`, `occurrence_count`, `total_cents`, `in_progress`, `prorated_signups_allowed` — mapped straight from `ClassSeriesHelper::GetUpcomingSeriesRunsAcrossClasses`, so the two feeds can never drift.
  - *Workshop half:* `class_schedule_slot_id`, `occurrence_date_us`, `event_session_id`. Occurrences are **derived** per date from the active instance's implementation slots (`ClassScheduleHelper::GetDerivedSessionsForRange`), priced via `CatalogHelper::ResolveBestPriceForPerson` and gated via `ClassAccessHelper::Annotate`, with the sign-up window computed per occurrence from `SignupReminderHelper::ResolveBestAdvanceDaysForPerson`.
  - Cancelled occurrences are skipped. A workshop class with no run (hence no product) is skipped.
- [x] **`event_session_id` is the de-duplication key.** A materialized workshop occurrence is *also* an `event_sessions` row, so it already appears on Upcoming Events as a normal bookable event. The feed reports the persisted id (0 while purely derived) so a host can drop what it is already listing — and so the client can skip the materialize round-trip when booking.
- [x] **Default 120-day horizon** (`kDefaultWorkshopHorizonDays`) when the caller gives no `to_us`: a series is bounded by its own run window, but workshop derivation is open-ended and would otherwise walk forward forever. `to_us` overrides it; series are never clipped by it.
- [x] Derivation walks whole UTC days from `midnight(asOfUs)`, so the first day can yield an occurrence that **already started** — those are dropped (`startUs < asOfUs`), or "10am today" would still read as upcoming at 6pm.
- [x] **At most 3 dates per workshop class** (`kMaxOccurrencesPerWorkshopClass`). A workshop modeled with a weekly slot derives ~18 dates over the horizon, and one class must not flood a cross-class advertisement. The soonest three are kept; the class page (which caps at 5 and says so) is where you go for every date.
- [x] Endpoint renamed: `GET /api/upcoming_series_runs` → **`GET /api/upcoming_offerings?to_us&limit`** (public / anonymous-safe) → `{ items: [...] }`. `get_upcoming_series_runs.*` deleted; `web_app.cpp` anchor + `endpoints/CMakeLists.txt` re-pointed.
- [x] KVT converters `UpcomingOfferingToKeyValueTable` / `...ToKeyValueTableArray` + 5 `scheduling_key_value_table_test.cpp` cases (series half, workshop half, access annotation, closed window, array order).
- [x] Tests: 14 helper cases (`upcoming_offerings_helper_test.cpp` — workshop listed with pricing, recurring excluded, cancelled skipped, materialized carries its session id, no-instance skipped, members-only still listed, closed sign-up window, past-today occurrence dropped, series+workshop merge order, limit, ended run excluded, in-progress run kept, per-class cap keeps the soonest three, default horizon applied + narrowed by `to_us`) + 5 endpoint cases (`get_upcoming_offerings_test.cpp` — both kinds for anonymous, soonest-first, limit, `to_us`, gate carried). `series_run_test_fixture.h` gained a `BuildWorkshop` builder.
- [x] 🐞 **Fixture trap found while writing these:** `CatalogHelper` resolves every price against **one** active price schedule (lowest id wins). A test that built two priced things created two schedules, and whichever lost silently priced at **0**. Both builders now go through a shared `EnsurePriceSchedule` that reuses the existing one.

**Frontend**
- [x] New `shared/types/offering.types.ts` (`UpcomingOffering`, `OfferingKind`). `SeriesRun` is unchanged — the per-class list and the single-run read still use it.
- [x] Seam: `getUpcomingSeriesRuns` → **`getUpcomingOfferings(toUs?, limit?)`** across interface / network / proxy / mock, with a new `normalizeOffering` in the network layer.
- [x] `shared/components/series-highlight/` → **`shared/components/offering-highlight/`** (`app-offering-highlight`, input `offerings`). Each row now carries a **Series** / **Workshop** chip; a workshop shows its date + duration + single-occurrence price where a series shows its window + session count + run total (with the per-session breakdown).
- [x] Workshop CTA is **"Book this date"** → materialize the slot+date into an `event_session`, then hand off to `/shop/event/:id` — the same path the class-detail page uses, so every payment option comes along. When the feed already reported an `event_session_id` the round-trip is skipped, and a signed-out visitor is sent to `/login` first. A failed materialize shows an inline message instead of navigating.
- [x] The collapsed summary counts both kinds ("2 series, 1 workshop") and names each class **once**, so the same workshop on three dates doesn't repeat.
- [x] **Upcoming Events** filters the card through `visibleOfferings`, dropping any offering whose `event_session_id` is already in the events list below — no date advertised twice on one page. Heading is now "Series & workshops coming up".
- [x] **Our Classes** heading is now "Series & workshops"; still collapsed by default and still week-independent.
- [x] Specs: 23 cases on `offering-highlight` (both kinds rendered, kind chip, count/summary dedupe, distinct rows per workshop date, every CTA state incl. the three workshop-booking paths, locks, formatting), plus updated host specs (Upcoming Events gained "lists workshop dates" and "drops an offering the events list is already showing"; Our Classes gained "lists workshop dates in the card") and 8 mock-seam cases (merge order, series mapping, workshop keys, under-way run kept, ended run dropped, past occurrence dropped, limit + `to_us`, gate carried). The mock is seeded with two dates of one workshop.

**Live hand-testing (OQ-11 — workshops in the feed).** Fresh database, on top of the Phase 3B steps in 3B.7 above.
1. Via the test-helper app create a **workshop** class ("Handstand Intensive") with a weekly slot and a per-occurrence price, and leave every date underived (nobody has booked one).
2. Logged out, open **Upcoming Events**: the **Series & workshops coming up** card now lists the workshop's next dates, each tagged **Workshop**, showing the date, the length in minutes and the single-date price — interleaved with the series runs by time, soonest first.
3. Click **Book this date** while logged out: you land on the sign-in page. Sign in as a member and click it again: you land on the event booking page **for that exact date**, with the workshop's price.
4. Go back to **Upcoming Events**: the date you just booked is now in the one-off events list below — and it is **gone** from the card, not shown twice.
5. Open **Our Classes**: the collapsed card's count reads e.g. **"2 series, 3 workshops"** and the summary names *Handstand Intensive* **once**, not once per date. Expand it — each date is its own row.
6. Cancel one of the workshop dates via the test-helper app: that row disappears from both cards while the other dates stay.
7. Give the workshop product a booking window of, say, 7 days: dates further out than that show **Sign-ups open {date}** instead of a Book button.
8. Gate the workshop behind Gold Member: every date shows **Requires a membership** and lands on the Memberships page.


---

## Phase 4 — Series & workshop sign-up discoverability

> Backend first: make a series run addressable by `class_instance_id`; then wire Book CTAs everywhere the Overview asks for (profile tab, calendar, class page — the class page already books).

### 4.1 Backend — `GET /api/class_series_run/<int:classInstanceId>` (public) — **DONE (8/2/2026, with Phase 3B)**
- [x] `business_logic/scheduling/`: `ClassSeriesHelper` gains `GetRunByClassInstanceId(Transaction&, classInstanceId, personId, asOfUs)` returning the same run shape as the per-class reader (both now share the private `BuildSeriesInfo`, so pricing and the access annotation resolve one way).
- [x] New thin endpoint `endpoints/get_class_series_run.{h,cpp}` → `{ run: {...} }`, 404 when the instance doesn't exist or isn't a series run; registered in `web_app.cpp` + `endpoints/CMakeLists.txt`.
- [x] Tests: helper test (found / wrong-kind / missing) + endpoint test (`get_class_series_run_test.cpp`: 200 shape for anonymous, 404 unknown, 404 non-series instance, membership gate carried).

### 4.2 Backend — `class_instance_id` on `UpcomingSignupOffering`
- [ ] `SignupReminderHelper::GetUpcomingSignupOfferings` (Phase 11 code): resolve and carry `class_instance_id` for each offering (series and workshop rows both have an owning instance).
- [ ] KVT converter + `scheduling_key_value_table_test.cpp` (or the Phase 11 KVT test file) updated; `get_my_upcoming_signup_offerings_test.cpp` asserts the field.

### 4.3 Frontend — `ServerAccess` additions
- [ ] `getSeriesRunByInstance(classInstanceId)` → `GET /api/class_series_run/:classInstanceId` across interface / network / proxy / mock; `UpcomingSignupOffering` type gains `class_instance_id` (+ network numeric normalization).
- [ ] `ServerAccess.mock.spec.ts`: new-method cases (found/404/logged-out allowed) + offering field coverage.

### 4.4 Frontend — self-sufficient `SeriesBookingComponent` — **DONE (8/2/2026, with Phase 3B)**
- [x] When `history.state.run` is absent, fetch via `getSeriesRunByInstance(classInstanceId)` instead of dead-ending in `'missing'` (keep the error state for a real 404). The `prorated` flag still comes from router state when present, so the card's prorated CTA survives the self-load.
- [x] `series-booking.component.spec.ts`: state-provided path unchanged (and asserted **not** to refetch); state-missing path fetches from the route id and renders; 404 shows the missing state; neither state nor route id still shows missing.

> **4.3 note:** the `getSeriesRunByInstance` half of 4.3 shipped with 3B (seam + mock + mock-spec cases). What's left of 4.3 is the `class_instance_id` field on `UpcomingSignupOffering`, which belongs with 4.2.

### 4.5 Frontend — Book CTAs on the Workshops & Series tab
- [ ] `upcoming-offerings.component`: when `window_open`, replace the inert "Sign-ups are open" text with a **Book** button — `kind === 'workshop'` → `materializeClassOccurrence(slot, occurrence)` then `/shop/event/:sessionId` (reuse the shared `CalendarNavigationService` flow); `kind === 'series'` → `/shop/series/:classInstanceId`.
- [ ] `upcoming-offerings.component.spec.ts`: workshop CTA materializes + navigates; series CTA navigates with the instance id; closed-window rows still show the reminder button.

### 4.6 Frontend — calendar click routing books open offerings
- [ ] Shared `CalendarNavigationService`: an `eligible` + `signup_window_open` **series** occurrence → `/shop/series/:classInstanceId` (was: class page); an `eligible` + open **workshop** occurrence → materialize + `/shop/event/:sessionId`; all other cases keep today's routing (class page / lock handling / reminder).
- [ ] `calendar-navigation.service.spec.ts`: the two new routes + unchanged fallbacks (members-only, needs-skill, not-open, anonymous).

### 4.7 Live hand-testing (Phase 4)
Seed a series run and a workshop occurrence with open sign-ups via the test-helper app (series with a future run window; workshop with a dated occurrence).
1. Logged in, open the account dashboard card **Workshops & Series**: the open-window series row shows **Book**; clicking it lands directly on the series booking page with the run's dates and price (no prior visit to the class page needed), and payment with the Square sandbox card completes.
2. The open-window workshop row's **Book** lands on the event booking page for that date; booking completes and the item appears under **My Bookings**.
3. On **Your Calendar**, click the same series occurrence: it goes straight to the series booking page; click the workshop occurrence: it goes straight to its event booking page.

---

## Phase 5 — Home page redesign

> No backend changes expected — every section rides existing endpoints. Frontend: shared tier-card extraction first, then the page.

### 5.1 Frontend — shared membership tier cards
- [ ] Extract the inline tier-card markup from `subscription-catalog.component.html` into `shared/components/membership-tier-cards/` (input: `CatalogProduct[]`; output/behavior: Subscribe → `/shop/subscribe/:productId`). `SubscriptionCatalogComponent` consumes it.
- [ ] Specs: new `membership-tier-cards.component.spec.ts` (renders tiers, monthly price format, subscribe navigation) + updated `subscription-catalog.component.spec.ts`.

### 5.2 Frontend — home page sections (both auth states)
- [ ] Keep the photo carousel (with the Phase 1 `id="gallery"`) and the hero marketing block.
- [ ] **Getting Started CTA banner** → `/start` (both states; copy differs slightly when logged in).

### 5.3 Frontend — logged-out sections
- [ ] **Featured events strip**: up to 4 cards from `getVisibleEventSessions('home_page')` (name, time, facility, price, Book Now → `/shop/event/:id`) + "View All Events" → `/events`; empty state unchanged. Logged-in visitors get the personalized event sections below instead — the curated `home_page` placement stays a visitor-marketing surface.
- [ ] **Membership tiers section**: `getCatalogProducts()` filtered to `kind === 'subscription'` rendered via the shared tier cards (OQ-8 ✅), headed with short marketing copy; Subscribe click follows the existing auth-guarded flow (redirects to login).

### 5.4 Frontend — logged-in sections
- [ ] **Your upcoming events**: `getMyBookings('upcoming')`, first 3 cards (event name, date, facility, status incl. waitlisted) + "All my bookings" → `/my/events`; section hidden when empty.
- [ ] **Events you could sign up for** (OQ-5 ✅): `getVisibleEventSessions('upcoming')` — the same list the Upcoming Events page shows (ad-hoc events plus materialized workshop/series occurrences) — minus sessions already in the user's upcoming bookings, capped at the **4 soonest**; cards Book Now → `/shop/event/:id`; section hidden when empty.
- [ ] **Next 4 days at the studio**: embed `app-upcoming-classes` with a new `@Input() daysAhead = 28` set to 4 and `[embedded]="true"` (attend / can't-make-it toggles come with the component); headline links "Plan your weekly schedule" → `/my/my-schedule`.
- [ ] `upcoming-classes.component`: add the `daysAhead` input (window = now → now + daysAhead days) + spec case.
- [ ] **Membership tiers when no membership**: on `getSubscriptions()` returning no active subscription and no shared entitlement (the profile-hub rule — OQ-6 ✅), show the same tier section with "Become a member" copy; hidden for members.
- [ ] Auth branching on `AuthService.authData$` (subscribe/refresh on auth change, like the header does).
- [ ] Specs: `home-page.component.spec.ts` rewritten — logged-out: hero + featured strip + tiers + no personalized sections; logged-in with membership: bookings/could-sign-up/4-day feed present, tiers + featured strip absent; logged-in without membership: tiers present; could-sign-up caps at the 4 soonest and excludes booked sessions. Mock `SquarePaymentService` is not needed (no embedded payment control), but keep the rule in mind if a card control ever lands here.

### 5.5 Live hand-testing (Phase 5)
Fresh database. Via **Manage** → **Events** → **Create Event**, create two future **Intro Workshop** sessions, both visible on the upcoming list and at least one flagged for the home page placement (fields per the create-event form labels) — or use the test-helper app equivalents.
1. Logged out, open the home page: carousel + hero render; the **featured events** strip shows the Intro Workshop card(s) with **Book Now**; the **membership tiers** section shows the three Gold memberships with **Subscribe** buttons; a **Get Started** banner is present. Clicking **Subscribe** routes through **Sign In**.
2. Log in as a fresh account (no membership): the curated featured strip is replaced by **Events you could sign up for** listing both Intro Workshop sessions (soonest first); **Next 4 days at the studio** appears (empty-state text until you're eligible for a class); the tier section stays visible.
3. Buy **Knotty Yoga Gold Membership** (Square sandbox card). Back on the home page: the tier section is gone; the **Next 4 days** feed lists Monday/Wednesday **Knotty Yoga** rows with **I'll be there / I can't make it** controls that flip state; **Plan your weekly schedule** opens My Schedule.
4. Book one Intro Workshop session; the home page now shows it under **Your upcoming events**, and it disappears from **Events you could sign up for**.

---

## Phase 6 — Getting Started page

### 6.1 Frontend — page + route
- [ ] New `pages/public/getting-started/getting-started.component.*` (separate template/styles per convention); `public.routes.ts`: `start` → `GettingStartedComponent` (menu already points there).
- [ ] Content — numbered step cards with CTA buttons (draft copy ships now; text gets revised together after it renders — OQ-7 ✅):
  1. **Try an intro workshop** — what to expect at your first visit; CTA → Upcoming Events.
  2. **Create your account** — CTA → Register (hidden when logged in).
  3. **Pick your membership** — what each tier includes; CTA → Memberships.
  4. **Plan your weekly schedule** — explains the attendance template ("tell us which classes you normally attend — it's a plan, not a booking"); CTA → My Schedule (login-aware).
  5. **Show up and check in** — membership classes need no advance booking; staff checks you in at the door.
  6. **Workshops & series** — where to find and book them (calendar, class pages, your Workshops & Series tab).
  7. **Stay in the loop** — weekly digest, sign-up reminders, and the iCal feed under Notification Preferences.
- [ ] Auth-aware rendering (step 2 hidden when logged in; step 4/7 CTAs go to login first when logged out — standard guard behavior).
- [ ] `getting-started.component.spec.ts`: steps render; register step hidden when authed; CTAs navigate.

### 6.2 Live hand-testing (Phase 6)
1. Logged out, open **Get Started** from the menu: the step list renders with working CTAs — **Try an intro workshop** opens Upcoming Events, **Create your account** opens registration, **Pick your membership** opens the Memberships page.
2. Log in and reopen **Get Started**: the create-account step is gone; **Plan your weekly schedule** opens My Schedule directly.

---

## Cross-phase test summary

- **Backend test files touched**: `calendar_helper_test.cpp` (2.1, 2.2), `scheduling_key_value_table_test.cpp` (2.2, 3B.8, 4.2), new `get_class_series_run_test.cpp` + series-helper tests (4.1), new `upcoming_offerings_helper_test.cpp` + `get_upcoming_offerings_test.cpp` (3B.8), `get_my_upcoming_signup_offerings_test.cpp` (4.2); **deleted** with their production code: the `schedule_week` endpoint test + `WeeklyScheduleHelper` test (2.5), and `get_upcoming_series_runs_test.cpp` (superseded by the offerings feed, 3B.8).
- **Frontend spec files touched**: header/service + mobile menu (1.1), `class-info` (1.2), `calendar-home` (1.3), `my-schedule` (1.4), `home-page` (1.5, 5.4), relocated calendar chip/dialog/navigation specs + new attendance-map util spec (2.3), `our-schedule` rewrite (2.4), `class-detail` (3.4), `ServerAccess.mock.spec.ts` (2.3, 2.5 `getWeeklySchedule` removal, 4.3), `series-booking` (4.4), `upcoming-offerings` (4.5), `calendar-navigation.service` (4.6), new `offering-highlight` replacing `series-highlight` + `offering.types` (3B.8), `public.routes` link/route churn across `mockHeaderResponse`, `header.service`, `class-detail`, `our-classes` (3.7), new `membership-tier-cards` + `subscription-catalog` (5.1), `upcoming-classes` (5.4), new `getting-started` (6.1).
- Angular gates per phase: affected specs green + `ng build` clean. C++ builds/tests run by Mason (Linux docker clients are the gate).

# Open Questions

> ✅ **OQ-11 – OQ-13 (Phase 3B series-discoverability work) — answered by Mason 8/2/2026 and implemented the same day.** OQ-11 widened the feed to workshops (**3B.8**); OQ-12 confirmed the shipped defaults (no work); OQ-13 reshuffled the URLs (**3.7**).
>
> ✅ **OQ-1 – OQ-10 all resolved (7/30/2026, round 2)** and folded into the plan: OQ-1 follow-up — account-dropdown "Memberships" entry removed (Phase 1.1); OQ-3 follow-up — day-grouped weekly timetable confirmed (Phase 2.4); OQ-9 — option (c), the weekly-schedule surface is deleted end to end (new Phase 2.5). The full Q&A dialog is kept below for the record.

- **OQ-11 — Does the series card cover workshops too?** ✅ **RESOLVED — Mason: *"Let's do workshops too."*** (8/2/2026) The series-only feed became one **offerings** feed: `GET /api/upcoming_offerings` returns series runs **and** derived workshop occurrences, and the card renders both with a Series/Workshop chip. A materialized workshop date is dropped from the card on Upcoming Events (it is already in the events list below), keyed on `event_session_id`. Implemented in **3B.8**.
	- Mason- Let's do workshops too.
- **OQ-12 — Collapsed or expanded by default?** ✅ **RESOLVED — Mason: *"These defaults are fine."*** (8/2/2026) No work: **expanded on Upcoming Events**, **collapsed on Our Classes** (and the home page when Phase 5 places it), exactly as shipped.
	- Mason- These defaults are fine.
- **OQ-13 — Should the URLs be reshuffled now that the pages are renamed?** ✅ **RESOLVED — Mason: *"This sounds like a good change."*** (8/2/2026) `/classes` → **Our Classes**, `/classes/all` → **All Classes**, `/schedule` redirects to `/classes`; every link into the catalog re-pointed. Implemented in **3.7**. *(Note: you flagged this one as "shouldn't involve any work" — it did, since it is a route change plus the links that fed it. OQ-12 was the one that needed none.)*
	- Mason- This sounds like a good change.

- **OQ-1 — "Memberships" menu target.** I plan to point the renamed **Memberships** item at `/shop/subscriptions` (the pure tier list). The generic `/shop` catalog (one-time products + subscriptions — currently also massage/spa entry points live under Services) stays routed but loses its menu entry. OK? If you'd rather keep the full catalog reachable, I can point Memberships at `/shop` unchanged-but-renamed, or add a "Full shop" link on the Memberships page.
	- Mason- We currently have Services, Upcoming Events, and Shop. Services is currently a drop down that has Browse Services and Upcoming Events. Given that Upcoming Events also has a separate menu item, it seems like making Services not a dropdown and just going straight to the current Browse Services item would make the most sense. There isn't anything under Shop besides memberships so it would make more sense to just change that label to memberships.
	- Claude- ✅ Applied as a **label-only rename**: the top-level item now reads **Memberships** and keeps targeting the existing shop page — which today lists exactly the three membership products (nothing else seeded is a shop-catalog kind), so the page already looks like a memberships page. **Follow-up:** the account dropdown also has a "Memberships" entry pointing at the *separate* tier-list page (`/shop/subscriptions`). After the rename, two menu items would share the label "Memberships" while opening different pages. I recommend removing the account-dropdown entry — the top-level item covers the storefront, and "My Subscriptions" already covers what you own. Remove it?
	- Mason- Sure. Let's make it as simple as possible.
	- Claude- ✅ Done — Phase 1.1 now removes the account-dropdown "Memberships" entry; the only "Memberships" in the menu is the top-level item (the `/shop/subscriptions` route itself stays).
- **OQ-2 — Top-level "Upcoming Events" item.** You asked to remove Upcoming Events from the *Services* dropdown (done in 1.1); a separate top-level **Upcoming Events** menu item also exists. I plan to **keep** the top-level item. Confirm, or should it go too (home page + calendar would then be the discovery surfaces)?
	- Mason- Let's leave it as a top level menu item.
	- Claude- ✅ Kept; no plan change needed.
- **OQ-3 — Logged-out Our Schedule presentation.** "Have all the classes just up with their time slots" — I read that as the current **day-grouped weekly timetable** (each day listing its class occurrences), same layout for logged-in plus attend chips. The alternative reading is a **class-grouped** list (each class card with its weekly slots, e.g. "Knotty Yoga — Mon 6pm, Wed 6pm"). Which do you want for logged-out?
	- Mason- For logged in users, the default calendar view should just show the classes that the person is eligible based on their membership (with a toggle to see the full schedule). For non-logged in, they should see the full schedule as an advertisement.
	- Claude- Two notes. **(1)** Your answer describes the eligibility filter — good news: the **calendar page already works exactly that way**. `CalendarService` defaults logged-in users to "My Schedule" (only classes they're eligible for, with a "Full Schedule" toggle) and anonymous visitors to the full schedule. So no calendar-side change is needed, and I've added the same defaults + toggle to the **redone Our Schedule page** (Phase 2.4). **(2) Follow-up — the layout half of the original question went unanswered:** I'm proceeding with the **day-grouped weekly timetable** (each day lists its class occurrences with time slots, like the current page). The alternative was a class-grouped card list ("Knotty Yoga — Mon 6pm, Wed 6pm"). Speak up only if you'd rather have the class-grouped form.
	- Mason- This is fine with me.
	- Claude- ✅ Day-grouped weekly timetable locked into Phase 2.4.
- **OQ-4 — Workshop occurrence cap on the class page.** The class page must keep a bookable occurrence list for workshops (it's how a workshop occurrence gets materialized + booked from there). I plan to cap it at the next **5** with "See all dates on the calendar". OK, or different number / different approach?
	- Mason- Sure, that seems reasonable.
	- Claude- ✅ Cap of 5 locked into Phase 3.2.
- **OQ-5 — "Events you could sign up for" composition.** Planned: featured events (`placement=home_page`) the user hasn't booked **plus** open-window workshop/series offerings from `upcoming_signup_offerings` over the next 30 days, capped at ~4 combined. Should it instead use the broader `placement=upcoming` event list, a different window, or a different cap?
	- Mason- Let's just use the upcoming events list that already exists. Capping at the next four closest to happening events seems like a good idea.
	- Claude- ✅ Applied: "Events you could sign up for" now reads straight from the existing Upcoming Events list (`placement=upcoming`), drops sessions you've already booked, and shows the **4 soonest**. The offerings merge is gone. One consequence to be aware of: the **logged-out** home strip keeps the *curated* home-page placement (the show-on-home-page flags you set in Manage → Events) since that's the existing visitor-marketing surface, while logged-in users see the personalized sections instead — flag it if you'd rather both use the same upcoming list.
- **OQ-6 — "Has a membership" rule for hiding the tier section.** Planned: same derivation the profile hub uses — active subscription OR a shared entitlement from `GET /api/subscriptions`. Note this misses an admin-comped membership entitlement with no subscription row (such a member would still see the tier pitch). Acceptable for now, or should I add a small backend "has active membership" signal (e.g. person holds any `is_pricing_eligible` permission) and use that instead?
	- Mason- That is acceptable.
	- Claude- ✅ Sticking with the profile-hub derivation (`getSubscriptions()`); no backend signal added.
- **OQ-7 — Getting Started copy.** Phase 6 ships my draft (7 steps above, written around the intro-workshop-first funnel from [[Marketing plan for studio]]). Want to red-pen the copy in this doc before I build, or edit after it's rendered?
	- Mason- We can tweak it after. I'd like to get the structural stuff done now and we can revise text later.
	- Claude- ✅ Phase 6 ships the draft copy as-is; we'll revise the wording together once it renders.
- **OQ-8 — Show membership tiers to logged-out visitors?** Your sentence scopes the tier section to users without a membership; logged-out visitors also have none, and it's strong marketing material, so Phase 5 shows tiers to them too. Confirm?
	- Mason- Yes, that is important for marketing. They need to see the membership tiers so they can buy one.
	- Claude- ✅ Tier cards show for logged-out visitors (Phase 5.3).
- **OQ-9 — `/api/schedule/week` after the redo.** Phase 2 leaves the endpoint + `getWeeklySchedule` in place but UI-unused. Fine to leave for now (I'd note it as a later removal), or should Phase 2 delete the frontend method (backend endpoint stays for any external consumers)?
	- Mason- Can you explain this more?
	- Claude- Sure — here's the fuller picture. `GET /api/schedule/week` is the backend endpoint the **current** Our Schedule page is built on, and that page is its **only caller in the entire app**. Phase 2 rebuilds the page on the calendar feed instead (the weekly-schedule payload has no slot ids or per-viewer access state, so it can't drive attend toggles), which means that after the redo **nothing calls this endpoint anymore**. The question is what to do with the leftovers, which live in two halves: the *frontend* half is the `getWeeklySchedule` method, present in four files (the `ServerAccess` interface, the network implementation, the proxy, and the mock) plus its `ServerAccess.mock.spec.ts` tests; the *backend* half is the endpoint + `WeeklyScheduleHelper` + their C++ tests. Options: **(a) leave everything** — zero work now, but the app permanently carries a dead API method that every future `ServerAccess` maintenance pass has to step around; **(b) delete the frontend method, keep the backend** *(my recommendation)* — the Angular side stays clean, and the public endpoint remains available if anything external or a future page ever wants a lightweight weekly grid; small addition to Phase 2; **(c) delete both ends** — cleanest end-state, but adds C++ churn (endpoint, helper, tests, `web_app.cpp` registration) to an already-large phase; better as a standalone cleanup later once the redo has soaked. Phase 2.4 is currently written assuming **(b)** — say the word if you'd rather have (a) or (c).
	- Mason- Let's go with C. I understand the concern about code churn but I hate dead code.
	- Claude- ✅ Option (c) it is. Phase 2 gained a dedicated removal subsection (2.5): a no-remaining-callers pre-check, then the frontend `getWeeklySchedule` seam method + `WeeklySchedule`/`ScheduleDay`/`ScheduleSlot` types, then the backend `/api/schedule/week` endpoint + `WeeklyScheduleHelper` + their tests and CMake/`web_app.cpp` registrations — all deleted in the same change set as the page rebuild so the endpoint never lingers caller-less.
- **OQ-10 — Calendar deep-link filter.** Class-page "See on the calendar" links land on the unfiltered calendar. Want a small enhancement where `/calendar?class_id=<id>` pre-filters to that class (extra work item in Phase 3), or is the plain link fine?
	- Mason- Let's just go straight to the calendar unfiltered. I like the idea of them looking up the calendar this way but then seeing what else there is.
	- Claude- ✅ Plain unfiltered `/calendar` links locked into Phase 3.