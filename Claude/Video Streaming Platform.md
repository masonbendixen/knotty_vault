---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 11/16/2026
Version: 0.7
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Use [[C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\Streaming Platform for Creators Business Plan.md]] for context.

I want to get actually implementing a video delivery platform. The other document has the business long term needs but I want a plan to start building out the implementation.

These are the things we need in phases
- Record and serve simple video
	- Capture video from a phone / camera
	- Serve video from sharing control
	- Choose a video playback solution like unlisted YouTube video initially
- Record and serve branded video
	- All features of previous section
	- Put title intro and closing video at start and end of video with transitions
- Record and serve edited video
	- All features of previous section
	- Be able to cut and rearrange sections of the video
	- Be able to splice in other video
	- Be able to do voice overs over sections of video
	- Be able to put text and overlay graphics over sections of video
- Be able to change playback speed
	- All features of previous section
	- Be able to playback video at slower or faster than normal speed and do overlays, voice over, and other inputs while this video is at a different speed or fully paused 
- Record and serve multiple source video
	- All features of previous section
	- Have multiple cameras / video inputs
	- Have a set of layouts that you can go through and toggle between at various points
		- Each input being full screen
		- Side by side horizontally with two of the inputs
		- Side by side vertically with two of the inputs
		- One input full screen and the other a positionable PIP window
		- Four quadrants
		- One input full screen and multiple PIP
- Be able to stream video in real time
	- All features of previous section
		- For instance, you should be able to start the intro sequence while streaming, choose to play a video file, and be able to toggle between inputs (like two cameras or a video playback file)
	- In addition to streaming, all the inputs and decisions should be saved locally so that a fully edited video can be generated later
	- Have a control that can play the video in real time
- Have a good editing solution for previously streamed video
	- All features of previous section
	- For instance, we might use OBS studio for streaming but it might create a Davinci resolve solution (or something else or stay in OBS) for editing later
	- The saved input sources should be high quality
- Be able to put section titles in the video
	- All features of previous section
	- YouTube and other platforms have markers in the video for things like explanation here, breakdown of move, questions and answer, demo, etc.
- Allow interaction with viewers
	- All features of previous section
	- Have people be able to watch and click a button desiring to interact
	- Allow creator to see the people asking to interact and upgrade their connection
		- Upgrading the connection switches to a low latency connection to the video from the creator
		- It also sends a stream from the viewer being upgraded that becomes an input source to the creator so the creator can switch the viewer to full screen, PIP, etc.
		- We should allow multiple viewers to be upgraded
		- Should be able to also interactively downgrade back to normal viewer
		- The viewers audio should be mixed in as well
		- Record the video locally from the viewer

This is all of the phases. Please note that I would like this to largely be done in server components and web components. Eventually, these client playback controls will be used in knotty yoga and community finder but I think we should create a new repo that has the creator application that is probably a C++ native app for doing the capture of video and editing with a dummy webserver based on crow to host the server components pieces and the dummy website to host the web components playback / client recording control. I think we should have a phase to explore various solutions like OBS, davinci resolve, or even AI based video solutions at least for intro sequences and overlays. I'm leaning towards Youtube Unlisted videos initially but open to suggestions. I'm thinking the creator app will probably be a QT native C++ app that deep hosts OBS lib but we could start with early phases driving OBS through the socket API and do deep integration in a later phase. 

Let's start putting together the implementation plan but let's start with you offering a lot of ideas, doing research, and asking me questions.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Plan status

**v0.7 (11/16/2026): this is now the hands-on document.** Caleb, Mason's partner, is taking on much of this project. Caleb is technical but not a developer, so the plan is now in two documents:
- **This document:** gear, accounts, installs, studio setup, experiments, scripting (command lines, batch files, small Python or Blender scripts), brand assets, rehearsals and hand testing. Nothing here needs application code.
- **[[Video Streaming Platform API]]:** the development work. That covers the architecture, the code in each phase, the research behind it, the technical decisions (D-n) and Open Questions 1–22. The history of v0.2–v0.6 moved there too, since it's about code.
- **New in this version:**
  - **a pipeline lab,** for choosing:
    - how intros and outros get made, with the class title set by a script
    - how recordings get edited after a stream
    - how to make a logo morph from an aerial move
  - **a Tool landscape,** expanded from the old tool survey with the tools the lab compares
  - **what Caleb does alongside each development phase**
- **Moved in from the old Phase 0:**
  - 0.2, the cameras and mics, with Mason's purchases
  - 0.3
  - the hands-on parts of 0.4, 0.5, 0.7 and 0.8

  Section numbers match the API document's, so each document can point to the other.
- **Gear status (from v0.5):** two OBSBOT Tiny 3 Lite cameras were bought on 10/30, and one OBSBOT Vox SE mic was ordered the same day.

# How to use this document

- **What the project is.** Mason is building a video platform for movement teachers, starting with Knotty Yoga:
  - record classes with OBS, two OBSBOT cameras and a wireless mic
  - add the studio's intro and outro, and publish to YouTube as unlisted videos
  - later, stream live and bring viewers on camera

  The business case is in [[Streaming Platform for Creators Business Plan]].
- **Your part.** Items tagged **[Caleb]** are yours. Items tagged **[Mason]** need Mason: purchases, accounts that must be in Mason's name, and decisions. Untagged items are shared.
- **Recording results.** Each experiment has a **Findings** item and a **Recommendation** line. Fill them in as you go, including what didn't work. Mason reads the recommendation and decides, and the decision goes in **Production decisions**.
- **Ticking boxes.** Tick an item when it's done and add the date, like `✅ 2026-11-20`.
- **Questions.** Answer inline under each question in **Open Questions**, starting your answer with "Caleb-". Mason's answers start with "Mason-". Every question has a default, which is used if nobody answers.
- **Files.** Lab inputs, scripts and renders go in the shared folder from OQ-23, in one subfolder per experiment (`E-03 Blender`, …). Name each render after its experiment and title, like `E-03_aerial-rope-foot-locks_v2.mp4`.
- **Scripts.** The lab is about command lines and small scripts, because the winning pipeline has to run with nobody at the keyboard. Claude can write a first version of any script here (OQ-30). Paste section L.1, *Shared inputs*, into the request along with what the script should do.

## Suggested order
1. **0.2, multi-source capture:** the cameras are here, and the code is waiting on its findings.
2. **E-1, the FFmpeg baseline** (an evening): it sets the bar every other intro has to beat, and most flows reuse its join script.
3. **Book the logo-morph shoot** (L.5): it needs a studio slot and a performer, so get it on the calendar early. E-10 uses the footage.
4. **E-3 (Blender) and E-7 (AI backgrounds),** then E-2, which combines them.
5. **E-9, the editing bake-off,** once 0.2 has produced a real two-camera recording.
6. The rest of the lab, then 0.3, 0.4, 0.5, 0.7 and 0.8.

# Pipeline lab

**Goal:** find the cheapest way to make every class video look branded, where a new class title is one line in a spreadsheet rather than an hour in an editor. The lab has three questions:
1. **Intros and outros:** which tool makes a good-looking intro and outro whose class title a script can change? (L.3)
2. **After the stream:** should editing happen in OBS, or in something like DaVinci Resolve? (L.4)
3. **The logo morph:** can an aerial move on red fabric turn into the Knotty Yoga mark, and how? (L.5)

Each answer becomes a production decision, PD-1 to PD-5. PD-1 also decides how the creator app builds intros in Phase 2 of the API document.

## L.1 Shared inputs

Every experiment uses the same inputs, so the results compare fairly.

