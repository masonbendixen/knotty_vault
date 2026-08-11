---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/19/2026
Version: 0.2
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I have been working with you to create my website in angular and it is close to being functionally complete. However, the global SCSS is pretty minimal and nearly all of the the styling is bespoke per component. I've done some leg work to ad hoc make things somewhat consistent but have known for a while that we needed to do some pretty heavy duty lifting to make it a professional looking website.

A friend has taken screenshots of each of the pages and created mock ups in Figma to get a much better looking and cohesive design. I'm really happy with his work and would like to migrate to having the website be styled accordingly. I think this is a chance to accomplish a lot of objectives though.

Eventually, I would like to have this be a product that other people can use to run their businesses so I would like the UI to be pluggable and database driven so that various users can alter the look and feel for their needs.

I feel like there is probably a order of operations here. I think that it probably looks like this:

- Migrate the existing pages to move most of the SCSS out of each individual component into either the global style sheet or some set of modular style sheets.
- This probably means assigning various classes that can be referenced from the HTML. We can probably switch to these standardized CSS classes even locally to each component SCSS file and migrating the component to use standardized classes before moving to centralized SCSS files.
- Then we migrate from the component SCSS files to use the global files.
- Then we migrate the global SCSS file to use his setting from the Figma project. I'm not sure if we can export this from Figma manually or if you can connect through an MCP server to extract the values for me. Please advise and give recommendations.
- After we have all of this nicely done, I would like to move the actual values out of the SCSS directly into values in the database so that this can be altered per instance of the site based on user preferences.

Honestly, this is kind of just me spit balling, I'm very open to suggestions here. This is the start of a brainstorming process so let's start the brainstorming.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here

# Plan

## Designer Instructions

> **If you're the designer, read this section first.** It's a punch list of what to do. The rest of this document is engineering planning you don't need to dig into. You have two companion documents in this Obsidian vault that you'll work from:
>
> 1. **[[Component Inventory for Designer]]** — the list of every reusable Figma component to build (Button, Card, Badge, Bottom Sheet, etc.) and every screen to design (Public / Home, Shop / Checkout, Account / Profile, etc.). Each item has a suggested Figma name and a plain-English description of where it lives in the app.
> 2. **Phase 1.1 of this document** (just below the next two sections) — the structured guide to setting up your Figma file: design tokens as Variables, component library structure, per-screen discipline, brand assets, and file access. You've agreed to do all five of those (A–E).

### Decisions Mason has already made

- ✅ **Use Figma Variables for every colour, font size, spacing, and radius — not Styles.** If your current file is on Styles, please convert it. (Closes Open Question 2.)
- ✅ **Two-layer token scheme** — primitive tokens (e.g. `color/red/500`) plus semantic tokens (e.g. `color/brand`) that reference them. All component frames should reference the semantic layer, never the primitive layer directly. (Closes Open Question 6.)
- ✅ **Material Icons stay** — please design with Google Material Icons (the same set used today via `<mat-icon>`). Don't introduce a new icon family. (Closes Open Question 7.)
- ✅ **Dark mode is in scope.** Two ways to implement it on your side:
    - **Pro path (cleaner):** add a `Dark` mode to your Colors Variable collection so a single semantic token like `color/brand` switches values between modes. Requires Figma Professional (see Open Question 3).
    - **Free path (workaround, also fine):** keep one mode per collection and define *paired* semantic tokens — `color/brand-light` and `color/brand-dark`, both referencing the shared primitive layer. The engineer ingests the JSON, pairs them up, and emits matching CSS variables under `:root` and `:root[data-theme="dark"]`. Trade-off: you can't toggle light/dark inside Figma to preview, and you have to remember to add a `-dark` for every `-light`. Otherwise the user-facing result is identical. (Closes Open Question 9. See Open Question 3 for whether to upgrade.)
- ✅ **Breakpoints** — design at 375 (mobile, the primary canvas) and 1280 (desktop) **only. No tablet frames** — Mason confirmed desktop + mobile are enough. (Closes Open Questions 10 and 20.)
- ✅ **Mobile is the primary canvas.** Start every screen at the 375-wide mobile frame and expand outward — never the other way around. Most of this app's traffic is mobile.
- ✅ **Mobile navigation stays as a hamburger menu** — no bottom tab bar. (Closes Open Question 15.)
- ✅ **Calendar on mobile defaults to day view** with swipe-between-days; week/month stay desktop views. (Closes Open Question 19.)
- ✅ **No motion / animation work in this pass** — transitions and micro-interactions wait until the functional redesign ships. (Closes Open Question 24.)
- ✅ **The back-office portals are NOT being redesigned** — no Figma frames for anything in the inventory's Manage / Staff / Admin sections; those screens pick up the new look automatically from the shared building blocks. (Closes Open Question 21. One optional exception flagged in the inventory: Staff / Check-In, the lone back-office screen used on a phone in-studio.)

### What to do, in order

1. **Read [[Component Inventory for Designer]]** end-to-end. It tells you the names to use and the patterns to follow.
2. **Read Phase 1.1 of this document** (sections A–E below) for the technical structure of the Figma file.
3. **Set up your Figma file:**
    - Sort out the Figma plan question (see Open Question 3 — updated 7/19/2026 with MCP rate-limit research). Pro multi-mode Variables are the convenient dark-mode path, and a paid seat is also what makes the MCP hand-off to engineering usable (the free tier is capped at 6 MCP calls/month). The free-path workaround — paired `-light`/`-dark` tokens + the Variables-to-JSON export plugin — remains fully viable if you stay on Starter.
    - Create a `Foundations` page + a `Screens` page (or two separate files).
    - Build the token Variables: Colors (with Light + Dark modes), Typography, Spacing, Radius.
    - Build the components listed in Part 1 of the Inventory, naming them **verbatim** from the inventory (this is the single highest-leverage thing for fast engineer-side mapping).
