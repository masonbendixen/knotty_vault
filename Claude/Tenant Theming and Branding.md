---
fileClass: Reference
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/4/2026
Version: 0.4
tags: 
---

# Tenant Theming and Branding

# Overview

This is the plan for making the frontend **whitelabel-ready**: a new studio signs up as a tenant and runs the site *as is* — their own name, logo, colors, fonts, photos, and marketing copy — with zero code changes and zero layout changes. Mason requested this 8/4/2026 (in [[Component Inventory for Designer]], "Mason - Updates to the document 8/4/2026") and confirmed it should be its own document (OQ-D2 there).

This doc **fuses and supersedes** the theming sketches scattered across three other plans:

- **[[Website Makeover]] Phase 5** ("Database-driven per-tenant theming") — superseded by this doc. The makeover keeps Phases 1–4 (Ryan's tokens → CSS variables → component sweep) and 6; when its Phase 5 comes up in sequence, execute *this* doc instead.
- **[[Converting the server to a multi tenant Saas architecture]] Phase 7** — shipped 7/23/2026; its as-built hooks (`/api/site_info`, `SiteConfigService`) are this doc's foundation, and its 7.4 note ("align `site_info` so it can carry or coexist with theme tokens rather than duplicating them") is honored by decision D2 below.
- **[[Componentizing the frontend]]** styling rules — the `@honuware/ui` library is CSS-framework-agnostic, bans hardcoded colors, and never ships a brand. *(v0.1 said "nothing here touches the library" — revised in v0.2: CommunityFinder will consume runtime branding, so the frontend machinery reaches the library on its clock. See D11 and OQ-T7.)*

The designer-facing view of this work (the 🎨 markers, the five design rules, the fake-second-studio proof frame) lives in [[Component Inventory for Designer]]; this doc is the engineering side. Planning here is iterative — answer open questions inline, keep the Overview intact, and tick checkboxes as phases land.

> **v0.2 (8/4/2026):** Mason answered OQ-T1/T3/T4/T5 (OQ-T2 pending Ryan) and dropped the plan's biggest assumption: **CommunityFinder will be a branded, multi-site consumer** (`gay.seattle.beyondthefreeze.com`, then `lesbian.seattle.…`, `gay.portland.…`). Consequences threaded through: theming machinery is **framework surface** (new D11), the Getting Started steps became **database rows with an editor** instead of copy keys (D6 rewritten, new Phase 3), phases renumbered 1–8 with `[hw] / [hw-lib] / [app]` placement tags, and two new honuware questions opened (OQ-T6 site granularity, OQ-T7 frontend-lift timing).

> **v0.3 (8/4/2026):** OQ-T6 and OQ-T7 resolved — **every architecture question is now closed** and the plan is execution-ready up to Phase 7. OQ-T6: **tenant per subdomain**, each beyondthefreeze site its own database — the stack as shipped, **zero new honuware work** (Mason's cross-city affinity observation recorded as a future `--copy-theme-from` provisioning nicety in Phase 7). OQ-T7: **Knotty Yoga first**, the `@honuware/ui` lift stays Phase 8 on CommunityFinder's clock. The one open item is OQ-T2 (font list) — non-blocking, agenda'd for the Ryan working session later today.

> **v0.4 (8/4/2026, from the Ryan session):** OQ-T2 resolved by **dissolving it** — there is no curated font list. Ryan: Google fonts are all open-licensed but ubiquitous, so a short list can't make studios distinct. Mason's counter-proposal adopted: **fonts become per-tenant data** — a font entry is either a reference to a trusted CDN family (allow-listed origins, starting with Google Fonts) or an **uploaded font file stored in the tenant's database** (the photo-storage precedent). D4 and D10 rewritten, Phase 4 gains the fonts-as-data cluster, Phase 6's admin section becomes a font manager. The token layer is untouched — `--font-*` roles still map to families; only where families *come from* changed. **Every open question is now closed.**

---

# Goal & Non-Goals

**Goal.** One built Angular bundle + one server deployment serve N studios. Everything brand-specific arrives at runtime from the tenant's own data: token values, copy, imagery. "Onboard a studio" becomes: create the tenant (`--create_tenant`), fill in the Site Theme form, upload photos. No forks, no per-tenant builds, no CSS edits.

**Non-goals (explicitly out of scope).**
- **Per-tenant layout or feature differences.** Every tenant gets the same pages, routes, components, and features. A studio that wants a different *shape* of site is a different app, not a theme.
- **Per-tenant custom pages / arbitrary HTML injection.** Content slots are typed and enumerated; there is no "paste your own page" escape hatch (XSS surface, support burden).
- **A theme marketplace / multiple switchable themes per tenant.** One theme per tenant, editable.
- **Speculative `@honuware/ui` surface growth.** The library stays brand-free and token-consuming. CommunityFinder *will* consume runtime branding (v0.2 revision — see D11), so the frontend theming machinery does eventually reach the library — but on the second-consumer clock (Phase 8, OQ-T7), never ahead of a consumer.
- **CommunityFinder's own adoption plan.** Its bootstrap doc (other vault) owns when and how it consumes this machinery — this doc only guarantees the honuware surface it needs, and hands over the OQ-T6 granularity decision when that project resumes.

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
- **D4 — Fonts are per-tenant data, not a curated list** *(rewritten 8/4/2026 from the Ryan session — supersedes v0.1's curated-set design)*. Ryan's point: every Google font is open-licensed, but ubiquity means a short list can't make studios distinct; and real studios arrive with brand fonts (Knotty Yoga's own D-DIN is the proof — no curated list would have contained it). So the font *inventory* is data — a per-tenant `site_fonts` table where each family is one of two source kinds:
	- **CDN reference** — stored as **family name + weights, never a free-form URL**; the client constructs the stylesheet URL from an **allow-listed origin** template (Google Fonts first; adding a second trusted CDN later is a one-constant change). Free-form URLs are rejected: they'd put arbitrary third-party hosts in tenants' pages and make a strict CSP impossible to enumerate. Recorded caveat, not blocker: CDN mode discloses visitor IPs to the CDN (the German-court GDPR finding) — privacy-sensitive tenants use the upload path.
	- **Uploaded file** — binary faces stored in the tenant's database (the photo-storage precedent), validated by **magic bytes** (`wOF2`/`wOFF`/TTF/OTF signatures) + a per-face size cap, served by a public cacheable endpoint with the correct `font/*` MIME + `nosniff` (uploaded bytes must never be reinterpretable as script from our origin; browsers additionally run webfonts through the OpenType Sanitizer). **License responsibility rides with the uploader** — same contract as any user-uploaded content, noted in the admin UI.
	- The **token layer is untouched**: `--font-display` / `--font-heading` / `--font-body` role mapping stays in the `site_theme_*` keys; the table only changes where *families* come from. Every role keeps a system-font fallback stack with `font-display: swap` — a dead CDN or deleted upload degrades to readable text, never blank. D-DIN stays bundled as Knotty Yoga's default values; the `.din*` classes still become thin wrappers over the `--font-*` variables. Honest cost note: this is *more* mechanism than the curated list (table, upload path, serving endpoint, font manager UI) — accepted because it buys the actual whitelabel promise and permanently deletes the curation decision.
- **D5 — Long copy is markdown.** `about` (and any future long slot) is a markdown string rendered with the blog's ngx-markdown pipeline and prose styles — sanitization on, one rendering path, one set of text styles. The About page becomes the second consumer of the blog's prose system.
- **D6 — Structured content is data, not copy keys** *(rewritten 8/4/2026 per Mason's OQ-T3 answer)*. The Getting Started steps become **per-tenant database rows**, not fourteen `config_secrets` keys: a `getting_started_steps` table (ordinal, Material icon name, title, body, link route + label, `hidden_when_logged_in`) with a dedicated editor page and a **curated icon picker**. v0.1's "no add/remove — the flow is the product" stance dissolves with the move to rows: an ordered list of rows has no fixed arity, so tenants can reword, reorder, add, and remove. The page's behaviors survive as data: numbering renders from *position* (so it still closes up when a row is hidden), the signed-in "Create your account" drop becomes the `hidden_when_logged_in` flag (seeded true on that row). Same precedent as `home_page_photos`: **structured per-tenant content = a table; prose = a slot.** Full design in Phase 3.
- **D11 — Framework-first placement** *(new 8/4/2026 — driven by Mason's CommunityFinder note in Sequencing)*. CommunityFinder will be a *branded, multi-site* consumer, so the theming machinery is framework surface, not app convenience: the secret keys, validation, the extended `/api/site_info`, and the Site Theme read/write endpoints all land in **honuware** (per OQ-T5 ✅), with each app contributing its slot *defaults* exactly as `app_secret_values.cpp` does today — the same split the secrets system already uses. The **frontend** half lifts into `@honuware/ui` on CommunityFinder's clock (Phase 8, OQ-T7). This supersedes the componentization plan's Q12 deferral ("no library surface needs branding") and the static-branding assumption in the tenancy plan's §1.9/§1.10 — update those docs' CF notes next time they're touched.
- **D7 — Imagery v1 is URL + existing photo plumbing, no new upload surface.** `site_logo_url` already exists; add `site_hero_image_url` and `site_favicon_url` the same way. Carousel photos already upload through `home_page_photos`. A dedicated `site_assets` upload flow is a later nicety, not v1 (no-premature-code).
- **D8 — Dark mode: structure now, pairs later.** Tokens are defined so a `-dark` variant *can* exist per key, but v1 ships a single (light) value set. The makeover's dark-mode phase consumes the token structure when it lands.
- **D9 — Material bridge via system variables.** Where Material components must follow tenant color, override Material's CSS custom properties (`--mat-*` system tokens) from the same boot application rather than compiling per-tenant SCSS themes. The full Material-theme realignment stays Makeover 2.3; this doc only wires the bridge for the tenant-variable values.
- **D10 — Validation is server-side and strict.** Colors must parse as `#RRGGBB`; URLs must be http(s) or root-relative; markdown capped (e.g. 64 KB); font role selections must name a family that exists for the tenant (bundled default or a `site_fonts` row); CDN font entries must resolve against the origin allow-list (family + weights, no free-form URLs); uploaded faces must pass magic-byte + size checks (D4). A junk value falls back to the default rather than breaking every page of a tenant's site (same philosophy as the blog's out-of-range month).

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

The three `--font-*` rows hold a **family name**, and the family inventory is per-tenant data (`site_fonts`, D4 — CDN reference or uploaded file); the bundled D-DIN stacks are Knotty Yoga's defaults and the fallback when a named family is missing.

The exact status-tone hex values in use get pinned during Phase 4 grounding (they're scattered across badge SCSS today; the makeover's badge consolidation is where they become one set).

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
| ~~Getting Started steps 1–7~~ | *moved to **data** — the `getting_started_steps` table (D6, Phase 3), not slots* | rows | `/start` step cards | the seven shipped steps become seed rows |
| Membership blurb | `site_membership_blurb` | line | home Memberships section | "Unlimited classes, priority sign-ups, and the whole community — pick the tier that fits how you train." |
| Social links | `site_social_links` | lines (`label|url`) | footer icon row | current links |

Not slots (already per-tenant data, no work): carousel photos (`home_page_photos`), blog posts, class/product/instructor content, membership tier names (they're `products` rows).

---

# Phased Implementation Plan

Conventions per [[../CLAUDE.md|CLAUDE.md]] + standing memory: backend before frontend inside every phase; every item lands with its tests in the same session; Linux docker is the C++ gate, `ng test`/`ng build`/`ng lint` the Angular gate; live hand-testing steps close each phase. Placement tags per D11: **[hw]** = honuware server components (needs a pin bump), **[hw-lib]** = `@honuware/ui`, **[app]** = knottyyoga. Phases 1–3 have no dependency on Ryan or the makeover and can start immediately; Phase 4 defines the CSS variables itself if Makeover 2.1 hasn't landed first (they're the same variables — first mover creates, second consumes); Phase 8 waits for CommunityFinder.

## Phase 1 — Content slots, server side

- [ ] Register the new content-slot secret keys (brand-free *names*) in honuware's `secret_keys.h` + `FillInSecretsStringView`, with empty/neutral framework defaults — mirroring how `kSiteLogoUrl` landed. *(hw — needs a pin bump)*
- [ ] Register Knotty Yoga default *values* app-side in `business_logic/app_secret_values.cpp` (the table above's right-hand column). *(app)*
- [ ] Extend `GET /api/site_info` with the `content` object (slot key → resolved value, defaults filled) and the `theme` object (empty until Phase 4 — present so the payload shape is stable). Keep the endpoint public + cacheable; keep the pure builder split. *(hw)*
- [ ] Server-side validation helpers per D10 (hex color, URL shape, size caps) — shared by site_info's read path (defensive normalize) and Phase 6's write path. *(hw)*
- [ ] Tests: builder field-mapping for `content`/`theme`; defaults fill when unset; validation accepts/rejects the documented shapes; app suite green proving the new defaults seed on a fresh DB.

## Phase 2 — Content slots, frontend

- [ ] `SiteConfig` grows the slot fields; `load()` merges `content` over the existing static defaults (same non-empty-wins rule). The current hardcoded values stay as the fallback constants — a dead API never blanks the site.
- [ ] De-hardcode the consumers: home hero headline/subline/secondary image, membership blurb, Getting Started intro + step copy (titles/bodies from config, CTAs/routes/icons from the component), `document.title`, favicon swap, logo alt.
- [ ] `/about` renders `site_about_markdown` through ngx-markdown with the blog's prose styles (route-scoped `provideMarkdown()`, same as `/blog`).
- [ ] Specs per consumer (the standing component-spec rule) + `SiteConfigService` merge cases + `ServerAccess.mock` returns the dev content block (+ mock spec).

## Phase 3 — Getting Started steps as data (D6)

Backend first, then the page rewire, then the editor.

- [ ] `db_schema/getting_started_steps.{h,cpp}` — **app-side** table, the `home_page_photos` precedent: `id`, `ordinal`, `mat_icon`, `title`, `body`, `link_route`, `link_label`, `hidden_when_logged_in`, timestamps. Registered through the full new-table checklist (`make_app_tables.cpp` + create_database's 11 registration points — admin registration included, so the generic editor doubles as the debug view). App-side because knottyyoga is the only app with a Getting Started page today; the table is a leaf and promotes to honuware cheaply if CommunityFinder ever builds one. *(app)*
- [ ] Curated **icon allow-list** constant (Material icon names) — one source of truth serving both server-side `mat_icon` validation and the editor's picker grid. *(app)*
- [ ] Table helper + `GET /api/getting_started_steps` (anonymous, ordered by `ordinal`) — the public page's feed. Writes ride the generic CRUD endpoints via the admin table registration (admin permission). Route values validated to be root-relative (`/...`). *(app)*
- [ ] Seed the seven shipped steps in `create_database.cpp` — the "Create your account" row seeded `hidden_when_logged_in = true`. *(app)*
- [ ] Frontend: `/start` renders from the endpoint — numbering from array position (the signed-in close-up behavior falls out for free), icon/title/body/CTA from the row. `ServerAccess` seam ×5 files per the standing rule. Component + mock specs.
- [ ] Frontend: dedicated **Manage → Getting Started** editor page (per the "Manage Data is a debug hack" rule — this is the business workflow surface): ordered row list with reorder, add/remove, the icon-picker grid over the curated set, title/body/route/label fields, the hidden-when-signed-in toggle. Specs.
- [ ] Tests: table helper CRUD + ordering; endpoint anonymity + order; icon + route validation; seed presence on a fresh DB.

## Phase 4 — Token pipeline (incl. fonts as data — D4)

Colors/radius first (pure variable plumbing), then the font machinery.

- [ ] Server: accept + serve `site_theme_*` keys in the `theme` object (CSS-var name → validated value). *(hw)*
- [ ] Frontend: boot application — write each `theme` entry onto `document.documentElement`; alias the legacy `--red`/`--orange`/`--gray` to the new `--theme-*` so existing Tailwind mappings restyle immediately.
- [ ] **`site_fonts` table** (framework tenant-DB table per D11 — CommunityFinder wants fonts too): family name, face rows (weight, style), source kind `cdn`/`uploaded`, CDN family + weights for `cdn`, binary bytes + format for `uploaded`. Table helper + magic-byte/size validation + the CDN origin allow-list constant (Google Fonts first). *(hw)*
- [ ] **Font serving endpoint** — public, cacheable (long max-age; CloudFront caches it like photos), correct `font/*` Content-Type + `nosniff`. *(hw)*
- [ ] `site_info`'s `theme` object gains **font-face descriptors** per tenant family: `cdn` entries as family+weights (client constructs the allow-listed stylesheet URL), `uploaded` entries as family/weight/style/format + the serving URL. *(hw)*
- [ ] Frontend boot: inject the constructed CDN `<link>`s and generated `@font-face` rules in the same pre-render initializer; set the `--font-*` role variables; every role carries a system-font fallback stack with `font-display: swap` (a dead CDN or deleted upload degrades to readable text, never blank). *(app)*
- [ ] `.din*` font classes re-based onto `--font-*` variables; the bundled D-DIN faces stay as Knotty Yoga's default role values (no `site_fonts` rows needed for KY). *(app)*
- [ ] Material bridge: override the `--mat-*` system tokens that map to primary/accent at boot (D9) — scoped to what's visibly brand-colored today (buttons, toggles, spinner), not a full re-theme.
- [ ] Reconcile token names with Ryan's Foundations file when it lands (his names win; update the catalog above).
- [ ] Tests: token application function (writes vars, skips invalid); a themed boot restyles a `theme-red` consumer; font-class regression spec; `site_fonts` helper CRUD; upload validation accepts woff2/woff/ttf/otf magic bytes and rejects junk + oversize; serving endpoint MIME/nosniff/cache headers; CDN descriptor round-trip; URL-construction unit test (allow-listed origin only).

## Phase 5 — Imagery v1 (URL-based)

- [ ] Wire `site_hero_image_url` + `site_favicon_url` consumers (Phase 2 stubs them behind defaults; this phase completes fallback behavior + error handling — a 404'd image falls back to the bundled asset, never a broken-image icon).
- [ ] Document the constraint set for each slot (logo: light-on-dark, height-bounded; hero image: aspect guidance) in the onboarding runbook — these match Ryan's slot-constraint rules in the inventory doc.
- [ ] *(Deferred, recorded not planned: a `site_assets` upload flow so tenants don't need externally hosted URLs — revisit after the first real second tenant.)*

## Phase 6 — Admin "Site Theme" page

- [ ] Backend: curated endpoints `GET /api/manage/site_theme` (full editable set, unset-vs-default distinguished) + `PUT /api/manage/site_theme` (validated writes to the tenant's `config_secrets`). Admin-only via the existing permission machinery. **Never** the generic CRUD surface — `config_secrets` stays excluded. *(hw — resolved by OQ-T5/D11; the app contributes its slot list the same way it contributes defaults)*
- [ ] Frontend: Manage-portal page (back office — inherits building blocks, no Ryan design) with sections **Brand basics** (name, title, logo/favicon URLs, contact, address, socials) / **Copy** (hero, tagline, blurbs — the Getting Started steps have their own editor, Phase 3) / **About** (markdown textarea + live prose preview, reusing the blog editor's split-pane pattern) / **Colors** (color pickers per token role) / **Fonts** (the D4 **font manager**: list the tenant's families; add one as either a CDN family + weights from the allow-listed origin, or a file upload with the license-responsibility note; per-face preview text; assign families to the display/heading/body roles) — with per-field "reset to default". *(app for now; lift candidate in Phase 8)*
- [ ] "Changes appear within ~5 minutes" note (the site_info cache) + a save-confirmation.
- [ ] Tests: endpoint auth/validation/round-trip; component specs per section.

## Phase 7 — Provisioning, runbook, and the fake-studio proof

- [ ] `--create_tenant` verification: a fresh tenant boots with every slot on defaults (this mostly falls out of the defaults machinery — the test proves it) and the seven seeded Getting Started rows present.
- [ ] Onboarding runbook: the checklist a new studio walks (Site Theme form top to bottom, Getting Started editor, photo uploads, DNS/CloudFront per the tenancy plan's Phase 8).
- [ ] **The fake-studio smoke test** — engineering twin of Ryan's OQ-D3 proof frame: a second local tenant with an invented brand (different palette, wide logo, two-line name, different hero copy); walk Home / Our Classes / Blog / Getting Started / About / an email; nothing Knotty-branded may leak. This list = the regression checklist for every future theming change.
- [ ] Live hand-testing steps for the whole feature (blank DB, two tenants, exact form fields + values, per the precise-instructions rule).
- [ ] Hand-off: copy the OQ-T6 decision (**tenant per subdomain** — each beyondthefreeze site its own database, standard onboarding runbook per site) + the honuware surface list into CommunityFinder's bootstrap doc (other vault) when that project resumes.
- [ ] *(Recorded, not planned — from Mason's OQ-T6 affinity note: a `--copy-theme-from <tenant>` provisioning option that clones the `site_*` content slots + theme tokens from a related tenant at create time — gay.portland starting from gay.seattle's look, data fully isolated. Revisit when the second related tenant actually onboards.)*

## Phase 8 — `@honuware/ui` lift (on CommunityFinder's clock — OQ-T7 ✅)

> Deliberately last and deliberately gated — **confirmed by Mason (OQ-T7): "get this working in Knotty Yoga first, then move to community finder."** The lift happens when CommunityFinder's frontend actually bootstraps, not speculatively. Until then knottyyoga's app-side implementation is the reference implementation.

- [ ] Lift the site-config surface into `@honuware/ui`: a `SiteAccess` seam beside `CrudAccess`/`AuthAccess`/`PhotoAccess` (`getSiteInfo()`), the config service (merge + fallback contract), and the token boot-applier. This *is* the branding surface the componentization plan's Q12 reserved `HONUWARE_BRANDING` for — the service supersedes the token idea. knottyyoga's `SiteConfigService` becomes the app-side provider/consumer; its specs move with it; brand-free library assertions stay. *(hw-lib)*
- [ ] Evaluate lifting the Site Theme manage page + the Getting Started steps table/editor once CommunityFinder has the surfaces to mount them — recorded, not planned (no-premature-code). *(hw-lib, speculative)*

---

# Sequencing with the other plans

- **Independent now:** Phases 1–3 (content slots + the steps table) touch nothing Ryan or the makeover owns.
- **Coordinates with Makeover Phase 2:** Phase 4 shares the CSS-variable layer. Either lands first; whoever is second consumes the other's variables. Ryan's Figma token *names* (Makeover 1.1.A/4.1) get reconciled into the catalog when they exist — they gate nothing.
- **Supersedes Makeover Phase 5** (noted there when the makeover doc next gets touched; the makeover's dark-mode phase later consumes D8's structure).
- **Tenancy Phase 8 (CloudFront/DNS per tenant)** remains the ops half of onboarding — unchanged by this doc, and it is exactly the wiring each `beyondthefreeze.com` subdomain will reuse.
- **CommunityFinder:** ~~unaffected (static branding, no `/api/site_info`)~~ — **revised 8/4/2026: a driving consumer.** *(Both follow-on questions now resolved: OQ-T6 → tenant per subdomain, no new honuware mode; OQ-T7 → Knotty Yoga first, Phase 8 lift on CF's clock.)*
	- Mason- I very much want community finder to be branded. I intend to start with gay.seattle.beyondthefreeze.com but branch out to lesbian.seattle.beyondthefreeze.com and gay.portland.beyondthefreeze.com.
	- Claude- ✅ Plan updated throughout. What this changes: (1) the server-side theming surface is **framework** (new D11 — keys, validation, extended `site_info`, manage endpoints all in honuware), so CommunityFinder gets the whole stack by consuming honuware: per-subdomain CloudFront distribution → `X-Honuware-Site` header → tenant resolution → that site's theme, exactly the tenancy plan's Phase 8 wiring; (2) the frontend machinery lifts into `@honuware/ui` when CommunityFinder's frontend bootstraps (new Phase 8 here, OQ-T7); (3) this supersedes the static-branding notes in the tenancy plan (§1.9 #2/#3, §1.10, its 7.4 note) and the componentization plan's Q12 deferral — those docs get their CF lines updated next time they're touched, and CommunityFinder's own bootstrap doc inherits the OQ-T6 decision (Phase 7 hand-off item). One architecture question falls out of the subdomain list — **OQ-T6 below**: whether each subdomain is its own tenant (own database) or several subdomains share one community's data and differ only in skin. The stack as shipped is strictly one site key → one tenant DB, so the answer decides whether honuware needs a new mode.

---

# Open Questions

- **OQ-T1 — Danger tone vs brand red.** ✅ **RESOLVED — split.** *(Mason: "Let's split them.")* `--theme-primary` and `--theme-danger` are independent tokens; Knotty Yoga just sets both to red-ish values. The catalog above stands as written.
- **OQ-T2 — Curated font list contents.** ✅ **RESOLVED (Ryan session, 8/4/2026) — by dissolving the question: there is no curated list; fonts are per-tenant data.**
	- Mason- I need to ask Ryan.
	- Mason (with Ryan, 8/4)- Ryan: all of the Google fonts are open source and free to use — the issue is they're widely used, so they aren't particularly unique. Proposal: delay the list decision entirely by putting fonts in the database — a tenant either references a known internet font at a trusted CDN location (like Google) or uploads a font that's stored in the database and referenced from there.
	- Claude- ✅ **Adopted — it's the better product design, with guardrails.** Ryan's objection lands on the curated list's real weakness: a short list can't make studios distinct, and real studios arrive with brand fonts (D-DIN itself would never have been on anyone's curated list). Fonts-as-data covers both ends — CDN breadth, upload uniqueness — and *deletes* the curation decision instead of deferring it. The guardrails that make it sound: **(1)** CDN entries are stored as **family + weights, never free-form URLs** — the client constructs the stylesheet URL from an allow-listed origin (Google Fonts first), keeping a strict CSP enumerable and third-party hosts out of tenant pages; **(2)** uploads are validated by **magic bytes + size cap** and served with correct `font/*` MIME + `nosniff` so uploaded bytes can never come back as script; **(3)** **license responsibility rides with the uploader**, noted in the UI — the same contract as every user-uploaded asset; **(4)** the token layer is untouched (`--font-*` roles still map to families) and every role keeps a system-font fallback with `font-display: swap`; **(5)** one recorded caveat: CDN mode discloses visitor IPs to the CDN — privacy-sensitive tenants use the upload path. Honest cost: more mechanism than the list (table, upload path, serving endpoint, font manager) — worth it. Rewritten into **D4/D10**, planned in **Phase 4** (`site_fonts` as a framework tenant-DB table per D11 — CommunityFinder gets it too) and **Phase 6** (the font manager section). The two remaining Ryan-session items stand on their own below.
	- **Still for Ryan (same session, unchanged):** (1) confirm/overrule the Token Catalog v1 names against his Figma Variables — his names win, write them inline above; (2) the fake-studio proof frame (OQ-D3 in [[Component Inventory for Designer]]) should use a distinct (uploaded-style) font *and* a non-red primary so it exercises both new mechanisms.
- **OQ-T3 — Getting Started step copy.** ✅ **RESOLVED — database rows with an editor, not copy keys.**
	- Mason- Those all seems to have an icon, headline text, a longer text description, and a link. It feels like these could be rows in a database and we could have a page to edit this (with an ordinal number). The only issue is the icon but we could probably have an icon picker and a group of standard icons. What do you think?
	- Claude- ✅ Agreed — rows are the better design, and the icon is not really an obstacle. Your four fields are exactly the anatomy (`mat_icon`, `title`, `body`, `link_route` + `link_label`), plus `ordinal` and one behavior flag: `hidden_when_logged_in`, which is how the "Create your account" step keeps disappearing for signed-in users with the numbering closing up (numbering renders from row *position*, so that behavior is free). The icon picker is a **curated allow-list of Material icon names** — one constant serves both server-side validation and the editor's picker grid, and since Material icons are already the app's icon system there is nothing to upload or license. This also quietly improves on the keys design: steps become add/remove/reorder-able per tenant, and it matches the existing precedent (`home_page_photos` — structured per-tenant content is a table; prose is a slot). Rewritten as **D6**, planned as **Phase 3** (backend table + seed + anonymous feed endpoint, then the `/start` rewire, then a dedicated **Manage → Getting Started** editor — dedicated page per the "Manage Data is debug-only" rule). One placement note: the table lands **app-side** like `home_page_photos` — the Getting Started *page* is a knottyyoga surface today, and a leaf table promotes to honuware cheaply if CommunityFinder ever builds a start page (Phase 8 records that as a lift candidate).
- **OQ-T4 — Social links format.** ✅ **RESOLVED — freeform `label|url` lines.** *(Mason: "I'll go with your recommendation.")* The footer maps known labels to icons and falls back to a plain link.
- **OQ-T5 — Where does the slot registry live?** ✅ **RESOLVED — honuware.** *(Mason: "Given that I want this in community finder … I think that this needs to be in honuware.")* Keys + validation + `site_info`/manage endpoints in honuware; each app contributes its slot defaults exactly as `app_secret_values.cpp` does today. Promoted into **D11** together with your CommunityFinder note — the framework-first placement is now a design decision, not just a storage answer.

## New (8/4/2026, from the CommunityFinder revelation — the honuware questions you asked for)

- **OQ-T6 — CommunityFinder site granularity: is each subdomain its own tenant, or a skin?** `gay.seattle` / `lesbian.seattle` / `gay.portland` — two readings, and they diverge hard at the honuware layer:
	- **(a) Tenant per subdomain.** Each site key resolves to its **own database** — separate members, events, groups, content, theme. This is the stack exactly as shipped: per-subdomain CloudFront distribution injecting `X-Honuware-Site`, a control-DB row each, `--create_tenant` each. **Zero new honuware work.**
	- **(b) Shared data, branded doors.** e.g. one Seattle community database that both `gay.seattle` and `lesbian.seattle` serve, differing only in branding/theme (and perhaps content filtering). Today's architecture **cannot** express this: site key → tenant is strictly 1:1 and branding lives *inside* the tenant's own DB. It would need a new honuware mode — several site keys resolving to one tenant DB with a per-site-key branding overlay (branding keyed by *site*, data keyed by *tenant*) — a real architecture change touching the resolver, the secrets model, and `site_info`.
	- *(Recommendation: **(a)** tenant-per-subdomain. It is free today, community member lists stay isolated — a lesbian.seattle member should not implicitly exist in gay.seattle's data — and if cross-community sharing is ever wanted, that reads like an app-level cross-posting/federation feature between tenants, not a tenancy mode. If your product vision for CommunityFinder is genuinely "one city community, several branded doors over the same people and events," say so and (b) gets designed properly here before CommunityFinder bootstraps — retrofitting it after tenants exist would be painful.)*
	- Mason- I think that gay.seattle and lesbian.seattle should be entirely different tenants with separate databases. Honestly, there might be more in common between gay.seattle and gay.portland versus lesbian.seattle but they are all different tenants.
	- Claude- ✅ **RESOLVED — (a), tenant per subdomain.** No new honuware mode; the stack as shipped covers CommunityFinder's whole roadmap, and onboarding a subdomain = the standard runbook (control row + `--create_tenant` + CloudFront distribution). Your affinity observation — gay.seattle sharing more with gay.portland than with lesbian.seattle — fits cleanly *inside* (a): what related tenants share is **theme and copy**, not data, which is a provisioning convenience rather than an architecture feature. Recorded in Phase 7 as a future `--copy-theme-from <tenant>` nicety (clone the `site_*` slots and theme tokens at create time; everything else stays isolated). Decision flows to CommunityFinder's bootstrap doc via the existing Phase 7 hand-off item.
- **OQ-T7 — When does the frontend theming machinery move into `@honuware/ui`?** The server side is honuware (OQ-T5 ✅). The web pieces — the `SiteAccess` fetch seam, the config service, the token boot-applier, eventually the Site Theme page — could be **(a)** built app-side in knottyyoga now and lifted into the library when CommunityFinder's frontend bootstraps (this plan's Phase 8), or **(b)** built in the library from day one. *(Recommendation: **(a)**. Knotty Yoga iterates without a publish/version-bump cycle while the surface is still settling; the library boundary rule has always been second-consumer-driven, and CommunityFinder's frontend start date is unknown; the lift is bounded and the extraction pattern is well-practiced. Honest downside: if CommunityFinder's frontend starts before the lift, there is briefly a second implementation — if you know its frontend work starts soon, flip to (b) and I'll fold the library work into Phases 2/4 instead.)*
	- Mason- I'd like to get this working in Knotty Yoga first then move to community finder.
	- Claude- ✅ **RESOLVED — (a), Knotty Yoga first.** Phases 1–7 build and prove everything app-side in knottyyoga; Phase 8 lifts the settled surface into `@honuware/ui` when CommunityFinder's frontend bootstraps. The phase structure already matched this answer, so no plan change — Phase 8's gate note now cites your decision.
