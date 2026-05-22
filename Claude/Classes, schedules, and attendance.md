---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/22/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I offer a set of classes at my studio. A class has a name, description, and a photo. At a given facility, there is a class schedule. This binds specific classes to being taught at a given facility on given days at given times by given instructors. For each scheduled class instance, there is information for which memberships are allowed to attend and to sign up for and attend specific classes or if people without memberships are allowed to attend. There are classes which are available to attend as part of a membership as included at no extra cost. Some classes are not included with a membership but can be signed up for by people with given memberships at membership specific prices or they might even allow non members to attend at a different price. Note that when I say membership, the real gate is a permission that is granted by the membership tier.

Some classes will be part of a series. The series will have a start and end date as well as a number of instances (like every tuesday 8pm-9pm between the given start and end date). For these series, it will be specified which memberships are allowed to sign up (and if non members can sign up at all) and what the price is for the whole series and if pro-rated signups are allowed for an in session series. For each of these series there is a base price per singular class instance that varies based on membership tier that is used to calculate the sign up price for a the series as a whole (and also used for pro-rating). I have staff that are just on the schedule and then I have specialty people who will be paid just to teach a given class, workshop, or event. The specialty people incur a cost that I need to recoup. For things taught by the specialty instructors, I'm not trying to make a profit off of membership students since I'm already getting their membership fee but I do want to recoup my cost for the specialty instructor. For non-members or even lower tier membership students to a degree, I'm looking at making a profit in addition to recouping my instructor cost.

I would also like to introduce skill levels (like can invert in the air or kick up to a handstand against a wall). Skill levels will have a name, description, and photograph. There will be bindings to skill levels that a person has been validated by staff as achieving and a class can specify skill levels as a requirement to sign up for / attend. We need a staff portal entry for staff to assign skill levels to a person as well as a user portal entry for people to view their skill levels.

For series, events, and workshops, I would like a maximum number of people who can attend as well as an optional minimum number. For the minimum number, we should specify a date by which this number needs to be hit and a policy to specify if we should auto refund people's money and cancel the session if it did not hit that minimum number by that date. For maximum numbers, we should piggy back on the existing waitlist mechanism used for events to allow a waitlist to get into the session. In general, we should use as much of the existing infrastructure for events and services as we can.

I would like a schedule template that people can view the classes that are included in their membership that they can mark as their attendance template which are the classes that they plan to attend in general as their regularly planned weekly fitness routine. I would like an email to go out at a fixed time each week (like Sunday at noon) that reminds people of their planned class attendances based on their template as well as classes, services, and events that they have signed up for. The email should also have iCal attachments for the sessions listed. I would like people to see a calendar view of these things and be able to go through and mark classes that they are are eligible to attend that they haven't marked on their template as well as being able to note classes that they normally plan to attend on their template that they won't be able to make this instance.  For class marking template exceptions, we should have an optional note to that goes to the instructor (like people letting the instructor know they are on vacation). Needless to see, people should be able to edit their templates. On creation or edit of a template, an email should be sent for each new class signed up for with an iCal attachment for the session(s) with the appropriate recurrence. For the classes a student is eligible to attend on that given day, their homepage should show the classes that they have indicated they are going to attend (with checks next to them) and the classes they are eligible to attend but haven't indicated that they are going to attend (without check boxes next to them) to allow them to quickly see and modify these choices.

Staff should be able to check in people for a given class. We should do autocomplete and have a list of clickable people to mark as attending based on people marking their expected attendance as well as prior history of class attendance for this class instance over the previous four weeks. We should have a configurable time window for which class check-in is allowed in advance and post class end (I would suggest that this default to one hour before and three hours after). Check-in for class in this window should show up on their home page. In the user portal, the user should be able to view their past attendances in a paginated view and be able to filter by year / month / class name / instructor. Note that students indicate planned attendance but staff is the only one who can mark actual attendance.

We should have scheduling exceptions. There should be studio closure instances as well as schedule exceptions for a given class. Also, classes are assigned to be taught be a given teacher but we need a way to handle instructors having sick days or vacation and marking a class instance as being cancelled or taught by someone else. We need UI in the admin portal to manage these scheduling exceptions. Also, we should have sign up windows for class series and have per membership / permission sign up windows (for instance, platinum can sign up 56 days in advance, gold 42 days, silver 35, and non-members 21). We should reuse the same UI / code as we do for allowing people to sign up for massage in advance. People should be able to see when they will be able to sign up for upcoming sessions and even be able to click to receive an email on the day for which they can sign up for a given class / session.

Staff should be able to do shift trades / transfers like service providers and we should reuse as much of that infrastructure as possible. We need this in the portal as well as an admin view of who is teaching what. Also, students should be able to click on instances in the calendar and see who is teaching a given class. Unlike massage, a change in instructor will not result in refund capability. An admin should be able to cancel a given instance of a class which results in email going out to those who have marked themselves as attending as well as a pro-rated refund for people who had a cost for attending that class (i.e.. if a class was just included with a membership, there is no refund for a cancelled instance).

Long term, it would be nice to see people who routinely indicate that they are coming to things but then don't attend. As we get to where attendance caps are real, this could become a real probably that we want to have some mitigation / penalty for.

Long term, we will also need to keep track of specialty instructors and note their rate for teaching a class as well as possibly bonuses per student or possibly bonuses per student past a certain attendance target. They might also have personal minimum / maximum numbers that would be nice to be able to configure per class type (for instance someone might be willing to take more people in a handstand class than an aerial class).

I would like to build a document with a list of use cases, group the use cases by category, suggest other use cases, and then work towards bucketing them into must have, should have, nice to have, could have, and stretch. From these buckets, I'll create separate implementation documents to complete individual buckets.

Please do outside research and the code base to build your plan as well as these documents:

- [[Payment Design Document]]
- [[Product browsing and quoting endpoints]]
- [[Purchase creation with server-side pricing]]
- [[Scheduling thin slice]]
- [[Square credentials and Sandbox setup]]
- [[Subscriptions- Recurring billing and card management]]
- [[Support for scheduled purchases]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Event Polish- Scheduling Should Have Items]]
- [[Multi seat and bundled products]]
- [[Payment Should Have- Multi Seat and Bundled Pricing]]
- [[Product, Event, and Subscription Admin Portal]]
- [[Provider Portal]]
- [[Scheduled Jobs]]
- 

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here