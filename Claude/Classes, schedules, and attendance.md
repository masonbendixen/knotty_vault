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
- [[Vouchers and Refunds]]

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here

# 1. Context Recap: What Already Exists

Before listing use cases, here is what the codebase already provides — the plan reuses these aggressively rather than rebuilding.

**Catalog & pricing (Payment Design + Multi-seat docs):**
- `products`, `product_variants`, `price_schedules`, `product_prices` (per-permission tiered pricing already supported)
- `product_entitlement_rules` (entitlement validity + `grants_permission_id` — this is the "membership tier grants permission" mechanism)
- `purchases`, `purchase_items`, `payments`, `purchase_payments`
- `entitlements`, `entitlement_assignments` (multi-seat, gift permissions)
- `subscriptions`, `subscription_charges`, `saved_cards` (recurring memberships, grace periods, retries)

**Scheduling (Scheduling thin slice + Event Polish + Provider Portal docs):**
- `facilities`, `location_room_types`, `location_rooms` (capacity, timezone)
- `event_sessions` (capacity, booked_count, status, visibility flags, room) — the row class occurrences persist into (lazily, only when something is recorded; see redesign)
- `bookings` (status: confirmed/waitlisted/cancelled/attended/no_show, `waitlist_position`, `checked_in_us`, `is_walkin`, `free_cancel_until_us`, `reminder_sent_us`)
- `event_session_staffing` (multi-staff per session with role)
- `cancellation_policies`, `cancellation_policy_windows`
- `provider_availability`, `time_off_requests`, `shift_change_requests` (Provider Portal — already supports shift trades, time off, admin approval when bookings affected)
- `schedule_templates`, `schedule_template_entries` (recurring weekly availability generation)
- `scheduling_exceptions` (studio-wide closures + per-provider blackouts with cascade)
- `product_booking_windows` (per-permission `advance_days` — exactly the "platinum 56, gold 42, silver 35, non-member 21" pattern)
- `RecurringSessionHelper::CreateRecurringSessions` (weekly/biweekly/custom pattern → batch `event_sessions`)
- `BookingHelper` (book, cancel, waitlist auto-promote, refund computation)
- `SessionCancellationHelper` (admin cancels session → 100% refunds + emails)
- `EventReminderHelper` (async reminder job)
- `ShiftChangeHelper` (trades, transfers, admin review when bookings exist, `free_cancel_until_us` set for affected attendees)

**Cross-cutting:**
- Photo infrastructure (`photo_support_tables` whitelist, `table_item_photos`, scaled photo cache + reaper, `photo-upload` Angular component, admin console thumbnail support) — `classes` is already in the photo whitelist
- `Mail::MailAttachment` (filename + content + contentType) — already used for `.ics` attachments
- **iCal generator** (`util/ical_generator.h/cpp`) — `ICalGenerator::GenerateICalendar(const ICalEvent&)` returns RFC 5545 `VCALENDAR` text for a single event. Struct fields: `title`, `startTimeUs`, `endTimeUs`, `timezone`, `location`, `description`. Text escaping per RFC 5545 (commas, semicolons, newlines), CRLF line endings, UTC `Z` form for `DTSTART`/`DTEND`. Already wired into `book_event`, `book_service`, `cart_checkout`, `payment_helper`, `staff_upgrade_session` to attach `.ics` to confirmation emails. Tests in `util/ical_generator_test.cpp`. **What it does NOT yet have (extensions required for templates / digest / cancellations):** `UID` field (RFC 5545 requires it; calendar apps de-dupe and update by UID), `RRULE` for recurring entries, multi-VEVENT bundle in a single VCALENDAR, `STATUS:CANCELLED` flag, `ORGANIZER` / `ATTENDEE` fields, and `VTIMEZONE` block (the `timezone` struct field is currently unused — output is always UTC, which is fine for one-offs but lossy for recurring entries that cross DST boundaries).
- `ThreadPool::QueuePeriodic` (Phase I in progress) and `knottyyoga_helper` (scheduled jobs daemon) — both available for the Sunday-noon weekly digest email
- Permissions / roles / `ActiveUserHasPermission` (RBAC) and entitlement-derived permissions
- **Permission-based class access (built 2026-05, see [[Permission-based class access redesign]]):** `permission_implications` (closure-expanded hierarchy so platinum ⇒ gold ⇒ …), `class_requirement_groups` + `class_requirement_group_literals` (CNF access rule keyed on `classes`), `booking_requirement_overrides` (staff-override audit), `Scheduling::ClassAccessHelper` (the one access gate), and `CatalogHelper::GetEffectivePermissionIds` (closure-expanded union of role/membership/attendance/skill permissions). This **replaced** the Phase-2 binary `products.is_membership_included` flag. *(Authoring UI for requirement groups is still pending — a Phase 1 follow-up, see §4.2 of the redesign.)*
- `admin_top_level_tables` / `admin_nested_tables` / `admin_table_permissions` / friendly names / display templates infrastructure for generic CRUD

> **Schedule model redesign (2026-05-28):** the class-schedule data model was redesigned after the first Phase 1 draft. See [[Class Schedule Implementations Redesign]] for the full design and locked decisions. Summary: a three-level hierarchy — `classes` (marketing identity + `kind` enum) → `class_instances` (a run / unit of purchase, carries the `product_id`) → `class_schedules` (versioned implementations with priority + validity window, scoped to one instance) → `class_schedule_slots` (recurring day/time/room/instructor tuples). Sessions are **derived on the fly** from the active instance + active impl + slots (lazy instantiation — no pre-materialization); `event_sessions` rows persist only when something is recorded about a specific occurrence (cancellation, instructor sub, note, paid booking, check-in, no-show). Holidays / closures / schedule changes are expressed as higher-priority impls, not a separate "exceptions" mechanism. The gaps and use cases below are written against this model.

**Notable gaps the plan must close:**
- No `class_instances` / `class_schedules` / `class_schedule_slots` concept yet — `event_sessions` exist but aren't tied to a versioned "class on Tuesday 6pm taught by Sara, under the May 2026 run" parent chain
- No skill levels concept (table, assignments, requirements, validation by staff)
- No attendance templates (planned-attendance-by-class for the week, with per-instance exceptions and instructor notes)
- No "drop-in price varies by membership tier" wired in for class instances (the table machinery exists; class instances need to know to use it)
- No "min attendees by date → auto cancel + refund" policy
- No "email me when sign-ups open" reminder system
- iCal generator exists but lacks `UID`, `RRULE`, multi-VEVENT, `STATUS:CANCELLED`, `VTIMEZONE` — extensions needed for attendance templates (recurring), weekly digest (bundle of events), and cancellation-syncs-to-calendar
- No "no-show penalty / reliability score" infrastructure
- No specialty-instructor cost tracking / payroll model

---

# 2. Use Case Catalog

Use cases drawn from the Overview, plus suggestions in §3. IDs are stable handles for the bucketing in §4. Categories group conceptually related items; ordering inside a category is not priority.

