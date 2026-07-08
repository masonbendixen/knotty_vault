---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/8/2026
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

- **Multi-tenant SaaS plan (not yet implemented):** components must stay tenancy-agnostic. Providers (`DatabaseHelper`, `SecretsHelper`, `SquareClient`, `MailHelper`) will move behind a per-tenant registry — so components must accept these as injected parameters, never as global singletons. The planned `business_logic/tenancy/` module is itself future shared surface. `EndpointAuthHelper` is the load-bearing seam for both tenancy and this split.
- **Test-speed work (implemented):** the 88% speedup depends on assembling the *complete* `DatabaseInfo` once (`GlobalDatabaseTestSupport::SetupAllTables()` + `SetIfNotExistsMode`). Splitting the schema must preserve a way to compose framework tables + app tables into one `DatabaseInfo` and run the one-time committed setup.
- **AWS deployment (implemented):** one multi-stage Dockerfile produces one image with all binaries and one version string, built by GitLab CI. Packaging changes must not break the single-image pipeline.
- **Scheduler (implemented):** communicates with the server purely over HTTP; only drags in `knotty_yoga_core` for `HttpClient` + logging. Extracting `util/http` frees it.

# Target Component Architecture

## Proposed layered components

Names use a placeholder namespace `<ns>` until we pick one (Open Question 1). Each component depends only on those above it in this list:

```
<ns>_foundation      util core: types, json_value/json_util, logging, thread_pool,
                     date_time_util, error_codes/error_response, file_util,
                     bounding_rect, image_resize, destructive_guard, http/
        ▲
<ns>_data            sql_util core: database_access, schema, json,
                     stored_procedures (generic), database_common (parameterized)
        ▲
<ns>_services        util/secrets (incl. secrets_at_rest moved down), util/mail
        ▲
<ns>_platform        framework db_schema + table_helpers (~33 tables),
                     business_logic/auth core, business_logic/images,
                     business_logic/migration (runner), web core (web_app,
                     middleware guards, endpoint_auth_helper, generic CRUD
                     endpoints + admin metadata), future business_logic/tenancy
        ▲
<ns>_scheduler       scheduler engine (job_scheduler, scheduler, api_client,
                     ScheduledJob struct) — depends only on foundation
        ▲
<ns>_testing         GlobalDatabaseTestSupport, TestDatabaseUtil,
                     TestTransactionProvider, matchers, EndpointTestHelper,
                     test doubles (TestMailHelper, SecretsHelperTest, ...)
─────────────────────────────────────────────────────────────────────────
knottyyoga app       app db_schema/table_helpers (~55 tables), payment/,
(stays in app repo)  scheduling/, skills/, app endpoints, ical branding config,
                     Square client?, database_helper, test_helper, main.cpp,
                     scheduler job catalog
```

Notes:
- `util/square` is a judgment call: it is generic "Square API client" code (reusable by any site that takes payments via Square) but is payment-specific. Suggest: keep it app-side initially, promote to a component later if a second site needs Square (Open Question 9).
- `util/ical_generator` is generic once the PRODID/UID domain are parameterized — goes in `<ns>_foundation` or `<ns>_platform`.
- The generic table CRUD endpoints (`get_row`, `add_item`, `update_item`, `delete_item`, `get_fk_options`, `db_schema`, etc.) plus the `admin_*` metadata system are one of the most valuable reusable assets — they give any new site a free admin data editor.

## Granularity options

**Option A — one component package** ("the platform"). Everything reusable in a single package/repo with internal CMake targets.
- ✅ Trivial version management (one version), one CI, easiest for collaborators.
- ❌ Coarse dependency: a site wanting only the scheduler engine pulls Postgres/libpqxx/mailio/etc.

**Option B — the six layered packages above, separately versioned.**
- ✅ Precise dependencies; clean layering enforced by packaging.
- ❌ Heavy version-management ceremony (bump foundation → rev data → rev services → …) while the code is still evolving fast. This is the classic "too many packages too early" trap.

**Option C (recommended) — one repo, multiple CMake targets, one version.** A single shared repo (and later a single Conan package, or one package per target with lockstep versions) exposing `<ns>_foundation`, `<ns>_data`, `<ns>_services`, `<ns>_platform`, `<ns>_scheduler`, `<ns>_testing` as separate link targets. Layering is enforced by target link rules inside the repo, but you release everything together.
- ✅ One version to pin, one CI, layering still build-enforced, consumers link only the targets they need. Easy to split into Option B later (the target boundaries are already the package boundaries).
- ❌ A consumer's `conan install` builds the whole repo even if it links one target (acceptable at this scale).

