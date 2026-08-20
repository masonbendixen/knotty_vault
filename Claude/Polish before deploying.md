---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/20/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I've been demoing the application getting it ready to deploy and have found a number of issues. I'm just going to list them. If you can try to group them into buckets to implement, ask me questions, and build an implementation plan to accomplish these, that would be great. Please use the code base and these documents for context:

- [[Component Inventory for Designer]]
- [[Splitting the server up into components]]
- [[Componentizing the frontend]]
- [[Converting the server to a multi tenant Saas architecture]]
- [[Deploying to AWS]]
- [[Home page work and cleanup items]]
- [[Tenant Theming and Branding]]
- [[Website Makeover]]

Here are the items that we need to improve before being ready to ship:
- After changing the contact email for the site, the footer doesn’t refresh until refreshing the page.
- The How test looks section does the font sizes in rem units. Can we have a toggle to switch this to points and pixels in addition to REM units?
- Font service doesn’t seem to persist in the Fonts admin portal. I set it, and it works, but returning to the page to add a new one, I need to Select Google for each one.
- We need to alphabetize the portal entries for the admin, staff, and personal profile portals
- Studio locations need a timezone associated with them (this should be noted in one of the documents)
- Let’s make the font weight for various things configurable in page settings. We currently hard coded it to semi bold for the menu font but it would be nice if that was configurable (generally the 100...900). It would be nice if there was both the number and a description like bold, normal, demibold, etc.
- The image carousel shows up above the banner. Let’s put this as a configurable item in the home page editor that we can move the location. I don’t also really love the cropping. Instead of forcing the aspect ratio to match one image, let certain images just be wider. Also, have an option to decide if we show the text.
	- I'd like an image carousel editor in the home page. You can name various image carousels and add images to them. Currently there is an image carousel table with just a single set of images. I'd like to support creating multiple names image carousels and have each carousel have a set of images and descriptions with CRUD type support and reordering. Each carousel should have a display name and description.
	- I'd like to add a new item to the home page portal with the type image carousel that lets you select from the list of names image carousels to display on the home page and be positioned.
	- I'd like an Image Carousels tab in the site theme admin portal editor that let's you order the carousels to be shown on the About / Gallery menu item page and have the ability to add / remove / reorder names carousel entries as well as choose whether the title and/or description is shown.
	- All these stuff should be persisted in the JSON site theme file.
- Membership, Upcoming Classes, and Upcoming Events should be different items that can be added to the home page and the position on the home page should be able to be ordered with the existing home page items. (Currently they just get added before or below all the configurable items). These will all be new types that can be chosen when adding an item in addition to carousel.
- The upcoming classes on the home page background should be –theme-surface-tint but the individual cards should be --theme-background
- Can the background image of banner / Come join the fun! be parallax scroll?
- Do you still have the access to Ryan's figma? If so, in Ryan’s design for Get started, there is an image with a Get Started with a fire font. This image, the text, and the button should all be stacked on top of each other, centered, and use the menu color scheme and be the full width of the page. The image is at Icon / GetStarted in Frame 127
- Can we get the icons for C:\Users\mason\source\repos\knottyyoga\server\knottyyoga_server\out\build\x64-Debug\src\database_helper\img tier_icon_solo.png and the other three images and use those as the icons for the Become a member on the home page and on the Memberships page
	- Can you make the membership panel on the home page look like the items when you click the Memberships menu item and go to that page?
- The That which doesn’t kill you makes you hotter on the footer should be an image and then get the image from Ryan's figma.
- Let’s change the About page from being just plain markdown to supporting alternating image / text blocks like the home page. And allow them to be reordered.
- Add an instruction on the favorite support for instructors on the instructors page
- What do preferences do in the Instructors portal?
- Have Our Classes go to the Browse all classes (for non logged in users) on the current Our Classes page and then have a My Classes that takes you to the current Our Classes page. Let's keep things the same as now for logged in users.
- On services, let’s add an image for each service and then show the image above each service in the Services page from main menu. Need to add image support to the database and then the Bookable Service panel in the product editor.
- Events don’t have an image. Let’s add an image. Let’s also add this as another configurable thing that can be placed on the home page. The home page shows a truncated version of the event session whereas the upcoming events page shows more. Let’s have the home page match the Upcoming Events page. Let's show the description from the product page.
- About / Our location with image / description. Have a button that triggers a map.
- On the Memberships page, let's call the page Memberships, not Shop. Let's also make the prices --theme-on-accent
- Can we have the icons in the portal use –theme-primary
- On the Booking summary page for a service, there is a solo person icon and a map pin that I’m not sure what they are for and they take up valuable, vertical space that I would like back. Also, the day that you are booking for should be highlighted with color.
- In the home page configuration section, let’s have an option to have things not show up for logged in users and also an option to not show up if the user has a membership.
- Let's create an announcements with a title, description, and a show from start / end date for when announcements should be shown.  If there are any entries for the current date, make them show up on the top of the home page.
- Let's have home page entry type for a video and allow various video links to be placed (like an embedded YouTube video)

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here

> Plan drafted 8/20/2026 from a full read of [[Tenant Theming and Branding]], [[Home page work and cleanup items]], [[Website Makeover]], [[Component Inventory for Designer]], and a four-way deep exploration of the codebase (home system, theme/fonts/site_info, portals/pages, backend data layer). Checkboxes get ticked as items land. Questions are collected in **Open Questions** at the bottom — please answer there.

## Grounding — what the exploration found (so the phases below are uncontroversial)

Facts that shaped the plan; each one changed what a naive reading of the item list would have built:

- **The footer bug is three bugs stacked.** The Site Theme editor's save only re-applies fields that have a `css_variable` — content slots like `site_contact_email` never reach `SiteConfigService` (`site-theme.component.ts:528-533`). Even if it did, `applySiteInfo()` swaps `_config` for a **new object** while the footer captured the old reference in its constructor (`footer.component.ts:19`), and the footer lives outside `<router-outlet>` so it is never re-created. Header, Home, About, and Getting Started snapshot the config the same way.
- **The fonts-portal bug is a round-trip omission.** `GET /api/manage/site_fonts` emits the numeric `font_source_id` but the UI's service `mat-select` binds `family.source_key`, which the payload never carries — so every load renders the select blank even though the value saved fine. (honuware `manage_site_fonts.cpp:210-219` vs `site-fonts.component.html:106`.)
- **Font-weight *scale* tokens already exist** (`site_theme_weight_regular/medium/semibold/bold`, editable in the "How text looks" section as bare number inputs). What's missing is (i) a named picker control and (ii) **role** weights — the menu is welded to `--weight-semibold` in `header.component.scss:21` and `.din-bold`/`.din-condensed-bold` in `_fonts.scss:25,35`, so "make the menu lighter" today means changing what semibold *is* everywhere.
- **The server refuses `pt`.** `IsValidCssLength` (honuware `site_value_validation.cpp`) accepts only `px|rem|em|%` — a pt value would save and then be silently dropped on read. The unit toggle needs a one-line honuware change.
- **The home page is one ordered list plus eight hardcoded islands.** `home_sections` rows (hero/feature/banner/artwork) render in *three separate* `@for` blocks at fixed template positions, and the carousel, intro strip, Get Started banner, events strips, offerings card, membership tiers, and upcoming-classes embed are all hardcoded `<section>`s around them. That's exactly the "they just get added before or below all the configurable items" complaint — the fix is one `@for` over the whole ordered list with a `@switch` on `kind`.
- **The carousel table can't do carousels.** `home_page_photos` = `id, title, description, created_at_us` — no name/group, no ordinal, no active — and the query is literally `ORDER BY random() * created_at_us`. The component fetches `description` and never renders it, forces 320px-wide cards with `height: auto` (no aspect handling), and has **no admin UI at all** (raw `/admin/tables` only). It is also **absent from the theme bundle**, so carousel photos are lost on theme export today.
- **"Studio locations" = the `facilities` table, and it already has a `timezone` column** (IANA string, default `America/Los_Angeles`), consumed by booking emails, digests, and reporting. What's missing is any *editing surface* beyond the raw generic CRUD (no dedicated Manage page; the CRUD form's column list even omits `address_line_2`/`country`), plus a public display of location info. There is no `studio_locations` table — [[Bookable Service Foundation]] decision 10 is the doc that recorded the facility-timezone rule.
- **Products, events, and services have zero image support** at any layer — no image column, not in `photo_support_tables`, not in the anonymous scaled-photo allow-list. The photo system is polymorphic (`table_item_photos`, one photo per row), so "add an image to services/events/tiers" = register `products` for photos and reuse everything.
- **The three tier icons are already on disk and completely unreferenced** — `src/database_helper/img/tier_icon_{solo,couple,family}.png`, exported from Ryan's file on 8/11. No Figma access needed for that item.
- **Figma access is currently broken.** Previous sessions read Ryan's file via the REST API with a token at `~/.figma_token` — **that file no longer exists**. The two items that need new exports (the Get Started fire-font image at `Icon/GetStarted` in Frame 127, the footer tagline image) are blocked until it's restored (OQ-1). The file key is known: `figma.com/design/IxWR3NfPQbJfYJ7oCPmaER/Knotty-Yoga`.
- **Instructor "preferences" are write-only.** `instructor_class_preferences` (per-class min/max attendees + notes) is authored in the Instructors manage UI and **consumed by nothing** — the min-attendees sweep that actually runs reads `class_series_instances.min_attendees` instead. Full answer under Phase 1.7.
- **Booking-summary day highlight is a missing-CSS bug.** The template already sets `.selected`/`.available`/`.unavailable` on the week-strip buttons; no rule for any of them exists anywhere.
- **Every new table needs BOTH db_schema registration and an idempotent migration** (`BuildAppMigrations()` — currently `{}`), and every new content table/column must join the theme bundle's `page_content` section or themes silently lose it. There is no `ALTER TABLE` anywhere yet — the first column migration establishes the pattern (guard with `DbMeta::ListColumns`, then `ALTER TABLE … ADD COLUMN IF NOT EXISTS`).
- **Parallax gotcha:** the app shell scrolls an inner `overflow-y-scroll` div, not the window — `window:scroll` listeners and naive `background-attachment: fixed` won't behave. The effect must key off the app's scroll container.
- **"Hide when the user has a membership" cannot use `/api/user_info`** — its `permissions` array walks role grants only, not entitlement grants. The home page already derives `hasMembership` from `getSubscriptions()` (the profile-hub rule, accepted in the earlier OQ-6 of [[Home page work and cleanup items]]); the visibility flags reuse that.

## Buckets — every Overview item, grouped, with its phase

| # | Overview item (abbreviated) | Bucket | Phase |
|---|---|---|---|
| 1 | Footer doesn't refresh after contact-email change | A. Live-refresh & small fixes | 1.1 |
| 2 | "How text looks" rem → pt/px toggle | B. Theme typography | 2.2 |
| 3 | Fonts portal forgets the font service | A | 1.2 |
| 4 | Alphabetize portal entries | A | 1.3 |
| 5 | Studio locations need a timezone | E. Locations & About | 7.1 (already exists — expose it) |
| 6 | Configurable font weights (menu semibold) | B | 2.3 |
| 7 | Carousel above banner / position / cropping / text toggle | D. Carousels | 4 (position via 3) |
| 7a | Multiple named carousels, CRUD + reorder | D | 4.1–4.4 |
| 7b | Home item type "image carousel" | D | 4.5 |
| 7c | Image Carousels tab (gallery ordering, title/desc toggles) | D | 4.4, 4.6 |
| 7d | All persisted in the theme JSON | D | 4.7 |
| 8 | Membership / Upcoming Classes / Upcoming Events as orderable items | C. Home unification | 3 |
| 9 | Upcoming-classes colors (surface-tint bg, background cards) | A | 1.6 |
| 10 | Parallax banner | F. Visual bands | 8.4 |
| 11 | Get Started band (fire-font image, stacked, menu colors) | F | 8.2 (blocked on OQ-1) |
| 12 | Tier icons from `img/` on home + Memberships | G. Product imagery | 5.4 |
| 12a | Home membership panel look = Memberships page | G | 5.5 |
| 13 | Footer tagline as image from Figma | F | 8.3 (blocked on OQ-1) |
| 14 | About page → alternating image/text blocks | E | 7.3 |
| 15 | Instructions for instructor favorites | A | 1.7 |
| 16 | What do Instructors-portal preferences do? | A | answered in 1.7 |
| 17 | Our Classes → browse-all for anon; My Classes | A | 1.8 (OQ-2) |
| 18 | Service images (DB + product editor + Services page) | G | 5.1–5.3 |
| 19 | Event images + richer home event cards + home item type | G + C | 5.2, 3.6 |
| 20 | About / Our Location page with map button | E | 7.2 |
| 21 | Memberships page title + price color | A | 1.5 |
| 22 | Portal icons --theme-primary | A | 1.3 |
| 23 | Booking summary icons + day highlight | A | 1.4 |
| 24 | Per-item visibility (logged-in / has-membership) | C | 3.4 |
| 25 | Announcements with date windows | H. New content types | 6.1 |
| 26 | Video home item type | H | 6.2 |

