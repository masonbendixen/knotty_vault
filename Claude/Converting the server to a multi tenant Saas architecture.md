---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 6/1/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Deploying to AWS (see Deploying to AWS.md), I have realized that there is an opportunity given that Cloudfront serves the client and reverse proxies to the webserver for /api. I have always wanted to support multiple clients with my server code but I have realized I can support multiple clients with the same server instance and probably the same RDS database.

I can have either the same Angular client with a client personalization layer OR have fully separate client bundles that share a lot of components and client infrastructure with their own top level app for maximum flexibility. Regardless, I can have all of them share the same server. Cloudfront can pass a unique identifier for each front end site. We can modify the server to take this site identifier. to multiplex the same server for multiple sites.

We need to figure out how to differentiate different clients in the database. One way would be to add a site identifier to every table. Another would be separate schemas. A more costly but fully isolated option would be multiple databases. Can you help me explore the various alternatives here to make a decision?

Can you start with brainstorming the high level work here before moving on to the actual implementation plan. I have a lot of places where we add an item and fetch the primary key. It's not clear to me now if the primary key for each table stays the same and we just ad

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here