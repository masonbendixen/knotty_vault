---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/17/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Take a look at [[Splitting the server up into components]]. We want to do something similar for the frontend. I don't think there are as many things to componentize on the front end but I think the following are good candidates:

- Most of the controls for the CRUD stuff under knottyyoga/ui/src/app/controls
- The auth service stuff knottyyoga/ui/src/app/core/services
- The CRUD stuff, and just the CRUD stuff, under knottyyoga/ui/src/app/pages/admin
- The auth stuff under knottyyoga/ui/src/app/pages/auth
- There is a photo editor / upload thing I think somewhere

A lot of these things use ServerAccess so we might want to create some kind of indirection layer on top of server access that calls server access and have these components call that layer. Then other clients can provide some kind of callback / interface that sits on their equivalent to server access.

Please scan the code base for other possible nuggets that are good candidates for componentization.

I'm not sure what the best mechanism is for a separate component. I imagine we will have a new repo on github that is a sibling to https://github.com/honuware/server_components. We will probably eventually have its own CI/CD pipeline. I'm not sure what is the best way to pull this component down to our existing front end and other frontends.

Eventually, I'd like to do deeper frontend componentization to have a family of frontend sites for other studios that reuse a lot of this code in the frontend but that's a bigger task and for now, I mostly want to do a fairly unrelated website that just makes use of the CRUD editor, auth, photo support, possibly payment, and other generic non fitness studio / spa functionality that is generally applicable to most any website.

Please be creative and give me a number of ideas and options. Please start an open questions / discussion section with me. Please extensively review the code base and this vault.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Current State Assessment

## Build & workspace structure today

One Angular **19.2** application project (`ui`), esbuild `application` builder. **Zero library scaffolding**: `newProjectRoot: "projects"` is configured but the folder doesn't exist, ng-packagr isn't installed, and there are no publish scripts (`ui/package/build_ui_release.sh` is an app-deployment tarball, not library packaging). The app is effectively **100% standalone components** (153 components; the only non-standalone declarable is a dead legacy pipe), which makes library extraction dramatically easier — there are no NgModules to redesign. The one real NgModule is `SharedModule`, a convenience barrel (CommonModule + forms + router + the ~23-module `MatImports` array) imported by ~98 components. Path aliases: `@core`, `@pages`, `@controls`, `@shared`, `@app`. Strict TypeScript throughout, `importHelpers` (tslib).

Three build configurations: `production`, `development` (Angular dev proxy → C++ server at `127.0.0.1:18080`; the network layer uses relative `/api/...` paths, no env base URL), and `local` — the default for `ng serve` — which **file-replaces** `ServerAccessNetwork.ts` with `ServerAccess.mock.ts`. Both files export a same-named `SERVER_ACCESS_IMPLEMENTATION_TOKEN`, so the root `SERVER_ACCESS_TOKEN` factory transparently builds `ServerAccessProxy` over whichever implementation got compiled in. Two facts matter for this plan:

- **`ServerAccessProxy` is not a passthrough.** It serializes all requests (one in flight at a time) because the C++ backend allows one transaction per session. Any client of a honuware server needs this behavior — it belongs to the shared layer, not the app.
- **The mock swap is a build-level path replacement** hard-wired to `src/app/shared/services/network/`. It survives this plan unchanged *only because* the app's `ServerAccessNetwork`/`ServerAccessMock` stay app-side (they implement the ~250-method app interface); the shared library gets its own DI-based mock story instead.

Styling: single global SCSS barrel; a **custom M2 Material theme still on indigo/pink palettes** (the brand mismatch [[Website Makeover]] plans to fix, with an M2→M3 migration); Tailwind with colors bridged to `:root` CSS variables; self-hosted D-DIN font. 121 of 152 templates use Tailwind utilities — but the *extraction candidates* are light users (mostly a `flex flex-row gap-2` wrapper); the heavy Tailwind consumers are app pages that stay behind. ESLint runs the legacy `.eslintrc.json` (an unused, non-Angular flat config coexists — consolidation debt), and **no import-boundary rules exist today**. Karma/Jasmine tests; spec coverage in the candidate set is near-universal (gaps noted below).

## Inventory — the candidate areas

