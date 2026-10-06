---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 10/5/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I want you to make mobile look functional. Right now, many of the pages in the system look terrible on mobile. I would like you to put forth a plan to exercise the various pieces of the UI, do screen captures, look at the UI to see if the spacing or layout is off, and make the CSS / layout changes to fix this. I would like you to put forth a plan to get through to the top level pages first that are directly, publicly accessible, make the changes and then work on the flows involving having an account and being logged. Then work on the admin, staff, and user portal flows. We need to have various phone / tablet sized targets to check this out with. I'd also like to have you fully complete these stages, once defined, with little to no interaction from me minus design choices that really need my input. I read that you can spin up a browser and do things like this. Please put together a multi phase plan to accomplish this. Please use the code base and these documents for context:
- [[Component Inventory for Designer]]
- [[Website Makeover]]

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Plan

> **Status (10/5/2026): plan written, no implementation yet.** Phases run 0 → 6 in order. Phase 0 builds the screenshot/audit harness everything else depends on; Phase 1 fixes the shared layers every page inherits; Phases 2–5 walk the pages in the order the Overview asks for — public, then account, then staff, then manage/admin. Each phase ends with a full audit run whose numbers get recorded here. **All ten open questions are resolved (10/5) — every default accepted — and the plan text below states them as decisions. Ready to start Phase 0.**

## How this works — the loop for every page

1. **Shoot it.** The Phase 0 harness opens the page in headless Chromium at every target size (§0.3), saves a full-page screenshot, and runs automated layout checks (§0.4). I read the screenshots myself — the image files are readable to me — so no human screenshotting is needed.
2. **Fix it** in the lowest layer that owns the problem: a token or mixin before a shared class, a shared class before a shared component, a shared component before a page. A fix that would be copy-pasted onto three pages belongs one layer down.
3. **Prove it.**
   - The audit for that page reports **zero horizontal overflow** at every phone width, and no new tap-target or input-font findings.
   - A **geometry spec** pins the fix, asserting `getBoundingClientRect()` (nothing wider than the viewport, columns actually stacked, the button actually full width). Presence-only specs are not accepted — Polish Phase 13 records two layouts that passed presence specs while visibly broken. Where it lives depends on what drives the layout (learned in §1.1):
     - **Container-driven** (flex/grid that reacts to its own width): a Karma `*.component.spec.ts` that sizes the host — `site-theme.component.spec.ts:237` is the template.
     - **Global classes behind a media query**: a Karma spec using the iframe-of-exact-width helper in `design-tokens.spec.ts`.
     - **A component's own media query**: Karma can't do it — the test window can't be resized, and a TestBed component can't be mounted in an iframe. These get a **Playwright layout spec** at a real phone viewport, `e2e/mobile-audit/layout/*.spec.ts`, run against the mock app like the audit.
   - The **desktop (1280) screenshot is unchanged** against the Phase 0 baseline, or the change is intended and noted. Mobile work must not quietly break desktop.
   - `ng test`, `ng lint` and `ng build --configuration=production` pass.
4. **Tick it** here, with the audit numbers.

**What I run myself:** `ng serve`, the harness, `ng test`, `ng lint`, `ng build`. This is frontend-only work: no C++ changes are expected, so nothing for you to build. No git commands. If a C++ change turns out to be needed, it gets its own item and I'll say so.

## Decisions already made that this plan follows

From `Website Makeover.md` and `Component Inventory for Designer.md` — not re-asked here:

- **Mobile-first CSS**: default styles target the phone, `md:`/`lg:` add desktop. 44px tap targets, 16px minimum input font (below that iOS zooms the page on focus), safe-area insets on anything sticky, no hover-only affordances.
- **`md` (768px) is the collapse line.** Tailwind's default screens (640/768/1024/1280) **stay** — the Makeover's `sm:375 / lg:1280` change is not adopted (OQ-2, decided).
- **Navigation:** keep the hamburger and the full-screen `header-mobile-menu`; no bottom tab bar (Makeover OQ 15).
- **Tables (Makeover OQ 18):** `/manage` and `/admin` tables scroll horizontally with a sticky first column; customer lists (`my-events`, `purchase-history`, `my-vouchers`, cart, dashboard alert lists) collapse to cards.
- **Calendar (Makeover OQ 19):** day view below `md`; week/month are desktop views. Same as Polish 13.1.
- **Back office is not redesigned** — it inherits the shared blocks, made to work at phone width, nothing more.
- **Tokens only.** No new literals (`ui/CLAUDE.md`), no `--color-*` set; dark mode and per-studio theming will override tokens later, so a literal now is a bug later.
- **Don't wait for Ryan's 375 frames** (OQ-10, decided). Make today's layouts work on phones now. When his mobile frames arrive they replace these layouts, and this plan's harness and geometry specs become the safety net for that port. No page is held back for him.
- **Polish before deploying Phase 13** (13.1–13.6) is folded into this plan — §2.4, §2.5, §2.7 — and gets ticked there too when done.

## Target sizes

| Name | CSS px | Why |
|---|---|---|
| `phone-small` | 360 × 780 | Most common small Android width (Galaxy S-class). The tightest real target. |
| `phone-se` | 375 × 667 | iPhone SE / mini. Short screen — catches sticky bars eating the page. Ryan's design width. |
| `phone` | 390 × 844 | iPhone 13/14/15. The typical phone. |
| `phone-large` | 430 × 932 | iPhone Pro Max / Plus. |
| `tablet` | 768 × 1024 | iPad mini portrait — exactly the `md` line, where most layouts switch. |
| `tablet-landscape` | 1024 × 768 | iPad landscape — wide enough for desktop layouts, short enough to break them. |
| `desktop` | 1280 × 800 | Regression baseline only. |

Phones are emulated as real mobile devices (touch, device scale factor 2–3, mobile user agent) via Playwright's device descriptors, not just narrow windows — so `hover` media queries and touch behave as on a phone. **Emulation is not a real device**; §6.4 covers the one real-device pass that remains yours.

---

## Phase 0 — Tooling and baseline

> Nothing visual changes in this phase. It builds the instrument, then measures every page once so later phases have a "before" to beat.

### 0.1 Mock personas (lowest layer: the data the pages render)
Today `ServerAccessMock` starts logged in as an admin (`userInfoDefault`, `ServerAccess.mock.ts` ~408–420), so a public page always renders with the admin header and there is no way to see what a customer or a logged-out visitor sees.
- [x] Add a persona switch to the mock: `anonymous`, `customer`, `staff`, `admin` (default stays `admin`, so nothing changes for anyone running `ng serve` today). Selected by a `localStorage` key the harness sets before the app boots; mock-only — `ServerAccessNetwork` is untouched and nothing ships to production. ✅ 10/5 — `MockPersona`, `readMockPersona`, key `knottyyoga.mockPersona`. Every signed-in persona is the **same person (id 1)** with different roles, so their purchases/schedule still have data; `anonymous` starts signed out and a login from the login page then makes it an ordinary customer.
- [x] Each persona gets the roles/permissions that make the real guards (`AuthGuard`, `StaffGuard`, `AdminGuard`, `ManageProductsGuard`, `AuthorBlogGuard`) and the header menu behave as for that kind of user. ✅ `MOCK_PERSONA_ACCESS` — staff is `instructor` + `provider` (passes `StaffGuard`, not admin/manage).
- [x] Specs in `ServerAccess.mock.spec.ts`: each persona's login state, roles and permissions; default is still admin. ✅ 7 specs, run through the **library's own `AuthService` + `hasStaffAccess`/`hasManageProducts`** — what the real guards see, not a re-statement of the roles table.

### 0.2 Stress data in the mock
Layouts break on long content, and the mock is mostly tidy short strings.
- [x] Add a few deliberately hard records reachable from the main lists. ✅ 10/5 — **opt-in**, behind a second key (`knottyyoga.mockAuditData` = `1`), because seeding by default would shift ids that ~600 existing mock specs assert (`purchase.id === 1`, empty lists). Built through the mock's own `createPurchase` → `purchasePayCard` → `createSubscription`, so the records have exactly a real local-mode checkout's shape:
	- product 6, *Restorative Partner Acro & Thai Massage Intensive Weekend*, long description, $12,345.00;
	- purchase 1 — paid, three lines (product 6 ×2, a 90-min variant, a plain product) and a long portal note;
	- subscription 1 — active Gold Membership.
	- That fills `/my/purchases/1` and `/my/subscriptions/1`, which **404 on a fresh mock**. The cart is filled per-state (§0.5). More hard records get added as Phases 2–5 reach the pages that need them.
