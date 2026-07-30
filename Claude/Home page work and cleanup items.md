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
   `Get Started` → `/start` (real page, Phase 6) · `About ▾` (unchanged) · `Our Classes` → `/schedule` (direct item, no dropdown) · `Services` → `/shop/services` (direct item, no dropdown) · `Upcoming Events` → `/events` (kept top-level — OQ-2) · `Memberships` → `/shop/subscriptions` (renamed from Shop — OQ-1) · [auth] `Your Calendar` · staff/admin unchanged · user dropdown minus the two dead links. With the per-class dropdown entries gone, `HeaderService` no longer needs to fetch `/api/classes` at all — the fetch, per-identity cache, and the `classes` parameter of `mockHeaderResponse` are removed.
2. **The redone Our Schedule page is rebuilt on the calendar feed**, not `GET /api/schedule/week`. Rationale: attend toggles need `class_schedule_slot_id` + `occurrence_date_us` + per-viewer `access` + the attendance overlay — exactly what `/api/calendar` + `/api/me/upcoming_classes` provide and what `/api/schedule/week` lacks. The calendar's chip mechanics ("like the calendar", per the Overview) are extracted to `shared/` and reused. Two small backend additions make one data source serve both auth states: anonymous access states, and the predecessor-class note (parity with the old page). `/api/schedule/week` stays in place but becomes UI-unused (retirement is out of scope — OQ-9).
3. **Class detail keeps a booking surface for paid kinds.** "Link to calendar instead of showing upcoming sessions" is applied literally to recurring classes (their derived list is the one that explodes — 90 days × slots). Workshops keep a **capped** occurrence list (it is the only place a workshop occurrence can be materialized + booked from the class page) plus the calendar link; series keep the runs/buy section. (OQ-4 confirms the cap.)
4. **Series booking becomes deep-linkable.** New public endpoint `GET /api/class_series_run/<classInstanceId>` returns the single run; `SeriesBookingComponent` self-loads when router state is missing. `class_instance_id` is added to `UpcomingSignupOffering`. Then the Workshops & Series tab gets real Book CTAs and the calendar routes eligible open-window series/workshop clicks straight into the booking flow.
5. **Home page composition** (one `HomePageComponent`, sections toggled on `authData$`):
   - Both states: photo carousel (gains `id="gallery"`) + hero marketing + featured events strip (`placement=home_page`, several cards instead of one).
   - Logged out: membership tier cards (shared component extracted from the subscriptions catalog — OQ-8) + "Get Started" CTA banner.
   - Logged in: "Your upcoming events" (`my_bookings` upcoming, first few), "Events you could sign up for" (featured events not already booked + `upcoming_signup_offerings` — OQ-5), "Next 4 days at the studio" (embedded `UpcomingClassesComponent` with a new `daysAhead` input — attend/not-attend toggles come for free), tier cards **only when no membership** (same `getSubscriptions()` derivation the profile hub uses — OQ-6), and a "Plan your weekly schedule" link → `/my/my-schedule`.
6. **Getting Started** becomes a real public page at `/start` with step-by-step instructions (draft copy in Phase 6; final copy — OQ-7).
7. **Testing discipline**: every backend change lands with helper + endpoint tests; every frontend change lands with component specs, and any `ServerAccess` change updates interface/network/proxy/mock + `ServerAccess.mock.spec.ts`. Each phase ends with live-server hand-testing steps against the seeded database.

---

## Phase 1 — Menu & cross-link cleanup (frontend only)

> No backend work in this phase. All items are small, independent, and unblock the later phases' navigation story.

