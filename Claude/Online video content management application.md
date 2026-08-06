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
- Mason- I very much want Conan. I also want a script that does winget, etc to pull down the dependencies onto a clean machine. The only reason I have these libraries on my machine is because of Conan. Please look at knotty yoga server and/or server components to do the conan

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

## Remaining open questions (numbering continues; answer inline)

13. (was Q5 — still open) Where does the existing video library live on disk, should Phase 3.6 adopt it into the catalog, and what categories should be seeded besides **Inbox** (rope, partner acro, handstands, fitness idea + …)?
14. Qt install: the official online installer (needs a free Qt account) or an unattended `aqtinstall` script I provide (no account needed)? And which Visual Studio do you have — 2019 or 2022? Qt 6.8's prebuilt Windows binaries are MSVC-2022-built, so VS 2022 is the smooth path.
15. QtWebEngine is the one heavyweight module (roughly a GB installed; ~150 MB added to the deployed app folder). It powers the embedded Instagram login — our only reliable cookie source given Chrome — and in-app post preview. Include it (**my recommendation**), or skip it and manually export `cookies.txt` from a Chrome extension whenever the session expires?
16. How will you and your assistant share the library folder — copied/external drive, NAS, or a synced folder (OneDrive/Dropbox)? SQLite wants one writer at a time: totally fine as long as only one machine has the app open at once. I'll add a stale-lock warning either way, but the real setup shapes the run-book guidance.

# Implementation Plan

Layer order within every phase, lowest first: **data** (schema, repositories) → **low-level services** (process runner, tool registry, settings) → **business_logic** (acquisition, library operations, instagram, player/notes logic) → **ui** (widgets, models, presenters). Dependencies point downward only — knottyyoga's layering discipline, adapted to a desktop app. Checkboxes get checked off as items are implemented.

Design principles baked into every phase:
- **Event-driven, single process**: yt-dlp/ffmpeg/python run as async `QProcess`es on the event loop — background downloads with no worker threads. All SQLite access stays on the GUI thread (Qt SQL connections are thread-bound), which also avoids write contention. Long filesystem scans run on a worker and marshal results back.
- **Portable library**: every path stored in the DB is library-root-relative with forward slashes; `library.db`, thumbnails, staging, and backups all live under the root (`.videolibrary/` for app-managed folders). Per-machine state (tool paths, recent libraries, window geometry, Instagram session/cookies) lives in QSettings/AppData only.
- **Tests**: GoogleTest via FetchContent, knottyyoga conventions — no fixtures, self-contained tests beside sources, no assumed collection order; the test main spins up a `QCoreApplication`. UI widgets stay thin; logic lives in models, presenters, and pure functions where tests can reach it.

## Phase 0 — Toolchain and app skeleton

