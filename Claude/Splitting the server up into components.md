---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/9/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

This web server has grown into a product with a lot of interesting code. I have some other websites I'm interested in creating. I was thinking I could factor a lot of this code into components that could be shared by a set of webservers. I'm trying to figure out how best to do this.

I currently am using CMake and conan. I have my code in Gitlab. I have my Obsidian vaults in Github. I wasn't sure if it would be a good idea to package up components and place them in a repository like Conan. I think I read Gitlab has a repository as well. Not sure about Github. Some friends are interested in working on this with me and would most likely want to work in Github so they can point potential employers to their work. I am open to leaving my server and the components in Gitlab or moving them to Github. If you can help me think through this process, that would be great.

I'm also trying to figure out what code to factor out into component(s). I feel like all the authentication stuff (business_logic/auth), images (business_logic/images), endpoint helper stuff (endpoints/ enpoint_auth_helper.* endpoint_test_helper.\*), scheduler, most of the stuff under sql_util minus some of the table_helpers stuff (which should probably be grouped), util (factoring out the server specific secret keys and values), and a good chunk of the stuff under test. I'm thinking some of these could be separate components that are layered and possibly depend on each other.

Let's explore a lot of options before committing to anything concrete.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Current State Assessment

## Build structure today

One static library, `knotty_yoga_core`, contains the entire server (util, sql_util, db_schema, table_helpers, business_logic, endpoints). Five executables link against it: `knottyyoga_the_server`, `knottyyoga_database_helper`, `knottyyoga_helper` (scheduler), `knottyyoga_test_helper`, and `knottyyoga_tests`. All tests aggregate into a single `knotty_yoga_tests` library (hand-listed per directory, not globbed). Conan 2 (`conanfile.py`) generates `ConanLibImports.cmake` which defines the `${XXX_LIB}` variables every target uses. Deployment builds one Docker image containing all binaries with a single version string.

The layering (db_schema → sql_util → util services → business_logic → endpoints) is culturally enforced but not build-enforced — everything is one library, so nothing stops an accidental upward include.

## Coupling analysis — what is reusable today

**Clean (extractable nearly as-is):**
- `util/` core: `types`, `json_value`, `json_util`, `logging`, `thread_pool`, `date_time_util`, `error_codes`, `error_response`, `file_util`, `bounding_rect`, `image_resize`, `destructive_guard`, `http/` — no includes outside `util/`.
- `sql_util/{database_access, json, schema, stored_procedures}` — generic DB framework; only issue is `kDatabaseName = "knottyyoga"` hard-coded in `database_common.h:29`.
- `business_logic/images` — the most self-contained module: depends only on photo tables, sql_util, and util. No auth/payment/yoga coupling, no brand literals.
- `scheduler/` engine (`scheduler`, `job_scheduler`, `api_client`, `scheduler_config`) — fully generic; only `scheduled_job.cpp:BuildStandardJobs` (≈21 hard-coded yoga endpoint paths) and the fixed `SchedulerConfig::intervals` struct are app-specific.

**Needs decoupling work (the blockers):**
1. `util/secrets/secrets_helper.cpp` → `business_logic/auth/secrets_at_rest.h` — a **back-edge creating a util ↔ auth cycle**.
2. `sql_util/table_helpers/email_verifications.cpp` → `business_logic/auth/auth_helper.h` — a **back-edge from the table layer into business logic**.
3. `business_logic/auth/quick_account_helper.cpp` → `business_logic/payment/gift_permission_helper.h` — auth's one payment dependency.
4. `business_logic/auth/server_config.h` and `cookie_manager.h` → `endpoints/web_app.h` — upward edges into the endpoints layer (they need the Crow app/middleware types).
5. `endpoints/endpoint_auth_helper.*` and `endpoint_test_helper.*` hard-wire the injected service set including `Square::SquareClient` (payment-specific).
6. `business_logic/auth/instructor_helper*` / `instructor_key_value_table*` — yoga-domain code living inside the auth directory.

**Brand/app literals inside otherwise-generic code:**
- `util/secrets/secret_keys.h` — `kMailSenderName = "Knotty Yoga and Spa"`, `kMailSenderAddress`, plus a mix of framework keys (mail/auth/argon2/rate-limit/image) and app keys (Square, subscriptions, scheduling, provider, facility).
- `util/secrets/secret_values.cpp` — Knotty Yoga default strings (subjects, website URL).
- `util/ical_generator.cpp` — hard-coded `@knottyyoga.com` UID domain and `PRODID:-//Knotty Yoga//Booking//EN`.
- `sql_util/database_common.h` — `kDatabaseName = "knottyyoga"`.
- Cosmetic: env var names (`KNOTTYYOGA_LOG_DEST`, `KNOTTYYOGA_ALLOW_DESTRUCTIVE`) and CMake target names.

**Framework vs app tables.** The schema splits cleanly. Framework (~33): `people, sessions, device_tokens, email_verifications, login_attempts, auth_events, roles, permissions, role_permissions, role_assignments, permission_implications, config_secrets, idempotency_keys, schema_migrations, allowed_tables, admin_alerts`, all `admin_*` metadata tables, and the photo tables (`photo_instances, source_photos, scaled_photos, photo_support_tables, table_item_photos`). App (~55+): classes, bookings, products, purchases, payments, subscriptions, coupons, vouchers, entitlements, event sessions, providers, facilities, etc. (`home_page_photos` is app-side despite living near the photo tables.)

## Constraints from other active plans

