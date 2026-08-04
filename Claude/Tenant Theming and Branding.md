---
fileClass: Reference
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/4/2026
Version: 0.1
tags: 
---

# Tenant Theming and Branding

# Overview

This is the plan for making the frontend **whitelabel-ready**: a new studio signs up as a tenant and runs the site *as is* — their own name, logo, colors, fonts, photos, and marketing copy — with zero code changes and zero layout changes. Mason requested this 8/4/2026 (in [[Component Inventory for Designer]], "Mason - Updates to the document 8/4/2026") and confirmed it should be its own document (OQ-D2 there).

This doc **fuses and supersedes** the theming sketches scattered across three other plans:

- **[[Website Makeover]] Phase 5** ("Database-driven per-tenant theming") — superseded by this doc. The makeover keeps Phases 1–4 (Ryan's tokens → CSS variables → component sweep) and 6; when its Phase 5 comes up in sequence, execute *this* doc instead.
- **[[Converting the server to a multi tenant Saas architecture]] Phase 7** — shipped 7/23/2026; its as-built hooks (`/api/site_info`, `SiteConfigService`) are this doc's foundation, and its 7.4 note ("align `site_info` so it can carry or coexist with theme tokens rather than duplicating them") is honored by decision D2 below.
- **[[Componentizing the frontend]]** styling rules — the `@honuware/ui` library is CSS-framework-agnostic, bans hardcoded colors, and never ships a brand; nothing here touches the library.

The designer-facing view of this work (the 🎨 markers, the five design rules, the fake-second-studio proof frame) lives in [[Component Inventory for Designer]]; this doc is the engineering side. Planning here is iterative — answer open questions inline, keep the Overview intact, and tick checkboxes as phases land.

---

# Goal & Non-Goals

**Goal.** One built Angular bundle + one server deployment serve N studios. Everything brand-specific arrives at runtime from the tenant's own data: token values, copy, imagery. "Onboard a studio" becomes: create the tenant (`--create_tenant`), fill in the Site Theme form, upload photos. No forks, no per-tenant builds, no CSS edits.

**Non-goals (explicitly out of scope).**
- **Per-tenant layout or feature differences.** Every tenant gets the same pages, routes, components, and features. A studio that wants a different *shape* of site is a different app, not a theme.
- **Per-tenant custom pages / arbitrary HTML injection.** Content slots are typed and enumerated; there is no "paste your own page" escape hatch (XSS surface, support burden).
- **A theme marketplace / multiple switchable themes per tenant.** One theme per tenant, editable.
- **`@honuware/ui` changes.** The library stays brand-free and token-consuming; per the componentization plan's Q12, the `HONUWARE_BRANDING` token is created only if a library surface ever needs it (none does today).
- **CommunityFinder.** It runs static compile-time branding and never calls `/api/site_info` — unaffected by anything here.

---

# Current State (as-built, verified 8/4/2026)

What already exists — this plan builds on it rather than re-inventing it:

**Server (honuware + app):**
- Multi-tenant Model C is live: per-request tenant resolution (`X-Honuware-Site` / `FixedTenantResolver`), one database per tenant, per-tenant `config_secrets` with at-rest encryption.
- `GET /api/site_info` (honuware `components/platform/endpoints/site_info.*`) returns `display_name`, `website_url`, `logo_url` for the resolved tenant. Unauthenticated, `Cache-Control: public, max-age=300`. Sources: `::Mail::LoadTenantBranding` (studio name ← `kMailSenderName`, website ← `kWebsiteAddressLogin`) + `Secrets::kSiteLogoUrl` (default `""`), falling back to `TenantContext.displayName`.
- `Mail::TenantBranding` — framework mail templates are already brand-free; app mail templates were parameterized in tenancy Phase 4.5. Emails already carry the right studio name, sender, and links per tenant.
- App-side secret **defaults** registration exists (`business_logic/app_secret_values.cpp`) — the mechanism that seeds brand default values into every tenant's `config_secrets`. This is the natural home for new content-slot defaults.
- `config_secrets` is **deliberately not surfaced** in the generic admin CRUD (it holds live credentials) — so anything stored there needs curated endpoints, which Phase 5's Site Theme page provides anyway.
- Home-page carousel photos are already per-tenant data: `home_page_photos` table + `GET /api/home_page_photos/<count>` + the generic photo system.

**Frontend (app shell):**
- `SiteConfigService` (`core/services/site-config.service.ts`): APP_INITIALIZER fetches `/api/site_info` before first render, merges non-empty API fields over `DEFAULT_SITE_CONFIG`, never blocks render on failure. Today's shape: API-driven `displayName` / `websiteUrl` / `logoUrl` + app-static `logoAsset`, `logoAltText`, `addressLines`, `contactEmail`, `taglineLines`, `aboutText`.
- Header (logo + alt), footer (address, email, tagline, copyright name), and the About text already read from `SiteConfigService`.
- A small CSS-variable palette seam already exists: `assets/styles/_variables.scss` defines `--red #F50C22`, `--orange #FF9933`, `--gray #666666`, `--black`, `--white` on `:root`, and `tailwind.config.js` maps `theme-red` → `var(--red)` etc. **Changing `:root` variables at runtime already restyles every Tailwind `text-theme-*` / `bg-theme-*` usage.** The Material theme does not read these yet.
- Fonts: D-DIN self-hosted in `assets/fonts/d-din`, exposed as `.din` / `.din-bold` / `.din-condensed-bold` classes (`assets/styles/_fonts.scss`) — class-based, not variable-based.

**Still hardcoded (the gap this plan closes):**
- Home hero headline + subline and the secondary hero image (`safeSpaceLogo`) — in the home template.
- Getting Started: the seven steps' copy — in the component's `allSteps` array.
- The membership-section marketing blurb — in the home template.
- `index.html`: `<title>Knotty Yoga Fitness Studio</title>` + `favicon.ico`.
- Token *values* beyond the 5 palette colors: no radius/spacing/type variables; Material theme static; font families not swappable.
- The `SiteConfig` "app-static" fields (address, tagline, about…) are centralized but **not yet per-tenant** — they come from code defaults, not from the tenant's data.

---

# The Three Layers of a Theme

1. **Design tokens** — *values* for named roles: colors, font families, radius (spacing/elevation stay app-fixed for now). Applied as CSS custom properties before first paint. A tenant sets `primary = #0B6E4F`; every button, link, and chip follows.
2. **Content slots** — typed, enumerated copy: studio name, hero headline, tagline, about text, getting-started copy, footer address/contact, marketing blurbs. Short slots are single-line strings; long ones are **markdown** rendered through the blog's existing ngx-markdown pipeline + prose styles.
3. **Imagery** — logo, secondary hero image, carousel photos (done), favicon.

**Never themable:** layout, component anatomy, routes, feature set, icon set, status-color *semantics* (success stays green-ish per tenant tone, but "booked/waitlisted/cancelled" meanings never change).

---

# Design Decisions

Locked as working decisions; each has an open question below only where a real alternative survives scrutiny.

- **D1 — Storage: `config_secrets` keys, not a new table.** The branding trio (`kMailSenderName`, `kWebsiteAddressLogin`, `kSiteLogoUrl`) already lives there; the per-tenant provisioning, defaults registration (`app_secret_values.cpp`), and at-rest encryption all exist. Theme/content values are just more keys with brand-free names and Knotty Yoga default *values*. Encryption of non-secret copy is harmless overhead; a second per-tenant KV store would be pure duplication. The generic admin exclusion is a feature: edits go through the curated Site Theme endpoints only.
- **D2 — Transport: extend `/api/site_info`, no second bootstrap call.** The response grows two objects — `theme` (token key → value) and `content` (slot key → value) — beside the existing three fields. One cached public fetch per boot; honors tenancy 7.4's "carry, don't duplicate" note; supersedes Makeover 5.2's separate `/api/site_theme`. The payload stays small (tens of short strings + one about-markdown blob).
- **D3 — Application: the existing APP_INITIALIZER applies tokens to `:root` before first render.** `SiteConfigService.load()` already runs pre-render; it gains "write `theme` entries as CSS custom properties on `document.documentElement`, set `document.title`". No FOUC beyond today's (defaults render if the fetch is slow/failed — same graceful-fallback contract as now).
- **D4 — Fonts: a curated, self-hosted set; role-based selection.** Tenants pick `display` / `heading` / `body` families from a short list the app ships (D-DIN + a few open-licensed families, self-hosted — no external font CDNs). Arbitrary font upload is out (licensing, loading, CSP). D-DIN is Knotty Yoga's *value*, not the system. The `.din*` classes become thin wrappers over `--font-display` / `--font-heading` / `--font-body` so existing templates keep working unchanged.
- **D5 — Long copy is markdown.** `about` (and any future long slot) is a markdown string rendered with the blog's ngx-markdown pipeline and prose styles — sanitization on, one rendering path, one set of text styles. The About page becomes the second consumer of the blog's prose system.
- **D6 — Structured copy keeps its structure.** Getting Started stays seven app-defined steps with fixed CTAs/routes; tenants edit each step's title + body text (and the page intro). No per-tenant step add/remove — the flow *is* the product.
- **D7 — Imagery v1 is URL + existing photo plumbing, no new upload surface.** `site_logo_url` already exists; add `site_hero_image_url` and `site_favicon_url` the same way. Carousel photos already upload through `home_page_photos`. A dedicated `site_assets` upload flow is a later nicety, not v1 (no-premature-code).
- **D8 — Dark mode: structure now, pairs later.** Tokens are defined so a `-dark` variant *can* exist per key, but v1 ships a single (light) value set. The makeover's dark-mode phase consumes the token structure when it lands.
- **D9 — Material bridge via system variables.** Where Material components must follow tenant color, override Material's CSS custom properties (`--mat-*` system tokens) from the same boot application rather than compiling per-tenant SCSS themes. The full Material-theme realignment stays Makeover 2.3; this doc only wires the bridge for the tenant-variable values.
- **D10 — Validation is server-side and strict.** Colors must parse as `#RRGGBB`; URLs must be http(s) or root-relative; markdown capped (e.g. 64 KB); font selections must be in the curated list. A junk value falls back to the default rather than breaking every page of a tenant's site (same philosophy as the blog's out-of-range month).

