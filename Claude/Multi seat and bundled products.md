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

For bundled products, there should be base products and add ons. Add on means a discount on the price of the add on product if purchased with the base product. For each add on, we should keep track of if the discount is stackable (ie. can there be multiple discounts on this product based on it being combined with other discounts). The default should be no for stackable. Let's add a spa entry one time product. We also need to add a room to the facility called "Spa" that is a sibling to main gym. We should have different variants of spa entry for different time lengths. We should also have different variants also for early bird (ie. certain days of the week for early, less popular hours). I would like a separate product that is "Late night, post workout spa access" that is a last hour thing on certain weeknights after the last workout class finished for muscle recovery. I would like users to be able to book these at five minute intervals during available blocks. We need to have a spa capacity that is tracked. We should probably make this a property of rooms since massage room should have capacity of 1 (unless we later create a room with two tables for couple massage at a later date), the main gym should have a max capacity, and so should the spa. We should allow people to do bookings at start intervals that they like as long as there is space in the gym for the whole interval time being booked (we should not show time slots that would cause a portion of the booking to exceed capacity). We need an entry in the staff portal to check in people for their spa visit. They are allowed to check in up to a product configurable window before their booked time (default to 15min). The duration of the visit is based on their checkin time. If checking in early would exceed spa capacity, they must wait until their is space. Checking in late does not extend their visit length since that would cause complications for capacity for already booked clients. People are allowed to book slots that would not allow the full time because the space will close before the whole time is passed. There will not be a discount or prorating but they will be warned. We also need to allow drop ins where the staff portal has the ability to create a booking on the fly if there is space available as well as creating an account if the person does not have one. This will involve collecting first and last name as well as email, creating the account, and automatically generating a password that is emailed to them that they should change immediately. We need a bundled option that can specify massage as the base product and spa entry as the add on. There will be a discount for the spa entry if purchased bundled with a massage. Also, the length of the spa visit is automatically extended by the length of the massage (which needs to be factored in for the checking of spa capacity during booking). Canceling either the base product or a bundled product cancels the whole booking and all components. Refunds are based on the refund policy for each product except if the provider cancels the massage, the WHOLE bundle is fully refunded regardless of individual refund policies. I'm not really sure about the best way to expose booking this to the user. Perhaps book the massage and have an Add Ons button that bring up a modal UI where you can see the add ons and then click on one like Spa Entry and then see if it available and have options of how far before the massage to start the spa entry (which has to start at or before the time of the massage). If a certain spa entry time would not allow full duration of the spa entry length because it runs into spa closure or a period of out of capacity, we could warn the user of such but allow them to still book it if they want to.

This is a pretty big item. Please use the code base and these documents for context:

- [[Nested item support]]
- [[Payment Design Document]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Product browsing and quoting endpoints]]
- [[Product, Event, and Subscription Admin Portal]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]
- [[Event Polish- Scheduling Should Have Items]]
- [[Bookable Service Foundation]]
- [[Provider Portal]]
- [[Vouchers and Refunds]]

Also, please create a section noting the changes we will need to make to [[Payment Design Document]] and [[Support for scheduled purchases]]. After we have locked down the design, the first phase of implementation should be to update those documents with the changes needed to support these features. Please start a discussion with me on ideas of how to expose choosing bundled products to link them and compatibly schedule them. Also, please critique what I have written, come up with other suggestions, and list possible other complimentary work that also seems related.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here