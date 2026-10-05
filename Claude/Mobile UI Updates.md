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
   - A **geometry spec** in the component's `*.component.spec.ts` pins the fix at 375px: it sets the host width and asserts `getBoundingClientRect()` (nothing wider than the viewport, columns actually stacked, the button actually full width). Presence-only specs are not accepted — Polish Phase 13 records two layouts that passed presence specs while visibly broken. `site-theme.component.spec.ts:237` is the template.
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
- [ ] Full audit at all seven sizes. Record per tier: pages, pages with overflow, worst offenders. Keep the desktop screenshots as the regression baseline for the rest of the plan.
- [ ] Re-scope Phases 2–5 from what the baseline actually shows: add items for problems the code survey missed, drop ones that turn out fine.

---

## Phase 1 — Shared foundation

> Every page inherits these, so fixing them first removes a large share of the per-page problems in one move. Ordered bottom-up: tokens and mixins → global classes → shared components → library → shell.

### 1.1 Breakpoint mixin and tokens
Today component SCSS uses six different breakpoints (600, 639, 700, 767, 768, 900px) and no shared definition.
- [ ] `src/assets/styles/mixins/_breakpoints.scss` with `below(md)` / `at-least(md)` etc., values matching Tailwind's screens so SCSS and Tailwind classes switch at the same width.
- [ ] Tokens still open from the Makeover: `--touch-target-min` (44px) and safe-area padding tokens (`env(safe-area-inset-*)`).
- [ ] Move the 17 existing `@media` rules onto the mixin. Where a rule used an odd width, check the page at both widths before moving it.
- [ ] Extend `design-tokens.spec.ts` for the new tokens; a spec that the mixin's widths equal Tailwind's.
- [ ] `ui/CLAUDE.md`: a short "Breakpoints" section so new code uses the mixin.

### 1.2 Global base rules
- [ ] Inputs, selects and textareas at least 16px on phones.
- [ ] Minimum tap-target size for Material buttons and icon buttons on touch devices.
- [ ] Long unbroken strings (emails, URLs) wrap instead of pushing the page wide.
- [ ] Specs at 375px for each rule.

### 1.3 Shared pattern classes (`_patterns.scss`, `_tables.scss`, `_layout.scss`)
- [ ] `.portal-card` has a fixed `width: 300px` (`_patterns.scss:282`) — every staff and manage dashboard tile. Make it fluid: one column on phones, the grid on wider screens.
- [ ] `.data-table-scroll`: exists, but **no page uses it** except the style guide. Add the sticky first column (Makeover OQ 18) and a visible hint that the table scrolls.
- [ ] Check `.page-header`, `.filter-row`, `.filter-bar`, `.form-actions`, `.form-grid`, `.field-row` and the four page containers at 360px; fix what fails.
- [ ] A reusable **card-list** pattern for the customer tables that collapse to cards (§3.4).
- [ ] Style-guide additions for each, and geometry specs in `style-guide.component.spec.ts` at 375px.

### 1.4 Dialogs
No dialog sets `maxWidth`, and several force a width or `min-width` wider than a phone: 780px (`class-schedule-manage.component.ts:540`), 460px and 420px (`today-classes`, `upcoming-classes`, `my-schedule`, `calendar-navigation.service`), `min-width: 360px` in three dialog stylesheets.
- [ ] App-wide dialog defaults: never wider than the viewport minus a margin.
- [ ] Below `md`: form dialogs (editors, booking, transfer requests) go full-screen; short confirm/info dialogs stay centered at full width with a margin. No bottom sheets (OQ-4, decided).
- [ ] Remove the fixed widths and min-widths listed above.
- [ ] Specs: each fixed dialog opens no wider than a 375px viewport.

### 1.5 Shared components (`src/app/shared/components`)
Lower than any page, used by many:
- [ ] `public-event-card`, `offering-highlight`, `event-session-card`, `calendar-event`, `seat-assignment`, `book-for-selector`, `event-session-price-override-editor` — audit inside a host page at each phone size, fix, geometry spec each.
- [ ] The Square card form containers (`payment-method`, `card-management`, `card-picker`): height is tokenized but width is unconstrained; the iframe must fit 360px.

### 1.6 `@honuware/ui` library components
The library (`C:\Users\mason\source\repos\honuware-web-components`) has **no** `@media` rules today. Its pages and components are used across every tier.
- [ ] Audit and fix in the library repo: `hw-photo-upload` (26 uses), `hw-confirm-dialog`, the form controls and `hw-composite-row-control`, and the auth card (`hw-login`/`hw-register`/`hw-verify`). The CRUD table pages wait for Phase 5.
- [ ] Library specs alongside each fix, per the library's own conventions.
- [ ] Work against the library source via the `tsconfig.json` path block during development, then **one batched release per phase** (OQ-6, decided). Releasing needs your git commit/tag/push, so at the end of each phase with library changes I write the exact release steps here and pause on that one item.