### 1.1 Menu restructure
- [ ] `mockHeaderResponse.ts`: replace the "Our Classes" dropdown with a direct `InternalLink` **Our Classes** → `/schedule`; delete the per-class entries and the "All Classes" dropdown entry (the All Classes page stays routed and gets linked from the redone schedule page in Phase 2).
- [ ] `mockHeaderResponse.ts`: replace the "Services" dropdown with a direct `InternalLink` **Services** → `/shop/services` (drops the duplicate "Upcoming Events" entry; the top-level Upcoming Events item stays — OQ-2).
- [ ] `mockHeaderResponse.ts`: rename **Shop** → **Memberships**, target `/shop/subscriptions` (OQ-1; flip the target back to `/shop` if OQ-1 answers differently).
- [ ] `mockHeaderResponse.ts`: remove the dead user-dropdown entries "My Goals" (`/goals`) and "My Classes" (`/myclasses`).
- [ ] Drop the now-unneeded `classes` parameter from `mockHeaderResponse(...)`; simplify `HeaderService.refreshHeaderData` to stop fetching `/api/classes` and delete the per-identity class cache.
- [ ] Specs: update `header.service.spec.ts` (no class fetch), `mockHeaderResponse` assertions wherever they live (header/component specs), and mobile-menu spec if it asserts menu contents. Assert: Our Classes is a direct link to `/schedule`; Services is a direct link to `/shop/services`; "Memberships" label present, "Shop" absent; no `/goals` / `/myclasses` entries; Upcoming Events appears exactly once.

### 1.2 Classes page → calendar link (and keep All Classes reachable)
- [ ] `class-info.component.html`: add a clearly visible "View the calendar" link/button (→ `/calendar`) near the page title, styled consistently with existing page-header actions.
- [ ] `our-schedule.component.html`: add a "Browse all classes" link (→ `/classes`) in the page header **now** — with the menu entry gone, this is the classes page's entry point until the Phase 2 redo (which keeps the link).
- [ ] Specs: `class-info.component.spec.ts` renders the calendar link; `our-schedule.component.spec.ts` renders the browse-all-classes link.

### 1.3 Calendar → schedule template link
- [ ] `calendar-home.component.html` toolbar: add a "My weekly plan" link (→ `/my/my-schedule`), shown only when logged in (next to the My Schedule/Full Schedule mode toggle).
- [ ] `calendar-home.component.spec.ts`: link renders when authed, absent when anonymous.

### 1.4 Schedule template page → notification settings link
- [ ] `my-schedule.component.html`: add a "Notification settings" link (→ `/my/notification-preferences`) in the page header area.
- [ ] `my-schedule.component.spec.ts`: link renders.

### 1.5 Gallery anchor fix
- [ ] `home-page.component.html`: give the photo-carousel section `id="gallery"` so the existing About ▸ Gallery `/#gallery` fragment link actually lands somewhere.
- [ ] `home-page.component.spec.ts`: anchor present.

### 1.6 Live hand-testing (Phase 1)
Fresh database via `knottyyoga_database_helper`, server + `ng serve` running, no extra data needed.
1. Logged out: the top menu reads **Get Started, About, Our Classes, Services, Upcoming Events, Memberships, Sign In**. **Our Classes** navigates straight to the weekly schedule page (no dropdown). **Services** navigates straight to the Browse Services page (no dropdown). **Memberships** opens the membership tier list showing **Knotty Yoga Gold Membership**, **Knotty Yoga Gold Couple's Membership**, **Knotty Yoga Gold Family Membership**.
2. Open **About** → **Gallery**: the page scrolls to the photo section of the home page.
3. Register/log in (menu **Sign In** → **Create one** flow), then open the account menu (your first name, far right): it lists **Profile, My Purchases, My Bookings, Memberships, My Subscriptions, Sign Out** — no "My Goals", no "My Classes".
4. Open **Your Calendar**: a **My weekly plan** link is visible and opens the My Schedule page; on that page a **Notification settings** link opens Notification Preferences.
5. Open **Our Classes**, then use the schedule page's **Browse all classes** link: the All Classes page opens and shows the **View the calendar** link, which opens the calendar.

---

## Phase 2 — Our Schedule page redo (the "Our Classes" landing page)

> Backend first (data layer additions to the calendar feed), then the shared-component extraction, then the page rebuild.