---

# Token Catalog v1

The starting contract between Ryan's Figma Variables (Makeover 1.1.A / 4.1) and the code. **Names get reconciled once Ryan's Foundations file lands** — if his layer names differ, his win and this table updates; values below are Knotty Yoga's defaults (= today's live values).

| Role | CSS variable | KY default | Notes |
|---|---|---|---|
| Primary / brand | `--theme-primary` | `#F50C22` (the red) | Aliases today's `--red`; keep `--red` as a deprecated alias until the makeover sweep retires it |
| Secondary / accent | `--theme-accent` | `#FF9933` (the orange) | Aliases `--orange` |
| Neutral text | `--theme-neutral` | `#666666` | Aliases `--gray` |
| Ink / black | `--theme-ink` | `#000000` | Aliases `--black` |
| Surface / white | `--theme-surface` | `#FFFFFF` | Aliases `--white` |
| Page background | `--theme-background` | `#FFFFFF` | Distinct from surface so cards can sit on a tinted page |
| Success tone | `--theme-success` | (green in use by badges) | Status *semantics* fixed; tenants tune the shade only |
| Warn tone | `--theme-warn` | (amber in use) | 〃 |
| Danger tone | `--theme-danger` | (red in use — decouple from brand red) | 〃 |
| Info tone | `--theme-info` | (blue in use) | 〃 |
| Display font | `--font-display` | `D-DIN Condensed Bold` stack | `.din-condensed-bold` becomes `font-family: var(--font-display)` |
| Heading/bold font | `--font-heading` | `D-DIN Bold` stack | `.din-bold` → `var(--font-heading)` |
| Body font | `--font-body` | `D-DIN` stack | `.din` → `var(--font-body)` |
| Card radius | `--radius-card` | `8px` | The mat-card/post-card radius in use |
| Control radius | `--radius-control` | `4px` | Buttons/inputs/chips |

