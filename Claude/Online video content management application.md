---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/6/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

In running my acrobatics business, there is a lot of great content to learn from online. In particular, Instagram is a treasure trove of great material. YouTube to a degree. I see interesting videos and save them in Instagram. I end up manually copying the URLs to a command line with yt-dlp. That makes a local copy of the file. Then I need to watch it, rename it something sensible, catalog it (rope, partner acro, handstands, fitness idea) and place it within a folder for the catalog under a directory structure like {category}/{year}/{month}/{day}/{file}. I then take notes in Obsidian but reference the file and try and list timestamps of interesting parts in the video. It's a painful process.

Ideally I would have an app that could:
- Show a UI with the list of videos in Instagram that have not been downloaded locally from my saved videos list
- Do playback of the video or host the page embedded with the video to play it
- Choose to download the video which would start the download of the video in the background but allow me to keep watching and initiating the download of more videos
- Be able to go through the videos I've downloaded
	- Have metadata for the video (creator, original URL, title, category, when downloaded, etc)
	- Be able to change the name of the file in the UI that changes the file on disk
	- Categorize the file which causes it to be placed in the correct area on disk (they default to Inbox/{year}/{month}/{day_downloaded}/{filename})
	- Be able to go into categories and drill down into category/year/month/day
	- Be able to put a description on the video
	- Be able to assign tags (have a tag database with auto complete)
	- Be able to do a search by year / month / category / keyword / tag
	- Watch a video in the UI
		- Be able to watch in the window or full screen
		- Be able to control playback speed (.25x/.5x/.75x/1x/1.25x/1.5x/2x)
		- Be able to slide around within the video and see timestamp and time remaining
		- Have keyboard shortcuts for playback speed, play, pause, go forward 5/10 seconds and backwards
		- Be able to pause and add a note at a specific timestamp
		- Have a note show up during playback when the timestamp is hit and stay on screen until another timestamp with a note replaces it
		- Be able to see the list of notes in the UI and be able to click on a note and have playback jump to that time signature
		- Be able to edit and delete notes

I'm not sure how best to do this app. I'm leaning towards a native C++ app using Qt to do windowing and video playback that hosts sqllite to store data and calls libraries or calls external tools to do things like enumerate the Instagram saved list and download the videos from Instagram. I suppose I could go with a Crow webserver in C++ with a Angular frontend. I feel like the C++ app might be more powerful and easier to configure. What are your thoughts? Also give suggestions on things to download the instagram videos and enumerate the saved list of videos.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Architecture Assessment

## What I found in your environment

- yt-dlp `2026.07.04` and ffmpeg/ffprobe are already installed (winget), Python 3.13.1, CMake, Conan 2.15.1, Docker 27.5.1, Node 22.14.0 / npm 11.1. No Qt installation exists on this machine.
- You have the `knottyyoga` stack (Crow C++ server + Angular 21 + Material/Tailwind + PostgreSQL in Docker) and the reusable `honuware` component libraries (foundation: types/json/logging/thread_pool/file_util/http client; data: transactions/schema; platform: table_helpers framework, generic CRUD + admin endpoints, auth, web core) consumed via CMake FetchContent — plus an established GoogleTest setup with transaction-abort test cleanup.

## Options considered

**Option A — Native Qt 6 C++ desktop app** (your initial lean)
- How it meets the requirements: Qt Widgets UI, QMediaPlayer/QVideoWidget (FFmpeg backend, default on Windows in Qt 6.5+) for playback with `setPlaybackRate` (0.25x–2x), seek, fullscreen; QShortcut for keys; SQLite via Qt SQL; QProcess for yt-dlp; QtWebEngine (embedded Chromium) to show/log into Instagram in-app and harvest cookies.
- Pros: single-window "real app" feel; direct filesystem access; embedded Instagram login/preview is elegant; no server + browser juggling.
- Cons: an entirely new framework for you (multi-GB Qt install, new build kit, LGPL deployment care); Qt Widgets is much more verbose than Angular+Material for grids/forms/autocomplete; QMediaPlayer is decent but the browser `<video>` element is a more mature player; **zero reuse** of honuware, your Angular skills, your Postgres data layer, or your test infrastructure; desktop-only (no tablet in the studio).
- Variant: Qt + libmpv for a best-in-class player — even more power, even more new surface.