- **Multi-tenant SaaS plan (fully decided 7/9/2026, runs after the extraction):** components must stay tenancy-agnostic. Providers (`DatabaseHelper`, `SecretsHelper`, `SquareClient`, `MailHelper`) move behind a per-tenant registry — so components must accept these as injected parameters, never as global singletons. `business_logic/tenancy/` will be born inside `honuware_platform`. `EndpointAuthHelper` is the load-bearing seam for both tenancy and this split. The tenant plan's new §1.8 (in [[Converting the server to a multi tenant Saas architecture]]) maps its phases onto these components and hands five concrete requirements back to this plan — see **Hand-off requirements from the tenancy plan** below; the affected work items are tagged `⇦ tenancy`.
- **Test-speed work (implemented):** the 88% speedup depends on assembling the *complete* `DatabaseInfo` once (`GlobalDatabaseTestSupport::SetupAllTables()` + `SetIfNotExistsMode`). Splitting the schema must preserve a way to compose framework tables + app tables into one `DatabaseInfo` and run the one-time committed setup.
- **AWS deployment (implemented):** one multi-stage Dockerfile produces one image with all binaries and one version string, built by GitLab CI. Packaging changes must not break the single-image pipeline.
- **Scheduler (implemented):** communicates with the server purely over HTTP; only drags in `knotty_yoga_core` for `HttpClient` + logging. Extracting `util/http` frees it.

## Hand-off requirements from the tenancy plan (added 7/9/2026)

The multi-tenant plan is now fully decided and sequenced immediately after this project (componentize → extract → tenancy → deploy). Its §1.8 analyzed the interaction between the two plans; five of its requirements land **here** because doing them during componentization is free (or mandatory), while retrofitting them during the tenancy project would mean churning freshly-extracted honuware surfaces. Each is folded into the phase items below, tagged `⇦ tenancy`:

1. **Framework mail-builder branding is an extraction prerequisite** (Phase 1.3). `person_verify_mail.cpp` (and any other framework mail body) hardcodes "Knotty Yoga" — it cannot move into the *public* honuware repo with the brand baked in. The branding struct introduced here is the same one the tenant plan later sources per-tenant.
2. **Env-var renames go beyond `LOG_DEST`/`ALLOW_DESTRUCTIVE`** (Phase 1.1). Every env var read by code that moves into honuware renames to `HONUWARE_*`: `KNOTTYYOGA_ORIGIN_SECRET`, `KNOTTYYOGA_TRUST_PROXY`, `KNOTTYYOGA_SECRET_KEY`, `KNOTTYYOGA_DEV_CORS_ORIGIN`. The tenant plan references the new names throughout.
3. **`ScheduledJob` carries per-job request headers + login credentials** (Phase 1.7). The tenant plan expresses multi-tenancy purely as catalog × tenants built in app-side `main.cpp` — the engine never learns what a tenant is, which only works if a job is fully self-describing.
4. **The migration list splits framework/app alongside `DatabaseInfo`** (Phase 2.3), as two composed streams tracked under separate `schema_migrations` id namespaces, so honuware upgrades and app migrations can't collide once tenant databases are migrated in a loop.
5. **`GlobalDatabaseTestSupport` can create additional named test databases** (Phase 2.6). The tenant plan adds a physical-isolation test suite on a second real database (`test_honuware_tenant_b`), reusing the create-once pattern.

Confirmations flowing the other way (no work here): tenancy adopts the Phase 1.6 façade shape for its `TenantResources` app-factory (Square stays app-side, exactly as 1.6 designs it); the site header is `X-Honuware-Site` and the control DB is `honuware_control`, per this plan's naming; and single-tenant honuware consumers get a `FixedTenantResolver` needing no control DB and no CloudFront header — which is also how the Phase 5 example server and the friends' sites will run.

# Target Component Architecture

## Proposed layered components

Component namespace: **`honuware`** (decided, Q1). It becomes the repo name, CMake target prefix, and env-var prefix (`HONUWARE_*`). Each component depends only on those above it in this list (scheduler and square depend only on foundation):

```
honuware_foundation      util core: types, json_value/json_util, logging, thread_pool,
                     date_time_util, error_codes/error_response, file_util,
                     bounding_rect, image_resize, destructive_guard, http/
        ▲
honuware_data            sql_util core: database_access, schema, json,
                     stored_procedures (generic), database_common (parameterized)
        ▲
honuware_services        util/secrets (incl. secrets_at_rest moved down), util/mail
        ▲
honuware_platform        framework db_schema + table_helpers (~33 tables),
                     business_logic/auth core, business_logic/images,
                     business_logic/migration (runner), web core (web_app,
                     middleware guards, endpoint_auth_helper, generic CRUD
                     endpoints + admin metadata), future business_logic/tenancy
        ▲
honuware_scheduler       scheduler engine (job_scheduler, scheduler, api_client,
                     ScheduledJob struct) — depends only on foundation
        ▲
honuware_square          util/square Square API client (client constructed with
                     token; HTTP via foundation) — depends only on foundation
        ▲
honuware_testing         GlobalDatabaseTestSupport, TestDatabaseUtil,
                     TestTransactionProvider, matchers, EndpointTestHelper,
                     test doubles (TestMailHelper, SecretsHelperTest,
                     TestSquareClient ...)
─────────────────────────────────────────────────────────────────────────
knottyyoga app       app db_schema/table_helpers (~55 tables), payment/,
(stays in app repo)  scheduling/, skills/, app endpoints, ical branding config,
                     database_helper, test_helper, main.cpp,
                     scheduler job catalog, app secret keys/defaults
```

Notes:
- `util/square` becomes **`honuware_square`** (decided, Q9): the Square API client is generic (any site taking payments can use it) and already has no includes outside `util/` — client is constructed with a token, so no secrets coupling. Its test double (`square_client_test_util`) ships in `honuware_testing`. The *payment business logic* (PaymentHelper, purchases, entitlements) stays app-side — only the raw Square client is shared.
- `util/ical_generator` is generic once the PRODID/UID domain are parameterized — goes in `honuware_foundation` or `honuware_platform`.
- The generic table CRUD endpoints (`get_row`, `add_item`, `update_item`, `delete_item`, `get_fk_options`, `db_schema`, etc.) plus the `admin_*` metadata system are one of the most valuable reusable assets — they give any new site a free admin data editor.

