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

**Decided stack:** C++20 · Qt 6.8.3 LTS prebuilt MSVC2022 binaries (Widgets + Multimedia + MultimediaWidgets + Sql + Network + WebEngineWidgets), fetched unattended by script · **Conan 2 for all library dependencies** (knottyyoga `conanfile.py` pattern — gtest now, room to grow) · SQLite via Qt Sql (FTS5 keyword search with LIKE fallback) · CMake · external tools: yt-dlp + ffmpeg/ffprobe (winget) + Python 3.13 venv with Instaloader. No Postgres, no Angular, no web server.
- Mason- I very much want Conan. I also want a script that does winget, etc to pull down the dependencies onto a clean machine. The only reason I have these libraries on my machine is because of Conan. Please look at knotty yoga server and/or server components to do the conan / CMake integration.
	- → Claude: done — the build adopts the knottyyoga/server_components Conan integration wholesale (`conanfile.py` with the `Library` list + generated `ConanLibImports.cmake`, `conan_provider.cmake` invoked via `CMAKE_PROJECT_TOP_LEVEL_INCLUDES` in CMakeSettings.json), and `tools/setup_machine.ps1` bootstraps a clean machine (winget + pip + venv + Qt). Qt itself stays *outside* Conan for the validated reasons below.

### Toolchain validation: can Qt come from Conan? (checked 2026-08-06)

**Yes, technically — no, not for this app.** ConanCenter carries `qt/6.8.3`, `qt/6.10.3`, and `qt/6.11.1`; the 6.x recipe builds on MSVC and does package moc/rcc/uic and even windeployqt, pulling ninja/winflexbison as build tools — so AUTOMOC and friends would work. Three findings rule it out here:

1. **The Conan build of QtMultimedia has no FFmpeg backend.** The recipe declares no ffmpeg dependency anywhere (only ALSA/OpenAL audio bits), and Qt Multimedia's maintained playback backend *is* FFmpeg — official Qt binaries bundle FFmpeg *libraries* at Qt build time (unrelated to our ffmpeg.exe CLI tool). A Conan-built Qt leaves the player on the legacy Windows-native (WMF) backend at best, and users report self-built Qt with *no* working video backend at all. The player is the heart of this app; this alone is disqualifying.
2. **QtWebEngine does not build on Windows via the recipe** — open conan-center issue since Nov 2024 (Chromium's nested paths blow past Windows/Ninja path limits; no workaround found).
3. **No prebuilt Qt binaries exist on ConanCenter** for our configuration — the first `conan install` would compile Qt from source for multiple hours and ~10+ GB of build space, and any option change recompiles.

So the split is: **Conan owns every library dependency** (exactly the knottyyoga pattern), and **Qt 6.8.3 official prebuilt binaries are fetched by `tools/setup_machine.ps1`** using `aqtinstall` — a pip tool that downloads the same MSVC2022 binaries as the online installer, unattended, no Qt account — wired into CMake via `CMAKE_PREFIX_PATH`. One script still bootstraps a clean machine end-to-end. If full-Conan-Qt is ever insisted on, the price is explicit: hours-long builds, no WebEngine, and WMF-or-nothing playback.

**Portable-library contract:** everything the catalog needs lives under the library root — `library.db` at the root, media under `{Category}/{YYYY}/{MM}/{DD}/`, thumbnails/staging/backups under `.videolibrary/` — and every path stored in the DB is root-relative with forward slashes, so the folder can be copied/moved wholesale and opened on any machine. Per-machine state (tool paths, window geometry, recent libraries, Instagram session/cookies) lives in QSettings/AppData, never inside the library.

## Instagram tooling suggestions

**Downloading (keep yt-dlp).** Instagram now sits almost entirely behind a login wall — anonymous yt-dlp requests fail, so downloads need cookies. The cookie source is a **configurable setting with a login-status probe** (Phase 5.1), tried in this order: (1) `--cookies-from-browser chrome` passthrough — Mason's preference; (2) embedded QtWebEngine Instagram login that exports a Netscape `cookies.txt`; (3) a user-supplied `cookies.txt` (browser-extension export). The probe determines what actually works on each machine and the UI says so. The app shells out to yt-dlp with politeness delays and surfaces yt-dlp staleness (Instagram breaks old extractors regularly; yt-dlp stays winget-managed).

Mason- Please note that I do yt-dlp --cookies-from-browser on my machine for things that need login / aren't public. That always works and seems like the best option here.

→ Claude: it's the first thing the app tries — but a heads-up on current reality: Chrome 127+ (July 2024) moved its cookies to "app-bound encryption," and yt-dlp cannot decrypt those v20 cookies on Windows; the yt-dlp issue asking for a fix was still open as of March 2026. It genuinely does work for Firefox (and, so far, Edge), and it worked for Chrome before the rollout — which may explain the "always works" experience, or your Instagram session cookie predates the migration. Rather than argue from search results, the design makes it empirical: the login-status probe runs your preferred mode first and only falls back (embedded login → cookies.txt file) if Instagram reports "not logged in."

**Enumerating your saved list** (no official API for personal saved posts — all options are unofficial):
1. **Instaloader** (recommended) — mature Python library; the helper script builds its session from the same cookies the embedded login harvested (`load_session`), then `get_saved_posts()` yields shortcode/URL/owner/caption/date/thumbnail without downloading. We wrap it in a small helper script emitting JSON that the app invokes and reconciles.
2. **instagrapi** — private mobile-API library; more capable (per-collection access) but a heavier risk profile.
3. **gallery-dl** — multi-platform downloader that can target the saved page with cookies; less Instagram-specialized than Instaloader.
4. **Browser extension** that scrapes your saved page in your real session and posts to the local server — lowest ban risk, most build effort. (Fallback idea, not planned.)
5. **Manual paste of URLs** — ships first regardless, as the always-works path (and covers YouTube for free).

**Risk note:** any automated enumeration violates Instagram ToS and can flag the account it runs as. Mitigations planned: manual sync button (not a background poller) with a minimum interval, low volume, politeness delays, cached results, and the option to run against a secondary account (Open Questions #8).

Mason- I'm fine with using Instaloader. Please just make sure to make a config script for people to run to pull down the needed dependencies to their machine.

→ Claude: `tools/setup_machine.ps1` (Phase 0.1) is exactly that — winget for yt-dlp/ffmpeg/CMake, pip for aqtinstall + Qt, a per-machine Python venv with Instaloader, plus a `-Mode runtime` variant for the assistant's machine that skips the dev-only pieces.

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
	- Mason- Note taht we can use yt-dlp --cookies-from-browser and everything should just work.
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
	- Mason- I need to organize the existing library to match this schema. Could we have an option to manually import a local file like an Instagram download?
	- → Claude: yes — added Phase 2.7: Import files… (menu + drag-and-drop onto the window) runs local video files through the same import pipeline with a category/metadata dialog. Organize the existing tree whenever you like; Phase 3.6's adoption scan ingests it once it matches the schema.
14. Qt install: the official online installer (needs a free Qt account) or an unattended `aqtinstall` script I provide (no account needed)? And which Visual Studio do you have — 2019 or 2022? Qt 6.8's prebuilt Windows binaries are MSVC-2022-built, so VS 2022 is the smooth path.
	- Mason- I'm currently using VS2022. Do we need to install Qt? Qt is listed in conan center so I think it can just be pulled down via that. I think it should pull down MOC and all the stuff needed? It would be really nice to have this done via Conan instead of needed an installer.
	- → Claude: validated — Qt *is* on ConanCenter (6.8.3/6.10.3/6.11.1, and yes, moc/rcc/uic come with it) but it's source-only (hours-long first build), the Conan build has **no FFmpeg multimedia backend** (kills the player), and **WebEngine won't build on Windows** (open issue since Nov 2024). Full findings in "Toolchain validation" up in the Decision section. Resolution: Conan owns every library dependency in the knottyyoga pattern; Qt 6.8.3 prebuilt MSVC2022 binaries come from `tools/setup_machine.ps1` via aqtinstall — no Qt account, no installer clicking.
15. QtWebEngine is the one heavyweight module (roughly a GB installed; ~150 MB added to the deployed app folder). It powers the embedded Instagram login — our only reliable cookie source given Chrome — and in-app post preview. Include it (**my recommendation**), or skip it and manually export `cookies.txt` from a Chrome extension whenever the session expires?
	- Mason- I'm fine including it but if we use yt-dlp with the cookies from chome option, I'm thinking we only need this to be able to play instagram videos in the app.
	- → Claude: one correction to the premise: `--cookies-from-browser chrome` does not currently work on Windows (Chrome v20 app-bound encryption; open yt-dlp issue as of Mar 2026) — the app probes it first and falls back automatically. WebEngine stays in (prebuilt via the setup script; Conan couldn't build it anyway) as the fallback cookie source (embedded login) and the open-post-in-app view. Playing IG videos in-app doesn't need WebEngine — that's yt-dlp `-g` → the normal player.
16. How will you and your assistant share the library folder — copied/external drive, NAS, or a synced folder (OneDrive/Dropbox)? SQLite wants one writer at a time: totally fine as long as only one machine has the app open at once. I'll add a stale-lock warning either way, but the real setup shapes the run-book guidance.
	- Mason- Shared folder on a network drive and than copying the directory to my laptop to take with me to my studio.
	- → Claude: works with one rule — only one machine has the library open at a time, and close the app before copying. Phase 6.3 adds the stale-lock warning, on-open `quick_check`, and rotating backups; `journal_mode=DELETE` was chosen for exactly this copy-safety.

# Implementation Plan

Layer order within every phase, lowest first: **data** (schema, repositories) → **low-level services** (process runner, tool registry, settings) → **business_logic** (acquisition, library operations, instagram, player/notes logic) → **ui** (widgets, models, presenters). Dependencies point downward only — knottyyoga's layering discipline, adapted to a desktop app. Checkboxes get checked off as items are implemented.

Design principles baked into every phase:
- **Event-driven, single process**: yt-dlp/ffmpeg/python run as async `QProcess`es on the event loop — background downloads with no worker threads. All SQLite access stays on the GUI thread (Qt SQL connections are thread-bound), which also avoids write contention. Long filesystem scans run on a worker and marshal results back.
- **Portable library**: every path stored in the DB is library-root-relative with forward slashes; `library.db`, thumbnails, staging, and backups all live under the root (`.videolibrary/` for app-managed folders). Per-machine state (tool paths, recent libraries, window geometry, Instagram session/cookies) lives in QSettings/AppData only.
- **Tests**: GoogleTest via Conan (knottyyoga `conanfile.py` pattern), knottyyoga conventions — no fixtures, self-contained tests beside sources, no assumed collection order; the test main spins up a `QCoreApplication`. UI widgets stay thin; logic lives in models, presenters, and pure functions where tests can reach it.

## Phase 0 — Toolchain and app skeleton

### 0.1 Toolchain setup (Mason runs; Claude provides scripts and docs)
- [x] `tools/setup_machine.ps1` — one idempotent clean-machine bootstrap (per the Decision-section toolchain note): winget installs (yt-dlp, ffmpeg, CMake, Git, GitHub CLI, Python), `pip install aqtinstall` + fetch the newest Qt 6.8.x `win64_msvc2022_64` with qtmultimedia/qtwebengine (+qtwebchannel/qtpositioning/qtimageformats) into `C:\Qt`, per-machine Python venv with Instaloader, Conan + `conan profile detect` if absent, sets `Qt6_ROOT` and Qt's `bin` on PATH; prints a version summary at the end
- [x] `-Mode runtime` variant for the assistant's machine: winget tools + Python venv only (the dist folder already bundles Qt)
- [x] README prerequisites section: VS2022, then "run the script"; the script's summary table is what verifies yt-dlp/ffmpeg resolve

Implementation notes: the venv lives at `%LOCALAPPDATA%\Honuware\VideoLibrary\python`, which is exactly what `AppPaths::pythonVenvDirectory()` returns — the script and `foundation/app_info.h` must stay in agreement. Qt is discovered through the `Qt6_ROOT` env var rather than a hard-coded `CMAKE_PREFIX_PATH`, so the version can move without editing `CMakeSettings.json`. PATH is only rewritten when the raw user PATH has no `%VARIABLE%` references (rewriting one would freeze them); otherwise the script prints the entry to add by hand.

### 0.2 Repository and build skeleton
- [x] Repo at `C:\Users\mason\source\repos\video_library`: top-level CMakeLists.txt (C++20, `qt_standard_project_setup`, AUTOMOC) with targets `video_library_core` (static lib — everything testable), `video_library_app` (WIN32 exe — `main.cpp` only, output `VideoLibrary.exe`), `video_library_tests` (GoogleTest exe)
- [x] Conan integration mirrored from knottyyoga/server_components: `conanfile.py` with the `Library` list (gtest pinned to knottyyoga's 1.12.1, so it is already in the local cache) generating `ConanLibImports.cmake`, vendored `conan_provider.cmake`, a benign `find_package(GTest)` to force the provider, `${GTEST_LIB}`-style variables
- [x] GoogleTest via Conan; custom test `main.cpp` that creates a `QCoreApplication` and enables `QStandardPaths` test mode; CTest wiring
- [x] CMakeSettings.json for VS (x64-Debug + x64-Release) with `CMAKE_PROJECT_TOP_LEVEL_INCLUDES=conan_provider.cmake`; `.gitignore` / `.gitattributes`; README with build steps, the Qt-not-from-Conan rationale, and troubleshooting
- [x] CLAUDE.md seeded with this app's conventions (layer DAG, portability rule, naming, testing rules, CMake header listing, Qt gotchas, planning-directory rule)
- [x] Smoke: `test/smoke_test.cpp` proves Conan+Qt+GoogleTest are linked and the runner's isolation is on

Deviations from the plan text, and why: `CONAN_CMD` is not in CMakeSettings.json — the vendored provider reads `CONAN_COMMAND`, and `conan` is already on PATH, so nothing is needed (the override is documented in the README instead). `CMAKE_PREFIX_PATH` is likewise absent in favour of `Qt6_ROOT`. Qt Multimedia/MultimediaWidgets/WebEngineWidgets are found as *optional* components now: a missing module prints a warning naming the phase that needs it instead of blocking Phase 0.

- [x] Qt runtime deployment into the build tree (added after first run failed with "Qt6Guid.dll was not found"): a `windeployqt` POST_BUILD step on the app target, plus a `$<TARGET_RUNTIME_DLLS>` copy on the test runner. The first attempt put Qt's `bin` on PATH from the setup script, which is the traditional fix but a bad one — rewriting PATH through the .NET API expands and freezes any `%VARIABLE%` references it contains (Mason's PATH has two), and the safety check that skipped the rewrite left the app unable to start. PATH manipulation is now gone from the setup script entirely; the build directory is self-contained instead. Copying only the linked DLLs would not have been enough either: Qt resolves its plugin directory relative to the loaded Qt6Core DLL, so a local copy of Qt6Core needs `platforms/qwindows[d].dll` beside it — and later phases add the SQLite driver, image formats, multimedia, and WebEngine resources to that list, which is exactly the bookkeeping windeployqt exists to do. Phase 6.4 packaging now reuses this same step.

### 0.3 App bootstrap and logging
- [x] `main.cpp` + `MainWindow` shell: left navigation (Library, Instagram, Downloads, Search, Settings) over a stacked central area, File/Help menus, status bar showing the open library
- [x] Logging: `qInstallMessageHandler` → size-rotating file in AppData (5 MB, 5 generations) + stderr + `OutputDebugStringW` for the VS Output window; startup banner with app/Qt versions and the settings/log paths (tool versions join it in 2.2)
- [x] `MachineSettings` (QSettings wrapper): recent libraries, window geometry/state, tool-path overrides, reopen-on-launch; `QSettings` is injected so tests use a throwaway .ini
- [x] Tests: `logging_test.cpp` (format for every severity, category and source-location rules, rotation generations, reopen-appends, no-op when unopened), `machine_settings_test.cpp` (MRU order, case-insensitive dedupe, cap, binary round-trip, tool overrides), `app_paths_test.cpp`

### 0.4 Library open/create
- [x] `LibraryContext`: holds the root; creates `.videolibrary/{thumbnails,thumbnails/sources,staging,backups}`; resolve/relativize helpers that enforce root-relative forward-slash paths and reject escapes; `prepare()` is idempotent so opening also repairs a missing subdirectory
- [x] Open/Create Library folder flow (folder picker with confirmation prompts), recent-libraries menu with Clear, reopen-last-on-launch, "library missing/moved" handling
- [x] Tests: `library_context_test.cpp` (normalization, layout, safe/unsafe relative paths, sibling-prefix containment `C:/Videos/Acro` vs `C:/Videos/AcroOld`, root-is-not-contained, prepare idempotence and repair, isLibrary before/after)

`library.db` itself is deliberately NOT created here — `DatabaseManager` owns that in Phase 1.1. A folder counts as a library once `.videolibrary/` exists, so "Open Library…" can tell an existing library from an ordinary folder and offer to create one. A last library that is unreachable stays in the recent list rather than being forgotten, because the usual cause is a network share that is not connected yet.

### 0.5 GitHub repository under honuware (Mason runs)

Timing: run this as soon as 0.2 produces a first successful build. Claude's part is making sure the repo is push-ready; the commands are Mason's to run (per process notes, Claude never runs git).

- [x] (Claude, folded into 0.2) `.gitignore` covers `out/`, `build/`, `.vs/`, `dist/`, generated files (`ConanLibImports.cmake` is generated by the conanfile and must not be committed), and `CMakeUserPresets.json`
- [x] (Mason) create `honuware/video_library` on GitHub and push, using the commands below ✅ 2026-08-06

Notes before running: the GitHub CLI is **not currently installed** on this machine — the block installs it via winget (one time). The default branch is `master` to match knottyyoga and server_components. The repo is created **private**; flip it later with `gh repo edit honuware/video_library --visibility public --accept-visibility-change-consequences` or in the repo's web Settings. `gh auth login` must use the account that owns the honuware org.

```powershell
# --- One-time: install the GitHub CLI and log in ---
winget install --id GitHub.cli -e
# Close and reopen this PowerShell window so gh is on PATH, then:
gh auth login
#   -> GitHub.com -> HTTPS -> Login with a web browser
#   (use the account that owns the honuware org)

# --- Initialize the local repo (from the project root) ---
cd C:\Users\mason\source\repos\video_library
git init -b master
git add .
git status
#   sanity check: no out/, build/, .vs/, or ConanLibImports.cmake in the list —
#   if any appear, stop and fix .gitignore before committing
git commit -m "Initial scaffold: CMake + Conan + Qt app skeleton"

# --- Create the private repo under the honuware org and push ---
gh repo create honuware/video_library --private --source . --remote origin --push

# --- Verify (opens the repo in the browser) ---
gh repo view honuware/video_library --web
```

If you'd rather not install gh: create an **empty** repo named `video_library` at github.com under the honuware org (no README/license/.gitignore, so the push isn't rejected), then from the project root:

```powershell
cd C:\Users\mason\source\repos\video_library
git init -b master
git add .
git commit -m "Initial scaffold: CMake + Conan + Qt app skeleton"
git remote add origin https://github.com/honuware/video_library.git
git push -u origin master
```

After this, day-to-day flow is the usual `git add` / `git commit` / `git push` from any machine with access to the honuware org.

## Phase 1 — Data layer (SQLite)

### 1.1 Database manager and migrations
- [x] `DatabaseManager`: opens `library.db` (QSQLITE) on a per-instance connection name, pragmas (`foreign_keys=ON`, `busy_timeout=5000`, `journal_mode=DELETE` for copy-safety), creates missing parent folders, `schema_migrations` table + version-sorted migration runner (each migration in its own transaction)
- [x] FTS5 availability probe at open (`sqlite_compileoption_used('ENABLE_FTS5')`) setting a capability flag; LIKE-based fallback path when absent
- [x] Tests: fresh create, reopen idempotence (no re-seeding), migration recording, version uniqueness, journal mode, foreign-key enforcement, probe/index agreement, two catalogues open at once, failure paths

Design note: the FTS tables are created *outside* the migration list (`ensureSearchIndex`), because a migration has to apply on any SQLite build and FTS5 is a compile-time option. They index the base tables and hold no original data, so they can be created — and rebuilt — at any point. `probeFullTextSearch` also warns loudly in the one bad case (index present, FTS5 missing), where writes would otherwise fail with an error that points nowhere.

### 1.2 Schema v1
- [x] `categories` (id, name UNIQUE COLLATE NOCASE, directory_name UNIQUE COLLATE NOCASE, created_at) — seeded with Inbox, Rope, Partner Acro, Handstands, Fitness Ideas
- [x] `videos` (id, category_id FK RESTRICT, relative_path UNIQUE COLLATE NOCASE, file_name, title, creator, platform, source_url, source_id UNIQUE NULL, description, duration_ms, width, height, file_size_bytes, downloaded_at, published_at NULL, thumbnail_relative_path, created_at, updated_at)
- [x] `tags` (id, name UNIQUE COLLATE NOCASE) and `video_tags` (video_id FK, tag_id FK, PK(video_id, tag_id), CASCADE)
- [x] `notes` (id, video_id FK CASCADE, timestamp_ms, body, created_at, updated_at)
- [x] `source_items` (…, state CHECK IN (new|queued|downloaded|ignored|gone), video_id FK SET NULL) — the cached Instagram saved list
- [x] `downloads` (…, state CHECK IN (queued|running|success|error|canceled), progress_percent, error_message, staging_name, **video_id FK SET NULL**, created_at, started_at, finished_at)
- [x] `library_settings` (key, value) — per-library preferences (e.g., last playback positions)
- [x] FTS5: `videos_fts` (title, description, creator) and `notes_fts` (body) as external-content tables with insert/update/delete triggers; indexes on FKs, downloaded_at, and state columns
- [x] Tests (`schema_test.cpp`): seeding, case-insensitive uniqueness on category names and relative paths, source_id unique-but-optional, FK enforcement, category delete RESTRICT, video delete cascading to notes/tags while leaving the tag and the source_item, CHECK rejection of unknown states, FTS trigger sync across insert/update/delete

Additions beyond the plan text, and why: `downloads.video_id` (Phase 2.6 needs "jump to the video" once a download succeeds); `CHECK` constraints on both state columns (a typo'd state would otherwise sit in the database silently); `relative_path UNIQUE COLLATE NOCASE` (Windows treats `Rope/clip.mp4` and `rope/CLIP.mp4` as one file, and two rows for one file would corrupt every later rename).

### 1.3 Repositories
- [x] One repository per table, thin CRUD only (knottyyoga table_helpers spirit): `CategoryRepository` (find-or-create, safe folder-name derivation with collision suffix), `VideoRepository` (+ `setLocation` for rename/recategorize), `TagRepository` (find-or-create, prefix search for autocomplete, transactional `setTagsForVideo`), `NoteRepository` (ordered by timestamp then id), `SourceItemRepository` (state transitions, `linkToVideo`), `DownloadRepository` (queue queries, startup requeue), `LibrarySettingsRepository` (upsert, typed helpers)
- [x] Shared `RepositoryBase` + `SqlUtil` (prepare/exec/executeAll/tableExists + RAII `Transaction`) so every repository reports failures identically
- [x] Tests per repository on temp-file databases: uniqueness conflicts, cascades, state transitions, ordering, prefix search, LIKE-wildcard escaping, path-escape rejection, clamping

Design note: repositories store the **connection name**, not a `QSqlDatabase`. A `QSqlDatabase` is a reference-counted handle, and Qt warns ("connection is still in use") if a copy outlives `removeDatabase()`; holding only the name means repositories and the `DatabaseManager` can be destroyed in any order.

### 1.4 Search queries
- [x] `VideoQuery`: filters (category, year/month/day derived from downloaded_at, tags all-of, keyword via FTS5-or-LIKE across title/description/creator/note bodies), five sort orders, limit/offset paging with a whole-set `count()`
- [x] `LibraryTreeQuery`: category → year → month → day with counts rolled up, empty categories included, newest years first
- [x] Tests: year boundary half an hour apart, month/day narrowing, tag intersection vs. union, keyword found via title / description / creator / note body, punctuation that is FTS5 syntax, prefix completion, combined filters, sorting, paging — the keyword tests run against **both** backends and assert the same result set

### 1.5 Path planning and file operations
- [x] `PathPlanner`: `{directory_name}/{YYYY}/{MM}/{DD}/{file_name}`; collision resolution via " (2)" suffix before the extension; keeps the original extension when a new name brings none
- [x] `PathUtil::sanitizeFileName` (in foundation, so the data layer can use it too): illegal characters, control characters, reserved device names (CON/NUL/COM1…), trailing dots and spaces, 120-character cap preserving the extension
- [x] `FileOperations`: rename, move-with-directory-creation, import from outside (copy or move), delete → `QFile::moveToTrash` (Q6), empty-parent pruning that stops at the category folder, plus the `applyFileChangeThenDatabaseChange` sequencing helper with revert on catalogue failure
- [x] Tests with QTemporaryDir: happy paths, collisions, sanitization table, refusal to overwrite or escape the library, pruning vs. non-empty folders, real Recycle Bin behaviour, and all four sequencing outcomes

### 1.6 Wired into the shell (added so Phase 1 is verifiable by hand)
- [x] `MainWindow` opens the catalogue when a library opens and closes it when the library closes; a library whose database will not open is refused rather than half-opened
- [x] The Library page shows video count, schema version, and which search backend is active — enough to confirm Phase 1 without a SQLite browser

## Phase 2 — Acquisition pipeline (milestone: replaces the manual yt-dlp workflow)

### 2.1 Process runner
- [x] `LineBuffer` (foundation): turns arbitrary pipe chunks into whole lines; handles `\n`, `\r\n`, and the bare `\r` ffmpeg uses, including a `\r` that lands on a chunk boundary
- [x] `IProcessRunner` + `QProcessRunner`/`QProcessHandle` (deferred start, line-buffered stdout/stderr signals, exit code, kill/cancel, timeout, kills the child on handle destruction) and `TestProcessRunner` scripted fake with auto and manual modes
- [x] `waitForCondition`/`pumpEventLoop` (foundation, test-only): async tests fail with a readable assertion instead of hanging
- [x] Tests: 11 for line splitting across chunk boundaries; 10 against **real** `cmd.exe` processes (output, working directory, exit codes, missing executable, cancel, cancel-before-start, timeout, no-orphan-on-destroy); 10 for the fake

Design note: both runners start the process from the event loop rather than from `start()`. A caller connects *after* `start()` returns, so a synchronous start would let it miss the first output — and worse, the fake would pass where the real thing fails. Making the fake asynchronous in the same way is what keeps the acquisition tests honest.

### 2.2 Tool registry
- [x] Resolves yt-dlp/ffmpeg/ffprobe/python from a Settings override → the setup script's venv (Python) → the system PATH; a configured path that no longer exists is ignored rather than permanently hiding a working tool
- [x] Asynchronous, time-boxed version probes; per-tool `statusChanged` plus `refreshFinished`; status carries version, path, and a problem message that names the fix
- [x] Tests with the fake runner: version parsing per tool (yt-dlp bare, ffmpeg banner, `Python 3.13.1`, blank/odd output), override wins over PATH, missing tool advises `setup_machine.ps1`, stale override ignored, present-but-unrunnable reported distinctly, all four probed, probes carry a timeout

### 2.3 yt-dlp client
- [x] `SourceUrl`: Instagram reel/reels/p/tv (including `/creator/reel/<code>`) and YouTube watch/youtu.be/shorts all collapse to one reference; `igsh`, `img_index`, `list`, `t` dropped; unknown hosts still download, just without a dedupe id; `extractUrls` pulls links out of a pasted block
- [x] `YtDlpOutput` (pure): progress percent, the final-path marker line, `--dump-single-json` → `VideoMetadata`, `YYYYMMDD` → storage timestamp, and an error taxonomy (login-required / unavailable / network / unsupported / unknown) with an `isWorthRetrying` verdict
- [x] `YtDlpClient` + `IDownloadClient`: async download and metadata tasks over the process runner, cookie settings (`--cookies` file wins over `--cookies-from-browser`), staging via `-P`, cancel, per-outcome results
- [x] Tests: 10 for URL handling, 14 for output parsing, 11 for the client — all against scripted output, no network

Discovery worth recording: `--print` **implies `--quiet` and `--simulate`**. Left alone, the download command would report a file path for a download that never happened, and show no progress at all. It therefore passes `--no-simulate` and `--progress` explicitly, and a test pins both so they do not get tidied away later. The final path comes from `--print after_move:` rather than from parsing `[download] Destination:`, because that fires after any merge or remux and is where the file actually ended up.

### 2.4 Download manager
- [x] Queue persisted in `downloads`; concurrency capped (default 2); state machine queued→running→success|error|canceled; retry; startup requeue of interrupted items; progress/state signals for the UI; URLs canonicalized on the way in
- [x] Tests with a scripted fake client: nothing runs before `start()`, restart recovery, concurrency cap, next-starts-on-finish, progress persisted and announced, success/failure recording, cancel while running and while queued, retry clears the old error, staging directory and politeness delay reach the client, queue drains

### 2.5 Import service
- [x] `MediaProbe`: ffprobe JSON → duration/dimensions/size (first *video* stream, container duration with a stream fallback), ffmpeg single-frame thumbnail seeking a quarter of the way in
- [x] `ImportService`: probe → duplicate check by `source_id` → PathPlanner target under `{Category}/{Y}/{M}/{D}` → move (download) or copy (manual import) → `videos` row → link `source_items` and the `downloads` row → thumbnail named after the row id. Imports run one at a time so two cannot both believe the same file name is free.
- [x] Tests: 6 for MediaProbe (argument construction and JSON parsing), 9 for ImportService end to end against a real temp library with scripted tools — file filed and catalogued, manual import copies and leaves the original, duplicate surfaced not re-imported, same-name files both survive, unreadable file still imported, missing source fails cleanly, download row and saved post tied to the video, thumbnail recorded when ffmpeg produces one, imports serialised

Design note: a file ffprobe cannot read is still imported, with no duration or dimensions and a warning in the log — losing a downloaded video because its metadata was unreadable would be worse than an incomplete catalogue row. A missing thumbnail is likewise cosmetic and never fails an import. The file moves before the catalogue row is written, through `applyFileChangeThenDatabaseChange`, so a rejected row puts the file back instead of leaving it invisible inside the library.

### 2.6 UI: Add-by-URL and Downloads view
- [x] Add by Link (File menu, Ctrl+D): multi-line paste box that extracts the links out of whatever is pasted and shows the count before committing; queues them and switches to Downloads
- [x] Downloads view: state / progress / link / details table, cancel, try again, show video, clear finished, and a status-bar count of running downloads
- [x] Tests: 8 for the `DownloadRowView` state mapping (waiting shows no progress bar, running rounds and clamps its percentage, done offers the video only once it is filed, failed shows the reason and offers a retry, cancelled can be restarted, unknown state still shows something)

The table is rebuilt from the database on every change rather than kept in step by hand, so what it shows survives a restart and cannot drift. Progress is the one exception — it arrives many times a second, so only that cell is touched, and selection is restored by download id rather than row number so a moving queue does not move the user's selection.

### 2.7 Manual local-file import (per #13)
- [x] Import Files (File menu, Ctrl+I) and drag-and-drop onto the window: pick video files → dialog for category, optional creator, copy-vs-move → the same ImportService path (ffprobe, thumbnail, catalogue row)
- [x] Covered by the ImportService tests above: copy from outside the library leaves the original, move from staging does not, no title falls back to the file's own name, colliding names both survive
- [x] Copy is the default, and a drop of mixed files imports the videos and ignores the rest

### 2.8 Cookie configuration (pulled forward from 5.1)

Found by testing: a login-walled Instagram post failed with "would not serve this without being signed in", because `CookieSettings` was never populated — no cookie flags reached yt-dlp at all. Waiting for Phase 5.1 would have left Instagram downloads broken in the meantime, which is most of the point of the app.

- [x] `MachineSettings` stores the cookie source per machine: browser name for `--cookies-from-browser`, or a `cookies.txt` path for `--cookies`. **Cookies are off until they are asked for** — see 2.9 for why the original Chrome default was wrong. `suggestedCookieBrowser()` is what the Settings picker starts on, not something that gets sent.
- [x] Real Settings page replacing the placeholder: which external tools were found with their versions and locations (the status model 2.2 promised), a Check Again button, and the cookie source — browser picker, cookies.txt picker, or off. Changes apply as they are made and are pushed straight into the download client.
- [x] Startup logs the cookie source, and warns when there is none, so the log says why Instagram downloads are failing.
- [x] Tests: default, deliberate-off, browser name normalised to lower case, cookie path stored with forward slashes

**Q7 answered, by testing on the machine itself (2026-08-07).** `--cookies-from-browser chrome` fails here with `Could not copy Chrome cookie database` ([yt-dlp #7271](https://github.com/yt-dlp/yt-dlp/issues/7271)) — not the app-bound encryption I had assumed from release notes, but a plain file lock: Chrome was running with 56 processes and holds the database open. `edge` gives the same Chromium error; `firefox` is not installed. So browser cookie extraction works only with the browser fully closed, and **the cookies.txt route is the practical default here**.

Two bugs this exposed, both fixed:
- `classifyError` did not recognise the message ("cookie database" is not "cookies"), so it fell through to Unknown. There is now a `CookieAccess` kind, checked before the login-wall patterns, whose message names the actual fix — close the browser, or export a cookies.txt. It is retryable, because closing the browser and trying again *is* the fix.
- An unrecognised failure summarised as "The download failed." and put yt-dlp's own words on the third line, so the status-bar toast (first line only) said nothing at all. `summarizeError` now returns the tool's own words for anything unrecognised, and `describeError` no longer repeats them underneath.

Improvement made while wiring this up: the download command now also asks yt-dlp to print the title, creator, id, platform, and upload date (a second `--print after_move:` with a marker and separator). Without it an imported video arrived titled with its platform id — "ABC123" — which is exactly the renaming chore the app exists to remove. It costs no extra request, since it comes from the run that already happened.

### 2.9 Cookies are opt-in, and never fatal (correction to 2.8)

Found by using it: with Chrome as the default cookie source, downloads started failing for **public** posts that never needed a login at all. Defaulting cookies on was the wrong call. `--cookies-from-browser chrome` fails outright while Chrome is running, and yt-dlp treats that as fatal — it gives up on the cookie database *before* it asks the site anything, so the post's own permissions never come into it. Turning cookies on to help with login-walled posts made every other post fail.

- [x] `cookieBrowser()` returns empty until someone chooses one. `defaultCookieBrowser()` is renamed `suggestedCookieBrowser()` so the name stops claiming to be something that gets applied — it is only where the Settings picker lands when nothing has been chosen.
- [x] **A cookie failure now falls back instead of failing.** When cookies were sent and the attempt dies with `CookieAccess`, `YtDlpDownloadTask` starts the same download once more with the cookie flags stripped. A public post downloads; a genuinely walled one comes back as a login wall, which is the honest answer. Once only — a second cookie failure is a real one.
- [x] Tests (3 on the client, 2 on the settings): the fallback runs and succeeds with the second command line free of cookie flags but still carrying the same URL, the fallback is attempted only once, no second attempt is made when no cookies were sent, cookies are off by default, and the picker still starts on a browser.

The lifetime detail worth recording: the retry replaces the process handle from inside that handle's own `finished` signal, so the spent handle is disconnected and handed to `deleteLater()` rather than destroyed underneath the call that is still running — the same rule `DownloadManager::releaseTask` follows.

This leaves Q7's conclusion intact (browser cookie extraction only works with the browser fully closed, so cookies.txt is the practical route here) but changes what the app does about it: cookies are a thing you turn on when a post needs one, not a thing that can take down downloads that don't.

### 2.10 Downloads were arriving silent

Found by noticing no video had any sound. The download command never passed `--ffmpeg-location`. yt-dlp's default format choice fetches video and audio as **separate streams and merges them with ffmpeg**, so a yt-dlp that cannot find one falls back to whatever single stream it can get whole — video, no audio. Our ffmpeg is often somewhere only the Settings page knows about, so leaving it to PATH was never safe.

It presents as a broken player rather than a broken download, which is why it survived all of Phase 4 unnoticed: the files play perfectly, they simply have no audio track in them.

- [x] `buildDownloadArguments` takes the ffmpeg path and passes `--ffmpeg-location`; `startDownload` fills it from the `ToolRegistry`. An empty path passes nothing at all rather than an empty flag — yt-dlp would take an empty location at its word and stop looking on PATH.
- [x] Startup warns when ffmpeg is missing, saying what actually happens: the download still succeeds and the file still plays, it just has no sound.
- [x] Tests (3): the flag and its value are present, nothing is emitted when there is no ffmpeg to point at, and a *started* download really carries the resolved path — building the arguments correctly is no use if the client never passes them in. The bench now resolves a fake ffmpeg as well as a fake yt-dlp, since a bench that cannot resolve one would let that go untested.
- **Already-downloaded files are not repaired by this.** They have no audio track and have to be fetched again.

Hardening on the player side while chasing this, neither of them the cause: the audio output's volume and muted state are now set explicitly at construction rather than inherited from whatever `QAudioOutput` starts at, and the volume slider is connected *before* its initial value is set, so that value reaches the backend instead of the slider merely claiming to be at full volume.

**The merge fix was necessary but not sufficient** — files downloaded after it do contain AAC stereo audio (confirmed with ffprobe against the library) and were still silent. Ruled out by inspection rather than guesswork: the audio stream is normal in every respect including disposition and duration, every Qt multimedia DLL and the FFmpeg media plugin are deployed beside the executable, and the app logged no playback error at all.

- [x] `handleMediaStatusChanged` selects the first audio track when the file has tracks but the backend chose none. A file can load with audio present and nothing selected — the video plays, no error is raised, and nothing reaches the speakers, which is indistinguishable from a file with no sound in it. yt-dlp's merge output here is VP9 muxed into MP4, unusual enough for track auto-selection to come up empty.
- [x] The same handler logs the track count, the active track, the output device, and the volume, once per source. When sound is missing that one line replaces three separate guesses.

**Still silent, and the diagnostic is what finally located it.** The log line read `Audio: 1 track(s), active 0, output "Speakers (Realtek(R) Audio)", volume 1` — a track selected, a real device, full volume, not muted. Everything on the decode side was working; the sound was going somewhere nobody was listening. The machine has five *active* render endpoints (Speakers, Headphones, Realtek Digital Output, and two monitors that accept audio over DisplayPort), and a default-constructed `QAudioOutput` takes the default device once at construction and keeps it forever.

- [x] `followDefaultAudioDevice`, called at construction and again on `QMediaDevices::audioOutputsChanged`, moves playback to whatever Windows currently calls the default output, and logs the move. Plugging in headphones now takes the sound with it instead of leaving the app playing to the device that happened to be default at startup.
- `QMediaDevices` is declared before `QAudioOutput` so it outlives the output watching it — the same reverse-declaration-order rule that put `QAudioOutput` before `QMediaPlayer`.

**Still silent with the default confirmed as Realtek**, so following the default was not enough either. Enumerating the machine's *active* render endpoints is what finally explained it:

| Shown as | Actually is |
| --- | --- |
| Speakers | Realtek(R) Audio — **where playback was going** |
| Speakers | Mason's iPhone Hands-Free (Bluetooth) |
| Headphones | Oculus Virtual Audio Device |
| Realtek Digital Output | Realtek(R) Audio (S/PDIF) |
| BenQ PD3200U | NVIDIA High Definition Audio |
| LG Ultra HD | NVIDIA High Definition Audio |

Six active outputs, **two of them both called "Speakers"**, and two different Realtek endpoints — analog and S/PDIF — that both read as "Realtek Audio" in the tray. Windows' own per-app session record confirmed the binding: `singlelineouttopo_c1` on the Realtek codec, the rear analog jack. Nothing was broken at any layer; a working pipeline was pointed at a jack nobody was listening to, and the device names made that impossible to see.

- [x] `MachineSettings::audioOutputDeviceId` stores a chosen output as `QAudioDevice::id()`. Empty follows the system default, which stays the sane starting point.
- [x] A **Playback** group on the Settings page lists every output, marks which one Windows currently considers the default, and lets one be named outright. The id identifies the device because the description cannot — two entries share a name.
- [x] The backend honours the choice while the device exists and falls back to the default when it does not, so unplugging a headset gives sound somewhere rather than silence.
- [x] Test: the setting defaults to empty, round-trips an id, and going back to the default sticks rather than springing back to the last pick.

**The actual root cause, found by comparing a working device against a broken one.** Sound played through either monitor but not through the analog speakers, and every other application on those same speakers was fine. Reading each endpoint's configured format settled it:

| Device | Format | Plays in the app |
| --- | --- | --- |
| BenQ PD3200U | 48000 Hz **16-bit** | yes |
| LG Ultra HD | 48000 Hz **16-bit** | yes |
| Realtek Digital Output | 48000 Hz 16-bit | — |
| **Speakers (Realtek)** | 48000 Hz **32-bit** | **no** |

Qt's FFmpeg backend goes silent on an endpoint whose shared format is not 16-bit — no error, no warning, nothing. Every 16-bit endpoint works; the one 32-bit endpoint does not. This is also almost certainly the same defect as the speed-change crash: both are the FFmpeg backend's audio conversion path, which is `swresample`, which is where that stack trace pointed.

- [x] The player warns when the chosen output is not 16-bit, naming the device, the depth, and both fixes — change the format in the Windows Sound control panel, or pick another output under Settings → Playback. The format belongs to Windows, so warning is the whole of what can be done from inside the app.
- The user-side fix is Sound control panel → the device → Properties → Advanced → **16 bit, 48000 Hz**.
- If the FFmpeg backend keeps costing time, `QT_MEDIA_BACKEND=windows` swaps in Media Foundation and takes `swresample` out of the picture entirely. It would want the download format preference moved to H.264 first, since Media Foundation cannot be relied on for VP9 — which is what yt-dlp currently produces.

**A workaround that outlived its theory.** With Media Foundation the speakers finally played, but toggling speeds gave a black screen after a while. That was ours, not Qt's: the audio detach-and-reattach written to dodge the "rate change" crash was still there, and pulling an audio output mid-playback rebuilds Media Foundation's whole topology. The crash it was written for turned out to be the 32-bit endpoint, so the workaround was solving nothing and breaking something.

- [x] `applyPlaybackRate` sets the rate and does nothing else.
- [x] `audioPlaysAt` / `isAudioPlaying` / `updateAudioControls` removed with it, along with their two tests. **Sound plays at every speed again.** The restriction was reasoned from a wrong cause and dressed up as a product decision about pitch correction, which made it sound more considered than it was.
- [x] The decoder is a setting: **Settings → Playback → Decoder**, written to `QT_MEDIA_BACKEND` from `main()` before `QApplication` exists, because Qt reads it when the multimedia plugin first loads. Changing it needs a restart and says so rather than looking like a setting that did not take. The startup banner logs which backend is running.
- [x] Test: the setting is unset until chosen and normalises case, since it becomes an environment variable.

Both backends have now cost real time around playback speed — the feature this app exists for. If it keeps happening, **libmpv is the answer rather than another workaround**: it changes speed as a first-class operation and has pitch correction, so audio stays intelligible at 0.5x. `VideoPlayerBackend` was put there for exactly this, and `PlayerController` and its tests would not change.

Three lessons from this one, all of them about diagnosis rather than code. The first fix was necessary but not sufficient, and stopping there would have looked like a failure. Every layer reported success — no error from Qt, a valid file, correct deployment — because nothing was *broken*; a working pipeline was pointed at the wrong end. And the thing that finally cracked it was logging the state rather than reasoning about it: three rounds of plausible theories cost more than one line of output.

Worth keeping in mind for anything similar: some Instagram posts genuinely have no audio track. `Video by stefan.crainic.mp4` in the test library is one, so testing sound against the wrong clip proves nothing.

## Phase 3 — Library browsing and management

### 3.1 Library tree
- [x] `LibraryTreeModel`, a QAbstractItemModel over `LibraryTreeQuery`: Category → Year → Month → Day with counts rolled up into the label, months shown by name, empty categories included, and `filterFor(index)` so selecting a node and filtering the grid are one act
- [x] Tests: nesting, `parent()` walking back up from every level, counts in labels, per-node filters, month naming, `indexForCategory`, clearing

### 3.2 Video grid
- [x] `VideoListModel` over `VideoQuery` + a QListView in icon mode; five sort orders; tooltip carries what a tile has no room for
- [x] Tests: filtering, exposed roles, sorting without refiltering, `indexForVideo`, duration/size/title formatting
- Deviation: thumbnails load lazily on first draw and are cached (including the misses) rather than on a background thread. They are ~30 KB local JPEGs decoded only for rows on screen, so a loader thread would add concurrency for a delay nobody could perceive. Revisit if a library of thousands makes scrolling stutter.

### 3.3 Details panel
- [x] `VideoEditor` (business logic): rename, change category, edit details, delete — every file-moving operation paired with its catalogue update through `applyFileChangeThenDatabaseChange`
- [x] `VideoDetailsPanel`: inline rename, category picker that shows the resulting path and asks before moving, title/creator/notes, facts line (duration, dimensions, size, date), clickable source URL, Show in Explorer, delete to the Recycle Bin with a confirm
- [x] Tests (10): rename moves file and row together, collision steps past rather than overwrites, empty name and missing video refused, category change keeps the download date and prunes emptied folders, same-category is a no-op, **a move the catalogue refuses is undone on disk**, preview without side effects, details edit never moves the file, delete takes file + thumbnail + row + notes, deleting a video whose file is already gone still clears the row

### 3.4 Tag editor
- [x] `TagEditor`: chips in a wrapping row, a box completing against tags already in the library, Enter to add (case-insensitive, so "rope" attaches the existing "Rope"), Remove button and double-click to detach
- The assignment logic itself is `TagRepository`, already covered by 12 tests in Phase 1; the widget adds no logic worth a second set

### 3.5 Search view
- [x] Keyword box and sort selector in the library toolbar, composing with whatever the tree has selected — narrowing by folder and searching by word combine rather than fighting
- [x] Standalone `SearchView` page: keyword, category, year and month pickers, tag multi-select, sort, and a Clear all. Double-click plays; Show in Library jumps to the video in the tree
- [x] Tags narrow rather than widen — a video must carry *every* ticked tag. That is the useful direction for finding one drill among hundreds, and the opposite of what most search boxes do, so the group box says so outright
- [x] The year picker offers only years the library actually has, rather than a fixed range mostly pointing at nothing. Month is disabled until a year is chosen: a month alone would match August of every year, which is not what a month picker looks like it promises
- No new data-layer work was needed. `VideoFilter` already carried `tagNames` with AND semantics, year/month/day, category, keyword, and sort, and `VideoQuery` already implemented them — Phase 1 built the filter for the tree and the search page at once, and this page is the part that finally uses all of it

### 3.6 Existing-library adoption (per #13)

Two pieces, split so the rule can be tested away from both the disk and the catalogue.

- [x] `LibraryScan` reads a filed tree back out of its folders: `Rope/2026/08/06/entry drill.mp4` is the Rope category, downloaded that day. `parseRelativePath` is pure and holds the whole rule — exactly five path segments, a real date, zero-padded the way the app writes it, a video extension. **The scan writes nothing.**
- [x] `LibraryAdopter` writes the report into the catalogue in one transaction. It moves no files: the videos are already where they belong, which is why the folders could be read at all. Titles come from file names with underscores turned back into spaces
- [x] `AdoptFilesDialog` (File → Adopt Files Already Filed): scans on open, shows what will be added and what is being left alone with a reason for each, then adopts on request and rescans so the report stays honest
The schema seeds five categories (Inbox, Rope, Partner Acro, Handstands, Fitness Ideas), so a library filed the way this app files things adopts without inventing a single one. The adoption tests now say which side of that line each category name is on, because two of them originally asserted counts that assumed an empty table and failed against correct code.

- [x] Tests (14): date padding refused where it does not match, 2026/02/31 rejected while 2024/02/29 is kept, wrong depths in both directions, non-video files skipped for that specific reason, the app's own `.videolibrary` subtree ignored entirely, already-catalogued paths compared case-insensitively so Windows casing cannot duplicate a video, a walked tree grouped correctly, adoption round-tripped into rows with the folder date and file size, adopting twice adding nothing, a stale report replayed adding nothing, and one category created once for many videos

Two deliberate limits, both stated in the dialog rather than left to be discovered: **duration, dimensions, and thumbnails are not filled in** — those need ffprobe per file, and holding a folder-sized adoption behind hundreds of probes would turn a quick operation into a long one. Adopted rows carry title, category, path, date, and file size, which is enough to browse and search. And the date is midnight, because a folder path carries no time of day.

- [ ] **Follow-up**: backfill duration, dimensions, and thumbnails for rows that have none, as a background pass. Worth doing before Phase 6.

## Phase 4 — Player and notes

### 4.1 Player controller
- [x] `VideoPlayerBackend` seam (named for what it is rather than `IVideoPlayer`, matching the rest of the codebase) + `QtVideoPlayerBackend` over QMediaPlayer/QAudioOutput; libmpv remains the swap-in if playback quality disappoints, and the seam is the only thing that would have to change
- [x] `PlayerController` holds every decision: speed presets 0.25/0.5/0.75/1/1.25/1.5/2, clamped seek, throttled position ticks, and resume-on-open
- [x] Tests (19) against `FakeVideoPlayerBackend`: speed steps stop at the ends rather than wrapping, speed carries across videos, seek clamps to both ends and works before the duration is known, ticks are throttled but a state change always forces one, resume waits for the duration and is ignored when it is trivially small or within five seconds of the end

**Changing the playback speed crashed**, and the fix is a workaround for a Qt bug rather than a correction to our code.

Located in two steps. First, it happened from the speed box with the mouse *and* from the Up/Down keys — and those two routes share exactly one thing that does real work, `setPlaybackRate`. That one fact ruled out the key handling, the shortcut table, and the combo-box round trip without needing to read any of them. Second, the stack trace: **swresample below ffmpegmediaplugin**. swresample is FFmpeg's audio resampler, so Qt's FFmpeg backend is crashing while reconfiguring a running audio renderer for the new rate. Nothing in the player's own code was ever involved.

Detaching the audio renderer *around* the rate change was the first attempt and it did not work — it only moved the crash a second later, to the moment audio first reached the freshly built resampler. That ruled out "reconfiguring a live resampler" and established the real rule: this backend cannot resample audio at any rate but normal, full stop.

- [x] **Sound plays at 1x and nowhere else.** `applyPlaybackRate` detaches the audio output before leaving normal speed and reattaches it on returning, so a resampler never exists at a rate it cannot survive. Qt has no pitch correction, so off-speed audio was a growl or a chipmunk regardless — this is a rule the app can state plainly rather than an apology.
- [x] `PlayerController::audioPlaysAt(speed)` / `isAudioPlaying()` put that rule in the tested layer instead of leaving it buried in the backend, with tests over every preset. The mute button and volume slider grey out below and above 1x and say why in a tooltip — a volume slider that moves without changing anything is worse than one that plainly cannot be moved.
- [x] The rate is applied from the event loop rather than in the caller's stack. This did not fix the crash and was not kept for that reason: detaching and rebuilding an audio renderer is heavier work than a property set, and both routes in arrive from inside widget event delivery — one with a combo box popup closing on the same stack. Rapid presses coalesce to the last value, and `playbackRate()` reports the pending one so a caller is told what it asked for.
- Unchanged either side of this: `PlayerController` and its 17 tests. The fake backend is synchronous and knows nothing about audio renderers, which is the seam earning its keep — the bug was entirely on the far side of it.
- **If this resurfaces**, the escalation is `QT_MEDIA_BACKEND=windows` (Media Foundation instead of FFmpeg — fewer codecs, no swresample) and then libmpv, which `VideoPlayerBackend` exists to make a contained change.

Two further crash-class defects found while chasing it, both real, neither confirmed as the cause:
- `QtVideoPlayerBackend` declared `QMediaPlayer player_` before `QAudioOutput audioOutput_`. Members die in reverse declaration order, so the audio output was destroyed first and the player spent its own destructor holding a borrowed pointer to a dead object. The declaration order is now reversed, which is what makes the player die first.
- `PlayerView` showed the playback-error dialog on a **direct** connection. The backend emits `errorOccurred` from inside its own calls (`setSource`, `setPlaybackRate`), and a modal dialog runs a nested event loop — opening one on top of the media pipeline's own stack re-enters it mid-operation. The connection is now `Qt::QueuedConnection`, so the dialog opens after the backend's call has unwound.

Two decisions worth recording. **Resume is deferred until `durationChanged`**: seeking before the backend knows the length is silently dropped by QMediaPlayer, so a naive resume-on-open loses the position on every video. **A resume point near the end is discarded** — finishing a video and reopening it should start at the beginning, not at the last frame.

### 4.2 Player view
- [x] `PlayerView`: `QGraphicsVideoItem` in a `QGraphicsScene`, so the note overlay is a sibling graphics item rather than a widget layered over native video output — the latter is unreliable on Windows, the former always composites
- [x] Controls bar: play/pause, speed selector, seek slider, elapsed/remaining clock, volume, mute, full-screen toggle. The clock follows the slider while it is being dragged and only commits the seek on release, so scrubbing does not fight the position the backend is still reporting
- [x] Remembers the last playback position per video in `library_settings`, written on close, on switching videos, and on the way out of the window
- [x] Tests: `NoteTimeline::elapsedAndRemaining` (elapsed plus time left, clamped so a position that overshoots the duration reads `-0:00`, and a not-yet-known duration reads sensibly)

Full screen belongs to `MainWindow`, not to the view: the window is the thing with a menu bar, a navigation list, and a status bar to hide. `PlayerView` emits `fullScreenRequested(bool)` and the window does the rest, restoring a maximized window as maximized rather than quietly un-maximizing it. Leaving full screen also happens on close, so the saved geometry is never a chromeless window the next launch would reopen into.

Deviation from "in-window and true full screen": there is one full-screen mode, not two. The notes panel stays visible in it deliberately — the whole point of watching a drill full screen is to write about what you are seeing, and hiding the panel would make N open an editor nobody can see. Everything that is *not* the video or the notes goes away.

Two lifetime bugs found while wiring it up, both fixed rather than left to bite later: Qt deletes child widgets in `~QObject`, *after* `MainWindow`'s own members are gone, so the player would have written its resume position through an already-destroyed `DatabaseManager` — `~MainWindow` now hands the player a null catalogue while there is still something to write to. Closing a library does the same thing for the same reason.

### 4.3 Keyboard shortcuts
- [x] Data-driven `playerShortcuts()`: Space play/pause; ←/→ ±5s; Shift+←/→ ±10s; **Ctrl+←/→ jump to the previous/next note**; ↑/↓ speed step; F full screen; Esc exits it; M mute; N new note; ? cheat sheet. One list feeds both the key handler and the cheat sheet, so they cannot disagree about what a key does
- The arrows are one progression rather than three unrelated keys: bare moves five seconds, Shift ten, Ctrl all the way to the next thing worth looking at. Learnable instead of memorised, and there is a test that pins the whole ladder
- [x] **Every control carries its key**: "Play (Space)", "Full Screen (F)", "Mute (M)", "Speed (↑↓)", "Add Note (N)", "Notes (Ctrl+← prev, Ctrl+→ next)", and the seek slider's ladder in its tooltip. The labels are built by `withPlayerShortcut(text, action)` from the same `playerShortcuts()` list, never spelled out at the call site — rebinding a key relabels the button instead of leaving it advertising a key that moved. A test pins that, and another caps every key label at eight characters so a binding cannot quietly grow too long to sit next to a control
- [x] Key labels use arrow glyphs (←, →, ↑, ↓) rather than the words, written as UTF-8 escapes in the source so nothing depends on how the file is decoded. "Ctrl+Left" crowds a button in a way "Ctrl+←" does not
- [x] A **Keys (?)** button in the controls bar opens the cheat sheet. It was previously reachable only by pressing `?`, which is the one thing someone looking for the keys does not know yet
- [x] Tests (12): every bound key resolves, the whole arrow ladder resolves in order, unbound keys resolve to `None`, modifiers are matched exactly (Left and Shift+Left are different keys), irrelevant modifiers such as Num Lock are masked off, no two bindings claim the same key, control labels are built from the list rather than spelled out, every label is short enough to sit next to a control, and the cheat sheet lists every binding

Writing the last of those tests found a dead binding: `?` is Shift+/ on most layouts, so Qt folds the Shift into the key code *and* still reports the modifier. Matching modifiers strictly meant `Qt::Key_Question` + `NoModifier` could never match anything a keyboard can produce, and the cheat sheet was unreachable. Shift is now dropped for that one key, with a test that presses it the way a keyboard really does.
- Every control in the view is `Qt::NoFocus`, including the notes list, so clicking a button or a note never takes Space and the arrows away from the video. The note editor is the one exception, and while it has focus the keys are its own — typing "n" in a note must not start another note.

### 4.4 Active note resolution
- [x] `NoteTimeline::activeIndexAt`: the active note is the most recent one at or before the current position, held until the next note's timestamp; nothing before the first, and the last one stays after it
- [x] Tests: before the first, between two, exactly on a timestamp, after the last, an empty list, and a list that changes mid-playback

### 4.5 Notes UX
- [x] Add note (N or the button) pauses playback first — a note is written about the frame on screen, and the video running on underneath would move it — then opens an inline editor pre-stamped with the current timestamp. **Ctrl+Enter saves, Esc cancels**, so a note can be written start to finish without touching the mouse. Both are widget shortcuts rather than key handling, because `QPlainTextEdit` swallows Ctrl+Enter as a newline before a `keyPressEvent` would ever see it. `Key_Return` and `Key_Enter` are both bound — the main Enter and the keypad's are separate keys to Qt, and binding one leaves the shortcut silently dead on some keyboards. The shortcuts are written into the button labels rather than hidden in tooltips
- [x] On-video overlay showing the active note over a translucent backdrop, replaced when the next noted timestamp is reached. The backdrop is not decoration: white text alone vanishes over a bright frame, which is most handstand footage shot against a window
- [x] Notes list: click seeks to the timestamp, Edit reopens the same inline editor over the existing note, Delete confirms by naming the timestamp. Ctrl+←/→ steps between notes from the keyboard, and the list highlight follows the video rather than drifting out of step with it
- [x] `NoteTimeline::nextIndexAfter` / `previousIndexFrom` step **the note that is showing**, one either way, so the overlay moves by exactly one note per press. Neither wraps: running off the end stays put, because jumping from the last note back to the first mid-drill is disorienting

**Corrected after use.** The first version measured from the raw position — "previous" meant the last note strictly before it. That reads fine on paper and is wrong in the hand: a second of playback leaves the position past the current note's timestamp, so Ctrl+← went to the start of the note already on screen instead of the one before it. Both directions now derive from `activeIndexAt`, which makes them exact inverses by construction rather than by coincidence, and there is a test asserting that stepping forward then back lands where it started from any position. The old tests encoded the wrong rule and were replaced, not added to.
- [x] Copy notes as Markdown for pasting into Obsidian — `## title` then `- **0:05** body`, with multi-line bodies indented so they stay inside the bullet
- [x] Tests (22 across `NoteTimeline`): timestamp formatting either side of an hour, Markdown shape, multi-line indenting, the empty-list case saying so rather than handing over an empty clipboard, note lookup by id returning -1 for a stale id rather than falling back to the first note, and stepping forward/back including both ends, sitting exactly on a note, and a video with no notes at all

Deviation: the notes list is a plain `QListWidget` rebuilt from the repository on every change, not a `QAbstractListModel`. A video has tens of notes, not thousands, and every mutation already round-trips through SQLite; a model would add a layer with nothing in it worth testing. The logic that *is* worth testing — which note is active, how a timestamp reads, what the Markdown looks like, finding a note by id — is in `NoteTimeline`, where `QCoreApplication`-only tests can reach it.

## Phase 5 — Instagram integration

### 5.1 Cookie sources and login status (per Q7/#15)
- [x] Cookie-source setting, done early as **2.8/2.9** because Instagram downloads were broken without it: browser passthrough or a `cookies.txt` path, off by default, with a cookie failure falling back to a cookie-less attempt rather than sinking the download
- [x] `NetscapeCookies`: the format yt-dlp and Instaloader both accept, which makes it the one thing the browser, the downloader, and the Python helper can all agree on. Serialize, parse, filter
- [x] Tests (8): the header yt-dlp insists on, a round trip, comments and malformed lines stepped over (a jar exported by a browser extension routinely has something in it this does not need), **session cookies and expired ones dropped** with the clock passed in so expiry is testable, only the cookies Instagram actually needs kept — writing the whole jar would put more of someone's browsing on disk than the job calls for — and signed-in meaning `sessionid` and nothing less
**The embedded browser does not work, and cookies.txt is the supported route.** Instagram's login page loads inside QtWebEngine and renders an empty body: the log shows its own script running and emitting Permissions-Policy warnings, with no load failure and no GPU error. It detects the embedded browser and refuses. A Chrome user agent, `AA_ShareOpenGLContexts`, and enabled local storage each moved it closer and none of them fixed it, and no user-agent string wins that argument for long — the check is not only the UA and changes whenever Meta wants it to.

- [x] **Use cookies.txt...** on the Instagram page: pick a file exported by a browser extension, and it is read, filtered, and rewritten into the app's own session file. Copied rather than referenced, so deleting the export does not sign anybody out. A file with no `sessionid` is refused with the actual likely cause named — exported from a browser not signed in to Instagram — rather than "invalid file"
- [x] Tests (3 more): an export adopted and surviving the original's deletion with the tracking cookies filtered out, a file from the wrong site refused with a useful reason, and a missing file saying so
- This is what Q7 already concluded on this machine. The embedded browser stays in the build as a second option in case Instagram's behaviour changes, but it is not the path the app recommends.

- [x] `InstagramLoginDialog`: real Instagram in an embedded QtWebEngine view with a persistent per-machine profile, harvesting cookies from `QWebEngineCookieStore`. Nothing typed there reaches the app -- only the cookies Instagram hands back, and only the few it needs. WebEngine is optional at build time, and without it the dialog **explains the cookies.txt route** rather than leaving a dead button
- [x] `InstagramSession`: the harvested file lives under `AppPaths`, never in the library -- a session is not something to copy to a network share -- and is written owner-only, because it is a password in all but name
- [x] Login-status probe with `classifyProbeOutput`, pure so every branch is testable. **A rejected session says "sign in again" rather than "sign in"**, and a network failure reports as unavailable rather than signed out: sending somebody to re-authenticate because their wifi dropped teaches them to ignore the message
- [x] Tests (10 more): the file outside the library, a round trip, writing refused for cookies that are not a session or are already expired, signing out twice not an error, and each probe outcome

### 5.2 Saved-list helper script
- [x] `tools/list_saved_posts.py`: Instaloader session from a Netscape `cookies.txt`, iterate saved posts, emit JSONL (shortcode, url, owner, caption excerpt, taken_at, is_video, thumbnail_url), `--max-items`, politeness delay. One object per line rather than one array, so a long list is consumed as it arrives and one unreadable post cannot cost the run
- [x] `SavedPostParser` with tests (10) on fixture output: one line to one post, a broken line costing only itself, unreadable lines **counted rather than swallowed** (silently dropping them would make a broken sync look like a short saved list, which is the one thing the user could not tell apart), the script's own error line read as a message, a missing URL rebuilt from the shortcode, photos filtered out per Q9
- [x] The script ends with `{"complete": true}` and the parser carries it through. "No more lines" and "the list ended" are identical on a pipe, and only a list read to the end may retire posts that did not appear in it — so the script says so outright rather than leaving it to be inferred

### 5.3 Sync service
- [x] `SavedListReconciler`: takes the fetched list and the stored rows and returns a **plan**, changing nothing. The caller applies it, which is what lets every rule below be tested with no database and no network
- [x] Tests (10) on the state machine: a new post inserted, a known one only touched, a retired post revived when saved again, already-gone rows not retired over and over (which would report a change on every sync forever), and a whole mixed sync planned in one pass
- [x] **A decision outlives the post.** Downloaded, ignored, and queued rows are never retired when a post leaves the saved list: unsaving something on a phone does not un-download the video, undo a months-old "no", or cancel a download in flight. Re-saving an ignored post touches it rather than offering it again
- [x] **An incomplete fetch retires nothing.** A `--max-items` cap or a run that stopped early has not seen the posts it never reached; absence proves nothing about a list that was never read to the end, and retiring on it would quietly hide posts still saved
- [x] `SavedListSyncService`: runs the helper through `ProcessRunner`, parses as lines arrive, reconciles, applies the plan, and fetches preview images into `.videolibrary/thumbnails/sources/`. A failed image is a cosmetic loss, not a failed sync
- [x] **Rate limiting is a first-class outcome, not a hang.** Instaloader answers a 429 by sleeping eleven minutes and retrying, silently — from the app that is indistinguishable from a deadlock and cannot be cancelled. The helper now runs with `max_connection_attempts=1` so it fails immediately, and the service recognises a 429 and says what to do about it: wait a quarter of an hour, and stay off Instagram meanwhile, since every client counts against the same limit
- [x] The helper narrates each step to stderr — reading cookies, signing in, loading the profile, walking the list — and the service logs stderr **as it arrives** rather than holding it until the process exits. Holding it was useless for exactly the complaint that mattered: that the process never exits. `python -u` too, so a traceback cannot die in a buffer
- [x] Tests (2 more): a 429 explained rather than shown raw, and a rate-limited run retiring nothing — a sync that never reached the list says nothing about what is still saved
- [x] Minimum-interval guard (Q10), default 15 minutes. Instagram counts requests and the Sync button is something a person can lean on; refusing early is politer to the account than being rate limited by it. Injectable so the guard is tested without waiting a quarter of an hour
- [x] Tests (11) against a fake process runner and a real catalogue: a fresh list inserted, a second sync touching rather than duplicating, photos never becoming rows, an incomplete run retiring nothing, a complete one retiring what left, **a downloaded item surviving being unsaved**, the interval guard never reaching the network, the helper's own error surfaced, unreadable lines counted into the result, no session failing before any network call, and the last-sync time stored in `library.db` so it travels with the folder

### 5.4 Instagram view
- [x] `InstagramView` replaces the placeholder: a grid of saved posts with preview image, creator, and caption snippet, a Sync button showing when it last ran, and a state filter that defaults to **what is still worth acting on** — downloaded and ignored posts are decided, and showing them would bury the ones that are not
- [x] Actions: Download (multi-select, straight into the Phase 2 pipeline), Ignore, Preview, Open Post. Downloading marks the row queued so a sync running alongside cannot retire it mid-download
- [x] `SourceItemModel` rebuilt from the catalogue on every change rather than kept in step by hand, so what it shows survives a restart and cannot drift. Preview images load lazily and are cached, misses included
- [x] Tests (7) on the model: the default view, the state filter, what a tile says, a 400-character caption cut without breaking the grid while the tooltip keeps all of it, a post with no creator falling back to its shortcode, lookup by id, and a library closing underneath it leaving the model empty rather than stale

### 5.5 In-app preview (per Q12)
- [x] `MediaUrlResolver` runs yt-dlp `-g` and hands the URL to `PlayerView::openExternal`. **Nothing is written**: no file, no catalogue row, no resume position. Most saved posts turn out not to be worth keeping, and finding that out should not cost a download
- [x] Resolved per view, never cached: these URLs are signed and expire in minutes, so a stored one would be a broken link by the time anybody used it
- [x] Progressive formats only (`acodec!=none`) — a plain player cannot merge separate video and audio streams, and a preview that played silently would look like a broken app rather than a format choice
- [x] Open Post opens the real page in the system browser as the alternate action
- [x] Tests (6): the command prints instead of downloading, nothing that would write a file, cookies sent only when there are some, the first of two printed URLs taken, noise around the URL stepped over, and output with no URL resolving to nothing

### 5.6 YouTube parity
- [x] Verified by test rather than by assertion: every shape of YouTube link — `watch?v=`, `youtu.be`, `youtube-nocookie`, and one with a playlist and timestamp hung off it — reduces to **one canonical URL and one external id**, which is what stops the same video arriving twice under two links
- [x] A pasted batch mixing Instagram and YouTube links is picked up as both; a channel or search page is correctly not a video
- [x] The rest of the pipeline was already shared: `--ffmpeg-location` (2.10) matters most for YouTube, where video and audio genuinely do arrive as separate streams
- Watch Later enumeration (cookies + `:ytwatchlater`) stays a future provider. `SourceItemRepository` is already keyed by platform, so it would be a second sync service beside this one rather than a change to any of it

## Phase 6 — Hardening, portability, packaging

### 6.1 Consistency checker
- [ ] Report: DB rows with missing files, files under the root not in the DB, stale source_item links; repair actions (relink, adopt, remove row); checker-logic tests; Settings section UI

### 6.2 Tool health
- [ ] Settings panel: tool versions, yt-dlp staleness warning (Instagram breaks old versions; winget upgrade hint), python venv health, Instagram session status

### 6.3 Portability and backup (per #16)
- [ ] On open: rotate a copy of `library.db` into `.videolibrary/backups` (keep 5) + `PRAGMA quick_check`; stale-lock detection with a clear "library in use elsewhere / copied while open?" warning; relative-path audit test (no absolute paths ever stored)

### 6.4 Packaging and per-machine setup
- [ ] `tools/make_dist.ps1`: windeployqt into a self-contained folder (copy to any Windows machine), app icon, version stamp
- [ ] Assistant run-book in README: copy dist folder, run `tools/setup_machine.ps1 -Mode runtime` (winget yt-dlp/ffmpeg + Python venv), open the library, set up Instagram cookies (probe-guided), sync + download

### 6.5 UX polish
- [ ] Empty states, busy indicators, status-bar toasts for background completions/errors, dark-theme pass, shortcut cheat sheet

# Verification

Per phase, after Claude finishes the code (Mason builds and runs — no builds or git from Claude):
- **Tests**: build and run `video_library_tests` (CTest or the exe directly). The data/services/business-logic layers are covered there; UI glue is covered by the smoke lists.
- **Manual smoke per phase**: 0 — `tools/setup_machine.ps1` completes, first CMake configure resolves Conan deps + Qt kit, app launches, creates/opens a library folder, log file appears, Settings shows tool versions. 1 — `library.db` appears with all tables (inspect with any SQLite browser); copy the library folder elsewhere and reopen it. 2 — paste a real Instagram/YouTube URL; it downloads in the background and lands in `Inbox/{y}/{m}/{d}` with a thumbnail; queue two while one runs; cancel one. 3 — browse the tree, rename a file (check disk), recategorize (file moves), tag with autocomplete, search by tag/keyword/date; adopt the existing library and spot-check it. 4 — play a video: every speed, scrub with elapsed/remaining shown, fullscreen, every shortcut; add notes at two timestamps, watch the overlay switch at the second, click a note to jump, copy notes into Obsidian. 5 — set up Instagram cookies (probe confirms login, via browser passthrough or in-app login); Sync lists saved videos not yet downloaded; download two at once; ignore one; preview one without downloading; re-sync preserves states. 6 — delete a file on disk manually and confirm the checker reports/repairs it; build the dist folder and run it from a clean directory or second machine.

# Process Notes

- Mason builds the app, runs all tests, and handles all git commits/pushes; Claude writes code + tests, checks off plan checkboxes as items complete, and avoids git and external prompt-requiring tools (internal file tools only).
- Questions go into the Open Questions section of this document, not interactive prompts.
- Every change that can be tested gets tests (GoogleTest), following knottyyoga conventions: no test fixtures, self-contained tests beside sources, no assumed collection order. Qt-heavy UI glue is verified via the per-phase smoke lists instead.