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
>
> **Update 8/20/2026 — all twelve open questions answered and folded in; the plan is execution-ready.** Scope changes from the answers: instructor class preferences are **deleted end-to-end** (1.7, which now carries the first app migration — later migration ids renumbered), the video kind supports **raw video-file URLs** as well as YouTube (6.2), the About page drops markdown **entirely** (7.3, including the Site Theme editor's About tab), and the Manage Data table dropdown joins the alphabetize pass (1.3). OQ-1 (the Figma token) is ⏸ parked until Ryan is back — only Phases 8.2/8.3 wait on it.

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
| 16 | What do Instructors-portal preferences do? | A | answered — surface deleted in 1.7 (OQ-5) |
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

> The demo blockers and one-sitting fixes. Mostly frontend; 1.2 and possibly 1.3's dropdown sort have small [hw] halves, and 1.7 now carries a C++ deletion plus the first app migration (OQ-5).

> ✅ **Phase 1 implemented 8/20–8/21/2026.** Gates green: honuware **1733/1733**, knottyyoga C++ **5038/5038** (both Linux docker), `knottyyoga_database_helper` builds explicitly, Angular **3164/3164** + `tsc --noEmit` clean on both configs. Deviations from the plan text are noted inline per subsection.
>
> **Pin bumped 8/21/2026** — honuware `17a818d` ("Phase 1 — Live-refresh, fonts portal, and small polish fixes") is pushed and CI-green; both consumers re-pinned and verified against the **pinned clone** (not the co-dev override): knottyyoga **5038/5038**, communityfinder **1754/1754**.

### 1.1 Footer (and everything else) refreshes when site content is saved
The fix is structural, not a footer patch — five components snapshot `SiteConfigService.config` at construction.
- [x] `SiteConfigService`: expose the config reactively — a `BehaviorSubject<SiteConfig>`/`config$` (getter stays for existing callers), emitted from `applySiteInfo()`. Add a public `applyContent(content: KeyValueTable-shaped map)` that runs the existing slot parsing/merge rules over the current config and emits — the content-slot twin of `applyTheme()`. *(As built: `applyContent` touches only the keys PRESENT in the map — the editor saves one section at a time, so absent means "not part of this save"; present-and-empty falls back to the bundled default, matching the server's reset. A single `commitConfig()` swaps the snapshot, restamps the document title/favicon — both are content slots — and emits.)*
- [x] `SiteThemeComponent.save()`: after a successful PUT, call `applyContent()` with the saved content fields (it already calls `applyTheme()` for tokens). The bundle-apply path already calls `load()` — leave it.
- [x] Convert the stale-snapshot consumers to live reads (getter or `config$` + async pipe): `footer.component.ts`, `header.component.ts`, `home-page.component.ts`, `about.component.ts`, `getting-started.component.ts`. *(All five became getters. Header's `logoUrl` and Home's `heroBadge` cache the sanitized URL per raw value so change detection doesn't re-set the `<img>` src each cycle; their 404-fallback handlers now just set the flag the getter derives from.)*
- [x] Specs: `site-config.service.spec.ts` (7 `applyContent` cases + `config$` emission), `footer.component.spec.ts` (the regression test: save → footer shows the new email/address without re-creation), `site-theme.component.spec.ts` (save hands the section's content slots to `applyContent`), header spec's logo-fallback test rewritten to drive through the service config.

### 1.2 Fonts admin portal — the font service persists
- [x] [hw] `manage_site_fonts.cpp` GET: emit `source_key` on each family (resolve `font_source_id` → `site_font_sources.source_key`); test asserts a cdn family round-trips its key. Pin bump with the next hw batch (2.x rides the same bump).
- [x] [app] `site-fonts.component.ts` `load()`: also map `font_source_id` → `source_key` from the already-present `sources` array (covers the payload from any not-yet-bumped server, and is the self-contained fix).
- [x] [app] `addFamily()`: preselect the source when exactly one active font service exists. *(Found already implemented — `addFamily()` and `onKindChange()` both defaulted to `sources[0]`; the whole reported symptom was the load round-trip. A spec now pins the preselect anyway.)*
- [x] Specs: component spec — a loaded cdn family shows its service selected **including off an id-only payload**; a new family defaults to the sole active service; `ServerAccess.mock` families carry `source_key` + `font_source_id` matching the real payload (+ mock spec case). *(The mock had always carried `source_key` — which is exactly why local mode never showed the bug.)*

### 1.3 Alphabetize the portals; portal icons → `--theme-primary`
Tiles are hardcoded `<mat-card>` blocks in three templates; converting to typed arrays + `@for` makes ordering assertable.
- [x] `manage-dashboard` (31 tiles), `staff-dashboard` (5 + 5 provider tiles — alphabetize within each group), `profile/user.component` (15 tiles): convert each to a typed tile array (icon, title, route/click), sorted alphabetically by title, rendered with `@for`. *(Shared `DashboardTile` + `byTileTitle` in `@shared/types/dashboard-tile.types.ts`; arrays stay authored in feature order and sort before render. The profile's fifteen `onX()` methods collapsed to one `onTile()`; its `card-*` element ids survive.)*
- [x] Icon color: `mat-icon[mat-card-avatar]` (`--theme-info-strong`) and `.card-icon` (`--theme-text-muted`) both become `var(--theme-primary)`. *(Hoist: the duplicated manage/staff rule became a global `.portal-card` in `_patterns.scss` — a new name because the profile hub uses `.dashboard-card` for a different grid style; the redundant per-card border dropped since the surfaces layer already provides it.)*
- [x] Specs: manage (31 titles alphabetical + every tile routes under /manage + icon color pinned via a token probe), staff (both groups alphabetical), profile (15 tiles alphabetical, click-navigation cases kept on the `card-*` ids).
- [x] **Manage Data table dropdown too** (OQ-11): alphabetized by displayed friendly name. *(Landed app-side, no [hw]: the picker is knottyyoga's own `admin.component.ts` — only the inner table CRUD comes from `@honuware/ui` — so `loadDbTables()` sorts `root_tables` by `getFriendlyName`, which also makes the existing "navigate to first table alphabetically" comment true for the first time. Spec covers friendly-name order including the no-friendly-name fallback.)*

### 1.4 Booking summary — reclaim the vertical space, highlight the day
- [x] `service-booking.component.html`: remove the `person` (provider name) and `location_on` (facility name) rows from the Booking Summary card. (Provider/facility remain visible earlier in the flow where the slot is chosen.)
- [x] Add the missing week-strip state CSS (none of these classes had rules): `.date-btn.selected` → `--theme-primary` background + `--theme-on-primary` text; `.available` → primary border; `.unavailable`/`:disabled` → muted surface-subtle. Selected wins over the other states.
- [x] Specs: summary card asserts the two icon rows and their values are gone (and the old positional test updated); selected-day background pinned to the primary token via a probe.

### 1.5 Memberships page: title + price color
- [x] `catalog.component.html` (`/shop`, what the Memberships menu item opens): `<h1>` "Shop" → **"Memberships"**.
- [x] Price color → `var(--theme-on-accent)` (OQ-3 ✅ — KY's on-accent resolves to `#1b0e00`, the near-black ink, which is the goal). *(Found FOUR copies, not three: catalog, membership-tier-cards, product-detail, and service-booking — whose summary total used the class with **no rule at all**, so it now gets one too. All four aligned.)*
- [x] Specs: title text in `catalog.component.spec.ts`; computed price color pinned to the token in `membership-tier-cards.component.spec.ts`.

### 1.6 Upcoming-classes section colors on Home
- [x] The home section wrapper around `<app-upcoming-classes [embedded]="true">` gets `background: var(--theme-surface-tint)` (new `home-section--tinted` modifier, padded and rounded); the embedded day/class cards get `background: var(--theme-background)` and the sticky day labels blend with the tinted band — embedded mode only, via a `[class.embedded]` host binding + `:host(.embedded)` rules, so the standalone `/my/my-schedule` page keeps its current look.
- [x] Specs: the home spec asserts the section carries the tinted modifier; the upcoming-classes spec asserts the host class toggles with the input and the embedded row background resolves to the token.

### 1.7 Instructors: favorites instruction + delete the unused class-preferences surface
- [x] Public `/instructors` page: add a `.hint` line under the `<h1>` — logged-in: "Tap the heart on an instructor to follow them — we'll email you when they're newly scheduled to teach a class."; logged-out variant: "Sign in to follow instructors and get an email when they're newly scheduled to teach a class." Specs for both states.

**Answer to "what do preferences do?"** — today, *nothing*. `instructor_class_preferences` (per-class Min/Max attendees + Notes, edited in Manage → Instructors → edit panel) is stored and displayed back but **no scheduling, booking, capacity, or roster code reads it**; the min-attendee auto-cancel sweep that actually runs reads `class_series_instances.min_attendees` (set per series run) instead. Per **OQ-5 ✅ ("Let's delete")**, the whole surface goes — top-down so nothing is ever caller-less:
- [x] Pre-check: repo-wide search confirmed the only consumers are the panel, the seam, the four endpoints, the helper, and their tests — 23 server files + 10 UI files, nothing else.
- [x] Frontend: deleted `instructor-class-preferences.component.*` (4 files incl. spec) and its mount in `instructors-admin.component.html`; the four seam methods across interface / network / proxy / mock, their 3 `ServerAccess.mock.spec.ts` cases, the `normalizePreference` normalizer, the mock state, and the 3 request/row interfaces in `specialty-cost.types.ts`.
- [x] Backend: deleted the four endpoint pairs + their `web_app.cpp` anchors + includes, `endpoints/CMakeLists.txt` entries (8 source + 4 test), the 4 endpoint tests; `TableHelpers::InstructorClassPreferences` + test + CMakeLists entries; `db_schema/instructor_class_preferences.{h,cpp}` + test + CMakeLists entries and every registration in `make_app_tables.cpp` / `CreateTables()` / allowed tables / table permissions / column data info / friendly names / table description / display template. *(Two of those seed lines were merged two-statements-on-one-line — unmerged while deleting.)*
- [x] Migration `app/0001_drop_instructor_class_preferences` — **the first real app migration**: `DROP TABLE IF EXISTS instructor_class_preferences`, IF EXISTS as the idempotence guard (the table name is a string literal since its DbSchema constants are gone). New `app_migrations_test.cpp` (2 tests: a pre-1.7 database converts — table created raw, dropped, verified via `DbMeta::ListTables`; a fresh database no-ops, twice). `AppStreamEmptyPreDeploy` retired per its own comment, replaced by `AppStreamStartsWithTheDropPreferencesMigration` so the per-stream tests can't pass vacuously.
- [x] Gates: honuware 1733, knottyyoga C++ 5038, `knottyyoga_database_helper` builds, Angular 3164 — all green with the surface gone; `instructors-admin` spec updated (spy removed).

### 1.8 Menu: Our Classes / My Classes (OQ-2 ✅ — the first option confirmed)
One auth-dependent menu item:
- [x] `mockHeaderResponse.ts`: **logged-out** → item labeled **Our Classes** targeting `/classes/all` (the All Classes catalog); **logged-in** → item labeled **My Classes** targeting `/classes` (the weekly page, unchanged behavior — it already defaults to eligible-only for members).
- [x] `class-info.component` (All Classes): "Weekly schedule" link → `/classes` beside the calendar link, so anonymous visitors can still reach the advertisement schedule.
- [x] Specs: `mockHeaderResponse.spec.ts` per auth state (visitor sees Our Classes → catalog and no My Classes; member sees My Classes → weekly and no Our Classes; exactly one classes entry either way); `header.service.spec.ts` updated to the visitor expectation; the new link in `class-info.component.spec.ts`.

### 1.9 Live hand-testing (Phase 1)
- [x] Steps written (below) — awaiting your run against a live server.

Fresh database (`knottyyoga_database_helper --recreate_database`), server + Angular dev server running. Sign in as the admin (Mason's seeded account).

1. **Footer live-refresh (the headline fix).** Top menu → **Admin** ▸ **Manage Products** → **Site Theme** card → **Brand basics** tab. In **Contact email**, replace `info@knottyyoga.com` with `hello@knottyyoga.com` and press **Save this section**. **Without reloading**, scroll to the page footer: the **Email:** line already reads `hello@knottyyoga.com` and opens a mail client addressed to it. Change it back and save; the footer follows again.
2. **Fonts service persists.** Manage Products → **Fonts** card. On the **Your fonts** tab, the **Roboto** and **Barlow** rows both show **Font service: Google Fonts** already selected — no blank picker. Press **Add a font**: the new row starts with **Google Fonts** preselected. (Before this fix every visit required re-picking Google per row.)
3. **Portals alphabetized, icons brand-red.** Open **Manage Products**: the tiles run **Attendance Templates → Bundles → … → Who's Teaching What**, alphabetically, every tile icon in the brand red. Top menu **Staff**: **Check-In, Class Check-In, Event Sessions, Person Skills, Student Notes** (then, as admin, the five provider tiles alphabetized after). Account menu (your name) → **Profile**: fifteen tiles from **Attendance History** to **Workshops & Series**, icons red.
4. **Manage Data dropdown.** Admin ▸ **Manage Data** → open **Select Table**: the entries are alphabetical by their friendly names.
5. **Booking summary.** Menu **Services** → pick **60 Minute Massage** → **Book Now** → choose a day on the week strip — the chosen day fills solid brand-red with white text, available days carry a red border, unavailable days are greyed. Pick a time and variant to reach the summary: the **Booking Summary** card shows just the service line and the schedule line — no lone person icon, no map pin — and the total price in near-black ink.
6. **Memberships title + prices.** Top menu **Memberships**: the page is headed **Memberships** (not "Shop") and the tier prices render in near-black, not blue.
7. **Home upcoming-classes band.** Signed in (with the membership from the seed or a purchased one), open **Home** and scroll to **Upcoming classes**: the section sits on a light grey tinted band and each class row is a white card on it. Open **Profile → My Schedule** and confirm the standalone page looks unchanged (no band).
8. **Instructors hint.** Top menu **About** ▸ **Instructors**: under the title, "Tap the heart on an instructor to follow them — we'll email you when they're newly scheduled to teach a class." Sign out and revisit: the line now starts "Sign in to follow instructors…".
9. **Preferences gone.** Manage Products → **Instructors** → edit **Mason**: the edit panel ends at the bio Save/Cancel — **no Class preferences section**. In Manage Data's table dropdown there is no "Instructor Class Preferences" entry.
10. **Menu in both states.** Signed out, the top menu shows **Our Classes**, which opens **All Classes**; that page's header has **Weekly schedule** (→ the weekly page) beside **View the calendar**. Sign in: the item reads **My Classes** and opens the weekly page exactly as before.
11. **Migration on your real DB** (instead of the fresh one): run `knottyyoga_database_helper --migrate` against it once — the report lists `app/0001_drop_instructor_class_preferences` applied, and the table is gone from Manage Data.

---

## Phase 2 — Theme typography: units and weight roles

> The "How text looks" upgrades. Backend first ([hw] — one pin bump covers 1.2 + all of Phase 2).

> ✅ **Phase 2 implemented 8/21/2026.** Gates green: honuware **1735/1735**, knottyyoga C++ **5040/5040** (co-dev, both Linux docker), Angular **3171/3171** + `tsc --noEmit` clean ×2 + `ng build` clean + lint at its 262-problem pre-existing baseline. One deliberate deviation, noted in 2.1: the weight-role tokens live in the **TypeScale** group (not FontRole) so "How text looks" renders and live-previews them.
>
> **Pin bumped 8/21/2026** — honuware `bc206bf` ("Phase 2 — Theme typography: units and weight roles") is pushed and CI-green; both consumers re-pinned and verified against the **pinned clone**: knottyyoga **5040/5040**, communityfinder **1756/1756**.

### 2.1 [hw] Server-side groundwork
- [x] `IsValidCssLength`: accept `pt` alongside `px|rem|em|%` (+ tests: `12pt`/`13.5pt` accepted, junk still refused — the old explicit `8pt`-is-refused assertion flipped to the accepted list).
- [x] Register three **role** weight tokens in `SiteThemeTokens()`: `site_theme_weight_menu` → `--weight-menu`, `site_theme_weight_heading` → `--weight-heading`, `site_theme_weight_display` → `--weight-display` (`ThemeTokenType::Weight`, plain-English descriptions). **Deviation: they sit in the `TypeScale` group, not FontRole** — the editor renders that group in "How text looks", where `pendingStyle` previews them live on the specimen and they share the weight control; FontRole's rows are family pickers. Tests: registry entries + descriptions, `pt` length validation, and a `LoadSiteTheme` case proving `site_theme_weight_menu = 300` + `site_theme_text_base = 12pt` serve as `--weight-menu`/`--text-base`. *(The theme bundle picks the three keys up automatically — the exporter is registry-driven and its coverage guard is dynamic.)*

### 2.2 [app] Unit toggle in "How text looks"
- [x] A `rem / px / pt` `mat-button-toggle` at the top of the section (default rem). Conversion at 1rem = 16px = 12pt. The six `--text-*` rows display their value converted to the selected unit as a number-plus-unit-suffix control; typing writes in the active unit; placeholders show the converted default (`1rem` / `16px` / `12pt`). *(Two as-built properties worth knowing: flipping the toggle converts only the **display** — the stored value keeps whatever unit it was typed in, so browsing units can never dirty a field; and a full CSS size typed against the active unit ("1.5rem" while in px mode) is stored verbatim rather than mangled.)*
- [x] Weight rows are unit-less — the toggle skips them (they get the 2.3 control instead); radius tokens are untouched (the branch keys on `type_scale` lengths only).
- [x] Specs: placeholder conversion per unit; typing `18` in px mode stores `18px` and displays `1.125`/`13.5` under rem/pt; `13.5pt` round-trips; rem mode unchanged; blank clears; verbatim passthrough.

### 2.3 [app] Named weight picker + role wiring
- [x] `_tokens.scss`: `--weight-menu` / `--weight-heading` / `--weight-display`, all defaulting to `var(--weight-semibold)`. Re-pointed: `.header-button` → `var(--weight-menu)`; `.din-bold` → `var(--weight-heading)`; `.din-condensed-bold` → `var(--weight-display)`. Nothing moved visually.
- [x] Site Theme editor: `type === 'weight'` fields render a `mat-select` — the nine named cuts ("100 — Thin" … "900 — Black", the `FACE_WEIGHTS` list **moved to `@shared/types/site-fonts.types`** so the font manager and the editor share one copy) plus "Use the default (600 — Semi Bold)" resolved from the stylesheet. Applies to the three role weights **and** the four scale weights.
- [x] Specimen drift fixed: display/heading lines now ask for the weight **roles** (they overstated headings at `--weight-bold`), and the specimen gained a **menu line** ("Home · Classes · Memberships", uppercase, `var(--weight-menu)`) so the new role previews live as you pick.
- [x] Specs: the weight select renders with the nine named options and the named default; picking 300 for the menu restyles the specimen menu line to computed weight 300; `design-tokens.spec.ts` pins the three role tokens (default = semibold), that a `--weight-menu` override reaches a consumer without moving `.din-bold`, and that `.din-bold` follows a `--weight-heading` override.

### 2.4 Live hand-testing (Phase 2)
- [x] Steps written (below) — awaiting your run against a live server.

Fresh or existing database, server + Angular dev server running, signed in as the admin.

1. **Unit toggle.** Admin ▸ **Manage Products** → **Site Theme** → **Fonts** tab → scroll to **How text looks**. A **rem / px / pt** toggle sits beside the heading, on **rem**. The `--text-base` row's placeholder reads **1rem**; click **px** and it reads **16px**; click **pt** and it reads **12pt**.
2. **Enter a size in px.** With **px** selected, type **18** in the `--text-base` row and press **Save this section**. Body copy across the site (and the specimen's body line) grows; reopen the row — it shows **18** with the px suffix, and flipping to **rem** shows **1.125** without re-dirtying the field. Clear the row and save to go back to the default.
3. **Named weights.** Still in **How text looks**: every `--weight-*` row is now a dropdown of named cuts, first option **"Use the default (600 — Semi Bold)"**.
4. **Menu weight, live.** In the **--weight-menu** row ("Menu items in the top bar") pick **300 — Light**: the specimen's "Home · Classes · Memberships" line thins immediately, before saving. Press **Save this section** and look at the real top menu — it's Light now, while headings everywhere are still semibold. Pick **"Use the default (600 — Semi Bold)"** and save to restore.
5. **Headings decoupled.** Set **--weight-heading** to **800 — Extra Bold** and save: section headings and buttons heavy up while the top menu (still on its own role) doesn't move. Reset to default.
6. **Theme file carries it.** Site Theme → **Theme file** → **Download**: open the zip's `theme.json` — `site_theme_weight_menu` (and any pt size you saved) appears under `tokens` when set.

---

## Phase 3 — Home page: one ordered list of sections

> The architectural core. Everything on the home page between the announcements strip (Phase 6) and the footer becomes a `home_sections` row: position is data, visibility is data. Also delivers the D15 leftover from [[Tenant Theming and Branding]] Phase 6B (per-kind components).

> ✅ **Phase 3 implemented 8/21/2026.** Gates green: knottyyoga C++ **5044/5044** + `knottyyoga_database_helper` builds (Linux docker, co-dev), Angular **3179/3179** + `tsc --noEmit` clean ×2 + `ng build` clean + lint at its 262 baseline. No honuware changes — no pin bump. As-built deviations worth knowing, detailed inline: the page **sorts defensively by ordinal** client-side (insurance, the endpoint already orders); the extraction **deliberately dropped** the legacy page-scoped `h1`-uppercase / 1.5rem-button-pill / letter-spacing quirks (pre-makeover relics the makeover's own Phase 3.2 flagged — buttons now follow the design-system radius); the my-bookings strip keeps its **compact booking card** rather than the shared public event card (a booking is yours already — it needs when/where/waitlist, not a description, spot count and Book button); and the offerings section re-fetches the member's bookings for its suppression (one duplicate GET, accepted to keep sections self-contained).

### 3.1 [app] Schema + migration
- [x] `db_schema/home_sections`: new kind constants — `carousel`, `membership`, `upcoming_events`, `upcoming_classes`, `offerings`, `intro`, `get_started`, `video` (joining `hero|feature|banner|artwork`). New columns: `image_carousel_id` BIGINT nullable (FK added in Phase 4), `video_url` STRING nullable, `hidden_when_logged_in` / `hidden_when_member` BOOL default false.
- [x] Migration `app/0002_home_sections_functional_kinds`: `ADD COLUMN IF NOT EXISTS` ×4 (idempotent by itself — no `ListColumns` guard needed), then per-kind-guarded inserts of the seven functional rows so Mason's existing DB keeps its membership/events/classes sections when the hardcoded islands died. Ordinals reproduce today's exact order (carousel 2 · hero 5 · intro 6 · get_started 7 · upcoming_events 8 · offerings 9 · features 10/20/30 · banner 50 · membership 60 with `hidden_when_member = true` · upcoming_classes 70). Two behavior tests: a pre-Phase-3 database converts (columns dropped + rows deleted first), and a fresh database double-replay changes nothing.
- [x] `create_database.cpp` `PopulateHomeSections`: same functional rows seeded for fresh DBs (image-less; their titles are the labels the editor shows), mirrored comment warns the two lists must stay in step.
- [x] Table helper: `HomeSectionExtras` struct (videoUrl, both flags, imageCarouselId — 0 omits the key, since an empty string can't bind to a BIGINT) as a defaulted param on Add/Update, so the many six-field callers stayed signature-stable. Tests: defaults + full extras round-trip on add and update.
- [x] Admin metadata seeds: column data info + friendly names for the four new columns. Endpoint test extended: a functional row's flags reach the wire (`"t"/"f"`).

### 3.2 [app] Per-kind section components (the D15 extraction)
- [x] Twelve standalone components under `home-page/sections/`: the four content-block extractions (hero/feature/banner/artwork), the functional hosts (intro, get-started, events — all three strips with the auth branching —, offerings, membership, upcoming-classes with the 1.6 tinted band), `home-carousel-section` (wraps today's photo carousel + the `/#gallery` anchor — the carousel's **position is now data**, which closes the "carousel above the banner" complaint), and the `home-video-section` stub (renders nothing until Phase 6). Contained kinds bring their own `.page-container`; full-bleed kinds (hero, carousel) don't — the page no longer owns a single wrapper. Styling moved into each component's own SCSS.
- [x] `HomePageComponent`: **one `@for`** over the ordered, visibility-filtered list with a `@switch` on `kind`; unknown kinds skip. The page now owns exactly three things: the list (sorted by ordinal defensively), the visibility filter, and feature alternation (computed among *visible* features, so a hidden feature doesn't consume an alternation slot). All hardcoded islands deleted.
- [x] Specs: the page spec covers mixed-ordinal cross-kind ordering (the old template's bug), the seeded 12-row order, unknown-kind skipping, the flag matrix, no-flash membership hold, alternation-among-visible, fail-closed, and the auth-flip reload. Section specs carry the moved behavior: events (placement per auth state, caps 4/3/4, booked-exclusion, waitlist chip, cancelled/undated drops, auth swap), offerings (suppression + visitor passthrough + no bookings fetch for visitors), membership (tiers filter, heading by auth, empty ⇒ nothing), intro (headline, live content save, badge 404 fallback once), get-started (copy variants + live flip), upcoming-classes (visitor ⇒ nothing; tinted band — the 1.6 assertions moved here), and one combined file for hero/feature/banner/artwork/carousel/video.

### 3.3 [app] Types + seam
- [x] `HomeSectionKind` union grew the eight kinds; `HomeSection` carries the flags (required booleans), `image_carousel_id` and `video_url`. `getHomeSections` gained a **normalizer** it never had — un-normalized, a `"f"` flag is a truthy non-empty string that would hide every section from members (the `getting_started_steps` trap, again). Mock reseeded to the full 12-row list via a `homeRow()` defaults helper; two mock spec cases (carousel-first order; functional rows present with the membership row members-hidden).

### 3.4 [app] Visibility flags
- [x] Filter in the page: hide when (`hidden_when_logged_in` && logged in) or (`hidden_when_member` && member) — membership via the `getSubscriptions()` profile-hub derivation. While the membership answer is **pending**, members-hidden rows are HELD for a signed-in viewer, preserving the old `showTiers` no-flash contract. Flags apply to every kind.
- [x] Specs: the matrix (visitor / signed-in non-member / member), the pending-hold (a `Subject` keeps the answer open, then resolves), a members-hidden *feature* block, and the seeded membership row disappearing for a member.

### 3.5 [app] Page Content editor
- [x] Kind select grew the new kinds in two `mat-optgroup`s ("Content blocks" / "Built-in sections"), each shown by its plain-English label. Functional kinds keep the **title** (it is the label this list shows — picking one prefills it) and hide body/link/photo; the Get Started banner keeps its photo slot (Phase 8 stacks the fire-font artwork from it). Every home row gets the two visibility toggles. *(`video` and `carousel` config fields land with their phases — `video` isn't offered in the picker until Phase 6 renders it; `carousel` gets its picker in Phase 4.)*
- [x] Row list: photo-less functional kinds show their kind's icon on a tinted tile instead of a photo thumb (and skip the `has_photo` lookup entirely); the kind badge reads as its label; new badges for "Hidden when signed in" / "Hidden for members".
- [x] Specs: functional row rendering (icon + label + flag badge), the grouped picker (video excluded), title prefill + hidden copy/photo fields, both flags in the save payload, and flags round-tripping into the edit toggles; the pre-existing metadata test updated to the labels.

### 3.6 [app] Richer event cards (part of item 19)
- [x] New shared `public-event-card` (`shared/components/`), extracted from `/events`: name, **product description**, time, facility + room, spots remaining, price, Book Now / Members Only / Sold Out — Phase 5 adds the product image on top. `/events` and the home **featured** and **could-sign-up** strips all render it, so home matches the Upcoming Events page. *(Deviation: the my-bookings strip keeps its compact booking card — a booking is yours already; it needs when/where and the waitlist state, not a description, spot count and a Book button.)*
- [x] Specs: a dedicated card spec (fields, room, spots, price/Free, book link, Sold Out, Members Only); the events-section spec asserts description/room/spots on home; the `/events` page spec passes unchanged (the card kept its `.event-card`/`.book-button` classes).

### 3.7 [app] Theme bundle
- [x] `page_content_bundle_section.cpp`: `IsKnownHomeSectionKind` grew to all twelve kinds; the two flags always travel, `video_url` travels when set (like `image`); import writes them through the extras struct. The carousel *reference* joins in Phase 4, when carousels are in the bundle to point at. Round-trip tests: functional kinds + flags byte-identical (JSON and the landed columns both asserted); the unknown-kind refusal now uses `"marquee"` — its old fixture kind, `"carousel"`, became real, which is exactly the promotion path that test guards.

### 3.8 Live hand-testing (Phase 3)
- [x] Steps written (below) — awaiting your run against a live server.

Fresh database (`knottyyoga_database_helper --recreate_database`), server + Angular dev server running.

1. **Regression first.** Signed out, open **Home**: the page reads exactly as before — carousel space, the black hero band, the intro sentence with the badge, the Get Started banner, Upcoming events, Series & workshops, the three alternating features, "Come join the fun!", Memberships, footer. Nothing moved; that is the point.
2. **Position is data now.** Admin ▸ **Manage Products** → **Page Content** → **Home sections**: twelve rows, the built-in ones (Photo carousel, Intro strip, Get Started banner, Upcoming events, Series & workshops, Membership tiers, Upcoming classes) showing kind icons instead of photo thumbs. Use the arrows to move **Membership tiers** above **Why Knotty Yoga**, reload Home signed out — the tiers now sit above the features. Move it back.
3. **The carousel complaint, closed.** Move **Photo carousel** below **Come join the fun!** and reload Home: the carousel renders *below* the banner. (Its images arrive in Phase 4; the row position already obeys.)
4. **Visibility flags.** Edit **Why Knotty Yoga** → toggle **Hide from people who are signed in** → Save. Signed out it shows; sign in and it's gone (and the two remaining features still alternate left/right correctly). Untoggle. Then edit **Membership tiers**: **Hide from people with a membership** is already on — sign in as a member (buy Gold with the Square sandbox card or use the test-helper) and confirm no membership pitch on Home; a fresh non-member account still sees it headed **Become a member**.
5. **Home matches Upcoming Events.** Create a future Intro Workshop session (Manage → Events, Show on home page = yes). Signed out, the Home **Upcoming events** card now shows the product **description**, the room, and "N of M spots remaining" — the same card the **Upcoming Events** page shows.
6. **Functional kinds in the picker.** Page Content → **Add**: the Kind dropdown has two groups — Content blocks and Built-in sections. Pick **Upcoming events**: the title prefills, and body/link/photo fields disappear. Cancel.
7. **Theme file carries the arrangement.** Site Theme → **Theme file** → **Download**. Move two rows around, then **Upload** the file back (Apply): Home returns to the downloaded arrangement, visibility flags included.
8. **Your real DB.** Run `knottyyoga_database_helper --migrate` against it: the report lists `app/0002_home_sections_functional_kinds` (and `app/0003_home_sections_hidden_when_not_member` from 3.9); your existing home sections keep their order and the seven built-in rows appear in Page Content.

---

### 3.9 [app] Members-only rows — the complement flag (Mason, 8/21/2026)

> *"I would like to add a Don't show for people who are not members option for the home page items. There are certain things that I think only make sense to show for people with memberships."*

> ✅ **Implemented 8/21/2026.** Gates green: knottyyoga C++ **5045/5045** + `knottyyoga_database_helper` builds, Angular **3186/3186** (+7), `tsc --noEmit` clean ×2, `ng build` clean, lint at its 262 baseline. App-side only — no honuware change, no pin bump.

3.4 shipped `hidden_when_member` (hide the upsell from people who already bought). This is its mirror: `hidden_when_not_member` hides a row from everyone **without** a membership, so a studio can put member-only content — a members' notice, a perks block, a private-session pitch — on the public home page without it leaking to visitors.

**The rule, and why it is one line.** A row is hidden unless the viewer is a *confirmed* member (`hasMembership === true`). That single condition covers all three non-member states correctly and identically: an anonymous visitor (never a member — the page does not even fetch subscriptions for them), a signed-in non-member, and the window where a signed-in viewer's membership check is still **pending** — hidden while unknown, revealed once confirmed, so member-only content can never flash in front of a non-member. The same no-flash discipline as `hidden_when_member`, pointing the other way.

**Setting both membership flags is legal and means "never show".** Not worth blocking — it falls out of two independent toggles and harms nothing — but the editor warns, because it is far likelier to be a mistake than an intent.

**Not a security boundary, and the editor says so.** The feed stays anonymous and returns every active row; the filter is client-side, exactly like `hidden_when_logged_in` and the Getting Started steps. That is right for *tailoring* — it needs no per-viewer endpoint and no cache-busting — but a determined visitor can read a members-only row's copy in the network response. Anything genuinely confidential belongs behind an authenticated endpoint, not behind this flag; the toggle's hint says as much where the decision is being made.

- [x] [app] `db_schema/home_sections`: `hidden_when_not_member` BOOL NOT NULL DEFAULT false. Migration `app/0003_home_sections_hidden_when_not_member` — a bare `ADD COLUMN IF NOT EXISTS`. Test drops the column, applies, replays (no-op), then inserts a row and reads the default back, proving the column is *usable*, not merely present.
- [x] [app] `HomeSectionExtras` gains `hiddenWhenNotMember`; the helper test's defaults and full-extras round-trips both cover it.
- [x] [app] Theme bundle: the flag travels beside the other two; the functional-kinds round-trip test gained a members-only row (JSON both ways + the landed column).
- [x] [app] Admin column metadata ("Members Only" / "Only show this section to visitors with a membership") + friendly name.
- [x] [app] Seam: `HomeSection.hidden_when_not_member`, normalized through the same `toBoolean` as the others; mock `homeRow()` default + a mock-spec assertion that it defaults off across the seeded list.
- [x] [app] Page filter: `hidden_when_not_member && hasMembership !== true` — one condition covering anonymous, non-member and pending alike. Specs: the three viewer states, the pending hold (member-only copy can never flash in front of a non-member), a **shared entitlement** counting as membership (the profile-hub rule), and both-flags-on never showing for anyone.
- [x] [app] Page Content editor: the third toggle with the tailoring-not-secrecy hint, the "nobody will ever see this" warning when both membership toggles are on (a warning, **not** a block — asserted), and a "Members only" badge in the row list. Specs for all three.
- [x] Hand-testing (adds to 3.8): edit **Why Knotty Yoga** → turn on **Only show to people with a membership** → Save. Signed out: the block is gone. Sign in as a **non-member**: still gone. Sign in as a **member** (buy Gold with the Square sandbox card, or grant it with the test-helper): the block is back, and the features on either side still alternate left/right correctly. Turn on **Hide from people with a membership** as well and confirm the editor warns that the row will never show — then turn both off.

---

## Phase 4 — Named image carousels

> Depends on Phase 3 (the `carousel` kind renders through the unified list). Replaces the single random bag with named, ordered carousels; adds the Gallery page and the Site Theme tab; carousels join the theme bundle (item 7d).

### 4.1 [app] Schema
- [ ] `db_schema/image_carousels`: `image_carousel_id` PK, `name` STRING UNIQUE (stable handle for bundle references), `display_name`, `description` nullable, `show_title` BOOL default true, `show_description` BOOL default true, `show_on_gallery` BOOL default false, `gallery_ordinal` INT default 0, `active` BOOL default true, timestamps.
- [ ] `db_schema/image_carousel_photos`: `image_carousel_photo_id` PK, `image_carousel_id` FK, `title` nullable, `description` nullable, `ordinal` INT default 0, timestamps. Photos attach via the photo association (register in `photo_support_tables` + the anonymous `PublicPhotoTables` list in `main.cpp` — without the latter, visitors see broken images).
- [ ] Full new-table checklist ×2 (top-level `image_carousels`, nested `image_carousel_photos`), CMakeLists, friendly names, display template `{display_name}`.

### 4.2 [app] Migration `app/0004_image_carousels`
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
- [ ] `photo_support_tables` += `products` (seed + migration `app/0005_products_photo_support` inserting the row if absent); `PublicPhotoTables` += `products` (services/events/tiers render for visitors).
- [ ] Product editor (`product-detail.component`): a photo card with `hw-photo-upload` (table `products`) for every kind; product create keeps the save-then-attach flow.
- [ ] `CatalogProduct` type + payload: expose `has_photo` (and confirm `product_id` reaches every consumer needing a URL — `visible_event_sessions` payload gains `product_id`/`has_photo` if absent). Backend tests for the payload additions.

### 5.2 [app] Event images (item 19)
- [ ] The shared public event card (3.6) renders the product photo above the copy (placeholder-free fallback when absent) — on `/events` and all home event strips.
- [ ] Specs: card with/without photo on both surfaces.

### 5.3 [app] Service images (item 18)
- [ ] Services page (`service-catalog`): image above each service card from the product photo; graceful absence.
- [ ] Specs: with/without photo.

### 5.4 [app] Membership tier icons (item 12 — finishes theming Phase 3's leftover)
- [ ] Seed: `Seed::PopulateSeedPhotos` attaches `img/tier_icon_solo.png` / `tier_icon_couple.png` / `tier_icon_family.png` to the three membership products (solo → Gold, couple → Couple's, family → Family) — exactly three icons, confirmed (OQ-4 ✅). Existing DBs: covered by 5.1's registration + a re-runnable seed step or manual upload through the new editor field — noted in hand-testing.
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
- [ ] [app] `db_schema/announcements`: `announcement_id` PK, `title`, `body` (plain text, non-dismissible notice banner — OQ-6 ✅), `show_from_us` BIGINT, `show_until_us` BIGINT, `active` BOOL default true, `ordinal` INT default 0, timestamps. Full checklist + migration `app/0006_announcements`. **Deliberately NOT in the theme bundle** — announcements are timely content, not a look (stated here so the round-trip guard's exclusion is intentional).
- [ ] [app] Helper + `GET /api/announcements` (anonymous: `active` && server-now within `[show_from_us, show_until_us)`, ordered by `ordinal, id`). Endpoint tests: window edges, inactive filtered, anonymity.
- [ ] [app] Editor: an "Announcements" tab in Manage → Page Content — title, body, **date pickers** for the from/until dates (per [[feedback_date_time_pickers]]), active toggle, reorder.
- [ ] [app] Home: an announcements strip pinned **above everything** (top of the page, before the ordered sections) — one `.notice`-styled banner per current announcement. Seam + mock + specs (renders when current, absent otherwise, ordering).

### 6.2 Video home item type (item 26 — OQ-7 ✅: YouTube **and** raw video URLs)
- [ ] [app] Uses Phase 3's `video_url` column. Two accepted shapes, detected from the URL:
  - **YouTube** (`watch?v=`, `youtu.be/`, `shorts/`, `embed/`) → normalized to a privacy-enhanced `youtube-nocookie.com/embed/<id>` iframe (16:9, `loading="lazy"`, `referrerpolicy`).
  - **Direct video file** (`https://…/name.mp4|.webm|.ogg`) → native `<video controls preload="metadata">`, full width, natural aspect.
  - Anything else is refused in the editor and on bundle import.
- [ ] [app] `home-video-section` renders whichever shape the row carries, with the row's title above (optional); the editor's `video` kind shows the URL field with validation feedback naming both accepted shapes.
- [ ] Specs: URL-detection table across both shapes, invalid URL refused in the editor, iframe vs `<video>` element rendered per shape; bundle round-trips `video_url`.

### 6.3 Live hand-testing (Phase 6)
- [ ] Steps: create an announcement spanning today and see it atop Home (signed in and out); set the window to end yesterday and see it vanish; add a video row with a real YouTube link positioned mid-page and play it; swap the URL to a direct `.mp4` and confirm it plays through the native player.

---

## Phase 7 — Locations, timezone, and the About page

### 7.1 Studio-location timezone (item 5) — expose what exists
`facilities.timezone` already exists (IANA, default `America/Los_Angeles`) and drives booking emails/digests/reporting — recorded in [[Bookable Service Foundation]] (decision 10). The gap is surfacing:
- [ ] [app] Manage → **Locations** page (new; locations are currently editable only through the debug Manage Data surface, violating the dedicated-UI rule): list + edit for `facilities` — name, code, full address (including the `address_line_2`/`country` fields the CRUD metadata omits), **timezone** as a searchable select of IANA zones (curated common-US list + free entry), active toggle. Modeled on `instructors-admin`. Dashboard tile (alphabetized per 1.3).
- [ ] [app] `description` STRING nullable column on `facilities` (+ DSL + migration `app/0007_facilities_description`) and photo support (`photo_support_tables` += `facilities`, `PublicPhotoTables` += `facilities`, `hw-photo-upload` in the editor) — both feed 7.2.
- [ ] Recorded, not planned: the service-booking week strip builds days in **browser-local** time (`buildWeek()` uses `new Date()`; `setHours(0,…)`) — fine while all facilities are Pacific, wrong the day a second-timezone location exists. Noted here as the follow-up the timezone column exists to serve.
- [ ] Specs: page CRUD, timezone select persists, description/photo round-trip.

### 7.2 About ▸ Our Location (item 20)
- [ ] [app] Public `/location` page: each active facility — photo, name, description, full address, and an **"Open in Google Maps"** button (`https://www.google.com/maps/search/?api=1&query=<url-encoded address>`, new tab — link-out confirmed, OQ-9 ✅). Menu: About ▸ **Our Location**.
- [ ] Specs: renders active facilities, map href encodes the address, inactive hidden, menu entry.

### 7.3 About page → alternating blocks (item 14 — OQ-8 ✅: markdown drops entirely)
- [ ] [app] `db_schema/about_sections`: same shape as a feature row (`about_section_id`, `ordinal`, `title`, `body`, `link_route`, `link_label`, `active`, timestamps) + photo association (+ public allow-list). Full checklist + migration `app/0008_about_sections`. Seed one block from the current About copy so a fresh DB isn't empty.
- [ ] [app] Helper + `GET /api/about_sections` (anonymous, ordered, active).
- [ ] [app] Manage → Page Content gains an "About sections" list (same editor pattern; reorder/add/remove/photo).
- [ ] [app] About page: heading + the ordered blocks through the *same* `feature-section` component Phase 3 extracted (alternating sides) — **no markdown**. Remove the `<markdown>` render, the route-scoped `provideMarkdown()` on `/about`, and the `graySquare` placeholder.
- [ ] [app] Retire the markdown plumbing with it: the Site Theme editor's **About tab** (union / `sectionOrder` / `sectionKeys()` / the `mat-tab`) and the `aboutText` field in `SiteConfig` / `DEFAULT_SITE_CONFIG` / the merge. **[hw] note:** the `site_about_markdown` slot itself stays registered in honuware — CommunityFinder may consume it, and it travels harmlessly in bundles; revisit at extraction time.
- [ ] [app] Theme bundle: `about_sections` joins the `page_content` section (rows + image assets, order = array position). Round-trip tests.
- [ ] Specs: alternation, reorder reflected, no markdown element on the page, editor tab set shrinks, bundle round-trip.

### 7.4 Live hand-testing (Phase 7)
- [ ] Steps: Manage → Locations — set the studio's timezone and description, upload a photo; About → Our Location shows it and the button opens Maps on the address; add two About blocks in Page Content and see them alternate; export/import a theme and confirm About blocks survive.

---

## Phase 8 — Figma assets and the visual bands

> 8.2 and 8.3 are ⏸ **waiting on the Figma token** (OQ-1 — Ryan is out of town; Mason will supply `~/.figma_token` when he's back). 8.1 and 8.4 can land any time.

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

> ✅ **All twelve answered by Mason 8/20/2026 and folded into the phases the same day.** OQ-1 is the one ⏸ item — the Figma token arrives when Ryan is back, and only Phases 8.2/8.3 wait on it. The dialog is kept below for the record.

- **OQ-1 — Figma access is gone.** Previous sessions read Ryan's file via the REST API with a personal-access token at `~/.figma_token`; that file no longer exists. To unblock the Get Started image and the footer tagline export, either recreate `C:\Users\mason\.figma_token` containing the token (one line), or export the two assets yourself into `server/knottyyoga_server/src/database_helper/img/` (PNG, 2× — `Icon/GetStarted` from Frame 127, and the tagline artwork) and I'll take it from there.
	- Mason- Ryan is out of town. I'll get the token when he is back.
	- Claude- ⏸ Parked. Phases 8.2/8.3 wait for the token; 8.1 (the band restyle, image-optional) and 8.4 (parallax) proceed without it, as does everything else. When Ryan's back: `C:\Users\mason\.figma_token`, one line containing the personal-access token.
- **OQ-2 — "Our Classes / My Classes" menu shape.** Your sentence supports two readings. **Chosen:** one auth-dependent item — signed out: **Our Classes** → All Classes catalog (`/classes/all`); signed in: **My Classes** → the weekly page (`/classes`, unchanged behavior). The All Classes page gains a "Weekly schedule" link so visitors can still reach the schedule advertisement. **Alternate:** show *both* items when signed in (Our Classes → catalog, My Classes → weekly). Say the word if you want the alternate.
	- Mason- Yes, I want the first alternative.
	- Claude- ✅ RESOLVED — reading "the first alternative" as the **first option listed** (the chosen default): **one** auth-dependent item — signed out, **Our Classes** → `/classes/all` (All Classes catalog); signed in, **My Classes** → `/classes` (the weekly page, behavior unchanged), plus the "Weekly schedule" link on All Classes for visitors. Folded into 1.8. *(Stating it concretely on purpose: if you actually meant the show-both-items-when-signed-in variant, say so and 1.8 flips.)*
- **OQ-3 — Price color `--theme-on-accent`.** Implementing literally as asked. Flag: `on-accent` is the text-*on*-an-accent-fill token (white-ish for KY's amber accent) — on a white card it may be near-invisible. If the intent was "accent-colored prices," the token is `--theme-accent`. I'll implement `--theme-on-accent` and you can eyeball it; one word flips it.
	- Mason- --theme-on-accent is actually #1b0e00 which is nearly black (which is the goal)
	- Claude- ✅ RESOLVED — my contrast flag was wrong: KY's on-accent is the near-black ink, so the literal implementation is also the readable one. 1.5 proceeds exactly as asked.
- **OQ-4 — "the other three images."** Only **three** tier icons exist (`tier_icon_solo/couple/family.png`) — solo plus two others. If there's a fourth image you meant, name it.
	- Mason- Just the three
	- Claude- ✅ RESOLVED — solo/couple/family map to the three membership products in 5.4.
- **OQ-5 — Instructor class preferences.** They do nothing today (write-only notes; details in Phase 1.7). Options: **(a)** keep + label as "reference notes, not enforced" *(chosen default)*; **(b)** wire min/max into scheduling/capacity (real feature work — happy to scope); **(c)** delete the surface. Pick one.
	- Mason- Let's delete
	- Claude- ✅ RESOLVED — (c). 1.7 is rewritten as a full top-down removal: the panel + its mount, the four seam methods, the four endpoints, the table helper, and the schema registrations, plus `app/0001_drop_instructor_class_preferences` — now the **first** app migration (later migration ids renumbered 0002–0008 across Phases 3–7 — 3.9 later claimed 0003, shifting Phases 4–7 up by one).
- **OQ-6 — Announcement body.** Chosen: plain text (title + body + date window), styled as a notice banner, not dismissible. Alternates: markdown body, or per-user dismissal (needs storage). Speak up if wanted.
	- Mason- That sounds good.
	- Claude- ✅ Confirmed — plain text, notice banner, no dismissal. 6.1 as written.
- **OQ-7 — Video providers.** Chosen: YouTube only (privacy-enhanced nocookie embed). Vimeo or raw-URL `<video>` can be added later behind the same column.
	- Mason- Let's do YouTube and raw video URL
	- Claude- ✅ RESOLVED — both shapes: YouTube forms → the nocookie iframe embed, and direct `https://…/*.mp4|.webm|.ogg` file URLs → a native `<video controls>` player. 6.2 rewritten with shape detection + validation for both.
- **OQ-8 — About page markdown.** Chosen: the existing `site_about_markdown` renders as an intro above the new blocks when non-empty, so your current About copy survives. Clear the slot in Site Theme when you want blocks only. (Alternate: drop the markdown entirely.)
	- Mason- I think we can drop markdown entirely.
	- Claude- ✅ RESOLVED — About renders heading + blocks only. 7.3 rewritten: the `<markdown>` render, the route-scoped `provideMarkdown()`, the Site Theme editor's **About tab**, and the `aboutText` config field all go (app-side); a seed block carries your current About copy so nothing is lost on a fresh DB. The `site_about_markdown` slot stays registered in honuware (CommunityFinder may consume it) — revisit at extraction.
- **OQ-9 — "Triggers a map."** Chosen: an "Open in Google Maps" link-out built from the address (no API key, no third-party iframe/CSP surface). Alternate: an embedded map (iframe embed or Maps JS — needs a key + CSP allowance).
	- Mason- Let's just do the link to Google maps
	- Claude- ✅ Confirmed — link-out, no embed. 7.2 as written.
- **OQ-10 — Carousel data migration touches your live DB.** Phase 4.2 moves your existing `home_page_photos` rows (ids preserved, photo associations re-pointed) into the new carousel tables as a "Home page photos" carousel, then drops the old table + endpoint. It's idempotent and tested against both DB states, but it is a one-way conversion of real data — flagging it rather than doing it silently.
	- Mason- That's why we are doing this BEFORE I deploy.
	- Claude- ✅ Confirmed — the migration ships pre-deploy as planned (4.2), with the both-states idempotence tests.
- **OQ-11 — Which "admin portal" to alphabetize.** Chosen: the Manage dashboard (31 tiles), Staff Portal, and the personal profile tiles. The `/admin` Manage Data page has no tiles (it's a table dropdown from `@honuware/ui`) — left alone as the debug surface.
	- Mason- It would honestly be nice if the tables in Manage Data were also alphabetized.
	- Claude- ✅ RESOLVED — scope grew: the Manage Data table dropdown gets alphabetized by its displayed friendly name, sorted at the source (added to 1.3 — [hw] if the list is server-produced, app-side wrapper sort otherwise).
- **OQ-12 — Timezone scope.** `facilities.timezone` already exists and is consumed server-side; chosen scope is the editing surface + validation + the new Locations page (7.1), with the browser-local week-strip gap recorded as a follow-up. If you meant something more (e.g., rendering all class times in facility TZ on every page), say so and I'll scope it separately.
	- Mason- Yes, let's just expose a way to set the existing timezone.
	- Claude- ✅ Confirmed — 7.1 as written (the Locations page is the vehicle); the browser-local week-strip gap stays a recorded follow-up, not in scope.