**Suggestion: Option C.** Fine-grained packaging is easy to add later and painful to retract.

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

**Suggestion: R2 now, revisit R3 later.** If it turns out the friends mostly want to build the *new sites* with you (not just the components), R3 becomes more attractive and the app can migrate then. Migrating a git repo between hosts is cheap; nothing here is one-way.

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

**Suggestion: P1 (FetchContent) for Phase 5, graduating to P4 or P2 once the component API stabilizes.** While you and friends are actively co-developing both sides, source-based consumption with git-SHA pinning has the least ceremony and the fewest moving parts; during local co-development you can point FetchContent at a local checkout (`FETCHCONTENT_SOURCE_DIR_<NAME>`) for instant edit-rebuild across repos. Adopt versioned Conan packages when a component is stable enough that consumers *shouldn't* track head — that's also the natural moment to decide P2 vs P3/P4 based on where the repos ended up.

# Cross-Cutting Decisions

**Naming.** The components need a brand-neutral name — it becomes the repo name, CMake target prefix, C++ namespace flavor, and env-var prefix (`<NS>_LOG_DEST`). See Open Question 1. Existing C++ namespaces (`Auth`, `SqlUtil`, `Json`, `Secrets`, `Mail`, `Http`) are already brand-neutral and can stay; only file-level constants and target names need renaming.

**License.** A public component repo needs one. Suggest **Apache-2.0** (explicit patent grant, employer-friendly) or MIT (shorter). Open Question 5.

**Versioning.** With Option C: single version for the whole component repo, tagged releases (`v0.x` semantics — breaking changes allowed — until a second consumer exists). The app pins a SHA/tag and upgrades deliberately.

**History.** When extracting, either start the component repo fresh (simplest; GitLab history remains the archaeology) or use `git filter-repo` to carry per-file history over. Suggest fresh start with a "extracted from knottyyoga at SHA …" note. Open Question 11.

**Deployment invariant.** Through every phase, one Docker build must keep producing one image with all binaries and one version string. FetchContent keeps this true automatically (the builder clones the pinned component SHA during `cmake` configure; the Dockerfile needs git + network in the build stage, or a vendored source tarball).

**Windows + Linux.** Components must build in both (you develop on Windows/VS, deploy and test on Linux). Component CI should cover at least Linux (container, matching app CI) and ideally a Windows MSVC job. The Crow `HTTPMethod` PascalCase rule and other Windows gotchas move into the component repo's CLAUDE.md.

# Phased Implementation Plan

Phases 1–4 happen **entirely inside the knottyyoga repo** — no new repos, no packaging, no behavior changes. This is deliberate: extraction is trivial once the boundaries are real, and every phase leaves the app shippable. Phase 5 is the actual extraction; Phase 6 proves reusability. Within each phase, work proceeds lower layers first.

## Phase 1 — Break the coupling (in-place refactors)

### 1.1 util foundation cleanup
- [ ] Parameterize `util/ical_generator` — PRODID and UID domain become constructor/function parameters; knottyyoga values supplied at call sites (likely via a secret/config). Update all call sites + `ical_generator_test.cpp`.
- [ ] Decide env-var naming for `util/logging` (`KNOTTYYOGA_LOG_DEST`) and `util/destructive_guard` (`KNOTTYYOGA_ALLOW_DESTRUCTIVE`): make the prefix a compile-time constant the app sets, or accept a neutral name (`<NS>_LOG_DEST`) with the old name honored during transition. Implement + tests.

### 1.2 sql_util core cleanup
- [ ] Parameterize `kDatabaseName` in `sql_util/database_common.h` — database name becomes a runtime parameter owned by the app (this also aligns with the multi-tenant plan where DB name is per-tenant). Update call sites + tests.
- [ ] Break `sql_util/table_helpers/email_verifications.cpp` → `business_logic/auth/auth_helper.h` back-edge: move the token hashing up into the business_logic/auth caller (or inject a hash function). The table helper becomes pure CRUD. Update tests in both layers.

