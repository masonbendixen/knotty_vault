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

**Goal:** "Only auto-confirm me from a waitlist if a spot opens with at least N hours of notice before class start; otherwise leave me off (and eventually drop me)."

**Semantics (precise — the one-line goal above is loosely worded).** `waitlist_auto_confirm_max_hours_before` = the required *advance notice*, in whole hours before a session's start. It carves a **blackout window** out of the final `max_hours` before start. A waitlisted person is auto-promoted **only while `now ≤ start − max_hours·3600e6`**; once `now` crosses into that final window (`now > start − max_hours·3600e6`) the spot has opened with less lead time than they require, so they are **skipped** for promotion and later **dropped** by the expire sweep. `NULL` = no cap = today's behavior (auto-promote at any time before start). This is the exact condition in the plan's business-logic bullet and the only reading consistent with the expire job. (Implemented per resolved OQ-P15-1.)

- [x] DB: extend `user_notification_preferences` (Phase 6 table) with `waitlist_auto_confirm_max_hours_before INT` NULL, **default NULL = no cap (resolved OQ-P15-1)** — preserves today's behavior (auto-promote at any time before start). Only when a user sets a value does the promotion process honor the window. *(`db_schema/user_notification_preferences.h/.cpp` via `AddColumnNullable`; admin metadata in `create_database.cpp` `PopulateAdminColumnDataInfo`/`PopulateAdminColumnFriendlyNames`.)*
- [x] Business logic: extend `BookingHelper`'s waitlist-auto-promote path. When a paid attendee cancels and the next-waitlisted is up, check the preference — if a `waitlist_auto_confirm_max_hours_before` exists and `now > session.start_time_us - max_hours * 3600_000_000`, skip the user and check the next. *(`BookingHelper::CancelBooking` now walks the FIFO waitlist and promotes the first eligible person via new `IsWaitlistPromotionAllowedForPerson`; `CancelBookingResult.waitlistPromotionSkipped` flags a pass-over. Reads the pref through `TableHelpers::UserNotificationPreferences` — no CRUD SQL in business logic.)*
- [x] Add an hourly job: `POST /api/admin/expire_stale_waitlist` — drops users from waitlists for sessions where their `max_hours_before` window has elapsed. Idempotent. *(New endpoint `admin_expire_stale_waitlist.cpp` → `BookingHelper::ExpireStaleWaitlistEntries(tx, nowUs)`; permission `manage_subscriptions`; wired into the scheduler at hourly cadence — `scheduler_config.h`, `scheduler/main.cpp`, `scheduled_job.cpp`.)*
- [x] Frontend: extend Phase 6's preferences page with the new input — defaults to empty / "No cap (auto-confirm any time)" (the NULL default, resolved OQ-P15-1). *(New "Waitlist auto-confirm" card on `/my/notification-preferences`; `waitlistCapOptions` dropdown, null = "No cap". Types, `ServerAccessNetwork` normalizer, and mock updated.)*
- [x] Tests at all layers (incl. NULL = current behavior preserved; a set window skips a too-early promotion). *(Table helper: default-NULL/set/clear. BookingHelper: skip-inside-window-promotes-next, promote-when-window-open, seat-left-open, expire drops/ignores/idempotent. Endpoints: expire 200/401/403/idempotent + prefs GET/PUT set/clear/reject. Frontend: component specs + `ServerAccess.mock.spec.ts`.)*

### 1.1 Manual Testing

There are two surfaces to exercise: the **member-facing preference UI** and the **cap behavior** (skip-on-promote + expire sweep). The cap behavior is time-relative and the server reads the real clock (`now_us()`), so the test-helper commands *set the session start relative to now* to land inside/outside the blackout window.

**A. Preference UI (`ui/`)**

1. Run the frontend: from `...\knottyyoga\ui`, `npx ng serve` (with the C++ server running) — or `npx ng serve -c local` to use the in-memory mock.
2. Log in as any member, then navigate to route **`/my/notification-preferences`** (Account → **Notification Preferences**).
3. In the **"Waitlist auto-confirm"** card, open the **"Only auto-confirm me if a spot opens"** dropdown. Confirm the first option is **"No cap (auto-confirm any time)"** and there are hour options (1, 2, 3, 6, 12, 24, 48, 72).
4. Select **"At least 12 hours before"** and click **"Save preferences"** → the green **"Saved."** indicator appears.
5. Reload the page → the dropdown still reads **"At least 12 hours before"** (persisted).
6. Set it back to **"No cap (auto-confirm any time)"**, Save, reload → confirms clearing (NULL) round-trips.

**B. Cap behavior via the test helper (`knottyyoga_test_helper`)**

Reset/seed the DB first with `knottyyoga_database_helper`, then launch `knottyyoga_test_helper` (dashboard mode). Use the **Events & Bookings** screen, or `:` to drop into command mode / `--repl`. Commands (category *Events & Bookings*):

