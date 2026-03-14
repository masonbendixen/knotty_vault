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

Please use the code base and these documents for context:

- [[Payment Design Document]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]

We currently have an admin portal that exposes select tables to the user via Manage Data. See [[Nested item support]] for more information. Although one can kind of navigate products, events, and subscriptions through the raw database tables, it is hardly intuitive.

I'd like a separate page or set of pages to be able to do things like create an event, product, or subscription. You should be able to perform operations like:

- Create / edit a price schedule
- Create / edit an event
- Create / edit a product
- Create / edit a subscription
- Bind events, products, and subscriptions to different price schedules and create prices per price schedule and permission. Ad
- For subscriptions

# Steps
- List of steps to accomplish this task.