## Conventions for this plan

- **Layering inside every phase**: db_schema → migration → table helpers → business logic → endpoints → `ServerAccess` seam (interface / network / proxy / mock + mock spec) → components → component specs. Backend before frontend, always.
- **[hw]** marks work in the honuware repo (`server_components`, co-dev via `FETCHCONTENT_SOURCE_DIR_HONUWARE`) — needs a CI push + pin bump in both consumers before non-co-dev builds see it. **[app]** is knottyyoga. Frontend is all app-side.
- **Every new table**: full new-table checklist (`make_app_tables.cpp`, `CreateTables`, allowed/admin metadata seeds, CMakeLists, photo/public-photo registration where relevant) **plus** an idempotent entry in `BuildAppMigrations()` — registration alone only serves brand-new databases ([[gotcha_new_table_needs_migration]]). Every new column: DSL entry + a guarded `ALTER TABLE … ADD COLUMN IF NOT EXISTS` migration (first of its kind — establishes the pattern).
- **Theme bundle**: any new home/gallery content column, kind, or table must be added to the `page_content` bundle section (`page_content_bundle_section.cpp`) in the same phase, with round-trip test coverage — otherwise theme files silently lose it.
- **Tests land in the same session as the change**, every layer. C++ gate = the Linux docker clients (run proactively, incl. the explicit `knottyyoga_database_helper` build whenever `create_database.cpp`/seed files change); Angular gate = `ng test` + `tsc --noEmit` + `ng build`. Windows/VS builds are Mason's.
- **Live hand-testing steps** close every phase (blank DB + real seed, exact menu → page → field labels, per [[feedback_test_instructions_format]]).
- **In-flight work**: the uncommitted `seed_app_data.{h,cpp}` extraction is assumed to land before Phase 3 (both touch `create_database.cpp`). Nothing below conflicts with it; new seeders follow its `namespace Seed` pattern.
- No git writes; Mason commits. Dates in this doc are absolute.

---

## Phase 1 — Live-refresh, fonts portal, and small polish fixes

> The demo blockers and one-sitting fixes. Almost entirely frontend; 1.2 has a small [hw] half.

### 1.1 Footer (and everything else) refreshes when site content is saved
The fix is structural, not a footer patch — five components snapshot `SiteConfigService.config` at construction.
- [ ] `SiteConfigService`: expose the config reactively — a `BehaviorSubject<SiteConfig>`/`config$` (getter stays for existing callers), emitted from `applySiteInfo()`. Add a public `applyContent(content: KeyValueTable-shaped map)` that runs the existing slot parsing/merge rules over the current config and emits — the content-slot twin of `applyTheme()`.
- [ ] `SiteThemeComponent.save()`: after a successful PUT, call `applyContent()` with the saved content fields (it already calls `applyTheme()` for tokens). The bundle-apply path already calls `load()` — leave it.
- [ ] Convert the stale-snapshot consumers to live reads (getter or `config$` + async pipe): `footer.component.ts`, `header.component.ts`, `home-page.component.ts`, `about.component.ts`, `getting-started.component.ts`.
- [ ] Specs: `site-config.service.spec.ts` (`applyContent` merge rules + emission), `footer.component.spec.ts` (email/address/tagline update after an emission — the regression test for the reported bug), touched-consumer specs updated.

### 1.2 Fonts admin portal — the font service persists
- [ ] [hw] `manage_site_fonts.cpp` GET: emit `source_key` on each family (resolve `font_source_id` → `site_font_sources.source_key`); test asserts a cdn family round-trips its key. Pin bump with the next hw batch (2.x rides the same bump).
- [ ] [app] `site-fonts.component.ts` `load()`: also map `font_source_id` → `source_key` from the already-present `sources` array (covers the payload from any not-yet-bumped server, and is the self-contained fix).
- [ ] [app] `addFamily()`: preselect the source when exactly one active font service exists (the "I have to pick Google every time" half).
- [ ] Specs: component spec — a loaded cdn family shows its service selected; a new family defaults to the sole active service; `ServerAccess.mock` families carry `source_key` (+ mock spec).

### 1.3 Alphabetize the portals; portal icons → `--theme-primary`
Tiles are hardcoded `<mat-card>` blocks in three templates; converting to typed arrays + `@for` makes ordering assertable.
- [ ] `manage-dashboard` (31 tiles), `staff-dashboard` (5 + 5 provider tiles — alphabetize within each group), `profile/user.component` (15 tiles): convert each to a typed tile array (icon, title, route/click), sorted alphabetically by title, rendered with `@for`.
- [ ] Icon color: `mat-icon[mat-card-avatar]` (`--theme-info-strong`) and `.card-icon` (`--theme-text-muted`) both become `var(--theme-primary)`. While in there: the `.dashboard-card` rule is duplicated verbatim in manage + staff SCSS — hoist to one shared place.
- [ ] Specs: each dashboard spec asserts the rendered titles are in alphabetical order and every tile navigates; a computed-style assertion pins the icon color to the token.

### 1.4 Booking summary — reclaim the vertical space, highlight the day
- [ ] `service-booking.component.html`: remove the `person` (provider name) and `location_on` (facility name) rows from the Booking Summary card. (Provider/facility remain visible earlier in the flow where the slot is chosen.)
- [ ] Add the missing week-strip state CSS (none of these classes have rules today): `.date-btn.selected` → `--theme-primary` background + `--theme-on-primary` text; `.available` → visible affordance (e.g. `--theme-primary` border/dot); `.unavailable`/`:disabled` → muted.
- [ ] Specs: summary card renders without the two rows; selected-day button carries the selected class; available vs unavailable classes applied per the availability set.

