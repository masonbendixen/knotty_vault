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
- `GET /api/site_info` (honuware `components/platform/endpoints/site_info.*`) returns `display_name`, `website_url`, `logo_url` for the resolved tenant. Unauthenticated, `Cache-Control: no-cache` + `Vary: X-Honuware-Site` (was `public, max-age=300` until 8/14/2026). Sources: `::Mail::LoadTenantBranding` (studio name ← `kMailSenderName`, website ← `kWebsiteAddressLogin`) + `Secrets::kSiteLogoUrl` (default `""`), falling back to `TenantContext.displayName`.
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

The contract between Ryan's Figma Variables (Makeover 1.1.A / 4.1) and the code. **As-built 8/11/2026** — the table below is no longer a proposal: it is exactly what `ui/src/assets/styles/_tokens.scss` emits under `:root`, written during [[Component Inventory for Designer]] integration Track A. Values are Knotty Yoga's defaults; Phase 4 overrides the same custom-property names per tenant at boot.

| Role | CSS variable | KY default | Notes |
|---|---|---|---|
| Primary / brand | `--theme-primary` | `#ED1C26` | Ryan's brand red — replaced the old `#F50C22`. Legacy `--red` aliases onto it, so `theme-red` Tailwind usages already restyled |
| On-primary | `--theme-on-primary` | `#FFFFFF` | Text/icon color on a primary fill |
| Secondary / accent | `--theme-accent` | `var(--theme-primary)` | Role retained for other tenants; **retired for KY** (the orange does not survive) |
| On-accent | `--theme-on-accent` | `var(--theme-on-primary)` | 〃 |
| Neutral text | `--theme-neutral` | `#666666` | Aliases `--gray`. Ryan's replacement grey still pending — swap the value when it lands |
| Ink / black | `--theme-ink` | `#000000` | Aliases `--black` |
| Surface / white | `--theme-surface` | `#FFFFFF` | Aliases `--white` |
| Surface tint | `--theme-surface-tint` | `#F3F3F3` | Ryan's subtle fill (table headers, inactive chips); also backs the neutral badge tone |
| Page background | `--theme-background` | `#FFFFFF` | Distinct from surface so cards can sit on a tinted page |
| Border | `--theme-border` | `#D1D5DB` | The card/table border duplicated across 56+ component SCSS files — the token Makeover 2.5's `.surface-card` consumes |
| Success tone | `--theme-success` / `--theme-on-success` | `#BAF7CD` / `#000000` | Status *semantics* fixed; tenants tune the shade only |
| Warn tone | `--theme-warn` / `--theme-on-warn` | `#FFE4E5` / `#B4191D` | 〃 |
| Danger tone | `--theme-danger` / `--theme-on-danger` | `#ED1C26` / `#FFFFFF` | Separate role from primary (OQ-T1); KY sets both red |
| Info tone | `--theme-info` / `--theme-on-info` | `#7B7B7B` / `#FFFFFF` | 〃 |
| Muted / neutral tone | `--theme-muted` / `--theme-on-muted` | `var(--theme-surface-tint)` / `var(--theme-ink)` | The fifth Badge tone. Named `muted`, not `neutral`, so it never collides with the neutral *text* grey |
| Display font | `--font-display` | `D-DIN Condensed Bold` stack | `.din-condensed-bold` resolves through it (done) |
| Heading/bold font | `--font-heading` | `D-DIN Bold` stack | `.din-bold` → `var(--font-heading)` (done) |
| Body font | `--font-body` | `D-DIN` stack | `.din` + `body`/`*` → `var(--font-body)` (done) |
| Card radius | `--radius-card` | `8px` | The mat-card/post-card radius in use |
| Control radius | `--radius-control` | `4px` | Buttons/inputs/chips |
| Pill radius | `--radius-pill` | `9999px` | The Material button shape (`_mat-button.scss` consumes it) |

**Deliberate exception, carried in code:** `--orange` was *not* aliased onto `--theme-accent`. The accent role now resolves to the brand red, and `--orange`'s only consumer is the footer background — aliasing would turn the footer red ahead of its redesign. It stays a literal `#FF9933` until integration Track C restyles the footer, then the alias and the `.bg-theme-orange` usage get deleted together.

Config-secret key per token: `site_theme_<role>` (e.g. `site_theme_primary`, `site_theme_font_body`, `site_theme_radius_card`). The `theme` object in `/api/site_info` maps CSS-variable name → value, so the frontend applies it without a lookup table.

The three `--font-*` rows hold a **family name**, and the family inventory is per-tenant data (`site_fonts`, D4 — CDN reference or uploaded file); the bundled D-DIN stacks are Knotty Yoga's defaults and the fallback when a named family is missing.

> **History:** the three 8/4/2026 catalog updates mined from Ryan's Figma file (brand red `#ED1C26` replacing `#F50C22`; the accent role retired for KY; the status tones becoming real background/on-tone pairs) are **folded into the table above** and shipped in code on 8/11/2026. Nothing is pending from that list.

> **Superseded 8/11/2026 — the catalog above is now one layer of two.** Ryan's Figma file was read directly via the API and applied the same night. `_tokens.scss` gained a **palette layer** (`--palette-primary-100…700`, plus `secondary` amber, `tertiary` green, `quaternary` magenta, `quinary` cyan, and a warm `grey` ramp) with every role above re-pointed at a palette step. That is the two-layer scheme this doc's D-decisions and the makeover's Q6 both assumed, so per-tenant theming now swaps **palette** values and the roles follow.
>
> Value changes worth knowing when Phase 4 writes the per-tenant plumbing: **danger is magenta**, deliberately decoupled from the brand red; **the accent is amber** (`#f0a202`) after all; the **neutrals are warm greys** (`#edecec`…`#1a1616`) rather than the inherited blue-greys; the type roles are **Roboto** (body) and **Barlow** (headings/buttons/badges) on a 12/14/16/20/24/40 scale; and two new roles — `--theme-inverse-surface` / `--theme-on-inverse-surface` — carry the black header and footer. All of it is asserted in `design-tokens.spec.ts`, so a tenant override that breaks a role fails a test.

**Closed 8/11/2026:** the neutral text grey and the type scale/weights — both came from Ryan's file.

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

## Phase 1 — Content slots, server side ✅ **DONE 8/12/2026**

