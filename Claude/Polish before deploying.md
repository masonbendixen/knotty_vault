---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/20/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I've been demoing the application getting it ready to deploy and have found a number of issues. I'm just going to list them. If you can try to group them into buckets to implement, ask me questions, and build an implementation plan to accomplish these, that would be great. Please use the code base and these documents for context:

- [[Component Inventory for Designer]]
- [[Splitting the server up into components]]
- [[Componentizing the frontend]]
- [[Converting the server to a multi tenant Saas architecture]]
- [[Deploying to AWS]]
- [[Home page work and cleanup items]]
- [[Tenant Theming and Branding]]
- [[Website Makeover]]

Here are the items that we need to improve before being ready to ship:
- After changing the contact email for the site, the footer doesn’t refresh until refreshing the page.
- 

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here