### 0.1 Toolchain setup (Mason runs; Claude provides scripts and docs)
- [ ] `tools/install_qt.ps1` (aqtinstall-based, per #14) or documented online-installer steps: Qt 6.8 LTS, MSVC 64-bit kit, Multimedia + WebEngine modules (#15); README prerequisites section (VS, CMake ≥ 3.24)
- [ ] `tools/setup_python.ps1`: create a per-machine venv under `%LOCALAPPDATA%\VideoLibrary\python`, `pip install instaloader`, print versions — the "script to install the various python things"
- [ ] Verify yt-dlp/ffmpeg (already winget-installed) resolve from PATH; document the winget lines for a fresh machine (assistant setup)

### 0.2 Repository and build skeleton
- [ ] Repo at `C:\Users\mason\source\repos\video_library`: top-level CMakeLists.txt (C++20, `qt_standard_project_setup`, AUTOMOC) with targets `video_library_core` (static lib — everything testable), `video_library_app` (WIN32 exe — main + UI glue), `video_library_tests` (GoogleTest exe)
- [ ] GoogleTest via FetchContent; custom test `main.cpp` that creates a `QCoreApplication`; CTest wiring
- [ ] CMakePresets/CMakeSettings for VS; `.gitignore` / `.gitattributes`; README with build steps
- [ ] CLAUDE.md seeded with this app's conventions (layer DAG, naming, testing rules, CMake header listing, planning-directory rule)
- [ ] Smoke: one trivial test proving the test target builds and runs

### 0.3 App bootstrap and logging
- [ ] `main.cpp` + `MainWindow` shell: left navigation (Library, Instagram, Downloads, Search, Settings) over a stacked central area, menu bar, status bar
- [ ] Logging: `qInstallMessageHandler` → rotating file in AppData + IDE console; startup banner with app/Qt/tool versions
- [ ] `MachineSettings` (QSettings wrapper): recent libraries, window geometry, tool-path overrides; tests for the serialization logic

### 0.4 Library open/create
- [ ] `LibraryContext`: holds the root; creates `library.db` + `.videolibrary/{thumbnails,staging,backups}` on first open; resolve/relativize helpers that enforce root-relative forward-slash paths and reject escapes; tests (resolve, relativize, containment, slash normalization)
- [ ] Open/Create Library folder flow (folder picker), recent-libraries menu, reopen-last-on-launch, "library missing/moved" handling

## Phase 1 — Data layer (SQLite)

### 1.1 Database manager and migrations
- [ ] `DatabaseManager`: opens `library.db` (QSQLITE), pragmas (`foreign_keys=ON`, `busy_timeout`, `journal_mode=DELETE` for copy-safety), `schema_migrations` table + ordered migration runner
- [ ] FTS5 availability probe at open (`pragma compile_options`) setting a capability flag; LIKE-based fallback path when absent
- [ ] Tests: fresh create, reopen idempotence, migration ordering, probe behavior

### 1.2 Schema v1
- [ ] `categories` (id, name UNIQUE COLLATE NOCASE, directory_name UNIQUE, created_at) — seeded with Inbox (+ #13 list)
- [ ] `videos` (id, category_id FK, relative_path, file_name, title, creator, platform, source_url, source_id UNIQUE NULL, description, duration_ms, width, height, file_size_bytes, downloaded_at, published_at NULL, thumbnail_relative_path, created_at, updated_at)
- [ ] `tags` (id, name UNIQUE COLLATE NOCASE) and `video_tags` (video_id FK, tag_id FK, PK(video_id, tag_id), CASCADE)
- [ ] `notes` (id, video_id FK CASCADE, timestamp_ms, body, created_at, updated_at)
- [ ] `source_items` (id, platform, external_id UNIQUE, url, creator, caption, thumbnail_relative_path, posted_at NULL, first_seen_at, last_seen_at, state: new|queued|downloaded|ignored|gone, video_id FK NULL) — the cached Instagram saved list
- [ ] `downloads` (id, source_item_id FK NULL, url, state: queued|running|success|error|canceled, progress_percent, error_message, staging_name, created_at, started_at, finished_at)
- [ ] `library_settings` (key, value) — per-library preferences (e.g., last playback positions)
- [ ] FTS5: `videos_fts` (title, description, creator) and `notes_fts` (body) as external-content tables with sync triggers; indexes on FKs, downloaded_at, and state columns
- [ ] Tests: schema creation, FK cascade behavior, FTS trigger sync on insert/update/delete, fallback parity when FTS5 is flagged off

### 1.3 Repositories
- [ ] One repository per table, thin CRUD only (knottyyoga table_helpers spirit): `CategoryRepository`, `VideoRepository`, `TagRepository` (find-or-create, prefix search for autocomplete), `NoteRepository` (ordered by timestamp), `SourceItemRepository` (state transitions), `DownloadRepository` (queue queries, startup requeue), `LibrarySettingsRepository`
- [ ] Tests per repository on temp-file DBs: uniqueness conflicts, cascades, state transitions, ordering, prefix search

### 1.4 Search queries
- [ ] `VideoQuery`: filters (category, year/month/day derived from downloaded_at, tags all-of, keyword via FTS5-or-LIKE across title/description/creator/notes), sort (date/title/duration), paging
- [ ] `LibraryTreeQuery`: category → year → month → day with video counts
- [ ] Tests: month/year boundaries, tag intersection, keyword in title vs note body, empty filters, both search backends

### 1.5 Path planning and file operations
- [ ] `PathPlanner`: `{directory_name}/{YYYY}/{MM}/{DD}/{file_name}` from category + date; Windows-safe sanitization (illegal characters, reserved device names, trailing dots/spaces, length cap); collision resolution via numeric suffix
- [ ] `FileOperations`: rename, move-with-directory-creation (same-volume fast path), delete → `QFile::moveToTrash` (Q6), disk-op-first-then-DB sequencing helper with revert on DB failure
- [ ] Tests with QTemporaryDir: happy paths, collisions, sanitization table, trash behavior

## Phase 2 — Acquisition pipeline (milestone: replaces the manual yt-dlp workflow)

### 2.1 Process runner
- [ ] `IProcessRunner` + `QProcessRunner` (async start, line-buffered stdout/stderr signals, exit code, kill/cancel, timeout) and `TestProcessRunner` scripted fake (the knottyyoga TestHttpClient pattern)
- [ ] Tests: line splitting across chunk boundaries, exit/error propagation, cancel, fake behaviors

### 2.2 Tool registry
- [ ] Resolve yt-dlp/ffmpeg/ffprobe/python(venv) from machine-settings overrides → PATH; async version probes; status model consumed by the Settings view
- [ ] Tests with the fake runner: found/missing/bad-output parsing

### 2.3 yt-dlp client
- [ ] URL canonicalization for Instagram (reel/p/tv forms, strip `igsh`/share params) and YouTube passthrough; dedupe key extraction
- [ ] Metadata probe: `--dump-single-json` → `VideoMetadata` struct (id, title, uploader, duration, dimensions, canonical URL, upload date, thumbnail URL)
- [ ] Download into `.videolibrary/staging` with output template, `--newline --progress-template` progress parsing, `--cookies <file>` when configured, cancel via process kill, error taxonomy (login-required / gone / network / unknown)
- [ ] Tests against captured fixture outputs with TestProcessRunner — no network in tests

### 2.4 Download manager
- [ ] Queue persisted in `downloads`; up to N concurrent async QProcesses (default 2); state machine queued→running→success|error|canceled; retry; startup requeue of interrupted items; progress/state signals for the UI
- [ ] Tests with a scripted fake client: concurrency cap, mid-download cancel, error propagation, restart recovery

### 2.5 Import service
- [ ] On success: ffprobe metadata, thumbnail frame-grab into `.videolibrary/thumbnails/`, PathPlanner target under `Inbox/{download date}`, same-volume move out of staging, `videos` row insert, `source_items` link/mark downloaded; duplicate detection by source_id (skip + surface)
- [ ] Tests: full flow with temp root and fake tools; duplicates surfaced not re-imported; mid-import failure leaves staging intact

### 2.6 UI: Add-by-URL and Downloads view
- [ ] Add by URL action (toolbar + menu): multi-line paste dialog → probe + enqueue
- [ ] Downloads view: rows with progress bars, cancel/retry, error details, jump-to-video on success; status-bar active-downloads indicator
- [ ] Tests for the download-row view-model state mapping

## Phase 3 — Library browsing and management

### 3.1 Library tree
- [ ] Tree dock (Category → Year → Month → Day with counts) as a QAbstractItemModel over `LibraryTreeQuery`; selection drives the grid; refreshes on library changes; model tests

### 3.2 Video grid
- [ ] `VideoListModel` over `VideoQuery` + QListView icon mode with a thumbnail delegate (async thumbnail loading); sorting (date/title/duration); model tests

### 3.3 Details panel
- [ ] Inline rename (disk + DB), category picker showing the resulting path (disk move), description editor, creator/source-URL (opens browser)/date/size/resolution display, Open in Explorer, delete → Recycle Bin with confirm
- [ ] Presenter tests for every mutation path (rename collision, move-failure revert, delete)

### 3.4 Tag editor
- [ ] Chip-style tag row + QCompleter over TagRepository prefix search; Enter creates a new tag; per-chip remove; tests for assignment logic and the completer model

### 3.5 Search view
- [ ] Keyword box + filters (category, year/month, tags multi-select) driving the same grid; filter-state model with tests

### 3.6 Existing-library adoption (per #13)
- [ ] Background scan of a structured `{category}/{year}/{month}/{day}` tree (worker, filesystem-only) → dry-run report dialog → apply: categories/videos rows, ffprobe metadata, thumbnails, downloaded_at from the folder path
- [ ] Tests on fabricated temp trees: dry-run counts, apply idempotence, malformed folders skipped with a report

## Phase 4 — Player and notes

### 4.1 Player controller
- [ ] `IVideoPlayer` seam + QMediaPlayer/QAudioOutput implementation (libmpv is the swap-in if playback quality disappoints); state, speed presets 0.25/0.5/0.75/1/1.25/1.5/2, seek, throttled position ticks
- [ ] Controller logic tests against a fake player (speed cycling, seek clamping, tick throttling)

### 4.2 Player view
- [ ] QGraphicsVideoItem-based view (overlay-capable); controls bar: play/pause, speed selector, seek slider with elapsed and remaining time, volume/mute, fullscreen toggle; in-window and true fullscreen; remembers last playback position per video (`library_settings`)

### 4.3 Keyboard shortcuts
- [ ] Space play/pause; ←/→ ±5s; Shift+←/→ ±10s; ↑/↓ speed step; F fullscreen; M mute; N new note; Esc exits fullscreen; ? cheat-sheet overlay — data-driven `ShortcutMap` with tests

### 4.4 Active note resolution
- [ ] Pure `ActiveNoteResolver`: active note = most recent note at or before current time, held until the next note's timestamp; tests (before first, between, exact hit, after last, add/edit/delete mid-playback)

### 4.5 Notes UX
- [ ] Add note: pauses playback, inline editor pre-stamped with the current timestamp
- [ ] On-video overlay showing the active note, replaced when the next noted timestamp is reached
- [ ] Notes dock: click seeks to the timestamp, inline edit, delete with confirm — list-model tests
- [ ] Copy notes as Markdown (H:MM:SS + body) for pasting into Obsidian; formatter tests

## Phase 5 — Instagram integration

### 5.1 Embedded login and cookies (per #15)
- [ ] QtWebEngine login window with a persistent per-machine profile; `CookieHarvester` on QWebEngineCookieStore capturing instagram.com cookies; Netscape `cookies.txt` writer for yt-dlp; session values handed to the Python helper; login-status indicator + re-login flow
- [ ] Tests: Netscape serialization, cookie filtering/expiry logic (pure functions)

### 5.2 Saved-list helper script
- [ ] `tools/list_saved_posts.py`: Instaloader session from the harvested cookies (`load_session`), iterate saved posts, emit JSONL (shortcode, url, owner, caption excerpt, taken_at, is_video, thumbnail_url), `--max-items`, politeness delays; runs in the venv from 0.1
- [ ] C++ JSONL parser with tests on fixture output (the script itself is smoke-tested manually — it hits the real network)

### 5.3 Sync service
- [ ] `SavedListSyncService`: run the helper via ProcessRunner, reconcile into `source_items` (insert new, refresh last_seen, preserve ignored/downloaded, flag gone), videos-only filter (Q9), thumbnail fetch (QNetworkAccessManager → `.videolibrary/thumbnails/sources/`), minimum-interval sync guard (Q10)
- [ ] Tests: reconciliation state machine on fixtures (new/known/ignored/downloaded/gone), interval guard

### 5.4 Instagram view
- [ ] Grid of not-yet-downloaded saved items (thumbnail, creator, caption snippet, saved date), Sync button with last-synced time, state filter
- [ ] Actions: Download (single + multi-select → Phase 2 pipeline), Ignore, Preview, Open post (embedded browser); rows auto-update as downloads complete

### 5.5 In-app preview (per Q12)
- [ ] Preview resolves the direct media URL on click (yt-dlp `-g`; the URLs expire quickly, so resolve per view) and plays it in the player without saving; alternate action opens the post page in the embedded logged-in browser

### 5.6 YouTube parity
- [ ] Verify Add-by-URL handles YouTube end-to-end (formats, thumbnail, metadata); document Watch-Later enumeration (cookies + `:ytwatchlater`) as a future provider

## Phase 6 — Hardening, portability, packaging

### 6.1 Consistency checker
- [ ] Report: DB rows with missing files, files under the root not in the DB, stale source_item links; repair actions (relink, adopt, remove row); checker-logic tests; Settings section UI

### 6.2 Tool health
- [ ] Settings panel: tool versions, yt-dlp staleness warning (Instagram breaks old versions; winget upgrade hint), python venv health, Instagram session status

### 6.3 Portability and backup (per #16)
- [ ] On open: rotate a copy of `library.db` into `.videolibrary/backups` (keep 5) + `PRAGMA quick_check`; stale-lock detection with a clear "library in use elsewhere / copied while open?" warning; relative-path audit test (no absolute paths ever stored)

### 6.4 Packaging and per-machine setup
- [ ] `tools/make_dist.ps1`: windeployqt into a self-contained folder (copy to any Windows machine), app icon, version stamp
- [ ] Assistant run-book in README: copy dist folder, run `setup_python.ps1`, winget lines for yt-dlp/ffmpeg, open the library, log into Instagram, sync + download

### 6.5 UX polish
- [ ] Empty states, busy indicators, status-bar toasts for background completions/errors, dark-theme pass, shortcut cheat sheet

# Verification

Per phase, after Claude finishes the code (Mason builds and runs — no builds or git from Claude):
- **Tests**: build and run `video_library_tests` (CTest or the exe directly). The data/services/business-logic layers are covered there; UI glue is covered by the smoke lists.
- **Manual smoke per phase**: 0 — app launches, creates/opens a library folder, log file appears, Settings shows tool versions. 1 — `library.db` appears with all tables (inspect with any SQLite browser); copy the library folder elsewhere and reopen it. 2 — paste a real Instagram/YouTube URL; it downloads in the background and lands in `Inbox/{y}/{m}/{d}` with a thumbnail; queue two while one runs; cancel one. 3 — browse the tree, rename a file (check disk), recategorize (file moves), tag with autocomplete, search by tag/keyword/date; adopt the existing library and spot-check it. 4 — play a video: every speed, scrub with elapsed/remaining shown, fullscreen, every shortcut; add notes at two timestamps, watch the overlay switch at the second, click a note to jump, copy notes into Obsidian. 5 — log into Instagram in-app; Sync lists saved videos not yet downloaded; download two at once; ignore one; preview one without downloading; re-sync preserves states. 6 — delete a file on disk manually and confirm the checker reports/repairs it; build the dist folder and run it from a clean directory or second machine.

# Process Notes

- Mason builds the app, runs all tests, and handles all git commits/pushes; Claude writes code + tests, checks off plan checkboxes as items complete, and avoids git and external prompt-requiring tools (internal file tools only).
- Questions go into the Open Questions section of this document, not interactive prompts.
- Every change that can be tested gets tests (GoogleTest), following knottyyoga conventions: no test fixtures, self-contained tests beside sources, no assumed collection order. Qt-heavy UI glue is verified via the per-phase smoke lists instead.