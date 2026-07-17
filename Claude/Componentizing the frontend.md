---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/17/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Take a look at [[Splitting the server up into components]]. We want to do something similar for the frontend. I don't think there are as many things to componentize on the front end but I think the following are good candidates:

- Most of the controls for the CRUD stuff under knottyyoga/ui/src/app/controls
- The auth service stuff knottyyoga/ui/src/app/core/services
- The CRUD stuff, and just the CRUD stuff, under knottyyoga/ui/src/app/pages/admin
- The auth stuff under knottyyoga/ui/src/app/pages/auth
- There is a photo editor / upload thing I think somewhere

A lot of these things use ServerAccess so we might want to create some kind of indirection layer on top of server access that calls server access and have these components call that layer. Then other clients can provide some kind of callback / interface that sits on their equivalent to server access.

Please scan the code base for other possible nuggets that are good candidates for componentization.

I'm not sure what the best mechanism is for a separate component. I imagine we will have a new repo on github that is a sibling to 

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here