### 2.1 Backend — anonymous access states in the calendar feed
`business_logic/scheduling/calendar_helper.cpp` currently skips the access gate for `personId == 0`. Change: evaluate class access for anonymous viewers against an **empty** effective-permission/skill set, so anonymous items carry real `access` states and the membership deep-link fields.
- [ ] For anonymous viewers, run the same `ClassAccessHelper`-based evaluation used for logged-in viewers with empty permission + skill sets. Classification: any unsatisfied requirement group containing a permission literal → `members_only` (populate `required_permission_id`/`required_permission_name` from the group's permission literal exactly as the logged-in path does); unsatisfied groups with only skill literals → `needs_skill` + `missing_skill_ids`.
- [ ] Leave bookings/service-booking overlays and attendance state absent for anonymous (unchanged).
- [ ] Tests (`calendar_helper_test.cpp`): anonymous viewer sees `members_only` + required-permission fields on a permission-gated class; `eligible` on an open class; `needs_skill` on a skill-gated class; logged-in behavior unchanged (regression case).

### 2.2 Backend — predecessor note on `CalendarItem`
The old schedule page showed "Requires attending: {class}" from `/api/schedule/week`; the calendar feed lacks it. Parity addition:
- [ ] Add `predecessor_class_name` (empty when none) to `CalendarItem`: during class-occurrence derivation, when the slot has `predecessor_class_schedule_slot_id`, resolve that slot's class name.
- [ ] KVT converter + `scheduling_key_value_table_test.cpp` case.
- [ ] `calendar_helper_test.cpp`: slot with a predecessor surfaces the name; slot without stays empty.

### 2.3 Frontend — extract the calendar occurrence mechanics to `shared/`
So the schedule page (and later the home feed / offerings tab) can render occurrences "like the calendar" without cross-page imports:
- [ ] Move `pages/calendar/components/views/calendar-event/` → `shared/components/calendar-event/`, `pages/calendar/components/attendance-dialog/` → `shared/components/attendance-dialog/`, `pages/calendar/components/skill-requirement-dialog/` → `shared/components/skill-requirement-dialog/`, and `pages/calendar/calendar-navigation.service.ts` → `shared/services/calendar-navigation.service.ts` (selectors/class names unchanged — pure relocation; update all imports).
- [ ] Move their `.spec.ts` files alongside and keep them green.
- [ ] Extract the attendance-map builder (keyed `${slotId}-${occurrenceDateUs}`, currently inside `CalendarService`) into a small shared util so the schedule page can reuse it; `CalendarService` calls the util. Unit test for the util.
- [ ] `ui/src/app/shared/types/calendar-item.types.ts`: add `predecessor_class_name`; `ServerAccessNetwork.getCalendar` normalization updated if needed; `ServerAccess.mock.spec.ts` updated for the new field.

### 2.4 Frontend — rebuild `OurScheduleComponent`
- [ ] Data: `GET /api/calendar` for the selected week window (from `max(now, week start)` to week end) filtered to class kinds (`class`, `workshop`, `series` — standalone `event`s stay on Upcoming Events/home) + facility filter dropdown reused from the calendar toolbar pattern; when logged in, also `GET /api/me/upcoming_classes` for the same window to build the attendance map.
- [ ] Layout: day-grouped list for the week (keeps the current page's day-section feel — OQ-3), each occurrence row shows class photo, name (→ `/classes/:id`), time + duration, room, instructor (+ substitute note), `predecessor_class_name` note, and a **calendar-style chip** (`app-calendar-event` states: attend toggle for plannable recurring items, Booked/Waitlisted, Sold out, "Sign-ups open {date}" + Remind me, Cancelled, lock for `members_only`/`needs_skill`).
- [ ] Week navigation: **This week / Next week / following weeks** (past days of the current week are omitted — the feed clamps to now).
- [ ] Logged-in extras: a "Plan your weekly schedule" link → `/my/my-schedule` in the page header.
- [ ] Logged-out behavior: same list with time slots, no attend toggles; `members_only` rows show "Requires a membership" linking to the Memberships page (via the shared `CalendarNavigationService` behavior — same as the calendar's lock handling); page header gets "New here? Get started" (→ `/start`) and "Sign in" CTAs.
- [ ] Carry the Phase-1 "Browse all classes" link (→ `/classes`) through into the redone page header — it is where All Classes lives now that it left the menu.
- [ ] Remove the `getWeeklySchedule` usage from the page (the `ServerAccess` method and backend endpoint remain — OQ-9).
- [ ] Specs: rewrite `our-schedule.component.spec.ts` — logged-out rows without toggles + membership link on gated class; logged-in attend toggle calls `setException` optimistically; week navigation; browse-all-classes + weekly-plan links; facility filter.

### 2.5 Live hand-testing (Phase 2)
Fresh database; use the test-helper app to add a second, membership-gated class so both states are visible (per its documented commands — create a class with a requirement group referencing the Gold Member permission and a Tue/Thu slot).
1. Logged out, open **Our Classes**: the week list shows **Knotty Yoga** on Monday and Wednesday at 6:00 PM with its room and duration, with no attend controls; the membership-gated class shows a **Requires a membership** link that lands on the Memberships page; the header shows **New here? Get started** and **Sign in**, plus **Browse all classes** opening the All Classes page.
2. Log in as a member who passes the gate (buy **Knotty Yoga Gold Membership** with the Square sandbox card via **Memberships**, or grant the permission via the test-helper), reopen **Our Classes**: the Monday 6:00 PM **Knotty Yoga** row shows the **Tap to plan attendance** chip; tapping it and confirming **I'll be there** flips it to the checked "I'll be there" state; the same state is visible on **Your Calendar** for that Monday.
3. Use **Next week** and confirm the same weekly pattern renders for the following week.

---

## Phase 3 — Class detail page cleanup

> Frontend only. Preserves the workshop booking surface (Design Decision 3).

### 3.1 Upcoming Sessions → calendar link (recurring classes)
- [ ] `class-detail.component.html/.ts`: for `kind === 'recurring'`, replace the entire Upcoming Sessions list with a "See this class on the calendar" card/CTA → `/calendar` (OQ-10 covers an optional `?class_id=` pre-filter).
- [ ] Weekly-pattern summary (day/time slots) stays if already shown elsewhere on the page; nothing else about the recurring layout changes.

### 3.2 Workshops — capped list + calendar link
- [ ] For `kind === 'workshop'`, cap the rendered occurrence list to the next **5** (OQ-4) and add "See all dates on the calendar" → `/calendar` below it. The cap is a template slice — the bookable click-through (`materializeClassOccurrence` → `/shop/event/:sessionId`) is unchanged.

### 3.3 Series — unchanged runs section + calendar link
- [ ] Series keep the runs/buy section as-is; add the same "See this class on the calendar" link for consistency.

### 3.4 Specs
- [ ] `class-detail.component.spec.ts`: recurring shows the calendar link and no session cards; workshop shows ≤5 session cards + the link and still books an occurrence on click; series shows runs + link.

### 3.5 Live hand-testing (Phase 3)
1. From **Our Classes**, open **Knotty Yoga**: the page shows **See this class on the calendar** instead of a long Upcoming Sessions list, and the link opens the calendar.
2. Create a workshop class with several future occurrences via the test-helper app; open its class page: at most 5 dated session cards render plus **See all dates on the calendar**; clicking a session card still leads to the event booking page.

---

## Phase 4 — Series & workshop sign-up discoverability

> Backend first: make a series run addressable by `class_instance_id`; then wire Book CTAs everywhere the Overview asks for (profile tab, calendar, class page — the class page already books).

### 4.1 Backend — `GET /api/class_series_run/<int:classInstanceId>` (public)
- [ ] `business_logic/scheduling/`: `ClassSeriesHelper` gains `GetRunByClassInstanceId(Transaction&, classInstanceId, personId)` returning the same run shape as `GetSeriesRunsForClass` (reuse the existing resolution/pricing internals; personalized pricing when logged in, public otherwise).
- [ ] New thin endpoint `endpoints/get_class_series_run.{h,cpp}` → `{ run: {...} }`, 404 when the instance doesn't exist or isn't a series run; registered in `web_app.cpp` + `endpoints/CMakeLists.txt`.
- [ ] Tests: helper test (found / wrong-kind / missing) + endpoint test (`get_class_series_run_test.cpp`: 200 shape, 404, anonymous OK).

### 4.2 Backend — `class_instance_id` on `UpcomingSignupOffering`
- [ ] `SignupReminderHelper::GetUpcomingSignupOfferings` (Phase 11 code): resolve and carry `class_instance_id` for each offering (series and workshop rows both have an owning instance).
- [ ] KVT converter + `scheduling_key_value_table_test.cpp` (or the Phase 11 KVT test file) updated; `get_my_upcoming_signup_offerings_test.cpp` asserts the field.

### 4.3 Frontend — `ServerAccess` additions
- [ ] `getSeriesRunByInstance(classInstanceId)` → `GET /api/class_series_run/:classInstanceId` across interface / network / proxy / mock; `UpcomingSignupOffering` type gains `class_instance_id` (+ network numeric normalization).
- [ ] `ServerAccess.mock.spec.ts`: new-method cases (found/404/logged-out allowed) + offering field coverage.

### 4.4 Frontend — self-sufficient `SeriesBookingComponent`
- [ ] When `history.state.run` is absent, fetch via `getSeriesRunByInstance(classInstanceId)` instead of dead-ending in `'missing'` (keep the error state for a real 404).
- [ ] `series-booking.component.spec.ts`: state-provided path unchanged; state-missing path fetches and renders; 404 shows the error state.

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
- [ ] **Featured events strip**: render up to 4 cards from `getVisibleEventSessions('home_page')` (name, time, facility, price, Book Now → `/shop/event/:id`) + "View All Events" → `/events`; empty state unchanged.
- [ ] **Getting Started CTA banner** → `/start` (both states; copy differs slightly when logged in).

### 5.3 Frontend — logged-out sections
- [ ] **Membership tiers section**: `getCatalogProducts()` filtered to `kind === 'subscription'` rendered via the shared tier cards (OQ-8), headed with short marketing copy; Subscribe click follows the existing auth-guarded flow (redirects to login).

### 5.4 Frontend — logged-in sections
- [ ] **Your upcoming events**: `getMyBookings('upcoming')`, first 3 cards (event name, date, facility, status incl. waitlisted) + "All my bookings" → `/my/events`; section hidden when empty.
- [ ] **Events you could sign up for**: featured events minus ones already in the user's upcoming bookings, plus `getUpcomingSignupOfferings(now, +30d)` rows with `window_open` (Book CTAs reuse the Phase 4 flows); cap combined list (OQ-5); section hidden when empty.
- [ ] **Next 4 days at the studio**: embed `app-upcoming-classes` with a new `@Input() daysAhead = 28` set to 4 and `[embedded]="true"` (attend / can't-make-it toggles come with the component); headline links "Plan your weekly schedule" → `/my/my-schedule`.
- [ ] `upcoming-classes.component`: add the `daysAhead` input (window = now → now + daysAhead days) + spec case.
- [ ] **Membership tiers when no membership**: on `getSubscriptions()` returning no active subscription and no shared entitlement (the profile-hub rule — OQ-6), show the same tier section with "Become a member" copy; hidden for members.
- [ ] Auth branching on `AuthService.authData$` (subscribe/refresh on auth change, like the header does).
- [ ] Specs: `home-page.component.spec.ts` rewritten — logged-out: hero + events strip + tiers + no personalized sections; logged-in with membership: bookings/could-sign-up/4-day feed present, tiers absent; logged-in without membership: tiers present; events strip caps at 4; booked events excluded from "could sign up for". Mock `SquarePaymentService` is not needed (no embedded payment control), but keep the rule in mind if a card control ever lands here.

### 5.5 Live hand-testing (Phase 5)
Fresh database. Via **Manage** → **Events** → **Create Event**, create two future **Intro Workshop** sessions (one flagged for the home page placement, fields per the create-event form labels) — or use the test-helper app equivalents.
1. Logged out, open the home page: carousel + hero render; the **featured events** strip shows the Intro Workshop card(s) with **Book Now**; the **membership tiers** section shows the three Gold memberships with **Subscribe** buttons; a **Get Started** banner is present. Clicking **Subscribe** routes through **Sign In**.
2. Log in as a fresh account (no membership): the home page adds **Next 4 days at the studio** (empty-state text until you're eligible for a class), **Events you could sign up for** (the workshop sessions), and keeps the tier section visible.
3. Buy **Knotty Yoga Gold Membership** (Square sandbox card). Back on the home page: the tier section is gone; the **Next 4 days** feed lists Monday/Wednesday **Knotty Yoga** rows with **I'll be there / I can't make it** controls that flip state; **Plan your weekly schedule** opens My Schedule.
4. Book one Intro Workshop session; the home page now shows it under **Your upcoming events**, and it disappears from **Events you could sign up for**.

---

## Phase 6 — Getting Started page

### 6.1 Frontend — page + route
- [ ] New `pages/public/getting-started/getting-started.component.*` (separate template/styles per convention); `public.routes.ts`: `start` → `GettingStartedComponent` (menu already points there).
- [ ] Content — numbered step cards with CTA buttons (draft copy; final wording OQ-7):
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

- **Backend test files touched**: `calendar_helper_test.cpp` (2.1, 2.2), `scheduling_key_value_table_test.cpp` (2.2, 4.2), new `get_class_series_run_test.cpp` + series-helper tests (4.1), `get_my_upcoming_signup_offerings_test.cpp` (4.2).
- **Frontend spec files touched**: header/service + mobile menu (1.1), `class-info` (1.2), `calendar-home` (1.3), `my-schedule` (1.4), `home-page` (1.5, 5.4), relocated calendar chip/dialog/navigation specs + new attendance-map util spec (2.3), `our-schedule` rewrite (2.4), `class-detail` (3.4), `ServerAccess.mock.spec.ts` (2.3, 4.3), `series-booking` (4.4), `upcoming-offerings` (4.5), `calendar-navigation.service` (4.6), new `membership-tier-cards` + `subscription-catalog` (5.1), `upcoming-classes` (5.4), new `getting-started` (6.1).
- Angular gates per phase: affected specs green + `ng build` clean. C++ builds/tests run by Mason (Linux docker clients are the gate).

# Open Questions

Please answer inline under each.

- **OQ-1 — "Memberships" menu target.** I plan to point the renamed **Memberships** item at `/shop/subscriptions` (the pure tier list). The generic `/shop` catalog (one-time products + subscriptions — currently also massage/spa entry points live under Services) stays routed but loses its menu entry. OK? If you'd rather keep the full catalog reachable, I can point Memberships at `/shop` unchanged-but-renamed, or add a "Full shop" link on the Memberships page.
	- Mason- We currently have Services, Upcoming Events, and Shop. Services is currently a drop down that has Browse Services and Upcoming Events. Given that Upcoming Events also has a separate menu item 
- **OQ-2 — Top-level "Upcoming Events" item.** You asked to remove Upcoming Events from the *Services* dropdown (done in 1.1); a separate top-level **Upcoming Events** menu item also exists. I plan to **keep** the top-level item. Confirm, or should it go too (home page + calendar would then be the discovery surfaces)?
- **OQ-3 — Logged-out Our Schedule presentation.** "Have all the classes just up with their time slots" — I read that as the current **day-grouped weekly timetable** (each day listing its class occurrences), same layout for logged-in plus attend chips. The alternative reading is a **class-grouped** list (each class card with its weekly slots, e.g. "Knotty Yoga — Mon 6pm, Wed 6pm"). Which do you want for logged-out?
- **OQ-4 — Workshop occurrence cap on the class page.** The class page must keep a bookable occurrence list for workshops (it's how a workshop occurrence gets materialized + booked from there). I plan to cap it at the next **5** with "See all dates on the calendar". OK, or different number / different approach?
- **OQ-5 — "Events you could sign up for" composition.** Planned: featured events (`placement=home_page`) the user hasn't booked **plus** open-window workshop/series offerings from `upcoming_signup_offerings` over the next 30 days, capped at ~4 combined. Should it instead use the broader `placement=upcoming` event list, a different window, or a different cap?
- **OQ-6 — "Has a membership" rule for hiding the tier section.** Planned: same derivation the profile hub uses — active subscription OR a shared entitlement from `GET /api/subscriptions`. Note this misses an admin-comped membership entitlement with no subscription row (such a member would still see the tier pitch). Acceptable for now, or should I add a small backend "has active membership" signal (e.g. person holds any `is_pricing_eligible` permission) and use that instead?
- **OQ-7 — Getting Started copy.** Phase 6 ships my draft (7 steps above, written around the intro-workshop-first funnel from [[Marketing plan for studio]]). Want to red-pen the copy in this doc before I build, or edit after it's rendered?
- **OQ-8 — Show membership tiers to logged-out visitors?** Your sentence scopes the tier section to users without a membership; logged-out visitors also have none, and it's strong marketing material, so Phase 5 shows tiers to them too. Confirm?
- **OQ-9 — `/api/schedule/week` after the redo.** Phase 2 leaves the endpoint + `getWeeklySchedule` in place but UI-unused. Fine to leave for now (I'd note it as a later removal), or should Phase 2 delete the frontend method (backend endpoint stays for any external consumers)?
- **OQ-10 — Calendar deep-link filter.** Class-page "See on the calendar" links land on the unfiltered calendar. Want a small enhancement where `/calendar?class_id=<id>` pre-filters to that class (extra work item in Phase 3), or is the plain link fine?