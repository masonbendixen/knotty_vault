---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/23/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please use your built in tools for read only operations on the filesystem or just say yes but do NOT prompt me when performing work that only reads the filesystem. I want you to run to completion (putting questions to be answered) but DO NOT FUCKING PROMPT ME. Please leave this Overview alone and build the plan in the following sections.

Classes Phase 15 - Could Haves Batch

Please use the code base and the document Classes, schedules, and attendance.md. Please use these documents and the code base for context as well:

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

For each document, please extract the corresponding section from [[Classes, schedules, and attendance]] and place the information from that section of the document here as well as expanded details for the implementation as well as checkboxes to mark off as we complete them. For each piece, please start with the lowest layer of the system moving to higher layers on the server (db schema, table helpers, other helpers, business logic, and endpoints) and then client side work. Make sure to test everything that you can possibly test especially changes to existing files with tests.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here

## Phase Summary

**Could-have batch.** Small features that improve quality of life or operations but aren't differentiators. Each item below is independent — implement as time permits, in any order. Each gets its own implementation sub-section here (rather than a separate doc) because they're individually small.

Items included from the parent plan §5.4:
- **CAP-8** Waitlist auto-confirm cap (user preference).
- **M-13** Per-session price override (admin sets unusual price for one instance).
- **M-14** Partner / friend multi-attendee booking.
- **AR-4** Open-seat heatmap.
- **AR-5** Refund effectiveness report.
- **SL-9** Skill assignment history UI (table can already capture from day 1; UI surface).

**Prerequisites:** all earlier phases as appropriate per item.

## Layering & Conventions

Lowest layer first per item; each item respects the standard layering and tests at every level.

## 1. CAP-8 Waitlist Auto-Confirm Cap

**Goal:** "Auto-confirm me if a spot opens within N hours of class start, otherwise drop me from the waitlist."

- [ ] DB: extend `user_notification_preferences` (Phase 6 table) with `waitlist_auto_confirm_max_hours_before INT` NULL — if set, the waitlist promotion process honors it.
- [ ] Business logic: extend `BookingHelper`'s waitlist-auto-promote path. When a paid attendee cancels and the next-waitlisted is up, check the preference — if a `waitlist_auto_confirm_max_hours_before` exists and `now > session.start_time_us - max_hours * 3600_000_000`, skip the user and check the next.
- [ ] Add an hourly job: `POST /api/admin/expire_stale_waitlist` — drops users from waitlists for sessions where their `max_hours_before` window has elapsed. Idempotent.
- [ ] Frontend: extend Phase 6's preferences page with the new input.
- [ ] Tests at all layers.

## 2. M-13 Per-Session Price Override

**Goal:** Admin sets a higher / lower price for a single class series / workshop instance (special guest teacher week, holiday discount, sliding scale). Override is tied to the specific session, not the schedule.

- [ ] DB: new table `event_session_price_overrides`:
  - `id BIGSERIAL PK`
  - `event_session_id BIGINT NOT NULL REFERENCES event_sessions(id)`
  - `permission_id BIGINT` NULL (NULL = base override)
  - `price_cents BIGINT NOT NULL`
  - `created_us`, `updated_us`
  - `UNIQUE (event_session_id, permission_id)`