### 1.5 Memberships page: title + price color
- [ ] `catalog.component.html` (`/shop`, what the Memberships menu item opens): `<h1>` "Shop" → **"Memberships"**.
- [ ] Price color: the three per-component `.text-primary { color: var(--theme-info-strong) }` copies (catalog, membership-tier-cards, service-booking) currently paint prices **info-blue**. Change the price color to `var(--theme-on-accent)` per the ask — see **OQ-3** (contrast concern flagged; implementing literally unless overridden).
- [ ] Specs: title text; computed price color pinned to the token in `membership-tier-cards.component.spec.ts` and `catalog.component.spec.ts`.

### 1.6 Upcoming-classes section colors on Home
- [ ] The home section wrapper around `<app-upcoming-classes [embedded]="true">` gets `background: var(--theme-surface-tint)`; the embedded day/class cards get `background: var(--theme-background)` (embedded mode only — the standalone `/my/my-schedule` page keeps its current look).
- [ ] Specs: computed backgrounds asserted in `home-page.component.spec.ts` / `upcoming-classes.component.spec.ts`.

### 1.7 Instructors: favorites instruction + the "preferences" answer
- [ ] Public `/instructors` page: add a `.hint` line under the `<h1>` — logged-in: "Tap the heart on an instructor to follow them — we'll email you when they're newly scheduled to teach a class."; logged-out variant: "Sign in to follow instructors and get an email when they're newly scheduled." Spec for both states.
- [ ] **Answer to "what do preferences do?"** — today, *nothing*. `instructor_class_preferences` (per-class Min/Max attendees + Notes, edited in Manage → Instructors → edit panel) is stored and displayed back but **no scheduling, booking, capacity, or roster code reads it**; the min-attendee auto-cancel sweep that actually runs reads `class_series_instances.min_attendees` (set per series run) instead. It is write-only staff documentation.
- [ ] Pending OQ-5: add an explanatory hint to the preferences panel ("Reference notes for schedulers — not currently enforced anywhere.") so the UI stops implying enforcement. (If OQ-5 says wire-it-in or delete-it, that replaces this item.)