Config-secret key per token: `site_theme_<role>` (e.g. `site_theme_primary`, `site_theme_font_body`, `site_theme_radius_card`). The `theme` object in `/api/site_info` maps CSS-variable name → value, so the frontend applies it without a lookup table.

The exact status-tone hex values in use get pinned during Phase 3 grounding (they're scattered across badge SCSS today; the makeover's badge consolidation is where they become one set).

---

# Content-Slot Catalog v1

Engineering twin of the designer table in [[Component Inventory for Designer]]. Type `line` = single-line string, `lines` = ordered list (stored newline-separated), `md` = markdown, `url` = validated URL.

| Slot | Secret key | Type | Consumed by | KY default (sample content) |
|---|---|---|---|---|
| Studio display name | `mail_sender_name` *(exists)* | line | site_info `display_name`, emails, footer copyright, About menu label | "Knotty Yoga" |
| Website URL | `website_address_login` *(exists)* | url | email links, site_info | current URL |
| Logo | `site_logo_url` *(exists)* | url | header (falls back to bundled asset) | `""` → `KnottyYoga_logo_white.svg` |
| Logo alt text | `site_logo_alt` | line | header img alt | "Knotty Yoga logo" |
| Browser title | `site_browser_title` | line | `document.title` at boot | "Knotty Yoga Fitness Studio" |
| Favicon | `site_favicon_url` | url | `<link rel="icon">` swap at boot | `""` → bundled `favicon.ico` |
| Hero headline | `site_hero_headline` | line | home hero | "Knotty Yoga Is An Inclusive, High-Level Acrobatic Fitness Studio." |
| Hero subline | `site_hero_subline` | line | home hero | "Get into the best shape of your life, the right way." |
| Hero secondary image | `site_hero_image_url` | url | home hero right side | `""` → the safe-space logo asset |
| Tagline | `site_tagline_lines` | lines | footer | "That which doesn't kill you / makes you hotter. 💪🤸" |
| Address | `site_address_lines` | lines | footer | "2545 152nd Ave NE, Redmond, WA 98052" |
| Contact email | `site_contact_email` | line | footer mailto | `info@knottyyoga.com` |
| About | `site_about_markdown` | md | `/about` (rendered as prose) | current `aboutMe` blurb |
| Getting Started intro | `site_start_intro` | line | `/start` header | current intro line |
| Getting Started steps 1–7 | `site_start_step_<n>_title` / `_body` | line / line | `/start` step cards (CTAs/routes stay app-fixed) | the seven shipped steps |
| Membership blurb | `site_membership_blurb` | line | home Memberships section | "Unlimited classes, priority sign-ups, and the whole community — pick the tier that fits how you train." |
| Social links | `site_social_links` | lines (`label|url`) | footer icon row | current links |