- [ ] Business logic: extend `CatalogHelper::ResolveBestPriceForPerson` to first look up `event_session_price_overrides` for the specific session; if any match the user's permission set, use the lowest of those. Falls back to `product_prices` otherwise.
- [ ] Existing bookings are grandfathered (per parent OQ-41) — the override only affects future bookings.
- [ ] Endpoints: **bespoke admin CRUD, NOT the Manage Data generic editor** (memory `feedback_manage_data_is_debug_only.md`). Setting a one-off session price is a real admin workflow and is done from the §below "Price override" section on event-session-detail. Add `GET /api/admin/event_session/<id>/price_overrides`, `POST /api/admin/event_session_price_override`, `PUT`/`DELETE /api/admin/event_session_price_override/<id>` (permission `manage_class_schedule`; thin handlers → table helper → KeyValueTable). (These MAY be backed by the generic CRUD REST endpoints called from the bespoke section — the accepted Phase 1 `class-requirements-editor` pattern — but the authoring surface is the bespoke section, never the Manage Data table editor.)
- [ ] Frontend: extend the admin event-session-detail page with a bespoke "Price override" section (per-tier price rows: add/edit/delete inline, money inputs) — this is the workflow; do not send admins to Manage Data.
- [ ] Admin metadata: registering `event_session_price_overrides` for Manage Data is a debug/inspection fallback only (money column gets the cents edit type), never the authoring path.
- [ ] Tests at all layers (incl. the bespoke endpoints' 403 / validation / persist, the price-override section spec, and the `ServerAccess.mock.spec.ts` cases).

## 3. M-14 Partner / Friend Multi-Attendee Booking

**Goal:** An active member books a single additional adult (partner / friend) for a session using a multi-seat entitlement. Explicitly NOT for children.

- [ ] DB: reuse the existing multi-seat entitlement model (`entitlements.seats_total > 1`). Add `bookings.guest_person_id BIGINT` NULL `REFERENCES people(id)` — when set, indicates the primary attendee booked a guest under the same booking.
- [ ] Or alternatively: create a separate `booking` row for the guest tied to the same `purchase_id`. This is cleaner. Recommend separate bookings.
- [ ] Business logic: extend `BookingHelper::BookEvent` to accept an optional `guest_person_id` (or `guest_first_name + guest_last_name + guest_email` for new-person creation). Validate: the booker has an active multi-seat membership / paid booking that supports the extra seat; the guest is an adult.
- [ ] Endpoints: extend `POST /api/book_event/<id>` body with the new fields.
- [ ] Frontend: extend the booking-confirmation dialog with a "Add a partner / friend" toggle and form.
- [ ] Tests at all layers.

## 4. AR-4 Open-Seat Heatmap

**Goal:** Visual report showing which time slots are routinely underfilling — candidates for cuts or marketing.

- [ ] Business logic: `GetOpenSeatHeatmap(Transaction&, facilityId, weeksBack)` returns per (day-of-week, hour-of-day) cell: average fill rate over the last N weeks.
- [ ] Endpoint: `GET /api/admin/open_seat_heatmap?facility_id=&weeks_back=`.
- [ ] Frontend: heatmap component (CSS grid with color intensity scaled by fill rate, red = empty, green = full).
- [ ] Tests.

## 5. AR-5 Refund Effectiveness Report

**Goal:** Report on how many refunds happened under the no-refund policy vs as voucher / case-by-case grants.

- [ ] Business logic: `GetRefundEffectivenessReport(Transaction&, from, to)`:
  - Total user-initiated cancellations.
  - Total admin-issued vouchers (BC-6 / D-3) and total dollars.
  - Total admin-cancelled-session refunds and total dollars.
  - Total auto-cancelled-series refunds and total dollars.
- [ ] Endpoint: `GET /api/admin/refund_effectiveness?from=&to=`. Permission `admin`.
- [ ] Frontend: simple table view.
- [ ] Tests.

## 6. SL-9 Skill Assignment History UI

**Goal:** Surface the audit trail of who got which skill, who assigned it, who revoked it.

- [ ] Business logic: extend `SkillLevelHelper::GetPersonSkills` to include the revocation history (rows where `removed_us IS NOT NULL`).
- [ ] Endpoint: extend `GET /api/admin/person/<id>/skills?include_history=true`.
- [ ] Frontend: in the staff portal person-skills page (Phase 3), add a collapsible "History" section listing revoked + reassigned entries with timestamps + actor.
- [ ] Tests.

## 7. Tests-Required Summary

Each item above includes tests at all layers (table helpers, business logic, endpoint, frontend specs).

## 8. Open Questions

- **OQ-P15-1.** For CAP-8, what's the default `waitlist_auto_confirm_max_hours_before`? Recommended: NULL (= no cap; current behavior preserved).
- **OQ-P15-2.** For M-14, should `guest_person_id` create a real `people` row (auto-account) or be a "lightweight contact" (name only)? Recommended: real `people` row; mirrors the guest-pass flow from M-9.

## 9. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 15.
- Predecessors per item: see each subsection.
