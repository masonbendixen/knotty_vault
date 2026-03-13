---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/13/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

Look at the document: Subscriptions- Recurring billing and card management.md and the Scheduled Jobs section at the end of the document. I want to build a plan for a helper executable that does scheduled jobs and acts as a server watchdog. Please move the content from that document to here. Look at these documents:

- [[Payment Design Document]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]

And the code base for other ideas for things that need to be called from the helper process. For the watchdog process, I feel like there should be some kind of ping endpoint that this watchdog process calls every so often. If that doesn't return a response within a configurable interval, it should kill the webserver process and restart it. The name of the executable for the web process should be a configurable setting. The watchdog process should also spawn a separate instance that pings the first instance using some mechanism. If that doesn't return a response within a configurable interval, it should kill the other process, take over the watchdog responsibility, and kick off a new watchdog process to watch the watchdog. This should ideally be portable code that runs on windows and linux. Ideally we use the standard library and boost (or come up with other libraries on conan) to do this process orchestration support.

# Build plan here