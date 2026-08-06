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

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here