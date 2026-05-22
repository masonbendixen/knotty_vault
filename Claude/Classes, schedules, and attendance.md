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

For series, events, and workshops, I would like a maximum number of people who can attend as well as an optional minimum number. For the minimum number, we should specify a date by which this number needs to be hit and a policy to specify if we should auto refund people's money and cancel the session if it did not hit that minimum number. For maximum numbers, we should piggy back on the waitlist mechanism used for events to allow a waitlist to get into the session.

I would like a schedule template that people can view the classes that are included in their membership that they can mark as their attendance template which are the classes that they plan to attend in general. I would like an email to go out at a fixed time each week (like Sunday at noon) that reminds people of their planned class attendances based on their template as well as classes, services, and events that they have signed up for. I would like people to see a calendar view of these things and be able to go through and mark classes that they are 

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here