---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/3/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

We had a previous work item, [[Payment Should Have- Multi Seat and Bundled Pricing]] to implement the Should Have section of [[Payment Design Document]]. Unfortunately, scenarios 6 and 7 were not completed as part of that item. I would like to tackle them here.

Let's add a couple's massage product with variants for 60, 90, and 120min. We should rename the membership sharing thing to purchase sharing in the user console but keep the functionality the same. Booking a couple's massage would be a new thing under bookable services. Bookable services currently has separate providers listed with their available timeslots. For couple's massage, we would have a   similar UI but it would have slots listed where there are at least two providers available at the same time. Upon choosing a slot in the UI, they would select the other person as part of the couple's massage and, for each person, choose a provider. Note that the assignment of seats should be like membership (ie. the person booking the two person service can pay for it but could do it for two other people like a person buying it for their parents). The purchase would be under the person paying but the booking would show up under the two people recieving the service. If either user or either therapist cancels, the whole thing is cancelled for both people scheduled and both providers. If either provider cancels, the whole thing is refunded.

For bundled products, there should be base products and add ons. Add on means a discount on the price of the add on product if purchased with the base product. For each add on, we should keep track of if the discount is stackable (ie. can there be multiple discounts on this product based on it being combined with other discounts). The default should be no for stackable. Let's add a spa entry one time product. We also need to add a room to the facility called "Spa" that is a sibling to main gym. We should have different variants of spa entry for different time lengths. We should also have different variants also for early bird (ie. certain days of the week for early, less popular hours). I would like a separate product that is "Late night, post workout spa access" that is a last hour thing on certain weeknights after the last workout class finished for muscle recovery. I would like users to be able to book these at five minute intervals during available blocks. We need to have a spa capacity that is tracked. We should probably make this a property of rooms since massage room should have capacity of 1 (unless we later create a room with two tables for couple massage at a later date), the main gym should have a max capacity, and so should the spa. We should allow people to do bookings at intervals that they li

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here