- [x] Specs for any mock method whose output changes. ✅ 5 specs: off by default (purchase 1 is a 404, product 6 absent), on → purchase paid in full with product 6, subscription active, and it seeds even for `anonymous` while leaving it signed out.

### 0.3 The harness: `ui/e2e/mobile-audit/`
- [x] Add `@playwright/test` as a **dev** dependency and install its Chromium (OQ-1, approved). Nothing in the shipped bundle changes. ✅ `@playwright/test` 1.63.0, pinned exact.
- [x] A Playwright config with the seven target sizes as projects, starting `ng serve` (mock mode) itself and freezing the clock (`page.clock.setFixedTime`) so date-driven pages render identically every run. ✅ `playwright.config.ts` + `viewports.ts`: phones get touch, DPR 2–3 and a mobile UA; port **4300** so it never fights a dev server on 4200 (reuses one already there); clock frozen at **Wed 14 Oct 2026 10:00 Pacific**, timezone `America/Los_Angeles`, reduced motion.
	- The app scrolls inside an inner column (`app.component.html`), so Playwright's "full page" screenshot captured **one screen**. The harness lets that column grow for the screenshot only — after the checks have run on the real layout — keeping horizontal clipping so the picture shows what a phone shows.
	- A 10,000px page downscaled to one image is unreadable, and the crushed series cards (§0.6) looked fine in it. Each page is also saved as **`<page>.part-NN.png` slices of ~two screens** — those are what get reviewed.
- [x] Output (screenshots, JSON, HTML report) goes to a git-ignored folder. ✅ `ui/e2e/mobile-audit/{output,report,test-results}` in the root `.gitignore` (the repo has no `ui/.gitignore`). Each case overwrites only its own files, so a filtered run keeps the rest.
- [x] npm scripts. ✅ `audit:mobile`, `audit:mobile:phones`, `audit:mobile:self-test`; `AUDIT_TIER` / `AUDIT_ROUTE` / `AUDIT_STRICT` env filters; `--project=<size>`. All in `e2e/mobile-audit/README.md`.

### 0.4 The automated checks
Run on every page at every size, so I am not relying on eyes alone. ✅ `layout-checks.ts`; `summarize.ts` rolls results into `output/summary.md`.
- [x] **Horizontal overflow** (error) — measured per element; only the outermost overflowing element of a chain is reported, with the Angular component it is in.
	- ⚠️ **The first baseline reported zero overflow on all ~800 cases, and that was a bug in the check, not good news.** The app's page column is `overflow-y: scroll`, and CSS silently makes its `overflow-x` compute to `auto` — so the whole page looked like a "deliberately scrolling box" and everything in it was exempt. Fixed: a vertical scroller never counts as a deliberate horizontal container; an explicit horizontal scroller does; a clipping box only when narrower than the screen. Self-test case added for exactly that shape.