**Option B — Crow C++ server + Angular front end, running locally (recommended)**
- Every "hard" UI requirement is a native strength of the browser: `<video>` gives playback speed (0.25x–2x via `playbackRate`), frame-accurate seeking, fullscreen API; overlays/notes are trivial HTML/CSS; Material chips + autocomplete for tags; keyboard shortcuts are simple event handlers. The server side has identical power to a native app (it's local C++ with full disk access — rename/move/download all server-side).
- Massive reuse: honuware foundation (logging, json_value, thread_pool, http client, file_util), data layer (transactions), platform (table_helpers pattern, generic CRUD + free admin tables UI, endpoint/test harness), your Conan/CMake/VS workflow, your Angular 21 shell patterns, and your transaction-abort test framework.
- Bonus: reachable from a tablet/TV on your LAN later (watching reference video with notes *in the studio* seems like a real use case for an acrobatics business).
- Cons: two processes (server + browser — mitigated with a launcher script and Edge/Chrome `--app=` mode, which gives a chromeless app-like window); must implement an HTTP Range streaming endpoint for seek-able video (well-understood, planned below); can't embed instagram.com in an iframe (mitigation: cookie export + open-post-in-new-tab / stream-preview, below).

**Option C — Web backend now, desktop shell later**
- Option B's backend is UI-agnostic; if you ever want a "real window," wrap the Angular app in Edge app-mode (zero work), or Tauri/Electron later. This is a door Option B leaves open, not a separate build.

## Recommendation → Decision (2026-08-06)

My initial recommendation was Option B, leaning on four pillars: honuware/Postgres reuse, Angular familiarity, LAN/tablet access, and avoiding a new framework. Mason's answers to the open questions knock out three of them: SQLite is the choice (so the Postgres data layer, table_helpers, admin CRUD, and DB test infrastructure don't transfer anyway), LAN access is explicitly not wanted, and the goal is a **self-contained portable tool** — open a library folder from any machine, no server, no Docker, no browser. With those constraints, **Option A (native Qt) is the right call, not a compromise**:

- The portability requirement (library folder = `library.db` + media, copyable between machines, usable by an assistant) is exactly the desktop-app shape. A localhost web stack would fight it.
- Chrome being the browser actually *strengthens* Qt: Chrome's cookie store can't be read reliably on Windows (app-bound encryption), but a QtWebEngine **embedded Instagram login inside the app** produces its own cookies — one login powers yt-dlp downloads, Instaloader enumeration, and in-app post preview (wanted per Q12).
- Honest residual risks, with mitigations: (1) QMediaPlayer (FFmpeg backend) is good but less battle-tested than the browser player — the player sits behind an `IVideoPlayer` seam so libmpv can be swapped in if speed/scrub quality disappoints; (2) Qt is a new framework for Mason — Claude writes the code, and conventions stay knottyyoga-style (naming, layering, GoogleTest, no fixtures); (3) Qt + WebEngine is a chunky one-time install — Phase 0 scripts/documents it.

**Decided stack:** C++20 · Qt 6.8 LTS (Widgets + Multimedia + MultimediaWidgets + Sql + Network + WebEngineWidgets) · SQLite via Qt Sql (FTS5 keyword search with LIKE fallback) · CMake + GoogleTest (FetchContent) · external tools: yt-dlp + ffmpeg/ffprobe (already installed, winget) + Python 3.13 venv with Instaloader. No Conan, no Postgres, no Angular, no web server.

**Portable-library contract:** everything the catalog needs lives under the library root — `library.db` at the root, media under `{Category}/{YYYY}/{MM}/{DD}/`, thumbnails/staging/backups under `.videolibrary/` — and every path stored in the DB is root-relative with forward slashes, so the folder can be copied/moved wholesale and opened on any machine. Per-machine state (tool paths, window geometry, recent libraries, Instagram session/cookies) lives in QSettings/AppData, never inside the library.

## Instagram tooling suggestions