Not slots (already per-tenant data, no work): carousel photos (`home_page_photos`), blog posts, class/product/instructor content, membership tier names (they're `products` rows).

---

# Phased Implementation Plan

Conventions per [[../CLAUDE.md|CLAUDE.md]] + standing memory: backend before frontend inside every phase; every item lands with its tests in the same session; Linux docker is the C++ gate, `ng test`/`ng build`/`ng lint` the Angular gate; live hand-testing steps close each phase. Phases 1–2 have no dependency on Ryan or the makeover and can start immediately; Phase 3 defines the CSS variables itself if Makeover 2.1 hasn't landed first (they're the same variables — first mover creates, second consumes).

## Phase 1 — Content slots, server side

- [ ] Register the new content-slot secret keys (brand-free *names*) in honuware's `secret_keys.h` + `FillInSecretsStringView`, with empty/neutral framework defaults — mirroring how `kSiteLogoUrl` landed. *(hw — needs a pin bump)*
- [ ] Register Knotty Yoga default *values* app-side in `business_logic/app_secret_values.cpp` (the table above's right-hand column). *(app)*
- [ ] Extend `GET /api/site_info` with the `content` object (slot key → resolved value, defaults filled) and the `theme` object (empty until Phase 3 — present so the payload shape is stable). Keep the endpoint public + cacheable; keep the pure builder split. *(hw)*
- [ ] Server-side validation helpers per D10 (hex color, URL shape, size caps) — shared by site_info's read path (defensive normalize) and Phase 5's write path. *(hw)*
- [ ] Tests: builder field-mapping for `content`/`theme`; defaults fill when unset; validation accepts/rejects the documented shapes; app suite green proving the new defaults seed on a fresh DB.

## Phase 2 — Content slots, frontend

- [ ] `SiteConfig` grows the slot fields; `load()` merges `content` over the existing static defaults (same non-empty-wins rule). The current hardcoded values stay as the fallback constants — a dead API never blanks the site.
- [ ] De-hardcode the consumers: home hero headline/subline/secondary image, membership blurb, Getting Started intro + step copy (titles/bodies from config, CTAs/routes/icons from the component), `document.title`, favicon swap, logo alt.
- [ ] `/about` renders `site_about_markdown` through ngx-markdown with the blog's prose styles (route-scoped `provideMarkdown()`, same as `/blog`).
- [ ] Specs per consumer (the standing component-spec rule) + `SiteConfigService` merge cases + `ServerAccess.mock` returns the dev content block (+ mock spec).

## Phase 3 — Token pipeline

- [ ] Server: accept + serve `site_theme_*` keys in the `theme` object (CSS-var name → validated value). *(hw)*
- [ ] Frontend: boot application — write each `theme` entry onto `document.documentElement`; alias the legacy `--red`/`--orange`/`--gray` to the new `--theme-*` so existing Tailwind mappings restyle immediately.
- [ ] `.din*` font classes re-based onto `--font-*` variables; ship the curated font set self-hosted with `@font-face` for each (D4).
- [ ] Material bridge: override the `--mat-*` system tokens that map to primary/accent at boot (D9) — scoped to what's visibly brand-colored today (buttons, toggles, spinner), not a full re-theme.
- [ ] Reconcile token names with Ryan's Foundations file when it lands (his names win; update the catalog above).
- [ ] Tests: application function unit-tested (writes vars, skips invalid), spec that a themed boot restyles a `theme-red` consumer, font-class regression spec.

## Phase 4 — Imagery v1 (URL-based)

- [ ] Wire `site_hero_image_url` + `site_favicon_url` consumers (Phase 2 stubs them behind defaults; this phase completes fallback behavior + error handling — a 404'd image falls back to the bundled asset, never a broken-image icon).
- [ ] Document the constraint set for each slot (logo: light-on-dark, height-bounded; hero image: aspect guidance) in the onboarding runbook — these match Ryan's slot-constraint rules in the inventory doc.
- [ ] *(Deferred, recorded not planned: a `site_assets` upload flow so tenants don't need externally hosted URLs — revisit after the first real second tenant.)*

## Phase 5 — Admin "Site Theme" page

- [ ] Backend: curated endpoints `GET /api/manage/site_theme` (full editable set, unset-vs-default distinguished) + `PUT /api/manage/site_theme` (validated writes to the tenant's `config_secrets`). Admin-only via the existing permission machinery. **Never** the generic CRUD surface — `config_secrets` stays excluded. *(hw or app — decide at grounding by where the slot registry ends up)*
- [ ] Frontend: Manage-portal page (back office — inherits building blocks, no Ryan design) with sections **Brand basics** (name, title, logo/favicon URLs, contact, address, socials) / **Copy** (hero, tagline, blurbs, Getting Started steps) / **About** (markdown textarea + live prose preview, reusing the blog editor's split-pane pattern) / **Colors & fonts** (color pickers per token role, font dropdowns) — with per-field "reset to default".
- [ ] "Changes appear within ~5 minutes" note (the site_info cache) + a save-confirmation.
- [ ] Tests: endpoint auth/validation/round-trip; component specs per section.

## Phase 6 — Provisioning, runbook, and the fake-studio proof

- [ ] `--create_tenant` verification: a fresh tenant boots with every slot on defaults (this mostly falls out of the defaults machinery — the test proves it).
- [ ] Onboarding runbook: the checklist a new studio walks (Site Theme form top to bottom, photo uploads, DNS/CloudFront per the tenancy plan's Phase 8).
- [ ] **The fake-studio smoke test** — engineering twin of Ryan's OQ-D3 proof frame: a second local tenant with an invented brand (different palette, wide logo, two-line name, different hero copy); walk Home / Our Classes / Blog / Getting Started / About / an email; nothing Knotty-branded may leak. This list = the regression checklist for every future theming change.
- [ ] Live hand-testing steps for the whole feature (blank DB, two tenants, exact form fields + values, per the precise-instructions rule).

---

# Sequencing with the other plans

- **Independent now:** Phases 1–2 (content slots) touch nothing Ryan or the makeover owns.
- **Coordinates with Makeover Phase 2:** Phase 3 shares the CSS-variable layer. Either lands first; whoever is second consumes the other's variables. Ryan's Figma token *names* (Makeover 1.1.A/4.1) get reconciled into the catalog when they exist — they gate nothing.
- **Supersedes Makeover Phase 5** (noted there when the makeover doc next gets touched; the makeover's dark-mode phase later consumes D8's structure).
- **Tenancy Phase 8 (CloudFront/DNS per tenant)** remains the ops half of onboarding — unchanged by this doc.
- **CommunityFinder:** unaffected (static branding, no `/api/site_info`).

---

# Open Questions

- **OQ-T1 — Danger tone vs brand red.** Knotty Yoga's brand red doubles as the danger color today. Keep them one token (simpler, but a green-brand studio gets green "Delete" buttons) or split `--theme-primary` from `--theme-danger` with independent defaults? *(Recommendation: split — the catalog above already does; KY just sets both to red-ish values.)*
	- Mason- Let's split them.
- **OQ-T2 — Curated font list contents.** Which open-licensed families ship beside D-DIN? *(Recommendation: 3–4 with distinct personalities — e.g. Inter, Source Serif, Montserrat, Oswald — self-hosted; final pick with Ryan since he'll preview them in Figma.)*
	- Mason- I need to ask Ryan.
- **OQ-T3 — Getting Started step copy: per-step keys or accept the shipped copy as universal?** The seven steps read fairly studio-neutral already ("Try an intro workshop", "Pick your membership"…). Per-step keys are 14 slots of admin-form surface for copy most tenants may never touch. *(Recommendation: ship the keys anyway — the defaults machinery makes them free until someone edits, and "intro workshop" is exactly the phrase a pure-yoga studio would want to change.)*
	- Mason- Those all seems to have an icon, headline text, a longer text description, and a link. It feels like these could be rows in a database and we could have a page to edit this (with an ordinal number). The only issue is the icon but we could probably have an icon picker
- **OQ-T4 — Social links: typed per-network fields or the freeform `label|url` lines?** *(Recommendation: freeform lines v1; the footer maps known labels to icons and falls back to a link — no schema churn when a studio wants TikTok.)*
- **OQ-T5 — Where does the slot registry live, honuware or app?** The key *names* and validation are framework-ready (any consumer app wants them), but the slot *set* references app pages (Getting Started steps). *(Recommendation: keys + validation + site_info/manage endpoints in honuware; the app contributes its slot defaults exactly as `app_secret_values.cpp` does today — same split the secrets system already uses.)*
