---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/29/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I would like the home page of the website to show upcoming events and essentially marketing material for when a user is not logged in. Once they have created an account, I'd like to show any upcoming events they have signed up for as well as upcoming events they could sign up for. I'd like to show the upcoming class schedule for the next four days as items on the home page with options like being able to make themselves as attending or not attending. If they don't have a membership, show the membership tiers on the home page. We should have a link to setup their class schedule template.

Let's do the following cleanup items:
- Let’s simplify the menu structure. Let’s have Our Classes just have All Classes and Our schedule. Let’s not have entries for separate classes.
- Let’s have the classes page have a link to the calendar
- Let’s have the Calendar have a link to the schedule template
- Let’s have the schedule template page have a link to the notification settings
- Get rid of Upcoming Events from the Services menu and make the Services menu just go to the current Browse Services page (so not be a dropdown menu anymore)
- Get rid of the all classes under Our schedule
- Turn Our Classes go straight to the current Our Classes / Our schedule so it is a menu item not a drop down menu
	- Have the attend this class buttons like the calendar
	- Have a link to the schedule template
	- If someone isn’t logged in, have all the classes just up with their time slots
		- Have the classes that require a membership be a link to buy a membership for people who don’t have a membership
- Change shop to Memberships
- We need a getting started page with detailed instructions
- Let’s redo the Our schedule page
- For a class, the Upcoming Sessions has too many entries. Link to calendar instead of showing upcoming sessions.
- Note that series (and presumably workshops), don’t really have a place to sign up for them anymore. This should be easy to discover and sign up. They should be able to sign up from the series and workshops tab in the profile, from the calendar, and from the class page.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here