## Granularity options

**Option A — one component package** ("the platform"). Everything reusable in a single package/repo with internal CMake targets.
- ✅ Trivial version management (one version), one CI, easiest for collaborators.
- ❌ Coarse dependency: a site wanting only the scheduler engine pulls Postgres/libpqxx/mailio/etc.

**Option B — the six layered packages above, separately versioned.**
- ✅ Precise dependencies; clean layering enforced by packaging.
- ❌ Heavy version-management ceremony (bump foundation → rev data → rev services → …) while the code is still evolving fast. This is the classic "too many packages too early" trap.

**Option C (recommended) — one repo, multiple CMake targets, one version.** A single shared repo (and later a single Conan package, or one package per target with lockstep versions) exposing `honuware_foundation`, `honuware_data`, `honuware_services`, `honuware_platform`, `honuware_scheduler`, `honuware_testing` as separate link targets. Layering is enforced by target link rules inside the repo, but you release everything together.
- ✅ One version to pin, one CI, layering still build-enforced, consumers link only the targets they need. Easy to split into Option B later (the target boundaries are already the package boundaries).
- ❌ A consumer's `conan install` builds the whole repo even if it links one target (acceptable at this scale).

**Decided (Q4): Option C** — one repo, one version, layered CMake targets (now seven, with `honuware_square` added). Fine-grained packaging is easy to add later and painful to retract.

# Repository Hosting Options

Context: server code is in GitLab; friends want portfolio-visible work on GitHub; Obsidian vaults already on GitHub.

**Option R1 — everything stays in GitLab.** Components as a new GitLab repo (or subgroup).
- ✅ One home, GitLab CI already planned, GitLab package registry available.
- ❌ Fails the friends/portfolio goal — GitHub is where employers look.

**Option R2 (recommended) — components on GitHub (public), knottyyoga app stays on GitLab (private).**
- ✅ Friends contribute publicly on GitHub (commits, PRs, contribution graph). Public repos get free GitHub Actions CI. The app with your business logic, pricing, and studio specifics stays private.
- ✅ Forces the right discipline: anything moving to the component repo must be genuinely brand-free.
- ❌ Two hosts, two CIs. Needs a license and a bit of README polish since it's public.

**Option R3 — move everything to GitHub.** App becomes a private GitHub repo alongside the public components.
- ✅ One host, one CI system (Actions), simplest mental model, easiest collaboration if friends will also touch the app.
- ❌ You lose the GitLab CI work already planned/done; GitLab's Conan registry advantage goes away (see packaging).

**Option R4 — GitLab primary + push-mirror to GitHub.** Keep working in GitLab; mirror the component repo to GitHub for visibility.
- ✅ No workflow change for you.
- ❌ Mirrors are read-only second-class citizens: friends' PRs would still happen on GitLab, and a mirrored repo with no issues/PR activity looks dead to employers. Weak portfolio value.

**Decided (Q2): R2.** The `honuware` components and the friends' new sites live on GitHub; the knottyyoga app stays private on GitLab (friends won't work on the app). Note this means the new sites' CI is GitHub Actions while knottyyoga's stays GitLab CI — the component repo's CI setup (Phase 4.3) doubles as the template for the new sites.

# Packaging & Distribution Options

How the app (and future sites) consume the component repo:

**Option P1 — CMake FetchContent (source-based, no registry).** The app's CMake fetches the component repo at a pinned git tag/SHA and builds it in-tree; the component repo's Conan deps merge into the consumer's `conanfile.py` (or the component provides a `conandata`/requirements list the consumer includes).
- ✅ No registry infrastructure at all. Works identically on GitHub/GitLab, Windows/Linux. Pin = git SHA, fully reproducible. Great while the components change weekly.
- ❌ Every consumer rebuilds components from source (fine — you rebuild `knotty_yoga_core` today anyway). Conan dependency lists must be kept in sync between component repo and consumers by convention.

