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

# Place plan here