## 2.1 Class Catalog & Discovery
- **C-1** Admin creates a class definition: name, description, photo, default capacity, default cancellation policy, default room type, **access permission set** (the memberships/permissions for which the class is included — declared via the class's requirement groups, NOT a binary "included-with-membership" flag; see [[Permission-based class access redesign]]), skill-level requirements, list of allowed booking permissions for the non-included (paid) flavor
- **C-2** Admin edits / soft-deletes / re-photographs a class
- **C-3** Anonymous visitor browses the class catalog (cards w/ photo + description)
- **C-4** Logged-in member sees which classes are included in their membership and which require additional payment
- **C-5** Public class detail page (description, photo, sample times, instructors who typically teach it, skill-level requirements)
- **C-6** Filter / search class catalog by category, skill level, instructor
- **C-7** Class category / tag taxonomy stored as a controlled vocabulary — admin CRUDs a `class_tags` table (one row per tag: id, name, optional color); classes link to tags via a `class_tag_assignments` join table referencing the tag's primary key. No free-form tag strings on classes. Examples: "vinyasa", "aerial", "handstand", "partner-acro". Drives catalog filter, calendar color-coding, and the monthly-attendance prerequisite in §2.6 (SL-10)
- **C-8** Extended instructor profile pages: upcoming sessions, bio, photo; linked from each class and from calendar instances

## 2.2 Class Schedule Authoring
> Reframed for the three-level redesign (see [[Class Schedule Implementations Redesign]]). "Schedule entry" → "implementation (impl) under an instance"; "materialize" → "derive on the fly".
- **CS-1** Admin creates a class instance (a run) and one or more implementations under it. Each impl has a priority + validity window and a set of slots; each slot is a (day-of-week + start time + duration + facility + room + instructor) tuple. Same class can run multiple slots on the same day (morning + evening) and different times on different days
- **CS-2** No batch-materialization step — sessions are derived on the fly from the active instance + active impl + slots for any date range (lazy instantiation, per OQ-CSI-12). `event_sessions` rows persist only on a recording trigger (cancel / sub / note / paid booking / check-in / no-show)
- **CS-3** Admin edits an impl (slots, window, priority). The change takes effect immediately for all future derived sessions; the impl-save sweep deletes stale future-date admin-only rows scoped to that instance, and refuses if any orphan carries a `purchase_id` (OQ-CSI-19)
- **CS-4** Admin deactivates an impl or instance (soft-delete `is_active=false`); future derivation stops using it. Cancelling already-purchased sessions remains an explicit admin action (refund-bearing)
- **CS-5** Admin views the master class schedule grid (week or month view) per facility, computed from the derived-session view
- **CS-6** Derivation is inherently idempotent (no rows to double-create) and never blows away booked sessions — a higher-priority impl wins the lookup for future dates but persisted paid bookings stay put and surface for explicit admin reconciliation
- **CS-7** Per-instance class notes: the class's default description is preserved; in addition to it, the assigned instructor can attach a per-session note for sessions they teach ("today we'll work on forward compression and abdominals"), and staff with the right permission can attach a per-session note for any session. Attaching a note is one of the recording triggers that persists an `event_sessions` row. Notes are surfaced on the user homepage's today-classes feed (alongside the default description) as a teaser to drive attendance — but only for classes the user has on their attendance template or has booked, NOT for browsable-but-unclaimed eligible classes
- **CS-8** Multi-occupancy rooms: derivation + booking permit parallel sessions in the same room as long as combined occupancy ≤ `location_rooms.concurrent_capacity` (e.g. open gym + skill-development workshop running simultaneously). The room-conflict check sums derived + persisted overlapping sessions. Hard block only when capacity would overflow
- **CS-9** Admin previews "what's the active schedule on date X" for a class — resolves the active instance, the active impl under it (priority-resolved), and the slot list for that day. Backs the admin calendar-day preview

## 2.3 Class Series & Workshops (paid bundle of instances)
- **CSer-1** Admin creates a series: parent class, start date, end date, recurrence (day(s) + time), max attendees, optional min attendees, min-by date, min-not-met policy (auto-cancel + refund | proceed | admin decides)
- **CSer-2** Admin sets per-permission series pricing (platinum free, gold $X, silver $Y, non-member $Z)
- **CSer-3** Admin sets per-instance base price (used for pro-rating partial series sign-ups), per permission
- **CSer-4** Admin enables / disables pro-rated sign-ups mid-series
- **CSer-5** User views available series in their booking window, sees their tier price
- **CSer-6** User buys a full series, gets an entitlement covering all derived occurrences in the series run + auto-booked into each (occurrence `event_sessions` rows ensured at purchase)
- **CSer-7** User joins mid-series with pro-rated pricing computed from remaining sessions
- **CSer-8** Auto-job at min-by date: if confirmed count < min and policy = auto-cancel → cancel series, refund all, email all
- **CSer-9** Workshop = a one-off "series" of length 1 (same code path, but UI may call it "workshop")
- **CSer-10** Series booking is one purchase; cancelling the series cancels all child instance bookings together

## 2.4 Membership-Gated Access & Pricing
> Inclusion is **per-permission, not a boolean** — see [[Permission-based class access redesign]]. A class accepts a *set* of permissions (M-1's "memberships X, Y, Z"); memberships, attendance thresholds (SL-10), skills, and roles all grant permissions into one access gate (SL-12). A Phase-2 binary `is_membership_included` flag was a mistake and is being removed.
- **M-1** Admin marks a class as "included in memberships X, Y, Z" — members with those permissions can mark template and attend for free. This is a *set* of membership permissions, realized via the class's **access requirement groups** (closure-expanded so a higher tier satisfies a lower-tier requirement) — explicitly **not** a binary `is_membership_included` flag (see [[Permission-based class access redesign]]). "Included for this viewer" is derived at read time: a recurring class whose requirement groups the viewer's effective permissions satisfy
- **M-2** Per-tier pricing applies to workshops and series only (NOT individual recurring class instances — per P-1 there is no drop-in pricing for recurring classes); admin sets per-permission price including the non-member tier where the offering is open to non-members
- **M-3** Admin sets per-permission visibility (e.g. private classes visible only to platinum members)
- **M-4** User browses catalog and sees included classes (their tier) plus workshop / series offerings at their tier price
- **M-5** User with multiple permissions gets the best (lowest) price they qualify for, automatically server-side
- **M-6** Non-members see only the offerings open to non-members (workshops, series, the intro workshop — M-12) at the non-member price; recurring classes are not browsable / bookable to non-members at all
- **M-7** Membership-included class attendance is recorded via a `booking` row only — NO purchase row, no `purchase_id` tied to the booking. Bookings are an attendance / metrics record, not a money trail. The `bookings.purchase_id` column is nullable for class bookings under an active membership entitlement
- **M-8** Membership tier upgrade unlocks newly-eligible classes in real time (no manual rebooking)
- **M-9** Guest pass: an active member can book a non-member friend into a single class designated by admin as guest-pass eligible (allow-list per `classes` row, default off). Redemption may auto-create a minimal guest account on the spot, no pre-existing `gift_permissions` relationship required. Guest-pass frequency is configured per membership tier — different tiers can have different guest-pass allowances (fitness memberships typically frequent / marketing-oriented; spa memberships sparse / luxury perk). Per-tier configuration also includes an optional "guest must be a new person" flag that prevents repeatedly using the same friend
- **M-10** Effective-dated price changes via `price_schedules`: admin sets "starting July 1 the gold-tier workshop price becomes $X" without disrupting in-flight series purchases or subscription periods. All tiered pricing — class series, workshops, couple/family memberships, specialty-instructor rates (SI-6) — flows through this same mechanism
- **M-11** Couple / family membership tier: a single subscription covers multiple specified people via multi-seat entitlement (`seats_total > 1`). The studio already offers a **couple membership** (`seats_total = 2`) and a **family membership** for three (`seats_total = 3`) — these are existing products that must fit the model cleanly. No need to support `seats_total ≥ 4` (fifteen years of operation with no demand for it, and kids classes are explicitly out of scope). Presented in the UI as one product, not "buy two memberships". Per-tier pricing via `product_prices`
- **M-12** Intro workshop as the non-member discovery event: a `class_schedule` of length 1 (per P-3) sold to non-members. Functions as a one-time sample class + sales pitch; attending the intro workshop grants no entitlement, no class access, no permission. Non-members who want to attend recurring classes must purchase a membership afterward. The intro workshop is the only recurring-class-like offering visible to non-members beyond workshops and series
- **M-13** Per-session price override: admin sets a higher or lower price for a single class series / workshop instance (special guest teacher week, holiday discount, sliding scale). Override is tied to the specific session, not the schedule
- **M-14** Partner / friend booking: an active member books an additional adult attendee (partner, friend) for a session using a multi-seat entitlement. Explicitly NOT for children — no kids classes will ever be offered

## 2.5 Specialty Instructor Cost Recoupment
- **SI-1** Admin records a specialty-instructor cost at the class-schedule / series level (NOT per-instance) — flat rate, per-student bonus, per-student-past-target bonus
- **SI-6** Specialty-instructor compensation rates flow through the same `price_schedules` infrastructure as everything else (per P-2): a rate change rolls forward via a new `price_schedule` row without rewriting in-flight series sessions
- **SI-2** Admin runs a pricing assistant that suggests per-tier prices to (a) cover specialty cost on a break-even count and (b) profit above that count for non-members / lower tiers
- **SI-3** Specialty instructor has personal min / max per class type (handstand max 8, aerial max 6)
- **SI-4** Admin report: per-class / per-series cost vs. revenue (instructor pay + bonus thresholds vs. attendee revenue split by tier)
- **SI-5** Payroll export: list of specialty instructors with classes taught + hours + computed pay for a period

## 2.6 Skill Levels
- **SL-1** Admin creates / edits / soft-deletes a skill level: name, description, photo
- **SL-2** Staff assigns a skill level to a person (records who assigned, when, and an optional note)
- **SL-3** Staff revokes a skill-level assignment (records who revoked, when, and reason)
- **SL-4** User views their own skill levels in the user portal (cards w/ photo + description + earned date)
- **SL-5** Admin sets one or more skill levels as required for a class
- **SL-6** Booking attempt for a class with skill-level requirements is rejected unless user has all required skills (or staff override)
- **SL-7** Class detail page shows skill-level requirements prominently with "you have / don't have"
- **SL-8** Staff portal: search a person, view their current skill levels, assign new ones, revoke existing ones
- **SL-9** History of skill-level changes per person (audit trail)
- **SL-10** Calendar-month attendance threshold → auto-granted permission. A class can require a specific permission (e.g. `acro_club`) that is granted by a scheduled job: at the end of each calendar month, the job counts attendances against a configured set of source classes (e.g. all `partner-acro`-tagged classes); if the count meets the configured threshold for a user, the job grants that user the destination permission for the *current and next* calendar month. If the user doesn't meet the threshold again the following month, the permission auto-expires. Implementation = `attendance_threshold_rules` table (source-class-set, threshold, granted_permission_id) + monthly scheduled job + standard permission gate at booking time. No rule-engine DSL needed — pure SQL plus the job
- **SL-11** Same-day sequencing prerequisite via a `predecessor_class_schedule_slot_id` field on `class_schedule_slots` (per the redesign — predecessor is a slot-to-slot relationship, not schedule-to-schedule). When the user attempts to book a class occurrence whose slot has `predecessor_class_schedule_slot_id` set, the server verifies the user is also booked for the predecessor slot's occurrence on the same day. If the user cancels the predecessor booking, the dependent booking is auto-cancelled atomically in `BookingHelper::CancelBooking` (NO email — silent cascade)
- **SL-12** The three prerequisite mechanisms — skill mastery (SL-5/6), monthly-attendance permissions (SL-10), and same-day predecessor (SL-11) — all compose at the standard booking-permission gate. There is no separate "rule engine"; each mechanism is just a SQL check (skill check / permission check / same-day-booking check) wired into `BookingHelper`. Staff with `manage_classes` permission can override any of them with a logged reason. P-5 ensures the attendance counts that feed SL-10 are trustworthy because users cannot self-attribute attendance

## 2.7 Capacity, Waitlist, and Min Attendees
- **CAP-1** Each class instance has a max capacity from the class default or schedule override
- **CAP-2** When capacity is hit, further bookings go to a waitlist (existing `BookingHelper` waitlist)
- **CAP-3** When a confirmed attendee cancels, top of waitlist auto-promoted with notification (existing)
- **CAP-4** Series / workshop has optional min attendees + min-by date + min-not-met policy
- **CAP-5** Auto-cancellation job runs daily, scans series/workshops past their min-by date, cancels if under min, refunds 100%, emails all
- **CAP-6** Admin can override (e.g. "we're at 4/5, run anyway")
- **CAP-7** Capacity is enforced against the room's `concurrent_capacity` AND the session's `capacity` (the more restrictive wins); per CS-8, multiple sessions in the same room share the room's `concurrent_capacity` budget
- **CAP-8** Waitlist auto-confirm cap (user preference): "auto-confirm me if a spot opens within N hours of class start, otherwise drop me from the waitlist". Lets users avoid last-minute scrambles to a class they no longer plan to attend

## 2.8 Attendance Templates (Planned Recurring Attendance)

The template is a **personal fitness-planning tool**, not a booking. It captures the user's aspirational weekly routine ("I plan to work out Monday / Thursday / Saturday on a good week"). It does NOT create `booking` rows, does NOT consume `event_sessions.capacity`, and does NOT participate in the waitlist. The studio is membership-based — the goal is that people just show up when they can. If capacity issues ever force template entries to translate into hard bookings, that's a future "good problem to have" — punt until then.

- **AT-1** User views a weekly grid of classes they are eligible to attend, marks the ones they normally attend → that becomes their attendance template
- **AT-2** Template is per-slot (recurring) and forward-looking — it applies to all future derived occurrences of the marked slot, without creating any booking or `event_sessions` rows
- **AT-3** User edits / removes template entries freely — no bookings to sync, no cancellation cascade, just an update to the template entry table
- **AT-4** Adding a template entry sends a confirmation email with a recurring iCal attachment — a single `VEVENT` with an `RRULE` covering the future occurrences of the schedule (e.g. `FREQ=WEEKLY;BYDAY=TU;UNTIL=...`), so the user's calendar app expands the recurrence locally. Per-instance exceptions (AT-5) are emitted as `EXDATE` lines, not separate events
- **AT-5** Per-instance exception: user marks "won't attend this Tuesday" with an optional note to the instructor (e.g. "on vacation") — affects what shows on the homepage and weekly digest, does not affect any booking (none exist)
- **AT-6** Per-instance addition: user marks "will attend this Thursday in addition to my template" — same model as AT-5, just inverted (an extra-this-week marker)
- **AT-7** User homepage shows today's eligible classes with checkmarks for those on their template (and for one-off additions) and unchecked rows for eligible classes they haven't claimed; one click flips state
- **AT-8** Capacity accounting under M-7: bookings only get created at staff check-in (the user is actually present). Template entries never block capacity for anyone else
- **AT-9** Calendar view shows template attendance overlaid on the studio schedule
- **AT-10** Per-instance exception note is visible to the assigned instructor in their staff portal (per N-7, also delivered as a daily digest email of fresh exception notes)

## 2.9 Weekly Digest Email
- **WD-1** Sunday at noon (configurable wall clock + timezone per facility), a cron job sends each user a "this week" email
- **WD-2** Email lists each scheduled session (template-attended classes, one-off additions, paid bookings, services, events) for the upcoming week
- **WD-3** Email carries one combined `.ics` attachment with a separate `VEVENT` per session (NOT a single `RRULE`) — the digest is already filtered for this week's exceptions, so per-occurrence VEVENTs are simpler and exception-accurate
- **WD-4** User can disable weekly digest in notification preferences (new minimal preference table)
- **WD-5** Digest respects per-instance exceptions (skipped weeks don't appear)
- **WD-6** Per-user subscribable iCal feed URL: a personal `webcal://` URL the user adds to their calendar app once and it auto-updates as bookings change. Separate channel from the per-booking attachments — eliminates the "I missed the email" failure mode. URL is authenticated by an unguessable random token (hashed at rest; user can regenerate via portal). The feed sets `X-PUBLISHED-TTL:PT1H` to hint a 1-hour refresh cadence to subscribing calendar apps

## 2.10 Check-in (Attendance Marking)
- **CI-1** Staff opens check-in screen for a session; configurable window (default −1h, +3h) controls when it's accessible
- **CI-2** Check-in screen shows pre-populated list: template attendees + paid bookings + people who attended this class instance over the last 4 weeks
- **CI-3** Staff types a name to autocomplete and add an attendee not in the pre-populated list
- **CI-4** Staff marks attendance with one click. For membership-included recurring classes there is no advance booking (AT-8), so the check-in *creates* the `booking` row with `checked_in_us` set and `purchase_id IS NULL`. For paid bookings (workshops, series, intro, guest passes), the booking already exists and check-in just sets `checked_in_us` on it
- **CI-5** Walk-in flow: type a name → if no person, create on the spot (existing pattern); record as `is_walkin = true`
- **CI-7** Staff can mark `attended` / `no_show` after class has ended; for paid bookings, defaults to `no_show` if not checked in within window (existing `EventReminderHelper`-style hourly job)
- **CI-8** Staff can undo a check-in within the configurable post-window
- **CI-9** Configurable secrets: `class_checkin_window_before_minutes` (60), `class_checkin_window_after_minutes` (180)

## 2.11 Past Attendance & History
- **PA-1** User views paginated past attendances in user portal
- **PA-2** Filter by year, month, class name, instructor; combine filters
- **PA-3** Each row shows class, date/time, facility, room, instructor, status (attended / no_show / cancelled)
- **PA-4** Export CSV (nice to have)

## 2.12 Scheduling Exceptions & Instructor Substitutions
> Per the redesign (see [[Class Schedule Implementations Redesign]] §1.7, L-8/L-9): per-class schedule *exceptions* are no longer a distinct mechanism — they're expressed as higher-priority implementations (a "Memorial Day" impl with the Monday slot removed, a "Holiday Week" impl, an empty impl = closure). There is no global "studio closed" lever; even on a closure day a workshop may run under its own class. The use cases below are reframed accordingly; SE-2/3/4/5/6 (single-instance cancel, instructor sub, time-off, refund-on-cancel) remain real per-occurrence operations.
- **SE-1** Admin closes one or more recurring classes for a date range by creating an empty high-priority impl under each affected class's active instance. A Phase 10 batch UI ("close these N classes for this window", class-multiselect) makes this one action. Workshops / series scheduled in the window are unaffected (they live under their own classes)
- **SE-2** Admin marks a single class occurrence as cancelled (`SessionCancellationHelper` — this is a recording trigger that persists the `event_sessions` row with `status='cancelled'`)
- **SE-3** Admin substitutes the instructor for a single class instance (no refund) — uses existing `event_session_staffing`, updates booking emails informing of substitute
- **SE-4** Instructor reports they're sick / on vacation → time-off request flow (existing Provider Portal) → admin reviews → triggers substitute search or cancellation
- **SE-5** Cancellation of a class instance triggers: email to confirmed attendees + pro-rated refund (only for paid bookings, no refund for included-with-membership)
- **SE-6** Admin sees a list of pending substitution needs (sessions with instructor on time-off but no replacement)

## 2.13 Advance Sign-up Windows & Reminders
- **AW-1** Admin sets per-permission advance booking days at the product level (use existing `product_booking_windows`)
- **AW-2** User sees future class sessions including ones they can't yet book, with a "you can book this on `<date>`" hint
- **AW-3** User clicks "remind me on opening day" → row added to `signup_open_reminders` table
- **AW-4** Daily cron checks `signup_open_reminders`, emails users whose signup window has just opened for upcoming sessions
- **AW-5** Reminder de-dupes: once a user has booked, drop their pending reminders for that session

## 2.14 Shift Trades / Transfers (Instructor → Instructor)
- **ST-1** Instructor opens "request transfer" or "request trade" for one of their assigned class instances (reuse Provider Portal `shift_change_requests`)
- **ST-2** Target instructor accepts / rejects; admin reviews if bookings present (existing flow)
- **ST-3** Approval reassigns the instructor on the `event_session_staffing` row (NOT `provider_person_id` on a service session — class instances use the staffing table)
- **ST-4** Attendees of affected classes are NOT emailed about the instructor change (avoid excessive email). The new instructor is shown on the user's homepage today-classes feed and on the calendar — that's the channel of communication. **No refund** either (per Overview, instructor change ≠ refundable, unlike services)
- **ST-5** No `free_cancel_until_us` extension for class instructor changes (different from services)
- **ST-6** Admin view of "who is teaching what when" across the studio

## 2.15 Booking & Cancellation Mechanics
- **BC-1** User books a class instance — server verifies their tier includes the class (per P-1, no drop-in pricing); creates a $0 purchase + booking tied to their active membership entitlement; immediate confirmation
- **BC-2** User cancels a class / series / workshop booking — capacity is freed and the waitlist is advanced; no money is refunded automatically (per P-6, all class bookings are officially non-refundable from the user side)
- **BC-3** Admin cancels a class instance — studio-initiated cancellation; full refund issued for any paid bookings (workshops, series instances), capacity freed for membership-included bookings, email sent to all attendees
- **BC-4** Refund mechanism for admin-initiated cancellations uses existing `cancellation_policies` + `cancellation_policy_windows` + `RefundHelper`
- **BC-5** Cancellation policy displayed on the booking screen before the user clicks book — surfaces the refund breakdown ("48h: 100%, 24h: 50%, then no refund") OR an explicit "no refund" notice (per P-6 this is the default for classes), so users are not surprised
- **BC-6** Staff-issued voucher tool: staff issues a partial-credit voucher tied to (person, product / series, amount) as a discretionary alternative to a refund (reuses voucher infrastructure from `Vouchers and Refunds.md`). Used for mid-series cancellations, legitimate emergencies, and other case-by-case grants under P-6

## 2.16 Notifications & Communication (cross-cutting)
- **N-1** Booking confirmation email with iCal attachment for the session
- **N-2** Cancellation email with refund details
- **N-3** Waitlist promotion email
- **N-4** Series auto-cancellation email (min not met)
- **N-5** Instructor substitution is NOT emailed — surfaced on the user's homepage / calendar only (ST-4)
- **N-6** Weekly Sunday digest (§2.9)
- **N-7** Per-instance exception notes are visible to the assigned instructor in their staff portal, AND delivered as a daily digest email of fresh notes received in the last 24h (avoids per-note emails which would spam, but ensures instructors see the notes without having to log in)
- **N-8** Reminder email N hours before class (existing `EventReminderHelper`)
- **N-9** "Sign-up window opens" reminder (§2.13)
- **N-10** Minimal user notification preferences table for opt-outs
- **N-11** Favorite-instructor notifications (very low priority): user marks favorite instructors; system notifies on new classes / subs by a favorite

## 2.17 Reliability Tracking (Long-term)
- **R-1** Track per-user "indicated attending but didn't attend" rate over rolling window
- **R-2** Surface this in admin user-detail page
- **R-3** Configurable threshold above which user gets a soft email warning
- **R-4** Above harder threshold, optionally restrict ability to template-claim future capacity (Open Q on policy)
- **R-5** Class-level cap on consecutive no-shows: auto-suspend booking after N consecutive no-shows until the user contacts staff (low priority; only relevant once attendance caps actually bite)

## 2.18 Specialty Instructor Payroll (Long-term)
- **PR-1** `specialty_instructor_rates` table per (instructor, class type) with base rate + per-student bonus + per-student-past-target bonus + personal min / max attendance overrides
- **PR-2** Per-occurrence rate snapshot captured when the occurrence `event_sessions` row is ensured
- **PR-3** Payroll report: list of instructors, sessions taught in a period, attendees, computed pay
- **PR-4** CSV export for accounting

## 2.19 Admin Reporting & Operational Views
- **AR-1** Weekly schedule grid (facility × day × time slot) with capacity + booked counts color-coded
- **AR-2** Per-class enrollment trends (last N weeks)
- **AR-3** Per-instructor load / class count / fill rate
- **AR-4** Open seats heatmap (which time slots underfill, candidates for cuts)
- **AR-5** Cancellation policy effectiveness (how many refunds at each tier triggered)
- **AR-6** Series min-attendees risk dashboard (which series are below min as the min-by date approaches)

---

# 3. Design Principles

Foundational decisions that constrain the rest of the plan. Use cases and phases that conflict with them are out of scope (see §4 Alternatives Considered).

- **P-1 No per-class drop-in pricing.** Recurring class instance access is gated by membership tier only — your tier either includes a class or you need to upgrade. There is no "pay $X to attend this one class as a member-of-the-wrong-tier" flow. Workshops and series remain pay-per-attendee with per-tier pricing (including a non-member tier where the offering is open to non-members).
- **P-2 Price-schedule infrastructure for all tiered pricing.** All per-tier pricing — class series, workshops, couple/family memberships, AND specialty-instructor compensation — flows through the existing `product_prices` + `price_schedules` infrastructure. Rate changes roll forward via a new `price_schedule` row without disrupting in-flight series purchases or active subscription periods.
- **P-3 Workshop / series share the class-schedule infrastructure but differ by product kind.** Workshops and series are NOT a flag on the schedule. Each is its own `classes` row (`kind='workshop'` / `kind='series'`) with one `class_instances` row per run, its own product (`kind='workshop'` / `kind='class_series'`), and a bounded impl per run. A workshop is a single-slot bounded instance; a series is a multi-slot bounded instance with a `class_series_instances` bundle augmentation (Phase 7). The existing standalone "event" Product kind survives for non-class one-offs (parties, anniversary). (Supersedes the original "workshop = class_schedule with is_series=true" framing.)
- **P-4 Multi-occupancy rooms.** Session derivation + the booking room-conflict check permit parallel sessions in the same room as long as combined occupancy ≤ `location_rooms.concurrent_capacity` (open gym + skill-development workshop sharing a room is fine). The check sums derived + persisted overlapping sessions. Hard block only when capacity would overflow.
- **P-5 Staff-only attendance attribution.** Users mark *planned* attendance (template + per-instance exceptions); staff is the only role that records *actual* attendance. No kiosk / self-check-in — would allow users to mark friends present to game prerequisite gates (SL-10 / SL-11).
- **P-6 No user self-serve refunds.** Classes, series, and workshops are officially non-refundable from the user side. Staff may issue a voucher (BC-6) at their discretion as a case-by-case alternative. Admin-initiated cancellations (studio cancels a session) still issue automatic refunds — that's distinct from user-initiated cancels.
- **P-7 Versioned, priority-resolved schedule implementations with lazy session derivation.** A class's schedule is not a flat row but a stack of versioned implementations under a `class_instances` run, resolved by priority + validity window (higher priority wins; same priority may not overlap). Sessions are derived on the fly from the active instance + active impl + slots — there is no pre-materialization step and no "schedule exceptions" table. Holidays, closures, and schedule changes are all just higher-priority impls. `event_sessions` rows persist only when an occurrence is recorded against (cancellation, instructor sub, note, paid booking, check-in, no-show). See [[Class Schedule Implementations Redesign]] for the full model and the L-1..L-9 locked decisions.

# 4. Alternatives Considered

Options that surfaced during planning and were rejected. Rationale recorded here so future readers don't re-litigate.

### Rejected
- **Flat single-row class-schedule table + pre-materialized sessions.** The original Phase 1 design had one `class_schedules` row = class + facility + room + recurrence + single start time, and a "materialize through date X" button that pre-created `event_sessions`. Rejected (see [[Class Schedule Implementations Redesign]]): it couldn't express different times per day or multiple slots per day, had no clean way to do holiday overrides without a separate "exceptions" mechanism, conflated "the schedule" with "the bundle product", and the materialization horizon was a maintenance burden. Replaced by the three-level versioned-impl + lazy-derivation model (P-7).
- **`recurrence_pattern` enum (weekly / biweekly / custom).** Rejected — biweekly's odd/even-week semantics are ambiguous and convention-dependent, "custom" was undefined, and there's no demonstrated need. Weekly is implicit in the slot rows; complex cadences are expressible by stringing together short-window impls.
- **Free trial / first-class-free for non-members.** Classes are high-skill and ad-hoc drop-ins are disruptive. Non-members instead route through the new-member intro workshop (M-12).
- **Class packs (5 / 10 / 20-pack punch cards).** Conflicts with P-1 (no drop-in pricing). Pay-per-class models reduce commitment and total attendance. Access lives at the membership tier, not in pre-purchased class counts.
- **Late-cancellation / no-show fees.** Classes are officially non-refundable per P-6. The studio prefers a discretionary staff-issued voucher (BC-6) over a punitive fee schedule.
- **Same-day class swap endpoint** (cancel-one-and-book-another atomically). Sequential cancel + rebook is sufficient given typical class capacity.
- **Self check-in via kiosk or QR code.** Violates P-5. Users could mark friends as attending to satisfy attendance-count prerequisites (SL-10) and game the gate.
- **Per-user waitlist-default preference.** Default behavior (always offer waitlist when full) is good enough; per-user preferences add UI without a clear win.
- **Bulk booking ("book me into every Tuesday for 4 weeks") for paid drop-ins.** No drop-in pricing exists (P-1). For recurring attendance, users set an attendance template (§2.8).
- **Class-pack household sharing across multiple people.** Replaced by the couple / family membership tier (M-11), modeled at the membership level rather than the pack level.
- **Auto-template-suggest engagement nudge** ("you attended X last Tuesday — add to your template?"). User-initiated template is sufficient; no auto-nudging.
- **Substitute-instructor matching / qualified-pool ranking.** Admin reaches out directly when an instructor is unavailable. Not enough volume to warrant automation.
- **$0 purchase rows for membership-included class bookings.** Originally proposed to keep the `booking → purchase` invariant intact for audit, but rejected: class bookings are an *attendance / metrics* record, not a money trail. `bookings.purchase_id` is nullable for membership-included recurring class attendance. Money trails live on paid bookings (workshops, series, intro, guest pass).
- **Attendance templates as confirmed bookings.** Originally proposed to auto-create `booking` rows at session-materialization time, which would consume capacity and trigger the waitlist. Rejected: the template is a personal fitness-planning tool, not a reservation. No bookings are created from templates; bookings exist only when (a) staff checks the user in, or (b) the user has paid (workshops / series / intro / guest pass). Revisit if capacity issues ever force the studio to hard-reserve template entries — "good problem to have" territory.
- **Intro-workshop attendance grants temporary access to recurring classes.** Originally proposed as a non-member on-ramp ("attend intro workshop → 14 days of class access"). Rejected: the intro workshop is just a one-time sample event + sales pitch. It does NOT grant any entitlement or permission. Non-members who want recurring class access must buy a membership.
- **Email on instructor substitution.** Originally proposed as a >24h-in-advance email to affected attendees. Rejected to avoid email overload — the new instructor is shown on the user's homepage today-classes feed and on the calendar instead.
- **Pre-window check-in badge on user homepage.** Originally proposed as a "checked in" confirmation visible on the home page once staff has checked the user in. Rejected as too complicated for the value.
- **Weekly digest including eligible-but-untemplated suggestions.** Originally proposed as an engagement nudge ("classes you're eligible for this week — want to add to your template?"). Rejected: the digest only lists items the user has signed up for or marked on their template.
- **Email when same-day predecessor (SL-11) auto-cancels its dependent booking.** Originally proposed as a courtesy explanation. Rejected: the cascade is silent (no email). Documented in the use case (SL-11).

### Consolidated (not rejected, but not standalone)
- **Bookable-from-anyone (member books a guest with no prior gift-permission setup).** Folded into M-9 as the "auto-create a minimal guest account on guest-pass redemption" implementation note.
- **Workshop vs class-vs-event taxonomy.** Resolved by P-3 (workshop = `class_schedule` of length 1).
- **Hard room-conflict block on session creation.** Softened to P-4 (honor `location_rooms.concurrent_capacity`; allow parallel things in the same room when total occupancy fits).

---

# 5. Bucketing

Buckets express priority + sequencing constraint, not just nice-ness. "Must have" = required for first useful version of class scheduling; "Should have" = needed before broad rollout; "Nice to have" = improves engagement / retention; "Could have" = useful but not differentiated; "Stretch" = long-term / nontrivial.

## 5.1 Must Have (MVP — "we can run classes")
- C-1, C-2, C-3, C-4, C-5  *(class catalog + display)*
- CS-1 .. CS-6, CS-9  *(schedule authoring + lazy derivation + active-schedule preview)*
- CS-8  *(multi-occupancy rooms per P-4 — required from day 1 because rooms are shared)*
- M-1, M-3, M-4, M-5, M-6, M-7  *(membership-gated access and pricing — the central business model under P-1)*
- M-12  *(intro workshop as the non-member on-ramp — required so the gate has a way in)*
- CAP-1, CAP-2, CAP-3, CAP-7  *(capacity + waitlist for class instances)*
- BC-1, BC-2, BC-3, BC-4, BC-5  *(book, cancel-with-no-refund per P-6, admin cancel with refund, policy display)*
- SE-1, SE-2, SE-3, SE-5  *(closures, instance cancellation, substitution, refund-on-cancel)*
- AW-1, AW-2  *(per-permission advance booking days)*
- CI-1, CI-2, CI-4, CI-9  *(staff check-in with window, pre-pop from bookings + history)*
- N-1, N-2, N-3, N-5  *(confirmation, cancellation, waitlist promotion, substitution emails — existing)*
- PA-1, PA-2, PA-3  *(attendance history with filters)*

## 5.2 Should Have (Before Public Launch)
- C-7  *(class category / tag taxonomy — drives filter and prerequisites)*
- CS-7  *(per-instance description override + homepage surfacing — drives engagement)*
- SL-1 .. SL-8  *(skill levels end-to-end)*
- SL-10, SL-11, SL-12  *(attendance-count + same-day sequencing prerequisites — concrete near-term need from the partner-acro back-to-back pattern)*
- AT-1 through AT-10  *(attendance templates — core engagement feature)*
- WD-1, WD-2, WD-3, WD-5  *(weekly Sunday digest + iCal attachments)*
- iCal generator extensions — UID + RRULE + multi-VEVENT + STATUS:CANCELLED + VTIMEZONE on top of existing `util/ical_generator` (foundation for AT-4, WD-3, N-1)
- AW-3, AW-4, AW-5  *(sign-up open reminders)*
- ST-1 .. ST-6  *(instructor shift trades)*
- CSer-1 .. CSer-10  *(series + workshops, each a `classes` row of `kind='series'`/`'workshop'` with per-run `class_instances` + bounded impls per P-3, with min/max + pro-rating + auto-cancel)*
- CAP-4, CAP-5, CAP-6  *(min-attendees, auto-cancel cron)*
- CI-3, CI-5, CI-6, CI-7, CI-8  *(check-in autocomplete, walk-in, post-window edits)*
- BC-6  *(staff-issued voucher tool — operational lever under P-6)*
- M-10  *(effective-dated price changes — gates how rate edits roll out for workshops + series)*
- N-4, N-7, N-10  *(min-not-met emails, exception notes routed to instructor, notification preferences)*

## 5.3 Nice to Have
- C-8  *(extended instructor profile pages)*
- M-9  *(guest pass — bring-a-friend with auto-create of guest account)*
- M-11  *(couple / family membership tier)*
- SI-1, SI-2, SI-3, SI-6  *(specialty instructor cost recoupment + pricing assistant; rates flow through `price_schedules` per P-2)*
- SI-4  *(per-class cost vs. revenue report)*
- WD-6  *(per-user subscribable iCal feed URL)*
- AR-1, AR-2, AR-3  *(scheduling reports)*
- PA-4  *(CSV export of attendance history)*
- M-8  *(membership-upgrade unlocks classes in real time — depends on permission cache invalidation)*
- C-6  *(catalog filter / search)*

## 5.4 Could Have
- M-13  *(per-session price override)*
- M-14  *(partner / friend multi-attendee booking — NOT for kids)*
- CAP-8  *(waitlist auto-confirm cap)*
- AR-4, AR-5  *(open-seat heatmap, refund effectiveness)*
- SL-9  *(skill assignment history audit trail in the UI — the table can capture from day 1)*

## 5.5 Stretch (Long-Term)
- N-11  *(favorite-instructor notifications — very low priority)*
- R-1, R-2, R-3, R-4  *(reliability tracking + soft / hard penalties)*
- R-5  *(consecutive no-show cap — only matters once attendance caps bite)*
- PR-1, PR-2, PR-3, PR-4  *(specialty instructor payroll)*
- SI-5  *(payroll export — subset of PR)*
- AR-6  *(series min-attendees risk dashboard)*

---

# 6. Implementation Phases

Each phase is intended to land independently and be releasable. Within a phase, work is ordered lowest-layer-first: DB schema → table helpers → business logic → endpoints → frontend → admin metadata. Each phase will eventually have its own implementation document; this is the road map.

Layering reminder (from CLAUDE.md):
1. db_schema
2. sql_util/table_helpers
3. business_logic
4. endpoints (thin)
5. ui (Angular)
6. admin metadata registration in `create_database.cpp`
7. tests at every level

Subsections within each phase are numbered. Checkboxes are at the leaf-work-item level.

---

## Phase 1 — Foundations: Class Catalog & Schedule Authoring (Must Have core)

> **Redesigned 2026-05-28.** This roadmap outline reflects the three-level versioned-impl model. The detailed implementation plan lives in [[Classes Phase 1 - Catalog and Schedule Authoring]]; the design rationale + locked decisions (L-1..L-9, OQ-CSI-1..21) live in [[Class Schedule Implementations Redesign]].

**Goal:** admin can define a class, an instance (run), and versioned implementations with slots; the public catalog shows classes; sessions are derived on the fly (no materialization).

### 1.1 Design decisions — RESOLVED (see [[Class Schedule Implementations Redesign]] §2.0)
- [x] Taxonomy: "event" Product kind stays for non-class one-offs; `classes.kind` enum (`recurring` | `workshop` | `series`) discriminates class offerings; workshops + series are runs (`class_instances`) with their own product, not a flag on the schedule (L-7, P-3).
- [x] Attendance templates create NO bookings for membership-included classes — pure intent, no capacity consumption (resolved in Phase 5 redesign; lazy model, L-... / OQ-CSI-12).
- [x] Room conflict: honor `location_rooms.concurrent_capacity`; sum derived + persisted overlapping sessions (P-4).
- [x] Lazy session derivation instead of pre-materialization (OQ-CSI-12); product on `class_instances` (L-3); single `classes` row across runs (L-7).

### 1.2 Database schema (three-level)
- [ ] Extend `classes`: `default_capacity`, `is_active`, `created_us`, `updated_us` (already present) + **`kind` enum** (`recurring`|`workshop`|`series`, default `recurring`). NO `product_id` here.
- [ ] New `class_instances` table: `id`, `class_id` (FK), `name`, `valid_from_us`, `valid_to_us` nullable, `product_id` (FK — pricing / visibility / booking permission / cancellation policy / advance window), `is_active`, `created_us`, `updated_us`.
- [ ] New `class_schedules` table (the impl): `id`, `class_instance_id` (FK), `name`, `priority` (default 3), `valid_from_us`, `valid_to_us` nullable, `is_active`, `created_us`, `updated_us`.
- [ ] New `class_schedule_slots` table: `id`, `class_schedule_id` (FK), `day_of_week`, `start_time_minutes`, `duration_minutes` (default 60), `facility_id` (FK), `location_room_id` (FK), `instructor_person_id` (FK nullable), `predecessor_class_schedule_slot_id` (FK nullable, self-ref), `capacity_override` nullable, `created_us`, `updated_us`.
- [ ] Add `class_schedule_slot_id` int64 nullable + `occurrence_date_us` int64 nullable + `class_id` int64 nullable (denormalized) to `event_sessions`. (Existing service / event session usage keeps these NULL.)
- [ ] Wire into `make_database_info.cpp`, `CreateTables()`, and admin metadata. Forgetting `admin_top_level_tables` / `admin_nested_tables` is the most common mistake.

### 1.3 Table helpers (under `sql_util/table_helpers/`)
- [ ] `ClassInstances` helper: CRUD + `GetActiveInstance(classId, atUs)`, `GetInstancesByClass`, `GetUpcomingInstances`.
- [ ] `ClassSchedules` helper (instance-scoped): CRUD + `GetActiveImplementation(classInstanceId, atUs)`, `GetImplementationsByInstance`.
- [ ] `ClassScheduleSlots` helper: CRUD + `GetSlotsByImplementation`, `GetActiveSlotsForClassOnDay(classId, dateUs)`, `GetSlotsPotentiallyConflictingInRoom`.
- [ ] Extend `EventSessions`: `LookupBySlotAndDate`, `GetOrphanedFutureSessionsForInstance`.
- [ ] Tests for all using existing `TestDatabaseUtil` pattern.

### 1.4 Business logic (under `business_logic/scheduling/`)
- [ ] `ClassInstanceHelper`: `CreateInstance`, `UpdateInstance`, `CloseInstance`, `MigrateRecurringClassToNewProduct` (close perpetual + open new, copy slots forward).
- [ ] `ClassScheduleHelper`: `CreateImplementation(instanceId, req)` / `UpdateImplementation` (validates window-within-instance + same-priority-no-overlap; runs the impl-save sweep), slot CRUD, `EnsureSessionExists(slotId, occurrenceDateUs)` (lazy, idempotent), `GetDerivedSessionsForRange(classId, fromUs, toUs)`, `GetActiveScheduleView`, `SweepOrphanedFutureSessions`. No materializer.
- [ ] `ClassCatalogHelper::GetClassesVisibleToPerson(personId)` — resolves visible classes + best tier price + inclusion status via the instance's product (M-4/M-5/M-7).
- [ ] Tests for all helpers.

### 1.5 Endpoints (thin)
- [ ] `GET /api/classes` — public catalog; `GET /api/classes/<id>` — public detail (derived upcoming sessions + instructors; for workshop/series, list of upcoming instances).
- [ ] `POST/PUT/DELETE /api/admin/class_instance[/<id>]`; `POST /api/admin/class/<classId>/migrate_product`.
- [ ] `POST/PUT/DELETE /api/admin/class_schedule` impl endpoints (scoped under instance); slot endpoints.
- [ ] `GET /api/admin/class_instances?class_id=`; `GET /api/admin/class_schedules?class_instance_id=`; `GET /api/admin/class_schedule_preview?class_id=&date_us=`.
- [ ] No `materialize` endpoint. All admin endpoints behind `manage_class_schedule`. Endpoint tests for success + permission-denied + validation paths.

### 1.6 Frontend (Angular)
- [ ] Public `classes` catalog grid + detail page (recurring → derived upcoming sessions; workshop/series → upcoming-runs list).
- [ ] Admin three-level nav: class detail → instance list/detail → impl list/detail → slot editor (day-of-week dropdown, HH:MM time picker, duration default 60, facility/room/instructor autocompletes, predecessor-slot picker).
- [ ] "Migrate to new product" action; "copy slots from impl" picker on impl create; "schedule on date X" preview; impl-save sweep confirmation modal. No materialize dialog.
- [ ] Component specs for every component touched.

### 1.7 Admin metadata in `create_database.cpp`
- [ ] Register `class_instances`, `class_schedules`, `class_schedule_slots` (nesting: `classes` → `class_instances` → `class_schedules` → `class_schedule_slots`). Friendly names, column data info, display templates, permissions per CLAUDE.md.

### 1.8 Tests-required summary for the phase
- [ ] Table helpers, business logic helpers, endpoints (incl. ValidationError → 400), `ServerAccess.mock.spec.ts`, component specs — all per CLAUDE.md.

---

## Phase 2 — Membership-Gated Class Access & Drop-In Booking (Must Have)

**Goal:** members see classes included in their membership; non-included and non-members get tiered drop-in pricing; bookings are created, capacity tracked, waitlist works, cancellations refund correctly.

### 2.1 Design decisions
- [ ] Lock down "is template attendance a confirmed booking" question from §1.1.
- [ ] Define the "included" semantics: does an included-with-membership booking create a `purchase` row with $0 total + a one-shot entitlement assignment, or is it a `booking` with `purchase_id IS NULL`? Recommend the $0 purchase route — keeps the `booking → purchase` invariant intact and gives a place for refunds / audit. Document.

### 2.2 Database schema
- [ ] Reuse `product_prices` (per-permission) for class drop-in pricing — no new table.
- [ ] Reuse `product_booking_windows` for AW-1.
- [ ] Reuse `product_entitlement_rules.grants_permission_id` for tier-grants-permission.
- [ ] Reuse existing `cancellation_policies` linked from `products.cancellation_policy_id`.
- [ ] (Optional) Add `event_sessions.required_skill_level_ids` text[] — defer to Phase 3 (skill levels).

### 2.3 Table helpers
- [ ] No new helpers — extend existing `Products` / `ProductPrices` / `ProductBookingWindows` helpers if needed (they should already be complete).

### 2.4 Business logic
- [ ] Extend `EventSessionHelper::GetVisibleEventSessions` to surface `class_id`, `class_name`, `class_photo_url`, plus resolved per-user pricing including the "$0 because included in your membership" case (M-4 / M-5 / M-7).
- [ ] Add a `ClassBookingHelper` (or just extend `BookingHelper::BookEvent`) to handle the $0 / included path: detect the user has an active entitlement that grants the class's required permission, create the $0 purchase + booking, no Square call.
- [ ] Pricing resolution: pick lowest of all matching `product_prices` rows (one for `permission_id IS NULL` and one per permission the user has) (M-5).
- [ ] Refund pro-rating on admin cancel of a class instance: full refund for paid bookings, no refund for $0 included bookings (SE-5).
- [ ] Tests for both paid and included paths.

### 2.5 Endpoints
- [ ] Reuse `POST /api/book_event/<id>` — confirm it correctly handles $0 case (no Square call, immediate booking confirmation).
- [ ] Reuse `POST /api/cancel_booking/<id>`.
- [ ] Confirm `GET /api/visible_event_sessions?placement=upcoming` surfaces class metadata.
- [ ] Tests.

### 2.6 Frontend
- [ ] Class detail page → "Book this class" button shows the user's tier price (or "Included" if $0).
- [ ] Calendar view colors / labels classes by inclusion status (e.g. "Member" tag vs. price chip).
- [ ] My-bookings page already handles event bookings — confirm class instances render with class info (photo + name) rather than generic event chrome.
- [ ] Component specs.

### 2.7 Admin metadata
- [ ] No new admin tables, but verify admin UI for `product_prices` allows tier-per-product configuration end-to-end for class products.

---

## Phase 3 — Skill Levels (Should Have, blocking some Must Have edge cases)

**Goal:** skill levels exist, staff can assign them, classes can require them, booking enforces them.

### 3.1 Design decisions
- [ ] Confirm: skill levels are global (one set across the studio), not per-facility. Recommend global.
- [ ] Confirm: can a class have multiple skill requirements (AND) or just one? Recommend AND-of-multiple to match Overview phrasing.

### 3.2 Database schema
- [ ] `skill_levels`: id, code (unique), name, description, sort_order, is_active, created_us, updated_us. Photo support via `photo_support_tables` entry.
- [ ] `skill_level_assignments`: id, person_id (FK), skill_level_id (FK), assigned_by_person_id (FK), assigned_us, removed_us nullable, removed_by_person_id nullable, removed_reason text, note text, created_us, updated_us. UNIQUE on (person_id, skill_level_id) WHERE removed_us IS NULL.
- [ ] `class_skill_requirements`: id, class_id (FK) or event_session-level override, skill_level_id (FK), required_at_signup bool (vs attended-but-warn), created_us. Recommend keying off `classes` only; override-per-instance is Could Have.

### 3.3 Table helpers
- [ ] `SkillLevels`, `SkillLevelAssignments`, `ClassSkillRequirements` helpers, full CRUD + queries: `GetActiveAssignmentsForPerson`, `PersonHasSkill(personId, skillLevelId)`, `GetRequirementsForClass`.
- [ ] Tests.

### 3.4 Business logic
- [ ] `SkillLevelHelper`:
  - [ ] `AssignSkill(personId, skillLevelId, assignerPersonId, note)` — staff-only-callable (caller checks permission), idempotent if already active.
  - [ ] `RevokeSkill(personId, skillLevelId, revokerPersonId, reason)`.
  - [ ] `GetPersonSkills(personId)` → list with photo URLs and assignment dates.
  - [ ] `PersonMeetsClassRequirements(personId, classId)` → bool + missing skill list.
- [ ] Hook into booking flow: `BookEvent` / class booking calls `PersonMeetsClassRequirements` and rejects with `ErrorResponse::ValidationError` (400) if missing — admin can bypass via existing admin role.
- [ ] Tests including the bypass path.

### 3.5 Endpoints
- [ ] `GET /api/skill_levels` (public catalog).
- [ ] `GET /api/skill_levels/<id>` (public detail).
- [ ] `GET /api/me/skills` (logged-in user views own).
- [ ] `GET /api/admin/person/<id>/skills` (staff lookup; permission `manage_skills` or instructor role).
- [ ] `POST /api/admin/person/<id>/skill/<skillId>` (assign).
- [ ] `DELETE /api/admin/person/<id>/skill/<skillId>` (revoke).
- [ ] `POST /api/admin/class/<id>/skill_requirement` (set).
- [ ] `DELETE /api/admin/class/<id>/skill_requirement/<skillId>`.
- [ ] Endpoint tests.

### 3.6 Frontend
- [ ] User portal: `/my/account/skills` — grid of earned badges (photo + name + earned date).
- [ ] Class detail page: skill-requirement section ("you have / don't have", linked to skill detail).
- [ ] Staff portal: person-search page + assign / revoke UI with a note field (SL-8).
- [ ] Admin: class edit form gains a "requires skill levels" multiselect.
- [ ] Specs.

### 3.7 Admin metadata
- [ ] Register all three new tables in `admin_top_level_tables` / `admin_nested_tables` appropriately. `skill_level_assignments` is nested under `people`; `class_skill_requirements` is nested under `classes`.

---

## Phase 4 — iCal Generator Extensions (Should Have, foundation)

**Starting point:** `util/ical_generator.h/cpp` already exists — `ICalGenerator::GenerateICalendar(const ICalEvent&)` is wired into `book_event`, `book_service`, `cart_checkout`, `payment_helper`, `staff_upgrade_session`.

**Goal:** add the RFC 5545 features the current generator is missing — `UID`, `RRULE`, multi-VEVENT bundles, `STATUS:CANCELLED`, `ORGANIZER`/`ATTENDEE`, `VTIMEZONE`, line folding — to unblock attendance templates (Phase 5), the weekly digest (Phase 6), and cancellation-syncs-to-calendar.

### 4.1 Extend the existing `ICalEvent` / `GenerateICalendar` in `util/ical_generator.h/cpp` — extend in place, do not create a parallel module
- [ ] Add `std::string uid` field (RFC 5545 mandatory). UID convention: `booking-<bookingId>@knottyyoga.com`, `schedule-<scheduleId>-person-<personId>@knottyyoga.com`, or `session-<eventSessionId>@knottyyoga.com`.
- [ ] Add `std::string status` — `""` / `"CONFIRMED"` / `"CANCELLED"`. `"CANCELLED"` emits `STATUS:CANCELLED`.
- [ ] Add `std::string rrule` — emitted verbatim as `RRULE:<value>` when set. Built by helpers in 4.3.
- [ ] Add `std::string organizerEmail` / `organizerName` → `ORGANIZER;CN=...:mailto:...`.
- [ ] Add `std::string attendeeEmail` / `attendeeName` → `ATTENDEE;CN=...;RSVP=FALSE:mailto:...`.
- [ ] Add `std::string sequence` (default `"0"`) — increments on each update.
- [ ] Existing fields (`title`, `startTimeUs`, `endTimeUs`, `timezone`, `location`, `description`) stay; existing callers keep working with new fields unset.

### 4.2 Extend `GenerateICalendar` + add multi-event overload in `util/ical_generator.cpp`
- [ ] Update single-event `GenerateICalendar` to emit `UID`, `DTSTAMP`, `SEQUENCE`, plus conditional `STATUS`, `RRULE`, `ORGANIZER`, `ATTENDEE`.
- [ ] Add overload `GenerateICalendar(const std::vector<ICalEvent>&)` — one VCALENDAR wrapping multiple VEVENTs (for the weekly digest).
- [ ] Emit `VTIMEZONE` block when `timezone` is set AND an `RRULE` is present; use existing `date` library for DST transitions. Non-recurring entries continue with `DTSTART:...Z` UTC.
- [ ] Add long-line folding per RFC 5545 §3.1 (75 octets + CRLF + space continuation).

### 4.3 Helper functions for common patterns
- [ ] `BuildBookingUid(int64_t bookingId)` / `BuildSessionUid(int64_t sessionId)` / `BuildTemplateUid(int64_t classScheduleSlotId, int64_t personId)` — centralize UID format. (Template UID keys off the slot now, per the redesign.)
- [ ] `BuildWeeklyRRule(const std::vector<int>& daysOfWeek, int64_t untilUs)` → `FREQ=WEEKLY;BYDAY=...;UNTIL=...`. Used by attendance template emails (Phase 5). Weekly is the only cadence — `recurrence_pattern` was dropped in the redesign, so no biweekly/custom RRULE builders are needed.
- [ ] Extend `util/ical_generator_test.cpp`: golden-text fixtures for UID, RRULE, STATUS:CANCELLED, multi-VEVENT, VTIMEZONE, line folding.

### 4.4 Update existing email paths to use the new fields (.ics is already attached today)
**Note:** confirmation / cancellation mails already attach `.ics` today. This is about feeding them the new struct fields.
- [ ] In `book_event`, `book_service`, `cart_checkout`, `payment_helper`, `staff_upgrade_session` — populate `uid = BuildBookingUid(bookingId)`.
- [ ] `BookingCancellationMail` — set `status = "CANCELLED"`, same UID as original confirmation, `sequence = "1"`.
- [ ] `SessionCancellationMail` — same treatment per attendee.
- [ ] `WaitlistPromotionMail` — fresh `BuildBookingUid(newBookingId)` + `status = "CONFIRMED"`.
- [ ] `ProviderCancelledSessionMail` / `ProviderChangeClientMail` — `STATUS:CANCELLED` where original booking is torn down.
- [ ] Update existing email tests to assert UID; add tests for cancellation-with-`STATUS:CANCELLED`.

### 4.5 Frontend (no change today)
- [ ] No UI change, but verify the user portal shows email-receipt history correctly (out-of-scope for content of email, just confirming attachments don't break existing displays).

---

## Phase 5 — Attendance Templates (Should Have, core engagement)

**Goal:** users plan a weekly routine, see it on home + calendar, get a Sunday digest, mark per-instance exceptions with notes.

### 5.1 Design decisions — RESOLVED by the redesign
- [x] Template entries do NOT create `bookings` for membership-included classes — they are pure intent, consume no capacity, touch no waitlist (lazy model, Mason's §140 note). The earlier "templates = confirmed bookings" recommendation is reversed.
- [ ] Decide whether templates can include paid offerings (recommend: no, templates are only for included-with-membership recurring classes; paid workshops/series are explicit purchases).

### 5.2 Database schema
- [ ] `attendance_templates`: id, person_id (UNIQUE — one template per person), is_active, created_us, updated_us.
- [ ] `attendance_template_entries`: id, template_id (FK), `class_schedule_slot_id` (FK — bind to the slot, which has stable identity), created_us. UNIQUE on (template_id, class_schedule_slot_id).
- [ ] `attendance_template_exceptions`: id, template_id (FK), `class_schedule_slot_id` (FK) + `occurrence_date_us` (the specific occurrence), attending bool (false = skipping, true = one-off addition), note text nullable, created_us, updated_us. UNIQUE on (template_id, class_schedule_slot_id, occurrence_date_us). (Keys off the derived-occurrence identity, not an `event_session_id`, since no row may exist.)

### 5.3 Table helpers
- [ ] `AttendanceTemplates`, `AttendanceTemplateEntries`, `AttendanceTemplateExceptions` helpers.
- [ ] Tests.

### 5.4 Business logic
- [ ] `AttendanceTemplateHelper`:
  - [ ] `GetEligibleSlotsForPerson(personId)` — slots the user is permission-eligible to attend (intersects the slot's instance product's visibility / booking permissions, and skill requirements).
  - [ ] `AddTemplateEntry(personId, classScheduleSlotId)` — creates the entry only. NO bookings created. Sends one confirmation email with a recurring `.ics` (RRULE covering the slot's future occurrences). No capacity/waitlist interaction.
  - [ ] `RemoveTemplateEntry(personId, classScheduleSlotId)` — deletes the entry. Nothing else to clean up (no bookings exist).
  - [ ] `SetException(personId, classScheduleSlotId, occurrenceDateUs, attending bool, note)` — creates / updates the exception row only. Notifies instructor (staff portal feed; no email by default). No booking adjustment.
  - [ ] NO session-materialization hook — there's no materialization. Template entries are evaluated lazily against derived occurrences when rendering the homepage / digest / calendar.
  - [ ] Stale-slot handling: if a slot referenced by a template entry is deleted (impl edit / migration), surface it in the user portal as "this slot no longer exists, pick a new one".
  - [ ] Tests covering entry add/remove, exception set, eligibility filtering, stale-slot surfacing.
- [ ] Extend the derived-session / calendar query to surface per-user "is on my template" + "is exception" flags.

### 5.5 Endpoints
- [ ] `GET /api/me/eligible_slots` — grid view feed (eligible slots, not "schedules").
- [ ] `GET /api/me/template` — current template entries + exceptions.
- [ ] `POST /api/me/template/entry` { class_schedule_slot_id }.
- [ ] `DELETE /api/me/template/entry/<classScheduleSlotId>`.
- [ ] `POST /api/me/template/exception` { class_schedule_slot_id, occurrence_date_us, attending, note? }.
- [ ] `DELETE /api/me/template/exception` { class_schedule_slot_id, occurrence_date_us }.
- [ ] `GET /api/me/today_classes` — homepage feed (eligible today: checked = booked / template, unchecked = eligible-but-not-claimed).
- [ ] Endpoint tests.

### 5.6 Frontend
- [ ] User portal: "My Schedule" page with weekly grid of eligible classes; checkboxes toggle template membership.
- [ ] Calendar view: visually distinguish template-attending (filled), one-off addition (filled with star), exception (struck through with note tooltip).
- [ ] Home page: today's classes with checkmarks per AT-7.
- [ ] Per-instance "I can't make it" UI with note field.
- [ ] Specs.

---

## Phase 6 — Weekly Digest Email & Notification Preferences (Should Have)

**Goal:** Sunday-noon (facility-TZ-aware) digest email lands with multi-VEVENT `.ics`.

### 6.1 Database schema
- [ ] `user_notification_preferences`: id, person_id (UNIQUE), weekly_digest_enabled bool default true, digest_send_dow int default 0 (Sunday), digest_send_hour_local int default 12, created_us, updated_us. (Per-channel toggles can be added in Could Have.)

### 6.2 Table helpers
- [ ] `UserNotificationPreferences` helper + tests.

### 6.3 Business logic
- [ ] `WeeklyDigestHelper`:
  - [ ] `BuildDigestForPerson(personId, weekStartUs)` → struct with this week's bookings, services, events. Idempotent (no DB mutation).
  - [ ] `SendDigestForPerson(personId, weekStartUs)` → builds, formats email, attaches multi-VEVENT `.ics`, queues via MailHelper.
  - [ ] `SendPendingDigests(asOfUs)` → iterates active users with `weekly_digest_enabled=true` whose local Sunday-noon has just passed but no digest sent yet this week. Tracks per-user `last_digest_sent_us` to be idempotent.
- [ ] Tests with test mail helper.

### 6.4 Endpoints (admin only)
- [ ] `POST /api/admin/send_weekly_digests` — runs `SendPendingDigests(now)`. Idempotent. Called by `knottyyoga_helper`.

### 6.5 Scheduler integration
- [ ] Add new job to `knottyyoga_helper`: hourly tick that checks each facility's local time and triggers `send_weekly_digests` when any facility crosses Sunday noon. (Hourly cadence is fine because the endpoint is idempotent.)

### 6.6 Frontend
- [ ] `/my/account/preferences` page with weekly digest toggle.
- [ ] Spec.

---

## Phase 7 — Class Series + Workshops (Should Have)

**Goal:** admin can create series with start/end + min/max + per-tier series price + pro-rating; users buy whole series or join mid-series.

### 7.1 Database schema (per the redesign — see [[Class Schedule Implementations Redesign]] §1.5a)
- [ ] A series is a `classes` row of `kind='series'` (shared marketing identity across runs) + one `class_instances` row per run (May 2026, Oct 2026 …) + a `kind='class_series'` product on each instance + a base impl per instance (+ holiday-override impls). NOT a flag on the schedule.
- [ ] New `class_series_instances` table (1:1 augmentation of `class_instances`): `min_attendees`, `min_by_us`, `min_not_met_policy`, `prorated_signups_allowed`.
- [ ] Series purchase creates one `purchase` + one parent `entitlement` covering the run's derived occurrences; per-occurrence `event_sessions` rows are ensured (lazy → eager at purchase) with `series_purchase_id` set, and `bookings` reference the same `purchase_id`.
- [ ] Add `event_sessions.series_purchase_id` nullable.

### 7.2 Business logic
- [ ] `ClassSeriesHelper`:
  - [ ] `CreateSeriesInstance(...)` — creates the `class_instances` row + `class_series_instances` augmentation + base impl + slots, atomically. No materialization; occurrences derive.
  - [ ] `BookFullSeries(personId, classInstanceId)` — price = (derived-occurrence count in the instance's window) × (per-tier base, M-5); ensures all occurrence `event_sessions` rows + creates bookings under one purchase.
  - [ ] `BookProratedRemainingSeries(personId, classInstanceId, joinDateUs)` — counts remaining derived occurrences, pro-rates, ensures + books those.
  - [ ] `CancelSeriesInstance(classInstanceId, adminPersonId, reason)` — cancels each ensured occurrence, refunds each booker per their paid amount.
  - [ ] `CheckMinAttendees(classInstanceId)` — daily job; if past `min_by_us` and confirmed-count < `min_attendees` and policy is `auto_cancel_refund`, run `CancelSeriesInstance`.
- [ ] Tests covering all four lifecycle paths + the daily min-check + the empty-override-impl reduces-occurrence-count pricing case.

### 7.3 Endpoints
- [ ] `POST /api/admin/class_series_instance` (admin create a run — `class_instances` + `class_series_instances` + base impl under a `kind='series'` class).
- [ ] `POST /api/book_class_series/<classInstanceId>` (full or prorated; server decides based on `joinDateUs` vs the instance's `valid_from_us`).
- [ ] `POST /api/admin/series/<classInstanceId>/check_min_attendees` (admin manual run, also called by scheduler).
- [ ] Endpoint tests.

### 7.4 Scheduler integration
- [ ] New job in `knottyyoga_helper`: daily 03:00 local — call `check_series_min_attendees` for each active series.

### 7.5 Frontend
- [ ] Catalog and class detail show series cards with start/end date, # of instances, per-tier prices, min/max.
- [ ] Booking flow handles full vs. prorated automatically.
- [ ] Admin series creation UI (extension of class schedule create form).
- [ ] My-bookings shows the series as a parent row with child sessions.
- [ ] Specs.

---

## Phase 8 — Staff Check-in (Must Have core that benefits from earlier phases)

**Goal:** staff can mark attendance with autocomplete and pre-populated list including 4-week history.

### 8.1 Configuration
- [ ] Add config secrets: `class_checkin_window_before_minutes` (60), `class_checkin_window_after_minutes` (180).

### 8.2 Business logic
- [ ] `ClassCheckinHelper`:
  - [ ] `IsCheckinOpen(eventSessionId, asOfUs)` based on window secrets.
  - [ ] `GetCheckinList(eventSessionId)` → struct combining: (a) all `bookings` (confirmed or waitlisted) for the session, (b) people who attended this *class* (joined via `event_sessions.class_id` — the denormalized convenience column — across all persisted occurrences of the same class in the last 4 weeks) and were checked in. De-duplicate by person_id. (Check-in ensures the `event_sessions` row first — recording trigger #6.)
  - [ ] `CheckInPerson(eventSessionId, personId, staffPersonId)` — updates `bookings.checked_in_us` (creating a $0 walk-in booking if person has none).
  - [ ] `UndoCheckIn(eventSessionId, personId, staffPersonId)`.
  - [ ] `FinalizeAttendance(eventSessionId)` — at +N hours after end, marks unchecked confirmed bookings as `no_show`.
- [ ] Tests including the walk-in-creation path and finalize-no-shows path.

### 8.3 Endpoints
- [ ] `GET /api/staff/checkin/<eventSessionId>` — pre-populated list.
- [ ] `POST /api/staff/checkin/<eventSessionId>/person/<personId>` — check in.
- [ ] `DELETE /api/staff/checkin/<eventSessionId>/person/<personId>` — undo.
- [ ] `POST /api/staff/people/search?q=...` — autocomplete (reuse if exists).
- [ ] Permission gate: `staff` role or `manage_classes`.
- [ ] Endpoint tests.

### 8.4 Frontend
- [ ] Staff portal: check-in page with autocomplete and pre-pop list.
- [ ] Show "checked in" confirmation on user homepage (CI-6).
- [ ] Specs.

### 8.5 Scheduler integration
- [ ] Hourly job: `POST /api/admin/finalize_class_attendance` runs `FinalizeAttendance` for sessions whose checkin post-window has now closed.

---

## Phase 9 — Attendance History (Must Have)

**Goal:** user can see paginated past attendances filterable by year / month / class / instructor.

### 9.1 Business logic
- [ ] `AttendanceHistoryHelper::GetHistory(personId, filters, pageOffset, pageSize)` — joins `bookings` × `event_sessions` × `classes` × `event_session_staffing` × `people` (instructor). Filters: year, month, class_id, instructor_person_id. Status filter: `attended` only (default) or include `no_show` (Could Have UI).
- [ ] Counts query for paginator.
- [ ] Tests.

### 9.2 Endpoints
- [ ] `GET /api/me/attendance_history?year=&month=&class_id=&instructor_id=&offset=&limit=` — returns list + total count.
- [ ] Endpoint tests.

### 9.3 Frontend
- [ ] `/my/account/attendance-history` page with filters, paginator, table.
- [ ] Specs.

---

## Phase 10 — Scheduling Exceptions, Instructor Subs, Shift Trades (Must / Should Have)

> **Redesigned 2026-05-28.** Per-class schedule exceptions collapse into higher-priority implementations (L-8) — there is no `scheduling_exceptions` cascade for classes. There is no global "studio closed" lever (L-9); closures are per-class empty high-priority impls, and the closure batch UI deferred from Phase 1 (OQ-CSI-4) lands here. Instructor substitution becomes a recording trigger (ensure the `event_sessions` row, then write `event_session_staffing`). Studio-wide closures for *service sessions* keep using `scheduling_exceptions` as before — that table is untouched for the service-session path.

**Goal:** admin closes classes for date ranges (batch empty-impl action), cancels single occurrences, substitutes instructors; instructors initiate trades via existing Provider Portal.

### 10.1 Reuse audit
- [ ] Class closures: NOT via `scheduling_exceptions`. Implemented as empty high-priority impls under each affected class's active instance. Build the batch "close these N classes for this window" multiselect UI here (deferred OQ-CSI-4).
- [ ] Single-occurrence cancel: `SessionCancellationHelper` — a recording trigger that ensures + persists the `event_sessions` row with `status='cancelled'`.
- [ ] Confirm `ShiftChangeHelper` operates on `event_session_staffing` for class occurrences (currently keyed off `provider_availability`). Parallel path keyed off `event_session_staffing` rows; the row is ensured if it doesn't exist yet (lazy).

### 10.2 Database schema
- [ ] No new class-closure table (impls cover it). Service-session `scheduling_exceptions` unchanged.
- [ ] (If shift trade for classes needs its own request kind:) extend `shift_change_requests` with a nullable `event_session_id` or `event_session_staffing_id` column; document the union semantics.

### 10.3 Business logic
- [ ] Extend `SessionCancellationHelper` to handle pro-rated refunds (paid bookings get refund per cancellation policy; $0 / membership-included bookings get no refund) (SE-5). For class occurrences it ensures the `event_sessions` row first (lazy).
- [ ] New `InstructorSubstitutionHelper::Substitute(eventSessionSlotId, occurrenceDateUs, newInstructorPersonId, reason)` — ensures the `event_sessions` row (recording trigger #2), updates `event_session_staffing`. Per ST-4 there is NO attendee email — the new instructor surfaces on the homepage / calendar only.
- [ ] New `ClassClosureHelper::CloseClassesForRange(classIds, fromUs, toUs)` — creates an empty high-priority impl under each class's active instance (the batch closure action).
- [ ] Extend `ShiftChangeHelper` for class occurrences: trade affects `event_session_staffing` rows; on approval, no `free_cancel_until_us` extension (ST-5); no refund.
- [ ] Tests. Document the ensure-on-action invariant (every per-occurrence write ensures the `event_sessions` row first).

### 10.4 Endpoints
- [ ] `POST /api/admin/event_session/<id>/substitute` { new_instructor_person_id, reason }.
- [ ] `POST /api/provider/class_shift_change_request` — variant of existing endpoint scoped to class instructor assignments.
- [ ] Admin "who's teaching what" view: `GET /api/admin/instructor_load?facility_id=&date_from=&date_to=`.
- [ ] Endpoint tests.

### 10.5 Frontend
- [ ] Admin: scheduling-exceptions page already exists; extend "block" cascade to display affected class instances.
- [ ] Admin: instructor substitution dialog from event-session admin page.
- [ ] Admin: who's-teaching-what grid.
- [ ] Staff portal: shift-trade UI extension for class assignments (mostly reuse).
- [ ] Specs.

---

## Phase 11 — Sign-up Windows + Open-Reminder (Should Have)

**Goal:** users see future classes outside their booking window, request a reminder when their window opens.

### 11.1 Database schema
- [ ] `signup_open_reminders`: id, person_id (FK), event_session_id (FK), notify_at_us (when their window opens), notified_us nullable, cancelled_us nullable, created_us. UNIQUE on (person_id, event_session_id) WHERE notified_us IS NULL AND cancelled_us IS NULL.

### 11.2 Table helpers
- [ ] `SignupOpenReminders` + tests.

### 11.3 Business logic
- [ ] `SignupReminderHelper`:
  - [ ] `RequestReminder(personId, eventSessionId)` — computes `notify_at_us` = session start − user's best advance-window-days (per `product_booking_windows`); rejects if window already open.
  - [ ] `SendPendingReminders(asOfUs)` — find and send.
  - [ ] Auto-dedupe: when a booking is created, mark matching reminders cancelled.
- [ ] Tests.

### 11.4 Endpoints
- [ ] `POST /api/me/signup_reminder/<eventSessionId>` (create).
- [ ] `DELETE /api/me/signup_reminder/<eventSessionId>` (cancel).
- [ ] `POST /api/admin/send_signup_open_reminders` (cron-callable, idempotent).
- [ ] Endpoint tests.

### 11.5 Scheduler integration
- [ ] Hourly job in `knottyyoga_helper`: call `send_signup_open_reminders`.

### 11.6 Frontend
- [ ] Calendar / catalog cards for sessions in the future-but-unbookable zone show a "Sign-ups open `<date>`" hint + "Remind me" button.
- [ ] Spec.

---

## Phase 12 — Specialty Instructor Cost Recoupment (Nice to Have)

**Goal:** admin records specialty instructor costs, gets a pricing assistant + per-class cost/revenue report.

### 12.1 Database schema
- [ ] `specialty_instructor_costs`: id, class_instance_id (FK — a specialty instructor is hired for a specific run; per the redesign, cost keys off the instance not a flat schedule), instructor_person_id (FK), base_rate_cents, per_student_bonus_cents, bonus_threshold_count nullable, instructor_min_attendees nullable, instructor_max_attendees nullable, notes, created_us, updated_us. (Optional per-slot override if the specialty teacher only covers certain days of the instance.)

### 12.2 Business logic
- [ ] `SpecialtyCostHelper`:
  - [ ] `ComputeInstructorPay(sessionId, attendeeCount)` → cents.
  - [ ] `SuggestPricesForBreakeven(scheduleId or sessionId, targetAttendees, allowedTiers[])` → per-tier suggestion (cover cost on member attendance, profit on non-member).
- [ ] Tests.

### 12.3 Endpoints
- [ ] Admin CRUD via the generic CRUD endpoints (just metadata setup).
- [ ] `POST /api/admin/session/<id>/suggest_pricing` { target_attendees, tiers } → suggestion JSON.
- [ ] Endpoint tests.

### 12.4 Frontend
- [ ] Admin session detail: cost-input section + "Suggest prices" tool.
- [ ] Cost vs. revenue panel on session detail.
- [ ] Spec.

---

## Phase 13 — Discoverability Polish: Tags, Filters, Favorites (Nice to Have)

- [ ] Class category / tag taxonomy (`class_tags`, `class_tag_assignments`); UI for admin to assign tags; filter on catalog (S-8).
- [ ] Favorite-instructor feature (`user_favorite_instructors`); notification on schedule changes for favorites (S-7).
- [ ] Extended instructor profile page with class lists (S-16).
- [ ] Tests at all layers.

---

## Phase 14 — Reporting & Operational Dashboards (Nice to Have)

- [ ] Schedule grid (AR-1).
- [ ] Enrollment trends (AR-2).
- [ ] Instructor load (AR-3).
- [ ] All read-only endpoints + admin UI. Tests.

---

## Phase 15 — Could Have items batch

- [ ] Late cancel / no-show fees (S-4).
- [ ] Same-day swap endpoint (S-5).
- [ ] Waitlist auto-confirm cap (S-6).
- [ ] Per-instance price override (S-10).
- [ ] Kiosk self check-in (S-11).
- [ ] Waitlist preference per user (S-12).
- [ ] Bulk booking (S-13).
- [ ] Template-suggest engagement nudge (S-18).
- [ ] Open-seat heatmap (AR-4), refund effectiveness (AR-5).
- [ ] Each gets its own implementation doc when scheduled.

---

## Phase 16 — Stretch / long-term items

- [ ] Reliability tracking (R-1 / R-2 / R-3 / R-4) — design a "reliability score" rolled up from `bookings` history.
- [ ] Consecutive no-show auto-suspend (S-23).
- [ ] Specialty instructor full payroll model (PR-1 / PR-2 / PR-3 / PR-4 / SI-5).
- [ ] Series mid-cancel pro-rated refund (S-17) — possibly promote to Should Have if series sees high cancellation rate.
- [ ] Household / family sharing (S-14).
- [ ] Per-schedule-entry instructor pay rules (S-15).
- [ ] Quantitative completion prerequisites (S-9).
- [ ] Substitute instructor matching (S-24).
- [ ] Per-user subscribable iCal feed URL (S-26).
- [ ] Multi-attendee booking (S-28) — promote if kids' classes ship.

---

# 7. Implementation Cross-Cutting Concerns

### 6.1 Permissions to add
- [ ] `manage_class_schedule` — separate from `manage_products` so studio managers can edit schedules without product-edit rights.
- [ ] `manage_skills` — grants ability to assign / revoke skill levels. Staff role gets it by default.
- [ ] `view_admin_instructor_load` — for the "who's teaching what" admin view.

### 6.2 Permission cache invalidation
- [ ] Membership upgrades / new entitlements should refresh the session's permission set without re-login (M-8). Verify how `Session::ActiveUserHasPermission` is currently cached and whether anything beyond next request is required.

### 6.3 Test data + manual-testing-helper commands
- [ ] Add `knottyyoga_test_helper` commands: `list_class_instances`, `list_class_schedules <instance_id>`, `preview_schedule <class_id> <date>`, `assign_skill`, `set_template_entry`, `simulate_min_not_met`, `send_weekly_digest`, `simulate_signup_window_open`. (No `materialize_schedule` — derivation is lazy.)
- [ ] These commands accelerate manual QA across phases.

### 6.4 Bootstrap & seed data
- [ ] Seed a couple of demo recurring classes + a perpetual instance + a default impl with slots + a default skill level (e.g. "Beginner Inversion") in `database_helper` so a fresh DB shows something on the calendar (derived, no materialize step). Existing seeded classes (Knotty Yoga, etc.) get a perpetual instance + default impl.
- [ ] Seed default `cancellation_policy` if not already present.

### 6.5 Backwards-compat with existing event flow
- [ ] Existing one-off `event` Product kind continues to work unchanged. `event_sessions` rows without `class_schedule_slot_id` / `class_id` are valid and represent one-off events / service sessions.
- [ ] My-bookings, calendar, etc. must render both the derived-class view and the persisted event/service rows cleanly.

### 6.6 Phase 1 rewrite-vs-migrate (resolved: rewrite in place, OQ-CSI-10)
- [ ] Phase 1 was implemented against the old flat model but is **pre-deploy** — no production data to migrate. The redesign rewrites Phase 1's schema / helpers / endpoints / UI / tests in place (drop the old `class_schedules` flat table, build the three-level model fresh). No migration shims or "legacy host" branches per `feedback_no_premature_defensive_code.md`. Confirmed there is no deployed environment requiring a migration path.

### 6.6 Layering discipline reminder (from CLAUDE.md)
- [ ] No SQL or `DbCrud` calls in `business_logic/` — always go through a `TableHelpers::*` class. The "feedback_no_sql_in_business_logic" memory is binding.
- [ ] Sync SQL before any `ThreadPool::Queue` inside a single transaction (the LoginTest race).
- [ ] `ThreadPool::Shutdown()` before the next DB read in tests.
- [ ] Endpoint tests use `crow::query_string` for query params (per memory).

### 6.7 Testing rules (binding per memory)
- [ ] Every helper changed = test added in the same session.
- [ ] Every endpoint changed under `endpoints/` (except `web_app.cpp`) = endpoint test.
- [ ] `ServerAccess.mock.spec.ts` updated when adding `ServerAccess` methods.
- [ ] Component specs whenever components are touched.

---

# 8. Suggested Per-Phase Implementation Documents

Each of these will be its own `.md` in `C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\` when work begins:

- [ ] `Classes Phase 1 - Catalog and Schedule Authoring.md`
- [ ] `Classes Phase 2 - Membership-Gated Drop-In.md`
- [ ] `Classes Phase 3 - Skill Levels.md`
- [ ] `Classes Phase 4 - iCal Generator Extensions.md`
- [ ] `Classes Phase 5 - Attendance Templates.md`
- [ ] `Classes Phase 6 - Weekly Digest.md`
- [ ] `Classes Phase 7 - Class Series and Workshops.md`
- [ ] `Classes Phase 8 - Staff Check-in.md`
- [ ] `Classes Phase 9 - Attendance History.md`
- [ ] `Classes Phase 10 - Scheduling Exceptions and Shift Trades.md`
- [ ] `Classes Phase 11 - Signup Windows and Reminders.md`
- [ ] `Classes Phase 12 - Specialty Instructor Cost.md`
- [ ] `Classes Phase 13 - Tags Filters Favorites.md`
- [ ] `Classes Phase 14 - Reporting Dashboards.md`
- [ ] `Classes Phase 15 - Could Haves Batch.md`
- [ ] `Classes Phase 16 - Stretch Items.md`

---

# 9. Notes on Process

- Plan is intentionally heavy on phase boundaries so each phase can ship and be tested in isolation.
- Layering rule per phase: schema → table helpers → business logic → endpoints → frontend → admin metadata → tests at every layer (tests are not a separate phase).
- When a phase begins, copy its outline into a dedicated `Classes Phase N - <topic>.md` doc in this same directory and add the same numbered subsection structure but with finer-grained subsections + sample SQL / code stubs.
- When in doubt about backwards compat with existing event flow: don't add defensive code, just keep new columns nullable so old paths remain valid (per `feedback_no_premature_defensive_code.md`).
- Plan assumes user will build C++ and run tests; assume user handles git per memory.
- Open questions in §8 should be answered (even with "default") before kicking off the phase that depends on them.