**Brand kit**
- **Logo:** `ui/src/assets/svg/KnottyYoga_logo_white.svg` in the knottyyoga repo; Mason can send it.
  - **Left:** the mark, a figure hanging in red fabric. The fabric is red `#EC1C25` and the figure is white.
  - **Right:** the wordmark "KNOTTY YOGA", in white.
  - Because so much of it is white, this version only works on dark backgrounds.
- [ ] [Mason] Find the designer's source files (Illustrator or Figma), and export three versions: the mark alone, the wordmark alone, and the full logo for light backgrounds (OQ-29). Until then, the SVG is the source.
- [ ] [Caleb] Make transparent PNGs, 3840 px wide, of the full logo and of the mark alone. Some tools, FFmpeg among them, can't read SVG.
- **Colours:** the website's brand red is `#ED1C26`, the same red as the logo's `#EC1C25`. The site's full palette is on its style guide page (Admin › Style Guide); ask Mason for a screenshot.
- **Fonts:** headings and display text are in **Barlow** (bold and condensed weights), and body text is in **Roboto**. Both are free from Google Fonts. Install them on every machine that renders.

**Title data**

One spreadsheet, `titles.csv`, drives every pipeline. Its columns match the parameters the creator app will pass in Phase 2:

| Column | Example | Notes |
|---|---|---|
| `slug` | `aerial-rope-foot-locks` | Safe in a file name; names the output |
| `title` | `Aerial Rope: Foot Locks` | Required |
| `level` | `Beginner` | Optional |
| `instructor` | `Mason` | Optional |
| `date` | `2026-11-20` | Optional |

Use these five titles, because they break careless scripts:
1. an ordinary one: `Aerial Rope: Foot Locks`
2. a long one, about 50 characters: `Partner Acrobatics: Standing Two-High Transitions`
3. punctuation that command lines treat specially: `Rock 'n' Roll: Back-Bends & 50% Holds`
4. accented letters and a dash: `Café Flow — Señor's Stretch`
5. one short word: `Handstands`

A pipeline passes the **title test** when all five render correctly, centred and inside the frame, with no hand-editing.

**Output specs**
- **Finished video:**
  - 1920×1080, at the class recording's frame rate (30 fps, unless 0.2 picks 60)
  - H.264 video, AAC audio at 48 kHz
  - loudness about −14 LUFS, YouTube's playback level
- **Intro:** 4–8 s. It needs an audio track, either the sting from L.6 or silence, or joining it to the class fails.
- **Outro:** 10–20 s. YouTube's end-screen elements (subscribe, next video) can only go in the last 5–20 s of a video, so leave a clear area for them.
- **A transparent version** of each intro and outro, so OBS can play it live as a stinger transition:
  - Render a ProRes 4444 `.mov`, or a PNG sequence, as the master.
  - Convert it to WebM (VP9 with alpha) for OBS.
  - Test both in OBS: forum reports disagree on which plays more reliably.
- **Later, not now:** vertical 1080×1920 versions for Shorts and Reels (OQ-31).

## L.2 How to score a pipeline

Score each pipeline from 1 to 5 on each row, with a sentence on why.

| # | Criterion | 5 means | 1 means |
|---|---|---|---|
| 1 | **Look** | You'd put it on the studio's channel today | It looks like a template |
| 2 | **Title by script** | `titles.csv` goes in, five videos come out, and the title test passes | Every title is made by hand |
| 3 | **Time per video,** once set up | Under 2 minutes of your time; note the render time too | Over 30 minutes |
| 4 | **Setup effort** | Under 2 hours to the first good template | Days |
| 5 | **Cost** | Free | A subscription you can't drop after the lab |
| 6 | **Live use** | The same intro can play live in OBS | Offline only |
| 7 | **Rebrandable** | Another studio's logo, colours and fonts go in as data | It needs a redesign |
| 8 | **Runs unattended** | A program could run it: a command line, no sign-in, no clicking | Someone has to work a GUI |
| 9 | **Risk** | Free, offline tools with clear rights | Unclear rights to the output, or a vendor that could change it at any time |

Rows 7 and 8 matter more than they look. Mason's platform is meant for many teachers and studios, so the winner has to work for studios other than Knotty Yoga, and the creator app has to be able to run it with nobody at the keyboard.

## L.3 Intros and outros with a title set by script

Run E-1 first: every other experiment has to beat it. Each experiment makes an intro and an outro for all five titles in `titles.csv`, plus one finished sample: the intro, a minute of a real class, and the outro.