4. **Design the screens** listed in Part 2 of the Inventory, mobile-first — **skipping the Manage / Staff / Admin sections entirely** (decided 7/19/2026: the back office isn't being redesigned; it inherits the building blocks). Note that Part 2 grew on 7/19/2026: a wave of class features shipped after your first pass, and the inventory's new **"What changed since your first pass"** section is your delta punch list — new public pages (Our Schedule, Instructor Detail), a rebuilt Calendar, a new series-booking flow, and seven new account pages.
5. **When ready:** enable Dev Mode on the file, generate a read-only personal-access token, and send the file URL + token to Mason so the engineering side can pull the tokens straight into code.

### Questions for you (please answer here while you're with Mason)

> ✅ **All five answered with Mason 7/19/2026.** Each answer has been propagated to its numbered Open Question further down this document (with Claude's follow-ups), and the affected plan items + the [[Component Inventory for Designer]] have been updated. Kept here for the record.

For each, drop a `- Designer- ...` bullet under the corresponding Open Question further down this document.

- **Open Question 19 (Calendar on mobile).** Default to day view with swipe-between-days, or do you have a different vision?
	- Mason- Yeah, day view probably makes the most sense for mobile.
- **Open Question 20 (Tablet support).** Willing to design tablet (768px) frames for the calendar and admin tables specifically, or skip tablet entirely?
	- Mason- I think desktop and mobile are enough. I don't think we need to design explicitly for tablet.
- **Open Question 21 (Pages without mockups).** Of the existing pages listed in [[Component Inventory for Designer]] Part 2, which specific ones are you *not* redesigning?
	- Mason- We aren't redoing any of the admin or staff dashboards.
- **Open Question 24 (Animation / motion).** Are you planning any motion — transitions, micro-interactions, animated empty states, loading skeletons? If yes, please annotate in Dev Mode.
	- Mason- No plans currently. Mainly want to get a nice looking, functional website. Bells and whistles can wait.
- **Figma plan decision** (Open Question 3). Pro (~$15/month) unlocks multi-mode Variables — the convenient one-toggle path for dark mode and the ability to preview light/dark in Figma. **You don't need Pro for dark mode to ship** — there's a free-tier workaround using paired `*-light`/`*-dark` semantic tokens (described in Phase 1.1.A below) that produces identical runtime behaviour. Pro is therefore a quality-of-life choice. Do you want to upgrade for the smoother workflow, or stay free and maintain paired tokens manually?
	- Mason- Don't we need pro to connect in dev mode through the MCP server?
	- Claude- Short answer: **for practical purposes, yes — but it's a rate-limit gate, not a connection gate.** Figma's MCP server now technically works on every plan, but free Starter seats are capped at **6 tool calls per month** (a single screen inspection burns that), while a **Dev or Full seat on Professional** gets **200 calls/day**. Variables ARE readable through the MCP server with a paid seat, so Pro-tier access covers both the token pull and per-frame inspection. Full breakdown, costs, and recommendation under Open Question 3 below (researched 7/19/2026).

---

## Current State (snapshot, 2026-05-19)

A quick read of the repo before the plan so the phases below have grounding:

- **Global SCSS is genuinely minimal.** `ui/src/assets/styles/` is six small partials: `_variables.scss` (5 colors), `_fonts.scss` (4 DIN font-faces + a single `h1 { @extend .din-bold }`), `_html-overrides.scss` (letter-spacing, list reset, body font-family), `_theme.scss` (one class — `.page-container`), `_angular-material-theme.scss` (M2 indigo + pink palette — **not** the brand red/orange), and `angular-material-overrides/` (mdc-button radius, mdc-notch fix, spinner colour). That's the entire shared design layer.
- **Bespoke component SCSS dominates.** 97 component `.scss` files. A `rg` count shows **857 hard-coded hex literals across 76 files**, and the same `border: 1px solid #d1d5db` for `mat-card` is duplicated in **65 files** (CLAUDE.md already flags this as a known smell — "duplicated in 56+ files, should be global"). Status badges (`.role-instructor`, `.waitlisted`, `.bool-true/false`, `.applied-badge`, `.suggestion-badge`), container wrappers (`.xxx-container { max-width: …; margin: 0 auto; padding: …; }`), and table-row hover patterns are re-implemented per page.
- **Tailwind is configured** (`tailwind.config.js`) with `theme-red/orange/gray/black/white` mapped to the CSS variables in `:root`, plus base/components/utilities layers in `styles.scss`. So we already have CSS-variable-based theming hooked into the build — it's just only being used for five colors.
- **Material M2 palette mismatch.** `mat.m2-define-light-theme(...)` uses `m2-indigo-palette` / `m2-pink-palette` even though the visible brand colour is red/orange. Buttons with `color="primary"` therefore render indigo. This is the single highest-leverage cosmetic fix in the entire codebase.
- **No public bootstrap / theme-config endpoint exists.** `endpoints/` has nothing matching `config|theme|brand|tenant`. `getUserInfo()` on the Angular `ServerAccess` is the closest thing to a server-pushed boot payload today, and it's auth-gated.
- **The `Secrets` system in C++ (`util/secrets/`) is the ready-made home for runtime-configurable strings.** It's already DB-backed, has a typed key registry (`secret_keys.h`), default values (`secret_values.cpp`), and a `SecretsHelper` lookup. Per-tenant theming should sit on top of (or alongside) it rather than reinventing it.
- **The router has two truly public routes** (`/`, `/about`, `/classes`, `/staff`, `/events`, `/instructors`, `/providers/:id`) plus `/shop` which is partly public — those are the most visitor-facing pages and should be the first targets for visible polish once the foundation is in place.

So the order-of-operations the user sketched is fundamentally right, but it can be sequenced more aggressively: a couple of foundational changes (token layer + Material palette + a shared `surface-card` class) will visibly fix dozens of pages before any component-level rewriting starts.

---

## Mobile strategy & principles (cross-cutting, applies to every phase)

This isn't a phase — it's a set of rules that informs everything below. The reasoning: for this app specifically, mobile is the dominant case. Casual visitors browsing classes, customers booking and paying, and instructors doing in-studio attendee check-in are all mobile-primary use cases. Only the studio owner's bulk-edit `/manage` and `/admin` work is genuinely desktop-primary. So mobile is the default canvas, not the squeezed-down version of desktop.

### Principles

- [ ] **Mobile-first CSS, always.** Default styles target mobile; Tailwind `md:`/`lg:` prefixes layer on desktop expansion. Pages like `about.component.html` that currently use `w-1/2 pe-5` with no responsive prefix get fixed during Phase 3.
- [ ] **44px touch-target floor.** Every interactive element ≥ 44×44pt (Apple HIG) / 48×48dp (Material). Bake this in as a `--touch-target-min` token in Phase 2.1 and add an SCSS mixin / lint check so future buttons can't regress.
- [ ] **16px input font-size floor on mobile.** Anything smaller triggers iOS auto-zoom on focus, which is jarring and disorienting. Type ramp in Phase 2.1 needs a mobile floor.
- [ ] **Safe-area insets on every sticky element.** Header, footer, sticky bottom CTAs all need `padding-top: max(var(--space-3), env(safe-area-inset-top))` (or equivalent). iPhone notches and home indicators otherwise clip content.
- [ ] **No hover-only affordances.** Hover doesn't exist on touch. The `.graySquare > a { opacity: 0; … }` reveal pattern in `styles.scss` is the canonical example to fix.
- [ ] **Native form behaviours.** Every input gets `inputmode` and `autocomplete` attributes — `inputmode="email"`, `inputmode="numeric"`, `autocomplete="cc-number"`, etc. Native date pickers (`<input type="date">`) on mobile instead of bespoke calendar UIs where feasible.
- [ ] **No centred modals on mobile.** `mat-dialog` and especially `confirm-dialog` should be full-screen takeovers or bottom-sheets below the `md` breakpoint. Add this as a Phase 2 primitive.
- [ ] **Sticky bottom action bars for primary CTAs.** Checkout's Pay button, booking's Confirm button, cart's Checkout button. Thumb-reachable, always visible.

### Mobile-specific work items (folded into existing phases)

These are concrete deliverables; each is also listed under its host phase below for traceability.

- [ ] **Phase 2 — mobile-aware tokens:** `--touch-target-min`, mobile-floor type ramp, safe-area inset utility, container-padding tokens that step down on mobile.
- [ ] **Phase 2 — `bottom-sheet` and `full-screen-modal` primitives** to replace centred dialogs on mobile.
- [ ] **Phase 2 — `sticky-action-bar` primitive** for bottom CTAs.
- [ ] **Phase 2 — `responsive-data-table` strategy.** Three rendering modes, picked per-page (confirmed by Mason in Open Question 18): (a) horizontal-scroll with sticky first column for all `/manage/*` and `/admin/*` tables; (b) collapse-to-cards below `md` for `my-events`, `purchase-history`, `my-vouchers`, `cart`, and dashboard alert lists; (c) collapse-columns-with-tap-to-expand is rarely the right answer.
- [ ] **Phase 3 — calendar mobile defaults (confirmed, Open Question 19):** default to day-view below `md`, swipe-between-days, "jump to month" opens a month-picker bottom sheet. Week/month views stay desktop-only.
- [ ] **Phase 3 — checkout mobile polish:** sticky Pay button at the bottom of the viewport, one column layout, large input fields, native form behaviours.
- [ ] **Phase 3 — Staff Check-In mobile-first redesign.** This screen is *primarily* used on a phone in-studio; design it for one-handed thumb use, bottom action area, swipe-friendly attendee list.
- [ ] **Phase 3 — Apple Pay / Google Pay** via the Square Web SDK. The SDK already supports both; today only the card form is wired in `SquarePaymentService`. Native payment buttons on the checkout page can dramatically lift mobile checkout conversion. **Backend impact: minimal** — Square treats the resulting payment tokens identically; the existing `/api/purchase_pay_card` endpoint already accepts whatever token the SDK produces. Mostly an Angular wiring change plus a feature-flag secret to toggle availability.
- [ ] **Phase 3 — responsive images.** `<img srcset>` + `sizes` for hero carousel, instructor portraits, product photos. Avoid serving the 2000×1200 desktop hero to a 375-wide phone.
- [ ] **Phase 6 — PWA-ification.** `manifest.json` (install-to-home-screen, theme colour, icons at multiple resolutions), service worker (offline catalogue browse, asset cache), and eventually push notifications for class reminders / waitlist promotions / schedule changes. This is a genuinely high-value addition for a fitness studio app — recurring engagement is the use case PWAs were designed for.

### Mobile testing discipline

- [ ] **Real device testing**, not just Chrome DevTools' device emulator. Emulators don't reproduce iOS Safari's address-bar `100vh` shenanigans, the way `position: sticky` interacts with overscroll, or the keyboard's effect on viewport height. Minimum bar: a real iPhone and a real Android phone before merging any visible mobile change.
- [ ] **Touch target audit.** Phase 2.9's style-guide page should include a section that highlights every interactive element with a 44×44 overlay; anything that's smaller is a defect.
- [ ] **Lighthouse mobile score in CI.** Track score over time so we know if a change makes mobile perf worse.

---

## Phase 1 — Audit & alignment (zero code changes)

> Lower layers first within each phase: capture inputs before touching code.

### 1.1 Designer prep in Figma — what your friend can do to make migration painless

The single biggest predictor of how smooth this whole project is, is **how disciplined the Figma file is about Variables and component structure**. If everything in the file is a hard-coded hex/px value applied straight to a frame, we're effectively reading screenshots and guessing. If everything references named Variables and named Components, we can extract the tokens mechanically and the code-side mapping becomes obvious. Below is a "menu" your friend can work through; everything in **A** and **E** is high-leverage and worth pushing on, **B** is highly recommended, **C** and **D** are polish that pays off later.

#### A. Foundation: define design tokens as Figma **Variables** (not Styles)

Figma Variables (the design-tokens feature introduced in 2023) are the cleanly-exportable, MCP-readable, mode-aware version of design tokens. The older Color Styles / Text Styles still work but don't carry the same semantic / mode metadata.

- [ ] **Create a `Colors` Variable Collection** organised in two layers:
    - **Primitive layer** — the raw palette: `color/red/50…900`, `color/orange/50…900`, `color/gray/50…900`, `color/green/…`, `color/yellow/…`. These are the only values that hold raw hexes.
    - **Semantic layer** — references to the primitive layer: `color/brand`, `color/brand-on` (text colour to use on brand backgrounds), `color/accent`, `color/surface`, `color/surface-muted`, `color/border`, `color/border-strong`, `color/text`, `color/text-muted`, `color/text-inverse`, `color/success`, `color/success-bg`, `color/warn`, `color/warn-bg`, `color/danger`, `color/danger-bg`, `color/info`, `color/info-bg`. **All component frames reference the semantic layer**, never the primitive layer directly. This is the pattern that makes runtime theming (Phase 5) and dark mode (Phase 6.2) trivial.
    - **Dark mode IS in scope (Open Question 9 confirmed).** Two ways to set it up in Figma:
        - **Pro path:** add a `Dark` mode to this Colors collection. Same semantic names, different primitive references per mode. One token (`color/brand`) flips values via the mode toggle in Figma's right sidebar.
        - **Free path:** keep one mode per collection. Define *paired* semantic tokens: `color/brand-light` AND `color/brand-dark`, both referencing the shared primitive layer. JSON export round-trips cleanly; the engineer assembles them into `:root` (light defaults) and `:root[data-theme="dark"]` (overrides) at code time. Trade-off: no in-Figma preview toggle, slightly higher discipline cost to keep `-light` and `-dark` in sync.
        - Either way the runtime behaviour for end users (Phase 6.2 theme toggle) is identical. Pick based on whether the designer is upgrading to Pro — see Open Question 3.
- [ ] **Create a `Typography` Variable Collection** — font sizes, line heights, letter spacing, font weights, font families. Pair with **Text Styles** named semantically (`Display/Large`, `Display/Small`, `Heading/H1…H4`, `Body/Large`, `Body/Default`, `Body/Small`, `Label/Default`, `Caption`). Text Styles compose the typography Variables together so applying a style is one click.
- [ ] **Create a `Spacing` Variable Collection** — `space/0…12` on a strict 4px grid (4, 8, 12, 16, 20, 24, 32, 40, 48, 64, 80, 96). All paddings and gaps reference these; nothing on the screens should ever be `padding: 13px`.
- [ ] **Create a `Radius` Variable Collection** — `radius/none`, `radius/sm` (4), `radius/md` (8), `radius/lg` (12), `radius/full` (9999). Today the codebase has buttons at `9999px` and login card at `8px` — pick a small ramp and stick to it.
- [ ] **Effect Styles for shadows** — `shadow/sm`, `shadow/md`, `shadow/lg`. Same three rungs everywhere; the hover state of a card lifts by exactly one rung.
- [ ] **Apply Variables to every visual property.** The most common designer mistake is colouring a text layer via a Variable but stroking the surrounding frame via a hard-coded hex. We need every fill, stroke, text colour, corner radius, padding, gap, and shadow to reference a Variable. The Figma "Selection Colors" panel and the "Color → Detach" indicators surface where raw values are still in use; a clean file shows zero raw values.

#### B. Component library: build it once, use it on every screen

- [ ] **Split the file into two pages (or two separate files):** `Foundations` (tokens + base components) and `Screens` (page mockups using components from Foundations). This mirrors `ui/src/app/shared/components/` vs `ui/src/app/pages/` and means screen-level changes never accidentally drift the base components.
- [ ] **Build the base components as Figma Components with Variants.** At minimum:
    - `Button` — variants: kind (`primary`, `secondary`, `tertiary`, `ghost`, `destructive`); size (`sm`, `md`, `lg`); state (`default`, `hover`, `active`, `focus`, `disabled`, `loading`); icon (`none`, `leading`, `trailing`, `only`).
    - `Card / Surface` — variants: padding (`sm`, `md`, `lg`), interactive (`true`/`false`), tint (`default`, `success`, `warn`, `danger`).
    - `Input` — variants: state (`default`, `focus`, `disabled`, `error`), label (`floating`, `top`, `none`), prefix/suffix icon.
    - `Badge` — variants: tone (`success`, `warn`, `danger`, `info`, `neutral`), size (`sm`, `md`).
    - `Avatar` — variants: size (`xs`, `sm`, `md`, `lg`), state (`image`, `initials`, `placeholder`).
    - `Header / Nav`, `Footer`, `Modal`, `Toast`, `Tooltip`, `Empty State`, `Pagination`, `Tab Bar`, `Breadcrumb`, `Page Header / Back Nav`.
- [ ] **Name Figma components using the names from the [[Component Inventory for Designer]] companion doc.** That doc lists every reusable building block and every screen that currently exists in the app, with a suggested Title Case Figma name for each, plus a plain-English description of where it appears so you can decide what's worth designing. The designer doesn't need to know the codebase — just the inventory. Once the names line up, the engineer-side mapping in Phase 3 becomes a literal string-compare. If a suggested name in the inventory feels wrong, change it in the inventory doc and tell the engineer — the goal is names that match *somewhere*, not names that match this specific draft.
- [ ] **Use Auto Layout on every component and every screen.** Auto Layout declares padding, gap, alignment, and stretch behaviour — those are exactly the properties Flexbox needs, so it translates almost mechanically to `display: flex; gap: var(--space-3); padding: var(--space-4);`.
- [ ] **Set component descriptions in the Figma Inspect panel** — one or two sentences per base component explaining when to use it. These show up in Dev Mode and become the doc comment in the code.

#### C. Per-screen design discipline (so we don't reintroduce bespoke styling)

- [ ] **Mobile is the primary canvas.** This app's traffic for public / shop / account / staff-check-in is mobile-dominant, so the designer should start each screen at the **375-wide mobile frame** and expand *outward* to desktop, not the other way around. The order: 375 (mobile) → 1280 (desktop) — **no tablet frames** (Open Question 20, decided 7/19/2026). Naming: `Public / Home — Mobile`, `Public / Home — Desktop`.
- [ ] **Every screen built out of components from the Foundations file.** If a screen needs a one-off, the designer should ask whether it should be a new variant of an existing component, or a genuinely new component to add to the library — not a detached/overridden instance.
- [ ] **Annotate the responsive transitions.** At what width does a two-column layout collapse to one? What happens to a data table on mobile — does it scroll horizontally, collapse to cards, or hide secondary columns? Dev Mode annotations are the right place to capture this.
- [ ] **Design the touch states**, not just hover. Pressed, swipe (where relevant), long-press (rarely needed but worth flagging). Hover doesn't exist on touch, so any affordance that only appears on hover is a defect.
- [ ] **Show edge states next to the happy path.** For each meaningful screen, place additional frames showing: empty state, loading state, error state, long-content overflow, logged-out vs logged-in views (where relevant), zero / one / many list items. These are exactly the cases where bespoke CSS used to creep back into the codebase, and pre-designing them prevents that.
- [ ] **Mobile-specific elements** that don't exist on desktop and need their own design: **bottom tab bar** (for logged-in customer navigation), **sticky bottom action bar** (for primary CTAs like "Pay" or "Book Now"), **bottom sheet** (for the mobile version of modals and date pickers), **safe-area insets** at top and bottom for iPhone notches and home indicators.
- [ ] **Use Dev Mode annotations for interactions and edge cases** — "on hover: shadow elevates to `shadow/md`", "card is clickable, navigates to detail", "input shows inline validation under the field on blur". Dev Mode annotations are visible to the MCP server and saved alongside the design.
- [ ] **Name frames after routes, not iterations.** `Public / Home`, `Account / Purchase Detail`, `Manage / Products / Create` — not "Home v3 FINAL final2". The Angular `app.routes.ts` is a perfect source for this naming.
- [ ] **Snap to the 4px grid.** Treat any spacing that isn't on the 4px grid as a bug. (Figma's Layout Grid settings can enforce this visually.)

#### D. Brand & asset deliverables

- [ ] **Logo as SVG** (mono and full-colour versions, with safe-area padding documented). The current logo is referenced from `KNOTTY_YOGA_LOGO_URL` and stretched into a 120-px-wide slot in the header — see `header.component.html`.
- [ ] **Favicon source** as a 512×512 PNG (we can derive the .ico from this). Plus an OG image at 1200×630 for social shares.
- [ ] **Iconography decision** — are we keeping Material Icons (currently used everywhere via `<mat-icon>`), switching to Lucide / Phosphor / Heroicons / Material Symbols, or using bespoke SVGs? If bespoke: export each SVG with consistent stroke widths and `viewBox="0 0 24 24"`.
- [ ] **Real imagery to replace the `.graySquare` placeholders** (about page, hero, instructor portraits). Hi-res PNG or JPEG, plus alt text per image.
- [ ] **Accessibility check** — designer runs the Stark or Able plugin to confirm WCAG AA contrast (4.5:1 body text, 3:1 large text and UI components) for every colour pairing in the Foundations file. This catches "the brand red on white doesn't actually pass contrast" before we ship.

#### E. File access for the Figma MCP server (the engineering unlock)

- [ ] **Enable Dev Mode on the Figma file** (toggle in the file's top-right) and confirm the designer's plan supports it (Dev Mode is included in Figma's paid tiers and free for Dev seats).
- [ ] **Share the file URL with view access**, ideally to a dedicated email so revoking access later is clean.
- [ ] **Generate a Figma personal-access token** (or designer can generate one on their account) — needed by the Figma MCP server to read variables and component definitions. Token only needs read scopes.
- [ ] Once steps E1–E3 are done, Phase 4.1's MCP setup becomes "register the MCP server with the token and read the variables out" — at that point the JSON export step becomes optional rather than required.

#### Quick designer cheat-sheet (one-liner to send him)

> "Please make sure every colour, font, spacing, and radius is a **Figma Variable** (not a raw hex), name them semantically (`color/brand`, `space/md`), and build each screen out of named **Components with Variants**. There's a companion doc called **Component Inventory for Designer** in the shared Obsidian vault that lists every reusable building block and every screen in the app, with the suggested Figma name for each — please use those names so I can map them straight into the code. **Start every screen on a 375-wide mobile frame and expand outward to desktop**, not the other way around — most of our traffic is on phones. Design at least mobile + desktop frames per screen with empty / loading / error states beside the happy path, design touch states (not just hover), include the mobile-only patterns (bottom tab bar, sticky bottom action bar, bottom sheet for modals), enable Dev Mode on the file, and send me a view-access link + a read-only personal-access token."

### 1.2 Pull the Figma source of truth into the repo

- [ ] Once 1.1 lands (or as far as the designer is willing to take it), receive the Figma file URL + read-only token from Phase 1.1.E.
- [ ] Spot-check the file: how complete is the Variable usage? How complete is the component library? Note any gaps that will force a "redesign on the fly" during Phase 3 (so we set expectations upfront).
- [ ] Extract design tokens (covered in detail in Phase 4 — this step is *just* capturing the link and confirming variable usage).
- [ ] Inventory the screens that were re-designed and map each one to the matching Angular route(s) — store this in a markdown table in this doc so we have a checklist.
- [ ] Note any new components that don't yet exist in the app (hero variants, marketing sections, testimonials, etc.) so they're not lost during the SCSS migration.

### 1.3 Inventory the current visual surface

- [ ] Take "before" screenshots of every route currently in `app.routes.ts`, running in `ng serve` against either real or mock backend. Drop them in a sibling folder `Website Makeover Screenshots/` (Obsidian-side, not in the repo).
- [ ] Categorise pages by surface type: `marketing` (public), `auth`, `data-list` (admin/manage/staff list views), `data-form` (admin/manage entry forms), `account` (`my/*`), `shop-flow` (catalog/cart/checkout). The token + class system in Phase 2 should serve each category cleanly.

### 1.4 Decide the technical strategy (write decisions here, don't implement yet)

- [ ] **Material M2 vs M3.** M2 themes are SCSS-compile-time. M3 themes are CSS-token-based and natively support runtime swap. Runtime-driven multi-tenant theming is a stated end goal, so we should plan to migrate Material to M3 tokens (`mat.theme(...)`) before Phase 5. Capture the migration impact (mat-mdc-* classes still apply; M3 changes some density and ripple defaults).
- [ ] **Token transport mechanism for Phase 5.** Recommended: a public `GET /api/site_theme` endpoint that returns a flat `{ token_key: value }` JSON document, served unauthenticated, with a short cache header. An Angular `APP_INITIALIZER` fetches it, builds a `:root { --token: value; … }` string, and injects it into `<head>` before bootstrap. (Avoids a flash of unstyled content.) Document the alternative — inlining tokens server-side into `index.html` — and pick one.
- [ ] **Multi-tenant model.** Single-tenant today. The two realistic models: (a) one binary, separate Postgres database per tenant, theme keys in `config_secrets`; (b) one binary, row-level tenants in the existing schema with a `tenant_id` column on every table. (a) is dramatically simpler for the SaaS pivot and matches the current single-database `Secrets` shape — recommend (a) for now, defer (b) until there's a real second customer.

---

## Phase 2 — Token + utility-class foundation (additive, no rewrites yet)

The goal of this phase is to make every value that's currently hard-coded *expressible* as a token reference. We don't rewrite components yet; we just give them somewhere to point. After this phase a developer touching a new component should know exactly what class to use.

### 2.1 Design-token CSS variables (server: N/A, frontend: assets/styles)

> **Partly shipped 8/11/2026 — and under different names.** Integration Track A in [[Component Inventory for Designer]] wrote the color, type-family, and radius tokens using the **`--theme-*` / `--font-*` / `--radius-*`** names from [[Tenant Theming and Branding]]'s Token Catalog, not the `--color-*` names sketched below, so the per-tenant theming layer and this phase share one namespace. **Do not add a second `--color-*` set.** The bullets below are updated to the as-built reality; the unchecked ones are what's genuinely left.

- [x] ~~Expand `_variables.scss` … emitting CSS custom properties under `:root`~~ — **done**: new `ui/src/assets/styles/_tokens.scss` holds the token layer; `_variables.scss` demoted to a deprecated legacy-alias file (`--red` → `--theme-primary`, etc.).
- [x] **Colour tokens** — done as `--theme-primary` / `--theme-on-primary` / `--theme-accent` / `--theme-on-accent` / `--theme-ink` / `--theme-neutral` / `--theme-surface` / `--theme-surface-tint` / `--theme-background` / `--theme-border`, plus the five status tone pairs (`--theme-success` + `--theme-on-success`, warn, danger, info, muted). Defaults are Ryan's mined Figma values, so `theme-red` usages already moved to `#ED1C26`.
- [x] **Type tokens (families)** — `--font-body` / `--font-heading` / `--font-display`; the `.din*` classes and `body`/`*` resolve through them.
- [ ] **Type tokens (scale)** — the size + line-height ramp `--text-xs/sm/base/lg/xl/2xl/3xl/4xl` and weights `--weight-regular/medium/semibold/bold` are still outstanding; they need Ryan's type scale from his Figma Variables pass.
- [ ] **Space tokens** — `--space-1` through `--space-12` matching the Tailwind 4px grid (so they overlap with `p-2`, `gap-4` etc.).
- [x] **Radius tokens** — done as `--radius-card` (8px) / `--radius-control` (4px) / `--radius-pill` (9999px); the pill-button override in `_mat-button.scss` now reads `var(--radius-pill)`.
- [ ] **Shadow / elevation** — `--shadow-sm/md/lg`.
- [ ] **Z-index tokens** — `--z-header`, `--z-dropdown`, `--z-modal`, `--z-toast` (today the header uses `z-[999]` and `z-[9999]` ad-hoc).
- [x] ~~Add tests: a tiny `tokens.spec.ts`~~ — **done**: `ui/src/app/shared/design-tokens.spec.ts`, 10 specs covering token presence, the mined brand/tone values, legacy-alias wiring, the Tailwind `theme-red` → token chain, the `.din*` font classes, and the Material pill radius.

### 2.2 Wire tokens to Tailwind

- [ ] Extend `tailwind.config.js` to alias the new semantic colour tokens (`brand`, `accent`, `surface`, `border`, etc.) the same way `theme-red` is aliased today. Result: an existing `class="bg-theme-red"` continues to work, and new code uses `class="bg-brand"`.
- [ ] Likewise add `fontSize`, `spacing`, `borderRadius`, `boxShadow`, `fontFamily` extensions that reference the CSS variables. After this step, `class="text-2xl font-display shadow-md rounded-lg p-4"` resolves entirely through tokens.

### 2.3 Realign Material theme to brand (quick visible win)

- [ ] In `_angular-material-theme.scss`, replace `mat.m2-define-palette(mat.$m2-indigo-palette)` with a generated palette built from the brand red (and an accent palette from the brand orange). This alone re-skins every `mat-raised-button color="primary"`, `mat-form-field`, `mat-slide-toggle`, etc., across the whole app for free.
- [ ] Refresh the `_mat-button.scss` override to use the radius token.
- [ ] Add tests: extend `app.component.spec.ts` (or a new `theme.smoke.spec.ts`) to render a `<button mat-raised-button color="primary">` and assert the computed background is the brand red, not indigo. Catches a future palette regression.

### 2.4 Shared layout primitives (replace duplicated wrappers)

> **Shipped 8/11/2026** as `_layout.scss` (`_theme.scss` deleted): `.page-container` (unchanged), `.page-narrow` 640, `.page-medium` 800 (the width half the account pages actually use), `.page-wide` 1200, `.section-stack`. Adopted on the six account pages whose bespoke container matched exactly; pages with different padding kept theirs rather than being silently re-laid-out. Padding is literal `24px 16px` until the space tokens exist.

- [x] Extend `_theme.scss` (rename to `_layout.scss` or add a new partial) with:
  - `.page-container` — already exists, keep it for full-bleed pages.
  - `.page-narrow` — `max-width: 640px; margin: 0 auto; padding: var(--space-6) var(--space-4);` — replaces today's `.account-page-container`, `.cart-container`, `.checkout-container`, the inline `max-w-[420px]` on login, etc.
  - `.page-wide` — `max-width: 1200px; …` — replaces `.catalog .page-container` override.
  - `.section-stack > * + *` — vertical rhythm helper, kills repeated `margin-bottom: 1.5rem` declarations.
- [ ] Add unit tests: `layout-primitives.spec.ts` rendering each class in a sandbox component, asserting computed max-width and padding match the tokens.

### 2.5 Shared surface / card classes (kill the 65× mat-card border duplication)

> **Shipped 8/11/2026** as `_surfaces.scss`. The duplication was **170 copies across 99 files**, not 65. Customer-facing pages are fully swept (94 copies remain, all back office). Note the cascade trap: Material injects its card CSS at runtime, so the global rule must be `mat-card.mat-mdc-card` / `mat-expansion-panel.mat-expansion-panel` — a bare element or single-class selector loses. `.alert-card` ships in all five tones (not just danger/success) and is in use on the class-detail prerequisite banners.

- [x] Add `.surface-card` (border, radius, padding, background, optional shadow) and `.surface-card--interactive` (hover shadow + cursor) to a new `_surfaces.scss` partial. Apply it via a global selector that targets `mat-card` and `mat-expansion-panel` by default — this is allowed because CLAUDE.md already documents that every card in the app *should* have that border. **Result: ~65 component SCSS files lose their bespoke border declaration in one stroke.**
- [ ] Add `.alert-card`, `.alert-card--danger`, `.alert-card--success` for the recurring red/green tinted card pattern (visible in dashboard alerts, cart suggestions, cancel-session dialog).
- [ ] Add tests: `surface-card.spec.ts` renders a `<mat-card>` and asserts the computed border is the token value. Add a Jasmine matcher comparing to `getComputedStyle(document.documentElement).getPropertyValue('--color-border')`.

### 2.6 Shared badge / status classes

- [ ] Add `.badge`, `.badge--success`, `.badge--warn`, `.badge--danger`, `.badge--info`, `.badge--neutral`. Replace the inline `.applied-badge`, `.suggestion-badge`, `.status-badge`, `.role-badge`, `.bundle-savings-tag`, `.bool-true/false`, `.session-time` colour blocks one by one in Phase 3.
- [ ] Document the role-colour mapping in a comment block so Phase 3 refactors are mechanical.
- [ ] Add tests: render each badge variant in a sandbox spec, assert background colour token.

### 2.7 Shared data-density / table classes

- [ ] Add `.data-table` (the bordered, hover-row table pattern from `event-session-card.attendee-table`, `event-session-card.staff-table`, etc.). Promote the white header background, `font-size: 0.85rem`, hover rows. Tests: spec renders a table and asserts header background.
- [ ] Add `.empty-state` and `.empty-state__icon` for the recurring "no items yet" state with a large grey icon.

### 2.8 Form layout helpers

- [ ] Add `.form-card` (`.surface-card` + flex column + gap), `.form-actions`, `.field-row` (label + value flex row used on `user_information`). Helps the auth + account + admin entry-form pages.

### 2.9 Style guide page (the safety net for the rest of the migration)

- [ ] Add a dev-only `/admin/style-guide` route that renders every primitive (`page-narrow`, `surface-card`, all badge variants, the type ramp, the colour swatches, every form element in default + error + disabled states, Material buttons in every colour). This *is* the regression test for the design system — when Phase 3 starts ripping out bespoke SCSS, opening this page will tell you instantly if a token regressed.
- [ ] Add an `app-style-guide.component.spec.ts` that renders the component and snapshots/asserts the token strings.

---

## Phase 3 — Component refactor sweep (mechanical replacement)

Now that Phase 2 has given every component a target to point at, we mass-migrate. The work is largely mechanical and parallelisable. Go folder-by-folder to keep PRs reviewable.

### 3.1 Shared + control components first (lowest layer)

- [ ] `shared/components/header/`, `shared/components/footer/`, `shared/components/event-session-card/`, `shared/components/seat-assignment/`, `shared/components/book-for-selector/`, `shared/components/page-not-found/`, `shared/components/confirm-dialog/`, `shared/components/room-occupancy-badge/`.
- [ ] `controls/` (every form control): replace bespoke colour and spacing with tokens, lean on `.field-row` / `.surface-card` where they fit.
- [ ] For each refactored component, **delete obsolete SCSS** rather than leaving it. Verify each component's spec still passes. For UI-visible components, also add an explicit spec assertion that the component uses the shared class (`fixture.debugElement.query(By.css('.surface-card'))` etc.) so a future regression where someone re-inlines styles fails a test.

### 3.2 Public marketing pages (highest visibility — do these next)

- [ ] `pages/public/home-page/` + `photo-carousel/` — the home page currently does `h1 { text-transform: uppercase; }` and `button { border-radius: 1.5rem; }` inline. Both become token-driven. Also the `nextEvent` card is a perfect candidate for `.surface-card`.
- [ ] `pages/public/about/`, `class-info/` (+ tag-filter chip row), `class-detail/` (+ pricing states, skill chips, series runs), `our-schedule/` (added 7/2026 — the public weekly schedule), `instructors/` + `instructors/instructor-detail/` (added 7/2026), `provider-bio/` (currently a "coming soon" placeholder card), `upcoming-events/`. Delete the dead `staff/` stub component while here — its route was removed (Open Question 23).
- [ ] Replace the lone `.graySquare` placeholder in `styles.scss` and `about.component.html` once real imagery is in place.
- [ ] Add per-page spec coverage for any new bindings introduced (e.g. dynamic class names for variant cards).

### 3.3 Auth pages

- [ ] `login`, `register`, `verify` — move their bespoke `max-w-[420px] border border-gray-300 rounded-lg` to `.page-narrow .surface-card` composition.
- [ ] Update specs to assert the page wrapper uses the shared class.

### 3.4 Shop flow

- [ ] `catalog`, `product-detail`, `cart`, `checkout`, `event-booking` (+ guest-booking fields, added 7/2026), `series-booking` (added 7/2026), `service-booking`, `service-catalog`, `subscription-signup`, `subscriptions`. Cart and checkout have the largest bespoke SCSS files in the app — they'll see the biggest line-count drop.
- [ ] Replace `.suggestion-badge` / `.applied-badge` / `.bundle-savings-tag` with `.badge--info` / `.badge--success`. Update the related component specs.

### 3.5 Account pages

- [ ] `account/profile`, `account/user_information`, `account/update_user_info`, `account/update_user_password`, `account/cards`, `account/purchase_history`, `account/purchase-detail`, `account/my-events`, `account/my-vouchers`, `account/subscriptions/*`, `account/gift-permissions`, plus the seven pages added 7/2026: `account/my-skills`, `account/my-schedule` (+ embedded `upcoming-classes`), `account/today-classes`, `account/upcoming-offerings`, `account/notification-preferences`, `account/attendance-history`, `account/favorite-instructors`.
- [ ] Many of these are essentially the same `.page-narrow .surface-card` form pattern repeated — this slice is where the duplication payoff is largest.
- [ ] Confirm existing pattern from `feedback_account_page_layout.md` (consistent back nav, title font, RouterTestingModule in specs) keeps holding — add it as a `.page-back-nav` primitive in Phase 2.4 if it isn't there.

### 3.6 Admin / manage / staff portals

- [ ] `admin/dashboard`, `admin/edit-db-table`, `admin/table-*`, `admin/event-attendees`, `admin/event-session-staff`.
- [ ] `manage/dashboard`, `manage/products/*`, `manage/events/*`, `manage/subscriptions/*`, `manage/schedules/*`, `manage/providers/*`, `manage/vouchers`, `manage/coupons`, `manage/bundles`, `manage/comps`, `manage/entitlements/*`, `manage/pricing/*`, `manage/room-occupancy`, `manage/room-schedules`, `manage/scheduling-exceptions`, `manage/shift-request-review`, `manage/time-off-review`.
- [ ] `staff/dashboard`, `staff/check-in`, `staff/my-sessions`, `staff/provider-bookings`, `staff/provider-preferences`, `staff/provider-schedule`, `staff/provider-time-off`, `staff/shift-requests`.
- [ ] Many of these reach for the `.data-table` primitive from 2.7.

### 3.7 Calendar

- [ ] `calendar/components/calendar-home/`, `calendar-view-select/`, `date-select/` (still a stub — delete or implement), `views/{calendar-event,day-view,week-view,month-view}/`, `attendance-dialog/`, `skill-requirement-dialog/`. The calendar was rebuilt 7/2026 (public, live data, My Schedule toggle, facility filter, tag colours, rich status chips) — it's the visual-density edge case; keep an eye out for tokens that need adding (`--color-calendar-today-bg`, per-tag accent colours, status-chip colours, hour-line greys, etc.).

### 3.8 Cleanup pass

- [ ] Delete `styles.scss` `.graySquare` / `.toast-*` once each is covered by a tokenised global rule or a per-component class.
- [ ] Re-run `rg -c "#[0-9a-fA-F]{3,6}"` against `ui/src/app/**/*.scss` to confirm hex-literal count has dropped meaningfully (target: < 100, down from 857). Track in a checkbox here.
- [ ] Re-run `rg -c "border: 1px solid #d1d5db"` and confirm it's no longer the 65× duplication.

---

## Phase 4 — Replace token defaults with Figma values

By this point everything routes through tokens; this phase is purely "swap the numbers."

### 4.1 Pull tokens out of Figma

- [ ] **Recommended (needs a paid seat — see Open Question 3): the official Figma MCP server.** Figma's MCP server exposes design variables, component frames, and inspection data to LLM tools, in two flavours: a local server built into the **Figma desktop app** (enabled from the app's preferences) and a Figma-hosted **remote endpoint**. The gate is rate limits, not connection (researched 7/19/2026): free Starter seats get **6 tool calls/month** (unusable for real work); a **Dev or Full seat on Professional** gets **200/day**. With a paid seat: register the server in `~/.claude.json`, and I can read the Variables collections directly (the MCP server has a dedicated variables tool — the Enterprise-only Variables REST API is not needed) and emit the SCSS partial.
- [ ] **Fallback: manual export.** Figma's Variables panel supports JSON export (`Variables → ⋯ → Export variables` or the "Variables to JSON" community plugin). The user can drop the JSON in this Obsidian folder and I'll transform it into the token partial.
- [ ] **Fallback to the fallback: screenshots only.** If neither path is available, I can read swatches off the screenshots and approximate — but the values won't be authoritative and will need designer review before merge.

### 4.2 Overwrite token defaults

- [ ] Replace the values in the token partials from 2.1 with the Figma values. The names should already line up because we mirrored Figma's structure.
- [ ] Re-screenshot every route and diff against the Phase 1.2 "before" screenshots. Designer reviews diffs; iterate.
- [ ] Add Playwright visual-regression baseline if the designer wants change detection going forward (optional; flag for separate discussion).

### 4.3 Imagery, iconography, typography assets

- [ ] If the Figma file uses a new display font, host the WOFF2 in `assets/fonts/` and update `_fonts.scss` (or have it CDN-loaded via `<link>` in `index.html` — the user's call).
- [ ] If the friend produced a new logo, update `KNOTTY_YOGA_LOGO_URL` and the favicon.
- [ ] Replace placeholder `.graySquare` blocks (`about.component.html`) with real hero imagery from the design.

---

## Phase 5 — Database-driven per-tenant theming

This is the SaaS-enablement phase. The Phase 2 token layer is the prerequisite that makes it cheap.

### 5.1 Backend — schema + helpers (lowest layer first per CLAUDE.md)

- [ ] Add a `site_theme` table (`tenant_id` placeholder column for future multi-tenancy, plus `token_key TEXT PRIMARY KEY-ish`, `token_value TEXT`, `updated_at`, `updated_by`). Register it through every checklist step in `CLAUDE.md` "Adding a New Database Table" (db_schema header+cpp, make_database_info, CreateTables, admin_top_level_tables, admin_table_permissions, column data info, friendly names, display templates, CMakeLists).
- [ ] **OR alternative**: piggyback on the existing `Secrets` system and just add a `kSiteTheme*` keyspace (`kSiteThemeColorBrand`, etc.). Simpler today, but a dedicated table is cleaner once tenants get a self-service "Site Theme" page. Recommend a dedicated table; tests are easier.
- [ ] `sql_util/table_helpers/site_theme_table.h/cpp` — DbCrud-based CRUD per the [[feedback_no_sql_in_business_logic]] rule.
- [ ] Per-test-file `site_theme_table_test.cpp` covering insert/update/list/delete, asserting that querying a missing key returns the in-code default (so deployments don't break if the DB row is missing).
- [ ] `business_logic/site_theme/site_theme_helper.h/cpp` — owns the merge of DB rows + Phase 2 defaults; exposes `GetMergedTokens(Transaction&) -> KeyValueTable`. Per-method test in `site_theme_helper_test.cpp`.

### 5.2 Backend — public bootstrap endpoint

- [ ] `endpoints/site_theme.cpp` — `GET /api/site_theme` returns `{ "tokens": { … } }`, no auth required. Short `Cache-Control` (e.g. 60s) to allow CDN caching but still let admin updates propagate.
- [ ] `endpoints/site_theme.cpp` — `PUT /api/admin/site_theme` requires admin permission, updates one or more tokens at a time, invalidates the cache.
- [ ] Tests in `site_theme_test.cpp` per the [[feedback_test_business_logic_endpoints]] rule: anonymous GET, admin PUT, non-admin PUT (403), invalid token key (400), value round-trip.

### 5.3 Frontend — apply tokens at boot before render

- [ ] Add `getSiteTheme(): Observable<SiteTheme>` to `ServerAccess` (interface + Network + Mock + spec test for Mock, per [[feedback_always_test]]).
- [ ] Add a `ThemeService` (`core/services/theme.service.ts`) with `applyTokens(tokens: Record<string,string>)` that writes each as `document.documentElement.style.setProperty('--' + key, value)`.
- [ ] Wire as an `APP_INITIALIZER` in `app.config.ts` that fetches `/api/site_theme` and calls `applyTokens()` before bootstrap completes, so there's no flash-of-default-theme. Fall back silently to baked-in defaults on network error (the CSS already has them).
- [ ] Cache the response in `sessionStorage` for instant cold-start; refresh in the background.
- [ ] Tests in `theme.service.spec.ts` for `applyTokens` (asserts `getComputedStyle` reflects the change) and for the initializer (mock `ServerAccess`, assert `setProperty` calls).

### 5.4 Frontend — admin "Site Theme" page

- [ ] New route under `/admin/site-theme` (or `/manage/site-theme` — `admin` matches today's `admin` role gating).
- [ ] Per token group (colour, type, spacing, radius), render a labelled input with a live preview tile to its right. Submit calls `PUT /api/admin/site_theme` for the changed subset, then re-applies tokens in-place so the admin sees the change without reload.
- [ ] A "Reset to defaults" button that deletes overrides for the selected token group.
- [ ] Component spec (per [[feedback_always_test]]): renders the form, simulates a colour change, asserts the network call and the live `--color-brand` update.

### 5.5 Tenant scoping (skeleton only, no rollout)

- [ ] Add `tenant_id` to the `site_theme` table now (default 0 / "default") so the schema doesn't need a migration when the second tenant arrives.
- [ ] Resolve tenant from request host (subdomain pattern `<tenant>.knottyyoga.app`) in a single helper, even if production today only has one tenant. Document the decision in this section after Phase 1.3 finalises it.

### 5.6 Email / brand bleed-through

- [ ] Confirmation/verification email templates currently inline the brand colour as a literal — review `business_logic/auth/person_verify_mail.cpp` and `business_logic/payment/*_mail.cpp` for hard-coded colours and route them through the same merged-tokens helper so per-tenant email branding works out of the box.

---

## Phase 6 — Hardening, deployment, and longer-tail polish

### 6.1 Visual-regression CI (recommended, not required)

- [ ] Add a Playwright job that renders the Phase 2.9 style-guide page plus a handful of representative real pages and screenshots them. Diff against committed baselines on PRs. Catches accidental token regressions that no unit test can.

### 6.2 Dark mode (in scope per Open Question 9)

- [ ] Wire `:root[data-theme="dark"]` overrides for every colour token defined in Phase 2.1, mirroring the dark-mode Variables that the designer set up in Phase 1.1.A.
- [ ] Add a theme-toggle to the header (the persistent component) + persist the user's choice in `localStorage`. Default to `prefers-color-scheme` on first visit.
- [ ] In Phase 5.3, the `ThemeService.applyTokens` flow needs to know which mode (light / dark) to apply when both sets are loaded from the server, so the DB-driven theming and the user dark-mode toggle compose cleanly.
- [ ] Tests: extend the `tokens.spec.ts` from Phase 2.1 to assert that toggling `data-theme="dark"` on `<html>` produces a different computed `--color-bg` than the light default. Component spec for the header toggle.

### 6.3 Performance / payload

- [ ] After Phase 3, audit final `styles.scss` bundle size — should drop materially given the de-duplication. Adjust the `anyComponentStyle` budget in `angular.json` (currently `maximumWarning: 2kb, maximumError: 4kb`) if any component has legitimately hit the ceiling.

### 6.4 Documentation

- [ ] Update `ui/CLAUDE.md` with a new "Styling" section pointing future-Claude at the token system, the shared classes, and the style-guide page. The current `mat-card` border note can then point at `.surface-card` instead.
- [ ] Add a `ui/src/assets/styles/README.md` describing each partial and the token contract with the backend.

---

## Open questions

> Grouped by when the answer is needed. Earlier groups gate later phases; later groups can be answered as we get there.

### Need answers before Phase 1.1 hand-off to the designer

1. **How much of Phase 1.1 is your friend willing to do?** Sections A (Variables) and E (Dev Mode + token) are the high-leverage ones; B (component library) is hugely helpful but the largest time investment; C and D are polish. It's totally OK to come back with "he'll do A and E only" — we'll plan around it (manual JSON export, hand-extracted component palette). I just need to know what to expect so Phase 1.2 isn't blocked.
	- Mason- He's willing to do all of it. He is enlisted in this vault so please add an instruction section to this document for him.
	- Claude- ✅ Added the "Designer Instructions" section at the top of the Plan, with a punch list and pointers to the two reference docs. Plan now assumes full A–E participation.
2. **Is your friend already using Figma Variables, or the older Color/Text Styles?** If the file pre-dates Variables and is purely Styles-based, that's a migration step on his side. Styles can still be exported via plugins (e.g., "Design Tokens") but the cleanliness gap is significant.
	- Mason- He is going to move everything to Figma variables. He is enlisted in this vault so please add this as a work item.
	- Claude- ✅ Added to the Designer Instructions as the first decision ("Use Figma Variables for every colour, font size, spacing, and radius — not Styles. If your current file is on Styles, please convert it.").
3. **Does his Figma plan include Dev Mode?** Dev Mode is in Figma's paid tiers (Pro/Org/Enterprise) plus free Dev seats. Without Dev Mode we lose the MCP server path and the inline measurement/CSS-snippet inspect — the manual JSON export still works, but it's a slower loop. Worth checking before Phase 1.1.E.
	- Mason- He is using the free version. How much would it cost to do dev mode?
	- Claude- (Caveat: please verify on figma.com/pricing — Figma adjusts pricing periodically.) Free Starter *has* Variables, *has* JSON export, and *has* a limited Dev Mode. The Pro-gated piece is **multiple modes per Variable collection** (the convenient one-toggle way to do light + dark). **Professional** is approximately **$15/editor/month billed annually (~$180/year)** or ~$20/month billed monthly — if the designer upgrades just for the makeover (3–6 months active design) and pays monthly, the spend is roughly **$60–$120 total**. There's also a "Dev seats" SKU for engineers-only that doesn't apply here.
	- 🟢 **Important update on dark mode:** Dark mode is NOT actually Pro-gated. There's a clean free-tier workaround: define paired semantic tokens (`color/brand-light` and `color/brand-dark`) in a single-mode collection, both referencing the shared primitive layer. The engineer assembles them into `:root` + `:root[data-theme="dark"]` CSS at build time. JSON export round-trips fine. Runtime behaviour identical to Pro path. Designer loses the in-Figma "toggle to preview dark" affordance and has to keep `-light`/`-dark` pairs in sync manually. So Pro is now a **quality-of-life decision**, not a **feature-gate**. See Phase 1.1.A for the two paths laid out side-by-side.
	- ❓ **Decision needed:** Does the designer upgrade to Pro for the in-Figma toggle and drift-protection, or stay on free with paired tokens? Worth also checking whether Figma's student / non-profit / referral discounts apply if he goes Pro.
	- Mason- (7/19/2026) Don't we need pro to connect in dev mode through the MCP server?
	- Claude- **(researched 7/19/2026)** Practically yes — but the gate is **rate limits, not the ability to connect**. Current state of Figma's official MCP server (two flavours: a local server built into the Figma **desktop app**, and a Figma-hosted **remote endpoint**):
		- **Starter (free) / View / Collab seats: 6 tool calls per MONTH.** One screen inspection is several calls, so the free tier is a demo, not a workflow.
		- **Dev or Full seat on Professional: 200 calls/day** (10/min); Organization: 600/day. That's a real working budget.
		- **Variables ARE readable through the MCP server** with a paid seat (there's a dedicated variables tool) — so the MCP path covers both the token pull *and* per-frame inspection, without needing the Enterprise-only Variables REST API.
		- **Whose seat matters: the account that connects** (i.e., Mason's, for the engineering pull) — and separately the designer needs Pro only for multi-mode Variables (dark-mode toggle). So the clean setup is **one Professional team: designer on a Full seat (~$16/mo), Mason on a Dev seat (~$12/mo)** — ~$28/mo for the duration of the makeover, cancellable after.
		- **Zero-cost fallback remains fully workable:** stay free, use paired `-light`/`-dark` tokens, export Variables to JSON via plugin, and design-review from screenshots. You lose MCP inspection and the in-Figma dark toggle; the shipped result is identical.
		- Sources: [Figma MCP rate limits & access](https://developers.figma.com/docs/figma-mcp-server/rate-limits-access/), [Figma pricing](https://www.figma.com/pricing/). (Prices move — verify before purchasing.)
4. **Component-naming convention** — happy with the "Figma component name == Angular component name" rule from 1.1.B? If your friend already named components differently, we'll need a mapping table; tell me which way is easier.
	- Mason- Is everything that he would need to do in [[Component Inventory for Designer]]? If so, he basically just needs to give the things the names in that document and then you can map them over?
	- Claude- **Yes — that's exactly the design.** The naming is the linchpin: when I see a Figma frame called `Event Session Card`, I know to apply the `.event-session-card` class in code; when I see `Surface Card`, I know it's the shared `.surface-card` primitive. Without that mapping I'd be eyeballing screenshots and guessing. With it, the component identification step in Phase 3 is mechanical. ⚠️ **Caveat: the names handle *which* component, not *how it looks*.** Colours, spacing, type sizes, and shadows still need to come from Figma Variables (Phase 1.1.A) — names alone don't carry visual values. And the layout structure (gaps, padding, alignment) comes from Auto Layout (Phase 1.1.B). So the three stack: **names = mechanical component mapping**, **+ Variables = mechanical colour/type/spacing values**, **+ Auto Layout = mechanical Flexbox translation**. Since he's willing to do all three, we get the fastest path end-to-end. After he hands over the file I'll spot-check it and flag any frames whose names don't match the inventory — small drift is fine (we'll add a rename map), large drift means we regroup before I write code.

### Need answers before Phase 4 (Figma → tokens)

5. **Figma access.** Once Phase 1.1.E is done, share the Figma file URL + a read-only personal-access token so I can set up the MCP server. If you'd prefer not to expose a token, the manual JSON-export fallback works too — just say which you'd rather do.
6. **Token granularity for swap-ability.** Do you want the full two-layer token scheme (primitive `color/red/500` + semantic `color/brand`), or just the semantic layer? Two layers is more work for the designer but makes per-tenant rebranding much easier later (you can swap the brand primary by swapping one reference, vs. find-and-replacing across all semantic tokens).
	- Mason- He's okay with doing the full two-layer token scheme.
	- Claude- ✅ Locked in. Phase 1.1.A and Phase 2.1 already assume two-layer. Per-tenant theming in Phase 5 will swap semantic references, not primitives.
7. **Iconography decision** — keep `<mat-icon>` (Material Icons), or move to Lucide / Phosphor / Heroicons / Material Symbols / bespoke SVGs? Affects what your friend designs against and whether Phase 3 has to replace every icon as it goes.
	- Mason- Let's keep the material icons please.
	- Claude- ✅ Locked in. Designer should use Google Material Icons in his Figma designs (the same set Angular Material currently exposes via `<mat-icon>`). No icon-replacement work in Phase 3.

### Need answers before Phase 2 (foundation work)

8. **Material M3 migration.** Phase 1.4 recommends moving from M2 to M3 before Phase 5 because M3's CSS-token model makes runtime theming dramatically cleaner. M3 will cause small visual nudges (density, ripple, default radii). OK to do that migration as part of Phase 2.3, or do you want it kept as a separate, later step?
9. **Dark mode.** In scope for this initiative or deferred? It's cheap if we plan the tokens for it now (and Phase 1.1.A asks the designer to add a Dark mode to the Colors collection); expensive to retrofit later.
	- Mason- He is willing to do dark mode so let's do it.
	- Claude- ✅ In scope. Phase 1.1.A now documents two paths: Pro multi-mode Variables (cleaner, in-Figma toggle) or free-tier paired tokens (`*-light` + `*-dark` semantics referencing one primitive layer). Both produce identical runtime behaviour via Phase 6.2's `:root[data-theme="dark"]` overrides + header toggle + `prefers-color-scheme` default. Designer picks the path based on whether he upgrades — see Open Question 3.
10. **Breakpoints.** Pick the canonical set so the designer and Tailwind agree. Recommend three: 375 (mobile), 768 (tablet), 1280 (desktop). The existing Tailwind config uses defaults (sm:640, md:768, lg:1024, xl:1280) — should we shift Tailwind to match or shift design to match?
	- Mason- Let's go with your recommendation.
	- Claude- ✅ Locked in — **amended 7/19/2026 by Open Question 20: no tablet frames.** Designer designs at 375 and 1280 only. Tailwind still gets the aligned `screens` map in Phase 2.2 (`sm: 375`, `md: 768`, `lg: 1280`) because the CSS needs a mid breakpoint for where layouts collapse — but 768 is an engineering-side line driven by per-screen annotations, not a designed frame. Will document the breakpoint contract in `ui/src/assets/styles/README.md` in Phase 6.4.

### Need answers before Phase 5 (DB-driven theming)

11. **Multi-tenant model.** Phase 1.4 recommends "one DB per tenant" (simpler, matches today's `Secrets` shape) over row-level multi-tenancy. Confirm? Or do you have a target customer count or hosting constraint that pushes the other way?
12. **Scope of "per-tenant".** Just colour + type + radius, or also: logo, favicon, page title, email signature, hero copy, footer links, social URLs? Each of these is independently DB-driven-able and has different stakeholder review requirements.
13. **Tenant resolution mechanism.** If we adopt one DB per tenant, tenants are resolved by which Postgres connection the server points at, and Phase 5.5's `tenant_id` column becomes vestigial. Are we OK leaving the column unused in the dedicated-DB scenario as a "just-in-case" hedge, or do you want it dropped?
14. **Who can edit the site theme?** Admin role only, or a new `manage_theme` permission so studio owners can change colours without full admin? Affects Phase 5.4's route gating.

### Mobile-related questions (need answers as we approach Phase 2/3)

15. **Bottom tab bar vs. hamburger for logged-in users.** Recommended: add a bottom tab bar (Home / Schedule / My Bookings / Account) on mobile for `/my/*` and `/shop/*`, keep the hamburger as the secondary menu, and keep the header for branding + cart + account dropdown. Confirm? Or do you want to keep the hamburger as the only mobile navigation?
	- Mason- Let's keep the hamburger.
	- Claude- ✅ Locked in. Removed the `bottom-tab-bar` primitive from Phase 2's mobile work items and the corresponding Phase 3 task. The existing `header-mobile-menu` remains the canonical mobile nav. Sticky bottom action bars (for primary CTAs like "Pay") are *still* in scope — those are a different pattern from a persistent tab bar. Will update the [[Component Inventory for Designer]] to drop the `Bottom Tab Bar` entry, since the designer no longer needs to mock that up.
16. **Apple Pay / Google Pay.** Recommended: add as part of Phase 3 — Square's Web SDK supports both, the backend already accepts whatever token the SDK produces, and the conversion-rate impact on mobile is substantial. Any reason not to? (Note: Apple Pay specifically requires domain verification with Apple via Square's dashboard — a one-time setup step that should happen before launch.)
17. **PWA scope.** Phase 6 proposes install-to-home-screen + offline catalogue + push notifications. Push notifications in particular are powerful for a fitness studio (booking reminders, waitlist promotion, schedule changes) but add a privacy / consent surface and require backend work (web-push server keys, subscription storage). Three options: (a) full PWA with push, (b) PWA without push (install + offline only), (c) skip PWA entirely. Recommend (b) as Phase 6, (a) as a Phase 7 follow-up only if there's appetite.
18. **Mobile table strategy per page.** Three rendering modes available (horizontal-scroll / cards / collapse-and-tap). I'd pick: `my-events`, `purchase-history`, `my-vouchers`, `cart` → cards. All `/manage/*` and `/admin/*` tables → horizontal-scroll. The dashboard alert lists → cards. Does that match your intuition, or do you want a different cut?
	- Mason- that sounds fine.
	- Claude- ✅ Locked in. The `responsive-data-table` work item in the Mobile section now records the per-page assignment so Phase 3 doesn't have to re-decide.
19. **Calendar on mobile.** Recommended: default to day view, swipe between days, "jump to month" opens a bottom-sheet picker. Week view exists but is borderline on phones — punt to tablet+. OK to default-to-day, or do you want week view as the mobile default?
	- Mason- (7/19/2026) Yeah, day view probably makes the most sense for mobile.
	- Claude- ✅ Locked in. Mobile (below `md`) defaults to day view with swipe-between-days and a bottom-sheet month picker; week/month views stay desktop. Recorded in the Phase 3 calendar work item and the Decisions list. Note the calendar itself was substantially rebuilt since this plan was written (Classes Phase 13: now public, live data, My Schedule / Full Schedule toggle, facility filter, tag colours) — the [[Component Inventory for Designer]] entry describes the current state.
20. **Tablet support.** Today the breakpoint guidance is "design mobile + desktop, tablet optional". Confirm tablet isn't a priority surface, or call out which screens specifically should be designed for 768 (calendar and admin tables are the obvious candidates).
	- Mason- (7/19/2026) I think desktop and mobile are enough. I don't think we need to design explicitly for tablet.
	- Claude- ✅ Locked in. Design deliverables are 375 + 1280 frames only — no 768 frames anywhere, including the calendar and data tables. Engineering still keeps a `md` (768) breakpoint in the CSS for where layouts collapse; the in-between behaviour comes from per-screen Dev Mode annotations instead of dedicated tablet mockups. Updated the Decisions list, Phase 1.1.C, Open Question 10, and the inventory's Part 2 preamble.

### Can be answered any time

21. **Public marketing pages.** Are the friend's mockups limited to the existing pages, or did he design new ones (testimonials, pricing page, FAQ, blog landing)? Phase 1.2 needs to surface the gaps so we don't ship a half-redesigned public surface.
	- Mason- He's mostly redone existing pages but I'm working with him on fleshing out the missing stuff (which we know there are quite a few things to do)
	- Claude- 👍 Acknowledged. A short follow-up: **could the designer drop a "didn't redesign" / "to-do" list in this doc** (a `- Designer- ...` bullet under this question) so Phase 1.2's screen-to-route mapping knows which routes are mockup-driven vs. legacy-styling? Even a rough list helps prioritise the Phase 3 sweep.
	- Mason- (7/19/2026, answering the follow-up) We aren't redoing any of the admin or staff dashboards.
	- Claude- ✅ That closes the follow-up. Reading "admin" as the whole owner back-office: **no Figma frames for anything under `/manage`, `/admin`, or `/staff`** — all three portals inherit the new look from the shared building blocks during the Phase 3 engineering sweep (Phase 3.6 still happens, it's just code-side with no mockups to match). The three Part 2 sections in [[Component Inventory for Designer]] are now marked accordingly. One flagged exception worth a frame *if he finds time*: **Staff / Check-In** — it's the lone back-office screen used on a phone in-studio (the plan's Phase 3 mobile-first item); otherwise engineering composes it from the building blocks. Also note the inventory gained a "what changed since your first pass" section on 7/19/2026 — a wave of class-feature pages shipped after his mockups, and the new **customer-facing** ones (Our Schedule, Instructor Detail, series booking, the new account pages, the rebuilt calendar) DO want design attention even though the back office doesn't.
22. **Visual-regression CI.** Is there appetite to set up Playwright + screenshot diffs? It's the only realistic way to catch token regressions in PRs — but it adds CI minutes and baseline-maintenance overhead.
23. **The literal "staff works!" page** at `pages/public/staff/staff.component.html` — keep, delete, or design? It's currently routed nowhere in `public.routes.ts` so it may already be dead.
	- Claude- ✅ **Resolved by the July feature work (verified 7/19/2026):** the `/staff` public route no longer exists in `public.routes.ts`, so visitors can't reach it. The component files (`pages/public/staff/`) are still on disk as dead code — Phase 3.2 deletes them during the public-pages sweep. The stub note has been removed from the inventory; nothing for the designer here.
24. **Animation / motion.** Does the design include any motion (page transitions, micro-interactions on cards, animated empty states)? Phase 1.1.C asks the designer to annotate these in Dev Mode, but they're easy to forget. If yes, we'll want a `--motion-duration-*` and `--motion-easing-*` token family in Phase 2.1.
	- Mason- (7/19/2026) No plans currently. Mainly want to get a nice looking, functional website. Bells and whistles can wait.
	- Claude- ✅ Locked in. No `--motion-duration-*` / `--motion-easing-*` token family in Phase 2.1, and no motion-annotation ask on the designer. Material's stock transitions (ripples, dialog fades, drawer slides) stay as-is. A motion token family can be added later without reworking anything — the token system is additive.