**Option P2 — Conan packages published to GitLab's package registry.** GitLab's Conan 2 registry moved from Experimental to **Beta in GitLab 18.10 (March 2026)** with full v2 API compatibility and recipe-revision support ([docs](https://docs.gitlab.com/user/packages/conan_2_repository/), [release notes](https://about.gitlab.com/releases/2026/03/19/gitlab-18-10-released/)).
- ✅ Real versioned packages, `conan install` just works, binary caching possible, no third-party service.
- ❌ Beta, not GA. Auth tokens needed for friends. If the source lives on GitHub (R2) the packages living on GitLab is a slightly odd split.

**Option P3 — Conan packages on a third-party registry** (JFrog Artifactory cloud free tier, Cloudsmith free OSS tier). GitHub Packages does **not** support Conan natively, so this is the "binary registry + GitHub" answer.
- ✅ Proper Conan hosting with GitHub source.
- ❌ Another account/service dependency; free tiers change terms.

**Option P4 — Conan local-recipes-index repo.** Conan 2.2+ can treat a git repo laid out like conan-center-index as a read-only source-build remote. The component repo (or a sibling `recipes` repo) doubles as the Conan remote; consumers `conan remote add` the git URL and build from source.
- ✅ Registry-free but still "real Conan": versions, `conan install`, lockfiles. Plays perfectly with GitHub hosting.
- ❌ Source-build only (no binary hosting); newer workflow, less documentation/mileage.

**Option P5 — git submodule.** ❌ Not recommended — submodule ergonomics (detached heads, forgotten `--recursive`, sync pain for collaborators) are worse than P1 for no benefit.

**Decided (Q3): P1 (FetchContent, SHA-pinned) for Phase 4, graduating to P4 or P2 once the component API stabilizes.** While you and friends are actively co-developing both sides, source-based consumption with git-SHA pinning has the least ceremony and the fewest moving parts; during local co-development you can point FetchContent at a local checkout (`FETCHCONTENT_SOURCE_DIR_HONUWARE`) for instant edit-rebuild across repos. honuware's third-party Conan requirements live in one shared list the app's `conanfile.py` imports so the two can't drift. Adopt versioned Conan packages when a component is stable enough that consumers *shouldn't* track head — that's also the natural moment to decide P2 vs P3/P4 based on how the registries have matured.

# Cross-Cutting Decisions

**Naming (decided, Q1): `honuware`.** Repo name `honuware`, CMake target prefix `honuware_`, env-var prefix `HONUWARE_` (e.g. `HONUWARE_LOG_DEST`). Existing C++ namespaces (`Auth`, `SqlUtil`, `Json`, `Secrets`, `Mail`, `Http`) are already brand-neutral and stay; only file-level constants, env vars, and target names change.

**License (decided, Q5): Apache-2.0.** Explicit patent grant, patent-retaliation clause, §5 contribution licensing (built-in lightweight CLA for the friends' PRs), and trademark carve-out for the honuware name. Repo gets `LICENSE` + `NOTICE` files at creation (Phase 4.1).

**Versioning.** With Option C: single version for the whole component repo, tagged releases (`v0.x` semantics — breaking changes allowed — until a second consumer exists). The app pins a SHA/tag and upgrades deliberately.

**History (decided, Q11).** Fresh component repo with an "extracted from knottyyoga at SHA …" note in the initial commit; GitLab keeps the archaeology.

**Deployment invariant.** Through every phase, one Docker build must keep producing one image with all binaries and one version string. FetchContent keeps this true automatically (the builder clones the pinned component SHA during `cmake` configure; the Dockerfile needs git + network in the build stage, or a vendored source tarball).

**Windows + Linux (decided, Q10).** Components must build in both (you develop on Windows/VS, deploy and test on Linux). Component CI is Linux-only (container with Postgres service); Windows/MSVC is verified manually. The Crow `HTTPMethod` PascalCase rule and other Windows gotchas move into the component repo's CLAUDE.md so manual Windows verification stays cheap.

# Phased Implementation Plan

Phases 1–3 happen **entirely inside the knottyyoga repo** — no new repos, no packaging, no behavior changes. This is deliberate: extraction is trivial once the boundaries are real, and every phase leaves the app shippable. Phase 4 is the actual extraction; Phase 5 proves reusability. Within each phase, work proceeds lower layers first.

**Sequencing vs multi-tenancy (decided, Q6):** componentize (1–3) → extract (4–5) so the friends' site can start → multi-tenant conversion → first deploy. Extraction comes before tenancy because friends are waiting on the components; tenancy stays before deploy, which is the cheapest possible window for it (no production data to migrate). See the Q6 discussion in Open Questions for the trade-off analysis and expected cross-repo friction during the tenancy work.

## Phase 1 — Break the coupling (in-place refactors)

### 1.1 util foundation cleanup
- [x] Parameterize `util/ical_generator` — PRODID and UID domain become constructor/function parameters; knottyyoga values supplied at call sites (likely via a secret/config). Update all call sites + `ical_generator_test.cpp`.
    - **Done.** `GenerateICalendar` and both `GenerateICalendar`/`Build*Uid` overloads now take an `ICalGenerator::ICalConfig{prodId, uidDomain}` (or a `uidDomain` string_view) — a *required* parameter, so the compiler forces every call site to supply it and output is guaranteed unchanged. The knottyyoga values live in one place, `business_logic/app_ical_config.h` (`App::kICalProdId`, `App::kICalUidDomain`, `App::AppICalConfig()`) — the single seam where per-tenant branding will later plug in. 9 call sites updated (`attendance_template_helper`, `payment_helper`, `personal_ical_feed_helper`, `signup_reminder_helper`, `weekly_digest_helper`, `cancel_booking`, `cart_checkout`, `book_service`, `staff_upgrade_session`). `ical_generator_test.cpp` reworked to use a test config (`-//Test Studio//Booking//EN` / `test.example`), plus new `ProdIdAndUidDomainComeFromConfig` and `UidDomainIsCallerSupplied` tests.
- [x] Rename env vars in `util/logging` and `util/destructive_guard` to the honuware prefix (`HONUWARE_LOG_DEST`, `HONUWARE_ALLOW_DESTRUCTIVE`), honoring the old `KNOTTYYOGA_*` names during transition (read new name first, fall back to old). `⇦ tenancy`: apply the same convention to the other env vars read by moving code — `KNOTTYYOGA_ORIGIN_SECRET`, `KNOTTYYOGA_TRUST_PROXY`, `KNOTTYYOGA_SECRET_KEY`, `KNOTTYYOGA_DEV_CORS_ORIGIN` → `HONUWARE_*` (rename each when its code is touched here, or all at once — the tenant plan uses the new names). Update the env-var table in the server CLAUDE.md + tests.
    - **Done (all six, all at once).** Added `util/env.h`/`env.cpp` — `Util::GetEnvWithFallback(primary, fallback)`, a pure `getenv` wrapper (no logging, safe for the logging bootstrap) that reads the canonical name first, falls back to the legacy name, returns `nullptr` if neither is set, and lets a set-but-empty primary win. Tests in `env_test.cpp` (4 cases). Renamed all six brandable vars to `HONUWARE_*` with the `KNOTTYYOGA_*` fallback: `LOG_DEST` (`logging.cpp`), `ALLOW_DESTRUCTIVE` (`destructive_guard`), `ORIGIN_SECRET` (`cloudfront_origin_guard`), `TRUST_PROXY` (`proxy_trust`), `SECRET_KEY` (`secrets_at_rest`), `DEV_CORS_ORIGIN` (`server_config`). Error messages / log lines / help text now name the `HONUWARE_*` variant. Tests updated across `destructive_guard_test`, `proxy_trust_test`, `cloudfront_origin_guard_test`, `server_config_test`, `secrets_at_rest_test` — every env-scope RAII scrubs BOTH names (so a legacy var in the dev launch env can't leak through the fallback), error-message assertions flipped to `HONUWARE_*`, and each got new canonical-name + legacy-fallback + precedence coverage. CLAUDE.md env-var table rewritten with a Legacy-fallback column and a rename-convention note.

### 1.2 sql_util core cleanup
- [ ] Parameterize `kDatabaseName` in `sql_util/database_common.h` — database name becomes a runtime parameter owned by the app (this also aligns with the multi-tenant plan where DB name is per-tenant). Update call sites + tests.
- [ ] Break `sql_util/table_helpers/email_verifications.cpp` → `business_logic/auth/auth_helper.h` back-edge: move the token hashing up into the business_logic/auth caller (or inject a hash function). The table helper becomes pure CRUD. Update tests in both layers.

### 1.3 secrets layer fix (kills the util ↔ auth cycle)
- [ ] Move `secrets_at_rest.{h,cpp}` (+ its test) from `business_logic/auth/` into `util/secrets/` — verify its own includes are util-level (libsodium, types); adjust include paths and both CMakeLists.
- [ ] Split `util/secrets/secret_keys.h` into framework keys (mail, auth/argon2, rate-limit, image sizing) kept in `util/secrets/` and app keys (Square, subscription, scheduling, class check-in, provider, facility) moved to a new app-side header (e.g. `business_logic/app_secret_keys.h` — final location per the Phase 2 target split). Update all includers.
- [ ] Move brand defaults out of `util/secrets/secret_values.cpp` ("Knotty Yoga and Spa", gmail address, subjects, `www.knottyyoga.com`) into an app-side default-registration (secrets framework exposes `RegisterDefaults(...)`; `create_database`/app startup supplies the values). Tests for the registration mechanism + updated existing tests.
- [ ] `⇦ tenancy` (extraction prerequisite): parameterize the **framework** mail builders' branding. `person_verify_mail.cpp` (and any other auth/framework mail body) hardcodes "Knotty Yoga" and can't ship to the public repo that way. Introduce a branding struct (`TenantBranding { studioName; senderName; senderAddress; websiteUrl; }` — the same struct the tenant plan later sources per-tenant), populated from secrets/registered defaults, and replace the literals with `FormatString` placeholders (keep `NormalizeCrLf`). App-side mail builders (payment/scheduling) stay as-is until the tenancy project's Phase 4.5. Tests: each framework builder asserts the branding substitution and the absence of hardcoded literals.

### 1.4 mail layer position
- [ ] Document the intended layer order (foundation → data → services) in the server CLAUDE.md; `util/mail`'s include of `sql_util/database_access/transaction.h` is *conformant* under that order (services sit above data). No code change — just make the layering explicit so Phase 2 target boundaries are uncontroversial.

### 1.5 auth cleanup
- [ ] Move `instructor_helper.*` and `instructor_key_value_table.*` (+ tests) out of `business_logic/auth/` into `business_logic/scheduling/` (they're yoga-domain instructor profile code and scheduling already owns instructor concepts). Update includes + CMake.
- [ ] Break `quick_account_helper.cpp` → `payment/gift_permission_helper.h`: add an optional post-creation hook (`std::function<void(Transaction&, int64_t personId)>` or small interface) that the app wires to gift-permission granting at the endpoint layer. **Decided (Q8): quick accounts are a framework feature** — the hook approach is the path; knottyyoga wires gift permissions into the hook. Tests for both the hook path and the no-hook path.
- [ ] Break `server_config.h` / `cookie_manager.h` → `endpoints/web_app.h`: extract the Crow app typedef + middleware context types into a new lower-level header (e.g. `endpoints/web_app_types.h`) included by both; `web_app.h` re-exports it. (In Phase 2 this header lands in the web-core part of the platform target.) Update includes + tests.

### 1.6 endpoint façade generalization
- [ ] Restructure `EndpointAuthHelper` so the injected-service set isn't hard-wired to Square: framework façade exposes session/cookies/secrets/mail/db; the app layer provides an extended accessor for `SquareClient` (derived helper, or a typed service-registry on `WebApp`). Keep call-site ergonomics (`helper.GetSquareClient()`) via the app-derived type. Update `endpoint_test_helper` symmetrically. Tests.

### 1.7 scheduler catalog extraction
- [ ] Replace `BuildStandardJobs()`'s hard-coded yoga endpoint list and the fixed `SchedulerConfig::intervals` struct with a data-driven job list: engine consumes `std::vector<ScheduledJob>` + per-job interval config; the knottyyoga catalog moves next to `scheduler/main.cpp` (app-side). Flags stay generated from the job list. `⇦ tenancy`: give `ScheduledJob` per-job request headers and login credentials so each job is fully self-describing — the tenant plan later expresses multi-tenancy purely as catalog × tenants built in app-side `main.cpp`, with the engine never learning what a tenant is. Tests for engine-with-arbitrary-jobs (including arbitrary per-job headers) + existing catalog tests.

## Phase 2 — Enforce boundaries with CMake targets (still one repo)

### 2.1 Foundation and data targets
- [ ] Create `honuware_foundation` static lib target (util core files per the architecture section) and `honuware_data` (sql_util core); move `target_sources` entries from `knotty_yoga_core`; wire Conan lib links (Boost, date, PNG/TIFF/ZLIB/jpeg on foundation; PQXX/CURL on data).
- [ ] `knotty_yoga_core` links the new targets PUBLIC so nothing else changes yet.

### 2.2 Services and Square targets
- [ ] Create `honuware_services` (util/secrets, util/mail) linking foundation + data (+ mailio/OpenSSL).
- [ ] Create `honuware_square` (util/square) linking foundation only (+ CURL via http). Its test double joins `honuware_testing` in 2.6. Verify no secret-key or app includes remain in `square_client.*` (per the coupling audit there are none — the client takes its token at construction).

### 2.3 Platform target
- [ ] Create `honuware_platform`: framework db_schema + framework table_helpers (~33 tables per the assessment list), `business_logic/auth` core, `business_logic/images`, `business_logic/migration`, web core (web_app, middleware guards, endpoint_auth_helper, generic CRUD + admin metadata endpoints, health). This is the big one — expect several sittings; move one sub-group at a time, keeping the build green.
- [ ] Split `db_schema/make_database_info` into framework-table assembly (platform) + app-table assembly (app), composed by the app (preserves FK ordering and the test-speed create-once contract). `⇦ tenancy`: split the **migration list** the same way at the same time — framework migrations (shipped with honuware) and app migrations as two composed streams, tracked under separate `schema_migrations` id namespaces so honuware upgrades and app changes can't collide once the tenant plan migrates every tenant DB in a loop. Tests for composed-list application + namespace independence.

### 2.4 App target
- [ ] What remains of `knotty_yoga_core` becomes the app library (app schema/table_helpers, payment/, scheduling/, skills/, app endpoints, app secret keys/defaults, ical branding wiring); rename or keep name (cosmetic). Payment business logic links `honuware_square` for the client.

### 2.5 Scheduler target hygiene
- [ ] `knotty_yoga_scheduler` links `honuware_foundation` only (post-1.7 it no longer needs core); `main.cpp` (app-side job catalog + service-account env var) keeps its app deps.

### 2.6 Test target split
- [ ] Create `honuware_testing` from `test/src/util/` (GlobalDatabaseTestSupport, TestDatabaseUtil, TestTransactionProvider, matchers, json/table test utils) + `endpoint_test_helper` + the test doubles (`*_test_util` for secrets/mail/cookies/square). Fix the duplicate source listing between `test/CMakeLists.txt` and `test/src/util/CMakeLists.txt` while at it.
- [ ] `GlobalDatabaseTestSupport` takes the composed `DatabaseInfo` (framework + app tables) as input rather than calling `MakeDatabaseInfo()` directly, preserving the create-all-tables-once optimization for any consumer. `⇦ tenancy`: support creating **additional named test databases** once per run with the same create-once pattern — the tenant plan adds a physical-isolation suite on a second real database (`test_honuware_tenant_b`). Tests.
- [ ] Keep the single `knottyyoga_tests` executable; it now links app + platform + testing targets.

### 2.7 Layering enforcement

Build-system enforcement comes in tiers of increasing strength; each phase turns on the next tier:
- **Link-level (this phase, automatic):** layered static libs catch upward *calls* as undefined symbols at link time. Leaky — header-only usage (inline/templates/structs) generates no symbol, and CMake permits cycles between static libraries, so this alone is not enough.
- **Include-path-level (activates with 3.1):** once each component owns its include root and include paths propagate only via `target_link_libraries`, an upward `#include` fails at compile time ("header not found") — the violation becomes inexpressible. Blocked until the physical file move because today `src/` is one shared PUBLIC include root.
- **Repo-level (Phase 4):** app code doesn't exist in the honuware repo, so component→app coupling can't compile; standalone CI proves it continuously, and the Phase 5 example server does the same for the API surface.

Work items:
- [ ] Add a `honuware_add_component(NAME <c> DEPENDS <lower components...>)` CMake wrapper that declares each component against an explicit allowed-layer DAG and fatals on anything else — closes the static-lib-cycle loophole and makes the layer diagram executable.
- [ ] Set `CMAKE_LINK_LIBRARIES_ONLY_TARGETS ON` (CMake 3.23+) so raw/typo'd library names error instead of silently resolving.
- [ ] Add a small CI-runnable path-prefix include-audit script. This is the gap-coverage tier: it's the only check that catches violations *within* a single target (possible while all components still share the `src/` include root) and it works from day one. Keep it even after 3.1 as a fast pre-build check.
- [ ] Verify no include violations remain across the new target boundaries (each target's headers only reference its own layer or below).

## Phase 3 — Dry-run the extraction inside the repo

### 3.1 Physical layout
- [ ] Move component-target sources into a top-level `components/` directory tree inside the knottyyoga repo mirroring the future repo layout (`components/foundation/`, `components/data/`, …), updating include prefixes once (mechanical, like the 2023 refactor — ~300 includes). App code keeps `src/`.
- [ ] Give each component its own include root (`target_include_directories` per component, propagated only via `target_link_libraries`) instead of the single shared `src/` root. **This is the step that turns on compile-time layering enforcement** — an upward `#include` becomes "header not found" (see 2.7). Verify a deliberate violation fails to configure/compile on both Windows and Linux.
- [ ] Component tests live with their code (per testing conventions) and still feed the single test executable.

### 3.2 Bootstrap seam
- [ ] Ensure everything app-specific enters via explicit registration at startup: database name, secret defaults, admin-metadata population, scheduler job catalog, DatabaseInfo composition, endpoint registration. `main.cpp` + `create_database.cpp` become the single composition roots. (Most of this falls out of Phases 1–2; this item is the audit.)

## Phase 4 — Extract to the shared repo

### 4.1 Repo creation
- [ ] Create the `honuware` repo on GitHub (public, per Q2), with Apache-2.0 `LICENSE` + `NOTICE` (per Q5), README, CLAUDE.md (layering rules, Windows gotchas, testing conventions), and the `components/` tree moved over with fresh history + "extracted from knottyyoga at SHA …" note (per Q11).
- [ ] Component repo builds standalone: own `conanfile.py` (subset of deps), own CMake, own test executable for component tests (data/platform/testing tests need a Postgres container — reuse the `test_container` pattern).

### 4.2 Consumption from the app
- [ ] knottyyoga consumes via FetchContent pinned to a tag/SHA (P1), with `FETCHCONTENT_SOURCE_DIR` override documented for local co-development. Component tests that moved out are removed from the app's test lib; app test suite still green.
- [ ] Update `package/Dockerfile` / `build_linux_release.sh` so the single-image build fetches the pinned component source (git in builder stage or vendored tarball). One image, all binaries, one version string — unchanged contract.

### 4.3 CI
- [ ] Component repo CI (GitHub Actions, per Q2): Linux build + full component test run with Postgres service. No Windows CI job — Windows/MSVC verified manually (per Q10).
- [ ] App CI keeps working against the pinned component version.

### 4.4 Packaging graduation (deferred until API stabilizes)
- [ ] Add a `conanfile.py` package recipe to the component repo and publish v0.x to a registry (P2 GitLab / P3 Cloudsmith-Artifactory / P4 local-recipes-index — pick when we get here, based on how the registries have matured); switch the app from FetchContent to `conan install` of the pinned version.

## Phase 5 — Prove reuse with a second consumer

### 5.1 Example server
- [ ] Build a minimal `examples/hello_server` inside the component repo: boots WebApp + auth (register/login/sessions/RBAC) + admin CRUD over framework tables + images, with its own tiny `main.cpp` composition root and one example domain table. This is executable documentation, the onboarding path for friends, and the regression test that no yoga leaked in.
- [ ] Its existence gates calling the extraction "done": if the example needs to include anything from knottyyoga, a boundary is broken.

### 5.2 Docs & onboarding
- [ ] README quick-start (clone, conan, cmake, run example, run tests) verified on a clean machine; short "how to start a new site on the platform" doc.

# Alternatives Considered (and why not)

- **Copy-paste fork per new site.** Fast to start, but you named the goal: shared code across sites with friends. Divergence cost arrives within months. Rejected.
- **Extract first, decouple later** (move code to a new repo now, fix coupling there). The back-edges make the code un-liftable today — you'd copy the whole core and be back to a fork. Decouple-in-place first is strictly safer; every Phase 1–3 step ships. Rejected.
- **One mega "platform" repo including the frontend.** Angular admin-UI sharing is genuinely valuable (the generic CRUD UI pairs with the generic endpoints) but is a separate npm-flavored effort with different packaging; scoping it in now doubles the project. **Decided (Q7): deferred to a separate future job**, to be designed together with the multi-tenant frontend/branding question.
- **Microservice split** (auth service, image service as separate processes). Massive operational cost for a single-EC2 deployment; the multi-tenant plan already chose one process serving all tenants. Library components, not services. Rejected.

# Open Questions

**Status: all 12 questions are resolved (7/8/2026)** and folded into the plan above. The Q&A below is kept as the decision record. Summary: Q1 name = `honuware`; Q2 hosting = components + new sites public on GitHub, app private on GitLab; Q3 packaging = FetchContent SHA-pinned now, Conan registry later; Q4 granularity = one repo/one version/layered targets; Q5 license = Apache-2.0; Q6 sequencing = componentize → extract → tenancy → deploy; Q7 frontend sharing = separate future job; Q8 quick accounts = framework with hook; Q9 Square client = `honuware_square` component now; Q10 = Linux-only CI, manual Windows; Q11 = fresh git history; Q12 admin_alerts = framework.

1. **Component name/namespace?** Needs to be brand-neutral: repo name, CMake target prefix (`honuware_platform`), env-var prefix. Ideas to react to: `crowbase`, `croft`, `stonework`, `keystone`, `loom` — or pick your own. (Also: is this maybe a product someday, which would argue for a more distinctive name and Apache-2.0?)
	- Mason- let's go with honuware
2. **Repo hosting:** I recommend R2 (components public on GitHub, app stays private on GitLab). Confirm? And: will the friends work only on the components, or also on the new sites / the yoga app itself? (If the latter, R3 — everything on GitHub — gets stronger.)
	- Mason- The components and new sites will be on Github. I kind of like the app staying on Gitlab and keeping that private. I don't see any of them working on the app in general.
3. **Packaging:** I recommend P1 (FetchContent, SHA-pinned) first, graduating to a Conan registry later (GitLab's Conan 2 registry is Beta as of 18.10; Cloudsmith/Artifactory or local-recipes-index are the GitHub-side options). OK, or do you want real Conan packages from day one?
	- Mason- So CMake or Conan would fetch things from git by SHA?
	- Claude- They're two different mechanisms, and only the CMake one fetches from git directly:
		- **CMake FetchContent (P1) — yes, straight from git by SHA.** knottyyoga's top-level CMakeLists gains: `FetchContent_Declare(honuware GIT_REPOSITORY https://github.com/<you>/honuware.git GIT_TAG <full commit SHA or tag>)` + `FetchContent_MakeAvailable(honuware)`. At configure time CMake clones exactly that commit into the build tree and `add_subdirectory()`s it, so `honuware_platform` etc. become ordinary targets you link. Upgrading = edit the SHA, reconfigure. Identical behavior in Visual Studio on Windows and in the Docker build.
		- Conan is still in the picture, but only for **third-party** deps (boost, crow, libpqxx, …). honuware itself is not a Conan package in this mode. One wrinkle: knottyyoga's `conanfile.py` must also list honuware's third-party requirements; we'll keep honuware's requirement list in one file the app's conanfile imports, so the two can't drift.
		- Local co-development: passing `-DFETCHCONTENT_SOURCE_DIR_HONUWARE=C:\...\honuware` makes the build use your working checkout instead of cloning — you edit both repos and build once, no commit/push cycle needed.
		- **Conan registry (P2/P3) — fetches by version, not SHA.** `conan install` pulls `honuware/0.4.0` from a registry; the recipe maps version → source. That's the graduation step once the API stabilizes.
		- **Local-recipes-index (P4) — the hybrid.** Conan reads recipes from a git repo and builds from the tag each recipe references; "Conan from git," but versioned by recipe rather than raw SHA.
		- So the plan is: CMake-fetches-git-by-SHA now (P1), Conan-fetches-by-version later. OK to mark decided?
		- Mason- Sure, this sounds great.
4. **Granularity:** I recommend Option C (one repo/version, six CMake targets). OK, or do you prefer a single monolithic target (simpler) / fully separate packages (Option B)?
	- Mason- I'll go with your recommendation.
5. **License** for the public repo: Apache-2.0 (my suggestion) or MIT?
	- Mason- Can you list the advantages of each?
	- Claude- Both are permissive: anyone (including you) can use the code in closed-source commercial products, and neither imposes obligations beyond keeping the attribution notice. The differences:
		- **MIT advantages:** ~170 words and instantly understood by everyone; the most common license on GitHub, so zero friction for contributors and for employers skimming the repo; no patent language to reason about; nobody ever has to think about compliance beyond "keep the notice".
		- **Apache-2.0 advantages:** an explicit **patent grant** — every contributor automatically licenses any patents covering their contribution, so users of honuware can't later be patent-sued by a contributor; a **patent-retaliation clause** (sue the project over patents → lose your license) that deters trolling; **§5 makes every contribution automatically licensed under the project license** — a built-in lightweight CLA, which genuinely matters once several friends are submitting PRs; an explicit **trademark carve-out** (the "honuware" name isn't licensed with the code — relevant if it becomes a product); and it's the license enterprise legal reviews are most comfortable depending on. Costs: much longer text, plus minor NOTICE-file/"state your changes" housekeeping.
		- My recommendation stays **Apache-2.0** for this repo specifically: multiple outside contributors from day one (§5 earns its keep) and you've flagged possible productization (patent + trademark clauses earn theirs). MIT is the right choice if maximum simplicity outweighs those. Pick one and I'll fold it in.
	- Mason- I'll go with your recommendation.
6. **Timing vs multi-tenancy:** Phases 1–2 here overlap heavily with the tenant plan's "de-singleton" prep and make it easier. Do components come first, tenancy first, or interleaved (my suggestion: Phases 1–2 now, then tenancy, then extraction — tenancy lands *inside* the platform component)?
	- Mason- I haven't deployed the site yet. I was waiting to finish a few features and do multi tenant but I have friends wanting to work on a site now that uses my components. If I do the switch to multi tenant after doing this componentization but before deploying, will it be that hard?
	- Claude- No — and pre-deploy is actually the *cheapest possible* window for tenancy. What makes late tenancy conversions painful in general is production data: migrating a live single-tenant database, backfilling, cutting over users. You'll have none of that, so tenancy stays pure code + provisioning work no matter when it lands relative to this project.
		- Componentization does most of tenancy's prep for free: parameterized database name (1.2), secrets defaults via registration (1.3), the generalized `EndpointAuthHelper` (1.6), and the no-global-singletons audit (3.2). The tenant plan's `TenantContext`/`TenantResources` design drops into exactly those seams.
		- The one real (and modest) cost of doing tenancy *after* extraction: tenancy touches honuware surfaces — `EndpointAuthHelper` grows tenant resolution, provider factories move behind the per-tenant registry, the scheduler's `ApiClient` gains the site header, and `business_logic/tenancy/` is born inside the platform component. Expect a few weeks of editing both repos together and a handful of honuware version bumps. The `FETCHCONTENT_SOURCE_DIR` override makes both repos build as one during that stretch, so day-to-day it feels like working in a single repo.
		- Net: the tenancy project stays the same size it was going to be, plus a little cross-repo ceremony, and your friends aren't blocked waiting for it. I've updated the plan's sequencing to **componentize (Phases 1–3) → extract (4–5) → multi-tenant conversion → first deploy**. Confirm this ordering and I'll mark Q6 decided.
		- Mason- This sounds good.
7. **Frontend sharing:** in scope eventually? (Shared Angular admin CRUD UI, auth pages, header/footer as an npm package.) I've kept it out of this plan; confirm or ask me to add a frontend track.
	- Mason- I can see sharing some frontend components later but I think that we can tackle that as a separate job. I still need to figure that out for the multi tenant thing eventually.
8. **Quick accounts:** framework feature (with the gift-permission hook) or app-specific? I leaned framework-with-hook; fine either way.
	- Mason- I like putting this into the framework. The quick accounts kick ass and will be useful in all sites.
9. **`util/square`:** stay app-side for now (my suggestion) or extract as a `honuware_square` component immediately?
	- Mason- I'd like to extract this as a component now. I can see many sites wanting to take payment.
10. **Windows CI** for the component repo: worth a Windows MSVC job from day one (you develop on Windows), or Linux-only CI with Windows verified manually?
	- Mason- I think we can verify Windows manually.
11. **Git history:** fresh component repo with an "extracted from" note (my suggestion), or `git filter-repo` to preserve per-file history?
	- Mason- I'll go with your recommendation.
12. **Who owns `admin_alerts` + the digest stored procedure?** I classified them framework (any site wants ops alerts). Confirm.
	- Mason- Yes, I think that should be framework.

# Sources

- [GitLab Conan 2 package registry docs](https://docs.gitlab.com/user/packages/conan_2_repository/)
- [GitLab 18.10 release notes — Conan 2 registry promoted to Beta](https://about.gitlab.com/releases/2026/03/19/gitlab-18-10-released/)
- [GitLab Conan v2 API](https://docs.gitlab.com/api/packages/conan_v2/)
- [Conan issue: sharing built packages via GitLab package registry](https://github.com/conan-io/conan/issues/13765)