- `set_waitlist_auto_confirm_cap` (alias `wac`) — `--person_id=<id> --max_hours=<n>` sets a cap; `--person_id=<id> --clear` removes it. (Dashboard: **[c]**.)
- `expire_stale_waitlist` (alias `esw`) — runs the CAP-8 sweep at the current time. (Dashboard: **[e]**.)
- Supporting: `list_event_sessions` (`les`), `list_bookings` (`lb`), `simulate_sold_out_event`, `cancel_booking` (`cb`), `set_event_session_time` (`--session_id --hours_offset`).

**Scenario B1 — capped person is skipped, next person promoted:**
1. Pick a future event session with **capacity 1** (or note its id from `les`; use one with an upcoming start).
2. Book three members onto it through the app/UI so person 1 is **confirmed** and persons 2 and 3 are **waitlisted** (person 2 ahead of person 3). Confirm with `lb --session_id=<sid>`.
3. `wac --person_id=<person2_id> --max_hours=2` — person 2 now requires ≥2h notice. Leave person 3 uncapped.
4. `set_event_session_time --session_id=<sid> --hours_offset=1` — the session now starts in **1 hour**, inside person 2's 2h blackout window.
5. `cb --booking_id=<person1_booking_id>` — cancel person 1's confirmed booking.
6. **Expected:** the command prints "Auto-promoted booking … (person `<person3_id>`)". `lb --session_id=<sid>` shows person 2 still **waitlisted**, person 3 **confirmed**. (Person 2 was skipped because the seat opened <2h before start.)

**Scenario B2 — cap does NOT skip when there's ample notice:**
1. Same setup, but keep the session start far in the future (don't run `set_event_session_time`, or use `--hours_offset=100`).
2. `wac --person_id=<person2_id> --max_hours=2`.
3. `cb --booking_id=<person1_booking_id>` → **person 2 is promoted normally** (window still open).

**Scenario B3 — expire sweep drops a stale entry:**
1. Future capacity-1 session; person 1 confirmed, person 2 waitlisted.
2. `wac --person_id=<person2_id> --max_hours=2`.
3. `set_event_session_time --session_id=<sid> --hours_offset=1` (inside the 2h window).
4. `expire_stale_waitlist` → prints **"Expired 1 stale waitlist entr(ies)."** `lb` shows person 2 now **cancelled**.
5. Run `expire_stale_waitlist` again → **"Expired 0"** (idempotent).
6. Control: repeat with **no** cap set (skip step 2) → sweep reports **0** (a person with no cap is never dropped).

**C. The scheduled job**

The sweep is registered as the hourly `expire_stale_waitlist` job (`POST /api/admin/expire_stale_waitlist`, permission `manage_subscriptions`, run by the `scheduler` service account). To exercise the endpoint directly against a running server, POST to `/api/admin/expire_stale_waitlist` while authenticated as a user with `manage_subscriptions`; the JSON response is `{ "total_expired": <n> }`.

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
- [ ] **Guest is a real account, not a lightweight contact (resolved OQ-P15-2).** When the guest has no existing `people` row, create a **real `people` account** (email required), reusing the **M-9 guest-pass auto-account flow** rather than a name-only contact. So `guest_person_id` always references a genuine account.
- [ ] Business logic: extend `BookingHelper::BookEvent` to accept an optional `guest_person_id` (or `guest_first_name + guest_last_name + guest_email` → creates the real account via the M-9 flow). Validate: the booker has an active multi-seat membership / paid booking that supports the extra seat; the guest is an adult; an email is present when creating a new guest account.
- [ ] Endpoints: extend `POST /api/book_event/<id>` body with the new fields.
- [ ] Frontend: extend the booking-confirmation dialog with a "Add a partner / friend" toggle and form (first / last / email — email required, since the guest gets a real account).
- [ ] Tests at all layers (incl. new-guest creates a real `people` account via the M-9 flow; existing-person path links by `guest_person_id`; adult-only + seat-availability validation; missing-email rejected when creating a new guest).

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

Both resolved (Mason, 2026-06-09) and folded into the plan above (the cited item sections).

- **OQ-P15-1. — RESOLVED (Mason: "go with your recommendation").** CAP-8 `waitlist_auto_confirm_max_hours_before` defaults to NULL (= no cap; today's behavior preserved); only a user-set value gates promotion. Folded into §1.
- **OQ-P15-2. — RESOLVED (Mason: "I would like them to have an actual account").** M-14 guests get a **real `people` account** (email required), created via the M-9 guest-pass auto-account flow — not a name-only contact. Folded into §3.

## 9. Cross-References

- Parent plan: [[Classes, schedules, and attendance]] — §6 Phase 15.
- Predecessors per item: see each subsection.