### E-1 FFmpeg only (the baseline)
FFmpeg is a free command-line tool that can do nearly anything to video, with no window at all. Here it draws the title over a dark background with the logo, fades it in, and joins the pieces.
- [ ] [Caleb] Install FFmpeg (`winget install Gyan.FFmpeg`) on the machine you use. Mason's laptop already has it.
- [ ] [Caleb] Write `make_intro`, a PowerShell or Python script that reads `titles.csv` and makes one intro per row. A starting point for one title, in PowerShell:
  ```powershell
  $row = (Import-Csv titles.csv -Encoding UTF8)[0]
  [IO.File]::WriteAllText("$PWD\title.txt", $row.title)
  ffmpeg -f lavfi -i color=c=0x111111:s=1920x1080:r=30:d=6 -i logo.png -f lavfi -i anullsrc=r=48000:cl=stereo -filter_complex "[1:v]scale=1400:-1[logo];[0:v][logo]overlay=(W-w)/2:220[bg];[bg]drawtext=fontfile=Barlow-Bold.ttf:textfile=title.txt:expansion=none:fontsize=96:fontcolor=white:x=(w-text_w)/2:y=760:alpha='min(max(t-1,0),1)',fade=t=out:st=5.5:d=0.5[v]" -map "[v]" -map 2:a -t 6 -c:v libx264 -pix_fmt yuv420p -c:a aac "$($row.slug)_intro.mp4"
  ```
  - **Put the title in a file and use `textfile=`,** as above. A title typed inline with `text=` breaks on `:`, `'` and `\`, and fails the title test.
  - **Keep `expansion=none`.** Without it, FFmpeg reads `%` as the start of a code, and "50% Holds" fails.
  - **Write `title.txt` as UTF-8 without a BOM,** as above. In Windows PowerShell, `Set-Content` would mangle the accents, and `Import-Csv` needs `-Encoding UTF8` for the same reason.
  - **Keep the font file next to the script** (`fontfile=Barlow-Bold.ttf`). A Windows path such as `C:/…` needs its colon escaped inside a filter.
  - **Long titles:** `drawtext` doesn't wrap or shrink text, so the script has to pick a smaller `fontsize` for long titles or split them over two lines.
- [ ] [Caleb] Write `join`, which puts the intro, the class and the outro together with short cross-fades and evens out the loudness. The core of it, for the intro and the class:
  ```powershell
  ffmpeg -i intro.mp4 -i class.mp4 -filter_complex "[0:v][1:v]xfade=transition=fade:duration=0.5:offset=5.5[v];[0:a][1:a]acrossfade=d=0.5,loudnorm=I=-14:TP=-1[a]" -map "[v]" -map "[a]" -c:v libx264 -c:a aac -ar 48000 out.mp4
  ```
  - `offset` is the intro's length minus the fade's.
  - Both clips need the same size, frame rate and pixel format. If FFmpeg complains about timebases, add `settb=AVTB,fps=30` to each video input before `xfade`.
  - One pass of `loudnorm` is close enough to start with.
- [ ] Findings: the scorecard, the render time per intro, and which titles needed help.
- **Recommendation:**

### E-2 FFmpeg over a designed background
E-2 aims for the look of E-3 or E-7 at the speed of E-1. Render a background animation once, with a clear band for the title, and let FFmpeg draw each title over it.
- [ ] [Caleb] Make a 6–8 s background, from E-3 (Blender), from E-7 (AI), or from a stock motion background.
- [ ] [Caleb] Change E-1's script to use it in place of the plain colour (`-i background.mp4`).
- [ ] Findings: the scorecard.
- **Recommendation:**
- *This is the likely sweet spot. The expensive part is made once, and each new title costs seconds.*

### E-3 Blender template
Blender is free 3D and motion-graphics software. Python can control all of it, and it renders without opening its window (`-b`). One template file holds the design, and a short script swaps the title and renders.
- **Version:** use **Blender 5.2 LTS**, which is supported to July 2028, so the template keeps working. Blender 5.3 came out on 11/10/2026.
- [ ] [Caleb] Build `intro.blend`:
  - **The title:** a text object named `Title`, in Barlow. Give it a text box (Object Data properties › Text Boxes) and set **Paragraph › Overflow** to **Scale to Fit**, so long titles shrink to fit.
  - **The logo:** imported as curves (File › Import › SVG), given depth and a red material.
  - **Motion:** a camera move, lights, and a 4–8 s animation.
  - **Render engine:** **Eevee**, which is much faster than Cycles.
  - **Transparency:** turn on Film › **Transparent**, for the alpha version.
- [ ] [Caleb] Write `set_title.py`:
  ```python
  import bpy, sys
  title, output = sys.argv[sys.argv.index("--") + 1:][:2]
  bpy.data.objects["Title"].data.body = title
  bpy.context.scene.render.filepath = output
  bpy.ops.render.render(animation=True)
  ```
  Then run it once per row. Blender's installer doesn't put `blender` on the PATH, so add its folder or use the full path:
  ```powershell
  Import-Csv titles.csv -Encoding UTF8 | ForEach-Object { blender -b intro.blend -P set_title.py -- $_.title "out/$($_.slug)_" }
  ```
- [ ] [Caleb] Render to a PNG sequence with transparency. Then use FFmpeg to make the MP4, the ProRes 4444 master, and the WebM for OBS.
- [ ] Findings:
  - the scorecard
  - the render time per intro on each machine; Mason's laptop has no NVIDIA GPU
  - how the title test went
- **Recommendation:**

### E-4 OBS alone, live
There's no rendering in this one. The intro is an OBS scene that plays at the start of every class and is recorded along with it.
- [ ] [Caleb] Build an `Intro` scene:
  - **A looping background:** a media source playing E-2's background or E-7's loop.
  - **The logo:** an image source.
  - **The title:** a **Text (GDI+)** source with **Read from file** ticked, pointing at `current_title.txt`. Change the file, and the title changes, even live.
  - **Motion:** Exeldro's **Move** plugin, which moves and fades sources when the scene changes. Check that it loads on OBS 33, which moved where plugins go (API document R-1).
- [ ] [Caleb] Automate the start. Exeldro's **Advanced Scene Switcher** plugin can switch from `Intro` to the camera scene after 6 s, and play the outro on a hotkey.
- [ ] [Caleb] Add a stinger transition made in E-3 or E-6 (the transparent WebM), and check that it plays smoothly while recording.
- [ ] Findings: the scorecard, and whether anything stutters while recording and streaming at once.
- **Recommendation:**
- *Later, the creator app will set the title through OBS's websocket, and `current_title.txt` will go away.*

### E-5 DaVinci Resolve title template
Resolve's Fusion page builds animated titles that can be saved as templates. The title text then shows as a field you fill in on the Edit page.
- [ ] [Mason] Install DaVinci Resolve 21 (free), unless 0.5 already has.
- [ ] [Caleb] Build the intro as a Fusion title, and save it as a template with the text published as a control. Do the same for the outro.
- [ ] [Caleb] Make a project template containing the intro, a placeholder for the class, the outro, the transitions, and a "YouTube 1080p" render preset.
- [ ] Findings: the scorecard.
  - Since Resolve 21.1, scripting is Studio-only ($295; API document R-4), so in the free edition every title is typed by hand.
  - Is that acceptable when the video is being edited in Resolve anyway (L.4)?
- **Recommendation:**

### E-6 Template services: Canva and Jitter
This is the no-code end: online design tools with animated templates, where a spreadsheet fills in the text.
- [ ] [Caleb] **Canva:** design an intro and outro, then run **Bulk Create** with `titles.csv`. It needs Canva Pro or higher and the desktop website. Check that it makes one video per row; most guides only show it making images.
- [ ] [Caleb] **Jitter:** do the same with Jitter's Bulk Create, which added CSV import on 7/21/2026. Check which plan it needs, and whether it exports video with transparency.
- [ ] Findings: the scorecard. Both fail row 8 unless their APIs can render with nobody signed in, so check what each offers and on which plan.
- **Recommendation:**

### E-7 AI-generated background loops
Today's AI video models make beautiful motion, but still mangle exact text and logos (API document R-11). So use them for backgrounds only, and lay the title on top with E-2.
- [ ] [Caleb] Generate 6–8 s loops, such as red silk rippling on black, slow light moving across the studio, or chalk dust in a beam of light. Try two or three services among Veo 3.1 (Google), Kling and Luma.
- [ ] [Caleb] **Make them loop seamlessly.** Give a model that takes both a first and a last frame (see L.5's list) the same image for both, or use Luma's loop option.
- [ ] [Caleb] Record the cost of each usable clip. Read each service's terms for commercial use on the plan you used.
- [ ] Findings: the scorecard, judged with the background used in E-2.
- **Recommendation:**

### E-8 HTML/CSS title card in OBS (needs Mason)
This was the API document's default (idea I-3): the title card is a web page.
- **Live:** OBS shows it as a browser source.
- **Offline:** the creator app captures it to a video file.
- **Branding:** each studio's colours and fonts come from its website theme automatically.

It scores well on rows 6–8 by design, so the question is whether it can look as good as the others.
- [ ] [Mason] Build a throwaway card page that reads the title from its address (`?title=`). This is API document 0.6.
- [ ] [Caleb] Try it as an OBS browser source, and judge its look against E-3 and E-2.
- [ ] Findings: the scorecard.
- **Recommendation:**

## L.4 After the stream: edit in OBS or in Resolve?

**Short answer: OBS can't edit.** It's a live switcher and recorder, so once a recording stops, OBS can't trim, cut or rearrange it. The real choice is between two approaches:
- deciding as much as possible live in OBS, so the recording comes out nearly finished
- fixing things afterward in an editor

**The approach to test: decide live, fix little.**
- **Live, in OBS:** the intro, camera switches and layouts, chapter markers (on a hotkey), and the outro. For most weekly classes, the recording should be publishable as it is.
- **Afterward, by script:** trim the start and end, set the loudness, and upload. That takes minutes, not hours.
- **Afterward, in an editor:** only when something went wrong, or for a polished piece such as a highlight reel or a technique breakdown.

**Which editor, for those times:**
- **DaVinci Resolve (free):** the strongest free editor, with multi-camera editing, Fusion graphics and Fairlight audio.
  - **Limits:** since 21.1, scripting is Studio-only, and the free edition imports timelines only through its menus (R-4).
  - **Magic Mask,** which cuts a person out of the background, is Studio-only according to most sources.
- **Kdenlive:** free and open source, and already on Mason's laptop.
  - **Scriptable:** its project file is plain XML, so a script can swap in a title or a clip, and its engine, `melt`, renders a project from the command line.
  - **Caution:** forum posts report command-line renders failing silently, so test it.
- **Descript:** you edit the video by editing its transcript, and its Studio Sound cleans up speech. It's fast for talking but weaker for movement, and it's a subscription.
- **Blender's Video Sequencer:** 5.3 added ripple editing and early support for OpenTimelineIO. It's worth a look only if E-3 wins and you're working in Blender anyway.
- **CapCut (desktop or phone):** quick templates and captions. Read its terms before putting studio footage through it.

The results also tell Mason how much editing to build into the creator app (API document OQ-12).

### Flows to compare

| Flow | How a class becomes a video | Editor | Scripted? |
|---|---|---|---|
| **F-1 Live only** | OBS records the program with the intro and outro built in. Upload it as it is, trimming in YouTube Studio if needed. | none | no |
| **F-2 Record, then script** | Record. A script trims it (start and end times come from a small text file), adds the intro and outro, sets the loudness and exports. | none | yes |
| **F-3 Resolve** | Record, open the Resolve project template (E-5), cut by hand, render. | Resolve | no |
| **F-4 Kdenlive template + script** | Design the template in Kdenlive once. A script puts the title and the class clip into a copy of the project file, and `melt` renders it. | Kdenlive | yes |
| **F-5 Multi-camera re-edit** | OBS records each camera to its own file (Source Record, 0.2). Resolve lines the angles up by their sound, and you re-cut them. | Resolve | no |
| **F-6 Phone only** | The phone's camera, then a phone editing app's template, then upload. | phone app | no |
| **F-7 Descript** | Record, trim by transcript, add the intro, export. | Descript | partly |

F-5 is a hand-made rehearsal of what the creator app will automate in Phases 5 and 7. If re-cutting the angles afterward isn't worth the time when done by hand, Mason should know that before building it.

### E-9 Editing bake-off
- [ ] [Caleb] Take one real 15–20 minute class recorded with two cameras (from 0.2), and finish it through F-1 to F-5. Add F-6 or F-7 if you like.
- [ ] [Caleb] For each flow, log:
  - your hands-on minutes
  - the render time
  - what went wrong
  - whether someone else could repeat it from your notes
- [ ] [Caleb] Show Mason the finished videos side by side, without saying which flow made which.
- [ ] Findings: the scorecard rows that apply (1, 3, 4, 5 and 8).
- **Recommendation:** the default flow for a weekly class, and when to use something else.

## L.5 The logo morph

**The idea:** a 4–6 s opening. A performer does an aerial move on red fabric, and the shot transforms into the Knotty Yoga mark, a figure hanging in red fabric, before the wordmark appears. Where it could go:
- the start of every video, ahead of the title card
- the end of the outro, run in reverse: the logo becomes a person who drops away
- the website and social posts

**The key: choreograph backward from the logo.** The closer the last live frame matches the mark's silhouette, the easier every technique below gets, and the less an AI model has to invent.

### Shoot
- [ ] [Caleb] Study the mark. Print it, and note the body's angle, where the fabric runs, and where the head, hands and feet are.
- [ ] [Caleb] With the performer (OQ-27), choreograph a 3–5 s move that ends holding exactly that shape for 2 s.
- [ ] [Caleb] Shoot it:
  - with the camera **locked on a tripod**, at the angle that matches the mark
  - against a **plain background**: plain is what makes cutting the performer out possible, and dark suits the white logo
  - with even, steady light, and the white balance and exposure locked
  - at **4K and 60 fps** if the camera can, with a fast shutter (1/250 s or faster) to keep the fabric and limbs sharp
  - with fabric as close to the logo's red as you can get; it can be colour-corrected later
  - in 5–10 takes, plus 5 s of the empty background (a "clean plate") from each camera position
  - optionally with the move in reverse, rising into the pose, for the outro
- [ ] [Caleb] Pick the best take, and export its last frame as a still. Every technique starts from it.

### Ways to make the morph

| # | Technique | How | Tools | Effort | Control |
|---|---|---|---|---|---|
| **M-1** | **Match cut and dissolve** | Cut the performer out of the last frame and flatten them to the logo's colours: white body, red fabric. Dissolve into the vector mark laid exactly over them, then let the mark settle and the wordmark slide in. | Resolve or Blender's compositor, plus a background remover | About a day | High |
| **M-2** | **Shape morph** | Trace the performer's outline over the last frames, and animate that shape into the mark's shapes. | After Effects, or Blender (Grease Pencil or curves) | Days | Highest |
| **M-3** | **AI first and last frame** | Give a video model the real last frame as its *first* frame, and the logo on a matching background as its *last*, with a prompt such as "the aerialist and red silk fabric smoothly transform into a flat red-and-white logo". Then splice: real footage, the AI morph, then the real vector logo for the final frames. | A model that takes a last frame (below) | Hours, plus credits | Low to medium |
| **M-4** | **Stylise, then settle** | Restyle the clip step by step toward the logo's flat two-colour look, either with filters (posterise, threshold, isolate the fabric's red) or with an AI restyle of the footage. Then dissolve onto the mark. | Resolve, FFmpeg filters, AI video-to-video tools | 1–2 days | Medium |
| **M-5** | **Full 3D** | A rigged 3D figure on simulated fabric, posed from your footage with markerless motion capture and shaded flat. The camera ends square-on, where the silhouette *is* the logo. | Blender | Weeks | Highest, and the colours are data |

**Cutting the performer out** (M-1, M-2, M-4):
- **Most reliable:** shoot against a plain background and key it out.
- **AI background removers:** in Canva, CapCut and Runway.
- **Resolve's Magic Mask:** Studio-only, according to most sources.
- **Open models:** Meta's SAM 3 (its own license, with a gated download) and BiRefNet (MIT license). These need some technical setup.

**Models that take a last frame** (for M-3, and for E-7's loops), from a September 2026 survey of models sold through APIs:
- **Take a last frame:**
  - Veo 3.1: clips of 4, 6 or 8 s, and 1080p forces 8 s
  - Kling 3.0 and Kling O1
  - Seedance
  - Hailuo 3
  - Wan 2.7
  - LTX 2.5
  - Luma Ray
- **First frame only:** Runway Gen-4.5 and Wan 3.
  - *A first-frame-only model doesn't refuse a last frame: it ignores it and still charges.*
- **Prices** ran from about $0.03 to $0.40 per second of video, so each try costs between a few cents and a few dollars.
- **Not covered:** Kling 4.0 came out on 10/15/2026, after the survey.

### E-10 Logo morph
- [ ] [Caleb] Make **M-1 first**, from the best take. It's the safe version.
- [ ] [Caleb] Try **M-3 alongside it**, for about an hour: 5–10 tries on each of two models, from the same first and last frames.
- [ ] [Caleb] If M-3 makes a good middle, **combine them**: real footage, then the AI transformation, then M-1's exact landing on the vector mark.
- [ ] [Caleb] Export the result as an MP4 and as a transparent version, and try it ahead of E-1's intro.
- [ ] Findings:
  - the time and money spent
  - which looked best
  - whether it would wear well at the start of every video
- **Recommendation:**
- *Keep M-2 and M-5 in reserve, in case the first two look cheap.*

## L.6 Sound: the intro sting and music
- [ ] [Caleb] Find three candidate 2–4 s stings for the intro, and a 15–20 s music bed for the outro.
  - **Sources:** YouTube's Audio Library (free on YouTube), a licensed library such as Epidemic Sound or Artlist (subscriptions), or a commissioned piece.
  - **AI-made music:** read the service's terms on who owns the result before using it.
- [ ] [Caleb] Check each candidate against YouTube's copyright matching (Content ID). Upload a test video as private: the Checks step of the upload reports any copyright claim. A claim on the intro would mean a claim on every class.
- [ ] Findings.
- **Recommendation:** (PD-5)

## L.7 Summary for Mason

| Question | Recommended | Runner-up | Why | Decision |
|---|---|---|---|---|
| Intro and outro pipeline | | | | PD-1 |
| Logo morph technique | | | | PD-2 |
| Editor for fixes after a stream | | | | PD-3 |
| Default flow for a weekly class | | | | PD-4 |
| Intro sting and music source | | | | PD-5 |

# Tool landscape

What exists for each job, as of November 2026. This extends the original plan's survey with the tools the lab compares. Prices come from third-party summaries, so check them before paying.

- **Capture and switching:**
  - **OBS:** chosen (API document D-1).
  - **vMix:** Windows, paid. Remote guests, instant replay, several recorders.
  - **Ecamm:** Mac only.
  - **StreamYard and Restream Studio:** run in a browser, with several guests and layouts.
  - **Blackmagic ATEM Mini ISO switchers:** hardware that records every input, plus a Resolve project of the live cuts (R-13).
- **OBS add-ons the lab uses,** all by Exeldro (check each on OBS 33):
  - **Move:** animated moves between scenes.
  - **Advanced Scene Switcher:** automation without code.
  - **Source Record:** a separate file per camera.
  - **Replay Source:** instant replay at a chosen speed.
  - **Freeze filter:** holds a frame.
- **Phone as a camera:**
  - **The phone's own camera app,** then copying the file over: the simplest, and the best quality.
  - **Camo, DroidCam, Iriun:** turn the phone into a webcam.
  - **Windows 11 Phone Link:** turns an Android phone into a webcam.
  - **NDI HX Camera** with the DistroAV plugin.
  - **VDO.Ninja** in the phone's browser.
- **Intros, titles and motion graphics:**
  - **FFmpeg:** command line, free. `drawtext`, `overlay`, `xfade`, `loudnorm`. (E-1, E-2)
  - **Blender:** 3D and motion graphics, free, scripted in Python, and renders without its window. (E-3, M-1, M-5)
    - **5.2 LTS** came out in July 2026 and is supported to July 2028.
    - **5.3** came out on 11/10/2026, adding Vulkan by default, Gaussian splats and improvements to its Video Sequencer.
  - **OBS** text, image and media sources with Move: live only. (E-4)
  - **DaVinci Resolve Fusion:** title templates with published text fields; scripting is Studio-only. (E-5)
  - **Canva** and **Jitter:** online, template-driven, with Bulk Create from a CSV file. Subscriptions. (E-6)
  - **After Effects:** the industry standard, on a Creative Cloud subscription. Add it to the lab only if you already know it.
    - Essential Graphics templates (MOGRTs) with editable text
    - unattended rendering with `aerender`
    - data-driven plugins such as Templater
  - **Cavalry:** procedural 2D motion design with scripting. It has a free tier; check what that includes.
  - **Lottie:** vector animations stored as JSON files, whose text can be replaced. They're made in After Effects or LottieFiles' web editor, and play in web pages, so in OBS through a browser source (E-8).
  - **Remotion** and **Motion Canvas:** video written as code (React, TypeScript), so Mason's side if ever. Remotion is free for companies of up to 3 people.
  - **Natron:** a free node-based compositor, scriptable in Python, but little developed in recent years.
- **AI video:**
  - **Models that take a first and a last frame:** Veo 3.1, Kling 3.0 and O1, Seedance, Hailuo 3, Wan 2.7, LTX 2.5, Luma Ray. (M-3, E-7)
  - **First frame only:** Runway Gen-4.5, Wan 3.
  - **Open-weight models** (Wan, LTX) can run on your own GPU through ComfyUI, but 8 GB of video memory (the studio PC's RTX 3070 Ti) is tight.
  - Good at motion and texture; still unreliable for exact text and logos.
- **Cutting people out of video:**
  - chroma or luma keying, in every editor
  - background removers in Canva, CapCut and Runway
  - Resolve's Magic Mask: Studio-only, according to most sources
  - Meta's SAM 3: makes masks; its own license
  - BiRefNet: MIT license; soft edges
- **Editing:**
  - **DaVinci Resolve:** free, or Studio for $295. Since 21.1, the free edition imports timelines only through its menus.
  - **Kdenlive:** open source, reads OTIO, and renders from the command line with `melt`.
  - **Shotcut:** also built on MLT, so the same scripting approach works.
  - **Descript:** edit by editing the transcript; Studio Sound.
  - **CapCut desktop:** templates.
  - **Blender's Video Sequencer.**
  - **Interchange formats,** for building a timeline in one program and opening it in another: OTIO, FCPXML, EDL.
- **Speed and replay:**
  - OBS's media source: a speed setting (`speed_percent`), plus play, pause and seek
  - Replay Source, and the Freeze filter
  - FFmpeg's `setpts`, `atempo` and `tpad`, in rendered video
  - the mpv player, for listening tests
- **Real-time:** VDO.Ninja, LiveKit, MediaMTX, Broadcast Box, Cloudflare Stream's WebRTC, coturn, and libdatachannel (C++, for Mason's Phase 10).
- **Hosting:** YouTube unlisted (decided for now); Bunny Stream, Cloudflare Stream, Mux; S3 + CloudFront.
- **Speech:** whisper.cpp for transcripts, captions and suggested chapters; OBS's live captions.

# Phase 0 — Hands-on probes

These are the original plan's probes that need hands rather than code. Their numbers match Phase 0 of [[Video Streaming Platform API]]. Probe 0.1, the OBS control probe, is done, and stays there. Each probe here ends with findings that Mason's code depends on, listed there under "What the code needs".

## 0.2 Multi-source capture probe

*In the camera and microphone notes, "you" means the teacher on camera.*

**Camera recommendation (10/29/2026, from Mason's question: matching cameras, not a lot of money).** Prices are from listings checked 10/29, so verify them at checkout.

**How each camera connects:**

| Connection | Use it when | Pros | Cons |
|---|---|---|---|
| **USB-C (UVC webcam)** | The cameras can sit within ~5 m of the PC, or ~10 m with an active USB 3 cable or a USB-C fiber cable (~$30–80; check it's rated for 5 Gbps). | • **Cheapest.**<br>• **Lowest latency:** a frame or two, so no audio delay to set.<br>• **One cable:** power and video.<br>• **Full control:** exposure and white balance from the app, and later from the creator app through the camera's SDK. | • **Cable length.**<br>• **USB ports:** each camera wants its own USB 3 port. Several identical webcams can get mixed up in OBS on Windows. The usual fix is to put each camera on a different port or controller and keep it there. |
| **Ethernet (NDI HX, PoE)** | A permanent installation with cameras far from the PC (ceiling corners, the far wall). | • **One cable:** power, video and control, up to 100 m through a PoE switch.<br>• **Saved camera positions** (PTZ presets).<br>• **Scales** to many cameras. | • **About 2–3× the price** per camera.<br>• **About 100–150 ms of latency:** set a matching audio delay in OBS.<br>• **Compressed video.**<br>• **Needs the DistroAV plugin** (6.2.2 is "OBS 33+ ready") and the NDI runtime. |
| **Wireless (Wi-Fi NDI, phone apps, wireless HDMI)** | Only for a roaming or temporary angle. | • **No cables.** | • **Unpredictable:** latency varies, and the picture can drop out.<br>• **Batteries.**<br>• **Interference** in a busy room.<br>• **Not suitable for the main angles.**<br>• The phone already covers this role (0.3, I-4). |

**Recommendation:**
- **Buy:** two identical **OBSBOT Tiny 3 Lite** USB-C webcams. They list at $199 and were $169 on sale in June.
- **Total:** about **$400–520**, including two 10 m active USB 3 cables if the PC is far from where the cameras go.
- **Third angle:** your phone, as the plan already has it.
- **Why this camera:**
  - **AI tracking on a motorized mount:** it pans and tilts to follow you, which matters for inversions and rope work that a fixed wide shot would crop.
  - **4K:** room to crop and reframe in the edit, even though the stream goes out at 720p or 1080p.
  - **Manual exposure and white balance:** you can set both cameras the same, so they match.
  - **An SDK:** the creator app can push identical settings to every camera.
  - **Current model:** this year's model is still for sale, so a matching third camera will be available later. A cheaper previous-generation model may not be.
- **If the studio light is dim:** spend $150 more per camera on the **Tiny 3**, which has a much larger 1/1.28" sensor (the Lite's is 1/2"). The same advice applies to both: buy the same model throughout.
- **Making the angles match:**
  - Lock each camera to manual white balance at the same Kelvin value.
  - Lock exposure (shutter and ISO).
  - Turn off HDR and auto-framing exposure.
  - Save the same profile in OBSBOT Center.
  - Identical cameras still won't match if light from windows changes during a class. Steady LED studio lighting is the real fix, and can come later.
- **Where to buy:** a shop with a 30-day return window, so 0.2 runs before it closes.
- **Not recommended at this budget:**
  - **Mirrorless cameras plus capture cards:** they look the best, but the cheapest suitable body (Sony ZV-E10 II) is about $1,000–1,200, and reviews report overheating in long 4K sessions.
  - **NDI PTZ cameras:** consider them if you later want a ceiling-mounted permanent installation. Options:
    - AVKANS LV20N, about $430
    - SMTAV 4K NDI, about $650
    - OBSBOT Tail Air, about $600 with NDI, plus a $99 Ethernet/PoE adapter
    - OBSBOT Tail 2, about $1,300, with built-in PoE

**Microphone recommendation (10/30/2026, after Mason bought the two Tiny 3 Lite cameras: is the built-in mic enough?)**
- **Use a separate wireless mic for your voice.** Each camera has three built-in mics with AI noise reduction. They're fine at arm's length on a desk, but not across a teaching room:
  - **Distance:** at 2–4 m your voice competes with everything else in the room: echo off hard floors and mirrors, mats, breathing, students, and music.
  - **Processing:** noise reduction working that hard to pull a distant voice forward tends to sound processed.
  - **Movement:** you face away from the cameras for much of a class, and sometimes you're upside down. A worn mic stays the same distance from your mouth whichever way you face.
- **Buy: one OBSBOT Vox SE, $59.** It's made for the Tiny 3 series:
  - **No receiver:** it pairs with a camera directly (in OBSBOT Center: the camera's Audio settings, then OBSBOT Microphone, then add TX1).
  - **One cable:** the camera passes your voice to the PC as its own microphone, over the same USB cable as the picture, so voice and picture stay in step and no extra USB port is used.
  - **Specs:** about 9 g, 20 hours per charge, magnetic or clip mount, up to 100 m in the open.
  - **A second one** ($59) covers a partner or co-teacher for acro. One camera takes two.
  - **Limits:** it pairs only with Tiny 3 and Tiny 3 Lite cameras, has no recording of its own, and your voice depends on that one camera staying connected.
- **Wearing it through inversions:**
  - Clip it to the centre of a neckline or strap, about 15–20 cm below your chin, where clothing won't rub it.
  - Don't trust the magnet alone upside down. Use the clip, or the back clip under a layer.
  - Run your most inverted sequence once as a test and check that it stayed put and didn't rustle.
- **In OBS:**
  - **One voice source:** the camera with the Vox SE paired. Mute the other camera's mic in the stream. Two live mics on one voice sound hollow.
  - **A backup track:** record the other camera's built-in mic on a separate recording track, muted in the stream. It's a spare if the Vox SE drops out, and gives room sound for the edit.
  - **Noise suppression:** start on Low in OBSBOT Center. A worn mic doesn't need much.
- **Music:** a worn mic still picks up music from the room's speakers. Commercial music in a YouTube stream or upload can get the video muted or claimed (Content ID), so play music you're licensed to stream.
- **Only if you want your voice independent of the cameras:** a USB-C wireless kit such as the DJI Mic Mini (two transmitters and a receiver, about $45–80 in the US lately) plugs its receiver into the PC. It takes a USB port, and its latency differs from the cameras', so OBS needs a sync offset. Not needed now.

- [ ] [Caleb] Inventory the cameras, capture cards, phones and microphones you'd actually use (OQ-11). If buying, the default is two OBSBOT Tiny 3 Lite cameras (see above).
  - *10/30: two Tiny 3 Lite cameras bought, and one Vox SE mic ordered. Phone: still to confirm.*
- [ ] [Caleb] Install *Source Record* on OBS 33, in the new plugin folder.
- [ ] [Caleb] Record two cameras and one phone (as a webcam) for 10 minutes, with a separate ISO file per source.
- [ ] Findings:
  - GPU and CPU load on the 3070 Ti, using NVENC for H.264/HEVC (30-series cards can't encode AV1)
  - whether the ISO files start at the same moment
  - disk use per hour
  - any crashes
  - whether the record clock (API document D-5) drifts over 10 minutes. Tell Mason before you record, so Mason can run `obs_probe watch` alongside it and compare its stamps with the file.
  - whether two identical USB cameras keep their identities in OBS after unplugging and rebooting
  - whether the cameras are best run at 1080p60 or 4K30 (USB bandwidth and decode load against room to crop)
  - how many NVENC sessions the run uses: one ISO per camera, plus the program recording, plus a stream. Recent drivers allow 8 at once on GeForce cards; confirm that on this driver.
  - whether the Vox SE's audio reaches OBS through the camera's USB audio device, and in sync with that camera's picture (clap on camera, then compare the clap's frame with the audio spike in the file)
  - how the Vox SE sounds next to a camera's built-in mic, recorded on separate tracks in the same session

## 0.3 Phone capture probe
- [ ] [Caleb] Find out which phone or phones would be used: iPhone or Android, and which model. Answer under OQ-11 in the API document, or tell Mason.
- [ ] [Caleb] Compare four ways of getting a phone's picture in, on quality, delay and fuss:
  - the phone's camera app, then copying the file over
  - Camo, DroidCam or Iriun, which make the phone a webcam
  - the NDI HX Camera app with the DistroAV plugin in OBS
  - VDO.Ninja in the phone's browser, standing in for Mason's future web component
- [ ] Findings:
  - the route for Phase 1 (default: the camera app, then copy the file)
  - the route for a phone as a live camera in Phase 5
- **Recommendation:**

## 0.4 YouTube accounts and upload checks
- [ ] [Mason] Pick the channel or channels (API document OQ-7), and give Caleb access to it. If it's a Brand Account, the Manager role is enough.
- [ ] [Caleb] Turn on live streaming for the channel now. It takes about 24 hours plus phone verification, and Phase 6 needs it.
- [ ] [Caleb] Create a Google Cloud project, turn on the YouTube Data API v3, and create an OAuth client of type *Desktop app*. Leave the consent screen in **Testing**, and add the channel owner's Google account as a test user.
- [ ] [Caleb] Upload a short clip as **unlisted** with a Python script that uses Google's API client library. Give it a title, and a description with chapter timestamps (`0:00 Intro`, `0:15 Warm-up`, `0:40 Demo`). Claude can write the script (OQ-30).
- [ ] [Caleb] Check two things Mason's code depends on:
  - **R-5:** does the upload stay **unlisted**, or does YouTube force it to private because the project hasn't been audited?
  - **R-6:** while the consent screen is in Testing, does sign-in stop working after 7 days? Run the script again on day 8, without signing in again.
- [ ] Findings.

## 0.5 Editing hand-off check
Mason's creator app will save each session's editing decisions as an OpenTimelineIO (`.otio`) file, which editors can import. This checks what survives the trip.
- [ ] [Mason] Install DaVinci Resolve 21 (free), unless E-5 already has.
- [ ] [Caleb] Once Mason's `otio_probe` has written a timeline from 0.2's camera files, import it into free Resolve (File › Import › Timeline) and into Kdenlive.
- [ ] [Caleb] Note what survives in each: the cuts between cameras, the picture-in-picture position, the chapter markers, and the audio tracks.
- [ ] [Caleb] Check that free Resolve can play 0.2's camera files at all. 10-bit and 4:2:2 HEVC files need Studio.
- [ ] Findings: is free Resolve with an imported timeline good enough, or is Studio ($295) worth buying (API document OQ-15)?

## 0.7 Interaction probe
This is for Phase 9, where a viewer can be brought on camera. All three of these are set up through their own apps and settings files.
- [ ] [Caleb] **VDO.Ninja:** bring one remote guest (a friend at home) into OBS as a browser source. Check three things:
  - that the guest doesn't hear themselves (mix-minus)
  - the round-trip delay
  - the picture quality
- [ ] [Caleb] **LiveKit:** run a room with two guests on LiveKit Cloud's free tier, and record each guest separately (track egress, through LiveKit's `lk` command-line tool).
- [ ] [Caleb] **MediaMTX:**
  - Run MediaMTX on a spare machine.
  - Send OBS's output to it over WHIP (OBS › Settings › Stream › Service: WHIP).
  - Watch it over WHEP in a browser.
  - Compare the delay with YouTube Live at its ultra-low latency setting.
- [ ] [Caleb] Test the studio's upload speed at class times, several times, since streaming depends on it.
- [ ] Findings:
  - which stack Phase 9 should use (API document OQ-16)
  - whether YouTube stays the broadcast for everyone else (OQ-17)

## 0.8 Variable-speed checks
This is for Phase 4: slow motion and freeze frames while teaching.
- [ ] [Caleb] **OBS media source:** play a clip at 25%, 50% and 200% using its speed setting. Pause and resume, hold a freeze frame, and narrate live over it.
- [ ] [Caleb] **Replay Source on OBS 33:** an instant replay of a camera at half speed, with frame stepping.
- [ ] [Caleb] **Drawing on the picture:** try an OBS drawing plugin for arrows and circles on a paused frame (API document I-13). Note which ones work on OBS 33.
- [ ] [Caleb] **Listening test:** play a class clip with speech in the mpv player at 0.25×, 0.5×, 1.5× and 2× (`mpv --speed=0.5 clip.mp4`). How does the voice sound? Mason's editor will probably use the same engine.
- [ ] Findings: the Phase 4 approach.

# Alongside each development phase

This is the hands-on work for each phase of [[Video Streaming Platform API]]. Each phase there ends with a **Verification** section of hand tests. Those are yours to run once Mason says a phase is ready: start from a blank database and follow the menus as written.

## 1. Record and serve simple video
- [ ] [Caleb] Record three real sample classes to test with: a short one, a long one, and one with music.
- [ ] [Caleb] Set up the YouTube channel's look:
  - the banner and profile picture
  - a playlist for each type of class
  - default upload settings: unlisted, a category, and a description template
- [ ] [Caleb] Run Phase 1's Verification.

## 2. Record and serve branded video
- [ ] [Caleb] Produce the final intro, outro and lower third with the pipeline chosen in PD-1, as reusable templates along with their scripts.
- [ ] [Caleb] Write down what each template needs: its fields, fonts, colours and files. Mason's code fills them in.
- [ ] [Caleb] Run Phase 2's Verification.

## 3. Record and serve edited video
- [ ] [Caleb] Once the creator app's editor exists, edit two of the sample classes both in it and in PD-3's editor, and report what the app's editor is missing.
- [ ] [Caleb] Set up a quiet corner for voice-overs, and test the Vox SE, or another mic, for them.

## 4. Playback speed
- [ ] [Caleb] Film material for slow motion: aerial and acro moves at 60 fps, or 120 if a camera can. Phase 4's slow playback and analysis are tested with it.

## 5. Multiple cameras
- [ ] [Caleb] Plan the studio's camera positions, mounts and cable runs (10 m active USB cables), and its lighting. Steady LED light is what makes the cameras match.
- [ ] [Caleb] Build the six layouts by hand as OBS scenes, and use them in a real class:
  - each camera full screen
  - two side by side
  - two stacked
  - one camera with a picture-in-picture
  - four quadrants
  - one camera with several picture-in-pictures

  Report which ones mattered. Mason's app will generate these scenes, and your trial shows Mason which ones to polish.
- [ ] [Caleb] Try the sync slate (a clap on camera at the start; API document I-11), and check that external recordings line up by it.

## 6. Stream video in real time
- [ ] [Caleb] Write the run of show for a typical class as a cue list: intro, cameras, a clip, Q&A, outro (API document I-10).
- [ ] [Caleb] Stream three unlisted test classes to YouTube Live, and note dropped frames and other problems.
- [ ] [Caleb] Try a Stream Deck, a foot pedal, or a phone as a remote for changing scenes mid-move (API document OQ-22).

## 7. Edit previously streamed video
- [ ] [Caleb] Decide where session folders live (a fast SSD while recording, then the NAS), and script the copy to the NAS with `robocopy` (API document OQ-21).
- [ ] [Caleb] Measure the disk space a full session uses per hour, with every camera plus the program.

## 8. Section titles
- [ ] [Caleb] Write the list of chapter kinds (explanation, breakdown, Q&A, demo, …), with an icon for each.
- [ ] [Caleb] Check YouTube's chapter rules on a real upload: the first chapter at 0:00, at least three chapters, and each at least 10 s long.

## 9. Interact with viewers
- [ ] [Caleb] Recruit two or three friends as test guests, on different devices and connections.
- [ ] [Mason] Decide the consent wording guests agree to before going on camera (API document OQ-20).
- [ ] [Caleb] Run Phase 9's Verification with the test guests.

## 10. Deep integration
- Nothing hands-on.

## 11. Adopt in Knotty Yoga and CommunityFinder
- [ ] [Caleb] List the videos on Knotty Yoga's site now, and propose categories for a members' video library.

# Production decisions

These are filled in when Mason decides on a lab recommendation. Decisions that change code are copied into the API document's Decisions.

| # | Decision | Chosen | Date | Why |
|---|---|---|---|---|
| PD-1 | Intro and outro pipeline; also settles API OQ-14 and Phase 2's approach | | | |
| PD-2 | Logo morph technique | | | |
| PD-3 | Editor for fixes after a stream; also informs API OQ-12 and OQ-15 | | | |
| PD-4 | Default flow for a weekly class | | | |
| PD-5 | Intro sting and music source | | | |

# Open Questions

Answer inline under each question, starting with "Caleb-" or "Mason-". The **bold** text is the default, used if nobody answers. The numbers continue from the API document's 1–22, so no two questions in the two documents share a number.

23. **Shared files.** Where do the lab's inputs, scripts and renders live?
    - **Default: a shared cloud folder called "Video Lab", with a subfolder per experiment.**
    - The scripts that win move into the `creator_studio` repository later, under `brand/` or `lab/`.
24. **Reading and editing this document.**
    - **Default: Mason shares this Obsidian vault, or just these two documents, through the sync Mason already uses.** Caleb answers inline here.
25. **Which machine.**
    - **Default: Caleb's own computer for the lab, and the studio PC (RTX 3070 Ti) for anything involving OBS, the cameras or NVENC.**
    - Mason's laptop has FFmpeg and Kdenlive installed but no NVIDIA GPU, so Blender and local AI run slowly there.
    - What's in Caleb's computer (GPU, operating system)?
26. **Lab budget.**
    - **Default: up to $100 in total without asking first, for AI credits, a month of Canva Pro or Jitter, and music trials. Nothing billed annually.**
27. **The performer and the fabric for the logo morph.**
    - **Default: a studio instructor, on the studio's red aerial fabric.** If the fabric isn't red, colour-correct it, or borrow one for the shoot.
    - Who, and when is the studio free?
28. **What the morph ends on.**
    - **Default: the mark alone, then the wordmark "KNOTTY YOGA" slides in beside it**, so it ends on the full logo.
29. **Logo source files.**
    - **Default: Mason looks for the designer's Illustrator or Figma files.**
    - Until they turn up, the white SVG from the website is the source, and the version for light backgrounds is made from it by changing the colours.
30. **Using Claude for scripts.**
    - **Default: yes.** Caleb asks Claude, on claude.ai or in Claude Code, for first versions of scripts, pasting L.1 *Shared inputs* into the request. The scripts are kept in the shared folder.
31. **Vertical versions** (Shorts, Reels, TikTok).
    - **Default: not in the first round.** Add them once PD-1 is chosen.
32. **AI in the studio's brand.**
    - **Default: fine for backgrounds and for the morph's transformation, but never to fake a performance.**
    - When uploading, answer YouTube's question about altered or synthetic content honestly. A stylised logo morph shouldn't need the label, but check.

# Sources

**Pipeline lab (checked 11/16/2026)**
- [Blender 5.3 released (Phoronix)](https://phoronix.com/news/Blender-5.3-Released)
- [Blender 5.3 overview (Digital Production)](https://digitalproduction.com/2026/11/11/blender-5-3-splat-happens-and-everything-else-gets-better/)
- [Blender 5.2 LTS (Phoronix)](https://www.phoronix.com/news/Blender-5.2-Released)
- [Blender command-line arguments](https://docs.blender.org/manual/nb/latest/advanced/command_line/arguments.html)
- [Which AI video models take a first and last frame (September 2026)](https://mer.vin/2026/09/which-ai-video-models-actually-take-a-first-and-last-frame/)
- [Kling start and end frames (Higgsfield)](https://higgsfield.ai/blog/Kling-Start-End-Frames)
- [Jitter changelog: Bulk Create with CSV import](https://jitter.video/changelog/2026-07-21-bulk-create-csv-import)
- [Canva Bulk Create guide (MoClaw)](https://moclaw.ai/blog/canva-bulk-create)
- [Bulk-creating videos in Canva (walkthrough)](https://sophiecress.com/blog/create-bulk-videos-easily-with-1763131334323)
- [DaVinci Resolve 21 release notes](https://www.blackmagicdesign.com/support/content/readme/59dd4eef1f4941c29fb8dc48b33f5c87)
- [Resolve free vs Studio (CineD)](https://www.cined.com/davinci-resolve-an-in-depth-comparison-between-the-free-and-studio-version/)
- [Rendering a Kdenlive project with melt (KDE mailing list)](https://mail.kde.org/pipermail/kdenlive/2016-March/008945.html)
- [kdenlive_render man page](https://www.mankier.com/1/kdenlive_render)
- [OBS forum: WebM with alpha for stingers](https://obsproject.com/forum/threads/adding-alpha-channel-to-webm-files.61655/latest)
- [OBS forum: track matte stingers](https://obsproject.com/forum/threads/unwanted-transparency-in-track-matte-for-stinger-transitions.154700/latest)
- [Remotion license FAQ](https://www.remotion.dev/docs/license-pricing-compliance/faq)
- [AI background removal models (InVideo)](https://invideo.io/blog/ai-background-removal-models/)
- [Meta SAM 3 overview](https://studio.aifilms.ai/blog/meta-sam3-text-segmentation-tracking)

**Cameras (0.2, checked 10/29/2026)**
- [OBSBOT Tiny 3 Lite review (CGMagazine)](https://www.cgmagonline.com/review/hardware/obsbot-tiny-3-lite-webcam)
- [OBSBOT Prime Day 2026 prices](https://the-gadgeteer.com/2026/06/24/obsbot-prime-day-2026-the-tiny-3-drops-to-296-and-the-tiny-2-hits-a-historic-low/)
- [OBSBOT Tiny 3 specs](https://www.obsbot.com/obsbot-tiny-3-series-4k-ptz-webcam/specs)
- [OBSBOT SDK](https://www.obsbot.com/sdk)
- [Tiny 3 vs Insta360 Link 2 Pro](https://www.streamtechreviews.com/blog/tiny3-link2-pro)
- [Insta360 Link 2 and 2C pricing (TechCrunch)](https://techcrunch.com/?p=2883557)
- [OBS forum: identical webcams in OBS](https://obsproject.com/forum/threads/2-x-4k-webcams-exactly-the-same-wont-recognize-them-individually.171912/)
- [Lightware 10 m active USB cable, rated for 4K webcams](https://archive.lightware.com/cab-usb-amaf-t1000a)
- [OBSBOT Tail Air store page](https://www.obsbot.com/store/products/obsbot-tail-air)
- [OBSBOT Tail 2](https://www.obsbot.com/obsbot-tail-2-ptzr-4k-live-production-camera)
- [Videomaker: Tail Air review](https://www.videomaker.com/reviews/cameras/obsbot-tail-air-review-packed-with-features-but-not-without-flaws/)
- [AVKANS NDI PTZ cameras](https://avkans.com/collections/ndi-ptz-camera)
- [DistroAV (NDI for OBS)](https://obsproject.com/forum/resources/distroav-ndi%C2%AE-integration-into-obs-studio.528/)
- [Videomaker: Sony ZV-E10 II overheating](https://www.videomaker.com/reviews/cameras/sony-zv-e10-ii-review-a-powerful-camera-for-creators-held-back-by-overheating/)

**Microphones (0.2, checked 10/30/2026)**
- [OBSBOT Tiny 3 series launch (the triple mic array)](https://www.obsbot.com/news/tiny-3-series-launch)
- [OBSBOT Vox SE store page](https://www.obsbot.com/store/products/vox-se)
- [TechBuzz Ireland: Tiny 3 and Vox SE review](https://techbuzzireland.com/2026/02/04/obsbot-tiny-3-and-obsbot-vox-se-wireless-microphone-review/)
- [DJI Mic Mini (B&H)](https://www.bhphotovideo.com/c/product/803515260-USE/dji_cp_rn_00000432_01_mic_mini_compact_wireless.html)
- [DJI Mic Mini two-transmitter kit price, 9/2026 (Slickdeals)](https://slickdeals.net/f/19992681-45-dji-mic-mini-wireless-microphone-system-2-tx-1-mobile-rx-at-amazon)