| Area | What's there | Extraction readiness |
|---|---|---|
| `controls/` leaf fields | `simple-text`, `long-text`, `simple-bool`, `simple-enum`, `simple-date` — pure presentational, driven by the `ColumnDataInfo` schema descriptor, no server calls, no brand strings, all spec'd | **Cleanest candidates.** Only wrinkles: `simple-date` imports `TIME_SEGMENT_INCREMENT_MIN` from `@pages/calendar` (a controls→pages back-edge) and the `SharedModule` barrel; none implement `ControlValueAccessor` (bespoke `value`/`valueChanged` string contract) |
| `controls/` composite | `composite-control` (type-switch dispatcher over `html_input_type`), `fk-picker` (autocomplete calling `getFkOptions`/`resolveFkDisplay`) | Generic; fk-picker needs a narrow access seam instead of full `ServerAccess` |
| `controls/` CRUD containers | `table-view-control` (paginated schema-driven grid, FK display, photo thumbnails, nested child tables), `composite-row-control` (full row add/edit form incl. deferred photo upload) | The heart of the generic editor. Coupled to `@pages/admin` services (`DatabaseSchemaService`, `table-binding.utils`), hardcoded `/admin/tables/...` routes (7+ places), hardcoded `/api/get_scaled_photo/...` URLs, hardcoded `#3f51b5` button colors |
| `pages/admin` generic editor | `TableViewPage`/`TableEditPage`/`TableNewPage` (thin route plumbing over the containers), `DatabaseSchemaService` (schema cache+polling, **no spec**), `table-binding.utils`, `TableManagementService` — plus a **droppable legacy generation** (`EditDbTableComponent` + `TableEntryFormDialogComponent` on the `:tableName` route) | Thin and extractable once the controls-layer coupling is fixed. The field-behavior standardization from [[Dashboard cleanup work items]] (hidden/readonly flags, µs dates, enums, bools) all lives in this engine — it's what a new site inherits |
| `core/` auth stack | `AuthService` (memory-only state over ServerAccess; silent re-auth chain `me()` → 401 → `remember()` → `me()`), `auth.types` (AuthData union + permission helpers), five functional guards, `ErrorInterceptor` | Generic by design; hardcoded routes (`'/'`, `/login`, `/my/update-user-password`), the `'admin'` role literal, plus small debt: `udpateAuthData` typo, dead `AUTH_DATA_STORAGE_KEY`, native `confirm()` inside `doLogout`, and a real bug — `ErrorInterceptor` redirects to `/auth/login`, which doesn't exist (`/login` is correct) |
| `pages/auth` | `LoginComponent` (with the reusable `sanitizeReturnUrl` open-redirect guard + hardcoded prefix allowlist), `RegisterComponent`, `VerifyComponent` (query-secret scrubbing via `history.replaceState`). No forgot-password flow exists anywhere | Brand-free already; only route literals to parameterize |
| Photos | `photo-upload` control (drag-drop, client-side resize/re-encode to JPEG ≤4096px, deferred-upload mode for create flows, `userMode` avatar path) + `photo.types` + 5 ServerAccess methods. `photo-carousel` is read-only home-page display | Narrow and centralized — very extractable. `getHomePagePhotos` stays app-side (`home_page_photos` is an app table per the server split) |
| Network layer | `ServerAccess` interface (~250 methods, 52KB) + `ServerAccessNetwork` (100KB) + `ServerAccessMock` (253KB + 287KB spec) + serializing proxy | The *interface subsets*, the *proxy mechanism*, and *fresh framework impls* extract; the app's three giant files stay |
| Misc nuggets | `ToastService` (clean MatSnackBar wrapper), `ConfirmDialogComponent`, RFC 7807 stack (`ApiError.ts` + `ErrorService` + interceptor), `CsrfInterceptor` (double-submit `csrft` → `X-CSRF-Token`), `fuseAnimations`, `DateFormatting` µs utils, `SquarePaymentService` (Web Payments SDK loader/tokenizer — but imports `environment` directly) | All strong foundation citizens |
| Stays app-side | header/footer/`HeaderService` (branded; de-hardcoding is tenancy Phase 7's job), `CartService`, `about.ts`, all domain pages (shop/account/manage/staff/calendar/public), domain widgets (`event-session-card`, `seat-assignment`, `book-for-selector`, price-override editor, occupancy badge, signup-reminder button), `card-picker`/`payment-method` (see OQ9), `photo-carousel` | — |

[[Component Inventory for Designer]] already catalogs these same pieces for Figma and flags the future promotion candidates (Badge/Pill — re-implemented bespoke 10+ times, Page Header/Back Nav, Date Strip/Week Navigator, Empty State). Those aren't standardized components yet; they enter the library later via the promotion pipeline (see Ideas), not in the first extraction.

## Coupling analysis — what blocks extraction today

1. **The fat interface.** Everything injects the full ~250-method `ServerAccess` via `SERVER_ACCESS_TOKEN`, even `fk-picker` which uses 2 methods. There is no narrow seam a foreign app could implement. This is the indirection layer the Overview asks for — and it's the frontend twin of the server's `EndpointAuthHelper` façade generalization (Phase 1.6 there).
2. **controls → pages back-edges.** `simple-date` → `@pages/calendar` (a constant); `table-view-control`/`composite-row-control` → `@pages/admin/services` (`DatabaseSchemaService`, `table-binding.utils`) — creating a `pages/admin` ⇄ `controls` loop.
3. **core → pages back-edge.** `core/services/auth.types.ts` imports `UserRole` enum from `@pages/admin/types`.
4. **Hardcoded routes inside candidates.** `/admin/tables/...` navigation baked into both CRUD containers and the three table pages; `/login`, `/`, `/my/update-user-password` in guards/interceptor/AuthService; the `sanitizeReturnUrl` prefix allowlist; `/my/cards` inside `payment-method` (stays app-side, noted for completeness).
5. **Hardcoded API URLs bypassing ServerAccess.** `/api/get_scaled_photo/{table}/{id}/{w}/{h}` built by string concat in `photo-upload` and `table-view-control`.
6. **`environment` import inside `SquarePaymentService`** — libraries can't import a consumer's environment file.
7. **`SharedModule` barrel imports** inside candidates (`simple-date`; also `payment-method`) — a library can't export the app's barrel; moved components need explicit imports.
8. **Styling ties.** A handful of Tailwind utilities in candidate templates; hardcoded hex button colors in `composite-row-control`; the global M2 theme assumption; the mat-card border rule duplicated per-component (the 65-file problem [[Website Makeover]] solves with `.surface-card`).
9. **Local-mode mechanism** is the app-path file replacement (§ above) — the library needs a DI-based mock story instead.

**Quality debt inside the candidate set** (fix before/while moving, never after): missing `DatabaseSchemaService` spec; `confirm-delete-dialog` has no spec + inline template and duplicates the shared `ConfirmDialogComponent`; duplicate date-helper trio (`date-functions.service.ts` / `date.functions.ts` / `DateFormatting.ts`); dead files (`ItemInListPipe`, `TypeMetadata.d.ts`, `Person.d.ts`, admin `user.service` stub, `AUTH_DATA_STORAGE_KEY`); the `/auth/login` interceptor bug; the `udpateAuthData` typo; the legacy admin editor generation still routed.

## The ServerAccess surface: framework vs app

Mirroring the server's framework/app table split, the ~250-method interface splits cleanly. **Framework subset = 25 methods:**

- **CRUD (12):** `addItem`, `addItemFetchPrimaryKey`, `getTableRows`, `getRowsByColumn`, `getFilteredTableRows`, `getRow`, `getRowByValues`, `updateItem`, `deleteItem`, `getDbSchema`, `getFkOptions`, `resolveFkDisplay`
- **Auth/account (9):** `register`, `verify`, `login`, `logout`, `me`, `remember`, `getUserInfo`, `setUserInfo`, `updateUserPassword`
- **Photos (4):** `uploadPhoto`, `uploadUserPhoto`, `deletePhoto`, `hasPhoto`

Everything else (payments, subscriptions, scheduling, classes, providers, skills, check-in, `getHomePagePhotos`, …) is app-specific and stays. Server-side, those 25 methods hit endpoints that live in `honuware_platform` — **the client library is the matching half of endpoints the framework already standardizes.** That's the key reuse argument: any site running a honuware server gets working CRUD + auth + photos UI with zero server work.

## Constraints & coordination with other active plans

- **[[Splitting the server up into components]] (done through extraction).** `github.com/honuware/server_components` exists and is consumed via FetchContent SHA-pinning. Its Q7 explicitly deferred frontend sharing to *this* job, "to be designed together with the multi-tenant frontend/branding question." Decisions that carry over unchanged: the `honuware` name/org, components public on GitHub, Apache-2.0 + NOTICE, fresh git history, one-repo/one-version/many-targets granularity.
- **[[Converting the server to a multi tenant Saas architecture]].** The frontend strategy for more studio sites is decided (its Q10): **one shared Angular bundle + runtime branding**, per-tenant CloudFront distributions serving the same artifact. `X-Honuware-Site` is injected by CloudFront at the `/api/*` behavior — **the SPA never sets it, so the library needs zero tenant awareness.** Its Phase 7 adds `/api/site_info` (a `honuware_platform` endpoint) + an app-side `SiteConfigService` to de-hardcode header/footer/about. Consequence for this plan: library components take branding via DI with static defaults now; when tenancy Phase 7 lands, `SiteConfigService` becomes the provider of that same token. We must not build against `/api/site_info` before it exists.
- **[[Website Makeover]].** Owns design tokens (two-layer), the M2→M3 re-theme, `.surface-card`, and DB-driven per-tenant theming (`/api/site_theme`). This plan does **not** duplicate any of that; it only (a) removes hardcoded colors from extracted components in favor of CSS custom properties with fallbacks, so the makeover can restyle the library without code changes, and (b) **sequencing decided (Q5): extraction first** — the makeover then restyles the library like any other folder via ordinary version bumps, with the co-dev override making the cross-repo stretch painless.
- **CommunityFinder — the first consumer (decided, Q11).** The "fairly unrelated website" from the Overview is the CommunityFinder project, planned in its own vault (`C:\Users\mason\Documents\Obsidian\CommunityFinder\Claude\Setting up the project.md`). It needs CRUD editor + auth + photos + possibly payment — exactly the framework subset — and it is the effort driving this plan. It plays the proving-consumer role the example server played for `server_components` (Phase 5); once this document is final, Mason folds the `@honuware/ui` consumption model into that project's doc.

# Target Component Architecture

## Proposed layered entry points

**One npm package with layered secondary entry points** — the frontend expression of the server's Option C (one repo, one version, multiple targets). ng-packagr builds each secondary entry point as a separately importable unit and **fails the build on cycles between entry points**, giving the same build-enforced layering CMake target link rules give the server. Decided (Q1): repo `honuware/web_components`, package `@honuware/ui`.

```
@honuware/ui/foundation   ToastService, ConfirmDialogComponent, fuseAnimations,
                          µs-date utils (DateFormatting), sanitizeReturnUrl(allowlist),
                          [later, via promotion: badge, page-header, date-strip,
                          empty-state — after Website Makeover standardizes them]
        ▲
@honuware/ui/access       CrudAccess / AuthAccess / PhotoAccess interfaces + injection
                          tokens; framework DTOs (DataResults, DatabaseSchema,
                          TableSchema, ColumnDataInfo + JSON conversion, ForeignKeyInfo,
                          admin.types, UserInfo/LoginInfo/…, photo.types); RFC 7807
                          (ProblemDetails, ErrorTypes, ErrorService); CsrfInterceptor;
                          the request-serializing access proxy; default HTTP
                          implementations against honuware endpoints (base-path token);
                          PhotoUrlBuilder
        ▲
@honuware/ui/controls     simple-text, long-text, simple-bool, simple-enum,
                          simple-date, composite-control, fk-picker
        ▲
@honuware/ui/photos       photo-upload (drag-drop, client resize, deferred mode,
                          userMode avatar path)
        ▲
@honuware/ui/auth         AuthService + AuthData/permission helpers, the five guards,
                          ErrorInterceptor, login/register/verify pages,
                          AUTH_ROUTES + branding tokens, tryTokenLogin APP_INITIALIZER
        ▲
@honuware/ui/crud         DatabaseSchemaService, table-binding utils,
                          table-view-control, composite-row-control,
                          TableViewPage / TableEditPage / TableNewPage,
                          route-base token + route factory (+ optional table-picker
                          admin shell)
──────────────────────────────────────────────────────────────────────
side entries:
@honuware/ui/square       SquarePaymentService behind a SQUARE_CONFIG token
                          (depends on nothing internal — mirrors honuware_square)
@honuware/ui/testing      in-memory framework mocks (schema-driven CRUD store, auth
                          session sim, photo sim), provider helpers for mock mode,
                          spec utilities (depends on access + auth types — mirrors
                          honuware_testing)
──────────────────────────────────────────────────────────────────────
knottyyoga app (stays)    ServerAccess (~225 app methods) + ServerAccessNetwork/Mock/
                          proxy + file-replacement local mode; header/footer/
                          HeaderService + branding; cart; all domain pages and widgets;
                          card-picker / payment-method; photo-carousel; route tables,
                          provider wiring, environments, theme
```

**Layering rule:** an entry point may import only entry points above it in the diagram (foundation is the bottom). `controls`/`photos`/`auth` are mutually independent; `crud` sits on all of them (`DatabaseSchemaService` legitimately refreshes on auth changes — a downward edge, like the server's "services sit above data" rule). Enforced twice: ng-packagr cycle detection + ESLint boundary rules (Phase 2.3).

## The access indirection layer (the load-bearing design)

This is the Overview's "indirection layer on top of ServerAccess," designed so **knottyyoga adapts with ~zero adapter code** while new sites get a working client for free:

1. **Interface segregation, structurally free.** The library defines `CrudAccess` (12 methods), `AuthAccess` (9), `PhotoAccess` (4) — signatures copied verbatim from today's `ServerAccess`. The app then declares `export interface ServerAccess extends CrudAccess, AuthAccess, PhotoAccess { …app methods… }`. TypeScript's structural typing makes this a compile-time *proof of subset* with zero runtime change; if the library evolves a signature, the app fails to compile — drift is impossible.
2. **Tokens, not classes.** `HONUWARE_CRUD_ACCESS`, `HONUWARE_AUTH_ACCESS`, `HONUWARE_PHOTO_ACCESS` injection tokens. Library components/services inject only these. Knottyyoga provides all three via factories resolving to its existing `SERVER_ACCESS_TOKEN` proxy — so every framework call keeps flowing through the app's serializing queue, the mock file-replacement keeps working, and `ServerAccessNetwork`/`ServerAccessMock` don't move.
3. **Default implementations for everyone else.** The library also ships `CrudHttpAccess`/`AuthHttpAccess`/`PhotoHttpAccess` (HttpClient against the standard honuware endpoints, `withCredentials`, base path from a `HONUWARE_API_BASE` token defaulting to `/api`) plus the generalized serializing proxy. A new site writes no network code: `provideHonuwareAccess({ mode: 'http' })` wires proxy + impls + tokens; `{ mode: 'mock' }` wires the in-memory mocks from `/testing` — DI-based, no file replacement, tree-shaken out of production builds.
4. **`PhotoUrlBuilder`.** Scaled-photo `<img>` URLs (`/api/get_scaled_photo/...`) become a service call so no component string-concats API paths.
5. **Other clients** (the Overview's "callback/interface on their equivalent to server access"): any consumer with its own transport — a different backend, GraphQL, whatever — implements the three small interfaces and provides the tokens. The contract is 25 methods, not 250.

## Granularity options

- **Option A — one flat package, no entry points.** Simplest, but a site wanting only `ToastService` drags in the CRUD editor's compiled output, and nothing enforces internal layering. Rejected.
- **Option B — separate npm packages per layer** (`@honuware/ui-foundation`, `@honuware/ui-auth`, …). Precise, but version-bump ceremony across 6–8 packages while the API churns weekly — the same "too many packages too early" trap the server plan rejected. Rejected.
- **Option C — one package, layered secondary entry points, one version.** Consumers import only the entries they use (tree-shaking keeps bundles honest), layering is build-enforced, and splitting into Option B later is mechanical because the entry-point boundaries *are* the package boundaries.

**Decided (Q1): Option C** — repo `honuware/web_components`, one npm package `@honuware/ui`, layered secondary entry points, selector prefix `hw-`.

# Repository, Packaging & Distribution

## Hosting

Carried from server Q2: **public GitHub repo under the `honuware` org**, sibling of `server_components`; the knottyyoga app stays private on GitLab. **Decided (Q1): repo `honuware/web_components`** — mirrors `server_components`. Apache-2.0 + NOTICE, fresh history with an "extracted from knottyyoga at SHA …" note (server Q5/Q11 precedents).

## Distribution options — how consumers pull it down

This is where the frontend story is *simpler* than C++: a free, zero-auth public registry exists, so the FetchContent workaround isn't needed for distribution.

- **N1 (decided, Q2 — from day one) — publish `@honuware/ui` to the public npm registry.** Real semver, `npm install @honuware/ui@0.4.0`, lockfiles, provenance attestation from GitHub Actions. Free for public scoped packages; requires claiming the `honuware` npm org once (verify name availability at implementation). Consumers (friends' sites) do nothing special.
- **N2 — GitHub Packages npm registry.** Rejected: GitHub Packages requires an auth token to install npm packages **even when they're public** — permanent friction for every consumer for zero benefit.
- **N3 — GitHub Release tarball URLs.** CI attaches `ng build` + `npm pack` output to each release; consumers depend on the URL (`npm i https://github.com/honuware/web_components/releases/download/v0.1.0/honuware-ui-0.1.0.tgz`). Registry-free, pin-by-URL — the closest FetchContent analog. Viable interim before the first npm publish; loses semver ranges.
- **N4 — git dependency with a `prepare` build.** Rejected: an Angular workspace's repo root isn't the publishable package (the built `dist/honuware-ui` is), and build-on-install is slow and fragile — the frontend equivalent of the rejected submodule option.
- **N5 — source consumption via tsconfig paths** at a pinned checkout. Not a distribution mechanism — it's the co-development mode, below.

**Decided (Q2): N1 from day one.** Mason's call: if the goal is public npm anyway and the API won't rev that much, skip the tarball interim — publish `@honuware/ui` starting at `0.1.0` and rev freely under 0.x semantics (no stability promise until 1.0). CI publishes on tag with provenance (Phase 4.2); claiming the `@honuware` npm org is a Phase 4.1 item.

## Local co-development — the `FETCHCONTENT_SOURCE_DIR` analog

Day-to-day, you and the friends will edit library + consumer together. The documented override: a commented block in the consumer's `tsconfig.json` mapping `@honuware/ui/*` → `../../web_components/projects/honuware-ui/*/src/public-api` (exact mappings per entry point, written down in both READMEs). With the block active, the app compiles the library *source* directly — edit both repos, one build, no publish cycle; remove the block to return to the pinned npm version. Requires the same Angular major on both sides (which the version policy guarantees). This is deliberately the same muscle memory as `FETCHCONTENT_SOURCE_DIR_HONUWARE` on the server side.

# Styling & Theming Strategy

The extracted components must render correctly in a consumer that has *none* of knottyyoga's global styles. Options:

- **S1 — library stays Tailwind-dependent.** Ship a Tailwind preset; every consumer must run Tailwind with a content glob over `node_modules/@honuware/ui/**/*.mjs` and adopt the CSS-var palette. Works, standard practice — but forces Tailwind (and our config) on every consumer forever.
- **S2 (decided, Q4) — de-Tailwind the library components.** The candidate set uses only a handful of utilities (essentially the `flex flex-row gap-2` field wrapper; the CRUD containers and photo-upload are already custom-SCSS). Move those few rules into component SCSS during Phase 1. The library becomes CSS-framework-agnostic: consumers need **only an Angular Material theme**. Knottyyoga and any new site remain free to use Tailwind in their own pages.
- **S3 — compile Tailwind into the library's CSS at build time.** Rejected: build complexity and rule duplication for no gain over S2 at this scale.

Cross-cutting styling rules regardless of option:
- **Material theme is a documented consumer requirement** (any M2/M3 theme works; the library README shows a minimal setup). The library never ships a theme of its own — knottyyoga's M3 re-theme stays a [[Website Makeover]] deliverable.
- **No hardcoded colors in library components.** The `#3f51b5`/`#e0e0e0` buttons in `composite-row-control` become Material buttons (or `var(--honuware-*, fallback)` custom properties). This is what lets the makeover's token system — and later per-tenant `/api/site_theme` values — restyle the library **without a library release**.
- Library cards adopt one `.surface-card`-compatible class (same name the makeover chose) shipped in a small optional `@honuware/ui/styles` SCSS partial, so the 65-file border duplication is not re-exported to every new site.
- Fonts stay consumer-side (D-DIN is brand).

# Cross-Cutting Decisions

- **Naming (decided, Q1):** repo `honuware/web_components`, package `@honuware/ui`; component selector prefix changes `app-` → **`hw-`** as components move (a public library exporting `app-*` selectors invites collisions). The rename is mechanical (bounded template edits in composite-control, the table pages, manage pages, account page; specs catch misses) and happens once, at move time (Phase 2.2).
- **License:** Apache-2.0 + NOTICE (server Q5 precedent — same contributors, same productization logic).
- **Versioning:** single version for the whole package, tags, `0.x` until a second consumer exists; knottyyoga pins exact versions and upgrades deliberately.
- **Angular version policy (decided, Q3):** upgrade the knottyyoga workspace to the current stable major **now**, before any library work — new Phase 1.0 (verify the exact current major + Material availability when starting; v19's LTS window ends around Nov 2026 and fresh `ng new` apps will scaffold newer). The library's peerDependencies target the major it's born on; consumers match; thereafter the library upgrades majors first, consumers follow.
- **Testing policy:** every entry point ships its specs; the `/testing` mocks are themselves tested (the "test utilities have tests" rule); CI runs headless Chrome. Per the repo-wide rule: every Phase item below includes its tests in the same change.
- **Boundary enforcement:** consolidate to one Angular-aware ESLint flat config, then `no-restricted-imports` zones: library code may not import `@app/@core/@pages/@shared` or any app-relative path; entry points may only import downward; the app may import the library freely. ng-packagr's cycle check backs this at build time.
- **Branding/tenancy alignment (decided, Q12):** a `HONUWARE_BRANDING` token (studio/site display name, logo URL, website URL) with static per-app values now; when the tenancy plan's Phase 7 `SiteConfigService` exists, it becomes the provider of this token. The library never reads `X-Honuware-Site` (CloudFront injects it) and never calls `/api/site_info` until that endpoint exists — no premature upgrade-path code.

# Ideas Beyond the Core (a menu, not commitments)

1. **Showcase app as living documentation** (Phase 5.1): an application project inside the component workspace exercising every entry point, running on the `/testing` mocks by default — every PR to the library visibly demos itself. Lighter than Storybook and doubles as the new-site template. (Storybook remains an option later if the friends want isolated component docs.)
2. **`ng add @honuware/ui` schematic** (Phase 5.3 stretch): scaffolds provider wiring, theme include, `proxy.conf.json`, guard-wired route stubs — the "new site in an afternoon" story, and the frontend twin of the server's example-server onboarding.
3. **Full-stack pairing with the honuware example server** (server plan Phase 5.1): the showcase app pointed at the example server = an end-to-end integration environment for both repos; a small Playwright suite in the component repo could drive login → CRUD → photo upload against it.
4. **`honuware/site_template` repo**: a degit-able starter (showcase minus demo content) friends clone to start a site. Cheaper than a schematic, can precede it.
5. **Component promotion pipeline**: as [[Website Makeover]] standardizes Badge, Page Header/Back Nav, Date Strip, Empty State, Sticky Action Bar, promote each into `@honuware/ui/foundation` — the library grows by graduation, not big-bang.
6. **npm provenance + `sideEffects: false`**: free supply-chain credibility and optimal tree-shaking for a public package.
7. **Angular Elements build** (custom-elements bundle so non-Angular sites could embed the CRUD editor or photo-upload): genuinely creative reuse, **deferred** — payload and Material/DI friction outweigh demand until a non-Angular consumer actually exists.
8. **Nx workspace**: rejected for now — plain Angular CLI + ng-packagr covers one library + one showcase; Nx's graph/caching earns its ceremony only with many projects.

# Phased Implementation Plan

Phases 1–3 happen **entirely inside the knottyyoga repo** — no new repos, no packaging, no behavior changes; the app ships after every item. Phase 4 is the extraction; Phase 5 proves reuse. Within each phase, lower layers first. Every item lands with its tests in the same change. (Angular tests run with `npx ng test` from `ui\`; no C++ or git involvement anywhere in this plan — where a step needs a repo created or pushed, it's marked **[Mason]**.)

## Phase 1 — Break the coupling (in-place refactors)

### 1.0 Angular platform upgrade (decided, Q3 — do this first)
- [ ] Verify the current stable Angular major + matching Material/CDK releases (implementation-time check), then upgrade the `ui` workspace stepwise — `ng update` one major at a time from 19, Material/CDK in lockstep — applying each major's migration schematics and release-note items (builder/Karma/TypeScript versions ride along).
- [ ] After each major hop: `npx ng lint`, full `npx ng test`, and all **three** `ng build` configurations green (production, development, and `local` — the mock file replacement is the piece most worth re-verifying per hop); fix any deprecations the migrations surface before taking the next hop.

### 1.1 Hygiene + dead code (foundation)
- [ ] Fix the `ErrorInterceptor` redirect `/auth/login` → `/login`; add a spec case pinning the redirect target.
- [ ] Consolidate the date-helper trio: migrate any real usages of `date-functions.service.ts` and `date.functions.ts` onto `shared/utils/DateFormatting.ts` (after a reference sweep), delete the leftovers; specs updated/added.
- [ ] Consolidate confirm dialogs: retire `controls/table-view-control/confirm-delete-dialog` in favor of the shared `ConfirmDialogComponent` (parameterized title/description/button text); update `table-view-control` + spec; delete the stray (which had no spec and inline templates).
- [ ] Purify logout: move the native `confirm()` + `'/'` navigation out of `AuthService.doLogout` into the UI caller (header sign-out path, using `ConfirmDialogComponent`); `AuthService.logout()` stays pure state+server; specs on both sides.
- [ ] Fix the `udpateAuthData` → `updateAuthData` typo (all references); remove the dead `AUTH_DATA_STORAGE_KEY`.
- [ ] Delete dead files after reference verification: `pages/admin/pipes/item-in-list.pipe.ts` (+spec), `pages/admin/types/TypeMetadata.d.ts`, `pages/admin/types/Person.d.ts`, `pages/admin/services/user.service.ts` (+spec).
- [ ] Move the `UserRole` enum from `@pages/admin/types` into `core/` (kills the core→pages back-edge); update `auth.types` + any admin importers; specs still green.

### 1.2 Access layer (data) — the indirection seam
- [ ] Create the narrow interfaces + tokens (new `src/app/access/` staging area): `CrudAccess`, `AuthAccess`, `PhotoAccess` with signatures lifted verbatim from `ServerAccess`; declare `ServerAccess extends CrudAccess, AuthAccess, PhotoAccess` (compile-time subset proof); `HONUWARE_CRUD_ACCESS`/`HONUWARE_AUTH_ACCESS`/`HONUWARE_PHOTO_ACCESS` tokens defaulting (via factory) to the existing `SERVER_ACCESS_TOKEN` proxy. Zero behavior change. Specs: each token resolves to the proxy; a type-level conformance test.
- [ ] Add `HONUWARE_API_BASE` (default `'/api'`) + `PhotoUrlBuilder` (`scaledPhotoUrl(table, id, w, h)`); specs incl. a custom base path.
- [ ] Migrate `fk-picker` to `HONUWARE_CRUD_ACCESS` (it needs only `getFkOptions`/`resolveFkDisplay`); spec updated to mock the narrow interface — the first proof the seam works.

### 1.3 Controls (controls layer)
- [ ] `simple-date`: replace the `@pages/calendar` `TIME_SEGMENT_INCREMENT_MIN` import with an `@Input() timeIncrementMinutes` (default 20) — callers that need the calendar constant pass it; drop the `SharedModule` barrel for explicit Material imports; specs.
- [ ] Sweep the remaining extraction-set controls off `SharedModule` onto explicit imports (audit: the five leaf controls, composite-control, fk-picker); specs stay green.
- [ ] `composite-row-control`: replace the hardcoded `#3f51b5`/`#e0e0e0` submit/cancel buttons with Material buttons (theme-driven); visual parity check in the running app; spec.
- [ ] De-Tailwind the extraction-set control templates (the `form-fields flex flex-row gap-2` wrapper and friends → component SCSS) — decided S2 (Q4); the library ends Tailwind-free.

### 1.4 Photos
- [ ] `photo-upload`: migrate to `HONUWARE_PHOTO_ACCESS` + `PhotoUrlBuilder` (removes the literal `/api/get_scaled_photo/...`); specs updated (upload, defer mode, userMode, delete, URL building).

### 1.5 Auth (services layer)
- [ ] Introduce the `AUTH_ROUTES` config token `{ loginPath: '/login', postLogoutPath: '/', mustChangePasswordPath: '/my/update-user-password', returnUrlAllowlist: ['/my','/admin','/manage','/staff','/calendar','/shop'] }` with today's values as defaults; consume it from all five guards, `ErrorInterceptor`, `AuthService`, and `LoginComponent` (`sanitizeReturnUrl` takes the allowlist as a parameter); specs cover defaults + an overridden config.
- [ ] Migrate `AuthService` and the three auth pages to `HONUWARE_AUTH_ACCESS`; specs.
- [ ] Verify the three auth page templates are brand-free (they are, per audit — pin it with a no-brand-literal assertion in specs, mirroring the server's mail-branding tests).

### 1.6 CRUD editor (platform layer)
- [ ] Break the `controls` ⇄ `pages/admin` loop: move `DatabaseSchemaService` + `table-binding.utils` out of `pages/admin/services` into the framework staging area (`src/app/crud/`); update all importers (both CRUD containers, the three table pages, `pages/manage` consumers). **Write the missing `DatabaseSchemaService` spec** (cache, polling, auth-triggered refresh).
- [ ] Migrate `table-view-control`, `composite-row-control`, and the three table pages to `HONUWARE_CRUD_ACCESS` (+ `PhotoUrlBuilder` for the 50×50 thumbnails); specs.
- [ ] Parameterize editor routing: a `CRUD_EDITOR_ROUTES` token `{ basePath: '/admin/tables', adminHome: '/admin' }`; replace the 7+ hardcoded `router.navigate`/`routerLink` sites in the containers and table pages; specs cover default + custom base.
- [ ] Remove the legacy generation — `EditDbTableComponent`, `TableEntryFormDialogComponent`, the `:tableName` route (+ their specs), and `TableManagementService` if the admin shell's table picker is its last consumer (rework the shell onto `DatabaseSchemaService` directly) — after a reference sweep. Decided (Q8): delete, don't port.

### 1.7 Square (side)
- [ ] `SquarePaymentService`: replace the direct `environment` import with a `SQUARE_CONFIG` token (`applicationId`, `locationId`, `scriptUrl`) provided in `app.config.ts` from the environment; specs (config injection, script URL selection).

## Phase 2 — Draw the boundary (workspace library, still one repo)

### 2.1 Library scaffold
- [ ] Add ng-packagr; generate `projects/honuware-ui`; configure the eight secondary entry points (`foundation`, `access`, `controls`, `photos`, `auth`, `crud`, `square`, `testing`) each with its `ng-package.json` + `public-api.ts`; tsconfig paths `@honuware/ui/*` → library source; wire lib `test`/`lint` targets. App untouched so far.

### 2.2 Move code, lower layers first
- [ ] Move **foundation** (Toast, ConfirmDialog, fuseAnimations, DateFormatting, sanitizeReturnUrl); update app imports; selector rename `app-` → `hw-` for moved components with mechanical template updates; all specs move + stay green.
- [ ] Move **access** (interfaces, tokens, DTO types incl. `ColumnDataInfo`+conversion, ApiError + ErrorService, CsrfInterceptor, PhotoUrlBuilder); app's `ServerAccess` now imports the sub-interfaces from `@honuware/ui/access`; **new code:** the generalized serializing proxy + `CrudHttpAccess`/`AuthHttpAccess`/`PhotoHttpAccess` + `provideHonuwareAccess()` — fully spec'd with `HttpTestingController` (knottyyoga keeps its own proxy/impls; these are for new consumers).
- [ ] Move **controls** (5 leaf + composite + fk-picker); update composite/table/manage-page templates for `hw-` selectors; specs green.
- [ ] Move **photos** (`photo-upload`); update its five consumer sites; specs green.
- [ ] Move **auth** (AuthService, auth.types, guards, ErrorInterceptor, three pages, AUTH_ROUTES/branding tokens, `tryTokenLogin` initializer factory); app routes/`app.config.ts` import from `@honuware/ui/auth`; specs green.
- [ ] Move **crud** (DatabaseSchemaService, binding utils, containers, three pages, route token/factory); admin routes import from `@honuware/ui/crud`; specs green.
- [ ] Move **square** (`SquarePaymentService` + `SQUARE_CONFIG`); `card-picker`/`payment-method` (app-side) import it from `@honuware/ui/square`; specs green.
- [ ] Build **testing**: fresh, minimal in-memory framework mocks — a schema-driven `CrudAccess` store (tables/rows/FK options honoring `DatabaseSchema`), an `AuthAccess` session simulator (login/logout/me/remember/register/verify), a `PhotoAccess` simulator — plus `provideHonuwareAccess({ mode: 'mock' })`; the mocks get their own specs. (Knottyyoga's 253KB `ServerAccessMock` stays untouched app-side — it must mock all ~250 methods regardless; optionally refactor it later to delegate its framework subset to these mocks — decided, Q10: fresh mocks now, delegation stays an optional later cleanup.)

### 2.3 Enforcement
- [ ] Consolidate ESLint to a single Angular-aware **flat config** (port the legacy `.eslintrc.json` rules incl. selector prefixes — now `app` for the app, `hw` for the lib — and delete the stale config); add the boundary rules: library entries may not import app paths; entry-point imports only flow downward; run against both projects, fix any stragglers the rules catch.

### 2.4 Proof (still one repo)
- [ ] `ng build honuware-ui` green (partial-Ivy); app `ng build` green in **all three** configurations — explicitly re-verify the `local` mock file-replacement still functions (the swapped files never moved); full `npx ng test` green across app + library.

## Phase 3 — Dry-run the packaging (still one repo)

### 3.1 Consume the built artifact
- [ ] CI-style rehearsal: `ng build honuware-ui` → `npm pack` the dist; temporarily point the app's tsconfig paths at `dist/honuware-ui` and build + test the app against the **built** library (this is what catches deep imports, missing `public-api` exports, packaging of styles/assets, and peer-dep gaps); revert to source paths for daily dev; document both modes in `ui/README`/CLAUDE.md.

### 3.2 Package metadata
- [ ] Finalize `peerDependencies` (`@angular/core|common|forms|router`, `@angular/material` + `cdk`, `rxjs`, `tslib`) with correct ranges; `sideEffects` audit; per-entry README sections; the consumer styling doc (Material theme requirement, the CSS custom properties the library reads, the optional `styles` partial).

## Phase 4 — Extract to the shared repo

### 4.1 Repo creation
- [ ] **[Mason]** Create `github.com/honuware/web_components` (fresh history) and claim the `@honuware` org on the public npm registry (needed before the first publish — verify name availability; decided Q2). Then: scaffold the workspace on the current stable Angular major (the app already matches after 1.0), copy `projects/honuware-ui`, port the flat-config lint + boundary rules + headless Karma, add `LICENSE` (Apache-2.0) + `NOTICE` + `README` + `CONTRIBUTING` + the "extracted from knottyyoga at SHA …" note; full lint+test+build green.

### 4.2 CI
- [ ] GitHub Actions: PR/push → `npm ci`, lint, headless-Chrome tests, `ng build honuware-ui`, `npm pack`, upload the tarball artifact; on tag → `npm publish` to the public registry with provenance (decided, Q2 — npm from day one, 0.x). This workflow doubles as the CI template for the friends' sites (same role the server repo's CI plays).

### 4.3 Consumption from knottyyoga
- [ ] Remove `projects/honuware-ui` from the app repo; add the pinned `@honuware/ui` dependency (exact version); delete the source path aliases; commit the documented-but-commented local co-dev paths override (the `FETCHCONTENT_SOURCE_DIR` analog); all three app build configs + the full suite green against the published package. **[Mason]** pushes both repos.

## Phase 5 — Prove reuse (CommunityFinder + showcase)

### 5.1 Dev-harness / showcase app
- [ ] `projects/showcase` app in the component repo — **a dev harness + living documentation, not the seed of a real site** (decided, Q11: CommunityFinder is the real first consumer, in its own repo): login/register/verify flow, the generic data editor over the honuware framework tables, photo upload, toast/confirm/error handling — running on `@honuware/ui/testing` mocks by default (`ng serve` works with zero backend), with a `-c server` configuration proxying to the honuware example server (server plan Phase 5.1) for full-stack integration. Still the friends' quick-start template. Component specs for the showcase's own pages.

### 5.2 CommunityFinder consumption + docs (the real proof)
- [ ] Quickstart: "new site from `ng new`" — install, `provideHonuwareAccess`, theme include, proxy.conf, routes + guards wiring, branding token; a "which entry point do I need" table; entry-point API notes. Written against the showcase, validated by CommunityFinder.
- [ ] **CommunityFinder onboarding (decided, Q11):** its plan lives at `C:\Users\mason\Documents\Obsidian\CommunityFinder\Claude\Setting up the project.md` — once this document is final, Mason folds the `@honuware/ui` consumption model into that doc. CommunityFinder is the extraction's driving consumer; gaps or friction it hits flow back here as work items, and the extraction counts as *proven* when CommunityFinder ships auth + CRUD + photos on the published package.

### 5.3 Stretch
- [ ] `ng add @honuware/ui` schematic automating 5.2's wiring.

# Alternatives Considered (and why not)

- **Copy-paste the pieces into the new site.** Divergence within months, twice the bugfixing — the exact fork cost the server plan rejected. Rejected.
- **Extract first, decouple later.** The back-edges (`controls`→`pages`, fat `ServerAccess`, hardcoded routes) make today's code un-liftable; you'd copy the whole app and own a fork. Decouple-in-place first; every Phase 1–3 step ships. Rejected — same reasoning as the server plan, and it held up well there.
- **Nx monorepo hosting library + app.** Would give boundary lint + caching out of the box, but requires migrating the app's build and either moving the app to GitHub or splitting the monorepo across hosts — heavy machinery for one library and one app. Rejected for now; the entry-point + ESLint approach reaches the same enforcement.
- **Separate npm packages per layer (Option B).** Version-bump cascade while the API is hot. Rejected; entry points preserve the option.
- **Git submodule / git-dep consumption.** Submodule ergonomics, and npm git-deps don't fit an Angular workspace (repo root ≠ package). Rejected.
- **GitHub Packages as the registry.** Auth token required even for public installs. Rejected.
- **Angular Elements (custom elements) as the sharing mechanism.** Framework-agnostic but pays bundle + DI/theming costs on every use; no non-Angular consumer exists. Deferred (listed in Ideas).
- **Storybook-first component workshop.** Nice-to-have, not load-bearing; the showcase app covers the need and doubles as the site template. Deferred.

# Open Questions / Discussion

**Status: all 12 questions are resolved (7/17/2026)** and folded into the plan above. The Q&A below is kept as the decision record. Summary: Q1 repo `honuware/web_components` + package `@honuware/ui` + `hw-` prefix + Option C entry points; Q2 public npm from day one (0.x, CI publish with provenance, org claim in Phase 4.1); Q3 upgrade Angular to the current stable major now (new Phase 1.0); Q4 styling = S2 (de-Tailwind the library; a Material theme is the only consumer requirement); Q5 extraction before Website Makeover; Q6 auth pages ship in the library; Q7 ControlValueAccessor deferred (additive in the library later); Q8 legacy admin editor deleted, not ported; Q9 `@honuware/ui/square` = SDK wrapper only, card/checkout UI stays app-side; Q10 fresh library mocks, app's ServerAccessMock untouched; Q11 CommunityFinder (own vault/repo) is the first consumer — showcase demoted to dev harness; Q12 `HONUWARE_BRANDING` as a static token now, tenancy Phase 7's SiteConfigService feeds it later.

1. **Repo + package naming.** Recommendation: repo `honuware/web_components` (sibling symmetry with `server_components`), npm package `@honuware/ui` with the entry points as designed, selector prefix `hw-`. Alternatives: repo `ui_components`/`honuware-ui`; package-per-layer (Option B) if you disagree with C. Also note: the `@honuware` npm org name needs claiming (N1) — worth doing early even if we start with N3.
	- Mason- I like honuware/web_components. It mirrors the server_components nicely.
2. **Distribution mechanism + timing.** Recommendation: GitHub Release tarballs (N3) until the API settles, then public npm (N1) with CI publish + provenance; never GitHub Packages (N2). Alternatively go straight to N1 at `0.1.0` and publish freely — 0.x promises nothing. Which do you prefer, and do you want me to treat the npm org claim as a Phase 4 checklist item?
	- Mason- if the goal is eventually public npm, why not just start with that? I don't think we are going to need to rev the API that much.
3. **Angular version strategy.** v19.2 today; v19's LTS ends ~Nov 2026, and the friends' fresh `ng new` sites will scaffold on the then-current major. Recommendation: upgrade the knottyyoga workspace to the current stable major (19 → 20 → … stepwise) as a standalone pre-Phase-2 work item so the library is born current (I'll verify the exact current major + Material availability when we start). Alternative: extract on 19 and upgrade the library immediately after — workable (older-major libraries load in newer apps) but starts the public repo on a dying major. OK to add the upgrade item?
	- Mason- I'm fine with moving to a newer Angular version now.
4. **Styling: S1 (Tailwind preset requirement) vs S2 (de-Tailwind the library, Material-theme-only requirement)?** Recommendation: **S2** — the extraction set barely uses Tailwind, and it keeps every future consumer's CSS stack free. Costs a small Phase 1.3 sweep.
	- Mason- I'll go with your recommendation.
5. **Sequencing vs [[Website Makeover]].** Recommendation: componentize/extract **first**, with the narrow styling rules above (no hardcoded colors, `.surface-card`-compatible class, CSS-var awareness); the makeover then restyles the library like any other folder, via ordinary version bumps (and the co-dev override makes that painless). Alternative: run makeover Phases 2–3 first so extracted components are born token-native — cleaner but blocks this project on a large styling effort. Your call on ordering.
	- Mason- Let's do the extraction first.
6. **Auth scope: do the login/register/verify *pages* ship in the library** (recommendation: yes — a new site gets working auth screens with zero code; branding via the `HONUWARE_BRANDING` token; layout/styling intentionally plain so sites can restyle or replace them), or services/guards only, with each site owning its page shells? Note there's no forgot-password flow anywhere today — building one (server + UI) would be a separate work item; the library just shouldn't preclude it.
	- Mason- I'll go with your recommendation and I'd like each site to get these basically for free.
7. **`ControlValueAccessor` adoption for the field controls.** Today they use a bespoke `value`/`valueChanged` string contract. Recommendation: **defer** — extract as-is (renamed selectors only), then add CVA support *additively* in the library later (new sites get reactive-forms ergonomics; knottyyoga's composite plumbing keeps working). Adopting CVA during extraction would churn every consumer at the riskiest moment. Agree?
	- Mason- I'll go with your recommendation.
8. **Legacy admin editor generation** (`EditDbTableComponent` + `TableEntryFormDialogComponent` + the `:tableName` route): delete in Phase 1.6 rather than port (recommendation), assuming nothing you use daily still lives on the legacy route — confirm?
	- Mason- I'll go with your recommendation.
9. **Payment scope.** Recommendation (mirrors the server: `honuware_square` = raw client only, payment business logic app-side): `@honuware/ui/square` ships just `SquarePaymentService` behind `SQUARE_CONFIG`; `card-picker`/`payment-method`/checkout stay in the app because their server endpoints (cards, purchases) are app endpoints today. If/when saved-cards + purchases ever move into `honuware_platform` server-side, the matching UI follows. Confirm?
	- Mason- Your recommendation sounds good.
10. **Mock strategy for `/testing`.** Recommendation: write **fresh, minimal, spec'd framework mocks** in the library (schema-driven CRUD store + auth/photo sims) and leave knottyyoga's 253KB `ServerAccessMock` untouched (it must cover ~250 methods regardless). Accepted trade-off: two implementations of the framework subset exist, with drift possible; the optional later cleanup is delegating the app mock's framework subset to the library mocks. Or do you want that delegation done as part of Phase 2 (more churn in a giant file, single source of truth sooner)?
	- Mason- I'll go with your recommendation.
11. **The second consumer.** Is the Phase 5 showcase *literally the seed of your unrelated website* (my assumption — tell me its rough shape: content pages + auth + admin CRUD + photos + payment?), or do you want the showcase kept as a sterile demo and the real site started separately from the template? This decides how opinionated the showcase's shell (header/nav/theming) should be, and which entry points get built first if you want to prioritize.
	- Mason- I'm working on this site (C:\Users\mason\Documents\Obsidian\CommunityFinder\Claude\Setting up the project.md) to be the first consumer of this componentization effort. Once this document is complete, I will update that document to incorporate it and that effort is driving the need for this document and the work it will involve.
12. **Branding token timing.** Recommendation: introduce `HONUWARE_BRANDING` (display name, logo URL, website URL) in the library with static app-provided values, consumed only where library components genuinely need it (auth pages' headings, future shared shells); when the tenancy plan's Phase 7 `SiteConfigService`/`/api/site_info` lands, it becomes the token's provider. No library code touches `/api/site_info` before the endpoint exists. Confirm?
	- Mason- I'll go with your recommendation.