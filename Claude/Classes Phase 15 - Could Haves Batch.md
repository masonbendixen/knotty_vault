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

### 1.2 Live-server web walkthrough (from a blank database)

This is a click-by-click run against a real running server + Angular app, using only the seed accounts. It exercises the two observable CAP-8 behaviors: (1) a capped waitlister is **skipped** when a seat opens too close to start, and (2) the **expire sweep** drops them. Because *no bookable sessions are seeded*, step 1 creates one.

**Timing model.** The cap bites only when `now` is inside the final `max_hours` before start. We create a session **~24 h out** and set the tested member's cap to **"At least 48 hours before"** (48 > 24 → inside the window). The 48/24 gap is deliberately wide so it's robust regardless of the exact minute you pick.

**Prerequisites**
- Reset the DB from scratch with `knottyyoga_database_helper`, and start the C++ server (Windows: port 18080) and the Angular app (`npx ng serve` from `...\knottyyoga\ui`, then open http://localhost:4200). Dev `ng serve` uses the real backend via the Angular proxy.
- **Square sandbox must be configured** (`square_environment=sandbox` + a sandbox access token in `config_secrets`) — every booking, including a waitlist join, is a real "Book and Pay". See [[Square credentials and Sandbox setup]]. Sandbox test card: **4111 1111 1111 1111**, any future expiry, any 3-digit CVV, any postal code (e.g. 94103).

**Seed accounts used** (all pre-verified, so no email-verification step; booking the seed event is *not* permission-gated, so any of them can book):

| Role in test | Email | Password |
|---|---|---|
| Admin operator (creates session, runs sweep) | `masonbendixen@gmail.com` | `pass` |
| Member A — confirmed, will cancel | `kiteagle@gmail.com` (Kit) | `kit` |
| Member B — capped waitlister (gets skipped/dropped) | `mr.calebault@gmail.com` (Caleb) | `caleb` |
| Member C — uncapped waitlister (optional 3-person variant) | `masonbendixen@hotmail.com` | `mason` |

Use separate browsers / private windows per member so sessions don't collide.

**Part 1 — Admin creates a capacity-1 session ~24 h out**
1. Log in as the **admin operator** (`masonbendixen@gmail.com` / `pass`) at **`/login`**.
2. Top nav **Admin → Manage Products** (→ `/manage`). Click the **"Event Sessions"** card (→ `/manage/events`).
3. Click **"Create Event Session"** (→ `/manage/events/create`). Fill the form:
   - **Product**: `Intro Workshop` (the seed event product, $9).
   - **Facility**: `Knotty Yoga Studio`; **Room**: `Main Gym` (optional but fine to set).
   - **Capacity**: `1`.
   - **Recurring sessions**: leave the toggle **OFF**.
   - **Date**: **tomorrow's date**.
   - **Start Time**: `12:00 PM`; **End Time**: `1:00 PM` (any ~1 h block tomorrow works).
   - Leave the visibility checkboxes at their defaults.
4. Click **"Create Session"**. You return to `/manage/events`. On the new session's card, **note its numeric id** (shown on the card / in the **Edit** link `…/event_sessions/edit/<id>`). Call it **`SID`**. The session's booking URL is **`/shop/event/SID`**.

**Part 2 — Fill the seat, then create a waitlister**
5. In Member A's browser, log in as **Kit** (`kiteagle@gmail.com` / `kit`). Go to **`/shop/event/SID`** (or **Services → Upcoming Events** → the Intro Workshop card → the tomorrow session).
6. The button reads **"Book and Pay $9.00"**. Enter sandbox card **4111 1111 1111 1111**, future expiry, CVV `111`, postal `94103`, click it. You should land on **"Booking Confirmed!"**. (Capacity 1 is now full.)
7. In Member B's browser, log in as **Caleb** (`mr.calebault@gmail.com` / `caleb`). Go to **`/shop/event/SID`**. The capacity line now reads **"Sold out — you can join the waitlist"** and the button reads **"Join Waitlist and Pay $9.00"**. Pay with the same sandbox card. You should land on **"You're on the Waitlist!"**.

**Part 3 — Member B sets a cap that the freed seat won't satisfy**
8. Still as **Caleb**, open **`/my/account`** (nav: account dropdown → **Profile**) and click the **"Notifications"** card (→ `/my/notification-preferences`).
9. In the **"Waitlist auto-confirm"** card, set **"Only auto-confirm me if a spot opens"** to **"At least 48 hours before"**, click **"Save preferences"** (green **"Saved."** appears). Caleb now requires 48 h notice, but the session is only ~24 h away.

**Part 4 — Free the seat and observe the skip**
10. In Member A's browser (**Kit**), open **`/my/events`** ("My Bookings"). On the Intro Workshop booking click **"Cancel Booking"**, then **"Yes, Cancel"** (accept whatever refund line is shown).
11. **Verify the skip.** As the **admin operator**, open **`/admin/event-session/SID/attendees`** (type the URL — this page has no menu link). Expected:
    - **Confirmed (0)** — the seat is **left open** (Caleb was skipped because the opening came <48 h before start).
    - **Waitlist (1)** — Caleb is still listed.
    This is the core CAP-8 behavior: a normal build would have auto-promoted Caleb; with the cap, he is passed over.

**Part 5 — Expire sweep drops the stale waitlister (via the test helper)**

The sweep has no web page, so run it from **`knottyyoga_test_helper`**. The helper connects to the same production DB the running server uses, so it acts on exactly the data the web app just created. Any one of these forms works:
- **One-shot:** `knottyyoga_test_helper --command=expire_stale_waitlist`
- **REPL:** `knottyyoga_test_helper --repl`, then type `expire_stale_waitlist` (or the alias `esw`).
- **Dashboard:** `knottyyoga_test_helper`, choose **"Events & Bookings"**, press **`[e]`** (labeled *expire stale waitlist*).

12. Run it. Expected output: **`Expired 1 stale waitlist entr(ies).`**
13. Verify: reload **`/admin/event-session/SID/attendees`** → Caleb now appears under **Cancelled / No-Show**, and **Waitlist (0)**. (Or, still in the helper, run `list_bookings --session_id=SID` / alias `lb` — Caleb's row shows `cancelled`.) Run `expire_stale_waitlist` a second time → **`Expired 0 stale waitlist entr(ies).`** (idempotent).

**Variant V1 — cap does NOT skip when there's enough notice (control).** Repeat Parts 1–4 but in step 9 have Caleb pick **"At least 12 hours before"** (12 < ~24 → outside the blackout window). At step 10, cancelling Kit's booking **auto-promotes Caleb**: `/admin/event-session/SID/attendees` shows **Confirmed (1)** = Caleb, **Waitlist (0)**. The sweep in Part 5 then reports **`{ total_expired: 0 }`** (he's confirmed, not waitlisted).

**Variant V2 — skip B, promote the next eligible (C).** After step 7, also log in as **Member C** (`masonbendixen@hotmail.com` / `mason`), go to **`/shop/event/SID`**, and **"Join Waitlist and Pay"** (C sets no cap). Keep Caleb's 48 h cap from step 9. At step 10, cancelling Kit's booking **skips Caleb and promotes C**: attendees shows **Confirmed (1)** = the `masonbendixen@hotmail.com` account, **Waitlist (1)** = Caleb. Then Part 5 drops Caleb.

**Notes / gotchas**
- The `/admin/…` verification pages require the **admin** role; the seed operator account has it. A `manage_products`-only account can create sessions but will be bounced from `/admin/event-session/…/attendees`.
- Everything except the sweep is a real web page. The sweep has no UI surface, so it's run from `knottyyoga_test_helper` (`expire_stale_waitlist` / `esw` / dashboard **[e]**) — the same endpoint the hourly scheduler job POSTs in production. The helper talks to the same DB as the running server, so no extra wiring is needed.
- Bonus helper commands for this feature: `set_waitlist_auto_confirm_cap` / alias `wac` (`--person_id=<id> --max_hours=<n>`, or `--clear`) sets/clears a member's cap straight in the DB — handy if you'd rather not click through the preferences page in Part 3; and `list_bookings` / `lb` inspects a session's booking statuses.
- The 48/24-hour split keeps `now` firmly inside the blackout window for the whole test; you don't need to race the clock. If you'd rather use minutes, the invariant is simply: **chosen cap (hours) > hours until the session starts** for a skip/drop, and **cap < hours-until-start** for a normal promotion (Variant V1). Available cap choices in the UI are 1, 2, 3, 6, 12, 24, 48, 72 h.

## 2. M-13 Per-Session Price Override

**Goal:** Admin sets a higher / lower price for a single class series / workshop instance (special guest teacher week, holiday discount, sliding scale). Override is tied to the specific session, not the schedule.

- [x] DB: new table `event_session_price_overrides`:
  - `id BIGSERIAL PK`
  - `event_session_id BIGINT NOT NULL REFERENCES event_sessions(id)`
  - `permission_id BIGINT` NULL (NULL = base override)
  - `price_cents BIGINT NOT NULL`
  - `created_us`, `updated_us`
  - `UNIQUE (event_session_id, permission_id)`
  *(`db_schema/event_session_price_overrides.h/.cpp`; registered in `make_database_info.cpp` + `create_database.cpp` (`CreateTables`, admin-top-level + admin-nested allow-sets, table permission, column-data-info, friendly names, display template) + both CMakeLists. Ordered after `event_sessions` **and** `permissions` for the FKs.)*
- [x] Business logic: consult `event_session_price_overrides` for the specific session; if any match the user's permission set, use the lowest of those. Falls back to `product_prices` otherwise. **Correction to the original plan:** the injection point is **`CatalogHelper::ResolvePriceForProduct`**, not `ResolveBestPriceForPerson`. `ResolvePriceForProduct` is the single choke point shared by *both* the display path (`GetProduct` → `EventSessionHelper::ResolveSessionPricing`) and the charge path (`QuoteLineItem` → `GetQuote` → `PurchaseHelper::CreatePurchase` → `BookingHelper::BookEvent`); `ResolveBestPriceForPerson` is a *different* M-5 path not used for event booking. A new optional `eventSessionId` is threaded through `QuoteLineItem`, `GetProduct`, and `ResolvePriceForProduct`; the override is consulted only for product-level (no-variant) prices, and the lowest matching override (held tier or base) **replaces** the `product_prices` result. Resolution lives in `TableHelpers::EventSessionPriceOverrides::GetBestOverrideForSessionPermissions` (mirrors `ProductPrices`' lowest-matching-tier query — no CRUD SQL in business logic).
- [x] Existing bookings are grandfathered (per parent OQ-41) — the override only affects future bookings. *(Automatic: the price is snapshotted onto the `purchase_item` at booking time; changing/adding an override never rewrites an existing purchase. Proven by `BookingHelperTest.SessionPriceOverrideGrandfathersExistingBooking`.)*
- [x] Endpoints: **generic admin CRUD via the blessed `class-requirements-editor` pattern** — no new bespoke endpoints. The authoring surface is the bespoke "Price Overrides" section (below), which calls the generic CRUD REST (`addItemFetchPrimaryKey`/`getFilteredTableRows`/`updateItem`/`deleteItem`) against `event_session_price_overrides`; the Manage Data table editor is never the authoring path (memory `feedback_manage_data_is_debug_only.md`). **Correction to the original plan:** the table permission is **`manage_products`**, not `manage_class_schedule` — it matches its sibling event-session child tables (`event_session_staffing`) and, crucially, the Manage Products event-session card the section lives in, so any admin who can see the card can also save an override (a `manage_class_schedule`-only gate would 403 a pure-`manage_products` admin mid-form).
- [x] Frontend: bespoke **`event-session-price-override-editor`** component embedded as a **"Price Overrides"** expandable section on the (management-only) `event-session-card` — per-tier price rows with inline money inputs (add via a Tier picker + $ price, edit price inline on blur, delete). Base override option hides once one exists; used tiers drop out of the picker. *(New standalone component under `shared/components/event-session-price-override-editor/`; toggled by `[p]`-style "Price Overrides" button on the card. `event_session_price_overrides` seeded in `ServerAccess.mock.ts`.)*
- [x] Admin metadata: `event_session_price_overrides` is registered for Manage Data as a **debug/inspection fallback only** (price column gets the `number`/cents edit type; friendly names + display template set), never the authoring path.
- [x] Tests at all layers. *(Table helper `event_session_price_overrides_test.cpp`: CRUD + resolver base/tier/unheld-tier/empty. `CatalogHelperTest`: base override in `GetProduct`, per-tier only-for-holders, fallback-to-product-prices, override in `GetQuote`. `BookingHelperTest`: booking charges the override, and grandfathering. `EventSessionDetailTest.GetSessionReflectsPriceOverride`: the member-facing quote reflects a base override. Frontend: `event-session-price-override-editor.component.spec.ts` (load/add base+tier/validate/edit/delete/rounding/error) + two `event-session-card` toggle specs.)*

### 2.1 Live-server web walkthrough (from a blank database)

A click-by-click run against a real running server + Angular app, using only the seed accounts. It exercises the three observable M-13 behaviors: (1) a **base** override changes the price a member sees and pays; (2) an **existing** booking is **grandfathered** (keeps its old price); (3) a **per-tier** override applies only to members who hold that tier. Because *no bookable sessions are seeded*, Part 1 creates one.

**Prerequisites**
- Reset the DB from scratch with `knottyyoga_database_helper`, start the C++ server (Windows: port 18080) and the Angular app (`npx ng serve` from `...\knottyyoga\ui`, then open http://localhost:4200). Dev `ng serve` uses the real backend via the Angular proxy.
- **Square sandbox must be configured** (`square_environment=sandbox` + a sandbox access token in `config_secrets`) — every booking is a real "Book and Pay". See [[Square credentials and Sandbox setup]]. Sandbox test card: **4111 1111 1111 1111**, any future expiry, any 3-digit CVV, any postal (e.g. 94103).

**Seed accounts used** (all pre-verified):

| Role in test | Email | Password |
|---|---|---|
| Admin operator (creates session, sets overrides) | `masonbendixen@gmail.com` | `pass` |
| Member A — books at the base override, then is grandfathered | `kiteagle@gmail.com` (Kit) | `kit` |
| Member B — books after a price change | `mr.calebault@gmail.com` (Caleb) | `caleb` |

Use separate browsers / private windows per member so sessions don't collide.

**Part 1 — Admin creates a future session**
1. Log in as the **admin operator** (`masonbendixen@gmail.com` / `pass`) at **`/login`**.
2. Top nav **Admin → Manage Products** (→ `/manage`). Click the **"Event Sessions"** card (→ `/manage/events`).
3. Click **"Create Event Session"** (→ `/manage/events/create`). Fill the form:
   - **Product**: `Intro Workshop` (the seed event product, standard price **$9.00**).
   - **Facility**: `Knotty Yoga Studio`; **Room**: `Main Gym` (optional).
   - **Capacity**: `5`.
   - **Recurring sessions**: leave the toggle **OFF**.
   - **Date**: **tomorrow's date**. **Start Time**: `12:00 PM`; **End Time**: `1:00 PM`.
4. Click **"Create Session"** → back on `/manage/events`. On the new session's card, **note its numeric id** (the **Edit** link is `…/event_sessions/edit/<id>`). Call it **`SID`**; its booking URL is **`/shop/event/SID`**.

**Part 2 — Confirm the standard price, then set a base override**
5. Still as admin on **`/manage/events`**, find the session's card and confirm the price it advertises is the product default (**$9.00**). *(Optional: open `/shop/event/SID` in a private window to see the public "Book and Pay $9.00" button before the override.)*
6. On the session's card, click **"Price Overrides"** to expand the section. It reads *"No overrides — this session uses the product's standard pricing."*
7. In the add row: leave **Tier** = **"All members (base price)"**, type **`25`** in the **Price** ($) field, click **"Add override"**. A row appears: **All members (base price) — $25.00**.

**Part 3 — A member sees and pays the override**
8. In Member A's browser, log in as **Kit** (`kiteagle@gmail.com` / `kit`). Go to **`/shop/event/SID`**. The button now reads **"Book and Pay $25.00"** (the override, not $9.00).
9. Pay with sandbox card **4111 1111 1111 1111**, future expiry, CVV `111`, postal `94103` → **"Booking Confirmed!"**.
10. **Verify what was charged.** As admin, open the session card → **"Payment Info"** (`/manage/events/SID/payments`), or **"Attendees"** → Kit's row → **purchase** — the line total is **$25.00**.

**Part 4 — Grandfathering: change the price, old booking is untouched**
11. As admin on the session card → **"Price Overrides"**, change Kit-era **$25.00** row's **Price** to **`40`** (edit the field, click/tab out of it). The row now shows **$40.00**.
12. In Member B's browser, log in as **Caleb** (`mr.calebault@gmail.com` / `caleb`), go to **`/shop/event/SID`** → the button reads **"Book and Pay $40.00"**. Pay → **"Booking Confirmed!"**.
13. **Verify:** admin → **"Payment Info"** for `SID`. **Kit's** purchase is still **$25.00** (grandfathered — the price change did not touch it); **Caleb's** is **$40.00**. This is the core grandfathering behavior.

**Part 5 — Delete the override (revert to standard)**
14. As admin → **"Price Overrides"**, click the **delete** (trash) icon on the override row. The section returns to *"No overrides…"*.
15. Open `/shop/event/SID` in a private window → the button is back to **"Book and Pay $9.00"** (the product default). Existing bookings (Kit $25, Caleb $40) are unchanged.

**Variant — per-tier override (members-only discount).** M-13 also supports a per-tier override that applies only to holders of a membership permission:
1. As admin → **"Price Overrides"**, in the add row pick a **Tier** other than base (the picker lists the pricing-eligible membership tiers) and a lower **Price**, then **"Add override"**. Keep a base override too (e.g. base **$25**, gold-tier **$15**).
2. A member who **holds** that tier sees/pays **$15** at `/shop/event/SID`; a member who does not sees/pays the base **$25**. (The lowest override among the tiers a member qualifies for wins — exactly like `product_prices`.)
3. To grant a seed member a tier for this check, assign the tier's role to them via the test helper or Manage Users, then re-open the booking page.

**Test-helper shortcuts (optional).** Everything above is doable from the web UI. To set/inspect overrides straight in the DB (the helper talks to the same DB the server uses), launch `knottyyoga_test_helper`, **Events & Bookings** screen (category commands):
- `set_session_price_override` / alias `spo` — `--session_id=SID --price=25` (base) or add `--permission_id=<tierId>` for a tier override. Updates in place if one already exists. (Dashboard: **[p]**.)
- `list_session_price_overrides` / alias `lspo` — `--session_id=SID` lists tier + price. (Dashboard: **[o]**.)
- `clear_session_price_override` / alias `cspo` — `--override_id=<id>` deletes one, or `--session_id=SID` deletes all for the session.

**Notes / gotchas**
- The "Price Overrides" section lives on the **management** event-session card (Manage Products → Event Sessions), gated by `manage_products` — it never appears on the member-facing booking page. The seed admin has the permission.
- Overrides only change **future** bookings; existing purchases keep their snapshotted price. There is intentionally no "reprice existing bookings" action.
- A **base** override (Tier = "All members") replaces the price for everyone; a **tier** override replaces it only for holders of that membership. At most one base override per session (the base option disappears from the picker once set).

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