- [x] Register the new content-slot secret keys (brand-free *names*) in honuware's `secret_keys.h` + `FillInSecretsStringView`, with empty/neutral framework defaults — mirroring how `kSiteLogoUrl` landed. *(hw — needs a pin bump)*
- [x] Register Knotty Yoga default *values* app-side in `business_logic/app_secret_values.cpp` (the table above's right-hand column). *(app)*
- [x] Extend `GET /api/site_info` with the `content` object (slot key → resolved value, defaults filled) and the `theme` object (empty until Phase 4 — present so the payload shape is stable). Keep the endpoint public + cacheable; keep the pure builder split. *(hw)*
- [x] Server-side validation helpers per D10 (hex color, URL shape, size caps) — shared by site_info's read path (defensive normalize) and Phase 6's write path. *(hw)*
- [x] Tests: builder field-mapping for `content`/`theme`; defaults fill when unset; validation accepts/rejects the documented shapes; app suite green proving the new defaults seed on a fresh DB.

### As-built notes (8/12/2026)

**A slot registry, not a loose pile of keys.** The keys landed in honuware's `secret_keys.h`, but the thing that makes them *public* is a new framework module, `components/platform/business_logic/branding/`:

- `site_content_slots.{h,cpp}` — `SiteContentSlots()` is the registry (key + `SlotType`) and the single source of truth for what `/api/site_info` publishes, how each value is validated, and what Phase 6's editor will offer. `LoadSiteContent(secrets, transaction)` reads every slot and returns a `KeyValueTable` with **every** key present, empty when unset — so the SPA's non-empty-wins merge is total and the payload's shape never varies.
- `site_value_validation.{h,cpp}` — D10 in code. `SlotType` = `Line | Lines | Markdown | Url | Color`; `ValidateSlotValue` returns a *reason string* for Phase 6's 400 body, `NormalizeSlotValue` is the read path's repair-or-blank. Byte caps: line 1 KB, lines 4 KB, markdown 64 KB, URL 2 KB. `IsValidSiteUrl` accepts `http(s)://host…` and root-relative `/path`, and rejects `javascript:`, `data:`, bare relative paths, and **protocol-relative `//host/…`** (an off-origin fetch wearing a relative path's clothes). `IsValidHexColor` is strictly `#RRGGBB` — no shorthand, no alpha — because the value is written straight into a CSS custom property.
- A guard test asserts every registered slot is a `site_*` key, so no credential-bearing secret can ever be added to a public, cached, unauthenticated endpoint by accident.

**⚠️ The one plan detail that had to change: framework defaults vs app defaults are mutually exclusive.** The bullet above said "register the keys in `FillInSecretsStringView` with empty framework defaults" *and* "register Knotty Yoga values app-side" — but `config_secrets.name` is **UNIQUE** and the seed inserts the framework set then the app set, so defaulting a key on both sides aborts the seed. The split as built:

- **Framework defaults** (`secret_values.cpp`): `site_favicon_url`, `site_hero_image_url` — the two slots whose neutral default is genuinely framework-owned (`""` == "the SPA keeps its bundled asset", the contract `site_logo_url` established).
- **App defaults** (`app_secret_values.cpp`): the other eleven, all brand *copy* — exactly the `kMailSenderName` precedent, and exactly what D11 describes ("each app contributing its slot defaults").

Three tests pin this so it cannot drift: `EverySlotInTheRegistryHasADefaultFromOneSideOrTheOther` (exactly one side, for every slot), the pre-existing `FrameworkAndAppKeySetsDoNotOverlap`, and a `PopulateFrameworkTables` assertion that each framework default seeds as exactly one row. `components/services/util/secrets/CLAUDE.md` now documents the rule so the next person doesn't "fix" the apparent omission.

**Response shape** (`GET /api/site_info`; headers revised 8/14/2026 to `Cache-Control: no-cache` + `Vary: X-Honuware-Site` — see Phase 6's caching note):

```json
{ "display_name": "…", "website_url": "…", "logo_url": "…",
  "content": { "site_hero_headline": "…", …13 keys, always all of them… },
  "theme":   { } }
```

`theme` ships empty on purpose so Phase 4 fills it without changing the client contract. Values are emitted through `Json::Value`'s map constructor rather than `SqlUtil::KeyValueTableToJson`, because the latter promotes integer-looking strings to numbers and a browser title of "2026" is still text.

**Knotty Yoga's seeded copy** mirrors what the SPA hardcodes today, so Phase 2 can de-hardcode each consumer without changing a rendered page. Two catalog values were reconciled against as-built code: logo alt is `"Knotty Yoga Fitness"` (the shipped `DEFAULT_SITE_CONFIG` value, not the catalog's "Knotty Yoga logo"), and the hero headline is the shipped sentence casing (`"Knotty Yoga is an inclusive, high-level acrobatic fitness studio."`). Non-ASCII characters (the tagline's 💪🤸, the em dashes) are written as explicit `\x` UTF-8 escapes so the bytes survive MSVC's source-charset handling.

**Note for Phase 2 — the hero slots partly overlap `home_sections`.** Since this catalog was written, Home moved its hero and marketing bands to the `home_sections` table (Phase 3's home-sections half landed early). `site_hero_headline` / `site_hero_subline` / `site_hero_image_url` still exist and are still seeded, but Phase 2 should wire `site_hero_headline` to the **intro strip** (`home-intro__text`, currently hardcoded) rather than re-plumbing the hero band, and decide whether `site_hero_subline` has a consumer at all or should be retired from the catalog.

**Gate:** all green.

| Suite | Result | honuware source |
|---|---|---|
| honuware standalone | 1532 / 1532 | — |
| knottyyoga | 4779 / 4779 | co-dev (local tree) |
| knottyyoga | 4779 / 4779 | **pinned SHA** (real GitHub clone) |
| communityfinder | 1553 / 1553 | **pinned SHA** (real GitHub clone) |

**Pin bumped 8/12/2026** — honuware `5dcae2c` ("Phase 1 — Content slots, server side") is pushed and CI-green, and both consumers are re-pinned to it: `knottyyoga_server/CMakeLists.txt` (from `c258a8c`) and `communityfinder_server/CMakeLists.txt` (from `3b1c3dc`, so CF also picks up "Blog support server side"). Both verified against the pinned clone, not the co-dev override.

⚠️ **Never run two of these suites concurrently.** They own different main databases but create the *same* scratch tenant DBs (`test_honuware_tenant_a/b`), so a parallel run produces phantom failures in `TenantPhysicalIsolationTest.*` and `DatabaseUtilTest.MakeAndClearDatabaseBasic`. Run them serially.

## Phase 2 — Content slots, frontend ✅ **DONE 8/13/2026**

- [x] `SiteConfig` grows the slot fields; `load()` merges `content` over the existing static defaults (same non-empty-wins rule). The current hardcoded values stay as the fallback constants — a dead API never blanks the site.
- [x] De-hardcode the consumers: home hero headline/~~subline~~/secondary image, membership blurb, Getting Started intro ~~+ step copy~~, `document.title`, favicon swap, logo alt. *(Two strikeouts explained below — both are catalog drift, not skipped work.)*
- [x] `/about` renders `site_about_markdown` through ngx-markdown with the blog's prose styles (route-scoped `provideMarkdown()`, same as `/blog`).
- [x] Specs per consumer (the standing component-spec rule) + `SiteConfigService` merge cases + `ServerAccess.mock` returns the dev content block (+ mock spec).

### As-built notes (8/13/2026)

**`SiteConfig` is now the whole branding surface.** It grew `browserTitle`, `faviconUrl`/`faviconAsset`, `heroHeadline`, `heroSubline`, `heroImageUrl`/`heroImageAsset`, `membershipBlurb`, `startIntro` and `socialLinks: SocialLink[]`, and the previously "app-static" `logoAltText` / `addressLines` / `contactEmail` / `taglineLines` / `aboutText` became API-driven. `DEFAULT_SITE_CONFIG` still holds the exact strings the bundle shipped with, so the fallback contract is unchanged: **nothing on screen moved for Knotty Yoga.**

Merge rules, all exercised by spec:
- **line** slots — non-empty wins.
- **lines** slots — `parseLines()` splits on `\n`, trims, drops blank rows; a parsed **non-empty** list wins, so an all-whitespace value keeps the default instead of emptying the footer.
- **social links** — `parseSocialLinks()` reads `label|url` (first `|` only, so URLs keep their colons and slashes) and skips any line missing a half.
- **url** slots — stored verbatim, `''` included: that is the "use the bundled asset" contract `logoUrl` established.
- A response with **no `content` object at all** degrades to defaults rather than throwing inside the app initializer.

**D3's document branding landed here**, not in a component: `load()` stamps `document.title` and repoints `<link rel="icon">` (dropping the bundled `type="image/x-icon"` when a tenant URL wins, since it is as likely to be a PNG/SVG). It runs on the failure path too, so the tab title is right even when the fetch dies. Phase 4 adds the CSS-custom-property writes to the same method.

**The blog's prose styles were lifted, not copied.** `.post-body`'s typography moved out of `blog-list.component.scss` into a global `assets/styles/_prose.scss` `.prose` class; the blog adds `prose` alongside `post-body`, and About consumes the same class. Global on purpose — `<markdown>` renders its HTML at runtime, so a component-scoped rule would need `::ng-deep` in every consumer. Every value still resolves through a design token.

**About was rewritten.** It rendered a lorem-ipsum `aboutText` **twice** and fetched an `aboutMe` it never displayed. It now renders `site_about_markdown` through `<markdown class="prose">`, names the tenant in the heading (`What Makes {{ displayName }} an Amazing Studio?` — previously a hardcoded "Knotty Yoga"), and the dead "How to Join" duplicate section is gone (nothing linked its `#join` anchor). `HomepageService.getAboutText()` was deleted — About reads `SiteConfigService` directly now, one hop instead of two.

**⚠️ Two catalog entries drifted, and Phase 2 respected the drift rather than forcing them:**

1. **Getting Started *step copy* is NOT part of Phase 2.** The bullet predates D6, which moved the steps to `getting_started_steps` **rows**. Only `site_start_intro` is a slot, and only it was wired; the seven steps stay in `allSteps` until Phase 3 replaces them wholesale. Wiring them to config now would have been work Phase 3 immediately deletes.
2. **`site_hero_subline` has no consumer.** Since the catalog was written, Ryan's Home redesign moved the hero band to `home_sections` (title + body from data), so the slot's natural home is gone. `site_hero_headline` went to the **intro strip** (`home-intro__text`) and `site_hero_image_url` to the **badge** beside it — the "secondary hero image" the catalog describes. The subline is still stored, seeded, merged into `SiteConfig` and will appear in the Phase 6 editor, but nothing renders it. **Decision needed from Ryan/Mason:** give it a slot in the intro strip, or retire it from the catalog. Rendering it unilaterally would have added a line of marketing copy to Ryan's design.

**Bug found and fixed along the way:** `SiteConfigService` seeded itself with `{ ...DEFAULT_SITE_CONFIG }` — a shallow copy, so the live config *shared its arrays* with the exported constant. Any consumer mutating `config.socialLinks` / `addressLines` / `taglineLines` would have rewritten the shipped defaults for the whole app (and for every later spec in a run). Now cloned via `cloneSiteConfig()`, with a spec asserting the identity is never shared.

**Gate:** `npx tsc --noEmit` clean on both `tsconfig.app.json` and `tsconfig.spec.json`; **`ng test` 2989/2989 green**. `ng lint` reports 262 pre-existing problems repo-wide (unused imports, `any`) — **zero in any line this phase wrote or moved**; the only touched file that appears is `homepage.service.ts`, whose long-standing `Observable<any[]>` merely shifted line number.

### Hand-testing steps (live server, blank database)

Phase 2 is a **regression check**: every slot now travels from the database, so the visible test is that a freshly seeded Knotty Yoga looks exactly as it did. Recreate the database with `knottyyoga_database_helper --recreate_database`, start `knottyyoga_the_server`, run the Angular dev server, and sign out before starting.

1. **Browser tab.** With the site open, read the browser tab. It must say **Knotty Yoga Fitness Studio**, and the tab icon must be the Knotty Yoga favicon. *(This is the slot proving itself: the title now arrives from the database, not from `index.html`.)*
2. **Home — intro strip.** On the landing page, below the hero photograph, the centred sentence must read **"Knotty Yoga is an inclusive, high-level acrobatic fitness studio."** with the round safe-space badge directly beneath it.
3. **Home — memberships.** Scroll to the **Memberships** section (signed out) and confirm the paragraph under the heading reads **"Unlimited classes, priority sign-ups, and the whole community — pick the tier that fits how you train."**
4. **Get Started.** Top menu → **Get Started**. The paragraph under the **Get Started** title must read **"New to Knotty Yoga? Here is the whole path, start to finish. You do not have to do these in order — but this is the order most people find easiest."** The seven numbered steps below it are unchanged (still code — Phase 3 moves them).
5. **About.** Top menu → **About** ▸ **About**. The heading must read **"What Makes Knotty Yoga an Amazing Studio?"** and the body must be the studio blurb ("We are Knotty Yoga, and we've been in the business of…"). Confirm there is **no** "Lorem ipsum" text anywhere and **no** second "How to Join" block — both were placeholder and are gone.
6. **Footer — address and email.** At the bottom of any page, **Address:** must read **2545 152nd Ave NE, Redmond, WA 98052** and **Email:** must read **info@knottyyoga.com** and open a mail client when clicked.
7. **Footer — tagline.** The right-hand footer column must show two italic lines: **"That which doesn't kill you"** then **"makes you hotter. 💪🤸"**.
8. **Footer — social row.** Under **"Follow us on Social Media!"** there must be exactly three round buttons reading **Fa**, **In**, **Tw**, linking to facebook.com, instagram.com and twitter.com respectively.
9. **Header.** Hover the studio logo in the top-left; its alt/tooltip text is **Knotty Yoga Fitness**.

**Not hand-testable yet:** changing any of these values. `config_secrets` is deliberately excluded from Manage Data, and `knottyyoga_database_helper` has no secrets command (only `--recreate_database` / `--migrate` / `--create_tenant`). **Phase 6's Site Theme page is the first point where a studio can edit a slot**, and the "type a new headline, reload, see it change" walkthrough belongs to that phase's steps.

## Phase 3 — Getting Started steps *and home sections* as data (D6)

Backend first, then the page rewire, then the editor.

> **Scope grew 8/11/2026 (Mason, answering OQ-D10/D11/D12/D13 in [[Component Inventory for Designer]]).** Ryan's Home design adds marketing sections the product doesn't have — "Why Knotty Yoga", "Types of classes", "Additional health services", a "COME JOIN THE FUN" photo band and the "WELCOME TO KNOTTY YOGA" slab — and Mason's call is that **these are data, exactly like the Getting Started steps**: each row an image, title, body and link, with the layout alternating which side the image sits on. Membership tier icons go the same way (three distinct icons, per-tier, in the database).
>
> So this phase now covers **two** row-editors built on one pattern. The `home_sections` table and its seed land first (that's what the Home port needs); the Getting Started rewire follows on the same rails. **Both need the same Manage UI**, and it should be built once as an ordered-rows editor with a photo slot rather than twice — see the shared editor item below.
>
> Images are **uploadable per row** (OQ-D11) so a new studio replaces them without touching code; Knotty Yoga's are seeded from Ryan's exports by `create_database.cpp`, which is what makes a fresh database look like the design.

### Home sections (new — drives the Home port) ✅ **shipped before 8/13/2026**

- [x] `db_schema/home_sections.{h,cpp}` — app-side table. Full new-table checklist. *(app)*
- [x] Table helper + `GET /api/home_sections` (anonymous, ordered by `ordinal`, active only). *(app)*
- [x] Seed Ryan's rows + images in `create_database.cpp`. *(app)*
- [x] Frontend: Home renders features alternating image-left / image-right, and banners full-bleed. `ServerAccess` seam ×5 files. Specs.

### Membership tier icons

- [ ] Per-tier icon image (three distinct laurels), attached to the tier row through the same photo association; seeded from Ryan's exports. The tier card falls back to today's Material icon when a tier has no image. *(app)* — **NOT DONE; the one item left in Phase 3, see below.**

### The shared editor ✅ **DONE 8/13/2026**

- [x] **Manage → Page Content** — one ordered-rows editor serving Getting Started steps *and* home sections: reorder, add/remove, title/body/route/label fields, the icon-picker grid where a row uses a Material icon, and a photo upload where a row uses an image. Specs.

- [x] `db_schema/getting_started_steps.{h,cpp}` — **app-side** table. Registered through the full new-table checklist. *(app)*
- [x] Curated **icon allow-list** constant (Material icon names) — one source of truth serving both server-side `mat_icon` validation and the editor's picker grid. *(app)*
- [x] Table helper + `GET /api/getting_started_steps` (anonymous, ordered by `ordinal`). Writes ride the generic CRUD endpoints. Route validated root-relative — **client-side only for now; see the gap note.** *(app)*
- [x] Seed the seven shipped steps in `create_database.cpp` — "Create your account" seeded `hidden_when_logged_in = true`. *(app)*
- [x] Frontend: `/start` renders from the endpoint — numbering from array position, icon/title/body/CTA from the row. `ServerAccess` seam ×5 files. Component + mock specs.
- [x] Frontend: the editor page — delivered as the **shared** Page Content editor rather than a Getting-Started-only page, which is what the "build it once" item asks for.
- [x] Tests: table helper CRUD + ordering; endpoint anonymity + order; icon + route validation; seed presence on a fresh DB.

### As-built notes (8/13/2026)

**The home-sections half was already shipped** before this session (table, helper, endpoint, seed, Home rendering). Phase 3's remaining work was the steps table, the shared editor, and the tier icons.

**Backend.** `getting_started_steps` follows `home_sections` exactly: DDL, a `TableHelpers::GettingStartedSteps` CRUD wrapper, and an anonymous `GET /api/getting_started_steps`, registered through every checklist point (`make_app_tables`, `CreateTables`, `allowed_tables`, admin column-data-info, column + table friendly names, the `web_app.cpp` anchor, both CMakeLists). The curated icon vocabulary lives in `business_logic/page_content/getting_started_icons.{h,cpp}` and is **served in the endpoint payload** as `allowed_icons` — that is what makes the picker and the server's list structurally incapable of drifting.

**The feed returns EVERY row, including `hidden_when_logged_in` ones.** The client knows the viewer's auth state, so filtering there is what lets the numbering close up without a second round trip — and the rendered number is the array *position*, never `ordinal` (which is sparse: 10, 20, 30…).

**⚠️ Postgres booleans cross the wire as `"t"`/`"f"`** — not `true`/`false`, not a JSON bool. A test caught it. It matters twice: `ServerAccessNetwork.normalizeGettingStartedStep` converts it (an un-normalized `"f"` is a non-empty string, therefore truthy, which would have hidden *every* step from signed-in visitors), and the editor's `toBool` does the same for the generic-CRUD read. **Latent bug spotted next door, not fixed:** the Home page and `ServerAccess.mock` filter `home_sections` with `section.active !== false`, which has the same flaw — currently harmless only because the public endpoint already filters `active = true` server-side. Worth a cleanup pass.

**The editor is one component for both tables** (`/manage/page-content`, reachable from a Manage dashboard card). They are the same shape — ordinal + title + body + one link — differing only in the extras, so the shared parts are written once: reorder via ordinal swap, add/remove with confirm, the copy fields, and a reload-from-server after every write rather than trusting local state. Home sections get the `kind` select, an active toggle and the `hw-photo-upload` control; steps get the icon-picker grid and the hidden-when-signed-in toggle. Reads use `getTableRows`, so **inactive** home sections stay editable — the public feed hides them, the editor must not.

**Gap worth naming: the write path has no server-side validation.** The plan chose generic CRUD for writes, and those endpoints have no per-column hook, so the icon allow-list and the root-relative route rule are enforced **client-side only** today (the picker cannot offer an invalid icon; the route field refuses anything not starting with a single `/`). An admin posting directly to `/api/add_item` could still write junk. Phase 6 builds real validated manage endpoints — the right place to close this, and both the allow-list and the validator already exist server-side waiting for it.

**Deliberate deviation:** the plan listed a "Manage → Getting Started" page *and* a shared "Manage → Page Content" editor. Building both would be the exact duplication the plan warns against, so there is one editor with a list switcher.

**Gate:** C++ **4798/4798** (up from 4779 — 19 new tests) plus an explicit `knottyyoga_database_helper` build (the standard gate does not compile it, and `create_database.cpp` changed). Angular **3018/3018** (up from 2989), both `tsc --noEmit` projects clean.

### What is left in Phase 3

**Membership tier icons only.** It is independent of everything above and needs Ryan's three exported laurel images to seed, so it is a clean standalone slice: attach a per-tier image through the existing photo association, seed from the exports, and have the tier card fall back to today's Material icon when a tier has no image.

## Phase 4 — Token pipeline (incl. fonts as data — D4)

Colors/radius first (pure variable plumbing), then the font machinery.

> **Head start (8/11/2026):** the *default* half of this phase already shipped via integration Track A — `ui/src/assets/styles/_tokens.scss` emits the whole catalog above, the legacy aliases resolve through it, and `ui/src/app/shared/design-tokens.spec.ts` guards the values. What remains here is the **runtime override**: the server carrying `site_theme_*` values and the boot applying them over these defaults.

- [x] Server: accept + serve `site_theme_*` keys in the `theme` object (CSS-var name → validated value). *(hw)*
- [x] ~~Frontend: alias the legacy `--red`/`--orange`/`--gray` to the new `--theme-*` so existing Tailwind mappings restyle immediately~~ — **done 8/11/2026** (Track A; `--orange` intentionally excluded, see the catalog's exception note).
- [x] Frontend: boot application — write each `theme` entry onto `document.documentElement` (inline properties beat the `:root` defaults, so the token file stays the fallback).
> **⚠️ The five font items below are SUPERSEDED — do not work from them.** They are the original D4 sketch. Mason's 8/13/2026 font-source proposal and his three answers (D12/D13/D14) re-scoped all of it, and the work shipped that day under **Phase 4B** at the bottom of this document. They are ticked here so the phase reads honestly; **Phase 4B's checklist is the one that describes what was actually built.** Three places the built thing differs from the wording below are called out inline.

- [x] ~~**`site_fonts` table** … + the CDN origin allow-list constant (Google Fonts first)~~ → **built as THREE tables** (`site_font_sources` + `site_fonts` + `site_font_faces`), and **the origin allow-list was dropped** — D12 made sources tenant-editable rows at Mason's direction. *(hw)*
- [x] **Font serving endpoint** — public, cacheable, correct `font/*` Content-Type + `nosniff`. Built as `GET /api/site_font_face/<id>`. *(hw)*
- [x] ~~`site_info`'s **`theme`** object gains font-face descriptors~~ → **they went into a separate `fonts` object instead.** `theme` carries the `--font-*` ROLES (which family does which job); `fonts` carries the inventory (preconnects, stylesheet URLs, face descriptors). Two different questions deserved two different objects. *(hw)*
- [x] ~~client constructs the allow-listed stylesheet URL~~ → **the SERVER constructs it**, one URL per source with every family's spec joined. The client only injects what it is given, which is what keeps the allow-list-free design safe. *(hw)*
- [x] Frontend boot: inject the constructed `<link>`s and generated `@font-face` rules in the same pre-render initializer; set the `--font-*` role variables; every role carries its fallback with `font-display: swap`. The fallback is **per-row data** (D13), never an assumed generic. *(app)*
- [x] ~~`.din*` font classes re-based onto `--font-*` variables; the bundled D-DIN faces stay as Knotty Yoga's default role values (no `site_fonts` rows needed for KY)~~ — **done 8/11/2026** (Track A; `body`/`*` re-based too, `.din-italic` deliberately left on the bundled face since italic is a face, not a role). *(app)*
- [x] Material bridge: ~~override the `--mat-*` system tokens at boot~~ — **done in CSS instead**, see the as-built note. Scoped to the brand-carrying tokens, not a full re-theme. (D9)
- [ ] Reconcile token names with Ryan's Foundations file when it lands (his names win; update the catalog above). — *nothing pending; the catalog already matches the as-built `_tokens.scss` mined from his file.*
- [x] Tests: token application function (writes vars, skips invalid); a themed boot restyles a token consumer; font-class regression spec. **(Colour/radius half.)**
- [x] Tests: `site_fonts` helper CRUD; upload validation accepts woff2/woff/ttf/otf magic bytes and rejects junk + oversize; serving endpoint MIME/nosniff/cache headers; descriptor round-trip; URL-construction unit test. **(Font half — shipped under Phase 4B; the "allow-listed origin only" clause no longer applies, per D12.)**

### As-built notes — colour/radius half ✅ **DONE 8/13/2026**

The plan's own sequencing ("colours/radius first, then the font machinery") is the split this session shipped: **the token pipeline is live end to end; the fonts-as-data cluster is untouched.**

**The registry is two-layer, matching the stylesheet.** `components/platform/business_logic/branding/site_theme_tokens.{h,cpp}` maps each `site_theme_*` config-secret key to the CSS custom property it overrides, with a type (`Color` / `Length` / `FontFamily`). It registers all six seven-step **palette** ramps plus the standalone tints, the brand/neutral/status **roles**, the four radius tokens, and the three `--font-*` roles — ~60 tokens.

Registering the whole palette (not just the base step) is what makes a re-brand one layer: every role is `var(--palette-…)` and custom properties resolve at **use** time, so overriding `--palette-primary-400` moves `--theme-primary`, `--theme-primary-hover` and everything downstream. A spec proves exactly that.

**Unset tokens are OMITTED, not sent empty** — the opposite of the content slots, and deliberately so. The SPA's `_tokens.scss` holds every default, so the payload is purely the override layer; an empty custom property is invalid at computed-value time and would break the cascade the defaults provide. "Absent" is the only safe way to say "keep the default", which also means **a token has no config_secrets row until a tenant sets one** — the normal state, and why these keys have no default VALUES on either the framework or the app side.

**Why ~60 keys are not in `secret_keys.h`.** Nothing outside the theming code names a token individually, and the mapping that matters (key → CSS property → type) only a table can express. The registry is the single source of truth; `secret_keys.h` carries a pointer to it.

**New validators (D10).** `IsValidCssLength` takes `8px` / `0.5rem` / `50%` / bare `0` and refuses `calc()`, `var()`, unitless numbers and anything with a `;`. `IsValidFontFamilyList` allows letters, digits, spaces, hyphens, underscores, commas and balanced double quotes — refusing `;` `{` `}` `(` `)` `\` `/`, so a family name cannot become a rule of its own. Both matter because these values land in a custom property: a "corner radius" field must not be able to smuggle CSS.

**The boot applier** writes each entry with `setProperty` on `document.documentElement` — the same mechanism the style guide's live "try another studio" preview already uses. Two client-side guards on top of the server's validation: it only ever writes properties starting with `--` (so a payload can never set a real CSS property), and it skips blanks.

**⚠️ Deliberate deviation — the Material bridge is CSS, not JavaScript.** The plan said "override the `--mat-*` system tokens at boot". Angular Material 21 reads its colours through the M3 **system layer** (`--mat-sys-primary` and friends — verified against the shipped prebuilt theme), so a new `angular-material-overrides/_mat-system-bridge.scss` points those at our roles once. Strictly better than doing it at boot:
- the boot applier only ever writes `--theme-*` / `--palette-*` — one concept, not two;
- a tenant override reaches Material through the same `var()` chain, with nothing to keep in sync;
- the style guide's live preview restyles Material too, because it is the same cascade rather than a boot-time snapshot.

Scope is narrow as the plan asked: primary, accent and the error tone. Surface/container tokens stay Material's so its elevation ramp stays internally consistent. Buttons were already covered — `_mat-button.scss` restyles them from our roles directly; the bridge is what reaches the spinner, toggles, checkboxes, radios and the form-field focus ring. `design-tokens.spec.ts` now asserts every bridged token equals its role **and** that a runtime `--palette-primary-400` override reaches `--mat-sys-primary`.

**Gate:** honuware **1550/1550** (+18), knottyyoga C++ green in co-dev against the updated honuware, Angular **3026/3026** (+8), both `tsc --noEmit` projects clean. `ng lint` is unchanged at its pre-existing 262 repo-wide problems — zero new findings from Phases 3 or 4.

**⚠️ Pin bump owed again.** honuware gained `site_theme_tokens.{h,cpp}`, the two new validators and the `site_info` theme wiring after the 8/12 re-pin to `5dcae2c`. Both consumers (`knottyyoga_server/CMakeLists.txt` and `communityfinder_server/CMakeLists.txt`) still point at that SHA, so the honuware half needs pushing and re-pinning before a non-co-dev build sees any of Phase 4.

---

## Phase 4B — Fonts as data (D4), formalized

> **Design settled 8/13/2026 from Mason's font-source proposal.** He supplied the shape directly: *"have a font source like Google with the href values and a flag for the cross origin thing and then the base URL … and then each font could have the `family=Barlow:wght@100..900` for the link tag and then the font-family (like 'Roboto') and the sans-serif to append."* That is the Google Fonts API's own structure, and modelling it as data is what makes "add a font" a row instead of a code change. Three decisions were put to him; his answers are recorded inline below and **supersede D4's original CDN sketch**.

### D12 — Font sources are tenant-editable rows *(supersedes D4's allow-list)*

**Mason's call, over the recommendation.** D4 said CDN entries would be family+weights against a framework **allow-list of origins**, never a free-form URL. The allow-list is dropped: `site_font_sources` is a per-tenant table an admin can add to.

The three costs were put to him explicitly and accepted:
- an admin can point the site at any third-party origin;
- a strict CSP can no longer be enumerated ahead of time (it has to allow the tenant's configured font hosts, or run without one for font origins);
- CDN mode discloses visitor IPs to a host we never reviewed (D4's recorded GDPR caveat, now unbounded).

What is kept, because it is hygiene rather than policy: every URL must be **`https://`**, well-formed, and free of whitespace/control characters, and the values are length-capped. That does not re-introduce an allow-list — it stops a malformed row breaking every page.

### D13 — A font row carries family AND fallback as separate columns

**Mason's call, rejecting both offered options:** *"I think that we should have both as separate database entries. I don't like the full stack string but I also don't want to assume sanserif."*

So: `family` and `fallback` are **two required columns** on every `site_fonts` row, and **no generic family is ever defaulted in code**. A role token stores the family NAME only (`site_theme_font_body = "Barlow"`); the server composes the stack it serves by looking the row up — `'Barlow', sans-serif` — so the fallback is defined in exactly one place and the admin never types a CSS stack.

This adds a third source kind to D4's two. `source_kind`:

| kind | means | needs |
|---|---|---|
| `cdn` | loaded from a font source's stylesheet | a source + a `spec` fragment |
| `uploaded` | face files stored in the tenant's DB | one or more `site_font_faces` rows |
| `system` | already on the visitor's machine (Georgia, Helvetica) | nothing but family + fallback |

`system` is what makes "no assumed sans-serif" work without a special case: a studio that wants a pure system stack creates a real row, and the fallback is data like everywhere else.

### D14 — Uploads ship in this pass

**Mason's call:** both CDN and uploaded faces now, not CDN-first. That brings the plan's one security-sensitive surface into scope — binary upload plus a public byte-serving endpoint — so it carries D4's guardrails verbatim: **magic-byte validation** (`wOF2` / `wOFF` / `\x00\x01\x00\x00` / `OTTO` / `true`), a per-face **size cap**, and serving with the correct `font/*` Content-Type plus `X-Content-Type-Options: nosniff` so uploaded bytes can never come back as script. **License responsibility rides with the uploader**, noted in the admin UI.

### The data model

Three tables, all framework (D11 — CommunityFinder wants fonts too), all in the tenant's own database.

**`site_font_sources`** — where a family can be fetched from.

| column | notes |
|---|---|
| `source_key` | stable handle, e.g. `google` |
| `display_name` | what the editor shows, e.g. "Google Fonts" |
| `base_url` | `https://fonts.googleapis.com/css2` |
| `query_suffix` | appended once per constructed URL; seeded `display=swap` |
| `preconnect_lines` | newline-separated `href\|crossorigin`, the `site_social_links` precedent (OQ-T4) rather than a fourth table |
| `active`, timestamps | |

**`site_fonts`** — the tenant's family inventory. `family`, `fallback`, `source_kind`, `font_source_id` (nullable), `spec` (nullable — the `family=Barlow:wght@100..900` fragment), `ordinal`, `active`, timestamps.

**`site_font_faces`** — uploaded binaries only. `site_font_id`, `weight`, `style`, `format`, `bytes`, timestamps.

### How a page ends up with Barlow on it

1. `GET /api/site_info` gains a **`fonts`** object beside `content` and `theme`:
   - `preconnects`: `[{href, crossorigin}]` — every active source's, de-duplicated.
   - `stylesheets`: one constructed URL **per source**, not per font — the `cdn` rows for that source contribute their `spec` fragments joined with `&`, then `query_suffix`. Mason's own example falls straight out: `…/css2?family=Barlow:wght@100..900&family=Roboto:wght@100..900&display=swap`. One request, not two.
   - `faces`: `[{family, weight, style, format, url}]` for uploaded families, pointing at the serving endpoint.
2. The `theme` object's `--font-body` / `--font-heading` / `--font-display` arrive **already composed** (`'Barlow', sans-serif`) from the named row's family + fallback.
3. The boot initializer injects the preconnect links, the stylesheet links, and an `@font-face` rule per uploaded face — then applies the theme tokens exactly as Phase 4A already does.

### Checklist ✅ **DONE 8/13/2026**

- [x] `site_font_sources` + `site_fonts` + `site_font_faces` schema, table helpers, and URL/family/fallback/spec validation. *(hw)*
- [x] Magic-byte + size-cap validation for uploaded faces. *(hw)*
- [x] `GET /api/site_font_face/<id>` — public, long-cache, `font/*` + `nosniff`. *(hw)*
- [x] `site_info` gains `fonts`, and composes the `--font-*` role stacks from the named rows. *(hw)*
- [x] Seed the Google source + **Barlow** and **Roboto** rows in `create_database.cpp` — a fresh database reproduces today's `index.html` links from data. *(app)*
- [x] Frontend boot: inject preconnects, stylesheet links and `@font-face` rules pre-render; roles keep their fallback with `font-display: swap`. *(app)*
- [x] Tests: helper CRUD; URL construction; upload validation; serving endpoint MIME/nosniff/cache headers; role-stack composition; boot injection.

### As-built notes (8/13/2026)

**The URL builder reproduces Mason's example exactly**, and a test asserts that literal string: two `family=` fragments joined with `&`, then the source's `display=swap`, in **one** request per source rather than one per family. Font order comes from `ordinal` so the URL is byte-stable between requests — a churning URL would miss the CDN cache on every page load. A source that no active family references emits **nothing**, not even a preconnect: opening a connection to a third party for no reason is a cost with no benefit.

**⚠️ The spec grammar is richer than the design sketch.** Writing the validator's test surfaced that Google's real syntax uses **semicolons** (`family=Roboto:ital,wght@0,300;0,400;1,400`) — which the app's own shipped `index.html` URL relies on, and which my first validator rejected. `;`, `%` and `+` are all in the accepted set now; `&`, `?` and `#` stay out so a row can only ever contribute the one parameter its admin configured.

**Uploads (D14).** Format is decided by **magic bytes** (`wOF2`/`wOFF`/`OTTO`/`00 01 00 00`/`true`/`ttcf`), never the filename, and that stored format is what picks the MIME type on the way out — an unrecognised value is refused rather than guessed, because guessing would undo `nosniff`. Faces are served `font/*` + `nosniff` + `max-age=31536000, immutable` (a face is immutable; a replacement upload gets a new id). Byte reads are a separate call from descriptor reads so a manage page never drags font files into memory.

**A shared bytea codec fell out of this.** Binary column values cross the KeyValueTable layer as `\xABCD` hex, and encode/decode lived as two private functions inside `image_helper.cpp`. Fonts made that the second binary store, so it moved to `sql_util/database_access/bytea.{h,cpp}` with its own tests, and `image_helper` now calls it. The rewrite also fixed a latent throw: the old decoder used `std::stoi`, which raises on malformed input — on a read path handling untrusted database content. It now stops at junk instead.

**🐞 A REAL PRODUCTION BUG, found by this work and pre-existing.** `/api/site_font_face` 404'd in the app suite while passing standalone. Root cause: honuware's `Endpoints::RegisterFrameworkEndpoints()` — the function whose entire job is anchoring framework endpoints against `-O2` dead-stripping — **was never called by anything**. Framework endpoints reached the app's link only if the app happened to enumerate them itself in `web_app.cpp`, or if a *test* file referenced them.

`GetSiteInfo` is in neither the app's anchor list nor any app source. So **`/api/site_info` has been dead-stripped out of the production server binary** — the route 404s in production. It passed every test because honuware's `site_info_test.cpp` references `GetSiteInfo`, pulling the object into the *test* binary only. And the failure is invisible from the outside: `SiteConfigService.load()` swallows the error and falls back to bundled branding, so a 404 looks exactly like "tenant hasn't customised anything".

**This means Phases 1, 2 and 4A were never actually reachable in production** — the plumbing was right, the route was absent. Fixed by having the app's `RegisterAllEndpoints()` call `RegisterFrameworkEndpoints()`, which anchors honuware's whole list in one line and means a framework endpoint added later needs no app change. Worth a look at whether any *other* framework endpoint was only ever reachable via its test.

**Gate:** honuware **1601/1601** (+51), knottyyoga C++ **4867/4867** (+51) plus an explicit `knottyyoga_database_helper` build, Angular **3034/3034** (+8), both `tsc --noEmit` projects clean.

**Still to do (small):** `index.html` keeps its hardcoded Google Fonts `<link>` tags. They are now redundant for a seeded tenant but harmless — and removing them trades a parse-time font fetch for one that waits on `/api/site_info`. Worth a deliberate decision rather than a silent change; the fallback stacks and `font-display: swap` make either choice safe.

## Phase 5 — Imagery v1 (URL-based)

- [x] Wire `site_hero_image_url` + `site_favicon_url` consumers (Phase 2 stubs them behind defaults; this phase completes fallback behavior + error handling — a 404'd image falls back to the bundled asset, never a broken-image icon).
- [x] Document the constraint set for each slot (logo: light-on-dark, height-bounded; hero image: aspect guidance) — recorded below; Phase 7's runbook and Phase 6's editor hints both consume this table.
- [ ] *(Deferred, recorded not planned: a `site_assets` upload flow so tenants don't need externally hosted URLs — revisit after the first real second tenant.)*

### As-built notes (8/13/2026)

App-side only — both URL slots already existed from Phase 1, so no honuware change and no pin bump.

**The gap Phase 2 left was "well-formed" vs "resolves".** Server-side validation (D10) proves a URL *parses*; only the browser can discover it 404s. So each image consumer now degrades to its bundled asset:

| Consumer | Mechanism |
|---|---|
| Header logo | `(error)` on the `<img>` → swap to `logoAsset` |
| Home intro badge | `(error)` on the `<img>` → swap to `heroImageAsset` |
| Favicon | **probe-then-swap** (see below) |

Each fallback is attempted **once**, guarded by a flag: if the bundled asset were itself missing, an unguarded handler would re-fire on its own replacement and loop.

**The favicon needed a different mechanism.** `<link rel="icon">` fires no error event, so a dead tenant URL would silently blank the tab icon with nothing to catch. The bundled icon is installed **first**, then the tenant URL is loaded into an off-document `<img>` purely to see whether it resolves, and only swapped in on success. The probe is deliberately not awaited — boot never waits on an icon, and the bundled one is showing meanwhile.

### Slot constraint set

What a studio needs to know before it supplies each image. These are the engineering half of Ryan's slot-constraint rules in [[Component Inventory for Designer]]; Phase 6's editor should show them as field hints, and Phase 7's runbook should quote them.

| Slot | Constraint | Why |
|---|---|---|
| `site_logo_url` | **Light-on-dark.** Sits on the black header band (`--theme-inverse-surface`). A dark logo disappears. | Ryan's shell is solid black; the bundled KY mark is white for this reason. |
| | **Height-bounded**, rendered at 120px wide in a full-height slot. Supply something that reads at that size. | The header height is fixed; a tall logo is scaled down, not given more room. |
| | **SVG preferred**, PNG with transparency acceptable. Never a JPEG. | A JPEG's opaque box shows as a white rectangle on the black band. |
| `site_favicon_url` | Square, ≥32×32. ICO, PNG or SVG. | Browsers pick from what they're given; a non-square icon is letterboxed. |
| | The stale `type="image/x-icon"` is dropped when a tenant URL wins, so any format works. | index.html declares ICO for the bundled default only. |
| `site_hero_image_url` | Square-ish badge beside the intro sentence, rendered ~142px wide. | It is a mark, not a photograph — the bundled default is the safe-space badge. |
| | Transparent background. | It sits on the page background, not a card. |
| **All three** | `https://` or root-relative (`/…`); never `http://`, never protocol-relative. | Enforced server-side (D10). An `http://` asset would mixed-content-block on an https site. |
| **All three** | Must actually resolve — a 404 silently degrades to the bundled asset. | The tenant sees *our* branding, not theirs, with no error surfaced. Worth calling out in the editor. |

Carousel photos, home-section images and instructor photos are **not** in this table: they upload through the photo system and are already size-validated and scaled server-side.

### Hand-testing steps (live server, blank database)

Like Phase 2, the visible test is a **regression**: with no tenant URLs configured, every image must still be Knotty Yoga's own. Recreate the database with `knottyyoga_database_helper --recreate_database`, start `knottyyoga_the_server`, run the Angular dev server, and sign out first.

1. **Header logo.** On any page, the top-left mark is the white Knotty Yoga logo on the black band — not a broken-image icon and not an empty gap.
2. **Browser tab.** The tab icon is the Knotty Yoga favicon and the title reads **Knotty Yoga Fitness Studio**.
3. **Home badge.** On the landing page, under the intro sentence, the round safe-space badge renders.
4. **No console errors.** Open the browser console on the landing page: there must be no failed image requests. *(The one place a silent fallback would otherwise hide a problem.)*

**Not hand-testable until Phase 6.** Pointing a slot at a deliberately broken URL — the actual new behaviour — needs the Site Theme editor; `config_secrets` is excluded from Manage Data by design and `knottyyoga_database_helper` has no secrets command. The fallback paths are covered by spec instead (header logo, home badge, and both favicon outcomes). **Add to Phase 6's steps:** set `site_logo_url` to a URL that 404s, reload, and confirm the bundled logo appears rather than a broken image.

## Phase 6 — Admin "Site Theme" page

- [x] Backend: curated endpoints `GET /api/manage/site_theme` (full editable set, unset-vs-default distinguished) + `PUT /api/manage/site_theme` (validated writes to the tenant's `config_secrets`). Admin-only via the existing permission machinery. **Never** the generic CRUD surface — `config_secrets` stays excluded. *(hw)*
- [x] Frontend: Manage-portal page (back office — inherits building blocks, no Ryan design) with sections **Brand basics** / **Copy** / **About** (markdown + live prose preview) / **Colors** (pickers per token role) / **Fonts** (**role assignment**) — with per-field "reset to default". *(app for now; lift candidate in Phase 8)*
- [x] ~~"Changes appear within ~5 minutes" note (the site_info cache)~~ + a save-confirmation. → **The cache was removed instead (8/14/2026); a save now applies immediately and the note reads "Your site is using these changes now." See the as-built note below.**
- [x] Tests: endpoint auth/validation/round-trip; component specs per section.
- [x] **The font MANAGER** — add/remove families, CDN family+weights vs file upload with the license-responsibility note, per-face preview. ✅ **DONE 8/14/2026** — `/manage/site-fonts`, its own page + dashboard card. See the as-built note below.

### As-built notes (8/13/2026)

**The endpoints.** `GET/PUT /api/manage/site_theme` (honuware, admin-only). The GET returns every content slot and every theme token with `key`, `type`, `value` and **`is_set`** — the flag is the whole reason a curated endpoint exists rather than a generic one: a field the studio *cleared* and one they *never touched* are both `""` in `config_secrets`, but only one of them means "reset me to the default", and the editor's reset control has to tell them apart.

**The GET returns the RAW stored value, not the normalized one.** site_info blanks anything that fails validation so a page never breaks — but the editor must show what the studio actually typed, or they cannot see and fix the thing that isn't working. A test pins that.

**PUT validates the whole set before writing any of it.** A half-saved theme is worse than a refused one: the studio sees part of their change land with no idea which part failed. The good field is deliberately listed *first* in the atomicity test so a naive implementation would already have written it.

Three more properties worth naming, each with a test:
- **Only the keys present are written**, so saving one section can never blank another — which is what makes a section-at-a-time page safe.
- **Empty clears a theme token** (that IS "reset to default"), so empty is exempt from the value validator, which rightly refuses it everywhere else.
- **Unrecognised keys are ignored.** `config_secrets` holds live credentials; this curated surface must not become the way to write `mail_app_password`.

**The page** (`/manage/site-theme`, reachable from a Manage dashboard card) has the five sections, saves one section at a time, and shows the "~5 minutes" cache note on success — a studio that reloads immediately and sees nothing has not done anything wrong. Colour tokens are labelled by their **CSS variable** rather than a prettied-up name: that is the vocabulary a designer already uses, and inventing a second one is a sync problem. The colour swatch shows `#000000` for an unset token but keeps the stored value empty until something is actually picked, so "unset" survives being looked at. Phase 5's slot constraints appear as field hints, where the decision is being made rather than in a runbook nobody opens. Validation failures echo the server's own message — the difference between "something went wrong" and a studio fixing its own typo.

**⚠️ Split: the font MANAGER is not built.** The Fonts section does **role assignment** — which of the tenant's families is body/heading/display — because that is what lives in `config_secrets` and therefore belongs to this endpoint. Adding and removing families, and uploading faces, edits `site_fonts` / `site_font_sources` / `site_font_faces` **tables** and needs a multipart upload endpoint. Splitting on the **storage boundary** rather than an arbitrary line: everything the Site Theme page writes is a secret; the font inventory is rows and binaries. The tables, validation, and serving endpoint all exist from Phase 4B — what is missing is the write endpoints and the manager UI.

**Gate:** honuware **1616/1616** (+15), Angular **3060/3060** (+19), both `tsc --noEmit` projects clean.

**Pin bumped** — honuware `a4aeb18`; both consumers verified against the pinned clone (knottyyoga 4882, communityfinder 1637).

---

## Phase 6B — Editor usability revision

> **Driven by Mason's 8/14/2026 review of the shipped editors.** He used Page Content and Site Theme and reported what was wrong with them. Most of this is not polish: three items are functionality I simply did not build (the font manager, the type scale, per-token descriptions), and two are things I built that do not work well enough to use (the tab indicators, the colour swatches).
>
> Three design questions were put to him; all three answers took the **more thorough** option, so the scope below is the full version rather than the cheap one.

### What he reported

**Page Content**
1. No image preview — you have to click *edit* to see which photo a row uses.
2. Image dimensions are not shown anywhere.
3. **No visual indication of which list is selected** (Home sections vs Getting Started steps).
4. The row's `kind` is not shown.
5. *"Honestly, it would be really nice if you just previewed the entry like it will appear on the home page with the metadata below it."*

**Site Theme**
6. **No visual indication of which tab is selected.** → make them real tabs.
7. "Colours" → **"Colors"**. (US spelling; the rest of the app uses it.)
8. Colors **show black instead of the actual colour**, and no RGB value.
9. No plain-English description of what each colour is *for*.
10. Colours should be **grouped into sections, each with a visual representative that changes as you edit**.

**Fonts**
11. Wants to **add/alter font sources and families** — the CDN, the URL suffix, the CSS font indicator, the font name, the fallback. *(This is the font manager I deferred at the end of Phase 6.)*
12. **Example text that changes with the font.**
13. The **type scale** (small / medium / headline …) should be listed and editable.
14. **Weights and styles** should be settable for fonts that support them.

### Decisions (Mason, 8/14/2026)

- **D15 — Previews reuse the REAL Home components.** *(His pick over an editor-specific card.)* `home-page.component.html`'s inline markup splits into per-kind components — hero, feature, banner, artwork — which both the public page and the editor render. The editor draws them at reduced scale with metadata beneath. The cost is a refactor; the payoff is that a preview **cannot drift** from the live page, and the four kinds genuinely look different in the editor because they *are* different components. Getting Started steps get the same treatment (the numbered step card becomes a component).
- **D16 — Colors get five groups, each with a live specimen.** Brand → buttons and a link. Surfaces & text → a card with heading, body, muted line and border. Status tones → the four badges plus a tinted notice. Shell → the black header/footer bar. Palette ramps → the seven-step swatch row. Each specimen is built from the **real app classes**, so editing a token restyles it in place — the page becomes its own proof. Every row shows a swatch, the resolved `#RRGGBB`, a plain-English description, and reset.
- **D17 — The font manager ships in full, uploads included.** *(His pick over CDN-only.)* Sources and families become editable, CDN weights get a friendly range control that writes the `spec` string, and uploaded faces get drag-and-drop with the licence-responsibility note (D4) and a per-face specimen.

### The "black swatch" bug, and the right fix

The colour inputs show `#000000` for every unset token. That is my `colorFor()` returning black as a placeholder — but it is the wrong *idea*, not just a wrong value: an unset token is not black, it is **whatever the stylesheet's default is**, and that is the value a studio needs to see before deciding to change it.

The fix is available entirely client-side and needs no server change: the bundle's `_tokens.scss` is already loaded, so
`getComputedStyle(document.documentElement).getPropertyValue('--theme-primary')` returns the **effective** value — the tenant's override if one is set, the shipped default otherwise. The editor shows that, and uses the server's `is_set` to say whether it is *your value* or *the default*. Resolution happens against a `<div>` carrying the pending edits, so the specimens preview unsaved changes without touching the live page.

### Type scale and weights are missing from the registry

`--text-xs … --text-2xl` and `--weight-regular … --weight-bold` exist in `_tokens.scss` but were never registered as `site_theme_*` tokens, so there is no way to change them — item 13 is a genuine gap, not a UI omission. They join the registry as `Length` and a new numeric-weight type. *(hw)*

### Image dimensions

Nothing exposes a photo's size today. `GET /api/has_photo/<table>/<id>` already answers "is there an image here" for logged-in users; it gains `width`, `height` and `type` when one exists — the same question the editor is asking, one call, and **without fetching the bytes** (a list of eight sections must not pull eight full-size images). *(hw)*

### Checklist

**Backend (hw — needs a pin bump)** ✅ **DONE 8/14/2026**
- [x] Register the type-scale and weight tokens in the theme registry.
- [x] Extend `has_photo` with `width` / `height` / `type`, reading dimensions without the blob.
- [x] Font-manager write endpoints: `GET/PUT /api/manage/site_fonts` plus face upload and delete.
- [x] Serve per-token English descriptions (and the editor GROUP) from `GET /api/manage/site_theme`.
- [x] Tests for each.

**Page Content (app)**
- [ ] Split Home's inline section markup into `hero` / `feature` / `banner` / `artwork` components with a `preview` mode; Home renders them unchanged. **NOT DONE — see below.**
- [ ] Extract the Getting Started step card the same way. **NOT DONE.**
- [x] Editor rows show a **thumbnail** plus metadata (ordinal · kind · dimensions · shown/hidden) and the existing actions.
- [x] Real `mat-tab-group` for the two lists.

**Site Theme (app)** ✅ **DONE**
- [x] Real `mat-tab-group` for the five sections; "Colours" → "Colors".
- [x] Colors: resolved swatch + hex + description + reset, five groups, live specimens.
- [x] Type scale + weights editable.
- [x] Specs.
- [ ] Font manager UI (the backend is done; the screen is not). **NOT DONE — see below.**

### As-built notes (8/14/2026)

**The black-swatch fix went further than a default value.** `resolved()` returns the studio's override if set, otherwise the value `getComputedStyle` reports for that custom property on `:root` — the real shipped default, read from the stylesheet that is already loaded. Each row shows the swatch, that value, the English description from the registry, and a `yours` / `default` marker.

**Two bugs found while doing it:**
- Resolving against a `@ViewChild` probe carrying the pending edits caused **NG0100** — the probe does not exist on the first change-detection pass, so the binding was `''` then `#ed1c26`. Resolving against `document.documentElement` instead removes the view dependency. The value is cached per property, because a template binding that calls `getComputedStyle` runs on every cycle and these defaults never change.
- `SharedModule` does not export `MatTabsModule`, so both pages needed it imported explicitly — the tabs failed to render at all until then.

**Live specimens** get the pending overrides through `[ngStyle]`, so editing a token restyles the specimen in place while the row values stay stable. Consequence worth knowing: overriding a *palette* step updates the specimens immediately, but a *role* row still displays its original default until saved and reloaded, because the role's own resolution is cached. The specimen is the live preview; the row is the value.

**Metadata moved under the entry** rather than into its title: ordinal, kind, dimensions, and a Shown/Hidden badge. Dimensions come from the extended `has_photo`, one call per row, no image bytes.

### Caching: `max-age=300` was the wrong call ✅ **FIXED 8/14/2026**

Mason reported that changing `--theme-primary`, saving, and **reloading** left the site's colour unchanged — "it doesn't appear to ever override the default." Everything was, in fact, working: the value was in `config_secrets`, `/api/site_info` served it (verified by running the real server against the real database and curling the endpoint), the boot initializer applied it, and the CSS aliases pointed at it. The browser was replaying its cached copy of the bootstrap response, which a normal reload honours.

`Cache-Control: public, max-age=300` was written as an optimisation, and as a five-minute delay on a rarely-changed value it looks harmless. It isn't, for two reasons:

1. **A stale copy of this response is the tenant's branding silently not applying.** This is the one response that decides how the whole site looks. The failure mode is indistinguishable from the feature being broken — which is exactly the conclusion it produced. Anything that makes a working feature look broken is not a good trade for one saved request per boot on a ~1.5 KB body.
2. **`public` on a tenant-varying response is a correctness bug, not just a staleness one.** The tenant is resolved from `X-Honuware-Site`, which is not part of a shared cache's key by default. A shared cache in front of the origin could serve one studio's branding to another.

As built: `Cache-Control: no-cache` (still storable — it requires revalidation before reuse, so a shared cache keeps working) plus `Vary: X-Honuware-Site`. Both are asserted in `site_info_test.cpp`.

**The editor no longer waits for a fetch at all.** `SiteConfigService.applyTheme()` is now public, and `SiteThemeComponent.save()` calls it with what was just saved — the same applier boot uses, so what the admin sees is what the next visitor gets. The confirmation reads "Your site is using these changes now."

**The test gap that let this ship.** Every existing theme test asserted the custom *property* changed. None asserted that anything *rendered* followed it, so "applied to the document" and "visible on the site" were never distinguished. `site-config.service.spec.ts` now paints a real element with `var(--theme-primary)` and asserts its **computed colour** changes.

**Two related leaks found while fixing it** — both the same defect class (theme tokens written to `<html>`, which outlives the writer):
- The new `applyTheme()` call meant the existing save spec restyled the real `<html>` and never put it back, breaking three `design-tokens.spec.ts` assertions later in the run. The spec now snapshots and restores the root inline style rather than guessing which properties a test touched.
- **A genuine product bug:** `StyleGuideComponent.previewStudio()` writes an invented palette onto `<html>` with no `ngOnDestroy`, so previewing "Cedar & Salt" and navigating away left the **entire app** green until a full reload. Fixed with `ngOnDestroy → resetStudio()`, with a spec that destroys the fixture and asserts the token is back.

### Editor polish — field rhythm and naming the default ✅ **8/14/2026**

**Spacing.** A field's description is a `mat-hint`, which Material renders *inside* the field's own box. So a container gap is measured from below the description, and the description ends up sitting on the next field's outline. Worst on the Fonts tab, where `.token-group` is a plain block with no gap at all — the panels touched. Fixed with a `.token-group--fields` modifier (a `layout.stack` at `--space-5`) on the font-role section, and the same rhythm applied to the `.form-card` that Brand basics and Copy use. `.about-split` is excluded — it is a two-column grid, not a stack of fields. `.token-group__title` went from `--space-1` to `--space-2`: heading and blurb still read as a pair, and the air now comes from the group's gap.

**"Use the default" now names the default** — `Use the default (Roboto)`. Choosing it was otherwise a blind pick, when deciding whether the default is the font you want is the entire purpose of the row. `resolved()` deliberately prefers the studio's override, so this needed the other half: `stylesheetDefault()` (same cache, no override) plus `defaultFontLabel()`, which takes the leading family off the stack and unquotes it so it reads like the real options. Falls back to the bare wording when a token resolves to nothing.

### The font manager ✅ **DONE 8/14/2026**

Mason, plainly: *"I've asked SEVERAL times to have a font source like Google's CDN link and then being able to tie fonts to a source and being able to specify a typeface… You keep ignoring me."* He was right. Phase 4B built the entire backend — three tables, table helpers, validation, `GET/PUT /api/manage/site_fonts`, upload with magic-byte checking, `GET /api/site_font_face/<id>` — and then stopped at the screen. Everything worked over HTTP and none of it was reachable from the UI, so from the studio's chair the feature did not exist. **The lesson for the rest of this plan: a phase is not done when its endpoints are green. It is done when someone can use it.**

**`/manage/site-fonts`** — its own page, its own dashboard card, two tabs.

- **Your fonts** — one card per family. The card leads with a **specimen set in that family's own stack**, because the whole question is "what does this look like". Name, backup font (D13, per-row, never assumed), and where it comes from: a font service / files I upload / already on the device. A CDN family gets a service picker and a spec box with that service's grammar as the placeholder. An uploaded family gets a file list — **one specimen per FILE, at that file's own weight and style**, which is the only way to tell whether the file you just added is the one you meant — plus a named weight picker (`600 — Semi Bold`, not `600`) and the licence-responsibility note.
- **Font services** — sources as editable rows, added from **presets** (Google, Bunny, "Something else"). The preset only writes starting values; every field stays editable, so this is not the curated list D12 rejected — it just spares the studio from typing `https://fonts.googleapis.com/css2` and Google's preconnect pair from memory.

**Design points worth keeping:**
- **The preview is the real thing.** The page calls the boot-time font injector (`SiteConfigService.applyFonts`, made public) with the inventory as currently edited, so specimens render in the actual typeface rather than the fallback. Same code path as boot ⇒ a preview cannot drift from what a visitor gets. It is re-applied after every upload and tags what it injects so repeated calls don't pile up `<link>`s.
- **Removing a source rehomes its families** to "already on the device" instead of leaving rows that reference a source that no longer exists — otherwise the studio gets a validation error with no visible cause.
- **Adding a second copy of a preset de-duplicates the key** up front, rather than after a failed save.
- **Server error text is surfaced verbatim.** The server validates the whole payload before writing any of it and names the offending row; replacing that with "could not save" would throw away the only useful part.

**Which font does what** (Site Theme → Fonts) now **previews**: every option in the dropdown is set in the font it names, and each role has a live specimen underneath at the weight and size that role is actually used at — heading and display specimens are bold and larger, because a role preview that lies about weight is not a preview. Specimen text is editable; a studio usually wants to see its own name.

**⚠️ A real backend bug found while building this — the save destroyed uploaded fonts.** `PutManageSiteFonts` deleted every family and re-added it. Uploaded font FILES hang off the family row id, so recreating rows threw away every face the studio had uploaded — triggered by a save that merely renamed an unrelated family, with nothing to recover from. This made the uploaded-typeface path unusable in practice, which is half of what D4/D14 exist for.

Fixed by **reconciling** instead: match sources by `source_key` and families by family name, update in place (new `SiteFonts::UpdateSource` / `UpdateFont`), and delete only what the payload dropped — families before sources, faces before families. `UpdateFont` clears `font_source_id` to NULL via its own statement when a family stops being a `cdn` row: omitting the key would leave the stale id, and an empty string cannot be bound to a bigint. Six new C++ tests, including the one that matters — edit one family, assert another family's uploaded face survives. The Angular mock reconciles the same way, so a spec cannot pass against behaviour the server does not have.

Gate: honuware 1636, app 4902, Angular 3128. **Needs a pin bump after CI.**

### What is left in Phase 6B

Two items, both app-side frontend, both cleanly separable:

1. **The Home-component extraction (D15).** The editor currently shows a **thumbnail** and metadata, not the real hero/feature/banner/artwork rendering. Doing it properly means splitting `home-page.component.html`'s inline markup into four components with a `preview` mode and having both the public page and the editor render them — which is the only way a preview cannot drift. Deliberately not rushed at the end of a long pass: it touches the live home page.
2. **The font manager UI.** Every endpoint it needs exists and is tested (`GET/PUT /api/manage/site_fonts`, face upload with magic-byte validation, face delete) — what is missing is the screen: source editor, family editor with a CDN weight-range control, drag-and-drop upload with the licence note, and per-face specimens.

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

## Phase 9 — Theme files: save a whole look, load it back

> **Status: SCHEMA SIGNED OFF 8/17/2026 — all four OQs answered. Implementation plan below is execution-ready; nothing is built yet.** Mason: *"I'd like to create a JSON schema for saving the settings from Site Theme, Fonts, and Page Content… images be filenames to a pathless image file… loaded from and saved to the same directory as the json… It's important that we capture all of the settings. This will be a nice way to be able to work on site themes and try various alternatives."*
>
> **Answers (8/17/2026):** TF1 — download/upload in the **admin UI is first cut**, not a follow-on. TF2 — replace-by-default with `--merge` opt-in, confirmed. TF3 — **strictness is an OPTION, and the format supports schema migration.** TF4 — the logo **is** part of a theme.

### What this is for

Today a look lives only in one tenant's database. You can build it, but you cannot **keep** it, diff it, hand it to Ryan, or flip between two of them to compare. A theme file makes a look a portable artifact: authored in a directory, checked into git, applied to any tenant.

Three uses drive the design, in priority order:

1. **Try alternatives.** Export the current look, edit or duplicate it, load it back. Switching must be one command and must fully replace — a leftover token from the previous theme is the failure mode that makes A/B comparison worthless.
2. **Seed a new studio.** Provisioning starts from a theme rather than from Knotty Yoga's defaults. This is Phase 7's recorded `--copy-theme-from` idea, generalised: a file instead of a live tenant.
3. **Review and version.** A theme is reviewable as a text diff, and its images sit beside it.

### The bundle is a DIRECTORY, not a file

Per your ask: `theme.json` plus every binary as a **pathless sibling filename**.

```
sunrise-studio/
  theme.json
  logo.svg
  favicon.png
  hero.jpg
  home-classes.jpg
  home-community.jpg
  StudioSans-Regular.woff2
  StudioSans-Bold.woff2
```

Rules, so this stays unambiguous and safe to import from an untrusted bundle:

- Asset references are **bare filenames** — matching `^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$`. No `/`, no `\`, no `..`, no scheme, no leading dot. Anything else is rejected rather than resolved (an importer that resolves `../../etc/…` is a file-read primitive).
- A slot that legitimately holds an **external URL** keeps a full `https://…` value. The two forms are distinguished by shape: contains `://` ⇒ URL, otherwise ⇒ a file in this directory. This is why the filename pattern forbids `:` and `/`.
- Filenames are **case-insensitively unique** within a bundle (macOS/Windows would otherwise collide on import).
- Every file present must be referenced, and every reference must resolve. Both directions are validation errors, because a silent miss is exactly the bug that leaves you looking at the previous theme's logo.

**A `.zip` of that directory is the same bundle (TF1 ✅ — in the first cut).** A browser cannot be handed a folder, so the admin UI downloads and uploads a zip whose entries are exactly the files above. Two transports, one format:

| Transport | Who uses it | Why it exists |
|---|---|---|
| **Directory on disk** — `--export-theme <dir>` / `--import-theme <dir>` | the CLI helper, provisioning, git | a theme is reviewable as a text diff with its images beside it |
| **`.zip`** — Download / Upload on the admin page | a studio, Ryan, you | no shell, no server access, works from a laptop |

The zip is **flat** — no directories, matching the bundle layout. That is not cosmetic: refusing any entry whose name is not a bare filename is what closes zip-slip (`../../etc/passwd` as an entry name) without needing path-traversal logic at all. Entry count and total uncompressed size are capped, so a zip bomb is refused rather than expanded.

### Top-level shape

```json
{
  "format": "honuware.site-theme",
  "format_version": 1,
  "name": "Sunrise Studio",
  "description": "Warm palette, Barlow display, softer corners.",
  "exported_at": "2026-08-17T18:22:04Z",
  "exported_from": { "app": "knottyyoga", "site": "knottyyoga", "honuware": "1e7a3ef" },
  "theme":        { "content": { }, "tokens": { } },
  "fonts":        { "sources": [ ], "families": [ ] },
  "page_content": { "home_sections": [ ], "getting_started_steps": [ ] }
}
```

- `format` + `format_version` are the compatibility gate. An importer refuses an unknown `format`. A **lower** `format_version` is migrated forward (below); a **higher** one is refused, because a bundle from a newer build may mean things this one cannot honour.
- `exported_from` is **provenance for a human**, never used to make decisions on import. A theme from another app must still import — that is the point.
- `name` / `description` are what a future theme picker lists.
- The three payload sections mirror the three editors exactly, so "where do I change this" has one answer.

### `theme.content` — the 14 content slots

Flat map of `config_secrets` key → value. Every key in `SiteContentSlots()` plus `site_logo_url`, which is served by `/api/site_info` from its own secret rather than the slot registry.

```json
"content": {
  "site_browser_title": "Sunrise Studio",
  "site_logo_alt": "Sunrise Studio",
  "site_logo_url": "logo.svg",
  "site_favicon_url": "favicon.png",
  "site_hero_headline": "Move. Breathe. Belong.",
  "site_hero_subline": "Classes for every body, seven days a week.",
  "site_hero_image_url": "hero.jpg",
  "site_tagline_lines": ["Strength without strain", "Community without ego"],
  "site_address_lines": ["2545 152nd Ave NE", "Redmond, WA 98052"],
  "site_social_links": [
    { "label": "Instagram", "url": "https://instagram.com/sunrise" },
    { "label": "Facebook",  "url": "https://facebook.com/sunrise" }
  ],
  "site_contact_email": "hello@sunrisestudio.com",
  "site_start_intro": "New here? Start with these three steps.",
  "site_membership_blurb": "One pass, every class.",
  "site_about_markdown": "# About us\n\nWe opened in 2019…"
}
```

**The one place the file shape deliberately differs from storage.** Three slot types are stored as newline-packed strings because `config_secrets.value` is a single text column. In the file they take their natural JSON shape, because a theme file is meant to be edited by a person:

| Slot type | Stored as | In the file | Why |
|---|---|---|---|
| `line`, `markdown` | string | string | unchanged |
| `url` | string | string — bare filename **or** `https://…` | the asset rule above |
| `lines` | `"a\nb"` | `["a", "b"]` | `\n` inside a JSON string is unreadable and easy to corrupt by hand |
| `lines` (social links) | `"Instagram\|https://…"` | `[{label, url}]` | the `\|` packing is an internal storage detail, not a format anyone should have to know |

The importer packs these back to the stored form; the exporter unpacks. `site_social_links` is the only `lines` slot with the `label|url` convention, and the schema knows that by key — worth stating plainly because it is the one piece of per-key special-casing in the format.

### `theme.tokens` — all 83 design tokens

Flat map of `config_secrets` key → value, exactly as `SiteThemeTokens()` registers them. Flat rather than grouped-by-section: the group is presentation metadata that already lives in the server registry, and duplicating it into the file would let the two drift.

```json
"tokens": {
  "site_theme_palette_primary_400": "#e8743b",
  "site_theme_palette_primary_500": "#c85a26",
  "site_theme_primary": "#e8743b",
  "site_theme_on_primary": "#ffffff",
  "site_theme_radius_card": "12px",
  "site_theme_font_heading": "Barlow",
  "site_theme_text_base": "1rem",
  "site_theme_weight_bold": "700"
}
```

Full coverage, by group — this is the "capture all the settings" checklist:

| Group | Count | Keys |
|---|---:|---|
| Palette ramps | 42 | `site_theme_palette_{primary,secondary,tertiary,quaternary,quinary,grey}_{100…700}` |
| Palette extras | 2 | `…_surface_tint`, `…_surface_subtle` |
| Brand | 5 | `primary`, `on_primary`, `primary_hover`, `accent`, `on_accent` |
| Surfaces & text | 7 | `ink`, `text`, `text_muted`, `surface`, `surface_tint`, `background`, `border` |
| Header & footer | 2 | `inverse_surface`, `on_inverse_surface` |
| Status tones | 8 | `{success,warn,danger,info}` + their `on_` pairs |
| Corners | 4 | `radius_{control,panel,card,pill}` |
| Font roles | 3 | `font_{body,heading,display}` |
| Type scale | 6 | `text_{xs,sm,base,lg,xl,2xl}` |
| Weights | 4 | `weight_{regular,medium,semibold,bold}` |
| **Total** | **83** | |

A guard test asserts the exporter emits a key for every entry in `SiteThemeTokens()` and `SiteContentSlots()`, so **adding a token to the registry cannot silently fall out of the format.** That test is the real guarantee behind "captures all the settings" — a table in a document goes stale, a test does not.

### `fonts` — sources, families, and the actual font files

```json
"fonts": {
  "sources": [
    {
      "source_key": "google",
      "display_name": "Google Fonts",
      "base_url": "https://fonts.googleapis.com/css2",
      "query_suffix": "display=swap",
      "preconnects": [
        { "url": "https://fonts.googleapis.com", "crossorigin": false },
        { "url": "https://fonts.gstatic.com",    "crossorigin": true  }
      ]
    }
  ],
  "families": [
    {
      "family": "Barlow",
      "fallback": "sans-serif",
      "source_kind": "cdn",
      "source_key": "google",
      "spec": "family=Barlow:ital,wght@0,400;0,700;1,400"
    },
    {
      "family": "Studio Sans",
      "fallback": "serif",
      "source_kind": "uploaded",
      "faces": [
        { "weight": 400, "style": "normal", "file": "StudioSans-Regular.woff2" },
        { "weight": 700, "style": "normal", "file": "StudioSans-Bold.woff2" }
      ]
    },
    { "family": "Georgia", "fallback": "serif", "source_kind": "system" }
  ]
}
```

- **No ids anywhere.** Sources are identified by `source_key`, families by `family`, faces by `(weight, style)`. This is already how the server's PUT reconciles, so import reuses that path rather than inventing a second one.
- `preconnects` is structured, not the stored `url|crossorigin` newline packing — same reasoning as `lines` above.
- **Face files travel as real files** and keep their format. On import the format is re-derived from the **magic bytes**, exactly as the upload endpoint does; the extension in the filename is never trusted. A bundle carrying a renamed `.woff2` that is actually HTML is rejected.
- Family `ordinal` is **implied by array order** rather than stored. An explicit ordinal in a hand-edited file is a thing to keep consistent for no benefit.
- `source_kind` decides which fields are meaningful: `cdn` requires `source_key` + `spec`; `uploaded` requires ≥1 face; `system` requires neither. Validated on import, so a bundle cannot produce the "names a font nobody can render" state the editor warns about.

### `page_content` — home sections and Getting Started steps

```json
"page_content": {
  "home_sections": [
    {
      "kind": "hero",
      "title": "Move. Breathe. Belong.",
      "body": "Classes for every body.",
      "link_route": "/classes",
      "link_label": "See the schedule",
      "active": true,
      "image": "home-hero.jpg"
    },
    {
      "kind": "feature",
      "title": "Aerial & Acro",
      "body": "Find your feet in the air.",
      "link_route": "/classes",
      "link_label": "Browse classes",
      "active": true,
      "image": "home-classes.jpg"
    }
  ],
  "getting_started_steps": [
    {
      "mat_icon": "person_add",
      "title": "Create an account",
      "body": "Takes about a minute.",
      "link_route": "/register",
      "link_label": "Sign up",
      "hidden_when_logged_in": true
    }
  ]
}
```

- **Order is the ordinal** in both lists, same as fonts.
- `image` is a home section's uploaded photo, exported as a sibling file at its **source** resolution — scaled derivatives are a cache and rebuild themselves on demand. Absent ⇒ the section has no photo. Getting Started steps carry a `mat_icon` name, not an image, so they have no asset.
- `kind` is validated against the four known kinds (`hero`, `feature`, `banner`, `artwork`); `mat_icon` against the server's curated allow-list. An unknown value fails the import rather than seeding a row nothing renders.
- **These two tables are app-side** (`knottyyoga/src/db_schema`), not framework. See the ownership note below — it is the one structural decision this format forces.

### Import semantics

**Replace is the default, and it is the whole point.** Loading a theme must leave the tenant looking exactly like the bundle — so:

- Any registered content slot or theme token **absent** from the file is **reset to its default**, not left at the previous theme's value. Without this, flipping between two themes accumulates the union of both and neither look is what its author intended.
- `page_content` lists **replace wholesale** — the tenant ends with exactly the rows in the file, in that order.
- `fonts` reconcile by key/family (the existing PUT path), and families not in the bundle are removed along with their faces.
- `--merge` is offered as an explicit opt-in for "apply just these tokens on top of what's there", which is the mode you want while iterating on one colour.

**A theme file is not a database dump.** It carries settings, not identity or history: no ids, no `created_at_us` / `last_updated_at_us`, no `active` flags on fonts (a bundle only lists fonts it wants active).

**Import is atomic.** Validate the whole bundle — JSON shape, every asset reference, every enum, every font's magic bytes — before writing anything, inside one transaction. A half-applied theme is worse than a refused one. Same discipline as the existing `PUT /api/manage/site_theme`.

### Versioning: migrate forward, and let the caller choose how strict to be (TF3 ✅)

Two mechanisms, deliberately separate — they answer different questions.

**1. Schema migration** answers *"this bundle was written by an older build."* A registered, ordered chain of migrations rewrites the JSON from its `format_version` up to the current one before validation runs. Each migration is a pure `Json → Json` step with its own test and a one-line description of what it does.

```
Migration { from: 1, to: 2, "site_theme_brand → site_theme_primary" }
Migration { from: 2, to: 3, "hero_image slot moved into home_sections" }
```

The chain runs oldest-first; a gap in it is a startup error, not a runtime surprise. This is where a **renamed token** is handled — which is the case that actually matters, because a rename would otherwise turn every older theme file into a pile of unknown keys overnight. Migration runs before strictness is evaluated, so a bundle that only *looks* unknown because it is old is repaired rather than rejected.

**2. Strictness** answers *"this bundle has a key I still do not recognise after migrating."*

| Mode | Behaviour | Where it is the right answer |
|---|---|---|
| `strict` **(default)** | refuse the import, naming every unknown key | a typo'd token silently doing nothing is the worst outcome; also keeps `mail_app_password` out of a "theme" |
| `lenient` | apply what is understood, **report** every key skipped | recovering a hand-written or hand-edited bundle, or one from a build slightly ahead of this one |

Exposed as `--strict` / `--lenient` on the CLI (default strict) and as a checkbox on the upload dialog (default off ⇒ strict). **`lenient` never means silent**: the skipped keys are part of the import report the UI shows and the CLI prints. A mode that hides what it dropped would be worse than either option.

Both mechanisms feed the same **import report**, which is also what the dry-run endpoint returns:

```json
{
  "ok": true,
  "migrated_from": 1,
  "migrations_applied": ["site_theme_brand → site_theme_primary"],
  "unknown_keys": [],
  "skipped_sections": ["page_content"],
  "changes": { "content": 12, "tokens": 47, "font_families": 3, "assets": 5 }
}
```

### What deliberately does NOT travel

This is the security-critical half of the design. `config_secrets` holds **live credentials** — Square tokens, the SMTP password, at-rest-encrypted values. The format is an **allow-list**, never "export the config_secrets table":

- **Only** the keys registered in `SiteContentSlots()`, `SiteThemeTokens()`, and `site_logo_url` are exported. A key that is not in a registry cannot appear in a bundle, and an unrecognised key in a bundle being imported is refused (not ignored — a typo'd token key should be a visible error, and refusing keeps the door shut on `mail_app_password` arriving in a "theme").
- **Tenant identity stays put**: `website_address`, `website_address_login`, `mail_sender_name` describe *which studio this is*, not what it looks like. A theme applied to Sunrise must not rename it to Knotty Yoga. This is the trap worth naming explicitly, because `mail_sender_name` is the studio's display name and it is tempting to think of it as branding.
- No people, bookings, products, classes, or blog posts. A theme is a look.

A guard test asserts every exported key starts with `site_` and appears in a registry — the same shape as the existing guard that keeps credentials off the public `/api/site_info`.

### Round-trip is the acceptance test

> Export tenant → import into a blank tenant → export again ⇒ **byte-identical `theme.json` and byte-identical assets.**

Every ambiguity in this format shows up as a round-trip failure, which is why it is the test rather than a checklist. A second test does export → import **into the same tenant** → assert `/api/site_info` is unchanged, which catches the packing/unpacking conversions (`lines`, social links, preconnects) that a same-file comparison alone would not.

### Ownership: framework vs app (D11 tension)

`theme` and `fonts` are entirely honuware. `page_content` is entirely knottyyoga — `home_sections` and `getting_started_steps` are app tables. So the format cannot live wholly on either side.

**Proposal:** honuware owns the envelope, the asset rules, the validation, the atomic apply, and the `theme` + `fonts` sections. It exposes a registration seam for **named app sections**; knottyyoga registers `page_content`. A bundle whose app sections the current app does not recognise imports the framework half and **reports the skipped sections** rather than failing — that is what lets a CommunityFinder theme's colours and fonts be reused by Knotty Yoga even though CF has no `home_sections`.

### Resolved questions (8/17/2026)

- **OQ-TF1 — Delivery.** ✅ *"I'd like a download / upload in the admin UI as a first cut."* Both transports ship in v1: the zip via the admin page, the directory via the CLI. One format, two wrappers.
- **OQ-TF2 — Reset semantics.** ✅ *"That sounds fine."* Absent ⇒ reset to default; `--merge` is the explicit opt-in.
- **OQ-TF3 — Unknown keys.** ✅ *"Can we make 'Refuse unknown keys' an option but also allow schema migration?"* Both, as two separate mechanisms — see the versioning section above. Migration runs first and repairs old bundles; strictness then decides what to do with what is genuinely unrecognised. Default strict.
- **OQ-TF4 — Logo.** ✅ *"Yes, the logo should be included."* `site_logo_url` travels as a bundled asset.

---

## Phase 9 implementation plan

Backend before frontend throughout. Every slice is Linux-docker-green before the next starts; honuware slices need a CI push + pin bump in **both** consumers before the app can use them.

**Where things live.** `honuware_platform` owns the envelope, migrations, validation, the atomic apply, the zip codec, the `theme` + `fonts` sections, and both endpoints. knottyyoga registers `page_content` through the section seam. The CLI lives in `knottyyoga_database_helper`.

> ⚠️ `database_helper/` is **not compiled by the standard gate** (`build_and_test.sh` builds only `knottyyoga_tests`). Slice 9.7 must also run the explicit `knotty_yoga_build knottyyoga_database_helper` command — see [[reference_linux_docker_build_clients]]. A `Logging::Log()` typo shipped past a green 4745-test run this way once already.

> **9.1–9.7 BUILT 8/17/2026, shipped through CI and pinned in both consumers.** honuware 1722, knottyyoga 4984, CommunityFinder 1739.
>
> Two things the plan did not anticipate, both in the as-built note below: a **`site_assets` table** had to be added (a bundled logo had nowhere to live), and the **directory transport moved into honuware** beside the zip rather than living in the CLI.
>
> **Three boxes are qualified rather than clean** — see the ⚠️ marks inline: the migration-chain check is a test rather than a startup assertion, the `/api/site_info`-unchanged test was never written, and the CLI's own report-printing is untested. Ticking these boxes is also what surfaced a **real atomicity bug** in 9.4 and a **missing set of zip READER tests** in 9.5; both are now fixed, which is the argument for going back over a checklist rather than declaring from memory.

### 9.1 — The bundle model and its registries *(hw)* ✅

- [x] `business_logic/branding/theme_bundle.h/cpp` — the in-memory `ThemeBundle`: envelope fields, `content`, `tokens`, `fonts`, named app sections, and `assets` as `filename → bytes`. Pure data; no I/O, no SQL. This is what both transports produce and both directions consume.
- [x] `theme_bundle_assets.h/cpp` — the asset-name rule in one place: `IsValidBundleAssetName` (`^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$`), `IsBundleAssetReference` vs `IsExternalUrl` (the `://` test), case-insensitive uniqueness, and the extension for a given image/font format. Every other file asks this one.
- [x] `theme_bundle_sections.h/cpp` — the **app-section seam**: `RegisterThemeBundleSection(name, exporter, importer)` plus the lookup. Framework sections register themselves the same way, so there is one mechanism rather than a special case for `page_content`.
- [x] Tests: name rule accepts/rejects the documented shapes (`..`, `/`, `\`, `C:`, leading dot, 64+ chars, empty, `logo.png`); URL-vs-filename discrimination; duplicate-name detection differing only in case; section registry round-trip.

### 9.2 — JSON serialisation both ways *(hw)*

- [x] `theme_bundle_json.h/cpp` — `ThemeBundleToJson` / `ThemeBundleFromJson`. Owns the four storage↔file conversions: `lines` ⇄ array, `site_social_links` ⇄ `[{label,url}]`, `preconnect_lines` ⇄ `[{url,crossorigin}]`, and ordinals ⇄ array order.
- [x] Key emission is driven by `SiteContentSlots()` + `SiteThemeTokens()` + `kSiteLogoUrl` — never a hand-written list.
- [x] Tests: **the coverage guard** — every registry entry appears in an exported bundle, so a new token cannot fall out of the format silently. Plus each conversion round-tripping, including the awkward ones (a tagline containing a `|`, a social label containing a `|`, an empty `lines` value, markdown containing `\n` and `"`).

### 9.3 — Migration chain and strictness *(hw)*

- [x] `theme_bundle_migrations.h/cpp` — the ordered `Migration{from, to, description, apply}` registry, `CurrentBundleFormatVersion()`, and `MigrateBundleJson` returning the applied descriptions. v1 ships with an empty chain; the machinery exists so the first rename is a data change, not a redesign.
- [x] ~~**Startup** assertion~~ that the chain is contiguous from the oldest supported version to current — a gap is a build-time bug, not a runtime one. → **Built as a TEST** (`TheChainIsContiguousUpToTheCurrentVersion`), not a startup assertion. A test fails on the build machine; a startup assertion fails on a studio's server. Deliberate change, but it does mean a gap introduced without running the suite would reach a deploy.
- [x] `BundleStrictness{Strict, Lenient}` and the `ImportReport` struct (`migrated_from`, `migrations_applied`, `unknown_keys`, `skipped_sections`, `changes`).
- [x] Tests: a v-current bundle is untouched; a synthetic older bundle migrates and reports what ran; a higher `format_version` is refused; unknown key ⇒ refused under strict, applied-and-reported under lenient; a non-`site_` key is refused **under both modes** (that one is security, not preference).

### 9.4 — Export and import against the database *(hw)*

- [x] `theme_bundle_export.h/cpp` — read the allow-listed secrets, the three font tables, and each registered app section into a `ThemeBundle`; pull uploaded font faces and referenced photos in as assets at source resolution. Asset filenames are derived from what they are (`logo.png`, `StudioSans-700-normal.woff2`, `home-1-hero.jpg`) and de-duplicated deterministically, because a round-trip must be byte-identical.
- [x] `theme_bundle_import.h/cpp` — validate everything, then apply in one transaction: secrets (absent ⇒ delete the override so the default returns, unless merging), fonts through the existing reconcile path, app sections via the seam. Font formats re-derived from magic bytes; images re-validated as real images.
- [x] Tests: **round-trip export → import → export is byte-identical**; ~~export → import into the same tenant leaves `/api/site_info` unchanged~~; replace resets an absent token while `--merge` leaves it; a bundle naming a missing asset is refused; a `.woff2` that is really HTML is refused; import is atomic.
	- ⚠️ **The `/api/site_info`-unchanged test was NOT written.** The byte-identical round trip covers the same conversions from the other side, but the one that would prove the *rendered* payload is unchanged is missing — the same class of gap as the `--theme-primary` bug, where every test asserted the stored value and none asserted what the site actually served.
	- **Atomicity was a real bug, found only when asked to tick this box.** `RunInTransaction` **commits whenever its lambda returns normally** — it rolls back only on an exception. The import writes secrets and fonts *before* handing control to an app section, so a section that refused left the framework half committed while the response said the import failed. Fixed: the endpoint's failure path now throws and the report rides out on the exception. Test: `ARefusedImportRollsBackTheFrameworkHalf`. Nothing could have hit it yet (no section is registered until 9.8) — but 9.8 is exactly when it would have.

### 9.5 — Zip codec *(hw)*

- [x] Add **`libzip`** to `conanfile.py` in honuware **and** the app (the app's list is a superset). Not hand-rolled: the reader parses untrusted input, which is exactly the wrong place to save a dependency.
- [x] `theme_bundle_zip.h/cpp` — flat archive only. Reader refuses any entry whose name is not a bare asset name, caps entry count and total uncompressed bytes, and refuses an archive with no `theme.json`. Writer emits `theme.json` first (so a human opening the zip sees it at the top) and stores already-compressed assets rather than re-deflating them.
- [x] Tests: writer→reader round-trip; an entry named `../evil` is refused; an entry named `sub/dir.png` is refused; over-cap entry count and over-cap uncompressed size are refused; a truncated archive is refused rather than crashing.
	- ⚠️ **Initially only the WRITER was tested against path entries** — i.e. only that we do not *create* one. The reader is the side that takes untrusted input, and it had no test at all; one test was even *named* `RefusesAnArchiveWithNoThemeJson` while asserting a valid archive parses. Added `MakeRawZip` (builds entry names our own writer refuses) plus four reader tests: `../evil.png`, `sub/dir.png` and `/etc/passwd` refused, no-`theme.json` refused, over-cap entry count refused, and an oversized entry refused on its *declared* size before being expanded.

### 9.6 — Endpoints *(hw)*

- [x] `GET /api/manage/site_theme_bundle` — admin-only; `application/zip` + `Content-Disposition` naming the file after the studio and the date.
- [x] `POST /api/manage/site_theme_bundle/validate` — admin-only **dry run**: parse, migrate, validate, and return the `ImportReport` **without writing**. This is what makes "try alternatives" safe — you see what a theme will change before it changes it.
- [x] `POST /api/manage/site_theme_bundle` — admin-only apply; body is the zip, `?strict=`/`?merge=` as query flags; returns the same report.
- [x] Anchor all three in `register_framework_endpoints.cpp`. *(The Phase 6B lesson: an unanchored endpoint is dead-stripped from the production binary while every test still passes.)*
- [x] Tests: auth (anonymous 401, non-admin 403) on all three; download→upload round-trip through the endpoints; validate writes nothing; a malformed zip returns 400 with a useful message, not a 500.

### 9.7 — CLI *(app)*

- [x] `knottyyoga_database_helper --export-theme <dir>` / `--import-theme <dir>`, with `--strict`/`--lenient` and `--merge`. Export refuses to overwrite a non-empty directory unless `--force`; import prints the report.
- [x] **Build it explicitly** — see the gate warning above.
- [x] Tests: directory writer/reader round-trip; refusal to clobber (both in `theme_bundle_directory_test.cpp`, 8 tests).
- [ ] ⚠️ **"the report is printed on both paths" — NOT TESTED.** `RunExportTheme` / `RunImportTheme` themselves have no test: `knottyyoga_database_helper` is an executable with no test target, so its arg dispatch, its report printing and its `--dry_run` / `--merge` / `--lenient` wiring are covered only by compiling. The transports underneath are well tested; the CLI shell around them is not. Same exposure as the `Logging::Log()` typo that shipped past a green run — worth either a small test target or a scripted smoke run in Phase 9.10.

### As-built notes for 9.1–9.7 (8/17/2026)

**Two structural additions the plan missed.**

1. **`site_assets` — a bundled image had nowhere to live.** The URL slots (`site_logo_url`, `site_favicon_url`, `site_hero_image_url`) hold a *URL*, which was fine while Phase 5 was "imagery v1, URL-based". A bundle has to carry the image itself or a theme is not portable, so the bytes needed a home and a route. New table + `TableHelpers::SiteAssets` + `GET /api/site_asset/<name>`, mirroring `site_font_faces` — deliberately NOT the photo_instances machinery, which exists to scale derivatives of photos attached to a table row. A logo is one file, served as-is, addressed by name, belonging to no row. On export a slot pointing at `/api/site_asset/logo.png` comes out as the bare filename `logo.png`; on import the reverse. That keeps the stored value a servable URL, so nothing downstream of the store had to learn a second form.
2. **The directory transport went into honuware, not the CLI.** It is a transport like the zip, not a CLI concern — `theme_bundle_directory.{h,cpp}` sits beside `theme_bundle_zip.{h,cpp}` and both enforce the same asset-name rule, so neither is the soft way in.

**Asset content is checked, not trusted.** Fonts already derived their format from magic bytes (D14); images now do too (`ImageTypeFromMagicBytes`), and an asset that is **neither** a font nor an image is refused outright. Every asset is stored and then served back from our own origin, so a zip must not be usable as a way to host arbitrary bytes there.

**`GET /api/site_asset/<name>` uses `no-cache`, unlike `site_font_face`'s year-long cache.** A font *id* is immutable — a new upload is a new id — but an asset is addressed by NAME, so importing a different theme replaces `logo.png` in place. A held copy would be the previous studio's logo, which is the same failure the site_info cache produced in August; not one to repeat.

**Three bugs found while building, worth keeping:**
- **A null `const char*` into a `std::string` — segfault.** `Util::GetEnvWithFallback` returns **nullptr** when neither variable is set (its header says so), and the export endpoint fed that straight into `ThemeBundleExportOptions`. Every one of those vars is unset in a test environment, so this was not a rare edge — it was the default path, and it crashed the whole test binary rather than failing one test.
- **`GetAssetBytes` threw on a missing row.** `RunSqlStatementReturningOneValue` throws on zero rows, so a URL slot outliving its asset — exactly what a theme import that replaced the asset set produces — would have surfaced as a 500 on a public page instead of a missing image.
- **libzip's `zip_source_stat` after `zip_close` is not a reliable size.** Sizing the read buffer from it is how you get a wrong-length read; replaced with libzip's documented read-until-zero loop.

**A design property worth stating, because a test initially got it wrong:** the exporter is registry-driven, so it *physically cannot* emit an unknown or non-`site_` key. Unknown keys can only ever arrive from a file written elsewhere, which is why the strictness tests inject into the JSON rather than into the struct. The credential guard (a non-`site_` key is refused under **both** strictness modes) is tested on that incoming path.

**Not yet done in this slice:** the `page_content` section (9.8) is unregistered, so a bundle currently carries theme + fonts only and reports `page_content` as skipped. That is the seam working as designed, but it means an exported theme does not yet include home sections.

### 9.8 — `page_content` section *(app)*

- [ ] Register the `page_content` exporter/importer with the seam: `home_sections` (+ its photo asset per row) and `getting_started_steps`. Validate `kind` against the four kinds and `mat_icon` against the curated allow-list.
- [ ] Tests: sections and steps round-trip in order; a home section's photo survives; an unknown `kind` or icon is refused; a bundle with **no** `page_content` imports the framework half and reports it skipped.

### 9.9 — Admin UI *(app — TF1's first cut)*

- [ ] ServerAccess: `downloadSiteThemeBundle()` (blob), `validateSiteThemeBundle(file, opts)`, `uploadSiteThemeBundle(file, opts)` — interface, network, proxy, **mock**, and mock specs.
- [ ] A **Theme file** section on the Site Theme page (or its own small page if that tab gets crowded): **Download** writes the zip; **Upload** picks a file, runs the dry-run first, and shows the report — what will change, what was migrated, what is unknown — with Apply / Cancel. Strict is the default; lenient is a checkbox on the confirm step, where the consequence is visible, not buried in settings.
- [ ] Reuse `applyTheme()` / `applyFonts()` after a successful apply so the page restyles immediately — the same lesson as the `--theme-primary` fix: a save that appears to do nothing reads as broken.
- [ ] Component specs: download triggers, dry-run runs before apply, the report renders (including unknown keys and skipped sections), Cancel writes nothing, a rejected bundle shows the server's message verbatim.

### 9.10 — Proof

- [ ] Export Knotty Yoga, import into a blank second tenant, walk the Phase 7 fake-studio checklist: nothing Knotty-branded leaks, and the second tenant is visually identical to the first.
- [ ] Commit a `themes/` directory of at least two real bundles (Knotty Yoga as-is, plus one invented studio) — these become the fixtures for Phase 7's proof and the regression corpus for every future theming change.
- [ ] Live hand-testing steps per the precise-instructions rule: exact menu → page → button → file, both directions, both strictness modes.

---

# Sequencing with the other plans

- **Phase 9 (theme files) is independent of Ryan and the makeover** — it serialises settings that already exist. It does depend on Phases 1–6B being done (they are) and it **feeds Phase 7**: the fake-studio proof becomes "import this bundle" instead of "hand-enter a second brand", and Phase 7's recorded `--copy-theme-from <tenant>` idea is better served by a theme file than by tenant-to-tenant copying.
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
