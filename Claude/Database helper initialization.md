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
	- Image: D:\Acrobatics Photos\2025\Sep\Edited\20220407_033309094_iOS.jpg
	- Copy into tree as KnottyYoga.jpg
- Partner Acrobatics
	- Image: D:\Acrobatics Photos\2025\Oct\Oct Lorien Shoot\To Upload\20251028_054530759_iOS.png
	- Copy into tree as PartnerAcro.jpg
- Handstands
	- Image: D:\Acrobatics Photos\2025\Sep\Edited\20241201_201508414_iOS.jpg
	- Copy into tree as Handstand.jpg

Please update the photo for each class accordingly.

Up to this point, all of this stuff has been just the regular database creation (--recreate_database). The following should just be for --recreate_database_test.

Please create an Event Session on the two following Saturdays after the tool is run with from 10am-11am with the Product Intro workshop. Allow booking and show on home page 14 days in advance.

Please create a Product called Aerial Series with the code aerial-series and the Description Introduction to aerial acrobatics on rope and fabric with the Kind being Class Series. Make the price for everyone $30 per session and $10 per session for Knotty Yoga Gold members. Then under Class Schedules, make a class series class schedule that is Tue / Thu 6-7pm from the time the tool is run through the end of the FOLLOWING month. Please make Caleb Ault the instructor for those sessions.

For Caleb as a service provider, please 

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here