### 1.8 Menu: Our Classes / My Classes (interpretation — see OQ-2)
Chosen reading: one auth-dependent menu item.
- [ ] `mockHeaderResponse.ts`: **logged-out** → item labeled **Our Classes** targeting `/classes/all` (the All Classes catalog); **logged-in** → item labeled **My Classes** targeting `/classes` (the weekly page, unchanged behavior — it already defaults to eligible-only for members).
- [ ] `class-info.component` (All Classes): add a "Weekly schedule" link → `/classes` so anonymous visitors can still reach the advertisement schedule (today's reverse link only goes to the calendar).
- [ ] Specs: `mockHeaderResponse.spec.ts` per auth state; the new link in `class-info.component.spec.ts`.

### 1.9 Live hand-testing (Phase 1)
- [ ] Steps written when the phase lands: fresh DB, change the contact email in Manage → Site Theme → Brand basics and watch the footer update **without reloading**; reopen Manage → Fonts and confirm Google stays selected on the Roboto/Barlow rows; walk the three portals for alphabetical order and red icons; book a service far enough to see the summary card and the highlighted day; open Memberships from the menu and read the title; check the home upcoming-classes band colors; read the instructors hint; check the menu in both auth states.

---

## Phase 2 — Theme typography: units and weight roles

> The "How text looks" upgrades. Backend first ([hw] — one pin bump covers 1.2 + all of Phase 2).

### 2.1 [hw] Server-side groundwork
- [ ] `IsValidCssLength`: accept `pt` alongside `px|rem|em|%` (+ tests: `12pt` accepted, junk still refused).
- [ ] Register three **role** weight tokens in `SiteThemeTokens()`: `site_theme_weight_menu` → `--weight-menu`, `site_theme_weight_heading` → `--weight-heading`, `site_theme_weight_display` → `--weight-display` (`ThemeTokenType::Weight`, FontRole group, plain-English descriptions: "Menu items", "Headings and bold text", "Display / condensed titles"). Tests: registry entries present, weight validation applies, `site_info` serves an override.

### 2.2 [app] Unit toggle in "How text looks"
- [ ] A `rem / px / pt` `mat-button-toggle` at the top of the section (default rem). Conversion at 1rem = 16px = 12pt. The six `--text-*` rows display their resolved value converted to the selected unit; typing in px/pt mode writes the value in that unit (all three are now server-valid); placeholders show the converted default.
- [ ] Weight rows are unit-less — the toggle skips them (they get the 2.3 control instead).
- [ ] Specs: display conversion for each unit; entering `18px`/`13.5pt` round-trips; rem mode unchanged from today.

### 2.3 [app] Named weight picker + role wiring
- [ ] `_tokens.scss`: add `--weight-menu: var(--weight-semibold)`, `--weight-heading: var(--weight-semibold)`, `--weight-display: var(--weight-semibold)` defaults. Re-point the hardcodes: `.header-button` (`header.component.scss`) → `var(--weight-menu)`; `.din-bold` → `var(--weight-heading)`; `.din-condensed-bold` → `var(--weight-display)`. Nothing moves visually (defaults preserve semibold).
- [ ] Site Theme editor: fields with `type === 'weight'` render a `mat-select` instead of the bare input — options 100–900 with names (reuse the existing `FACE_WEIGHTS` list: "100 — Thin" … "900 — Black") plus "Use the default (600 — Semi Bold)" resolved from the stylesheet, mirroring the font-role dropdown's naming of defaults. Applies to the three new role weights **and** the four existing scale weights.
- [ ] Fix the specimen drift: the "How text looks" specimen uses `--weight-bold` where app chrome uses semibold — re-point the specimen at the role weights so the preview tells the truth.
- [ ] Specs: weight fields render selects with named options; picking 300 for "Menu items" restyles a probe styled with `var(--weight-menu)`; `design-tokens.spec.ts` pins the three new role tokens and the re-pointed classes.

### 2.4 Live hand-testing (Phase 2)
- [ ] Steps: Manage → Site Theme → Fonts → How text looks — flip the unit toggle and read `16px` / `12pt` for the base size; set "Menu items" to "300 — Light" and watch the top menu thin out immediately; export a theme file and confirm the weight keys are in `theme.json` (the registry-driven coverage guard makes this automatic — the step proves it end-to-end).

---

## Phase 3 — Home page: one ordered list of sections

> The architectural core. Everything on the home page between the announcements strip (Phase 6) and the footer becomes a `home_sections` row: position is data, visibility is data. Also delivers the D15 leftover from [[Tenant Theming and Branding]] Phase 6B (per-kind components).

### 3.1 [app] Schema + migration
- [ ] `db_schema/home_sections`: new kind constants — `carousel`, `membership`, `upcoming_events`, `upcoming_classes`, `offerings`, `intro`, `get_started`, `video` (joining `hero|feature|banner|artwork`). New columns: `image_carousel_id` BIGINT nullable (FK added in Phase 4; plain nullable column now), `video_url` STRING nullable, `hidden_when_logged_in` BOOL default false, `hidden_when_member` BOOL default false.
- [ ] Migration `app/0001_home_sections_functional_kinds` (the first app migration — retire `AppStreamEmptyPreDeploy` per its own comment): guarded `ALTER TABLE … ADD COLUMN IF NOT EXISTS` for the four columns, then **insert the functional rows if absent by kind** — existing databases (Mason's dev DB) must keep their membership/events/classes sections when the hardcoded islands are deleted. Seed ordinals reproduce today's exact order: carousel 2 · hero 5 (exists) · intro 6 · get_started 7 · upcoming_events 8 · offerings 9 · features 10/20/30 (exist) · banner 50 (exists) · membership 60 (`hidden_when_member = true` — this replaces the `showTiers` logic) · upcoming_classes 70.
- [ ] `create_database.cpp` `PopulateHomeSections`: seed the same functional rows for fresh DBs.
- [ ] Table helper: emit the new columns; tests for column round-trip and kind constants.
- [ ] Admin metadata seeds: column labels/hints for the new columns.

### 3.2 [app] Per-kind section components (the D15 extraction)
- [ ] Split `home-page.component.html`'s inline markup into standalone components: `home-hero-section`, `home-feature-section`, `home-banner-section`, `home-artwork-section` (extractions), plus functional hosts `home-intro-section`, `home-get-started-section`, `home-events-section` (wraps all three event strips with the existing auth branching), `home-offerings-section`, `home-membership-section`, `home-upcoming-classes-section` (carries the 1.6 colors), and stubs for `home-carousel-section` (Phase 4) / `home-video-section` (Phase 6). Each takes its `HomeSection` row as input; public page and (later) the Page Content preview render the same components — a preview that cannot drift.
- [ ] `HomePageComponent`: **one `@for`** over the ordered, visibility-filtered section list with a `@switch` on `kind`. The three separate kind loops and all hardcoded islands are deleted.
- [ ] Specs: each extracted component gets its own spec (moved assertions); the page spec asserts a mixed-ordinal list renders in ordinal order regardless of kind (the bug the old template had), and that today's seeded order reproduces today's page.

### 3.3 [app] Types + seam
- [ ] `HomeSectionKind` union + new fields in `home.types.ts`; `ServerAccessNetwork` normalization (Postgres `"t"/"f"` booleans for the two flags — the known trap); mock rows updated to the new seed; mock spec.

### 3.4 [app] Visibility flags
- [ ] Filter in the page: hide a row when (`hidden_when_logged_in` && logged in) or (`hidden_when_member` && `hasMembership`) — membership via the existing `getSubscriptions()` derivation already on the page. Flags apply to **every** kind (a feature block can be members-hidden too).
- [ ] Specs: flag matrix across auth/membership states; membership row disappears for a member (regression for the old `showTiers`).

### 3.5 [app] Page Content editor
- [ ] Kind select grows the new kinds, grouped ("Content blocks" vs "Built-in sections"); conditional fields — functional kinds hide title/body/link where unused, `video` shows a URL field (validated in Phase 6), `carousel` shows the picker (Phase 4); every row gets the two visibility toggles.
- [ ] Row list renders functional kinds with a descriptive label + icon instead of a photo thumb.
- [ ] Specs: add/edit each new kind; toggles persist.

### 3.6 [app] Richer event cards (part of item 19)
- [ ] Extract a shared `event-session-card`-style public card from `/events` (name, **product description**, time, facility + room, spots remaining, price, image slot for Phase 5, Book Now / badges) and use it for all three home event strips — home now matches the Upcoming Events page instead of the truncated inline cards.
- [ ] Specs: home strips render description/room/spots; `/events` page unchanged by the extraction.

### 3.7 [app] Theme bundle
- [ ] `page_content_bundle_section.cpp`: export/import the new columns and kinds (`IsKnownHomeSectionKind` grows; `video_url`, flags, and — in Phase 4 — the carousel reference travel). Round-trip tests: a bundle with functional kinds + flags is byte-identical; an unknown kind still refuses.

### 3.8 Live hand-testing (Phase 3)
- [ ] Steps: fresh DB looks **identical** to today (the whole phase is order-preserving); then in Manage → Page Content move "Memberships" above the features and reload Home to see it moved; toggle "Hide for members" off and see tiers as a member; export a theme, wipe, import, and confirm the arrangement survives.

---

## Phase 4 — Named image carousels

> Depends on Phase 3 (the `carousel` kind renders through the unified list). Replaces the single random bag with named, ordered carousels; adds the Gallery page and the Site Theme tab; carousels join the theme bundle (item 7d).

### 4.1 [app] Schema
- [ ] `db_schema/image_carousels`: `image_carousel_id` PK, `name` STRING UNIQUE (stable handle for bundle references), `display_name`, `description` nullable, `show_title` BOOL default true, `show_description` BOOL default true, `show_on_gallery` BOOL default false, `gallery_ordinal` INT default 0, `active` BOOL default true, timestamps.
- [ ] `db_schema/image_carousel_photos`: `image_carousel_photo_id` PK, `image_carousel_id` FK, `title` nullable, `description` nullable, `ordinal` INT default 0, timestamps. Photos attach via the photo association (register in `photo_support_tables` + the anonymous `PublicPhotoTables` list in `main.cpp` — without the latter, visitors see broken images).
- [ ] Full new-table checklist ×2 (top-level `image_carousels`, nested `image_carousel_photos`), CMakeLists, friendly names, display template `{display_name}`.

### 4.2 [app] Migration `app/0002_image_carousels`
- [ ] Create both tables if absent; insert a **"home"** carousel (display_name "Home page photos") if absent; copy `home_page_photos` rows into `image_carousel_photos` **preserving ids** (then fix the sequence) so `UPDATE table_item_photos SET table_name = 'image_carousel_photos' WHERE table_name = 'home_page_photos'` re-points every photo without an id map; swap the `photo_support_tables` row; point the Phase-3 carousel home row's `image_carousel_id` at "home"; drop `home_page_photos`. Idempotent at every step; test drops/recreates to prove a pre-migration DB converts and a fresh DB no-ops. (This migrates Mason's live photos — see OQ-10.)
- [ ] Delete the old surface end-to-end (no dead code): `home_page_photos` db_schema/helper/endpoint/`getHomePagePhotos` seam + mock + tests, admin metadata seeds, `PublicPhotoTables` entry.

### 4.3 [app] Helpers + endpoints
- [ ] `TableHelpers::ImageCarousels` (DbCrud CRUD + `GetActiveOrderedForGallery`, `GetPhotosForCarousel` ordered by `ordinal, id`).
- [ ] `GET /api/image_carousel/<id>` (anonymous: carousel row + ordered photos) and `GET /api/gallery_carousels` (anonymous: `show_on_gallery` carousels by `gallery_ordinal`, each with photos). Writes ride generic CRUD + the existing photo upload endpoints. Anchor in `web_app.cpp`; endpoint tests (anonymity, ordering, inactive filtered).

### 4.4 [app] The carousel manager — "Image carousels" tab in Site Theme
Per item 7c it lives in `/manage/site-theme` (union + `sectionOrder` + `sectionKeys()` switch + a `mat-tab`; section Save hidden like the Theme file tab — this tab saves through its own flow).
- [ ] Carousel list: add / rename / describe / activate / delete (confirm), reorder `gallery_ordinal`, toggle `show_on_gallery`, `show_title`, `show_description`.
- [ ] Per-carousel photo panel: upload (`hw-photo-upload` per row, same save-then-attach flow as Page Content), remove, reorder, edit title/description per photo, thumbnails via scaled-photo URLs.
- [ ] Specs: CRUD flows, reorder persists, toggles persist, photo list renders thumbs.

### 4.5 [app] Home `carousel` kind + component polish (items 7, 7b)
- [ ] `home-carousel-section` renders the row's `image_carousel_id` through the (renamed) `image-carousel` component; the Page Content editor's carousel field is a select over active carousels (`get_fk_options`).
- [ ] Component fixes: **variable-width cards** — uniform row height, each image at its natural aspect ratio (`height: <fixed>; width: auto`, scaled request sized by height) so wide images are wide (item 7's cropping ask); captions render title **and** description (description is fetched-but-never-shown today) gated by the carousel's `show_title`/`show_description`; fix the 3-visible-cards `maxOffset` assumption; dedupe the TS/SCSS width constants.
- [ ] Specs: aspect behavior, caption gating, nav bounds with mixed widths.

### 4.6 [app] Gallery page
- [ ] New public route `/gallery` rendering `gallery_carousels`: each carousel's display name (if `show_title`), description (if `show_description`), and its image-carousel. Menu: About ▸ Gallery → `/gallery` (replacing the `/#gallery` anchor; the home anchor id can go). Empty state when nothing is flagged for the gallery.
- [ ] Specs: ordering, flag gating, empty state, menu target.

### 4.7 [app] Theme bundle (item 7d)
- [ ] `page_content` section gains `image_carousels`: per carousel — name, display_name, description, flags, photos (title/description + image assets named `carousel-<n>-<m>.<ext>`), gallery order by array position; home `carousel` rows reference the carousel **by name** (no ids in bundles). Import replaces wholesale, validates every asset. Round-trip byte-identical test + photo-survives test.

### 4.8 Live hand-testing (Phase 4)
- [ ] Steps: create "Aerial shots" in Manage → Site Theme → Image carousels, upload three photos with captions, reorder them; flag it for the gallery and open About → Gallery; add a second home carousel row in Page Content positioned below the features; export/import a theme and confirm carousels and images survive.

---

## Phase 5 — Product images: services, events, membership tiers

> One mechanism serves all three: `products` joins the photo system (one photo per product).

### 5.1 [app] Products join the photo system
- [ ] `photo_support_tables` += `products` (seed + migration `app/0003_products_photo_support` inserting the row if absent); `PublicPhotoTables` += `products` (services/events/tiers render for visitors).
- [ ] Product editor (`product-detail.component`): a photo card with `hw-photo-upload` (table `products`) for every kind; product create keeps the save-then-attach flow.
- [ ] `CatalogProduct` type + payload: expose `has_photo` (and confirm `product_id` reaches every consumer needing a URL — `visible_event_sessions` payload gains `product_id`/`has_photo` if absent). Backend tests for the payload additions.

### 5.2 [app] Event images (item 19)
- [ ] The shared public event card (3.6) renders the product photo above the copy (placeholder-free fallback when absent) — on `/events` and all home event strips.
- [ ] Specs: card with/without photo on both surfaces.

### 5.3 [app] Service images (item 18)
- [ ] Services page (`service-catalog`): image above each service card from the product photo; graceful absence.
- [ ] Specs: with/without photo.

### 5.4 [app] Membership tier icons (item 12 — finishes theming Phase 3's leftover)
- [ ] Seed: `Seed::PopulateSeedPhotos` attaches `img/tier_icon_solo.png` / `tier_icon_couple.png` / `tier_icon_family.png` to the three membership products (solo → Gold, couple → Couple's, family → Family). (Only three tier icons exist — see OQ-4.) Existing DBs: covered by 5.1's registration + a re-runnable seed step or manual upload through the new editor field — noted in hand-testing.
- [ ] `membership-tier-cards`: render the product photo as the tier icon, falling back to today's `card_membership` Material icon.
- [ ] Specs: icon renders from photo, fallback intact.

### 5.5 [app] Home membership panel = Memberships page (item 12a)
- [ ] Upgrade the **shared** `membership-tier-cards` (icon image, name, description, price in the 1.5 color, Subscribe) and consume it on `/shop` for subscription products — replacing the raw-Tailwind catalog cards for that kind (other kinds keep the generic card). Home and the Memberships page now render byte-identical tiers.
- [ ] Specs: `/shop` renders subscriptions through the shared card; non-subscription kinds unaffected.

### 5.6 Live hand-testing (Phase 5)
- [ ] Steps: upload a photo to "Intro Workshop" in Manage → Products and see it on Upcoming Events + the home events section, signed out; upload to a massage service and check Services; fresh DB shows the three laurel tier icons on Home and Memberships, identically.

---

## Phase 6 — Announcements and video

### 6.1 Announcements (item 25)
- [ ] [app] `db_schema/announcements`: `announcement_id` PK, `title`, `body` (plain text — OQ-6), `show_from_us` BIGINT, `show_until_us` BIGINT, `active` BOOL default true, `ordinal` INT default 0, timestamps. Full checklist + migration `app/0004_announcements`. **Deliberately NOT in the theme bundle** — announcements are timely content, not a look (stated here so the round-trip guard's exclusion is intentional).
- [ ] [app] Helper + `GET /api/announcements` (anonymous: `active` && server-now within `[show_from_us, show_until_us)`, ordered by `ordinal, id`). Endpoint tests: window edges, inactive filtered, anonymity.
- [ ] [app] Editor: an "Announcements" tab in Manage → Page Content — title, body, **date pickers** for the from/until dates (per [[feedback_date_time_pickers]]), active toggle, reorder.
- [ ] [app] Home: an announcements strip pinned **above everything** (top of the page, before the ordered sections) — one `.notice`-styled banner per current announcement. Seam + mock + specs (renders when current, absent otherwise, ordering).

### 6.2 Video home item type (item 26)
- [ ] [app] Uses Phase 3's `video_url` column. Accept YouTube forms (`watch?v=`, `youtu.be/`, `shorts/`, `embed/`) → normalized to a privacy-enhanced `youtube-nocookie.com/embed/<id>` iframe (16:9, `loading="lazy"`, `referrerpolicy`), URL validated client-side in the editor and on bundle import. YouTube only for now — OQ-7.
- [ ] [app] `home-video-section` renders the iframe with the row's title above (optional); editor's `video` kind shows the URL field with validation feedback.
- [ ] Specs: URL-form parsing table, invalid URL refused in the editor, section renders the embed; bundle round-trips `video_url`.

### 6.3 Live hand-testing (Phase 6)
- [ ] Steps: create an announcement spanning today and see it atop Home (signed in and out); set the window to end yesterday and see it vanish; add a video row with a real YouTube link positioned mid-page and play it.

---

## Phase 7 — Locations, timezone, and the About page

### 7.1 Studio-location timezone (item 5) — expose what exists
`facilities.timezone` already exists (IANA, default `America/Los_Angeles`) and drives booking emails/digests/reporting — recorded in [[Bookable Service Foundation]] (decision 10). The gap is surfacing:
- [ ] [app] Manage → **Locations** page (new; locations are currently editable only through the debug Manage Data surface, violating the dedicated-UI rule): list + edit for `facilities` — name, code, full address (including the `address_line_2`/`country` fields the CRUD metadata omits), **timezone** as a searchable select of IANA zones (curated common-US list + free entry), active toggle. Modeled on `instructors-admin`. Dashboard tile (alphabetized per 1.3).
- [ ] [app] `description` STRING nullable column on `facilities` (+ DSL + migration `app/0005_facilities_description`) and photo support (`photo_support_tables` += `facilities`, `PublicPhotoTables` += `facilities`, `hw-photo-upload` in the editor) — both feed 7.2.
- [ ] Recorded, not planned: the service-booking week strip builds days in **browser-local** time (`buildWeek()` uses `new Date()`; `setHours(0,…)`) — fine while all facilities are Pacific, wrong the day a second-timezone location exists. Noted here as the follow-up the timezone column exists to serve.
- [ ] Specs: page CRUD, timezone select persists, description/photo round-trip.

### 7.2 About ▸ Our Location (item 20)
- [ ] [app] Public `/location` page: each active facility — photo, name, description, full address, and an **"Open in Google Maps"** button (`https://www.google.com/maps/search/?api=1&query=<url-encoded address>`, new tab — link-out, not an embedded iframe; OQ-9). Menu: About ▸ **Our Location**.
- [ ] Specs: renders active facilities, map href encodes the address, inactive hidden, menu entry.

### 7.3 About page → alternating blocks (item 14)
- [ ] [app] `db_schema/about_sections`: same shape as a feature row (`about_section_id`, `ordinal`, `title`, `body`, `link_route`, `link_label`, `active`, timestamps) + photo association (+ public allow-list). Full checklist + migration `app/0006_about_sections`.
- [ ] [app] Helper + `GET /api/about_sections` (anonymous, ordered, active).
- [ ] [app] Manage → Page Content gains an "About sections" list (same editor pattern; reorder/add/remove/photo).
- [ ] [app] About page: heading, then `site_about_markdown` rendered as an intro **when non-empty** (keeps Mason's existing content — OQ-8), then the ordered blocks through the *same* `feature-section` component Phase 3 extracted (alternating sides). The `graySquare` placeholder dies.
- [ ] [app] Theme bundle: `about_sections` joins the `page_content` section (rows + image assets, order = array position). Round-trip tests.
- [ ] Specs: alternation, markdown-intro gating, reorder reflected, bundle round-trip.

### 7.4 Live hand-testing (Phase 7)
- [ ] Steps: Manage → Locations — set the studio's timezone and description, upload a photo; About → Our Location shows it and the button opens Maps on the address; add two About blocks in Page Content and see them alternate; export/import a theme and confirm About blocks survive.

---

## Phase 8 — Figma assets and the visual bands

> 8.2 and 8.3 are **blocked on OQ-1** (restore `~/.figma_token`). 8.1 and 8.4 are not.

### 8.1 [app] Get Started band as a kind row (prep, unblocked)
- [ ] Phase 3 made `get_started` a row; this restyles its component to Ryan's band: full-bleed, `--theme-inverse-surface` background + `--theme-on-inverse-surface` text (the "menu color scheme"), stacked and centered: image (the row's attached photo, when present) → copy (existing auth-variant text) → red CTA → `/start`. Renders image-less until 8.2 supplies the asset.
- [ ] Specs: band colors from tokens, stacking, CTA target, image-optional.

### 8.2 [app] The fire-font "Get Started" image (item 11)
- [ ] Export `Icon/GetStarted` (Frame 127) from the Figma file via the REST API into `src/database_helper/img/`; seed it as the `get_started` row's photo (`AttachSeedPhoto`) — per-tenant replaceable through Page Content like every section image; existing DBs get it via the editor's photo upload (hand-testing step).
- [ ] Spec: seeded fresh-DB row carries a photo (seed test), band renders it.

### 8.3 [app] Footer tagline as an image (item 13)
- [ ] [hw] New `url`-type content slot `site_tagline_image_url` in `SiteContentSlots()` (registry-driven: `site_info`, the editor, and the theme bundle pick it up automatically). Framework default `""`.
- [ ] [app] Export the "That which doesn't kill you makes you hotter" artwork from Figma → seed into `site_assets` (`tagline.png`) + KY default value `/api/site_asset/tagline.png` in `app_secret_values.cpp`.
- [ ] [app] Footer: when the slot is non-empty render the image (alt = the tagline text lines); else today's italic text. Editable in Site Theme → Brand basics.
- [ ] Specs: both footer modes; slot round-trips in a theme bundle with its asset.

### 8.4 [app] Parallax banner (item 10)
- [ ] A `hwParallax` directive on the banner-section image: transform-based (`translateY` at a fraction of scroll), driven by the **app shell's scroll container** (the `overflow-y-scroll` div in `app.component.html` — window scroll never fires here; the container gets a template ref/service handle), `IntersectionObserver`-gated, inert under `prefers-reduced-motion`. Applied to the `banner` kind ("Come join the fun!"); no schema.
- [ ] Specs: transform updates on container scroll, disabled under reduced motion.

### 8.5 Live hand-testing (Phase 8)
- [ ] Steps: fresh DB — the Get Started band is black with the fire-font image stacked over the copy and button, full width; the footer shows the tagline artwork; scrolling Home moves the join-the-fun photo slower than the page.

---

# Open Questions

> Answer inline with `- Mason-` bullets. Only OQ-1 blocks work (Phase 8.2/8.3); everything else has a chosen default I'll run with.

- **OQ-1 — Figma access is gone.** Previous sessions read Ryan's file via the REST API with a personal-access token at `~/.figma_token`; that file no longer exists. To unblock the Get Started image and the footer tagline export, either recreate `C:\Users\mason\.figma_token` containing the token (one line), or export the two assets yourself into `server/knottyyoga_server/src/database_helper/img/` (PNG, 2× — `Icon/GetStarted` from Frame 127, and the tagline artwork) and I'll take it from there.
- **OQ-2 — "Our Classes / My Classes" menu shape.** Your sentence supports two readings. **Chosen:** one auth-dependent item — signed out: **Our Classes** → All Classes catalog (`/classes/all`); signed in: **My Classes** → the weekly page (`/classes`, unchanged behavior). The All Classes page gains a "Weekly schedule" link so visitors can still reach the schedule advertisement. **Alternate:** show *both* items when signed in (Our Classes → catalog, My Classes → weekly). Say the word if you want the alternate.
	- Mason- Yes, I want the first alternative.
- **OQ-3 — Price color `--theme-on-accent`.** Implementing literally as asked. Flag: `on-accent` is the text-*on*-an-accent-fill token (white-ish for KY's amber accent) — on a white card it may be near-invisible. If the intent was "accent-colored prices," the token is `--theme-accent`. I'll implement `--theme-on-accent` and you can eyeball it; one word flips it.
	- Mason- --theme-on-accent is actually #1b0e00 which is nearly black (which is the goal)
- **OQ-4 — "the other three images."** Only **three** tier icons exist (`tier_icon_solo/couple/family.png`) — solo plus two others. If there's a fourth image you meant, name it.
	- Mason- Just the three
- **OQ-5 — Instructor class preferences.** They do nothing today (write-only notes; details in Phase 1.7). Options: **(a)** keep + label as "reference notes, not enforced" *(chosen default)*; **(b)** wire min/max into scheduling/capacity (real feature work — happy to scope); **(c)** delete the surface. Pick one.
- **OQ-6 — Announcement body.** Chosen: plain text (title + body + date window), styled as a notice banner, not dismissible. Alternates: markdown body, or per-user dismissal (needs storage). Speak up if wanted.
- **OQ-7 — Video providers.** Chosen: YouTube only (privacy-enhanced nocookie embed). Vimeo or raw-URL `<video>` can be added later behind the same column.
- **OQ-8 — About page markdown.** Chosen: the existing `site_about_markdown` renders as an intro above the new blocks when non-empty, so your current About copy survives. Clear the slot in Site Theme when you want blocks only. (Alternate: drop the markdown entirely.)
- **OQ-9 — "Triggers a map."** Chosen: an "Open in Google Maps" link-out built from the address (no API key, no third-party iframe/CSP surface). Alternate: an embedded map (iframe embed or Maps JS — needs a key + CSP allowance).
- **OQ-10 — Carousel data migration touches your live DB.** Phase 4.2 moves your existing `home_page_photos` rows (ids preserved, photo associations re-pointed) into the new carousel tables as a "Home page photos" carousel, then drops the old table + endpoint. It's idempotent and tested against both DB states, but it is a one-way conversion of real data — flagging it rather than doing it silently.
- **OQ-11 — Which "admin portal" to alphabetize.** Chosen: the Manage dashboard (31 tiles), Staff Portal, and the personal profile tiles. The `/admin` Manage Data page has no tiles (it's a table dropdown from `@honuware/ui`) — left alone as the debug surface.
- **OQ-12 — Timezone scope.** `facilities.timezone` already exists and is consumed server-side; chosen scope is the editing surface + validation + the new Locations page (7.1), with the browser-local week-strip gap recorded as a follow-up. If you meant something more (e.g., rendering all class times in facility TZ on every page), say so and I'll scope it separately.