### 1.7 App shell: header, mobile menu, footer
- [ ] Header at 360px: logo, hamburger and anything else in the 55px bar fit without overlap.
- [ ] **Mobile cart affordance** — the cart badge renders only on desktop today. A cart icon with its count in the header bar, left of the hamburger, shown only when the cart has items (OQ-3, decided).
- [ ] Mobile menu: every menu item reachable, expanded submenus scroll, tap targets ≥ 44px, closes on navigation.
- [ ] Footer stacks cleanly.
- [ ] Safe-area insets on anything sticky.
- [ ] Geometry specs in the header, mobile-menu and footer specs.

### 1.8 Phase 1 audit
- [ ] Full run; record the drop in overflow counts per tier. Desktop screenshots compared against the baseline.

---

## Phase 2 — Public pages (no login)

> Persona `anonymous`. The pages a prospective student sees first. Shared public building blocks first, then pages roughly in visitor-traffic order.

### 2.1 Public building blocks
- [ ] Home-page sections (`pages/public/home-page/sections/*`), class-info cards, instructor cards, the image carousel — whatever §0.6 shows breaking across several public pages.

### 2.2 Home (`/`) — includes Polish 13.3 and 13.4
- [ ] Upcoming events and series sections: grids collapse to one column, no fixed widths (Polish 13.3, "the worst offenders").
- [ ] The "I'll be there" button (Polish 13.4).
- [ ] Hero, announcements, every other section.

### 2.3 Information pages
- [ ] `/start` (Getting Started), `/about`, `/location`, `/blog`, `/gallery`, not-found.

### 2.4 Classes — Polish 13.5
- [ ] `/classes` (Our Classes) — "needs the most work". Scoped properly after the baseline screenshots; expect several items.
- [ ] `/classes/all`, `/classes/:id`.

### 2.5 Calendar — Polish 13.1 and 13.2
- [ ] Day view by default below `md`, chosen at first render from the viewport; an explicit choice is left alone after that.
- [ ] Week/month views stay reachable but are desktop views; on a phone they scroll inside their own container (`min-w-[50rem]` today) rather than widening the page.
- [ ] Opens scrolled to the first entry of the day, not midnight (Polish 13.2).
- [ ] Day navigation with large prev/next buttons. No swipe gesture for now (OQ-5, decided — possible follow-up).

### 2.6 People and events
- [ ] `/instructors`, `/instructors/:id`, `/providers`, `/providers/:personId`, `/events`.

### 2.7 Shop browsing (still logged out)
- [ ] `/shop`, `/shop/services`, `/shop/subscriptions`, `/shop/:id`.

### 2.8 Phase 2 audit and sign-off
- [ ] Zero horizontal overflow on every public route at every phone size. Record the numbers.
- [ ] Tick Polish 13.1–13.5 in `Polish before deploying.md`.
- [ ] Real-device check of the public pages — yours (OQ-8, decided: real-device passes at the end of Phase 2, Phase 3, and §6.4). I write the exact steps here first.

---

## Phase 3 — Signing in, buying, and the customer account

> Persona `anonymous` for the auth pages, `customer` for the rest. This is where money changes hands, so the checkout path gets the closest look.

### 3.1 Auth pages (library)
- [ ] `/login`, `/register`, `/verify` — the components are in `@honuware/ui/auth`; fixes in the library (§1.6 process). Input types, `autocomplete`, `inputmode` on email/password fields.

### 3.2 Booking and checkout flow
In purchase order, each tested with the persona logged in and items in the cart:
- [ ] `/shop/cart` — collapses to cards (OQ 18).
- [ ] `/shop/checkout/:productId`, `/shop/service/:productId`, `/shop/subscribe/:productId`, `/shop/event/:sessionId`, `/shop/series/:classInstanceId`.
- [ ] Square card form fits at 360px; the pay button is reachable without scrolling past the card form on a 375 × 667 screen.
- [ ] **Sticky bottom action bar** for the primary action (Pay / Book / Subscribe) on checkout and booking pages below `md`, with safe-area padding — checkout/booking only, not back-office forms (OQ-7, decided). Built once as a shared component (lower layer first), then used by each page; geometry spec that it stays inside the viewport and does not cover the last form field.
- [ ] One end-to-end harness pass through the whole purchase in mock mode at `phone-se`.

### 3.3 Account hub and profile
- [ ] `/my/account`, `user-information`, `update-user-info`, `update-user-password`, `notification-preferences`, `cards`, `favorite-instructors`, `skills`.

### 3.4 Account lists — tables become cards
- [ ] `/my/purchases` and `/my/purchases/:id`, `/my/events`, `/my/vouchers`, `/my/subscriptions` and `:id`, `gift-permissions` — using the §1.3 card-list pattern.
- [ ] `attendance-history` — its five fixed columns (`180px 1.2fr 1.2fr 1fr 110px`) stack.

### 3.5 Schedules
- [ ] `/my/my-schedule` (tabs), `today-classes`, `upcoming-offerings`, and their dialogs (fixed 420/460px today, §1.4).

### 3.6 Phase 3 audit and sign-off
- [ ] Zero overflow on every account route at every phone size; the full purchase completes at `phone-se`.
- [ ] Real-device purchase with a Square sandbox card (§6.4 — yours).

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

> ✅ **All resolved 10/5/2026 — every default accepted.** Decisions are folded into the plan sections above (each cited as "OQ-n, decided"). New questions that come up during implementation get added below as OQ-11 onward, each with a default so work never stops on one.

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