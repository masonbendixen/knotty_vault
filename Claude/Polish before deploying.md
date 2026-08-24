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

> ✅ **Phase 4 implemented 8/22–23/2026.** Gates green: knottyyoga C++ **5055/5055** (+10 net — 16 new, minus the deleted `home_page_photos` tests) + `knottyyoga_database_helper` builds (Linux docker, co-dev), Angular **3222/3222** (+36) + `tsc --noEmit` clean ×2 + `ng build` clean + lint **261**, one *below* its 262 baseline with nothing in the new files. App-side only — no honuware change, no pin bump.
>
> **The migration is the part that matters, and its test earned its keep four times.** `app/0004_image_carousels` moves Mason's live photos, and four separate defects were caught that a fresh database exercises none of: the photo re-point ran *before* the new table was registered in `photo_support_tables` (an FK `table_item_photos` depends on — this would have failed the entire migration on any database with photos); `SELECT setval(...)` returns a row and so needs `RunSqlStatementReturningOneValue`; a `$1` used both bare and against a varchar needs explicit casts on both; and the final re-point step had no `carousel` home row to act on in the test. Details in 4.2.
>
> **A date-dependent test was fixed in passing** (not Phase 4 work, but Phase 4's gate is what surfaced it): `SeedAppDataTest.IntroWorkshopSessionsAppearOnBothTheHomePageAndEventsFeeds` asserted a hard `2` on the home-page feed. The seed books the next two Saturdays and the band hides a session until it is within 14 days, so **on a Saturday before 10am the second one is legitimately hidden** — the test failed every Saturday morning and passed the other six days. The expectation is now derived from the seeded rows and the same lead time the query uses, with an `ASSERT_GE(…, 1)` so it can never decay into a vacuous `0 == 0`.
>
> **Post-gate fix (8/23/2026), found by Mason's first `--recreate_database`:** `image_carousel_photos` was registered in `PopulateAdminNestedTables` only. `admin_column_data_info` and `admin_column_friendly_names` both carry a **foreign key onto `admin_top_level_tables`**, so giving the table column metadata without adding it to the allow list killed the whole seed. One line fixed it; the other ten checklist steps audited clean for both tables. **Nested is not an alternative to top-level for a table with per-column metadata** — though nested-only *is* legitimate for a table without it, since `EndpointAuthHelper::GetAllowedTables` unions both lists for an admin (four tables are in that state on purpose). New `create_database_test.cpp` (3 tests) closes the hole for every future table: the admin-metadata group is now one public `PopulateAdminMetadata()`, and simply RUNNING it in a test transaction lets the database check its own keys. Nothing in the gate executed that code before — `CreateSchemaAndPopulate` opens its own transaction and creates tables, so no test could call it, exactly the gap `seed_app_data.h` describes for the data seeders. Final: **5058/5058**.
>
> **A UI fix in the same pass:** the Handle field's hint wrapped to two lines and overlapped the Description field — Material's subscript area is a fixed one-line height unless the field sets `subscriptSizing="dynamic"`. Applied to the three hinted fields in the carousel form and to the new Carousel picker in Page Content.
>
> As-built deviations worth knowing, detailed inline: the home page keeps its **`#gallery` anchor** (the plan said it could go — old `/#gallery` links still land somewhere sensible now that the real page exists); the gallery reader is named **`GetGalleryCarousels`**, not `GetActiveOrderedForGallery`; a fresh site seeds **one empty "home" carousel** rather than any stock photos (matching the old empty `home_page_photos`); and the Site Theme tab body is **deferred** with `<ng-template matTabContent>` because Material builds every tab's content eagerly — without it the manager fetched two tables on every visit to Site Theme, and it took all 41 existing `SiteThemeComponent` specs down with it.

### 4.1 [app] Schema
- [x] `db_schema/image_carousels`: `image_carousel_id` PK, `name` STRING UNIQUE (stable handle for bundle references), `display_name`, `description` nullable, `show_title` BOOL default true, `show_description` BOOL default true, `show_on_gallery` BOOL default false, `gallery_ordinal` INT default 0, `active` BOOL default true, timestamps. `kImageCarouselHomeName = "home"` is a constant, not a literal — the seed, the migration and the seeded home row all name the same carousel.
- [x] `db_schema/image_carousel_photos`: `image_carousel_photo_id` PK, `image_carousel_id` FK, `title` nullable, `description` nullable, `ordinal` INT default 0, timestamps. Photos attach via the photo association (registered in `photo_support_tables` + the anonymous `PublicPhotoTables` list in `main.cpp` — without the latter, visitors see broken images).
- [x] Full new-table checklist ×2 (top-level `image_carousels`, nested `image_carousel_photos`), CMakeLists, friendly names, display template `{display_name}`. A fresh site is seeded with **one empty "home" carousel**, gallery-flagged, so the seeded `carousel` row has something to point at and the manage tab opens on a real example — matching the old behaviour, which also shipped an empty `home_page_photos` (there are no stock photos).

### 4.2 [app] Migration `app/0004_image_carousels`
- [x] Create both tables if absent; insert a **"home"** carousel (display_name "Home page photos") if absent; copy `home_page_photos` rows into `image_carousel_photos` **preserving ids** (then `setval` the sequence) so `UPDATE table_item_photos SET table_name = 'image_carousel_photos' WHERE table_name = 'home_page_photos'` re-points every photo without an id map; swap the `photo_support_tables` row; point the Phase-3 carousel home row's `image_carousel_id` at "home"; drop `home_page_photos`. Ordinals come from the old `created_at_us` order, so the studio's photos get a sensible sequence instead of the random one the old query produced. Idempotent at every step; two tests — a pre-migration DB converts, a fresh DB no-ops.
- [x] **The conversion test earned its keep three times over** — every one of these would have failed on Mason's real database and none would have shown up any other way, because the test harness pre-creates tables and a fresh DB exercises none of these paths:
	- **`photo_support_tables` ordering.** `table_item_photos.table_name` has a foreign key onto `photo_support_tables`, so registering the new table had to happen **before** the re-point, and de-registering `home_page_photos` **after** it. The first draft did both at the end and blew up on any database that actually had photos.
	- **`SELECT setval(...)` returns a row.** `RunSqlStatement` insists on none; it needs `RunSqlStatementReturningOneValue`. Noted inline so the next `setval` does not repeat it.
	- **`SELECT $1` vs a varchar comparison.** Postgres refuses to deduce one type for a parameter used both as a bare select value (text) and against a varchar column ("inconsistent types deduced for parameter $1"); both uses are cast explicitly.
- [x] Delete the old surface end-to-end (no dead code): all nine `home_page_photos` files (db_schema, table helper, endpoint + their three test files), the `getHomePagePhotos` seam method + mock + mock spec, the `HomePagePhotoInfo` app-side re-export shims, admin metadata seeds, and the `PublicPhotoTables` entry. `get_scaled_photo_test.cpp` was converted to carousel photos rather than deleted.

### 4.3 [app] Helpers + endpoints
- [x] `TableHelpers::ImageCarousels` — `ImageCarouselOptions` struct + `AddCarousel` / `GetCarousel` / `GetCarouselByName` / `GetAllCarousels` / `GetGalleryCarousels` / `UpdateCarousel` / `DeleteCarousel` and the five photo operations. Three orderings, each deliberate: all-carousels by `display_name` (the editor list), gallery by `gallery_ordinal` (the page), photos by `ordinal, id` (the carousel itself). 6 helper tests.
- [x] `GET /api/image_carousel/<int>` (anonymous: carousel row + ordered photos; **404 for missing OR inactive**, so retiring a carousel takes it off the site rather than leaving it reachable by id) and `GET /api/gallery_carousels` (anonymous: `show_on_gallery` carousels by `gallery_ordinal`, each with photos). One shared `CarouselWithPhotos()` builds both shapes. Writes ride generic CRUD + the existing photo upload endpoints. Anchored in `web_app.cpp` — without the anchor the endpoints dead-strip to a 404 in Release. 5 endpoint tests (anonymity, ordering, inactive filtered, gallery flag).

### 4.4 [app] The carousel manager — "Image carousels" tab in Site Theme
Per item 7c it lives in `/manage/site-theme` (union + `sectionOrder` + `sectionKeys()` switch + a `mat-tab`; section Save hidden like the Theme file tab — this tab saves through its own flow).
- [x] Carousel list: add / rename / describe / activate / delete (confirm), reorder `gallery_ordinal`, toggle `show_on_gallery`, `show_title`, `show_description`. The four toggles are **one-click on the row** rather than edit-panel fields — they are what a studio flips while looking at the list. Reads go through `getTableRows`, not the public endpoint, because the editor is where an **inactive** carousel gets switched back on.
- [x] Per-carousel photo panel: upload, remove, reorder, edit caption/description per photo, thumbnails via scaled-photo URLs. Adding a photo uses `hw-photo-upload` with **`deferUpload`** — the association is keyed on the row id, so the file is held and attached the moment the row exists (the instructors-admin pattern). One partial-failure message distinguishes "the row saved but the image did not" from a failed save.
- [x] **The tab body is deferred (`<ng-template matTabContent>`).** Material instantiates every tab's content eagerly, so without this the manager ran its two table reads on **every** visit to Site Theme — and it broke all 41 existing `SiteThemeComponent` specs, which is how it was caught. Every other tab renders from data the page already has; this is the only one with reads of its own.
- [x] Two orderings, deliberately different: the list is **alphabetical by display name** (matching `GetAllCarousels`, so the editor and any picker agree), while the gallery arrows move a carousel within the **`gallery_ordinal`** list — and a carousel joining the gallery is appended (max + 10) rather than silently landing in the middle.
- [x] The handle is derived from the display name on add (`Aerial shots` → `aerial-shots`) and **never rewritten on edit** — a theme file references it. Validated as a slug and for uniqueness, excluding the row being edited.
- [x] Specs: 22 covering the read translation ("t"/"f" as booleans, both orderings, counts and badges), carousel CRUD with the exact CRUD payloads, handle validation, the four toggles, gallery + photo reordering as ordinal swaps, photo add/edit/delete, both empty states, and the two error paths (a refused delete says *why* — a home row probably still points at it — rather than "try again").

### 4.5 [app] Home `carousel` kind + component polish (items 7, 7b)
- [x] `home-carousel-section` fetches the row's `image_carousel_id` and hands it to the renamed, now **presentational** `image-carousel` (`@Input() carousel`), so the home page and the Gallery page draw a carousel identically. The band's heading and blurb come from the **carousel**, not the row — the row's title is only the label the editor lists it by. A row pointing at nothing, or at a retired carousel (the endpoint 404s), renders nothing rather than an empty band under a heading. The `#gallery` anchor stays so old `/#gallery` links still land somewhere sensible.
- [x] Page Content editor: a **Carousel select** on `carousel` rows, from `getFkOptions('image_carousels', …)` so it names carousels the same way every other FK picker does; the row list shows which carousel a row points at without opening it; picking the carousel kind defaults to a real carousel rather than to nothing; and with no carousels yet the field is replaced by a notice pointing at Site Theme ▸ Image carousels. `image_carousel_id` is sent **only** for the kind that uses it and only when set — the column is a nullable BIGINT and `''` cannot bind to one.
- [x] Component fixes: **variable-width cards** — one uniform card height with `width: auto`, so each image keeps its own aspect ratio and a wide shot is wide (item 7's cropping ask); captions render title **and** description (the description was fetched and never shown) gated by the carousel's `show_title`/`show_description`, with the whole caption dropped when a photo has neither. The 3-visible-cards `maxOffset` guess is replaced by a **DOM measurement** (`track.scrollWidth − viewport.clientWidth`), the step is each card's real rendered width, and a drag snaps to the nearest card edge so no image is left sliced at the viewport border.
- [x] Specs: 11 on the carousel (card rendering, the empty carousel, all three caption-gating cases, measured `maxOffset`, the fits-entirely case, stepping by real mixed widths with clamping at both ends, and arrows appearing only where there is travel — the measurement is stubbed with real detached DOM nodes, since images never lay out in a headless run) plus 3 on the home band and 7 on the editor picker.

### 4.6 [app] Gallery page
- [x] New public route `/gallery` rendering `gallery_carousels`: each carousel's display name (if `show_title`), description (if `show_description`), and its image-carousel. Menu: About ▸ Gallery → `/gallery`, replacing the `/#gallery` anchor that only worked from the home page. The page renders the server's order rather than re-sorting, and **drops a flagged carousel with no photos** — a studio mid-setup should not get a heading over nothing. A failed fetch is an empty gallery, not a broken page.
- [x] Specs: 8 on the page (ordering preserved, one renderer per carousel, both flag gates, a description switched on but never written, the photo-less drop, the empty state, and the failed fetch) + 2 on the menu (the item targets `/gallery`, and no `#gallery` anchor survives anywhere in the menu).

### 4.7 [app] Theme bundle (item 7d)
- [x] `page_content` gains `image_carousels`: per carousel — name, display_name, description, the four flags, `gallery_ordinal`, and its photos (title/description + image assets named `carousel-<n>-<m>.<ext>`, from POSITION, never from an id). Home `carousel` rows carry `image_carousel` — the **name** — and no id.
- [x] **Order of operations on import: carousels are replaced BEFORE the sections.** A `carousel` section resolves its carousel by name against a `carouselIdByName` map built from what *this import just created*, so a reference either lands on the right carousel or is left empty — never on an unrelated carousel that happened to inherit the id. Import validates every carousel name and every photo asset before writing anything.
- [x] Three tests: carousels + photos round-trip byte-identical with the image surviving the delete-and-reupload; a `carousel` section travels by name and lands re-pointed at the carousel the **import** created (not the old id); and a photo naming an asset the bundle does not contain is refused with the carousel named in the message.

### 4.8 Live hand-testing (Phase 4)
- [x] Steps written (below) — awaiting your run against a live server.

Fresh database (`knottyyoga_database_helper --recreate_database`), server + Angular dev server running, signed in as an admin.

1. **What a fresh site starts with.** Admin ▸ **Manage Products** → **Site Theme** → the **Image carousels** tab: exactly one carousel, **Home page photos**, badged `0 photos` and `Gallery #1`. Home, signed out, shows no carousel band — an empty carousel draws nothing rather than a blank strip. **About → Gallery** shows "No photos yet".
2. **Fill the seeded carousel.** Back on **Image carousels**, click the chevron on **Home page photos** → **Add a photo**: choose an image, Caption `Partner acrobatics`, Description `Trust and teamwork in motion`, **Add photo**. Add two more the same way, one of them a deliberately **wide** landscape shot. Reload **Home** signed out: the carousel band is there, and the wide photo is genuinely wider than the others — every image is the same height at its own aspect ratio, not cropped to a square.
3. **Captions, and the switches that control them.** Each card shows its caption and description. On the carousel's row, untick **Show the description** — reload Home: captions keep their titles, the descriptions are gone. Untick **Show the name** too and reload: no captions at all. Tick both back on.
4. **Order is yours.** In the photo panel use the ↑/↓ arrows to move the third photo to the top; reload Home and confirm it now leads. (The old carousel shuffled randomly on every load — reload twice more and confirm the order holds.)
5. **A second carousel.** **Add a carousel** → Name `Aerial shots`. Watch the **Handle** fill itself in as `aerial-shots`. Description `Up in the silks.`, tick **Show on the Gallery page**, **Add carousel** — it opens straight into its photo panel. Add two photos.
6. **The Gallery page.** **About → Gallery**: both carousels, **Home page photos** first, each under its own heading with its blurb. Back in Image carousels, use the ↑ arrow on **Aerial shots** to move it up the gallery order; reload Gallery — it now leads. The badges read `Gallery #1` / `Gallery #2` to match.
7. **A home band pointing at the new carousel.** **Page Content** → **Home sections** → **Add**: Kind **Photo carousel** — a **Carousel** picker appears, already on a real carousel. Choose **Aerial shots**, set the position below **Come join the fun!** with the arrows, Save. The row list names the carousel it shows. Reload Home: two carousel bands, the aerial one below the banner, each with its own heading.
8. **Retiring one hides it everywhere.** Untick **Active** on **Aerial shots**. Reload Home — the aerial band is gone (not an empty heading), and it is off the Gallery page too. The carousel is still in the editor, badged **Inactive**, ready to switch back on. Tick it back.
9. **Delete refuses to orphan a row.** Try to delete **Aerial shots** while the home row still points at it: the editor explains that a home page row still references it rather than saying "try again". Change that row to **Home page photos**, then delete **Aerial shots** and confirm it disappears from both the list and the Gallery.
10. **Theme file carries carousels AND their images.** Site Theme → **Theme file** → **Download**. Delete a photo from **Home page photos**, then **Upload** the file back (Apply): the photo is back, with its caption, in its original position — the images travel inside the theme file, and the home row still points at the right carousel by name.
11. **Your real DB.** Run `knottyyoga_database_helper --migrate` against it: the report lists `app/0004_image_carousels`. Open Site Theme → Image carousels: your existing home page photos are all there, in one carousel called **Home page photos**, **with their images intact** and in a stable order — and Home still shows them. (This is the migration that moves your live photos; step 10's download is worth doing first as a backup.)

---

## Phase 5 — Product images: services, events, membership tiers

> One mechanism serves all three: `products` joins the photo system (one photo per product).

> ✅ **Phase 5 implemented 8/23/2026.** Gates green: knottyyoga C++ **5064/5064** (+6) + `knottyyoga_database_helper` builds (Linux docker, co-dev), Angular **3237/3237** (+15) + `tsc --noEmit` clean ×2 + `ng build` clean + lint **261**, unchanged from its Phase 4 baseline with nothing new in the touched files. App-side only — no honuware change, no pin bump.
>
> **The premise held: one photo per PRODUCT covers all three surfaces.** An event card's banner, a service card's image and a membership tier's icon are the same association read three ways, so a studio uploads in one place (Manage Products ▸ Image) and the editor's hint names which of the three that particular product will use. No second or third photo surface was built.
>
> **The `"false"` trap, for the third time.** `has_photo` / `product_has_photo` cross the wire as JSON **strings**, and `"false"` is truthy — un-normalized, every product claims an image and every card renders a broken one. Normalized in the seam for all three feeds (`getCatalogProducts`, `getVisibleEventSessions`, `getEventSessionDetail`) and guarded AGAIN in the event card the same defensive way `can_book` already was, with a spec pinning the string case. See [[gotcha_postgres_bool_strings]] territory — this one is now a reflex.
>
> **`product_has_photo` is resolved in SQL, not per row.** An `EXISTS` subquery inside the three visible-session queries; a home strip renders several cards and a lookup each would be one round trip per card. The `EventSessionHelper` test is the ONLY thing exercising that SQL — the KeyValueTable tests merely prove the struct field travels, which is exactly the distinction [[feedback_assert_through_the_reader]] is about.
>
> **One of my tests failed and the TEST was wrong, not the code.** The seed test asserted the tier icon reached `CatalogHelper::GetProduct`, which returns `nullopt` for a product with no price — and the seed fixture deliberately seeds no membership prices. The icons attach correctly (the gate log shows all three rows). The over-reaching assertion was dropped rather than an unrelated pricing fixture built; the catalog translation is covered by its own test.
>
> **Known deviation, deliberately flagged rather than quietly dropped:** 5.1 asked for the save-then-attach flow on product **create**. The Image card is on the product **detail** editor only. Create redirects to detail on save, so the photo is one click away and nothing is broken — but the create form has no deferred-upload control, unlike the carousel photo panel in 4.4.

### 5.1 [app] Products join the photo system
- [x] `photo_support_tables` += `products` (seed + migration `app/0005_products_photo_support`, guarded and cast the same way 0004 learned to be); `PublicPhotoTables` += `products` — without it every product image 401s for the signed-out visitor who is exactly the audience for services, events and tiers.
- [x] Product editor (`manage/products/product-detail`): an **Image** card with `hw-photo-upload` (table `products`) for every kind, with a per-kind hint naming where that image will actually be seen (tier icon / service image / event banner) so a studio knows what shape to upload.
- [x] `CatalogProduct` gains `has_photo`; `ProductInfo.hasPhoto` is set in `CatalogHelper` (both `GetCatalog` and `GetProduct`) via `TableItemPhotos::HasPhoto` — a lookup, not a column, so it cannot live in the `const` `ProductInfoFromKeyValueTable`. `visible_event_sessions` already carried `product_id`; it gains `product_has_photo`, resolved as an **EXISTS subquery inside the three visible-session queries** rather than a lookup per row (a home strip renders several cards).
- [x] **The wire value is the STRING `"true"`/`"false"`, and `"false"` is truthy.** Normalized in the seam for both feeds (`getCatalogProducts`, `getVisibleEventSessions`, `getEventSessionDetail`) through the existing `toBoolean`, AND guarded again in the event card the same defensive way `can_book` already was. Un-normalized, every product would claim an image and every card would show a broken one.
- [x] Tests: KeyValueTable carries the flag both ways (payment + scheduling); `CatalogHelper` reports it through **both** readers; `EventSessionHelper` reports it through **both** placements — that last one is the only thing exercising the SQL, since the KVT tests only prove the struct field travels.

### 5.2 [app] Event images (item 19)
- [x] The shared public event card (3.6) renders the product photo as a banner above the copy — so `/events` and every home event strip get it from one change. Keyed on `product_id`, not the session: one image serves every session of the same product. **Placeholder-free** when absent: a studio that has not uploaded one keeps the card it has today rather than a grey box on every event.
- [x] The card body switched from `height: 100%` to `flex: 1` — with the photo above it, the old rule added the image's height to a card already filling its grid cell.
- [x] Specs: the URL is built from the product (not the session id), no image when absent, no image for a missing flag, and no image for the string `"false"`.

### 5.3 [app] Service images (item 18)
- [x] Services page (`service-catalog`): the product image spans the top of each service card; graceful absence leaves the card starting at its title.
- [x] **Fixed 8/23 after Mason's first upload — the image hung off BOTH sides of its card.** The first version pulled it outward with `width: calc(100% + 2 * var(--space-4))` and negative side margins, on the assumption that it had to escape `mat-card`'s padding. **`mat-card` has no padding**: Material puts it on `.mat-mdc-card-header` / `mat-card-content` (this repo's own `_surfaces.scss` says so, and even patches the header's bottom padding for header-only cards). So the margins cancelled nothing and simply made the image 32px wider than its card. Now a plain `width: 100%`, with `overflow: hidden` on the card clipping the image's square top corners to the card radius.
- [x] Specs: only the service that has one gets an image, the URL is right, the rest of the card is unaffected when absent — **and one GEOMETRIC assertion** (`photo.width <= card.width`, both edges inside the card's) that the broken CSS fails. The three attribute-level specs all passed while the layout was visibly wrong, which is exactly why they did not help.

### 5.4 [app] Membership tier icons (item 12 — finishes theming Phase 3's leftover)
- [x] Seed: `Seed::PopulateSeedPhotos` attaches `tier_icon_solo.png` / `tier_icon_couple.png` / `tier_icon_family.png` to `gold-membership` / `gold-couples` / `gold-family` (PNG, not JPEG — flat artwork with transparency). It runs after `PopulateProducts`, which is what the lookup-by-code needs.
- [x] `membership-tier-cards`: the product photo IS the tier icon, falling back to today's `card_membership` Material icon. Because it is the same product-photo mechanism, a studio replaces an icon by uploading an image in Manage Products — no separate surface.
- [x] Specs: icon renders from the photo (and the Material icon steps aside rather than stacking), the fallback is intact, and a row can mix iconed and un-iconed tiers. Seed spec: exactly the three membership products get icons, and the Intro Workshop does **not** — an icon on the wrong product is the failure worth guarding.
- [x] **Existing DBs:** 5.1's migration registers `products` for photos; the icons themselves are a fresh-DB seed, so an existing site uploads them through the new editor field (in the hand-testing steps).
- [x] **⚠️ honuware fix (8/23) — the icons arrived as BLACK SQUARES, and it was not the artwork.** All three seed PNGs are colour type 6 (RGBA) with a real alpha channel. `ImageResize::ResizeImage` read **every** format into an `rgb8_image_t`; `read_and_convert_image` drops the alpha silently, and a fully transparent pixel is stored as `0,0,0,0` — so the transparency came back black and was written out opaque. **This affected every transparent PNG in the app** (logo, favicon, any uploaded artwork), not just these icons, because they all resolve through `/api/get_scaled_photo/...`. The stored ORIGINAL was always fine; only the scaled copy was flattened.
	- A PNG **that actually has alpha** now resizes through `rgba8_image_t`, calling `read_and_convert_image` / `write_view` with `png_tag()` **directly** rather than through the file's `ReadImage`/`WriteView` switches. That directness is load-bearing: a template `switch` instantiates **all four** format branches for whatever pixel type it gets, and `write_view` with `jpeg_tag` does not accept RGBA — `enable_if` fails to compile even though only the PNG branch could ever run.
	- **"Always use RGBA for PNG" is wrong**, and was the second failed attempt: an opaque RGB PNG read into `rgba8` comes back as garbage (alpha 13, shifted colour), which would have corrupted every opaque PNG upload. Routing asks the file — `PngHasAlphaChannel()` reads IHDR byte 25 and tests the alpha bit (4 = gray+alpha, 6 = RGBA). **That regression showed up ONLY in the app's co-dev run, not honuware's** — same source, different build. Gate both; either alone would have shipped it.
	- **A third, pre-existing hazard surfaced:** `boost::iostreams::stream` buffers and drains only on destruction — after `ResizeImage` has returned its vector. It has always worked by NRVO accident. Now flushed explicitly in both branches; without it you get a truncated file and `png_check_validity: ... iostream error`.
	- JPEG has no alpha and BMP/TIFF alpha is a separate question, so those keep the RGB path. Palette PNGs with a `tRNS` chunk are transparent too but still take the RGB path — unchanged behaviour, not a regression, but an indexed transparent PNG would still come back black.
	- Three honuware tests: transparent PNG keeps `alpha == 0`; opaque PNG stays on the RGB path with colours intact; and the routing input itself, so neither fixture can silently stop being what it claims.
	- **Cross-repo:** honuware gate green at **1738/1738**. Needs Mason's push + CI, then a pin bump in **knottyyoga and CommunityFinder** (CF shares the same resize path and the same latent bug).

### 5.5 [app] Home membership panel = Memberships page (item 12a)
- [x] `/shop` renders subscription products through the **shared** `membership-tier-cards` — the same component the home page's membership band uses — so the two surfaces cannot drift. Other kinds keep the generic catalog card below it.
- [x] Specs: subscriptions render through the shared card and NOT as generic anchors; non-subscription kinds are unaffected; a mixed catalog shows both; and the tier icon carries through.

### 5.6 Live hand-testing (Phase 5)
- [x] Steps written (below) — awaiting your run against a live server.

Fresh database (`knottyyoga_database_helper --recreate_database`), server + Angular dev server running.

1. **The tier icons are there out of the box.** Signed out, open **Home** and scroll to the membership band: the three tiers show the **laurel icons** (solo, couple, family) instead of the grey `card_membership` glyph. Open **Memberships** from the top menu — the same three cards, **identical** to the home band: same icon, name, description, price and Subscribe button. That is 5.5: one component, two places.
2. **An event image, seen by a visitor.** Admin ▸ **Manage Products** → **Products** → **Intro Workshop**: the new **Image** card sits under Product Information, and its hint says the picture becomes this event's card banner. Upload a landscape photo. Sign out, open **Upcoming Events**: the Intro Workshop card now leads with that image. Open **Home**: the same image on the same card in the Upcoming events strip — one upload, both surfaces.
3. **A service image.** Manage Products → a **bookable_service** product (e.g. a massage). Its Image hint says the Services page. Upload a photo, then visit **Services** signed out: the image spans the top of that service's card. The services you did *not* give a photo still look right — their cards simply start at the title, with no grey placeholder.
4. **Replacing a tier icon is just another upload.** Manage Products → **Knotty Yoga Gold Membership** → Image → upload something different. Reload Home and Memberships: both show the new artwork. Delete the image on that card and reload: that tier falls back to the `card_membership` icon while the other two keep their laurels — a mixed row is fine.
5. **Signed-out visibility is the point.** Do steps 2–4 in a private window with no session. Every product image must render; a broken picture there means `products` is missing from the anonymous allow-list.
6. **Your real DB.** Run `knottyyoga_database_helper --migrate` against it: the report lists `app/0005_products_photo_support`. Then open Manage Products → any product: the **Image** card accepts an upload. (Before this migration the upload is refused with "Photos are not supported for this table" — that row is what the migration adds.) The three laurel icons are a fresh-DB seed, so on your existing database you upload them yourself through this field.

---

## Phase 6 — Announcements and video

> ✅ **Phase 6 implemented 8/23/2026**, with 6.4 and 6.5 added the same day from Mason's live use. Final gates green: knottyyoga C++ **5089/5089** + both executables build (Linux docker, co-dev), Angular **3339/3339** + `tsc --noEmit` clean ×2 + `ng build` clean + lint **261** (baseline, nothing in the touched files). App-side only — no honuware change, no pin bump.
>
> **Three of the four rounds of feedback were on things the automated gates could not have caught** — a one-day announcement being impossible to express, the wrong alert colour and spacing, a date box rendering at twice the height of its neighbour, and lists that reset your scroll position. All shipped green. Where the fix was a layout or lifecycle bug, the new spec asserts the thing that actually broke (geometry, or "did this re-fetch?") rather than an attribute that was already passing.
>
> **The shape of 6.1 that matters:** the announcements strip is NOT a `home_sections` kind. It renders above the ordered list, so a studio cannot bury an urgent notice by reordering rows and does not need to add a row to post one. The visibility window is enforced SERVER-side in SQL against the database's own clock, which means the client has no date logic at all and structurally cannot render an expired notice — the strip's spec has no date tests because there is no date decision there to get wrong.
>
> **Two failures worth keeping.** (1) The video component derived its parsed embed in `ngOnChanges`, which fires only for TEMPLATE-BOUND inputs; three specs assigning the input directly rendered nothing. The fix — an input setter — is better than both the original and the obvious alternative, because a getter would recompute `bypassSecurityTrustResourceUrl` every change-detection pass, changing the iframe's `src` identity and reloading the player on a loop. That one would never have surfaced in a test. (2) An existing spec asserted `video` was ABSENT from the kind picker, annotated "joins the picker in Phase 6, when it renders" — a tripwire that was an instruction rather than a prohibition, unlike the cache-header one in Phase 5.
>
> **Deliberate exclusions, both recorded in code so they read as decisions:** announcements never enter the theme bundle (timely content, not a look — an old bundle would resurrect an expired notice), and palette-with-`tRNS` PNGs still take the RGB path from the Phase 5 image work.

### 6.1 Announcements (item 25)
- [x] [app] `db_schema/announcements`: `announcement_id` PK, `title`, `body` (plain text, non-dismissible notice banner — OQ-6 ✅), `show_from_us` / `show_until_us` BIGINT **nullable**, `active` BOOL default true, `ordinal` INT default 0, timestamps. Full checklist (all six `create_database` points, top-level allow-list included) + migration `app/0006_announcements`, guarded on `ListTables` so the DDL stays in db_schema rather than being written twice. **Deliberately NOT in the theme bundle** — announcements are timely content, not a look; exporting them would carry one studio's holiday hours into another's site, and re-importing an old bundle would resurrect a correctly-expired notice. Stated in the schema header so the exclusion reads as a decision.
- [x] [app] Helper + `GET /api/announcements` (anonymous; `active` && the database's own `now_us()` inside the window; ordered by `ordinal, id`). **Stored half-open `[from, until)`** — the right way to hold an instant. Null on either end = no bound there. 9 helper + 5 endpoint tests.
- [x] [app] **⚠️ Corrected 8/23 after Mason's first real use: "from today until today" was refused as "ends before it starts".** That is the single most common announcement there is — construction, smoke, a burst pipe — and the editor made it inexpressible. The storage contract was fine; the mistake was the EDITOR equating "the day you picked" with "the instant it ends". A date picker hands over midnight, so two identical dates produced an empty window. Nobody says "closed the 24th to the 26th" meaning it reopens on the 26th: two picked dates are **inclusive of both days**. The editor now converts — `showUntil` is the last day it APPEARS, saved as the start of the following day and converted back on load — using local-calendar arithmetic (`new Date(y, m, d + 1)`) rather than "+ 86400s", because a DST day is 23 or 25 hours and the wrong one would clip the notice. `windowIsBackwards` compares DAYS and allows equality. 4 new specs including the exact reported case.
- [x] [app] **The trap worth naming:** `MakeRow` omits an absent bound, which correctly stores NULL on an INSERT but on an UPDATE would leave the old value in place — a studio clearing an end date would find the notice still expiring on it. The update writes literal `NULL` through the same raw-SQL allow-list `BlogPosts` uses for `post_at_us`, and the test asserts through the reader (after clearing, an expired notice becomes current again).
- [x] [app] Editor: an **Announcements tab** in Manage ▸ Page Content — its own child component, because a notice is title + message + a date window rather than one of the ordered content rows. Material **date pickers** per [[feedback_date_time_pickers]], active toggle, reorder, delete-with-confirm. The list names what each notice is DOING right now (*On the site now / Scheduled / Finished / Switched off*) instead of showing two dates and leaving the studio to compare them; the "show until" hint says in words that the date is exclusive. A backwards window is refused — legal in the schema, but it could never appear, so it is always a mistake.
- [x] [app] Home: the strip is pinned **above the ordered list, not inside it** — an announcement outranks the page's own arrangement, so a studio cannot bury "we are closed today" by dragging rows, and does not need to add a row to say something urgent. Non-dismissible (OQ-6): a visitor clearing a closure notice on their phone and never seeing it again is the wrong behaviour. Seam + mock (three rows: live, expired, switched off) + 7 specs.
- [x] [app] Two normalization traps, both tested: an empty window end must read as **no bound** (`microsToDate('')` is a 1970 date, which would make every unbounded notice look expired), and `active` is a Postgres `"t"/"f"`.

### 6.2 Video home item type (item 26 — OQ-7 ✅: YouTube **and** raw video URLs)
- [x] [app] Uses Phase 3's `video_url` column. Two accepted shapes, detected from the URL:
  - **YouTube** (`watch?v=`, `youtu.be/`, `shorts/`, `embed/`, `/v/`) → normalized to a privacy-enhanced `youtube-nocookie.com/embed/<id>` iframe (16:9 via `aspect-ratio`, `loading="lazy"`, `referrerpolicy`, `allowfullscreen`).
  - **Direct video file** (`https://…/name.mp4|.webm|.ogg|.ogv`, query string allowed after the extension) → native `<video controls preload="metadata">` keeping its own aspect ratio, since we cannot know it in advance.
  - Anything else is refused in the editor AND on bundle import.
- [x] [app] **The logic exists twice, deliberately:** `parseVideoUrl` (ui/src/app/shared/types/video-url.types.ts) validates what a studio types and drives the rendering; `IsSupportedVideoUrl` (page_content_bundle_section.cpp) validates what an imported bundle carries, which can arrive from anywhere and cannot rely on client-side checks. Both files carry a note that a shape added to one must be added to the other.
- [x] [app] **Security, and why the sanitizer bypass is defensible.** Angular blocks an interpolated iframe `src` by default and that default is right. `bypassSecurityTrustResourceUrl` is used in exactly one place, on a string this code CONSTRUCTED — `youtube-nocookie.com/embed/` plus an id already matched against `[A-Za-z0-9_-]{6,20}` — so nothing the studio typed survives into it verbatim. Both validators parse the **host** rather than searching the string for "youtube.com", so `https://evil.test/?x=youtube.com/watch?v=abc` is refused; there is a test for that on each side, because that is the one that would put an attacker's page in an iframe on a studio's front page. https-only on both sides, so `javascript:` / `data:` cannot reach an iframe or a `<video>`.
- [x] [app] `home-video-section` renders whichever shape the row carries, with the row's title as an optional heading. The editor offers the `video` kind (withheld while it drew nothing — an existing spec asserted that and was updated), shows the URL field only for that kind, and the hint doubles as **confirmation**, naming which of the two shapes the pasted link was understood as.
- [x] **A design fix the specs forced:** the component first derived its parsed embed in `ngOnChanges`, which only fires for TEMPLATE-BOUND inputs — three specs that assign the input directly rendered nothing. Now an input **setter**, which is also better than a getter: `bypassSecurityTrustResourceUrl` returns a new object per call, so recomputing per change-detection pass would change the iframe's `src` identity every cycle and reload the player.
- [x] Specs: a 9-shape YouTube table + 6 file shapes + 14 rejections (including both host-spoofing attempts and the `javascript:`/`data:` cases), iframe vs `<video>` per shape, no render for an unplayable or empty URL, the editor's refusal and its shape-naming hint, and three C++ bundle-import tests (refuses an unplayable link, accepts both shapes plus an empty one, refuses a spoofed host).

### 6.4 [app] Video playback options (Mason, 8/23/2026)

> *"I added a video and it worked well but it doesn't play. I have to click play. Can we have an option on whether to have the video autoplay? And an option over whether or not the closed caption shows and have that default to false? Is there anyway to turn off the youtube branding…"*

- [x] Two new `home_sections` columns — `video_autoplay`, `video_show_captions`, both **NOT NULL DEFAULT FALSE** so every existing row keeps click-to-play. Migration `app/0007_video_playback_options` (`ADD COLUMN IF NOT EXISTS` ×2, idempotent by itself). Full admin metadata; they travel in the theme bundle **with the URL and only with it** — a `video_autoplay` on a feature block would be noise, and there is a test asserting non-video rows carry neither.
- [x] **Autoplay is always MUTED autoplay.** Every current browser blocks a clip that starts with sound, so `autoplay=1` always ships alongside `mute=1` (and `muted` + `playsinline` on a direct file). The editor says so under the checkbox — a studio ticking "autoplay" and getting silence would reasonably report that as a bug.
- [x] **Captions default off, and "off" means OMITTING `cc_load_policy`** rather than sending `0`: a viewer who keeps captions on for a reason keeps them. The checkbox is disabled for a direct file (with the reason shown, not hidden) because captions are a YouTube feature and an `.mp4` carries no track.
- [x] **⚠️ The YouTube branding answer is mostly "you cannot", and the internet is out of date about it.** `modestbranding=1` was **removed by YouTube in August 2023**; `showinfo=0` was **removed in 2018**. The title, the share/settings buttons and the watermark cannot be turned off through the IFrame API at all. `rel=0` still works but since 2018 it only *limits* end-of-video suggestions to the same channel rather than removing them. We now send `rel=0`, `iv_load_policy=3` (no annotations) and `playsinline=1`, and **deliberately do NOT send the dead parameters** — with a spec asserting they are absent, so nobody re-adds them as cargo cult. **The genuinely branding-free path is a direct video file**, which this feature already supports and which renders through the browser's own player.
- [x] Specs: 6 on the embed-URL builder (nocookie host, what is and is not suppressed, autoplay↔mute always paired, captions omitted when off, id-only interpolation), 6 on the renderer across both shapes, 5 on the editor. Plus 2 C++ bundle tests (options round-trip and land as columns; non-video rows carry none) and a migration test proving the columns are usable and default to today's behaviour.
- [x] **A spec needed updating, correctly:** 6.2's renderer test pinned the embed URL as an exact string with no query. The parameters are a real addition, so it now asserts the host and video id and leaves the query string to the dedicated parameter tests — narrowed to what that test is about rather than loosened into nothing.

### 6.5 [app] Editing lists without losing your place (Mason, 8/23/2026)

> *"clicking the up down on an item moves the item in the list but refreshes the page. If I have an item at the bottom and I want it near the top, I have to click up and then it resets me to the top of the page and I have to scroll down and then click up again. […] Clicking on edit makes an edit area at the top of the page but it isn't obvious and you have to scroll up to find it. I'd rather do this in place."*

Two separate complaints, both about the editors being unusable on a list longer than a screen. Neither is specific to announcements — they applied to **every** reorderable list, so both fixes went in across all of them.

- [x] **Reorder no longer reloads.** `move()` ended with `this.load()`, which re-fetched and re-rendered the whole list and threw the scroll position away. The swap is now applied to the local array immediately and the two ordinal writes go to the server in the background — the row moves under the cursor and nothing redraws. Applied to **four** lists: home sections, Getting Started steps, announcements, and both the gallery order and photo order in Image carousels.
- [x] A FAILED write still reloads, deliberately: at that point the on-screen order and the stored order genuinely disagree and the stored one is the truth.
- [x] **⚠️ The failure-path spec caught a real bug in the fix.** `load()` clears `this.error` as its first statement, so setting the message *before* calling it wiped the explanation — a failed reorder would have snapped the list back silently. The two lines are reversed in all four places, with a comment saying why the order matters.
- [x] **Editing happens in place.** The form renders inside the row being edited, tinted and spanning the row's grid so it reads as nested rather than as the next item. ADDING still opens above the list, since there is no row for it to live in yet. The form is extracted into ONE `<ng-template>` rendered in whichever position applies, rather than duplicating ~200 lines of markup that would then drift.
- [x] Specs (11): the reorder does not call `getTableRows`; two successive moves keep the ordinals strictly ordered so a later reload agrees with the screen; a failed move resyncs AND shows its message; the edit panel is `contains`-ed by the correct row element (not merely present somewhere on the page); adding is inside no row; and only one form is ever open at a time.

### 6.3 Live hand-testing (Phase 6)
- [x] Steps written (below) — awaiting your run against a live server.

Fresh database (`knottyyoga_database_helper --recreate_database`), server + Angular dev server running, signed in as an admin.

1. **A fresh site has no notices.** Home, signed out, starts at the carousel/hero as before — no empty banner strip. Admin ▸ **Manage Products** → **Page Content** → the new **Announcements** tab: "No notices yet".
2. **Write one and see it immediately.** **Add** → Title `Closed for the holidays`, Message `We reopen on the 27th.`, leave BOTH dates blank, **Add notice**. The list badges it **On the site now** and reads "From now — until further notice". Reload **Home** signed out: the notice sits above everything, including above the hero. Sign in and reload — still there; it is not auth-dependent.
3. **It cannot be buried.** Page Content → **Home sections**: move **Photo carousel** to the very top with the arrows. Reload Home — the announcement is *still* above it. That is the point of it not being a section row.
4. **Dates retire it on their own.** Edit the notice → set **Show until** to *yesterday* → Save. The list badges it **Finished** and it is gone from Home. Now set **Show from** to *tomorrow* and clear the until date: it badges **Scheduled** and is still off Home. Clear both dates again — **On the site now**, and back on the page.
5. **A one-day notice — the common case.** Set **Show from** AND **Show until** both to *today* and Save. It stays badged **On the site now** and is still on Home: two identical dates mean that one day, which is what "construction today" or "smoke in the building" needs. Reopen it and confirm the picker still reads *today*, not tomorrow. Tomorrow it will read **Finished** on its own.
6. **The editor refuses an impossible window.** Set **Show from** to the 27th and **Show until** to the 24th: a warning appears and **Save** is disabled. Fix the order and it saves.
7. **Switched off is separate from the dates.** Untick **Active** on a live notice → badged **Switched off**, gone from Home, but the dates it was written with are still there when you edit it. Tick it back.
8. **A second notice, in order.** Add `Therapy Room closed` and use the ↑/↓ arrows to put it above the holiday notice. Reload Home: two banners, in that order.
9. **A YouTube video mid-page.** Page Content → **Home sections** → **Add** → Kind **Video**. A **Video link** field appears. Paste a real YouTube URL — the hint should change to "YouTube video — plays in an embedded player." Title it `Studio tour`, position it below **Come join the fun!** with the arrows, Save. Reload Home: the video sits there under its heading and plays. Try the `youtu.be` share link and a Shorts URL too — all three should be accepted.
10. **A direct file.** Edit that row and replace the URL with a direct link to an `.mp4` (any public one). The hint changes to "Direct video file — plays in the browser's player." Save and reload: it now plays through the browser's own controls rather than a YouTube frame.
11. **A bad link is refused, not silently blank.** Edit the row and paste `https://example.com/not-a-video`: a warning appears and **Save** is disabled. Same for a channel URL like `https://www.youtube.com/@someone`. Restore a working link.
12. **Theme file leaves notices alone.** Site Theme → **Theme file** → **Download**, then **Upload** it back (Apply). Your video row returns with its URL — but the announcements are **untouched**, because they are timely content and deliberately not part of a theme.
13. **Autoplay, and what it costs.** Edit the video row → tick **Start playing automatically** → Save. Reload Home: the video starts on its own and is **muted** — that is not a bug, it is the only autoplay a browser permits. Untick it and the video waits for a click again.
14. **Captions.** With a YouTube link, tick **Turn captions on** and reload: captions are burned on from the start. Untick and reload: they follow your own YouTube setting rather than being forced off. Now swap the URL to a direct `.mp4` — the captions checkbox greys out and explains that captions are a YouTube feature.
15. **What YouTube branding survives.** On the YouTube row you will still see the title, the share and settings buttons and the watermark. Those cannot be removed — YouTube deleted the parameters that used to do it. If a clean frame matters for a particular video, host it as an `.mp4` and it renders through the browser's own player with none of that.
17. **Reordering a long list (6.5).** Page Content ▸ **Home sections**, which is long enough to scroll. Scroll to the BOTTOM row and click ↑ repeatedly: the row walks up under your cursor and the page does not jump or redraw — you should be able to move it several positions without touching the scrollbar. Reload the page afterwards and confirm the new order is what the server actually stored.
18. **Editing in place (6.5).** Click the pencil on a row near the bottom. The form opens **under that row**, tinted, not at the top of the page — you should not have to scroll to find it. Click the pencil on a different row: the form moves there rather than a second one opening. Press **Add** instead and the form opens above the list, since a new row has nowhere to nest yet.
19. **Same everywhere.** Repeat 17 and 18 on **Getting Started steps**, on **Announcements**, and in Site Theme ▸ **Image carousels** (both the gallery order arrows and the photo order inside a carousel).
20. **Your real DB.** Run `knottyyoga_database_helper --migrate` against it: the report lists `app/0006_announcements` and `app/0007_video_playback_options`. Open Page Content ▸ Announcements and add one; it appears on Home.

---

## Phase 7 — Locations, timezone, and the About page

### 7.1 Studio-location timezone (item 5) — expose what exists
`facilities.timezone` already exists (IANA, default `America/Los_Angeles`) and drives booking emails/digests/reporting — recorded in [[Bookable Service Foundation]] (decision 10). The gap is surfacing:
- [x] [app] Manage → **Locations** page (new; locations were editable only through the debug Manage Data surface, violating the dedicated-UI rule): list + edit for `facilities` — name, code, full address (including the `address_line_2`/`country` fields the CRUD metadata omitted), **timezone** as an autocomplete over a curated shortlist with free entry, active toggle, photo, description. Dashboard tile (alphabetized per 1.3). Edits in place and lists inactive locations, following 6.5.
- [x] [app] `description` STRING nullable column on `facilities` (+ DSL + migration — **renumbered `app/0008_facilities_description`**, since 6.4 took 0007) and photo support (`photo_support_tables` += `facilities`, `PublicPhotoTables` += `facilities`, `hw-photo-upload` in the editor) — both feed 7.2.
- [x] **The `address_line_2` and `country` columns had no admin metadata at all**, so *no* UI anywhere could fill them in — not the new page, not even Manage Data. Added to `PopulateAdminColumnDataInfo` and `PopulateAdminColumnFriendlyNames` alongside the rest of the address, which is what makes "Suite 200" printable on 7.2.
- [x] **The timezone field is validated by SHAPE, not by membership of the shortlist.** A curated list of 13 US/Canada zones covers every studio on the platform today, but a studio in Lisbon must be able to type `Europe/Lisbon` rather than be told their own timezone does not exist. What is refused is the empty string and free prose ("Pacific time please") — a value the server would store and then silently fail to interpret when it formats a booking email.
- [ ] Recorded, not planned: the service-booking week strip builds days in **browser-local** time (`buildWeek()` uses `new Date()`; `setHours(0,…)`) — fine while all facilities are Pacific, wrong the day a second-timezone location exists. Noted here as the follow-up the timezone column exists to serve.
- [x] Specs: 18 on the manage page (CRUD, the two rescued columns, timezone persists, shortlist-vs-typed, the "f"-is-truthy trap, delete-refused wording), 12 on the timezone vocabulary, 5 C++ on `GetActiveFacilities` + the description column.

### 7.2 About ▸ Our Location (item 20)
- [x] [app] Public `/location` page: each active facility — photo, name, description, full address, and an **"Open in Google Maps"** button (`https://www.google.com/maps/search/?api=1&query=<url-encoded address>`, new tab with `rel="noopener"` — link-out confirmed, OQ-9 ✅). Menu: About ▸ **Our Location**.
- [x] [app] `GET /api/locations` — anonymous, ACTIVE-only (filtered in SQL), name-ordered, each row carrying every address part, the description, the timezone and `has_photo`.
- [x] **Address assembly is its own tested function**, not template string-joining: parts that are empty drop out rather than leaving `1 Main St, , Redmond`, and the state and ZIP join with a space while only the city takes a comma. The same string feeds the Maps query, so the two can never disagree.
- [x] Specs: 12 on the page (renders active facilities, photo-optional, description-optional, map href encodes the address, `target=_blank` + `noopener`, empty state, dead-feed state), 14 on the address/URL helpers, 6 C++ on the endpoint.

### 7.3 About page → alternating blocks (item 14 — OQ-8 ✅: markdown drops entirely)
- [x] [app] `db_schema/about_sections`: same shape as a feature row (`about_section_id`, `ordinal`, `title`, `body`, `link_route`, `link_label`, `active`, timestamps) + photo association (+ public allow-list). Full checklist (all six registration points) + migration **`app/0009_about_sections`**.
- [x] [app] Helper + `GET /api/about_sections` (anonymous, ordered, active, with `has_photo`).
- [x] [app] **Seeds THREE blocks, not one.** A single block looks exactly like the old one-slab markdown page and teaches a studio nothing about what the editor can do; three demonstrate the alternating rhythm the page is built around. The first carries the copy the `site_about_markdown` default used to hold, so nothing that was on the page is lost.
- [x] [app] Manage → Page Content gains an **About sections** tab (second, beside Home — the two lists that carry pictures sit together). Same editor, same reorder, same in-place editing.
- [x] [app] About page: heading + the ordered blocks through the *same* feature-section component — **no markdown**. The `<markdown>` render, the route-scoped `provideMarkdown()` on `/about`, and the `graySquare` placeholder in `styles.scss` (with its "TODO remove after building out image renders") are all gone.
- [x] **The feature block was lifted into `shared/components/feature-section/`** and takes PLAIN inputs (title/body/link/imageUrl/imageRight) rather than a row object. Two callers read from different tables whose photo URLs are built from different ids — a shared component that knew about either table would not be shared at all. `app-home-feature-section` is now a thin adapter, so the home page's markup, SCSS and responsive reading-order rules are the same ones About uses.
- [x] [app] Retired the markdown plumbing with it: the Site Theme editor's **About tab** (union / `sectionOrder` / `sectionKeys()` / the `mat-tab` / its SCSS / the `MarkdownComponent` import / the route provider) and `aboutText` in `SiteConfig` / `DEFAULT_SITE_CONFIG` / both merges, plus the now-orphaned `@shared/constants/about.ts`. **[hw] note:** the `site_about_markdown` slot itself stays registered in honuware — CommunityFinder may consume it, and it travels harmlessly in bundles; revisit at extraction time.
- [x] [app] Theme bundle: `about_sections` joins the `page_content` section (rows + image assets named `about-N.ext` from POSITION, order = array position). Round-trip, replace-not-accumulate, inactive-stays-inactive, and three refusal tests.
- [x] **The migration deliberately seeds NO blocks.** An existing site already has About copy in `site_about_markdown`; converting one blob of markdown into blocks would either mangle it or duplicate it beside what the studio then writes by hand. The editor starts empty and the old text is still in the theme bundle to copy across — hand-testing step 12 covers it.
- [x] Specs: 13 on the About page (alternation from position, photo-optional, no `<markdown>` anywhere, empty + dead-feed states, tenant naming), 8 on the shared block, 9 on the Page Content About tab, 9 C++ on the table helper, 5 on the endpoint, 7 on the bundle, 3 on the migrations, 3 on the seed.

### 7.4 Live hand-testing (Phase 7)
- [x] Steps written (below) — awaiting your run against a live server.

Fresh database (`knottyyoga_database_helper --recreate_database`), server + Angular dev server running, signed in as an admin.

1. **The new tile.** Admin ▸ **Manage Products**. The tiles are alphabetical; **Locations** sits between *Instructors* and *Membership Tiers*, with a map-pin icon and the subtitle "Your address, photo, description and timezone". Open it.
2. **What a fresh database seeds.** One row: **Knotty Yoga Studio**, address `123 Main St, Los Angeles, CA 90001`, badges reading `knotty-yoga-studio`, **Pacific — Los Angeles** and **Shown**, and a dashed placeholder where the photo goes.
3. **Edit in place.** Click the pencil. The form opens **under that row**, tinted — not at the top of the page. Set **Suite / unit / floor** to `Suite 200` and **Country** to `USA`, then Save. The row's address line now reads `123 Main St, Suite 200, Los Angeles, CA 90001`. Those two fields had no admin metadata before this phase, so this is the first time either has been fillable anywhere in the app.
4. **The timezone picker.** Edit again and click into **Timezone**. A shortlist drops down — Hawaii through Newfoundland. Pick **Mountain — Denver** and Save: the row badge changes to *Mountain — Denver*.
5. **A zone that is not on the list.** Edit, clear the field and type `Europe/Lisbon`. No warning appears and **Save** is enabled — the shortlist is a convenience, not the vocabulary. Save, and the badge reads the raw `Europe/Lisbon` (there is no friendly name for it, which is correct).
6. **Free prose is refused.** Clear it and type `Pacific time please`. A warning appears explaining the Area/Location form, and **Save** is disabled. Now clear the field entirely — still refused. Set it back to **Pacific — Los Angeles** and Save.
7. **A photo and a description.** Edit the row, upload any photo, and write a description (a sentence about parking is the realistic case). Save. The row now shows the thumbnail instead of the placeholder, and the description underneath.
8. **About ▸ Our Location.** The **About** menu now has a fourth item, **Our Location**, after Gallery. Sign out first, then open it — this page is for people who have no account yet. The photo, the name, the description and the full address are all there.
9. **The Maps button.** Click **Open in Google Maps**: a **new tab** opens on Google Maps searching your full address, suite number included. Come back and check the address on the page reads as one clean line with no doubled commas.
10. **Switching a location off.** Back in Manage ▸ Locations, **Add** a second location (`The Annex`, code `annex`, any address) and Save. Reload About ▸ Our Location: both appear, in **name order** — The Annex first. Now edit The Annex, untick **Show this location on the website**, Save. The manage list badges it **Hidden** and keeps it; the public page drops it. That is the difference between switching off and deleting.
11. **Deleting says why it cannot.** Try to delete the seeded **Knotty Yoga Studio** (rooms and class slots reference it). The confirmation appears; confirm, and the error explains it is still referenced and suggests switching it off instead — rather than "please try again", which would never work. Delete **The Annex** instead: that one goes.
12. **About is blocks now.** Open **About Knotty Yoga**. Three blocks, alternating: the first with its image on the left, the second on the right, the third on the left again. No grey square anywhere. Each block's button goes where it says — the third one's **Our location** lands on the page from step 8.
13. **Editing them.** Manage Products ▸ **Page Content**. There are **four** tabs now: Home sections, **About sections**, Getting Started steps, Announcements. Open About sections: three rows, each with a thumbnail slot, in page order.
14. **Add a block with a picture.** **Add** → Title `Come train with us`, a Body, Link route `/start`, Link label `Get started`, Save. Reopen it with the pencil and upload a photo. Reload the public About page: the new block is at the bottom and its image is on the **right**, continuing the alternation — the sides come from position, not from a stored flag.
15. **Reorder and watch the alternation follow.** Move that fourth block to the top with ↑. The page does not jump while you click. Reload About: it is first, image on the **left** now, and every block below it has flipped. That is the point of computing the side from position.
16. **A block with no picture.** Add a block and do NOT give it a photo. On the public page it renders as copy across the full width rather than as text stranded in half the page beside an empty gap.
17. **Switching a block off.** Untick **Show this block on the About page** on one of them. It stays in the editor badged **Hidden** and disappears from the public page.
18. **The Site Theme About tab is gone.** Manage Products ▸ **Site Theme**. The tabs read Brand basics, Copy, Colors, Fonts, Image carousels, Theme file — **no About tab**. There is nowhere left on the site to type markdown, which is the intent.
19. **Theme file carries the blocks.** Site Theme ▸ **Theme file** → **Download**. Now go back to Page Content ▸ About sections and delete a block. Return to Theme file and **Upload** the file you downloaded (Apply). Reload About: the deleted block is back, with its photo, in its original position.
20. **Your real DB.** Run `knottyyoga_database_helper --migrate` against it: the report lists `app/0008_facilities_description` and `app/0009_about_sections`. Open Manage ▸ Locations — your real address is there, with an empty description and timezone `America/Los_Angeles`; fill in the description and add a photo. Then open Page Content ▸ **About sections**: it is **empty**, deliberately — the migration does not convert your existing markdown, because splitting one blob into blocks would either mangle it or duplicate it. Your old About text is still in a theme file you downloaded before this deploy (and in `config_secrets`), so copy it across by hand into as many blocks as it wants to be.

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