### 1.3 secrets layer fix (kills the util ↔ auth cycle)
- [ ] Move `secrets_at_rest.{h,cpp}` (+ its test) from `business_logic/auth/` into `util/secrets/` — verify its own includes are util-level (libsodium, types); adjust include paths and both CMakeLists.
- [ ] Split `util/secrets/secret_keys.h` into framework keys (mail, auth/argon2, rate-limit, image sizing) kept in `util/secrets/` and app keys (Square, subscription, scheduling, class check-in, provider, facility) moved to a new app-side header (e.g. `business_logic/app_secret_keys.h` — final location per the Phase 2 target split). Update all includers.
- [ ] Move brand defaults out of `util/secrets/secret_values.cpp` ("Knotty Yoga and Spa", gmail address, subjects, `www.knottyyoga.com`) into an app-side default-registration (secrets framework exposes `RegisterDefaults(...)`; `create_database`/app startup supplies the values). Tests for the registration mechanism + updated existing tests.

### 1.4 mail layer position
- [ ] Document the intended layer order (foundation → data → services) in the server CLAUDE.md; `util/mail`'s include of `sql_util/database_access/transaction.h` is *conformant* under that order (services sit above data). No code change — just make the layering explicit so Phase 2 target boundaries are uncontroversial.

### 1.5 auth cleanup
- [ ] Move `instructor_helper.*` and `instructor_key_value_table.*` (+ tests) out of `business_logic/auth/` into `business_logic/scheduling/` (they're yoga-domain instructor profile code and scheduling already owns instructor concepts). Update includes + CMake.
- [ ] Break `quick_account_helper.cpp` → `payment/gift_permission_helper.h`: add an optional post-creation hook (`std::function<void(Transaction&, int64_t personId)>` or small interface) that the app wires to gift-permission granting at the endpoint layer. Alternative if the hook feels forced: declare `quick_account_helper` app-side and exclude it from the framework (decide during implementation; hook preferred). Tests for both the hook path and the no-hook path.
- [ ] Break `server_config.h` / `cookie_manager.h` → `endpoints/web_app.h`: extract the Crow app typedef + middleware context types into a new lower-level header (e.g. `endpoints/web_app_types.h`) included by both; `web_app.h` re-exports it. (In Phase 2 this header lands in the web-core part of the platform target.) Update includes + tests.

### 1.6 endpoint façade generalization
- [ ] Restructure `EndpointAuthHelper` so the injected-service set isn't hard-wired to Square: framework façade exposes session/cookies/secrets/mail/db; the app layer provides an extended accessor for `SquareClient` (derived helper, or a typed service-registry on `WebApp`). Keep call-site ergonomics (`helper.GetSquareClient()`) via the app-derived type. Update `endpoint_test_helper` symmetrically. Tests.

### 1.7 scheduler catalog extraction
- [ ] Replace `BuildStandardJobs()`'s hard-coded yoga endpoint list and the fixed `SchedulerConfig::intervals` struct with a data-driven job list: engine consumes `std::vector<ScheduledJob>` + per-job interval config; the knottyyoga catalog moves next to `scheduler/main.cpp` (app-side). Flags stay generated from the job list. Tests for engine-with-arbitrary-jobs + existing catalog tests.

## Phase 2 — Enforce boundaries with CMake targets (still one repo)

### 2.1 Foundation and data targets
- [ ] Create `<ns>_foundation` static lib target (util core files per the architecture section) and `<ns>_data` (sql_util core); move `target_sources` entries from `knotty_yoga_core`; wire Conan lib links (Boost, date, PNG/TIFF/ZLIB/jpeg on foundation; PQXX/CURL on data).
- [ ] `knotty_yoga_core` links the new targets PUBLIC so nothing else changes yet.

### 2.2 Services target
- [ ] Create `<ns>_services` (util/secrets, util/mail) linking foundation + data (+ mailio/OpenSSL).

### 2.3 Platform target
- [ ] Create `<ns>_platform`: framework db_schema + framework table_helpers (~33 tables per the assessment list), `business_logic/auth` core, `business_logic/images`, `business_logic/migration`, web core (web_app, middleware guards, endpoint_auth_helper, generic CRUD + admin metadata endpoints, health). This is the big one — expect several sittings; move one sub-group at a time, keeping the build green.
- [ ] Split `db_schema/make_database_info` into framework-table assembly (platform) + app-table assembly (app), composed by the app (preserves FK ordering and the test-speed create-once contract).

### 2.4 App target
- [ ] What remains of `knotty_yoga_core` becomes the app library (app schema/table_helpers, payment/, scheduling/, skills/, app endpoints, util/square, ical branding wiring); rename or keep name (cosmetic).

### 2.5 Scheduler target hygiene
- [ ] `knotty_yoga_scheduler` links `<ns>_foundation` only (post-1.7 it no longer needs core); `main.cpp` (app-side job catalog + service-account env var) keeps its app deps.

### 2.6 Test target split
- [ ] Create `<ns>_testing` from `test/src/util/` (GlobalDatabaseTestSupport, TestDatabaseUtil, TestTransactionProvider, matchers, json/table test utils) + `endpoint_test_helper` + the test doubles (`*_test_util` for secrets/mail/cookies). Fix the duplicate source listing between `test/CMakeLists.txt` and `test/src/util/CMakeLists.txt` while at it.
- [ ] `GlobalDatabaseTestSupport` takes the composed `DatabaseInfo` (framework + app tables) as input rather than calling `MakeDatabaseInfo()` directly, preserving the create-all-tables-once optimization for any consumer. Tests.
- [ ] Keep the single `knottyyoga_tests` executable; it now links app + platform + testing targets.

### 2.7 Layering enforcement
- [ ] Verify no include violations remain across the new target boundaries (each target's headers only reference its own layer or below). Add a small CI-runnable check script (path-prefix include audit) to keep it true.

## Phase 3 — Dry-run the extraction inside the repo

### 3.1 Physical layout
- [ ] Move component-target sources into a top-level `components/` directory tree inside the knottyyoga repo mirroring the future repo layout (`components/foundation/`, `components/data/`, …), updating include prefixes once (mechanical, like the 2023 refactor — ~300 includes). App code keeps `src/`.
- [ ] Component tests live with their code (per testing conventions) and still feed the single test executable.

### 3.2 Bootstrap seam
- [ ] Ensure everything app-specific enters via explicit registration at startup: database name, secret defaults, admin-metadata population, scheduler job catalog, DatabaseInfo composition, endpoint registration. `main.cpp` + `create_database.cpp` become the single composition roots. (Most of this falls out of Phases 1–2; this item is the audit.)

## Phase 4 — Extract to the shared repo

### 4.1 Repo creation
- [ ] Create the component repo (host per Open Question 2), with license, README, CLAUDE.md (layering rules, Windows gotchas, testing conventions), and the `components/` tree moved over (fresh history unless Q11 says otherwise).
- [ ] Component repo builds standalone: own `conanfile.py` (subset of deps), own CMake, own test executable for component tests (data/platform/testing tests need a Postgres container — reuse the `test_container` pattern).

### 4.2 Consumption from the app
- [ ] knottyyoga consumes via FetchContent pinned to a tag/SHA (P1), with `FETCHCONTENT_SOURCE_DIR` override documented for local co-development. Component tests that moved out are removed from the app's test lib; app test suite still green.
- [ ] Update `package/Dockerfile` / `build_linux_release.sh` so the single-image build fetches the pinned component source (git in builder stage or vendored tarball). One image, all binaries, one version string — unchanged contract.

### 4.3 CI
- [ ] Component repo CI (GitHub Actions if R2/R3): Linux build + full component test run with Postgres service; optionally a Windows MSVC build job.
- [ ] App CI keeps working against the pinned component version.

### 4.4 Packaging graduation (deferred until API stabilizes)
- [ ] Add a `conanfile.py` package recipe to the component repo and publish v0.x to the chosen registry (P2 GitLab / P3 Cloudsmith-Artifactory / P4 local-recipes-index per Open Question 3); switch the app from FetchContent to `conan install` of the pinned version.

## Phase 5 — Prove reuse with a second consumer

### 5.1 Example server
- [ ] Build a minimal `examples/hello_server` inside the component repo: boots WebApp + auth (register/login/sessions/RBAC) + admin CRUD over framework tables + images, with its own tiny `main.cpp` composition root and one example domain table. This is executable documentation, the onboarding path for friends, and the regression test that no yoga leaked in.
- [ ] Its existence gates calling the extraction "done": if the example needs to include anything from knottyyoga, a boundary is broken.

### 5.2 Docs & onboarding
- [ ] README quick-start (clone, conan, cmake, run example, run tests) verified on a clean machine; short "how to start a new site on the platform" doc.

# Alternatives Considered (and why not)

- **Copy-paste fork per new site.** Fast to start, but you named the goal: shared code across sites with friends. Divergence cost arrives within months. Rejected.
- **Extract first, decouple later** (move code to a new repo now, fix coupling there). The back-edges make the code un-liftable today — you'd copy the whole core and be back to a fork. Decouple-in-place first is strictly safer; every Phase 1–3 step ships. Rejected.
- **One mega "platform" repo including the frontend.** Angular admin-UI sharing is genuinely valuable (the generic CRUD UI pairs with the generic endpoints) but is a separate npm-flavored effort with different packaging; scoping it in now doubles the project. Deferred — see Open Question 7.
- **Microservice split** (auth service, image service as separate processes). Massive operational cost for a single-EC2 deployment; the multi-tenant plan already chose one process serving all tenants. Library components, not services. Rejected.

# Open Questions

Please add answers inline; I'll fold them into the plan and adjust phases.

1. **Component name/namespace?** Needs to be brand-neutral: repo name, CMake target prefix (`<ns>_platform`), env-var prefix. Ideas to react to: `crowbase`, `croft`, `stonework`, `keystone`, `loom` — or pick your own. (Also: is this maybe a product someday, which would argue for a more distinctive name and Apache-2.0?)
	- Mason- let's go with honuware
2. **Repo hosting:** I recommend R2 (components public on GitHub, app stays private on GitLab). Confirm? And: will the friends work only on the components, or also on the new sites / the yoga app itself? (If the latter, R3 — everything on GitHub — gets stronger.)
	- Mason- The components and new sites will be on Github. I kind of like the app staying on Gitlab and keeping that private. I don't see any of them working on the app in general.
3. **Packaging:** I recommend P1 (FetchContent, SHA-pinned) first, graduating to a Conan registry later (GitLab's Conan 2 registry is Beta as of 18.10; Cloudsmith/Artifactory or local-recipes-index are the GitHub-side options). OK, or do you want real Conan packages from day one?
	- Mason- So CMake or Conan would fetch things from git by SHA?
4. **Granularity:** I recommend Option C (one repo/version, six CMake targets). OK, or do you prefer a single monolithic target (simpler) / fully separate packages (Option B)?
	- Mason- I'll go with your recommendation.
5. **License** for the public repo: Apache-2.0 (my suggestion) or MIT?
	- Mason- Can you list the advantages of each?
6. **Timing vs multi-tenancy:** Phases 1–2 here overlap heavily with the tenant plan's "de-singleton" prep and make it easier. Do components come first, tenancy first, or interleaved (my suggestion: Phases 1–2 now, then tenancy, then extraction — tenancy lands *inside* the platform component)?
	- Mason- I haven't deployed the site yet. I was waiting to finish a few features and do multi tenant but I have friends wanting to work on a site now that uses my components. If I do the switch to multi tenant after doing this componentization but before deploying, will it be that hard?
7. **Frontend sharing:** in scope eventually? (Shared Angular admin CRUD UI, auth pages, header/footer as an npm package.) I've kept it out of this plan; confirm or ask me to add a frontend track.
	- Mason- I can see sharing some frontend components later but I think that we can tackle that as a separate job. I still need to figure that out for the multi tenant thing eventually.
8. **Quick accounts:** framework feature (with the gift-permission hook) or app-specific? I leaned framework-with-hook; fine either way.
	- Mason- I like putting this into the framework. The quick accounts kick ass and will be useful in all sites.
9. **`util/square`:** stay app-side for now (my suggestion) or extract as a `<ns>_square` component immediately?
	- Mason- I'd like to extract this as a component now. I can see many sites wanting to take payment.
10. **Windows CI** for the component repo: worth a Windows MSVC job from day one (you develop on Windows), or Linux-only CI with Windows verified manually?
11. **Git history:** fresh component repo with an "extracted from" note (my suggestion), or `git filter-repo` to preserve per-file history?
12. **Who owns `admin_alerts` + the digest stored procedure?** I classified them framework (any site wants ops alerts). Confirm.

# Sources

- [GitLab Conan 2 package registry docs](https://docs.gitlab.com/user/packages/conan_2_repository/)
- [GitLab 18.10 release notes — Conan 2 registry promoted to Beta](https://about.gitlab.com/releases/2026/03/19/gitlab-18-10-released/)
- [GitLab Conan v2 API](https://docs.gitlab.com/api/packages/conan_v2/)
- [Conan issue: sharing built packages via GitLab package registry](https://github.com/conan-io/conan/issues/13765)