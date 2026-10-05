---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 10/5/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I want you to make mobile look functional. Right now, many of the pages in the system look terrible on mobile. I would like you to put forth a plan to exercise the various pieces of the UI, do screen captures, look at the UI to see if the spacing or layout is off, and make the CSS / layout changes to fix this. I would like you to put forth a plan to get through to the top level pages first that are directly, publicly accessible, make the changes and then work on the flows involving having an account and being logged. Then work on the admin, staff, and user portal flows. We need to have various phone / tablet sized targets to check this out with. I'd also like to have you fully complete these stages, once defined, with little to no interaction from me minus design choices that really need my input. I read that you can spin up a browser and do things like this. Please put together a multi phase plan to accomplish this.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here