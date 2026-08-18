---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/18/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

database_helper currently populates the database with a good amount of information when passed the flag --recreate_database. I'd like to add some things to this and then add another flag --recreate_database_test that does everything the --recreate_database does but then populates with some additional test data.

For these people, I'd like the following images to copied into the tree and added as their image associated with their account profile:

- Mason Bendixen (masonbendixen@gmail.com)
	- Image: D:\Pictures\Pics\Kauai.jpg
	- Copy into tree as Mason.jpg
- Caleb Ault
	- Image: D:\Pictures\Pics\Croc.jpg
	- Copy into tree as Caleb.jpg

In addition, I'd like to have Mason and Caleb both be configures as Instructors (with the same photo as their instructor photo for now) and also as service providers with the provider type as massage therapist.

Can you change the Knotty Yoga Monday 6pm slot so that the instructor is Mason Bendixen and the Wednesday one so that it is Caleb Ault. Can you add a Handstand class on Mondays at 7pm with Mason Bendixen as the instructor and require the person to attend the Monday 6pm Knotty Yoga class.

Can you add a class slot for Thursdays 6pm-7pm that is partner acrobatics taught by Mason Bendixen as well as Sunday 10am-11am.

Can you make these photos for these classes:
- Knotty Yoga
- Partner Acrobatics
	- Image: D:\Acrobatics Photos\2025\Oct\Oct Lorien Shoot\To Upload\20251028_054530759_iOS.png
	- Copy into tree as PartnerAcro.jpg
- Handstands
	- Image: D:\Acrobatics Photos\2025\Sep\Edited\



Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here