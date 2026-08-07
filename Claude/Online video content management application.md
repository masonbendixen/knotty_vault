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

Improvement made while wiring this up: the download command now also asks yt-dlp to print the title, creator, id, platform, and upload date (a second `--print after_move:` with a marker and separator). Without it an imported video arrived titled with its platform id — "ABC123" — which is exactly the renaming chore the app exists to remove. It costs no extra request, since it comes from the run that already happened.

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

### 5.1 Cookie sources and login status (per Q7/#15)
- [ ] Cookie-source setting, probed in order: (1) `--cookies-from-browser <browser>` passthrough (chrome first per Mason's experience; known-broken for Chrome v20 cookies on Windows — the probe decides), (2) embedded QtWebEngine login with a persistent per-machine profile — `CookieHarvester` on QWebEngineCookieStore writes a Netscape `cookies.txt` for yt-dlp and hands session values to the Python helper, (3) user-supplied `cookies.txt` path
- [ ] Login-status probe (yt-dlp simulate against a login-required URL) surfaced in Settings and the Instagram view; re-login flow
- [ ] Tests: Netscape serialization, cookie filtering/expiry logic, probe-output parsing (pure functions)

### 5.2 Saved-list helper script
- [ ] `tools/list_saved_posts.py`: Instaloader session from the active cookie source (`load_session` on values parsed from cookies.txt or harvested cookies), iterate saved posts, emit JSONL (shortcode, url, owner, caption excerpt, taken_at, is_video, thumbnail_url), `--max-items`, politeness delays; runs in the venv from 0.1
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