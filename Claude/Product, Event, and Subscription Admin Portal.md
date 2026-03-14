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

I'd like a separate page or set of pages to be able to do things based on a new permission, manage_products, that admins have but can be granted to employees other than full admins to do things like create an event, product, or subscription. You should be able to perform operations like:

- Create / edit a price schedule
- Create / edit an event
- Create / edit a product
- Create / edit a subscription
- Bind events, products, and subscriptions to different price schedules and create prices per price schedule and permission. Ad admin should be able to configure if the price for various permissions including whether or not it is available to user's with no permission.
- The admin should be able to perform CRUD operations on permissions
- For subscriptions, the admin should be able to specify various permission that a given subscription grants.
- The admin UI should be able to do CRUD operations for the various locations / facilities / rooms
- The admin UI should be able to define that various events or services need a given room type
- The admin UI should be able to create instantiations of events with a given start / end time, number of seats, facilities, and the other properties of and event through a user friendly UI.
- The UI should be able to enumerate various event instances to see who is signed up, payments for the event, and how much space is in the event. 
- The UI should be entitlement aware and be able to navigate doing CRUD style operations on the entitlements associated for product creation
- The UI should be able to enumerate the various events, products, and subscriptions and see for each instance which entitlements are granted and to whom.

Please start by listing these requirements and help me brainstorm possible other ones and then we can work on design and implementation plan.

# Steps
- List of steps to accomplish this task.