**Downloading (keep yt-dlp).** Instagram now sits almost entirely behind a login wall — anonymous yt-dlp requests fail, so downloads need cookies. Since Mason uses Chrome (whose cookie store can't be read reliably on Windows due to app-bound encryption), the app supplies its own: log into Instagram once inside the app (embedded QtWebEngine browser with a persistent per-machine profile) and the app exports a Netscape `cookies.txt` from that session for yt-dlp (`--cookies <file>`). Fallback if WebEngine is skipped: manual export via a cookies.txt Chrome extension. The app shells out to yt-dlp with politeness delays and surfaces yt-dlp staleness (Instagram breaks old extractors regularly; yt-dlp stays winget-managed).

**Enumerating your saved list** (no official API for personal saved posts — all options are unofficial):
1. **Instaloader** (recommended) — mature Python library; the helper script builds its session from the same cookies the embedded login harvested (`load_session`), then `get_saved_posts()` yields shortcode/URL/owner/caption/date/thumbnail without downloading. We wrap it in a small helper script emitting JSON that the app invokes and reconciles.
2. **instagrapi** — private mobile-API library; more capable (per-collection access) but a heavier risk profile.
3. **gallery-dl** — multi-platform downloader that can target the saved page with cookies; less Instagram-specialized than Instaloader.
4. **Browser extension** that scrapes your saved page in your real session and posts to the local server — lowest ban risk, most build effort. (Fallback idea, not planned.)
5. **Manual paste of URLs** — ships first regardless, as the always-works path (and covers YouTube for free).

**Risk note:** any automated enumeration violates Instagram ToS and can flag the account it runs as. Mitigations planned: manual sync button (not a background poller) with a minimum interval, low volume, politeness delays, cached results, and the option to run against a secondary account (Open Questions #8).

# Open Questions

Answer inline here (or in chat) — defaults in bold are what I'll assume if unanswered.

**Architecture**
1. Approve **Option B (Crow + Angular + honuware + Postgres)**? If you want Qt native anyway, say so and I'll restructure the plan before any code.
	- Mason- I believe that I want to have a native Qt app. This isn't something that I'm going to ship. I have an assistant who downloads these files for me. It would be nice to have a script to run to install the various python things but just have a self contained Windows app to run. It would be nice to be able to run it from any machine and just point it to the SQLLite database at the root of the file system where this stuff is stored (and use relative paths from here so this can be copied around and still opened)
	- → Claude: not a horrible idea at all — with these constraints it's the better fit. Decision locked: native Qt app; plan below rebuilt around it (see Recommendation → Decision).
2. Database: **PostgreSQL (reuse everything, Docker required at runtime)** vs SQLite (self-contained exe, new data layer, no admin/test reuse)?
	- Mason- Yes, I'm aware that using SQL lite unfortunately doesn't allow any reuse of my postgres database code but this also isn't a production app and the database needs are small. I also don't really care about ACID that much per se since I anticipate only one user at a time. Honestly, I could probably just do this as a big JSON file but the SQL support for queries and keyword searches is nice.
	- → Claude: SQLite it is, with FTS5 for keyword search (LIKE fallback). Agreed — SQL queries earn their keep over a JSON blob, and single-user means the ACID trade-offs don't bite.
3. Repo/app name and location? Default: **`C:\Users\mason\source\repos\video_library`**, working title "Video Library", honuware pinned to the same SHA knottyyoga uses. (Name ideas if you want one: Clip Vault, Move Library, Beta Reel.)
	- Mason- This is a fine spot for the repo.
4. Server port **18081** (knottyyoga uses 18080) — OK?
	- Mason- I am strongly leaning not to make this a web page.
	- → Claude: moot — native app, no server, no port.

**Library**
5. Where does your existing video library live on disk, should Phase 3.7 adopt it into the catalog, and what's the real category list to seed? (Overview mentions rope, partner acro, handstands, fitness idea — plus **Inbox** as the default.)
6. Delete behavior: **permanent delete with a confirm dialog**, or send to Recycle Bin?
	- Mason- Send to recycle bin.

**Instagram**
7. Which browser are you logged into Instagram with? (**Firefox → automatic `--cookies-from-browser`**; Chrome/Edge → manual cookies.txt export extension.)
	- Mason- Chrome
	- → Claude: Chrome's cookie store can't be read on Windows (app-bound encryption), so the app embeds its own Instagram login (QtWebEngine) and exports its own cookies — nothing needs to touch Chrome.
8. Run enumeration/downloads as your main account or a **secondary account**? And is "all saved posts" enough, or do you use specific saved *collections* (collections push toward instagrapi)?
	- Mason- Main account.
9. Saved *photo/carousel* posts: **videos only** for now, or should images be imported too (viewer instead of player)?
	- Mason- Just videos.
10. Auto-sync on a schedule, or **manual Sync button only** (safer for rate limits)?
	- Mason- Let's do manual for now.

**Scope**
11. LAN/tablet access wanted? Default **localhost-only, no auth**; LAN adds a small auth + bind item in Phase 6.
	- Mason- This is a big part of why I just want a local app that I can just open a directory with a sql lite file and the directory structure.
	- → Claude: this became the design's centerpiece — see the portable-library contract in the Decision section.
12. Preview-not-yet-downloaded videos: is **open-post-in-new-tab** enough, or do you want in-app streaming preview (Phase 5.4, resolves the direct media URL via yt-dlp on click)?
	- Mason- Given that I want a native app, let's go with in app preview.
	- → Claude: planned (Phase 5.5) — resolve the direct media URL via yt-dlp and play it in the player without saving, or open the post in the embedded logged-in browser.

# Implementation Plan

Layer order within every phase: db_schema → table_helpers → low-level services → business_logic → endpoints → Angular UI. Tests accompany every layer that is testable (GoogleTest server-side per knottyyoga conventions — no fixtures, tests beside sources; Jasmine/Karma for Angular logic). Checkboxes get checked as items are implemented.

## Phase 0 — Project scaffolding and infrastructure

### 0.1 Repository and build skeleton
- [ ] Create repo folder (per Q3) with top-level CMakeLists.txt consuming honuware via FetchContent (pinned SHA), `conanfile.py` as superset of honuware deps, `.gitignore`/`.gitattributes` copied from knottyyoga pattern
- [ ] App targets: `video_library_core` (static lib), `video_library_server` (Crow exe), `video_library_tests` (GoogleTest exe), `video_library_database_helper` (DB bootstrap exe)
- [ ] VS/Windows build config (CMakeSettings.json), README with build steps (you build; steps documented for Windows first)
- [ ] Seed CLAUDE.md distilled from knottyyoga conventions (layering, naming, testing, CMake header listing, planning-directory rules)
- [ ] Smoke test: one trivial GoogleTest proving the test target builds and runs

### 0.2 Database bootstrap
- [ ] New `video_library` PostgreSQL database using the existing database_server container pattern (reuse container/network; `HONUWARE_DB_NAME` override)
- [ ] `video_library_database_helper` following the create_database.cpp pattern (config secrets table via honuware; empty app schema at this point; destructive-guard env var respected)
- [ ] Wire GlobalDatabaseTestSupport so all table tests run inside aborted transactions against pre-created tables
- [ ] Tests: database_helper creates the database idempotently; config secret read/write round-trip

### 0.3 Process runner, tool registry, app configuration
- [ ] `ProcessRunner` low-level service (launch external exe with args, capture stdout/stderr incrementally, exit code, timeout, kill/cancel) — cross-platform (Boost.Process); `TestProcessRunner` fake for tests, plus its own tests
- [ ] App configuration (config secrets pattern): `library_root`, `staging_dir`, `thumbnail_dir`, `yt_dlp_path`, `ffmpeg_path`, `ffprobe_path`, `python_path`, `cookies_file_path` (or browser name), `max_concurrent_downloads`, `instagram_username`
- [ ] `ToolRegistry`: probe yt-dlp/ffmpeg/python versions at startup via ProcessRunner; results exposed on `/api/health`; parsing tests with fake runner
- [ ] Endpoint: `/api/health` reporting server + DB + tool status; endpoint test

### 0.4 Angular workspace scaffold
- [ ] `ng new` Angular 21 app (Material + Tailwind, matching knottyyoga ui conventions and path-alias style), proxy.conf.json → localhost:18081
- [ ] App shell: sidebar navigation (Library, Instagram, Downloads, Search, Settings), header, routing, empty pages
- [ ] `ServerAccessNetwork`-style HTTP service; Settings page showing `/api/health` tool status
- [ ] `ng test` wired with one passing service test

## Phase 1 — Catalog data layer

### 1.1 Schema (db_schema + registration)
- [ ] `categories` (id BIGSERIAL, name UNIQUE, directory_name UNIQUE, created_at) — seeded with Inbox + initial category list (Q5)
- [ ] `videos` (id, category_id FK, relative_path, file_name, title, creator, platform, source_url, source_id UNIQUE NULL, description, duration_ms, width, height, file_size_bytes, downloaded_at, published_at NULL, thumbnail_path, created_at, updated_at)
- [ ] `tags` (id, name UNIQUE case-insensitive) and `video_tags` (video_id FK, tag_id FK, PK(video_id, tag_id))
- [ ] `notes` (id, video_id FK, timestamp_ms, body, created_at, updated_at)
- [ ] `source_items` (id, platform, external_id UNIQUE, url, creator, caption, thumbnail_path, posted_at NULL, first_seen_at, last_seen_at, state: new|queued|downloaded|ignored|gone, video_id FK NULL) — the cached Instagram saved list
- [ ] `downloads` (id, source_item_id FK NULL, url, state: queued|running|success|error|canceled, progress_percent, error_message, staging_path, created_at, started_at, finished_at)
- [ ] Full admin registration for every table (all steps of the knottyyoga "Adding a New Database Table" checklist) so the generic admin CRUD UI works day one
- [ ] Search support: generated tsvector column on videos (title, description, creator) + GIN index
- [ ] Tests: schema creation via database_helper; FK ordering

### 1.2 Table helpers
- [ ] CRUD helpers per table following table_helpers conventions (KeyValueTable in/out, no business logic), each with tests: uniqueness conflicts, FK cascade behavior, `source_items` and `downloads` state transitions, tag find-or-create, tag prefix search (for autocomplete), notes ordered by timestamp

### 1.3 Search queries
- [ ] Filtered video query builder: category, year, month, day (derived from downloaded_at), tags (all-of), keyword (tsvector match on videos + join into notes bodies), with paging + sort
- [ ] Library tree counts query (category → year → month → day with video counts)
- [ ] Tests: month/year boundary correctness, tag intersection, keyword hits in title vs note body, empty-filter behavior

### 1.4 Path planning and file operations (business_logic/library)
- [ ] `PathPlanner`: target path `{directory_name}/{YYYY}/{MM}/{DD}/{file_name}` from category + downloaded_at; Windows-safe filename sanitization (reserved chars/names, length cap); collision resolution via numeric suffix; tests
- [ ] `FileOperations`: rename in place, move-with-directory-creation, cross-volume fallback, delete; disk-op-first-then-DB update sequencing with revert on DB failure; tests using temp directories

## Phase 2 — Acquisition pipeline (after this phase the app already replaces the manual yt-dlp workflow)

### 2.1 yt-dlp client (business_logic/acquisition)
- [ ] Metadata probe: `--dump-single-json` → `VideoMetadata` struct (id, title, uploader, duration, dimensions, canonical URL, upload timestamp, thumbnail URL); URL canonicalization for Instagram share links (reel/p/tv forms, strip `igsh` params)
- [ ] Download: into staging_dir with output template, parseable progress (`--newline --progress-template`), cookie args from config, cancel via process kill, error taxonomy (auth required / gone / network / unknown)
- [ ] Tests against captured fixture outputs with TestProcessRunner — no network in tests

### 2.2 Download manager
- [ ] Queue backed by `downloads` table; worker slots on honuware thread_pool (default 2 concurrent); state machine queued→running→success|error|canceled; retry; startup re-queue of interrupted items; progress readable for polling
- [ ] Tests with fake yt-dlp client: concurrency cap, cancel mid-download, error propagation, restart recovery

### 2.3 Import service
- [ ] On download success: ffprobe metadata, thumbnail extraction (ffmpeg frame grab into thumbnail_dir), PathPlanner target under `Inbox/{download date}`, move from staging, insert `videos` row, link + mark `source_items` row downloaded; duplicate detection by source_id (skip + surface)
- [ ] Tests: full import flow with temp library root and fake tools; duplicate handling; failure mid-import leaves staging intact

### 2.4 Acquisition endpoints
- [ ] `POST /api/videos/add_by_url` (accepts one or many URLs → probe + enqueue), `GET /api/downloads` (with progress), `POST /api/downloads/{id}/cancel`, `POST /api/downloads/{id}/retry` — thin endpoints per conventions, with endpoint tests

### 2.5 Angular: Downloads page + Add-by-URL
- [ ] Add-by-URL box (multi-line paste) in the shell toolbar
- [ ] Downloads page: rows with progress bars (1s polling), cancel/retry, error details, link to resulting video
- [ ] Jasmine tests for the download-state mapping service

## Phase 3 — Library browsing and management

### 3.1 Query endpoints
- [ ] `GET /api/library/tree` (categories → year → month → day + counts), `GET /api/videos` (filters + paging), `GET /api/videos/{id}` (full metadata incl. tags), `GET /api/videos/{id}/thumbnail`, `GET /api/tags?prefix=` (autocomplete); endpoint tests

### 3.2 Mutation endpoints
- [ ] Rename (new file_name → disk rename + DB), recategorize (→ disk move to `{category}/{y}/{m}/{d}` + DB), edit title/creator/description, set/add/remove tags (find-or-create), delete video (per Q6); endpoint tests exercising real disk effects under a temp library root

### 3.3 Angular: library browse
- [ ] Sidebar tree component: category → year → month → day with counts, selection drives the grid
- [ ] Video grid: thumbnail, title, creator, duration, downloaded date, tag chips; sorting (date, title, duration)

### 3.4 Angular: video details editor
- [ ] Details panel: inline rename, category picker (shows resulting path), description editor, creator/source URL display, size/resolution/date metadata
- [ ] Tag editor: Material chips with autocomplete from `/api/tags?prefix=` + create-on-enter
- [ ] Delete with confirmation
- [ ] Jasmine tests for edit-state service logic

### 3.5 Angular: search page
- [ ] Keyword box + filters (category, year/month, tags multi-select) → results grid; filter-serialization tests

### 3.6 Existing-library adoption
- [ ] Scanner service: walk an existing `{category}/{year}/{month}/{day}` tree, dry-run report (what would be created), apply mode (create categories/videos, ffprobe metadata, downloaded_at from folder path, generate thumbnails); tests with fabricated temp trees
- [ ] Endpoints + Settings-page trigger with dry-run preview and apply

## Phase 4 — Player and notes

### 4.1 Range streaming endpoint
- [ ] `GET /api/videos/{id}/stream` with HTTP Range support (206 Partial Content, single ranges, suffix ranges, 416 handling, chunk-size cap, correct Content-Type) — required for instant seeking in `<video>`
- [ ] Tests: no-range, mid-file range, suffix range, out-of-bounds, byte-exact content verification

### 4.2 Notes endpoints
- [ ] CRUD: create at timestamp_ms, edit body/timestamp, delete, list ordered by timestamp; endpoint tests

### 4.3 Angular: player component
- [ ] `<video>` wrapper playing `/api/videos/{id}/stream`; controls bar: play/pause, speed selector (0.25/0.5/0.75/1/1.25/1.5/2), seek slider with elapsed and remaining time, volume/mute, fullscreen (in-window and Fullscreen API)
- [ ] Remember last playback position per video (nice-to-have)

### 4.4 Keyboard shortcuts
- [ ] Space play/pause, ←/→ ±5s, Shift+←/→ ±10s, ↑/↓ speed step, F fullscreen, M mute, N new note, ? shortcut overlay — data-driven shortcut map service with Jasmine tests

### 4.5 Notes UX
- [ ] Add note: pauses playback, inline editor pre-stamped with current timestamp
- [ ] Overlay on the video showing the active note — active = most recent note at or before current time, stays until the next note's timestamp; pure resolver function with Jasmine tests (before-first, between, exact-hit, after-last, note added/deleted mid-playback)
- [ ] Notes side list: click seeks to timestamp, inline edit, delete with confirm
- [ ] "Copy notes as Markdown" (H:MM:SS + text) for pasting into Obsidian

## Phase 5 — Instagram saved-list integration

### 5.1 Cookie/session setup
- [ ] Settings UI + config for cookie source (cookies.txt path or `--cookies-from-browser firefox`); server-side login-status probe (yt-dlp simulate against an auth-required URL) surfaced in Settings; cookie-file format validation with tests

### 5.2 Saved-list enumeration
- [ ] Python helper `tools/list_saved_posts.py` using Instaloader (session created once via `instaloader --login`, supports 2FA): emits JSON lines (shortcode, url, owner, caption excerpt, posted_at, is_video, thumbnail_url) with politeness delays and a max-items argument
- [ ] `SavedListSyncService`: run helper via ProcessRunner, parse, reconcile into `source_items` (insert new, refresh last_seen, preserve ignored/downloaded states), download thumbnails server-side into thumbnail cache; minimum-interval guard between syncs
- [ ] `POST /api/instagram/sync` + status endpoint; tests with fixture JSONL covering the reconciliation state machine (new/known/ignored/downloaded/gone)

### 5.3 Angular: Instagram page
- [ ] Grid of not-yet-downloaded saved items (thumbnail, creator, caption snippet), state filter, Sync button with last-synced time
- [ ] Actions: Download (single + multi-select → Phase 2 pipeline), Ignore, Open post in new tab; rows auto-update as downloads complete

### 5.4 Preview without download (per Q12)
- [ ] Preview action: server resolves the direct media URL on click (yt-dlp `-g`) and the player streams it without saving; fallback is open-in-new-tab

### 5.5 Optional per Q10
- [ ] Scheduled background sync via the honuware scheduler pattern (default off)

### 5.6 YouTube parity
- [ ] Verify add-by-URL handles YouTube end-to-end (formats, thumbnails, metadata); document Watch-Later enumeration (cookies + `:ytwatchlater`) as a future provider

## Phase 6 — Hardening and polish

### 6.1 Consistency checker
- [ ] Report: DB rows with missing files, files under library_root not in DB, stale source_item links; repair actions (relink, adopt, remove row); tests on checker logic; Settings-page UI

### 6.2 Tool health and updates
- [ ] yt-dlp version staleness surfaced in Settings (Instagram breaks old versions); update guidance (winget)

### 6.3 Backup
- [ ] pg_dump backup script + restore documentation (media files are plain files covered by normal backup; DB holds the catalog)

### 6.4 Launcher
- [ ] `start_video_library.cmd`: ensure Postgres container running, start server, open browser in app-mode window; README run-book

### 6.5 UX polish
- [ ] Empty states, toasts for background completions/errors, busy indicators, keyboard cheat-sheet overlay, dark theme pass

### 6.6 Optional per Q11
- [ ] LAN access: bind beyond localhost + honuware auth login

# Verification

Per phase, after I finish the code (you build and run — I won't invoke builds or git):
- **Server tests**: build and run `video_library_tests` (GoogleTest; DB tests run in aborted transactions against the dev database).
- **Angular tests**: `ng test` in the ui folder.
- **Manual smoke per phase**: 0 — server starts, `/api/health` shows tools green, Angular shell loads. 1 — admin CRUD pages show the new tables. 2 — paste a real Instagram/YouTube URL, watch it download in the background and land in `Inbox/{y}/{m}/{d}` with a thumbnail; queue two while one runs. 3 — browse the tree, rename a file (verify on disk), recategorize (verify the file moved), tag with autocomplete, search by tag/keyword/date. 4 — play a video: all speeds, scrub, fullscreen, every shortcut; add notes at two timestamps and confirm the overlay switches at the second; click a note to jump; copy notes as Markdown into Obsidian. 5 — Sync shows saved posts not yet downloaded; download two at once; ignore one; confirm re-sync preserves states. 6 — kill a file on disk manually and confirm the checker reports it; launcher cold-starts everything.

# Process Notes

- Mason builds the server, runs all tests, and handles all git commits/pushes; Claude writes code + tests, checks off plan checkboxes as items complete, and avoids git and external prompt-requiring tools (internal file tools only).
- Questions go into the Open Questions section of this document, not interactive prompts.
- Every change that can be tested gets tests (GoogleTest server-side, Jasmine/Karma for Angular logic), following knottyyoga conventions: no test fixtures, tests beside sources, no assumed collection order.