- [x] **Tap targets** < 44 × 44 (warning). Inline links in running text are exempt (WCAG's inline exception); a control nested in a control is measured once.
- [x] **Input font size** < 16px on phones (error).
- [x] **Dialogs and menus** larger than the viewport (`overlay-overflow`, error).
- [x] **Text clipped** in an `overflow: hidden` box without ellipsis (warning).
- [x] **Added — cramped text** (warning). Found by *looking* at the first home-page screenshot: the series cards weren't too wide, they were **crushed** — the title column squeezed to a word per line. No overflow check can see that. Flags text wrapping to 3+ lines at under 10 characters a line, measured from the browser's real line boxes.
- [x] **Added — overlap** (error). Same screenshot: the "Join from today" / "Book this date" buttons are drawn **on top of** the dates. Flags a button/link covering text in the same layer (an open menu or dialog covering the page is excluded — that is its job).
- [x] Results per route × size as JSON, plus `summary.md` (errors per tier × size, findings by kind, worst components, worst pages).
- [x] **Self-test**: `layout-checks.spec.ts`, 5 tests. One fixture plants each problem **and a look-alike that must not be reported** (scroll wrapper, carousel, aria-hidden drawer, inline link, ellipsis text, button beside text, the open mobile menu), and pins the exact set found. Verified it catches regressions: breaking the shell rule fails it.

### 0.5 Route manifest
- [x] One file listing every route. ✅ `e2e/mobile-audit/routes.ts` — **113 routes**: public 20, auth 3, account 25, staff 11, manage/admin/blog-admin 54. Concrete ids for every parameterised route, each checked against the mock to render real content. `/verify` navigates to `/login` on success *and* failure, so it has no resting page — recorded as an expected redirect (`expectRedirectTo`) rather than a failure.
- [x] A spec that fails when a route is missing. ✅ `src/app/mobile-audit-routes.spec.ts` (runs in `ng test`): walks the real router config, resolving every lazy `loadChildren`, and fails on a route missing from the manifest, a stale manifest entry, duplicates, unaccounted redirects, an unfilled `:param`, or a persona that can't pass the route's gate. Guards against its own vacuity (must find 100+ routes). Verified by deleting one entry → fails.
- [x] Interaction states. ✅ `states.ts`: mobile menu open **with a submenu expanded** (phones/tablet below `md`; the page under the menu is excluded from that case), calendar **day / week / month**, cart **with two items** (one deliberately long). **Changed from the plan:** dialogs are *not* all added up front — each dialog gets a state when its page is worked on in Phases 2–5, so it is audited from then on. Doing ~40 dialogs blind now would mean driving each one without knowing yet what it needs. Checkout with the Square form is covered by the plain checkout route (it renders the card form on load).

### 0.6 Baseline run
- [x] Full audit at all seven sizes. ✅ 10/5 — **823 cases** (117 route/state combinations × 7 sizes, 3 skipped by design: the menu-open state above `md`), all ran, **7.0 min**. Desktop screenshots + this summary saved to `e2e/mobile-audit/baseline/` (git-ignored) as the regression "before".

**Pages with layout errors** (overflow, overlap, overlay overflow, input font) — cases with errors / cases:

| Tier | 360 | 375 | 390 | 430 | tablet 768 | tablet 1024 | desktop 1280 |
|---|---|---|---|---|---|---|---|
| public | 3 / 24 | 3 / 24 | 3 / 24 | 4 / 24 | **23 / 23** | 1 / 23 | 1 / 23 |
| auth | 0 / 3 | 0 / 3 | 0 / 3 | 0 / 3 | **3 / 3** | 0 / 3 | 0 / 3 |
| account | 2 / 26 | 2 / 26 | 2 / 26 | 1 / 26 | **26 / 26** | 0 / 26 | 0 / 26 |
| staff | 1 / 11 | 1 / 11 | 1 / 11 | 1 / 11 | **11 / 11** | **11 / 11** | 0 / 11 |
| manage | 11 / 54 | 10 / 54 | 10 / 54 | 9 / 54 | **54 / 54** | **54 / 54** | 0 / 54 |

**Findings on phones (all four sizes), by kind:** overflow 65 · overlap 28 · input-font 8 · overlay-overflow 0 · text-clipped 0 · cramped-text 166 (warning) · tap-target 1,798 (warning).

**How to read it.**
- **The tablet column is one bug, not 117.** At 768px the desktop menu bar doesn't fit: labels wrap to two lines, *Your Calendar* runs off the right edge, and **the logo is pushed out of view**. At 1024 the admin/staff menus (more items) still overflow; the customer menu just fits. → §1.7: hamburger below 1280 (**OQ-11, decided**).
- **Phones look better than they are.** Most of what is wrong on a phone isn't *too wide* — it's **crushed** (cramped-text, 166) or **drawn over** (overlap, 28). The screenshots confirm it: the home page's series cards and Our Classes' class cards squeeze their text to a word per line, and on the series cards the Join/Book button sits on top of the dates. Neither shows up as overflow; both are what "looks terrible on mobile" means.
- **Tap targets (1,798)** are almost all back-office icon buttons and table actions. §1.2's global minimum will remove most in one rule; the remainder are decided per page.

**Spot-checked by eye** (375px slices): home (series cards crushed + overlapped — confirmed), Our Classes (class cards crushed; *This week* / *Next week* buttons wrap mid-label), manage dashboard at 768 (header broken as above; `.portal-card`'s fixed 300px leaves two columns and a dead strip).

- [x] Re-scope Phases 2–5 from what the baseline shows. ✅ — what the audit added or sharpened, by where it lands:
	- **§1.3:** `.portal-card` fixed width confirmed on the manage and staff dashboards.
	- **§1.5:** `app-offering-highlight` is the **series/workshop card** used on home *and* `/events` — crushed text **and** a button over the dates. One shared fix, two pages. (Already listed; now known to be the biggest public offender.)
	- **§1.7:** the header at tablet widths (above) — the single largest finding.
	- **§2.4:** Our Classes: crushed class rows + wrapping week-nav buttons.
	- **§2.5:** calendar: `app-calendar-home` overflows (the `min-w-[50rem]` week/month container), `app-month-view` draws day buttons over text, and on week view the header overlaps.
	- **§2.6 / §2.7:** `/gallery` — `app-image-carousel` controls over the caption.
	- **§3.2:** `/shop/service/4` overflows (service booking slot grid).
	- **§3.4:** `/my/purchases/1` — `app-seat-assignment` overlaps text; `/my/purchases`, `/my/subscriptions`, `/my/upcoming-offerings` crushed.
	- **§4.1:** `/staff/check-in` — overlap. `/staff/schedule` crushed (7-column week grid).
	- **§5.1:** the library's `hw-table-view-control` overflows and crushes on `/admin` and `/admin/tables/…` → a library fix.
	- **§5.2 / 5.3:** overflow on `/manage/instructor-load`, `/manage/events/create`, `/manage/subscriptions` and `/1`, `/manage/entitlements`, `/blog-admin`; overlap on `/manage/close-classes`; **input font under 16px on the blog editor** (`/blog-admin/new`, `/edit/:id`).
	- **Nothing dropped yet.** The code survey's dialog hotspots (fixed 420/460/780px) don't appear because dialogs aren't open in a plain page load — they get states as each page is worked (§0.5), and §1.4 fixes the widths regardless.

### 0.7 Phase 0 verification
- [x] ✅ 10/5 — `ng test` **3485 passed**, `ng lint` clean, production `ng build` passes, harness type-checks (`npm run audit:mobile:typecheck`, new `e2e/mobile-audit/tsconfig.json` — Playwright only transpiles, ESLint only covers `src/`), self-test **5/5**, route-manifest spec **8/8**.
- [x] **Unplanned fix, found by the full suite:** `InstructorLoadComponent`'s default "next 30 days" range added `30 × 24h` — across the Nov 1 fall-back that lands at 23:00 the day before, so the **date picker showed the range a day short** while the request rounded to the right day. Its spec started failing on Oct 2 for the same reason (the third DST bug this week — see `Deploying to AWS.md` 6.2). Now 30 *calendar* days; the spec pins the clock to 10/14/2026 so the DST case runs every day, not only in October, and fails if the old arithmetic comes back (verified).

**Phase 0 complete.** Next: Phase 1, starting with §1.1. OQ-11 is decided (hamburger below 1280), so nothing blocks it.

---

## Phase 1 — Shared foundation

> Every page inherits these, so fixing them first removes a large share of the per-page problems in one move. Ordered bottom-up: tokens and mixins → global classes → shared components → library → shell.

### 1.1 Breakpoint mixin and tokens
Today component SCSS uses six different breakpoints (600, 639, 700, 767, 768, 900px) and no shared definition.
- [x] `src/assets/styles/mixins/_breakpoints.scss` with `below(md)` / `at-least(md)` etc., values matching Tailwind's screens so SCSS and Tailwind classes switch at the same width. ✅ 10/5 — `bp.at-least(name)` / `bp.below(name)` over `sm 640 · md 768 · lg 1024 · xl 1280`; an unknown name is a compile error. `below` is `width < X`, `at-least` is `width >= X`, so the two never overlap or leave a gap.
- [x] Tokens still open from the Makeover. ✅ `--touch-target-min: 44px`; `--safe-area-top|right|bottom|left` over `env(safe-area-inset-*, 0px)`.
	- **`viewport-fit=cover` deliberately NOT added to `index.html` yet.** Without it `env()` reports 0 everywhere; with it, landscape content slides under the notch unless the shell is padded. Nothing is pinned to an edge until Phase 3's sticky pay bar — **§3.2 adds the opt-in together with the bar** that needs it.
- [x] Move the existing `@media` rules onto the mixin. ✅ **19 rules in 16 files** (the survey's "17" missed two in `_patterns.scss`). No raw `@media` left in app SCSS. Odd widths mapped to the nearest shared line — 639 → `below(sm)`, 767/768 → `below(md)`; **600 → `sm`** (schedule-template-editor, announcements-editor, our-classes), **700 → `md`** (offering-highlight, site-fonts, our-classes), **900 → `lg`** (blog editor's side-by-side panes, which need the room). Those moved by up to 124px; their pages are re-checked in §1.8's audit at every size.
- [x] Extend `design-tokens.spec.ts`. ✅ 4 new specs. The breakpoint ones render probes inside an **iframe of an exact width** (its own viewport, so media queries fire — a Karma window can't be resized) at 639/640, 767/768, 1279/1280: the mixin (via `.form-grid`) and Tailwind's `sm:` flip at the same pixel. Verified non-vacuous: moving the mixin's `sm` to 600 fails it.
- [x] `ui/CLAUDE.md`: "Breakpoints" paragraph next to the layout mixins.

### 1.2 Global base rules
- [x] Inputs, selects and textareas at least 16px on phones. ✅ `_html-overrides.scss`, below `md`, `font-size: max(1rem, 1em)` on text inputs/textarea/select (not checkbox/radio/range/file/color). Element selectors so a component that sizes an input *larger* still wins.
	- ⚠️ **The spec caught a real bug in the first version:** Tailwind's preflight (`font-size: 100%` on form controls) is emitted *after* our styles at the same specificity, so a bare `textarea`/`select` rule silently lost. Fixed with one attribute selector (`:not([hidden])`), commented as load-bearing.
- [x] Minimum tap-target size for Material buttons and icon buttons on touch devices. ✅ **No CSS needed — and the audit was overcounting.** A probe of the real DOM showed Material draws buttons 36px tall but gives each an invisible **48px `.mat-mdc-button-touch-target`** (same for checkbox, radio, slide-toggle), and a `mat-select`'s trigger is its whole form field. The audit's tap-target check now measures that real tap area; the remaining warnings are app-made controls, fixed per page as they come up. Self-test case added (a 36px button with a 48px overlay is *not* reported).
- [x] Long unbroken strings wrap. ✅ `overflow-wrap: break-word` on `body` — breaks only a word that can't fit at all. Deliberately not `anywhere`, which also shrinks min-content and would squeeze auto-layout table columns to a letter wide.
- [x] Specs at 375px for each rule. ✅ In `design-tokens.spec.ts`, via an **iframe of exact width** (its own viewport, so media queries fire): inputs 16px at 375 / unchanged at 1280 / checkboxes untouched; a long email stays inside a 200px box.

### 1.3 Shared pattern classes (`_patterns.scss`, `_tables.scss`, `_layout.scss`)
- [x] `.portal-card` fluid. ✅ New shared **`.portal-grid`** (`layout.grid`, auto-fill `minmax(17rem, 1fr)`) replaces the identical `.dashboard-cards` flex row copied into the manage and staff dashboards; the tile lost its `width: 300px`. One full-width column on a phone, as many ~272px columns as fit above. Geometry specs in `manage-dashboard.component.spec.ts` (container-driven, so a sized host works): 1 full-width column at 375, ≥ 2 columns spanning the row at 768.
- [x] `.data-table-scroll` sticky first column. ✅ Works for `.data-table`, a plain `<table>` or a mat-table; the pinned cell gets an opaque fill so scrolled columns don't show through. Spec (iframe at 375): a 900px table stays inside the phone, really scrolls 300px, and the first column doesn't move.
	- **Scroll hint dropped.** The pure-CSS "scroll shadow" trick needs transparent table cells (ours are filled), and a fade mask would stay visible after scrolling to the end without scroll-driven animations, which Safari lacks. The pinned first column already shows there is more to the right.
- [x] Check `.page-header`, `.filter-row`, `.filter-bar`, `.form-actions`, `.form-grid`, `.field-row` and the four page containers at 360px. ✅ Measured by §1.8's audit across every page that uses them: no finding at any phone size names one of these classes as the overflowing or overlapping element — they already wrap/stack (the Makeover built them that way). Nothing to fix.
- [x] A reusable card-list pattern for the customer tables. ✅ **`.data-table--stack`**: below `md` each row is a card and each cell draws its column name from `data-label` beside the value; the header is visually hidden but stays in the DOM (screen readers keep real table semantics); a cell with no label (actions) spans the card. From `md` up it's an ordinary table. Spec (iframe at 375 and 1280).
- [x] Style-guide additions. ✅ "Wide table — scrolls on a phone, first column pinned" and "Stacked table — rows become cards"; 2 specs in `style-guide.component.spec.ts` (sticky first column present; every data cell's `data-label` matches its header — the wiring a page copies).

### 1.4 Dialogs
No dialog sets `maxWidth`, and several force a width or `min-width` wider than a phone: 780px (`class-schedule-manage.component.ts:540`), 460px and 420px (`today-classes`, `upcoming-classes`, `my-schedule`, `calendar-navigation.service`), `min-width: 360px` in three dialog stylesheets.
- [x] App-wide dialog defaults. ✅ **Correction to the survey: dialogs weren't overflowing — they were squeezed.** Material caps every dialog at `max-width: 80vw`, set *inline* on the pane, so the 420/460/780px dialogs became 300px-wide columns on a 375px phone. New `angular-material-overrides/_mat-dialog.scss`: below `md`, every dialog may use the full width minus 16px a side. `!important` is required to beat Material's inline style, and is confined to below-md so desktop dialogs are unchanged.
- [x] Below `md`: form dialogs go full-screen; short confirm/info dialogs stay centred at full width. ✅ New `shared/dialog-config.ts` — **`formDialogConfig({ data })`** adds the `dialog--form` panel class (keeps any existing class and the width). Applied at all **15 form-dialog call sites**: class / instance ×2 / series-run / schedule ×2 / slot ×2 / specialty-cost ×2 / migrate-product / substitute-instructor / cancel-occurrence (class schedules); assign-skill (person skills); request-class-transfer (shift requests). Left as plain dialogs: confirm dialogs, prerequisite / skill-requirement / attendance (info), revoke-skill and exception-note (one short field). No dialog wraps its fields in a `<form>`, so `:has(form)` couldn't do this automatically.
- [x] Remove the fixed widths and min-widths. ✅ The `width:` values stay — they are right on desktop, and the phone rules override them below `md`. The three `min-width: 360px` stay on desktop (they give those content-sized dialogs their width) and drop to `0` below `md`. Not `min(360px, 100%)`: a percentage min-width inside a content-sized dialog can resolve to 0 and shrink the **desktop** dialog.
- [x] Specs. ✅ `dialog-config.spec.ts` (4 — incl. the class landing on the real overlay pane the stylesheet targets). And the first **Playwright layout spec**, `e2e/mobile-audit/layout/dialogs.spec.ts`, because this is a global media query Karma can't exercise: *Add class* opens full-screen at 360 and 375 and as a centred dialog at 1280; a delete confirm on `/blog-admin` stays 16px clear of both edges with its cap at viewport − 32px. Verified non-vacuous (weakening either rule fails 4 of 5).

### 1.5 Shared components (`src/app/shared/components`)
Lower than any page, used by many:
- [x] Shared components, audited inside their host pages at each phone size. ✅ 10/5 — the baseline had phone findings in three; the others (`public-event-card`, `event-session-card`, `calendar-event`, `book-for-selector`, `event-session-price-override-editor`) had none, and the home page's event cards were checked by eye.
	- **`offering-highlight`** (the series & workshops card — home and `/events`, the biggest public offender): below `md` the action column was `width: 100%` in a row that **didn't wrap**, so it stayed on the same line — the text column was crushed to a word per line and the Join/Book button was painted over the dates. The row now wraps below `md`: photo + text on one line, a full-width button on its own line under them. Playwright layout spec `layout/offering-highlight.spec.ts` (home and `/events`, at 360/375/768/1280: text column > 55% of the row, action below the text, button > 80% wide on phones; action at the row's end from `md`). Verified: without the fix the 4 phone cases fail.
	- **`seat-assignment`** (`/my/purchases/1`): the search hint wraps at phone width, and Material's one-line subscript let it paint over the *Set up sharing* link — the known `mat-hint` trap (project memory). `subscriptSizing="dynamic"`. Karma geometry spec at a 300px host (it wraps on container width): asserts the hint *did* wrap and that its bottom is above the link's top. Verified: fails without the fix.
	- **`image-carousel`** (`/gallery`, 430px): the next-arrow over the partly-visible next card — **by design** (the peeking card is the "more" cue), reviewed by eye. Recorded as a route exemption in `routes.ts` with the reason.
- [x] The Square card form containers. ✅ The card iframe fits at 360px (checked by eye on checkout). The same screenshot found the real problem next to it: the **payment-method toggle** clipped *Card on File* to "Card on Fil…" — a flex item won't shrink below its content. `min-width: 0` on the toggles, and below `sm` the decorative icons are hidden (the labels say the same thing; the selected one keeps Material's check). Playwright spec `layout/payment-method.spec.ts`: neither label is clipped and both toggles sit inside the group at 360/375/1280. Verified: fails without the fix.
- **New harness module:** `e2e/mobile-audit/harness.ts` — persona/clock/seed setup (`openAs`) shared by the audit and the layout specs, so they cannot drift apart.

### 1.6 `@honuware/ui` library components
The library (`C:\Users\mason\source\repos\honuware-web-components`) has **no** `@media` rules today. Its pages and components are used across every tier.
- [x] Audit and fix in the library repo. ✅ 10/5 — re-measured with the corrected tap check, the library's only phone problems outside the CRUD tables (Phase 5, `hw-table-view-control`) are in the **auth pages**: the show/hide-password eye was **24×24** (login, register ×2 — fiddly, right next to the field you're typing in) and *Create an account* was 24px tall. Both now 44px (the eye's box grows but stays transparent — the icon looks identical). `hw-confirm-dialog` needed nothing of its own: it's a MatDialog, so §1.4's global rules cover it. `hw-photo-upload` and the form controls had no findings.
- [x] Library specs. ✅ 3 new specs (login: eye 44×44, link 44px; register: both eyes 44×44). Library suite **472 passed**, lint clean, `ng build honuware-ui` passes. Verified non-vacuous: reverting the login sizes fails 2.
- [x] ✅ **Released 10/5 and pulled in:** `npm view @honuware/ui version` → `0.1.3`; app on `"@honuware/ui": "0.1.3"` (exact). `ng test` 3503 passed, production build passes, and the auth pages now have **zero findings at all four phone sizes** (were 6 tap-target warnings). **Release `@honuware/ui` 0.1.3 — yours (OQ-6).** I've bumped `projects/honuware-ui/package.json` to `0.1.3`; nothing else to edit. In `C:\Users\mason\source\repos\honuware-web-components`:
	1. `git add -A && git commit -m "Release 0.1.3: 44px tap targets on the auth pages"`
	2. `git push`
	3. `git tag v0.1.3`  ← create the tag before pushing it
	4. `git push origin v0.1.3`  ← this is what publishes (GitHub → Actions → wait for green)
	5. Tell me when it's published and I'll pull it into the app (`npm install @honuware/ui@0.1.3 --save-exact` in `ui/`, then `ng test` + the auth audit at phone sizes, where the 6 tap-target warnings should be gone).
	- *Changed from the plan:* I verified the fixes in the library's own suite rather than switching the app's `tsconfig.json` to the library source — that switch is easy to commit by accident and would break CI (no sibling checkout there).

### 1.7 App shell: header, mobile menu, footer
- [x] **Header at tablet widths.** ✅ The hamburger + mobile menu now show **below 1280px** (`xl`), the desktop bar from 1280 (OQ-11) — in all four places that must agree: the hamburger and desktop bar (`header.component.html`), the mobile menu panel (`header-mobile-menu`), and the shell's menu/backdrop pair (`app.component.html`). Hooks `data-test="hamburger"` / `"desktop-menu"`; the hamburger also got an `aria-label` (it was an unlabelled icon button). Playwright `layout/header.spec.ts` at **all seven sizes, as admin** (the longest menu): logo on screen everywhere; below 1280 the hamburger visible and the bar hidden; at 1280 every top-level item inside the screen and on one line. Verified: switching back to `md` fails 6 (the tablet cases).
- [x] Header at 360px. ✅ Same spec at `phone-small`: logo and hamburger both inside the viewport (logo 152px + cart 48px + hamburger 56px = 256px of 360).
- [x] **Mobile cart.** ✅ Cart icon with its count, left of the hamburger, only when the cart has items; 48px wide; `aria-label` says the count ("Shopping cart, 2 items"). Spec: absent with an empty cart, shows the count with two items, sits left of the hamburger, ≥ 44px, and opens `/shop/cart`.
- [x] Mobile menu. ✅ The audit's menu-open state (submenu expanded) is **clean at all six sizes below 1280**. Spec added: the menu covers the page column and **closes on navigation**, landing on the chosen page.
- [x] Footer stacks cleanly. ✅ No findings at any phone size; checked by eye in the checkout and gallery screenshots.
- [x] Safe-area insets on anything sticky. ✅ Nothing is pinned to an edge yet — tokens ready (§1.1); applied with the sticky pay bar in §3.2.
- [x] Geometry specs. ✅ In `layout/header.spec.ts` (Playwright — the switch is a media query). Header/mobile-menu/app Karma specs still pass (9).
- **Desktop note:** at 1280 the top-level labels still wrap to two lines ("Get / Started") — unchanged from the baseline, it's the current design. Not a mobile problem, so not touched.

### 1.8 Phase 1 audit
- [x] ✅ 10/5 — full run, 825 cases, 7.8 min. Errors per tier (cases with errors / cases):

| Tier | 360 | 375 | 390 | 430 | tablet 768 | tablet 1024 | desktop |
|---|---|---|---|---|---|---|---|
| public | 3 / 24 | 3 / 24 | 3 / 24 | 3 / 24 | **3 / 24** (was 23) | 1 / 24 | 1 / 23 |
| auth | 0 / 3 | 0 / 3 | 0 / 3 | 0 / 3 | **0 / 3** (was 3) | 0 / 3 | 0 / 3 |
| account | 1 / 26 (was 2) | 1 / 26 (was 2) | 1 / 26 (was 2) | 1 / 26 | **0 / 26** (was 26) | 0 / 26 | 0 / 26 |
| staff | 1 / 11 | 1 / 11 | 1 / 11 | 1 / 11 | **0 / 11** (was 11) | **0 / 11** (was 11) | 0 / 11 |
| manage | 11 / 54 | 10 / 54 | 10 / 54 | 9 / 54 | **0 / 54** (was 54) | **0 / 54** (was 54) | 0 / 54 |

	- **Tablets are fixed:** 199 erroring tablet cases → 4 (the calendar, Phase 2). The header was the whole story there.
	- **Phones moved less — as expected.** Phase 1 fixed what's *shared*; most phone findings are per-page (overflow 65, overlap 28 → 24, cramped 166 → 153). Phases 2–5 take them page by page. Tap-target warnings 1,798 → **194**, almost entirely from measuring the real Material touch target (§1.2).
	- **The audit caught a bug in Phase 1's own work:** the new `.data-table-scroll` example on the style guide shrank its columns to fit instead of scrolling, crushing the pinned name column to 75px. Fixed (data cells keep to one line; the pinned column gets a 10rem floor) with an iframe spec; verified it fails without the fix.
- [x] Desktop screenshots vs the baseline. ✅ New tool `npm run audit:mobile:compare-desktop` (`compare-desktop.ts` — a minimal PNG decoder + pixel diff, self-tested in `compare-desktop.spec.ts`). Result: **5 changed, all intended** — the style guide (two new sections), the manage and staff dashboards (four even columns at 1280 instead of fixed 300px tiles — checked by eye), `/location` and `/shop/subscribe` (the map and Square iframes are now *masked*, see below). Everything else unchanged or noise.
	- **Two harness fixes this needed**, both so the comparison means something: (1) `settle()` now waits for the Material Icons font explicitly — it sometimes arrived after `fonts.ready`, giving 17 false "changed" pages; (2) desktop shots **mask third-party iframes** (Google map tiles vary per load). Differences under **200 px** are reported as *noise* — sub-pixel anti-aliasing on header icons measured ~130px and looked identical enlarged; a real change is thousands.

**Phase 1 verification:** `ng test` **3503 passed**, `ng lint` clean, production `ng build` passes; Playwright layout specs **35 passed** (21 skipped by design — each spec runs at the sizes it is about); harness self-tests **9 passed**; harness type-checks. Library: 472 passed, lint clean, builds.

**Phase 1 complete** (incl. the `@honuware/ui` 0.1.3 release, 10/5). Next: Phase 2.

---

## Phase 2 — Public pages (no login)

> Persona `anonymous`. The pages a prospective student sees first. Shared public building blocks first, then pages roughly in visitor-traffic order.

> **How Phase 2 was worked (10/5–10/6):** the automated findings first, then **every public page read by eye** at 375px (and 360px where it mattered) — 22 pages, ~45 slices. The eye-review found as much as the audit did: the page gutter, the Our Classes week buttons, the product options, the instructor line.

### 2.1 Public building blocks
- [x] ✅ **`.page-container` gutter 8px → 16px on phones** — found by eye on the home page (headings and body text almost against the screen edge). It's the full-bleed container behind every home section, the footer and the calendar; it now matches the 16px every other container already had. md/lg unchanged. Iframe spec in `design-tokens.spec.ts` (16px at 375, 32px at 800, 64px at 1280).
- [x] The series card (`offering-highlight`, home + `/events`) was fixed in §1.5. Class-info cards, instructor cards and the gallery carousel: no problems at any phone size.

### 2.2 Home (`/`) — includes Polish 13.3 and 13.4
- [x] Upcoming events and series sections (Polish 13.3, "the worst offenders"). ✅ The series card was the offender — crushed text and a button over the dates — fixed in §1.5. The event cards were already fine. Re-checked by eye at 375.
- [x] **"I'll be there" (Polish 13.4).** ✅ — *with a caveat.* It's the attendance control on a class chip (`app-calendar-event`, on the calendar and Our Classes for members). It was a line of muted text with a checkbox glyph, ~20px tall, nothing saying "tap me" — my reading of "looks bad". It now looks and sizes like a toggle: a bordered **44px** control, green when attending. And on a narrow card the Booked/Waitlisted badge, which was pinned in the corner where a long title ran under it, now sits above the title — a **container query** on the card (it's the card's width that matters, and that's testable by sizing the host). Karma specs at a 343px host (44px full and compact, badge above the title; badge still in the corner at 700px). Verified: 3 fail without the fix.
	- ⚠️ **Caveat — the mock never shows this control**, for any persona: it needs a class occurrence the member's attendance template matches, and the mock's data doesn't line up. So it's verified by component specs, not seen in a screenshot. **Please look at it on your phone** (step 5 below), and if "looks bad" meant something else, say what.
	- The *other* "I'll be there" buttons — on `/my/today-classes` and `/my/upcoming-classes` — are account pages: §3.5.
- [x] ✅ Section "View all events →" / "See all memberships →" links: 20px → **44px** tap rows (same look). Hero, announcements, intro, get-started, artwork, membership: fine by eye.

### 2.3 Information pages
- [x] ✅ `/start`, `/about`, `/location`, `/blog`, `/gallery`, not-found — all fine at 360/375 by eye; no findings. (Broken-image icons throughout are the mock's missing photos, not layout. The footer's "Fa In Tw" are deliberate short labels from the site config.)

### 2.4 Classes — Polish 13.5
- [x] `/classes` (Our Classes, "needs the most work"). ✅ Three problems:
	- **Class rows crushed to letter width** — "12:00 PM · 60 min" broke a letter per line at 360. The row did wrap below md, but `.slot-body` is `flex: 1` (a *zero* basis), so photo + text + status always "fit" on one line and the wrap never happened. Now the text has a 10rem basis (shares a line with the photo) and the status chip takes its own line underneath — the same shape as the series card.
	- **Week buttons wrapping mid-label** ("This / week", "Next / week"). Labels never break now; a button that doesn't fit moves to the next line, and the date range is its own line on small phones.
	- **Instructor line** ("with Grace Hopper (Substituting for Ada Lovelace)") squeezed into two narrow columns — now wraps.
	- Playwright `layout/our-classes.spec.ts`. ⚠️ **The first version of the week-button test was vacuous** — it measured button height, but a Material button's height is fixed and a wrapped label just overflows it. It now counts the label's line boxes. Verified against the original stylesheet: 5 cases fail.
- [x] `/classes/all`: ✅ filter chips 33px → **44px**. `/classes/:id`: fine.

### 2.5 Calendar — Polish 13.1 and 13.2
- [x] Day view by default below `md`. ✅ `defaultCalendarViewFor(isBelowMd)` (`calendar.types.ts`); the service reads `matchMedia('(width < 768px)')` at first render. An explicit `?view=` always wins. Specs: the pure function; the service on a phone / wide screen / a browser where matchMedia throws (→ month).
- [x] Week/month scroll inside their own box on a phone. ✅ The box was `overflow-scroll` (both axes, scrollbars always on) → `overflow-x-auto`. **And a worse bug the screenshots found:** the box centred its child with `items-center`, and a centred child wider than a scrolling box overflows to the *left*, where it can never be scrolled to — **at 768px the month grid lost its whole Sunday column.** Now centred with the child's `mx-auto`, which falls back to the left edge when there's no room.
- [x] Opens at the first entry (Polish 13.2). ✅ `firstEntryScrollSegment(days)` — the start of the hour of the earliest class across the week shown (whole hours, so the row above is visible); 10:00 for an empty week. Applied on load and on prev/next week. (The day view is a list, not a time grid, so this is the week view.) The mock's earliest class is 4 AM — under the old fixed 10:00 it was scrolled out of sight. Specs for the function.
- [x] Day navigation. ✅ Prev/next buttons 32px → **44px** below md (day, week and month views; desktop unchanged). No swipe (OQ-5).
- [x] Month view chips: the title couldn't shrink (a flex item without `min-width: 0`) — now ellipsises inside its day cell.
- Playwright `layout/calendar.spec.ts`: lands on day/month by width; explicit `?view=month` respected on a phone; month grid starts at the left edge, the box scrolls, the page doesn't; the week view opens with its first class in sight; 44px buttons. Verified against the original calendar files: **15 cases fail**.
- **Two audit false positives fixed in the checker on the way** (self-tests added for both): text scrolled out of its scroll box (the week grid's hour labels under the header) and the hidden tail of an ellipsised title were being reported as "covered". Line boxes are now clipped to what's actually visible before the overlap test.

### 2.6 People and events
- [x] ✅ `/instructors`, `/instructors/:id`, `/providers`, `/providers/:personId`, `/events` — fine by eye; no findings.

### 2.7 Shop browsing (still logged out)
- [x] ✅ `/shop`, `/shop/services`, `/shop/subscriptions` — fine (the 57-character stress product wraps cleanly). `/shop/:id`: the option buttons were ragged content-width boxes on a phone; below `sm` they're full-width rows.

### 2.8 Phase 2 audit and sign-off
- [x] ✅ **Public tier, all seven sizes: 0 errors, 0 overlaps, 0 cramped text** (167 cases). Baseline was 3 / 24 pages with errors on every phone and 23 / 23 on tablets. Remaining: tap-target *warnings* only, both by decision — month-view event chips (month is a desktop view on a phone; Makeover OQ 19) and Our Classes title links (30px; the big photo beside each goes to the same page).
- [x] Desktop vs baseline: everything changed is intended — the calendar week view (now opens at the first class), All Classes and home (a few px taller from the 44px chips/links), plus the Phase 1 items. Iframe masks explain the shop/location diffs.
- [x] **Verification:** `ng test` **3515 passed**, `ng lint` clean, production build passes; Playwright layout specs 66 passed (53 skipped by design); harness self-tests 11 passed; harness type-checks.
- [x] Tick Polish 13.1–13.5 in `Polish before deploying.md`. ✅
- [ ] **Real-device check of the public pages — yours (OQ-8).** On your phone, **signed out**, in the phone's normal browser (Safari on iPhone, Chrome on Android), go to the site and check:
	1. **Home** — scroll top to bottom. Text sits a thumb-width from both edges (not against them). Open the **Series & workshops** card: each class shows its name and dates readable, with its **Join / Book** button on its own line *below* the dates, full width.
	2. **Menu (☰) → Our Classes** — each class row: photo and name side by side, time on one line ("12:00 PM · 60 min"), the status (Cancelled / Requires a membership…) on its own line below. **Previous / This week / Next week**: no word broken across two lines.
	3. **Menu → Your Calendar** — it opens in **Day View** (not Month). Tap **›** a few times: big enough to hit first time. Then **Day View ▾ → Week**: the grid scrolls sideways inside its box, the page itself doesn't, and it opens with the earliest class visible. **Month**: the **Sun** column is visible at the left.
	4. **Classes → All Classes** — the tag chips (Yoga / Aerial / …) are easy to tap.
	5. **Sign in** (as a member with a weekly plan), then **Our Classes**: find a class showing **"I'll be there"** or **"Tap to plan attendance"** — it should look like a button (bordered, green when attending), not plain text. *This is the one change I could not see in the test data — tell me if it still looks wrong.*
	6. **Turn the phone sideways** on Home and Our Classes: nothing cut off at the right edge.
	- Anything that looks off: a screenshot + which step is all I need.

---

## Phase 3 — Signing in, buying, and the customer account

> Persona `anonymous` for the auth pages, `customer` for the rest. This is where money changes hands, so the checkout path gets the closest look.

### 3.1 Auth pages (library)
- [x] `/login`, `/register`, `/verify` — the components are in `@honuware/ui/auth`; fixes in the library (§1.6 process). Input types, `autocomplete`, `inputmode` on email/password fields. ✅ Already clean at every size from the 0.1.3 release (§1.6): 0 errors, 0 warnings on all three at all seven sizes. No further change.

### 3.2 Booking and checkout flow
In purchase order, each tested with the persona logged in and items in the cart:
- [x] `/shop/cart` — collapses to cards (OQ 18). ✅ The cart already read as one card per line at 375 (name + details left, price + remove right). The one problem was the remove "×": a 15×24 target, now a 44px box whose negative margin keeps the row exactly as tall as before. Spec: 44×44, labelled, layout footprint still 24px.
- [x] `/shop/checkout/:productId`, `/shop/service/:productId`, `/shop/subscribe/:productId`, `/shop/event/:sessionId`, `/shop/series/:classInstanceId`. ✅
	- **Service booking date strip** — seven 60px day buttons + chevrons need ~533px; at 360 the strip ran 150px off-screen (the account tier's only overflow). Below 34rem of its own width (container query) the chevrons and a new week heading ("Oct 14 – Oct 20") sit on top and the seven days become an equal-width 7-column grid, each 44px tall. Desktop (568px of page) keeps the one-row strip. Specs at 328px (all seven inside, ≥44px, heading shown) and at 700px (one row, no heading).
	- **Found on the way — DST bug in the same strip:** days were stepped by a fixed 86 400 000 ms, so the week of Oct 28 2026 (US fall-back Nov 1) showed **"Nov 1" twice** and every later day sat at 23:00 the previous evening. Now steps by calendar day (same fix as the provider-schedule DST bug). Spec pins Oct 28 → Nov 3 with seven distinct dates, each at local midnight.
	- The other four pages had no layout errors; their only change is the sticky bar below.
	- **Confirmation screens** (found by you on a device — the audit never reaches a success state): "Booking Confirmed!" put three buttons in one row at phone width, every label squeezed onto two lines inside a one-line button. The same row was on **seven** confirmations across six pages (service, event ×2, series, checkout, cart, subscribe). New shared class **`.result-actions`** (`_patterns.scss`): centred row on a wide screen, labels never wrap, below `sm` the buttons stack full width, primary first. Specs: `design-tokens.spec.ts` (real 375 and 1024 iframes; mutation-checked), service-booking's first confirmation-state test, and `layout/purchase-flow.spec.ts` now measures the cart's "Payment Complete!" buttons on a real 375 screen.
- [x] Square card form fits at 360px; the pay button is reachable without scrolling past the card form on a 375 × 667 screen. ✅ The form fit since §1.5; the pay button is now in the sticky bar, on screen from the first paint (`layout/sticky-action-bar.spec.ts` measures it before any scroll on cart, checkout, event, series and subscribe at all four phone sizes).
- [x] **Sticky bottom action bar** for the primary action (Pay / Book / Subscribe) on checkout and booking pages below `md`, with safe-area padding — checkout/booking only, not back-office forms (OQ-7, decided). Built once as a shared component (lower layer first), then used by each page; geometry spec that it stays inside the viewport and does not cover the last form field. ✅
	- **`app-sticky-action-bar`** (`shared/components/sticky-action-bar/`) — the page projects its own buttons. Below md: `position: sticky; bottom: 0`, surface fill, top divider, `padding-bottom: space-3 + --safe-area-bottom`. **Sticky, not fixed**: the bar keeps its place at the end of the form, so scrolled to the bottom it sits after the last field instead of over it, and it can never leave its page for the footer. Labels never wrap inside a button; two buttons that can't share a row (Book and Pay + Add to Cart at 360) put the second on its own full-width row. From md up: an ordinary row, no chrome — desktop screenshots identical.
	- Used on all six: cart, checkout, service, subscribe, event, series.
	- **`viewport-fit=cover` added to `index.html` with it**, as §1.1 promised. The shell (`.app-shell`) gives back the top and side insets, so in landscape the page sits beside the notch exactly as before; the footer pads its bottom by the home-indicator inset. All 0 on a screen without a notch.
	- Specs: Karma — buttons projected side by side; in a real 375px iframe the bar is sticky and adds the inset under the buttons (1024: static, no padding); shell pads top/right/left by the insets and not the bottom; footer pads its bottom. Playwright (`layout/sticky-action-bar.spec.ts`, 5 pages): on screen at the top of the page, back in place after the last field at the end, every label one line, nothing wider than the phone; from md up `position: static` with no border; `viewport-fit=cover` present. Mutation (bar made static): Pay lands at 856–1097px on a 667px screen — fails.
	- **Audit rule change:** a sticky/fixed element is now its own layer in the overlap check (like an open menu or dialog) — floating over the text scrolling beneath it is its job. Self-test added (page text under the bar: not reported; a control drawn over the bar's own text: still reported); mutation caught.
- [x] One end-to-end harness pass through the whole purchase in mock mode at `phone-se`. ✅ `layout/purchase-flow.spec.ts`: cart with two items → Card on File → Pay (on screen before and after choosing the card, never scrolled to) → "Payment Complete!" → Purchase Details shows purchase #2, Paid. Pays with a card on file because typing into Square's sandbox iframe is network-bound; the card comes from a **separate** mock flag (`knottyyoga.mockAuditCard`) — seeding it with the general audit data made every checkout open on Card on File and hid the Square form from the audit (caught by the desktop comparison).

### 3.3 Account hub and profile
- [x] `/my/account`, `user-information`, `update-user-info`, `update-user-password`, `notification-preferences`, `cards`, `favorite-instructors`, `skills`. ✅ All already free of errors from Phase 1's shared work. One fix: the three show/hide-password eyes were bare 24×24 icons with no name — now `mat-icon-button` (48px touch target) labelled "Show password" / "Hide password". Spec: 3 toggles, ≥44px, labelled, each toggles.

### 3.4 Account lists — tables become cards
- [x] `/my/purchases` and `/my/purchases/:id`, `/my/events`, `/my/vouchers`, `/my/subscriptions` and `:id`, `gift-permissions` — using the §1.3 card-list pattern. ✅ None of these is a table — each is already a list of cards/panels, so `.data-table--stack` had nothing to stack. What was wrong:
	- **Purchase history:** Material's panel header is a fixed 48px row; at 375 the date wrapped to three lines and was cut off top and bottom. A narrow panel (container query, < 30rem) lets the header grow and puts the date on its own line above total + status. Specs at 343 (date one line, above total, all inside the header) and 760 (one row).
	- **My subscriptions:** title, period and chevron shared one row at every width (a 3-line title, a 4-line period, clipped icons). The row now wraps on its own content (no breakpoint): title on its own line, details + chevron beneath. Back link now the shared `.back-link` (was a 16px link). **New spec file** `my-subscriptions.component.spec.ts` (7 tests — the component had none).
	- **Purchase history, expanded panel** (found by you on a device — the audit only ever saw collapsed panels): each entitlement's bare "active" sat beside a long validity that wrapped around it, and item prices were glued to their names. Status is now a `.badge` (`status-*` tone) with validity and seats on lines beneath; items are "name … price" like the payment rows. Specs at 343px for both (each mutation-checked on its own). **New audit state** `my/purchases [expanded]` so the panel body is measured from now on.
	- **Purchase detail:** the seat-assignment "Set up sharing…" link was a 16px target → 44px.
	- Events, vouchers, gift-permissions, subscription detail: clean, no change.
- [x] `attendance-history` — its five fixed columns (`180px 1.2fr 1.2fr 1fr 110px`) stack. ✅ Instructor and Status were cut off by the card's clipped edge. A table narrower than 40rem (container query) hides the header and lays each row out as a card: date with status beside it, then class, where, instructor one per line. Desktop (768px) keeps the table. Specs at 343 and 800.

### 3.5 Schedules
- [x] `/my/my-schedule` (tabs), `today-classes`, `upcoming-offerings`, and their dialogs (fixed 420/460px today, §1.4). ✅
	- **Today's Classes** and the **Upcoming** tab: "I'll be there" was squeezed beside the details into a two-line, 36px-tall button. Both rows now wrap on their own content: the button keeps its label on one line and drops under the details, right-aligned, when they can't share a line. Specs in both components (phone: one-line label, inside the row; wide: beside the details).
	- The "can't make it" note dialog is a confirm with one optional field, so it stays centred (the §1.4 rule) and already fits the phone width.
	- **New audit states:** `my/today-classes [skip-dialog]` (plans the first class, then opens the dialog) and `my/my-schedule [upcoming]` (the Upcoming tab). Both clean.
	- Upcoming offerings: clean; its only finding is the long heading "Upcoming Workshops & Series" wrapping to three lines (ordinary heading wrap, left as is).

### 3.6 Phase 3 audit and sign-off
- [x] Zero overflow on every account route at every phone size; the full purchase completes at `phone-se`. ✅ **Account tier: 0 errors at all seven sizes (28 cases each, incl. the 2 new states), 0 tap-target warnings.** Public and auth still 0. Full run: 942 passed. Desktop vs the Phase 0 baseline: the new Phase 3 differences are all intended — my-subscriptions (title on one line, badge beneath; shared back link), update-password (icon buttons), purchase detail (taller sharing link); the five checkout pages differ only by the Square-iframe mask the audit has applied since the baseline.
	- Final numbers (after your two device findings): `ng test` 3550 passed (3515 at end of Phase 2); `ng lint` clean; production build passes (pre-existing budget warnings only); layout specs 102 passed + purchase flow; harness self-tests 12 passed. Every fix mutation-checked (13 reverts, each fails its spec).
	- Note for Phase 4: the staff tier also shows 0 now, but **not because anything was fixed** — this run's `/staff/check-in` had no bookings in its 90-minute window, so the row with the old overlap never rendered. Re-check it with bookings present when Phase 4 starts.
- [x] Real-device purchase with a Square sandbox card (§6.4 — yours). Steps, on your phone, against the sandbox deploy: ✅ 2026-10-05
	1. Sign in. **Shop → Services → Deep Tissue Massage**: the seven days fit across the screen under a heading like "Oct 5 – Oct 11", nothing cut off at the right. Tap the right arrow until the week after **Sun Nov 1**: it must start on **Mon Nov 2** (before the fix it started on "Sun Nov 1" again and lost Nov 8).
	2. Pick a day and a time. On the confirm page, before scrolling: the red **Book and Pay** button is pinned at the bottom of the screen. Scroll down — it stays there; at the very bottom it sits just below the card form, not on top of it.
	3. Enter Square's sandbox card **4111 1111 1111 1111**, any future expiry, CVV **111**, ZIP **12345**, and tap **Book and Pay**. It should land on "Booking Confirmed!".
	4. Turn the phone sideways on the same kind of page: nothing slides under the notch, and the pay bar sits above the home bar.
	5. **Account → Purchase History**: the date reads on one line above the price. **Account → Attendance History**: each class is a small card with nothing cut off on the right.
	6. **Account → My Schedule → Upcoming**: "I'll be there" is one line, not squeezed.
	- Anything that looks off: a screenshot + the step number.

---

## Phase 4 — Staff portal

> Persona `staff`. Eleven pages. Not redesigned — made to work.

### 4.1 Check-in first
- [ ] `/staff/class-checkin` — the one staff page the Component Inventory flags as **mobile-first**: an instructor takes attendance standing in the studio, phone in hand. Large tap targets, no horizontal scroll, one-handed.
- [ ] `/staff/check-in` — six fixed widths in its stylesheet today.

### 4.2 The rest
- [ ] Dashboard (`/staff`; `.portal-card` from §1.3), `sessions`, `bookings`, `schedule` (7-column week grid), `time-off`, `shift-requests` and its dialogs, `preferences`, `person-skills` (table, dialogs), `exception-notes`.

### 4.3 Phase 4 audit
- [ ] Zero overflow on every staff route at phone sizes; record numbers.

---

## Phase 5 — Manage, admin and blog admin

> Persona `admin`. 54 routes, used mostly at a desk — the goal is **usable** on a phone, not designed for it: nothing wider than the screen except tables, which scroll on purpose with a sticky first column.

### 5.1 Library CRUD pages
- [ ] `/admin/tables/...` view/edit/new — `@honuware/ui/crud` (fix in the library, §1.6 process). `table-view-control` already scrolls; check the rest.

### 5.2 Tables
- [ ] Wrap every bare table in `.data-table-scroll`: `product-list`, `schedule-list`, `subscription-list`, `subscription-detail`, `entitlement-list`, `event-payments`, `membership-tiers`, `refund-effectiveness`, `specialty-cost-section`, `blog-list`, plus whatever §0.6 finds.
- [ ] Replace the six ad-hoc `overflow-x: auto` copies with the shared class.

### 5.3 Forms and editors
- [ ] Fixed widths in `event-create`, `voucher-management`, `class-schedule-manage`, `bundle-management`, `pricing-overview`, `subscription-revenue`; fixed-column grids in `event-create`, `page-content`, `image-carousels`.
- [ ] The admin table-picker row (`admin.component.html:4`) wraps.

### 5.4 Week grids and reports
- [ ] `room-schedule-editor`, `schedule-grid`, `instructor-load`, `open-seat-heatmap`, `enrollment-trend`: 7-column and wide grids scroll inside their own container instead of widening the page.

### 5.5 Everything else in `/manage`, `/admin`, `/blog-admin`
- [ ] Walk the remaining routes from the manifest; fix what the audit flags.

### 5.6 Phase 5 audit
- [ ] Zero page overflow on every manage/admin route at phone sizes (scrolling tables excepted by design).

---

## Phase 6 — Keep it fixed

### 6.1 Mobile audit in CI
- [ ] A GitLab job that serves the mock build and runs the overflow check at phone sizes, failing the pipeline on any new horizontal overflow — all tiers, phone sizes (OQ-9, decided; `test:mobile-audit`, a few minutes per pipeline). Desktop screenshot comparison stays a local tool — too brittle across font rendering to gate CI on.

### 6.2 Conventions
- [ ] `ui/CLAUDE.md`: a "Mobile" section — the breakpoint mixin, the shared classes to use, the dialog rule, "every layout change gets a 375px geometry spec", how to run the audit.

### 6.3 Final full audit
- [ ] All seven sizes, all tiers. Before/after table here.

### 6.4 Real devices (yours)
- [ ] One pass on a real iPhone and a real Android phone: the public pages, one full purchase, `/staff/class-checkin`. Emulation does not reproduce Safari's toolbar resizing, the on-screen keyboard covering inputs, or real touch. I'll write the exact steps (menu → page → what to tap) when Phase 2 is done.

---

# Open questions

> ✅ **OQ-1 to OQ-11 resolved 10/5/2026 — every default accepted** (OQ-11: hamburger below 1280, folded into §1.7). Decisions are folded into the plan sections above (each cited as "OQ-n, decided"). New questions that come up during implementation are added below from OQ-11, each with a default so work never stops on one.

11. **OQ-11 (new, from the baseline) — where should the header switch to the hamburger?** Today it switches at 768px (`md`), but the desktop menu doesn't fit until about 1100px for a customer and wider for admin/staff, who have more items: at iPad portrait the logo disappears and *Your Calendar* falls off the edge, on **every page**. Options: (a) hamburger below 1280 — iPads in both orientations get the phone menu, which works; (b) hamburger below 1024 and shrink the desktop menu (smaller labels/gaps) so it fits from 1024 — tighter, and the admin menu may still not fit; (c) keep 768 and make the bar scroll or wrap. *Default: (a), hamburger below 1280. It's the only option that fits every persona's menu at every tablet size without redesigning the menu, and the mobile menu is already the one built for touch.*
	- Mason- I'll go with your recommendation.

1. **OQ-1 Add Playwright as a dev dependency?** It's the screenshot/audit engine (§0.3). Dev-only — nothing changes in the shipped site — but it's a new tool in `package.json` and downloads a Chromium (~150 MB) on first install. *Default: yes.* (The alternative, driving your own Chrome through the browser extension, can't emulate phone sizes reliably and needs you present.)
	- Mason- Sure. This sounds fine.
2. **OQ-2 Breakpoints.** The Makeover planned Tailwind screens `sm:375 / md:768 / lg:1280`; today Tailwind's defaults are live (640/768/1024/1280) and the 24 existing `sm:`/`lg:` classes assume them. *Default: keep the defaults — `md` (768) is the line both plans agree on, and changing `sm`/`lg` would shift existing layouts for no mobile gain.*
	- Mason- I'll go with your recommendation.
3. **OQ-3 Cart on mobile.** The cart badge is desktop-only. *Default: a cart icon with its count in the header bar, left of the hamburger, shown only when the cart has items.*
	- Mason- I'll go with your recommendation.
4. **OQ-4 Dialogs on phones.** *Default: form dialogs (editors, booking, transfer requests) go full-screen below `md`; short confirm/info dialogs stay centered at full width with a margin. True bottom sheets (Makeover) are a larger change — deferred unless you want them.*
	- Mason- I'll go with your recommendation.
5. **OQ-5 Calendar swipe** between days. *Default: not now — large prev/next buttons only. Swipe is a gesture-library addition with its own edge cases; a follow-up item if you want it.*
	- Mason- I'll go with your recommendation.
6. **OQ-6 Library releases.** Fixes in `@honuware/ui` need a publish, and publishing needs git commit/tag/push, which you do. *Default: I develop against the library source, batch all library changes for a phase into one release, and give you the exact release steps at the end of that phase (Phase 1 and possibly 3 and 5).*
	- Mason- I'll go with your recommendation.
7. **OQ-7 Sticky bottom bars** for primary actions (Pay, Book, Save) on phones — in the Makeover inventory, not built. *Default: yes for checkout/booking only (Phase 3), where the button otherwise sits below a tall card form; not for back-office forms.*
	- Mason- I'll go with your recommendation.
8. **OQ-8 Real-device checks.** *Default: two short passes by you — end of Phase 2 (public) and end of Phase 3 (a purchase). §6.4's final pass covers staff check-in.*
	- Mason- I'll go with your recommendation.
9. **OQ-9 CI gate.** *Default: yes, a `test:mobile-audit` job failing on new horizontal overflow at phone sizes, all tiers. It adds a few minutes per pipeline.*
	- Mason- I'll go with your recommendation.
10. **OQ-10 Ryan's 375 frames.** The Makeover's policy is "mobile keeps today's responsive behavior until his 375 frames exist", and some pending Track C screen ports (Our Classes, Cart/Checkout, Calendar, Class Detail, My Events) would rewrite pages this plan touches. *Default: proceed — make today's layouts work on phones now; when his frames arrive they replace these layouts, and the harness and specs from this plan become the safety net for that port. Tell me if any page should wait for him instead.*
	- Mason- I'll go with your recommendation.