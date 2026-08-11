---
fileClass: Reference
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/4/2026
Version: 0.7
tags: 
---

# Component Inventory for Designer

## What this document is for

This is a companion to the **Website Makeover** plan. The plan asks you (the designer) to name the things you build in Figma so the engineer can map them 1:1 to existing code. This document tells you what names to use, without you needing to read any code.

It has two lists:

1. **Reusable building blocks.** Things that appear on many pages — a button, a card, a text input, a badge, etc. Each of these should be a **Figma Component** in your `Foundations` file, with variants for state and size. You design each one *once*, then drop instances into screens.
2. **Screens.** Specific pages on the site. Each of these is a **Figma Frame** (or a set of frames — desktop, tablet, mobile) in your `Screens` file, built by composing the building blocks above. Screens are *not* Figma Components.

Some practical guidance before the lists:

- **Use the bold "Figma name" from this doc verbatim** when you name your Figma components and frames. That's the magic that turns the engineer's migration work into a mechanical mapping. (If a name feels wrong to you, change it here and let the engineer know — the goal is that the names match *somewhere*, not that they match this draft of the doc.)
- **You don't need to design every screen separately.** A huge chunk of the studio-operations / admin / staff pages are "a list of things in a data table" or "a form on a card". If you design one beautiful list page and one beautiful form page, the engineer can apply those patterns to the other 50 list/form pages mechanically.
- **You don't need to design every building block.** The list below is exhaustive so the engineer doesn't surprise you with a missing piece during migration. If you only have time for the top ten or so, prioritise the ones marked **★** — those are visible everywhere.
- **The "Why this matters" lines under each building block** explain where it currently shows up in the live site, so you can decide how much polish each one deserves.
- **Mobile is the primary canvas.** Most of this site's traffic — customers browsing classes, booking events, paying, and instructors doing in-studio attendee check-in — happens on phones. Start every screen on a **375-wide mobile frame**, design it well, then expand outward to desktop. Don't build a desktop layout and squeeze it down.

## ⚡ What changed since your first pass (updated 7/19/2026)

You've already mocked up a lot of the site — thank you! Since those mockups, Mason shipped a large wave of class-scheduling features, so the app grew underneath you. This section is your **delta punch list**: everything new or changed relative to the site you designed against. Markers used through the rest of this doc: **🆕** = added since your first pass, **✏️** = existed before but changed enough to revisit, **🚫** = explicitly not being designed.

### Scope decisions locked in with Mason (7/19/2026)

- **No tablet frames.** Design 375 (mobile) + 1280 (desktop) only — everywhere, including the calendar and data tables.
- **The back office is NOT being redesigned.** Skip everything in the Manage portal, Staff portal, and Admin sections of Part 2 — those screens inherit your building blocks automatically. (One optional exception: the staff check-in screens — see the note in the Staff section.)
- **Calendar on mobile defaults to day view** with swipe-between-days; week/month are desktop views.
- **No motion / animation deliverables** — static designs are fine; Material's stock transitions stay.
- (Reconfirmed from earlier: hamburger stays as the mobile nav, Material Icons stay, Figma Variables with the two-layer token scheme, dark mode in scope.)

### New screens to design (all customer-facing)

In priority order — the first three are the big ones:

1. **🆕 Public / Our Schedule** (`/schedule`) — the public weekly class schedule. Likely one of the highest-traffic pages on the site.
2. **✏️ Calendar / Home** (`/calendar`) — completely rebuilt: now public, real data, month/week/day views, a My Schedule toggle, and a rich colour-coded status system. Treat it as a **new design**, not a touch-up.
3. **🆕 Shop / Series Booking** (`/shop/series/…`) — the checkout flow for multi-session series and workshops.
4. **🆕 Public / Instructor Detail** (`/instructors/:id`) — instructor profile with favourite heart, class chips, upcoming sessions.
5. **🆕 Seven new Account pages** — My Skills, My Schedule, Today's Classes, Workshops & Series, Notification Preferences, Attendance History, Favorite Instructors. All described in Part 2.

### Existing screens that changed under your mockups

- **✏️ Public / Classes** (`/classes`) — gained a tag-filter chip row; cards now carry coloured tag chips, "Series"/"Workshop" badges, and an upcoming-session count.
- **✏️ Public / Class Detail** (`/classes/:id`) — grew three sections: viewer-aware **Pricing** ("Included with your membership" / "$X per session" / "Members only"), **Required Skills** (chips + met/missing banners), and **Series Runs** (buy-full-series / prorated-join cards); the Upcoming Sessions list is now bookable for workshops.
- **✏️ Public / Instructors** (`/instructors`) — cards gained a favourite heart (logged-in) and names now link to the new profile page.
- **✏️ Account / My Events** ("My Bookings") — series bookings group into expandable rollup panels; a multi-step inline cancel flow (refund-policy check → no-refund gate → "refund as store credit" checkbox → confirm); new "Bundled with" chip and provider-change notice.
- **✏️ Account / Home** — the card grid grew to 15 navigation cards (seven new sections).
- **✏️ Shop / Event Booking** — gained a "bring a partner or friend" guest fieldset (guest name/email + adult checkbox; the total and pay button double).
- **✏️ Header** — logged-in users now also get **Your Calendar**, **Staff**, and **Admin** menu entries; "Our Classes" leads with "Our Schedule"; the account dropdown grew. The cart badge currently only renders on desktop — your design should give mobile a cart affordance too.
- **Public / Provider Bio** (`/providers/:personId`) — heads-up: this is currently just a "coming soon" placeholder card. Low priority; the real instructor profile is Instructor Detail.

### New building blocks (added to Part 1 below, marked 🆕)

The high-leverage new atoms, roughly in order of how often they appear: **Tag Chip** + **Filter Chip Row**, **Calendar Event Chip / Card** (the calendar's repeating unit — the single biggest design item in this batch), **Kind Badge** (Series/Workshop), **Favorite Toggle**, **Skill Chip + Prerequisite Banner**, **Attendance Status Chip**, **Booking Status Chip** vocabulary, **Signup-Window Indicator + Reminder Button**, **Attendance Plan Row**, **Segmented Toggle**, **Week Navigator**, **Eligible-Slot Checkbox Card**, **Series Run / Summary / Rollup** trio, **Coupon & Voucher Panel**, **Guest Booking Fieldset**, **Skill Badge Card**, **Skill Requirement Dialog**, **Inline Cancel Flow**. Full descriptions in Part 1.

### Not your problem (explicitly out of scope)

- **Everything under Manage / Staff / Admin** in Part 2 — including all the new authoring pages and reporting dashboards (schedule grid, enrollment trends, heatmaps) that shipped in the same wave. They're listed there only as an engineering checklist, marked 🚫.
- The old `/staff` "staff works!" stub — its route was deleted; the note about it is gone from Part 2.

## A note on mobile

A few constraints to keep top-of-mind across every building block and screen:

- **Tap targets ≥ 44×44pt.** Apple's minimum; Google's is 48dp. Buttons, icon buttons, list-row tap zones, form-field tap zones — all of them. If a target is smaller, it's a defect.
- **No hover-only affordances.** If something is only visible or interactive on hover, it doesn't exist on a phone. Every hover state needs an equivalent persistent / tap state for touch.
- **Input font size ≥ 16px on mobile.** Smaller and iOS auto-zooms the page on focus, which is jarring.
- **Mobile-only components.** Some of the building blocks below only appear on mobile (`Sticky Bottom Action Bar`, `Bottom Sheet`). Design those at 375 width only; don't worry about a desktop version. The `Mobile Menu Drawer` is also mobile-only — it's the slide-out triggered by the hamburger button.
- **Safe-area insets.** The persistent top header and any sticky bottom bar need extra padding to dodge the iPhone notch (top) and the iOS home-indicator strip (bottom). Show this in the mobile frames so the engineer doesn't have to guess.
- **Edge states matter more on mobile.** Long names that wrap to two lines, empty lists, slow-loading carousels — design these explicitly because mobile has less horizontal room to absorb them.

---

## Part 1 — Reusable building blocks (design as Figma Components)

### Foundational form controls

These are the atoms — every form in the studio uses some combination of these.

- **★ Button** — every "Sign in", "Save", "Book Now", "Cancel", "Add to Cart". Variants needed: kind (primary, secondary, tertiary/ghost, destructive), size (small / medium / large), state (default, hover, active/pressed, focus, disabled, loading), icon (none, leading, trailing, icon-only). *Where it lives in code: Angular Material `mat-raised-button`, `mat-stroked-button`, `mat-icon-button`.*
- **★ Text Input** — single-line text/email/password fields with a floating label. Variants: state (default, focus, filled, disabled, error), prefix icon, suffix icon, helper text, error message text. *Code name: `simple-text` (also `mat-form-field` from Angular Material.)*
- **Long Text Input** — multi-line textarea. Same variants as Text Input but taller. *Code name: `long-text`.*
- **Checkbox** — single boolean toggle. *Code name: `simple-bool`.*
- **Dropdown / Select** — pick one option from a fixed list. *Code name: `simple-enum`.*
- **Date Picker** — pick a single date with a calendar popover. *Code name: `simple-date`.*
- **Date Strip / Week Navigator** — a horizontal row of day buttons with prev/next-week chevrons, used to pick an appointment date when booking a one-on-one service. Variants: state per day (selected / available / unavailable / disabled). *Currently bespoke to the service-booking page (the "week navigator" markup inside `service-booking.component`); not the same as the Material `Date Picker` above — candidate for promotion to a shared component.*
- **Foreign-Key Picker** — a searchable picker for choosing a related record from the database (e.g. "select a product" or "select a customer"). Looks like a text input with a typeahead dropdown. *Code name: `fk-picker`.*
- **Photo Upload Control** — drag-and-drop or click-to-upload, with thumbnail preview and crop affordance. *Code name: `photo-upload`.*
- **Payment Method / Card Picker** — list of saved credit cards with radio selection, "add new card" button, and "remove" action per card. *Code names: `payment-method`, `card-picker`.*

### Surfaces and containers

- **★ Card / Surface** — the bordered, white-background panel used to group content on almost every page in the app. Variants: padding (small / medium / large), interactive (a hover-shadow state for cards that are clickable), tint (neutral, success, warn, danger). *Code name: `mat-card`. This is the #1 most-duplicated styling pattern in the codebase — get this one right and 65+ pages immediately look more consistent.*
- **Alert Card** — a tinted variant of Card used for "something needs your attention" messages. Variants: tone (info / success / warn / danger), dismissible (yes / no).
- **Page Header / Back Nav** — the title bar at the top of a sub-page: back-arrow link, page title, optional action button on the right. Almost every account / admin / manage page wants one of these. Currently bespoke per page.
- **Section Header** — sub-heading within a page that groups several cards together (e.g. "Alerts", "Upcoming Events", "Your Bookings"). Smaller than the page title, larger than a card title.
- **Empty State** — the "no items yet" placeholder shown when a list is empty: large grey icon, short message, optional call-to-action button. Currently re-implemented bespoke on every list page.

### Status and labelling

- **★ Badge / Pill** — small coloured label for status (e.g. "Waitlisted", "Active", "Inactive", "Applied", "30% off"). Variants: tone (success / warn / danger / info / neutral), size (small / medium). Currently re-implemented bespoke ~10+ times across the codebase (`.status-badge`, `.role-badge`, `.applied-badge`, `.bundle-savings-tag`, etc.).
- **Avatar** — circular profile image, falling back to initials or a placeholder icon. Variants: size (xs / sm / md / lg), state (image, initials, placeholder).
- **Room Occupancy Badge** — tiny indicator showing how full a studio room is. *Code name: `room-occupancy-badge`.*
- **🆕 ★ Tag Chip** — small coloured chip named after a class tag; the colour comes from the database per tag, so the text must auto-contrast (black/white) against any colour. Appears on class-catalog cards, the class-detail page, and calendar entries (full chips on the day card; a colour dot + left-edge accent on the compact week/month pills). *Code: tag chips in `class-info`, `class-detail`, `calendar-event`.*
- **🆕 ★ Filter Chip Row** — horizontal single-select row of pill buttons: an "All" chip plus one per option, selected state highlighted. Used for tag filtering on the class catalog and facility filtering on My Schedule. *Code: tag filter row in `class-info`; facility chips in `my-schedule`.*
- **🆕 Kind Badge** — small pill distinguishing **"Series"** and **"Workshop"** classes (ordinary recurring classes get no badge). Next to class names on catalog cards and as the kind chip on Workshops & Series offering cards.
- **🆕 Skill Chip + Prerequisite Banner** — one chip per required skill on class detail. Logged-out: plain chips. Logged-in: each chip carries a held (green check) or missing (red ✕) icon, plus a banner underneath — green "You meet the prerequisites" or red "You're missing: {skills}" with a talk-to-staff prompt.
- **🆕 Attendance Status Chip** — colour-coded chip on attendance-history rows: green **"Attended"**, red **"No-show"**, grey **"Cancelled"**; plus a small grey **"walk-in"** tag variant beside class names.
- **🆕 Booking Status Chip** — the status vocabulary on booking cards and calendar entries: **Confirmed · Attended · Cancelled · Waitlisted · No Show** (bookings) and **Booked · Sold out · "Happening Now!"** (calendar). Today these are ad-hoc colours — fold them into the Badge/Pill tone system so success/warn/danger/info map consistently.
- **🆕 Favorite Toggle** — heart icon toggle (outline ⇄ filled red), tooltips "Add to favorites" / "Remove from favorites"; rendered only when logged in. On instructor directory cards and the instructor-detail hero. Needs a ≥44px tap target.
- **🆕 Signup-Window Indicator + Reminder Button** — for workshops/series whose sign-ups aren't open yet: "Sign-ups open {date}" beside an outlined bell button **"Remind me when sign-ups open"** that flips to a static **"We'll remind you"** confirmation; when open, a green **"Sign-ups are open"** indicator instead. On the Workshops & Series page and calendar event cards. *Code name: `signup-reminder-button`.*
- **🆕 Substitute Note** — "(Substituting for {name})" inline note on the instructor line wherever a class shows its teacher (Our Schedule, Today's Classes, calendar event cards).
- **🆕 Schedule-Keeper Badges** — the two warning states on My Schedule's "Kept on your schedule" list: **"No longer eligible"** and **"This time slot no longer exists — pick a new one"**.

### Navigation and layout

- **★ Header / Top Nav Bar** — the persistent top bar across every page. Logo on the left; menu on the right. Variants needed: logged-out (login + register), logged-in (account dropdown with avatar), with-cart-badge (shows a red circle on the cart icon when there are items), mobile (hamburger collapsed). On mobile the header should also accommodate a safe-area inset for the iPhone notch. *Code name: `header.component`.*
    - ✏️ *Updated 7/19/2026 — the live menu grew.* Logged-in users also get **Your Calendar**, **Staff** (staff only), and **Admin** (admins/managers only) entries, and the "Our Classes" dropdown now leads with **Our Schedule**. The account dropdown currently holds Profile / My Purchases / My Bookings / Memberships / My Subscriptions / Sign Out (two dead items, "My Goals" and "My Classes", point at deleted routes and will be removed — don't design them). Also note: the cart badge currently renders **only in the desktop bar** — please give the mobile design a cart affordance too (header icon or a drawer row); the engineer will match it.
- **★ Footer** — persistent bottom bar with address, email, social-media icons, mailing-list signup. One variant is fine. On mobile, contend with the iOS home-indicator safe-area inset at the bottom. *Code name: `footer.component`.*
- **★ Mobile Menu Drawer** — slide-out menu that appears when the mobile hamburger is tapped. This is the *primary* mobile navigation pattern for the site (Mason decided against a bottom tab bar — Open Question 15 in the makeover plan). Variants: logged-out vs. logged-in (different menu items), with/without account-management subsection. *Code name: `header-mobile-menu`.*
- **★ Sticky Bottom Action Bar** *(mobile-only)* — fixed bar at the bottom of the viewport that holds the primary CTA for the current screen (e.g. "Pay $40", "Confirm Booking", "Checkout"). Should never overlap content above it. Has an iOS home-indicator safe-area inset at the bottom. Variants: single-button, button + secondary action, button + summary text on the left (e.g. "$40 total" on the left + "Pay" button on the right). *Currently doesn't exist — Phase 3 of the makeover plan introduces it.*
- **★ Bottom Sheet** *(mobile pattern for modals and pickers)* — full-width sheet that slides up from the bottom on mobile in place of a centered modal. Used for: confirmation dialogs, date/time pickers, "select an option" choosers. Variants: short (auto-height), tall (90% of viewport), with drag handle. On desktop the same content can render as a centered modal — the bottom-sheet is just the mobile shape.
- **Pagination Controls** — page-size selector + "page X of Y" + previous/next buttons. Used on every list/table page. *Code name: part of `table-view-control`.*

### Feedback

- **Modal / Dialog** — centred popover with title, body content, and a row of action buttons at the bottom. Used for confirmations ("Are you sure?"), forms, and warnings. *Code name: `confirm-dialog` (and Angular Material's `mat-dialog`).*
- **Toast / Snackbar** — bottom-anchored transient notification, auto-dismissing. Variants: tone (success / warn / danger / info).
- **Spinner / Loading Indicator** — circular spinner for in-flight requests. Variants: size (small / medium / large), colour (dark on light, light on dark). *Code name: `mat-spinner`.*
- **Tooltip** — small popover with explanatory text on hover. (Angular Material `matTooltip`.)

### Domain-specific cards / widgets

These are bespoke to this studio app and only appear in a few places, but they're complex enough to deserve dedicated component status.

- **★ Event Session Card** — the repeating tile that shows one event session: date/time, capacity, optional attendee list, optional staff list, action buttons (book / cancel / promote-from-waitlist / etc.). Variants: with/without attendee section, with/without staff section, public-view vs admin-view. Used on instructor schedule pages, admin attendee pages, my-events page, and others. *Code name: `event-session-card`.*
- **Seat Assignment Widget** — UI for assigning specific seats or spots to attendees of an event. *Code name: `seat-assignment`.*
- **Book-For Selector** — chooser that asks "which member of your household is this booking for?", shown in cart/checkout flows. *Code name: `book-for-selector`.*
- **Photo Carousel** — auto-advancing hero image carousel used on the home page. *Code name: `photo-carousel`.*
- **★ Data Table / List** — bordered table with sortable column headers, hover rows, a trailing actions column. The backbone of every admin / manage / staff list page. **On mobile, every data table needs an explicit responsive strategy** — pick one of: (a) horizontal scroll with a sticky first column (best for admin/manage tables where every cell matters); (b) collapse rows into stacked cards (best for customer-facing tables like `my-events` and `purchase-history`); (c) collapse columns into a summary row with tap-to-expand. Please show the mobile rendition for at least one example of each pattern. Variants: with/without pagination, with/without bulk-select. *Code name: `table-view-control`.*
- **Native Payment Buttons** *(checkout-only, mobile-primary)* — Apple Pay and Google Pay buttons rendered on the mobile checkout page above the manual card form. Square's Web SDK provides the official button artwork — please mock them in their native styling (don't restyle). Variants: Apple Pay (only on iOS Safari), Google Pay (cross-platform), both, neither. *Currently not implemented — Phase 3 of the makeover plan adds them.*
- **🆕 ★ Calendar Event Chip / Card** — the calendar's core repeating unit, in **three densities**: **month** = compact pill (leading status icon, optional tag colour dot, title, start time; colour-coded blue default / green booked / yellow waitlisted / dimmed when locked or cancelled; struck-through title when cancelled); **week** = time-positioned block (title + start–end); **day** = full card with tag chips, date / time / location lines, ONE status badge by priority (Cancelled → Booked or Waitlisted → Sold out → members-only lock), a red **"Happening Now!"** pill during the session, an optional "Sign-ups open {date}" + Remind-me row, an attendance checkbox row ("I'll be there" ⇄ "Tap to plan attendance"), and the substitute note. **The single biggest design item in the July batch — please design all three densities plus the status matrix.** *Code name: `calendar-event`.*
- **🆕 Tag Colour Legend** — the swatch + tag-name legend row in the calendar toolbar.
- **🆕 Segmented Toggle** — two-option button-toggle ("My Schedule" / "Full Schedule") in the calendar toolbar, logged-in only. Worth designing as a generic segmented-control primitive.
- **🆕 Week Navigator** — Our Schedule's header cluster: "Prev" / "This week" / "Next" outlined buttons. (Different from the service-booking Date Strip — this one steps whole weeks.)
- **🆕 Eligible-Slot Checkbox Card** — My Schedule's weekly-plan unit: a tappable card with a checkbox icon, class name, "time · duration", facility; checked = on-my-plan state. Seven day columns on desktop; stacks on mobile.
- **🆕 Attendance Plan Row** — the Today's Classes / Upcoming-list row: class info plus one toggle button — filled-primary **"I'll be there"** ⇄ outlined-warn **"I can't make it"**. Attending state = green check + tinted row; skipping state = struck-through + inline "Note: {note}". The can't-make-it tap opens a small **Exception Note dialog** (textarea + Save/Cancel).
- **🆕 Series Run Card** — on class detail: run name, full-series price, "N sessions · {date range}", per-session price, optional minimum-attendees warning, non-refundable note, CTAs ("Buy full series — $X", optional "Join from today — prorated", or "Log in to book").
- **🆕 Series Summary Card** — on the series-booking page: run name, date-range line, "N sessions · $X per session", optional amber "Prorated mid-run join" note, bold "Total due" row, non-refundable note.
- **🆕 Series Rollup Panel** — on My Bookings: an expansion panel titled "{series} · N sessions · {date range}" holding the individual session cards; defaults open.
- **🆕 Coupon & Voucher Panel** — collapsible panel with separate Coupon Code and Voucher Code fields + Apply buttons, green success confirmations ("Coupon {CODE} applied — $X off", "Remaining balance: $Y"), red error boxes, and an **"Applied"** badge in the collapsed header. On the cart and series-booking pages.
- **🆕 Guest Booking Fieldset** — event booking's "Bring a partner or friend" toggle revealing guest first/last/email fields + an adult-attestation checkbox; the total and pay button double when enabled.
- **🆕 Skill Badge Card** — the My Skills achievement-wall unit: badge photo, skill name, "Earned {date}", description, optional staff note.
- **🆕 Skill Requirement Dialog** — modal listing missing prerequisite skills (badge image + name + description) with a "talk to a staff member" prompt and a single "Got it" button. Opens from locked calendar items.
- **🆕 Inline Cancel Flow** — My Bookings' multi-step inline cancellation: refund-policy line ("Checking refund policy…" → refund / no-refund text) → optional no-refund gate ("Yes, Cancel Without Refund" / "No, Keep It") → optional "Refund as store credit instead of card refund" checkbox → final confirm. On mobile this wants to be a bottom sheet.

### 🚫 Back-office-only blocks (no design needed — listed so nothing surprises you)

These shipped in the same July feature wave but live exclusively in the Manage / Staff / Admin portals, which aren't being redesigned. The engineer restyles them with your tokens + Card / Button / Badge / Table primitives. One line each, purely for completeness: nested accordion authoring cards (class → instance → schedule → slot); the class requirements editor (AND-groups of OR'd permission chips + amber skill chips); a native colour-picker input (Manage Tags); the per-session price-override editor; the specialty-instructor-cost section + dialog + per-run cost & revenue block; fill-coloured schedule-grid tiles (green/amber/red) + legend; the CSS weekly bar chart; open-seat heatmap cells (red→green) + tooltips; sortable table headers (▲/▼) + relative-load bars; date-range pickers + facility selectors; close-classes preview chips + per-class outcome list; the class-check-in roster (grouped rows, big checkbox toggles, "Waitlist #{n}" + "Requirements not met" badges, "Check-in open / Window closed" badge, over-capacity toast, walk-in panel, override-reason panel); the exception-notes amber panel; the attendance-template support browser; membership-tier checkboxes; and the admin dialogs (instance/schedule/slot/class forms, series-run form, cancel-occurrence, substitute-instructor, assign/revoke skill).

### Error and not-found

- **404 / Page Not Found** — the screen shown when the user hits a route that doesn't exist. *Code name: `page-not-found`.*

---

## Part 2 — Screens (design as Figma Frames)

Naming convention: use the **bold Figma frame name** from each entry below. For each screen, design a **Desktop (1280)** and **Mobile (375)** frame. **No tablet frames** — Mason confirmed (7/19/2026) that desktop + mobile are enough, everywhere, including the calendar and data tables.

### Public website (no login required)

These are the screens an unauthenticated visitor sees. Highest visibility — they're the marketing surface — so probably worth the most design care.

| Figma frame name | URL | What it shows |
|---|---|---|
| **Public / Home** | `/` (also `/start`) | Hero carousel, brand value-prop, next-upcoming-event card, link to events list. |
| **Public / About** | `/about` | About text and "how to join" section. Currently mostly placeholder. |
| **🆕 Public / Our Schedule** | `/schedule` | The public weekly schedule — likely a top-traffic page. Sun–Sat day sections, each a stack of horizontal class cards: photo, class name, time · duration, instructor line (small round avatar + "with {name}", plus a "(Substituting for {name})" note when applicable), an optional lock-icon requirements blurb, and an optional "Requires attending: {class · day time}" prerequisite line. A "Prev / This week / Next" week navigator up top. **Read-only** — no filters, no prices, no book buttons; every card links to the class detail page. Per-day "No classes" and whole-week empty states. |
| **✏️ Public / Classes** | `/classes` | List of class types ("Our Classes"). Now topped by a tag-filter chip row ("All" + one chip per tag, single-select); each card shows photo, name + Series/Workshop kind badge, description, coloured tag chips, and "{n} upcoming session(s)". |
| **✏️ Public / Class Detail** | `/classes/:id` | Two-column hero (text + photo) with tag chips, then stacked sections: **Pricing** (three viewer-aware states — "Included with your membership / Just show up", "$X per session + no-refund note", "Members only / Upgrade to attend"), **Required Skills** (skill chips; logged-in adds held/missing icons + a green "You meet the prerequisites" or red "You're missing: …" banner), **Series Runs** (series classes only — run cards with prices and Buy-full-series / prorated-join CTAs), and **Upcoming Sessions** (cards with date/time, facility · room, instructors; workshop occurrences are tappable with "Book — $X" / "Reserve a free spot" / "Log in to book" CTAs). |
| **✏️ Public / Instructors** | `/instructors` | Stacked instructor cards — photo, name, bio. Names link to the profile page; a favourite heart toggle sits on each card (logged-in only). |
| **🆕 Public / Instructor Detail** | `/instructors/:id` | Instructor profile: hero card (large photo, name, favourite heart), full bio, a "Classes" row of chips linking to each class they teach, and an "Upcoming sessions" list for the next 4 weeks. Loading / not-found / no-sessions states. |
| **Public / Provider Bio** | `/providers/:personId` | ⚠️ Currently a "coming soon" placeholder card ("Provider profiles with bios and photos are coming soon" + a Browse Our Services button). Low priority — design only if you want to define what it becomes; the real instructor profile is Instructor Detail above. |
| **Public / Upcoming Events** | `/events` | List of bookable upcoming events with dates and prices. |

### Authentication

| Figma frame name | URL | What it shows |
|---|---|---|
| **Auth / Login** | `/login` | Email, password, remember-me, "create an account" link. |
| **Auth / Register** | `/register` | First name, last name, email, password, submit. |
| **Auth / Verify Email** | `/verify` | Confirmation screen shown after the user clicks the link in their welcome email. |

### Shop (browsing + checkout flow)

The conversion funnel. These pages turn visitors into customers, so they deserve careful attention.

| Figma frame name | URL | What it shows |
|---|---|---|
| **Shop / Catalog** | `/shop` | Grid of buyable products (classes, packages, retail). |
| **Shop / Product Detail** | `/shop/:id` | Single product page with description, price, and "add to cart" / "book now". |
| **Shop / Service Catalog** | `/shop/services` | Same as Catalog but specifically for one-on-one services. |
| **Shop / Service Booking** | `/shop/service/:productId` | Pick a date and provider for a one-on-one service. |
| **Shop / Subscription Catalog** | `/shop/subscriptions` | List of available membership plans. |
| **Shop / Subscription Signup** | `/shop/subscribe/:productId` | Sign-up form for a single membership plan. |
| **✏️ Shop / Event Booking** | `/shop/event/:sessionId` | Book a seat at a specific scheduled event or workshop occurrence. Now includes the "Bring a partner or friend" guest fieldset (toggle revealing guest first/last/email + an adult-attestation checkbox; the total and the "Book and Pay $X" button double when a guest is added). |
| **🆕 Shop / Series Booking** | `/shop/series/:classInstanceId` | Checkout for a whole multi-session series: a series summary card (run name, date range, "N sessions · $X per session", optional amber "Prorated mid-run join" note, bold "Total due" row, non-refundable notice), a collapsible "Book for someone else" panel, a collapsible Coupons & Vouchers panel, the Square payment card, and "Book and Pay" + "Add to Cart" buttons. Also design the "You're booked!" success state and the "We lost the series details" error state. |
| **Shop / Cart** | `/shop/cart` | Items in cart, voucher code field, upsell suggestions, totals, "proceed to checkout" button. |
| **Shop / Checkout** | `/shop/checkout/:productId` | Payment form (Square card input), order summary, final "pay" button. |

### My account (logged-in user pages)

| Figma frame name | URL | What it shows |
|---|---|---|
| **✏️ Account / Home** | `/my/account` | Landing page: welcome header (profile photo + "Welcome, {firstName}" + membership label) over a grid of **15 navigation cards** (icon + title + one-liner) — grew by seven sections in 7/2026. |
| **Account / Profile (View)** | `/my/user-information` | Photo, name, email, phone — read-only summary. |
| **Account / Profile (Edit)** | `/my/update-user-info` | Same fields, editable. |
| **Account / Change Password** | `/my/update-user-password` | Current password + new password + confirm. |
| **Account / Purchase History** | `/my/purchases` | List of past orders. |
| **Account / Purchase Detail** | `/my/purchases/:id` | Single order: line items, payment, receipt. |
| **✏️ Account / My Events** | `/my/events` | Titled "My Bookings". Upcoming bookings as cards with status chips (Confirmed / Attended / Cancelled / Waitlisted / No Show); series bookings group into expandable "{series} · N sessions · {date range}" rollup panels; new "Bundled with: {product}" link chip and a provider-change notice; the **multi-step inline cancel flow** (see Part 1); a past-bookings accordion; "Browse events / services" links + empty state. |
| **Account / Saved Cards** | `/my/cards` | Saved credit cards with add/remove. |
| **Account / My Subscriptions** | `/my/subscriptions` | Active membership(s). |
| **Account / Subscription Detail** | `/my/subscriptions/:id` | Single subscription with billing history, pause/cancel actions. |
| **Account / Gift Permissions** | `/my/gift-permissions` | Manage who can book on your behalf (and who you can book for). |
| **Account / My Vouchers** | `/my/vouchers` | Vouchers/credits available to the user. |
| **🆕 Account / My Skills** | `/my/skills` | Read-only achievement wall: a grid of earned skill-badge cards (badge photo, name, "Earned {date}", description, optional staff note) + an empty state pointing at staff evaluation. |
| **🆕 Account / My Schedule** | `/my/my-schedule` | Two tabs. **"My Weekly Plan"**: a Sun–Sat grid of eligible recurring classes as checkbox cards (checked = on my weekly template), optional facility filter chips, and a "Kept on your schedule" list with "No longer eligible" / "time slot no longer exists" badges. **"Upcoming"**: the next 4 weeks grouped by day (sticky day headers) — booked sessions with a green "Booked" badge plus per-occurrence "I'll be there / I can't make it" planning. |
| **🆕 Account / Today's Classes** | `/my/today-classes` | Today's eligible classes as rows: class name, time range, facility · room, instructor (+ substitute note). Attending rows get a green check + highlight; skipped rows go struck-through with the member's note. One toggle button per row: filled "I'll be there" ⇄ outlined-warn "I can't make it" (which opens the note dialog). |
| **🆕 Account / Workshops & Series** | `/my/upcoming-offerings` | Upcoming workshop/series occurrences (~4 months out): linked class name, when, a kind chip, and the sign-up-window state — green "Sign-ups are open", or "Sign-ups open {date}" with the Remind-me button / "We'll remind you" badge. |
| **🆕 Account / Notification Preferences** | `/my/notification-preferences` | Four stacked preference cards: **Weekly digest** (on/off toggle + Send-day and Send-time dropdowns, studio-time hint), **Waitlist auto-confirm** ("Only auto-confirm me if a spot opens" lead-time dropdown), **Subscribe to your calendar** (generated `webcal://` URL with Copy + Generate/Regenerate buttons), **Sign-up reminders** (pending-reminder rows with per-row Cancel). One "Save preferences" action with a "Saved." confirmation. |
| **🆕 Account / Attendance History** | `/my/attendance-history` | Filterable history of every past class: four dropdown filters (Year / Month / Class / Instructor) + a "Show no-shows & cancellations" toggle; a table of date/time, class (+ grey "walk-in" tag), where, instructor, and an Attended / No-show / Cancelled status chip; Previous/Next pager; two empty states (no history vs. no filter matches + Clear filters). |
| **🆕 Account / Favorite Instructors** | `/my/favorite-instructors` | The instructors you follow: horizontal cards (photo, name, bio) each with a Remove button; empty state points at the heart toggle on the Instructors page. |

### Studio operations — Manage portal (for the studio owner / managers)

> 🚫 **Not being redesigned (decided with Mason 7/19/2026).** Skip this whole section — no Figma frames needed. These screens all follow three patterns (**list**, **form**, **dashboard**) built from your Part 1 blocks, so they pick up the new look mechanically during the engineering sweep. The table below stays as an engineering checklist, extended with the twelve pages added since May (marked 🆕).

| Figma frame name | URL | Pattern |
|---|---|---|
| **Manage / Dashboard** | `/manage` | Dashboard (summary cards + alerts) |
| **Manage / Pricing Overview** | `/manage/pricing` | List |
| **Manage / Products** | `/manage/products` | List |
| **Manage / Product / New** | `/manage/products/new` | Form |
| **Manage / Product / Detail** | `/manage/products/:id` | Form (edit) |
| **Manage / Schedules** | `/manage/schedules` | List |
| **Manage / Schedule / New** | `/manage/schedules/new` | Form |
| **Manage / Schedule / Detail** | `/manage/schedules/:id` | Form (edit) |
| **Manage / Events** | `/manage/events` | List |
| **Manage / Event / Create** | `/manage/events/create` | Form |
| **Manage / Event / Payments** | `/manage/events/:sessionId/payments` | List (payments for one event) |
| **Manage / Subscriptions** | `/manage/subscriptions` | List |
| **Manage / Subscription / New** | `/manage/subscriptions/new` | Form |
| **Manage / Subscription / Revenue** | `/manage/subscriptions/revenue` | Dashboard |
| **Manage / Subscription / Detail** | `/manage/subscriptions/:id` | Form (edit) |
| **Manage / Entitlements** | `/manage/entitlements` | List |
| **Manage / Providers** | `/manage/providers` | List |
| **Manage / Provider / Availability** | `/manage/providers/:personId/availability` | Form (calendar) |
| **Manage / Provider / Schedule Template** | `/manage/providers/:personId/templates` | Form (calendar) |
| **Manage / Time-Off Review** | `/manage/time-off` | List with approve/deny actions |
| **Manage / Scheduling Exceptions** | `/manage/scheduling-exceptions` | List with form |
| **Manage / Shift Request Review** | `/manage/shift-requests` | List with approve/deny actions |
| **Manage / Vouchers** | `/manage/vouchers` | List with create-new dialog |
| **Manage / Coupons** | `/manage/coupons` | List with create-new dialog |
| **Manage / Comps** | `/manage/comps` | List with create-new dialog (complimentary entitlements) |
| **Manage / Room Schedules** | `/manage/room-schedules` | Form (calendar) |
| **Manage / Room Occupancy** | `/manage/room-occupancy` | Dashboard |
| **Manage / Bundles** | `/manage/bundles` | List with create-new dialog |
| **🆕 Manage / Class Schedules** | `/manage/class-schedules` | Authoring — nested accordion cards (class → instance → schedule → slot) + form dialogs, requirements editor, substitute-instructor action, specialty-cost + cost/revenue blocks |
| **🆕 Manage / Instructors** | `/manage/instructors` | List with edit panels (bio, photo, class preferences) |
| **🆕 Manage / Membership Tiers** | `/manage/membership-tiers` | Checkbox list (which permissions are customer-facing tiers) |
| **🆕 Manage / Skills** | `/manage/skills` | List + form with badge-photo upload |
| **🆕 Manage / Class Tags** | `/manage/class-tags` | List + form with a native colour picker |
| **🆕 Manage / Attendance Templates** | `/manage/attendance-templates` | Read-only support browser (member search → their weekly template + exceptions) |
| **🆕 Manage / Close Classes** | `/manage/close-classes` | Batch form (multi-select + date range + reason) with preview chips + outcome list |
| **🆕 Manage / Instructor Load** | `/manage/instructor-load` | Report table ("who's teaching what": sortable columns + relative-load bars) |
| **🆕 Manage / Schedule Grid** | `/manage/schedule-grid` | Report — weekly board of class tiles colour-coded by fill (green/amber/red) |
| **🆕 Manage / Enrollment Trend** | `/manage/enrollment-trend` | Report — per-class weekly CSS bar chart |
| **🆕 Manage / Open-Seat Heatmap** | `/manage/open-seat-heatmap` | Report — hour × day heatmap of open seats |
| **🆕 Manage / Refund Effectiveness** | `/manage/refund-effectiveness` | Report — refund categories summary table with totals |

### Studio operations — Staff portal (for instructors)

> 🚫 **Not being redesigned (decided with Mason 7/19/2026)** — same deal as the Manage portal. **One optional exception:** the two check-in screens are the only back-office surfaces used on a phone mid-class, one-handed. If you find spare time, a 375-wide frame for **Staff / Class Check-In** (the roster) would be genuinely valuable; otherwise engineering composes both from your building blocks.

| Figma frame name | URL | Pattern |
|---|---|---|
| **Staff / Dashboard** | `/staff` | Dashboard |
| **Staff / My Sessions** | `/staff/sessions` | List of sessions I'm teaching |
| **Staff / Bookings** | `/staff/bookings` | List of who's booked in to my sessions |
| **Staff / Schedule** | `/staff/schedule` | Calendar |
| **Staff / Time-Off** | `/staff/time-off` | List + form |
| **Staff / Shift Requests** | `/staff/shift-requests` | List + form (class-trade requests get a green "Class" chip) |
| **Staff / Check-In** | `/staff/check-in` | Front-desk console for service/event bookings: room-occupancy chips, time-window search, per-booking Check In, a walk-in *purchase* wizard, paid Extend upgrades |
| **🆕 Staff / Class Check-In** | `/staff/class-checkin` | Class-session roster: today's sessions list → per-session roster grouped by source, big check/uncheck toggles, "Check-in open / Window closed" badge, attended/capacity counter, can't-make-it notes panel, people search + walk-in add, skill-override reason panel |
| **🆕 Staff / Person Skills** | `/staff/person-skills` | Person search → their skills table with Assign / Revoke dialogs + a collapsible grant/revoke History (Active / Revoked badges) |
| **🆕 Staff / Exception Notes** | `/staff/exception-notes` | "Notes from your students" — colour-coded skip vs drop-in note rows for the instructor's classes |
| **Staff / Preferences** | `/staff/preferences` | Form |

### Calendar

> ✏️ **Rebuilt since your first pass — treat as a new design, not a touch-up.** The calendar is now **public**: logged-out visitors see the full studio schedule (recurring classes, workshops, series runs, standalone events); logged-in members land on a **"My Schedule"** view of the things they're eligible for plus their own bookings, with a My Schedule / Full Schedule segmented toggle. Ineligible items show greyed with a lock badge (tapping routes to the membership purchase page or opens the Skill Requirement dialog). Entries are tinted by class-tag colour; a facility dropdown and a tag-colour legend sit in the toolbar. The per-entry status system lives in Part 1 under **Calendar Event Chip / Card**. **Mobile defaults to day view** (decided 7/19/2026); week + month are desktop views.

| Figma frame name | URL | What it shows |
|---|---|---|
| **✏️ Calendar / Home** | `/calendar` | Month/week/day calendar of every class, workshop, series session, and event — plus the member's personal My Schedule view. Most visually-dense page in the app. Design: the toolbar (view-select menu, My/Full toggle, facility dropdown, tag legend), all three views at 1280, the **day view at 375** (the mobile default), and the two dialogs (plan/cancel attendance, skill requirement). Include the filtered-empty state ("Your current filters hide everything." + a Show-full-schedule reset). |

### Admin (database editor)

> 🚫 **Not being redesigned (decided with Mason 7/19/2026).** This was already flagged lowest-priority; now it's confirmed out of scope. It's the catch-all database editor used by the studio owner for things that don't have a dedicated `/manage` page yet — the "good enough" treatment it inherits from the building blocks is the plan.

| Figma frame name | URL | What it shows |
|---|---|---|
| **Admin / Dashboard** | `/admin` | List of editable database tables. |
| **Admin / Table View** | `/admin/tables/:tableName/view/:pageSize/:pageOffset` | Generic data table. |
| **Admin / Table Edit** | `/admin/tables/:tableName/edit/:id` | Generic form. |
| **Admin / Table New** | `/admin/tables/:tableName/new` | Generic form. |
| **Admin / Event Attendees** | `/admin/event-session/:sessionId/attendees` | List of attendees for an event. |
| **Admin / Event Session Staff** | `/admin/event-session/:sessionId/staff` | List of staff assigned to an event. |

---

## Quick wins (if you only have a weekend)

If your time is limited, the single biggest visual upgrade comes from getting these right and letting the rest follow. **Design every one of these on a 375-wide mobile frame first.**

1. **Header / Top Nav Bar** + **Footer** + **Mobile Menu Drawer** — the three persistent shells (header on every page, footer on every page, drawer is the primary mobile navigation). Get these right and the whole site instantly feels more cohesive on every page.
2. **Card / Surface** + **Button** + **Badge** + **Text Input** — the four atoms that compose 90% of every other page.
3. **Public / Home** + **Auth / Login** + **Shop / Catalog** + **Shop / Cart** + **Shop / Checkout** — the highest-traffic / highest-converting public-facing pages. Checkout especially is mobile-dominant — show the sticky bottom Pay button + Apple/Google Pay buttons.
4. **The colour, type, spacing, and radius Figma Variables** (covered in Phase 1.1.A of the makeover plan) — these are the "design tokens" that the engineer will pull straight into the code, so they're the single highest-leverage thing in the whole project even if you do nothing else.
5. **🆕 The July additions with real customer traffic** — **Public / Our Schedule** and the **Calendar Event Chip / Card** (all three densities + the status matrix), then **Tag Chip** + **Filter Chip Row**. These now carry as much visitor traffic as anything in item 3 — see the "What changed since your first pass" section at the top for the full delta.

Everything else can come later.

# Mason - Updates to the document 8/4/2026
- Ryan, the graphics designer has done the Figma for the document and app as things stood before. Since then, I have made the following changes:
	- [[Blog support with markdown]]
	- [[Converting the server to a multi tenant Saas architecture]]
	- [[Componentizing the frontend]]
	- [[Home page work and cleanup items]]
	- Classes Phase XXX (1-16)
- I don't want you to update the stuff that already exists since he has already based his work on that, but I'd like you to add sections below this one for work for these new sections. The changes to the home page, the menu structure, blog support, getting started guide. classes / series / workshops, the calendar, and general cleanup need some new changes. A bigger change is that I want the frontend to support multiple users so it would be nice to abstract the home page images, color scheme, fonts, and a lot of the text for Knotty Yoga, the about page, and getting started so that new users can use the site as is but support a "theme." The more I think about it, it might be better to tackle the theme support as a separate document entirely (but be theme aware in this document). Can you start a plan below this section based on these updates?

---

# ⚡ Delta punch list #2 — what changed while you were in Figma (8/4/2026)

*(Claude, responding to Mason's section above.)* Everything **above** Mason's section is frozen — it's the doc your current Figma pass was built against, so none of it has been rewritten. This section is the complete delta since 7/19, in the same style: **🆕** new, **✏️** existed but changed enough to revisit, **🚫** explicitly not designed. One **new marker: 🎨 = theme-flexible** — the item contains brand content (name, logo, colors, photos, copy) that will differ per studio; the "multi-studio" section right below explains what that means for how you design.

Three waves landed since 7/19:

1. **Navigation & home rework** — the menu flattened to direct links, two public pages traded names and URLs, the home page was recomposed for both auth states, and "Get Started" finally opens a real page.
2. **A blog** — a public reading page with markdown posts and per-post photos (plus back-office authoring, which you skip).
3. **Multi-studio / whitelabel** — the deep one. The same app will soon serve other studios, each with its own branding. It changes *how* you design more than *what* you design.

## The big one first: the site is going multi-studio 🎨

Mason is turning the platform into a product other studios can run. The server side is already multi-tenant, and the frontend already boots by asking the server "who am I?" — the studio's display name, website URL, and logo arrive from an API at load, and the footer's address / contact email / copyright name plus the About text are already centralized behind a config service with Knotty Yoga's values as fallbacks. Next in line: the hero copy, the Getting Started copy, colors, and fonts.

**Decision (Mason, 8/4/2026): full theme support is its own companion document — [[Tenant Theming and Branding]]** (drafted 8/4/2026, per OQ-D2). It fuses Website Makeover Phase 5 (DB-driven theme tokens + bootstrap endpoint + an admin "Site Theme" page — sketched there, now superseded by the theming doc) with the tenancy plan's shipped branding hooks, and it owns the token catalog and the full content-slot inventory. **This document stays the component/screen inventory** and just marks what must flex with 🎨 — the slot table below is the designer-facing summary; the theming doc holds the engineering version with storage keys and defaults.

What it changes about how you design — five rules:

1. **The tokens are the brand.** The Figma Variables layer (makeover Phase 1.1.A) stops being a nicety and becomes the product: another studio is *another set of token values plus different images and text, with zero layout changes*. Build every component off the variables; never hard-code a brand color into a component.
2. **Brand content is a slot, not a fixture.** Treat the logo, studio name, hero headline, tagline, photos, and marketing copy as **content slots with size constraints** (max logo box, name length, headline line count) — Knotty Yoga's actual values are just the sample content filling them.
3. **Type styles by role, not by font name.** D-DIN is a Knotty Yoga token *value*, not part of the system. Name text styles `display / heading / body / caption` so another studio can swap the font family without touching a single frame.
4. **Prove it once with a fake second studio.** One Home frame + header rendered with an invented brand (different palette, different logo shape, longer studio name, different photos) is the cheapest possible proof the layouts don't secretly depend on Knotty Yoga's assets. Worst-case content matters: a wide logo, a two-line studio name, a missing photo.
5. **No brand baked into artwork.** Nothing Knotty-specific lettered into imagery, icons, or decorative elements — those must survive a studio swap untouched.

**Content-slot inventory** (what varies per studio — today's values are the sample content):

| 🎨 Slot | Where it shows | Knotty Yoga sample value |
|---|---|---|
| Studio display name | Header logo alt, footer copyright, page copy, emails | "Knotty Yoga" |
| Logo | Header (light-on-dark SVG today) | `KnottyYoga_logo_white.svg` |
| Secondary hero image | Home hero, right side | The safe-space logo |
| Hero headline + subline | Home | "Knotty Yoga Is An Inclusive, High-Level Acrobatic Fitness Studio." / "Get into the best shape of your life, the right way." |
| Tagline | Footer | "That which doesn't kill you / makes you hotter. 💪🤸" |
| Address + contact email | Footer | Redmond address / info@knottyyoga.com |
| Social links | Footer | Current icon row |
| About page copy | `/about` (and the About ▾ menu label "About Knotty Yoga") | Current blurb |
| Getting Started copy | `/start` | The seven steps' text |
| Gallery / carousel photos | Home | Already DB-driven (`/api/home_page_photos`) |
| Membership tier names + marketing copy | Home, Memberships | "Knotty Yoga Gold …" tiers, "Unlimited classes, priority sign-ups…" |
| Colors, fonts, radius | Everywhere | The token values (the red, D-DIN, …) |

## Menu — flattened (✏️ Header + ✏️ Mobile Menu Drawer)

The 7/19 note ("the 'Our Classes' dropdown now leads with Our Schedule…") is obsolete — the per-class dropdowns are gone entirely. The live model, one refresh pass across your header/drawer frames:

- **Signed out:** `Get Started · About ▾ · Our Classes · Services · Upcoming Events · 🆕 Blog · Memberships · Sign In`
  - **About ▾** = About Knotty Yoga 🎨 (the label carries the studio name) · Instructors · Gallery. The only surviving dropdown for visitors.
  - **Our Classes** and **Services** are now *direct links* — no dropdowns, no per-class entries, no "All Classes" entry (that page is reached from the schedule page instead).
  - **Memberships** replaced the "Shop" label (same target — the shop sells only memberships today).
- **Signed in appends:** `Your Calendar · Staff (staff only) · Admin ▾ · {first name}`
  - **Admin ▾** = Manage Data (admins) · Manage Products (admins + product managers) · **🆕 Blog Posts** (admins + blog authors). For a user whose only elevated permission is blog authoring, the dropdown holds *just* Blog Posts — intended.
  - **Account dropdown** = Profile · My Purchases · My Bookings · My Subscriptions · Sign Out. The dead "My Goals" / "My Classes" are gone, the redundant "Memberships" entry is gone, and the trigger now shows the user's **profile photo as the avatar**.
- The 7/19 ask still stands: give mobile a cart affordance (the badge is desktop-only in code).

## Two pages traded names — fix the frames before anything else

| Your 7/19 frame | What happened | New Figma name | What to do |
|---|---|---|---|
| **Public / Our Schedule** (`/schedule`) | Renamed **"Our Classes"**, moved to `/classes` (`/schedule` redirects), and **rebuilt** — no longer read-only | **Public / Our Classes** | Rename the frame, then apply the ✏️ redesign below |
| **Public / Classes** (`/classes`) | Renamed **"All Classes"**, moved to `/classes/all`; reached from the schedule page's "Browse all classes" button, not the menu | **Public / All Classes** | Rename only — content as you designed it, plus a "View the calendar" header action |

## New and rebuilt screens

| Figma frame name | URL | What it shows |
|---|---|---|
| 🆕 **Public / Blog** 🎨 | `/blog` | Public blog, a single ~830px column. Top: a **date navigator** — Year and Month dropdowns *always visible*, defaulting to "All Years" / "All Months" (Month works without a year: "March" = March in *every* year), plus a Day dropdown that appears only once both are chosen. Then post cards, 5 per page: optional **photo banner** above everything (rendered at natural size, centred, capped ~350px tall — never stretched), title, byline "By {author} • {date}", and the body **rendered from markdown** — headings, bold, lists, links, blockquotes, code blocks. Signed-in blog authors additionally see an Edit Post / Delete Post row per card (delete confirms first). Previous/Next pager at the bottom; empty state ("No posts here yet — check back soon.") and an error state. Posts are tenant content — design the card, not the words. |
| 🆕 **Public / Getting Started** 🎨 | `/start` | The "Get Started" menu item finally opens a real page: **seven numbered step cards**, each an icon + title + a couple of sentences + exactly one CTA — 1 Try an intro workshop → Events · 2 Create your account → Register · 3 Pick your membership → Memberships · 4 Plan your weekly schedule ("a plan, not a booking") · 5 Show up and check in → Our Classes · 6 Workshops and series · 7 Stay in the loop → Notification Preferences. Signed in, step 2 disappears and the numbering **closes up** (1–6, no gap). Footer CTA back to Events for the undecided. All copy is 🎨 sample content. |
| ✏️ **Public / Home** 🎨 | `/` | **Recomposed top to bottom — treat as a new design.** Order: photo carousel (now the Gallery anchor) → hero (headline + subline + secondary brand image — all 🎨 slots) → **Get Started banner** (copy varies: "New here? …" vs "Not sure what to do next? …" + a "Show me how" button) → *then it branches.* **Visitors:** "Upcoming events" — up to 4 curated featured-event cards (name, when, facility, price, Book Now; View All Events link; empty state). **Members:** "Your upcoming events" (their 3 soonest bookings, Waitlisted chip, "All my bookings" link — hidden when empty) then "Events you could sign up for" (4 soonest not-yet-booked, Book Now — hidden when empty). → *Both:* the **Series & workshops coming up** panel, **expanded by default** (new building block below). → **Membership tiers** (shared tier cards + "See all memberships"): visitors always; members **only when they hold no membership**, headed "Become a member". → Members only: **"Upcoming classes"** — the embedded next-few-days class planner with per-row "I'll be there" / "I can't make it" toggles and a "Plan your weekly schedule" link. Copy rule from Mason: the window length (4 days) is *never* stated in any heading or copy. The old single "Next Upcoming Event" card is gone. |
| ✏️ **Public / Our Classes** | `/classes` | The rebuilt schedule page — **no longer read-only**. Header row: title + actions ("Browse all classes"; signed-in: "Plan your weekly schedule"; signed-out: "New here? Get started" + "Sign in"). Then a **collapsed** Series & workshops panel. Then a toolbar: week navigator (Previous / This week / Next week + a week label), a **My Schedule / Full Schedule** segmented toggle (signed-in only — signed-in defaults to *eligible classes only*; visitors always see everything), and a facility dropdown (only when >1). Body: Sun–Sat day sections of horizontal class cards — photo, name (→ class detail), time · duration, facility, instructor "with {name}" (+ substitute note), "Requires attending: {class}" prerequisite line — **plus a calendar-style status chip on the right** of every row: the attend toggle ("I'll be there" ⇄ plan), Booked / Waitlisted, Sold out, "Sign-ups open {date}" + Remind me, Cancelled, or a lock. Same chip vocabulary as Part 1's Calendar Event Chip, in a new compact in-row context. Still no prices on this page. Empty states: per-day "No classes", whole-week empty, and the filtered-empty "Your current filters hide everything." + a Show-full-schedule reset. |
| ✏️ **Public / Class Detail** | `/classes/:id` | The Upcoming Sessions section changed per class kind: **recurring** classes lost the (previously unbounded) session list entirely — replaced by a "When it runs" card ("just show up") + a **"See this class on the calendar"** CTA; **workshops** keep a bookable list but capped at the next **5**, with "See all dates on the calendar" below (+ a "Showing the next 5 dates." note when the cap hides some); **series** keep the runs/buy section unchanged, + the calendar link. |
| ✏️ **Account / Workshops & Series** | `/my/upcoming-offerings` | The inert "Sign-ups are open" text became **real Book buttons** — open offerings book straight into the series-booking / event-booking flow. Same signup-window vocabulary otherwise. |
| ✏️ **Calendar / Home** | `/calendar` | Two behavior notes, no visual rework: the toolbar gains a **"My weekly plan"** link (signed-in only, next to the mode toggle), and clicking an eligible open-window series/workshop now routes **straight into the booking flow** rather than to the class page. |

## New building blocks (Part 1 additions)

- **🆕 ★ Offering Highlight Panel** — the expandable "Series & workshops" panel used on **three** pages: Home ("Series & workshops coming up", expanded), Our Classes (collapsed), Upcoming Events (expanded). Collapsed: a single row — chevron, heading, "(N)" count, and a truncating names summary. Expanded: one row per offering — square class photo (icon fallback), class name (→ class detail), a **Series/Workshop kind chip**, optional subtitle, a when-line (date range + "N sessions" for series; date · time for workshops; an "Already under way" tag for prorated mid-run joins), a price line ("$X" + "($Y per session)"), and a right-side action that is exactly one of: **Book** button · "Sign-ups open {date}" text · a **locked** button (lock icon, e.g. members-only). *Code name: `offering-highlight`.*
- **🆕 ★ Blog Post Card** 🎨 — photo banner slot (natural size, centred, max-height capped — never upscaled), title, byline, prose body, optional author-controls row (Edit / Delete). *Code: the post card in `blog-list`.*
- **🆕 ★ Prose / long-form text styles** 🎨 — the blog renders arbitrary markdown, the site's first true long-form surface: define styles for h1–h3, paragraph, bold/italic, links, ul/ol, blockquote, inline code, and code block *inside a post body*. Make them Figma text styles tied to the token layer, so another studio's font swaps through them.
- **🆕 Date Navigator (Year / Month / Day)** — three dropdowns defaulting to All-everything; only periods that actually contain posts are offered; **Month is independent of Year** ("March in any year"); Day appears only when both are set. Public blog (+ the blog admin list, which is 🚫).
- **🆕 Membership Tier Card** 🎨 — extracted and now shared between the Memberships catalog and the home page: tier name, description, monthly price, Subscribe CTA. Tier names/copy are tenant content. *Code name: `membership-tier-cards`.*
- **🆕 Getting Started Step Card** 🎨 — numbered card: number, icon, title, 1–2 sentences, exactly one CTA button. Numbering must close up when a step is hidden.
- **🆕 Get Started Banner** 🎨 — the home-page CTA panel: short copy (two auth variants) + one button.
- **🆕 Home Summary Card** — the small card the home page repeats for bookings / featured events / could-sign-up events: name, when, facility, optional price + Book Now, optional Waitlisted chip. Worth one clean component with those option slots; today it's three near-identical bespoke blocks.
- **✏️ Calendar Event Chip** — no new states, but one new *context*: the compact chip now sits at the right edge of a photo class-row on Our Classes. Check your chip reads well inside a row layout, not just inside calendar cells.

## 🚫 Not your problem (new since 7/19)

- **Blog authoring** — the Blog Posts admin list (status chips Draft / Scheduled / Published / No date, year–month filters, table) and the **editor** (title/author/draft/post-date form, photo upload, split-pane markdown editor with live preview, Post Now / Save / Cancel). Back office; composes from your blocks. *(Confirmed strictly out of scope — OQ-D1, 8/4/2026.)*
- **Admin data-editor cleanups** (read-only field handling, date formatting, enum dropdowns) — invisible restyle targets.
- **Everything multi-tenant on the server** — tenancy is invisible in the UI beyond the 🎨 slots above.

## Quick wins #2 (priority order for this batch)

1. **Header + Mobile Menu Drawer refresh** — the flat menu shows on every page; cheapest site-wide win.
2. **Public / Home** — the recomposition touches the highest-traffic frame you have.
3. **Public / Our Classes** — top-traffic page and the new chip-in-row interaction column.
4. **Public / Blog + the prose text styles** — the prose styles outlive the blog (any future long-form content reuses them).
5. **Public / Getting Started** — seven cards, quick to compose from Button/Card/Icon.
6. **Make the brand swappable** — three concrete Figma tasks:
	1. **Create Figma Variables** for the colors, font families, and corner radii, then **rewire the components to reference the variables** instead of raw values — so `#ED1C26` lives in exactly one place and every button/chip/link points at it. (Same for the fonts: text styles reference a "display/heading/body" variable, not "D-DIN" directly.)
	2. **Mark the brand content as swappable**, not baked in: the logo, studio name, hero headline/subline, tagline, and photos should be their own layers/components that can be swapped per studio — never flattened into artwork. (The slot table in the multi-studio section above is the exact list.)
	3. **Build one proof frame**: duplicate Home + the header, then swap in an invented studio — different palette values, a different-shaped logo, a longer studio name, a different font, different photos. If the layout survives that swap untouched, the design is genuinely multi-studio; wherever it breaks is a fix.
	- Mason- Ryan is unsure what you mean when you say "The \<icon\>pass". The icon is cute but can you use a word here? What exactly do you want him to do here?
	- Claude- ✅ Fair — rewritten above as "Make the brand swappable" with the three concrete tasks. (The 🎨 marker elsewhere in the doc still just means "this item contains per-studio content".)

## 🔍 Figma name audit — live file vs this doc (run 8/4/2026, via the Figma API)

Audited `figma.com/design/IxWR3NfPQbJfYJ7oCPmaER/Knotty-Yoga` (last modified 8/4/2026; pages **Foundations** + **Screens**) against the names in this doc. Four buckets; the ✏️ renames are mechanical and worth doing first — they're what makes the engineering mapping automatic.

### ✅ Components that map cleanly (rename = case/punctuation only)

His → doc name: Event session card → **Event Session Card** · Section header → **Section Header** · Page header / Back nav → **Page Header / Back Nav** · Footer ✅ · Card / surface → **Card / Surface** · Checkbox ✅ · Tooltip ✅ · Avatar ✅ · Modal → **Modal / Dialog** · Pagination → **Pagination Controls** · Photo carousel → **Photo Carousel** · Photo upload control → **Photo Upload Control** · Foreign key picker → **Foreign-Key Picker** · Seat assignment widget → **Seat Assignment Widget** · Room occupancy badge → **Room Occupancy Badge** · Book for selector → **Book-For Selector** · Payment method → **Payment Method / Card Picker** · Text input → **Text Input** · Long Text input → **Long Text Input** · Drop down menu → **Dropdown / Select** · Top navigation → **Header / Top Nav Bar** · 404 error illustration → **404 / Page Not Found** · **Badge** → **Badge / Pill** — and his Badge's tone variants (Success/Warning/Danger/Info/Neutral) are *exactly* the doc's vocabulary. Card_membership → **Membership Tier Card** (a Delta #2 block he already built — nice).

### ✏️ Structural fixes (merges + variant hygiene)

- **Three button components → one.** Primary button / Secondary button / "Tertiary button " (note the trailing space) should be a single **Button** component set with a `Kind` property — plus his states (Default/Hover/Disabled/Down) are missing the doc's **Focus** and **Loading**. Calendar button / Large book now button / Nav button_2 / Button holder: fold into Button variants or give them real names.
- **Duplicate pairs to merge:** `Badge` + `Badge pill` (keep Badge, it has the right tones; Badge pill's variants are unnamed) · `Toast` + `Snackbar` → one **Toast / Snackbar** · `` `simple-text` `` + `Text input` → keep **Text Input** (ironically `simple-text`'s State=Default/Focus/Disabled/Error variants are the *correct* state set — merge them in).
- **Variant naming:** many components have `Property 1=Variant2 / Variant3 / t / ==` — these are the names the engineer maps to state/kind enums, so they need real values (e.g. `State=Filled`, `Kind=Secondary`).
- **Trailing spaces break exact-name mapping:** "Tertiary button ", "Upcoming events ", "Account > User information ", "Service calendar > Massage ".
- **Anonymous top-level frames** in Foundations (Frame 23/47/49/68/125/198…) — name or fold them.

### ❓ Four things only Ryan can answer (map or rename) — ✅ all settled 8/4/2026

- **Fill in the blank** — which doc block is this?
	- Mason- Ryan and I are unsure what you mean by this. Can you elaborate?
	- Claude- It's the literal name of a component **in Ryan's Foundations page** — I didn't coin it. It sits at roughly x 498, y −1052 (left side, above the color palette): a 27px circle with three variants — plain grey circle, and one with a white ring + a 17px **red dot** (`#ED1C26`) inside. That anatomy is a **Radio Button** (the "fill in the blank" is the dot). Rename it **Radio Button** — and since the doc's Part 1 never listed a radio atom (it only appears implicitly inside Payment Method / Card Picker), consider **Radio Button** officially added to the Foundational form controls as of this audit. Variants should be `State=Unselected / Selected / Disabled`.
- **Week calendar** — is it the **Date Strip / Week Navigator** (service booking) or a calendar view piece?
	- Mason- This is the date strip for service booking.
	- Claude- ✅ Rename it **Date Strip / Week Navigator**.
- **Calendar / Calendar select / Calendar button** — is this trio the **Date Picker**?
	- Mason- Yes
	- Claude- ✅ Fold the three into one **Date Picker** component (Calendar select / Calendar button become its variants or internals), and the `Date Picker*` entry comes off the missing-components list.
- **TV - 1** (1106×623 frame on Screens) — what is it?
	- Mason- We are unsure what you are talking about.
	- Claude- It's a frame on the **Screens page, far right of the grid** — at roughly x 2642, y 2658, sized 1106×623 (≈16:9) — containing nothing but a white **play-arrow icon**. It reads as a video-player placeholder, probably pasted and forgotten. If it's meant for something (a studio-TV display? an embedded video block?), name it for what it is; otherwise delete it.
- (Extras that are fine but unmapped: Card_product, Card_account, Card_split — if they stay, the underscore naming should at least become `Card / Product` style. KnottyYoga_Logo + SafeSpace_icon are 🎨 slots — name them **Logo** and **Hero Secondary Image** so the theming work lands on slots, not brand names.)
	- Mason- So just rename these to be Card / Thing?
	- Claude- ✅ Yes, slash style to match `Card / Surface`: **Card / Product**, **Card / Account**, **Card / Split** — with one exception: `Card_membership` should become **Membership Tier Card**, because that exact name already exists in this doc (Delta #2) and the code has the matching shared component. And the two brand assets rename to **Logo** and **Hero Secondary Image** as above.

### ✏️ Screens — rename map (his → doc)

| His frame | → Figma name from this doc |
|---|---|
| Homepage | **Public / Home** *(also needs the Delta #2 recomposition)* |
| Classes | **Public / All Classes** *(the catalog — the `/classes` URL now belongs to the rebuilt schedule page)* |
| Staff | **Public / Instructors** |
| Staff / Caleb, Staff / Mason | **Public / Instructor Detail** *(keep one)* |
| Upcoming events␣ | **Public / Upcoming Events** |
| Services | **Shop / Service Catalog** |
| Memberships **(duplicated — two identical frames)** | **Shop / Subscription Catalog** *(delete one)* |
| About Knotty Yoga | **Public / About** |
| Event details page > Intro workshop **(×3 same-size copies)** | **Shop / Event Booking** *(if the three are states, suffix them: `… / Default`, `… / Guest added`, …)* |
| Account > User information␣ | **Account / Profile (View)** |
| Service calendar > … (7-frame flow incl. Select time / Summary / Confirmed) | **Shop / Service Booking** + state suffixes — *the flow-state structure here is exactly right, keep it* |
| Account dashboard | **Account / Home** |

### ❌ Not in the file yet (the honest gap list)

- **Mobile: zero 375 frames — every screen is 1440-wide desktop only.** This is the single biggest gap vs the locked scope (375 + 1280, mobile first). Also note desktop is 1440 vs the doc's 1280 — fine either way, but pick one and it becomes the standard (updating the doc to 1440 is a one-line change if that's his canvas).
- **Screens (32 of 44 in scope):** all Auth (Login/Register/Verify), Shop Catalog / Product Detail / Cart / Checkout / Subscription Signup / Series Booking, Public Class Detail / **Our Classes** (rebuilt schedule — top traffic) / Blog / Getting Started / Provider Bio, **Calendar / Home**, and the Account wave (Profile Edit, Change Password, Purchase History + Detail, My Events, Saved Cards, My Subscriptions + Detail, Gift Permissions, My Vouchers, My Skills, My Schedule, Today's Classes, Workshops & Series, Notification Preferences, Attendance History, Favorite Instructors). *(12 of 44 are covered by his existing desktop frames — the mapping table above.)*
- **Components:** the July-wave blocks (Tag Chip, Filter Chip Row, **Calendar Event Chip/Card**, Kind Badge, Favorite Toggle, Skill Chip + Prerequisite Banner, Attendance/Booking Status Chips, Signup-Window Indicator, Segmented Toggle, Week Navigator, Eligible-Slot Checkbox Card, Attendance Plan Row, Series Run/Summary/Rollup, Coupon & Voucher Panel, Guest Booking Fieldset, Skill Badge Card, Skill Requirement Dialog, Inline Cancel Flow) + the base set still open (Alert Card, **Empty State**, Spinner, Data Table / List, **Mobile Menu Drawer**, Sticky Bottom Action Bar, Bottom Sheet, Native Payment Buttons) + the Delta #2 additions (Offering Highlight Panel, Blog Post Card, Prose text styles, Date Navigator, Getting Started Step Card, Get Started Banner, Home Summary Card). *(Date Picker and Date Strip / Week Navigator came off this list — the ❓ answers confirmed his Calendar trio and Week calendar are exactly those, pending rename.)*

Mason- Let's put together a plan. What do we need to do to start to integrate his work into my project? I'm guessing there is cleanup and standardization work I could do with you to get all the components CSS and so forth to move from a bunch of bespoke individual, separate CSS files into a standard file so that we can better integrate his stuff? Can we pull some of the work he's done over now to get the site looking more like his design? Can we create a list of things to get where we want to eventually?

Claude- ✅ Yes to all three — plan below. Short answers first: **(1)** the bespoke-CSS → standard-file consolidation is real, already scoped as [[Website Makeover]] Phase 2, and can start **immediately with zero Ryan dependency**; **(2)** yes, some of his work ports over *now* — his palette is already mined from the file (details in Track A), and once the shared classes exist, restyling the atoms he's finished restyles every page at once; **(3)** Track D is the eventually-list. The heavy engineering detail stays in [[Website Makeover]] (its Phases 2–4 are exactly this work — note its "Current State" is a 5/19 snapshot, so each phase gets re-grounded at execution); this section is the sequencing against today's reality.

### 🧩 Integration plan — getting Ryan's design into the product

**Track A — Lock the design tokens (with Ryan, ~one session).** The bridge between his Figma and the code is the CSS-variable token layer — the same layer [[Tenant Theming and Branding]] Phase 4 later serves per-tenant and [[Website Makeover]] 2.1 specs. Build it once; every other track consumes it.

- [x] Mine his palette from the live file *(done 8/4 via the API)*: **primary `#ED1C26`** — note this is NOT the code's current red (`#F50C22`); his is presumably the intended brand value — plus ink `#000000`, surface `#FFFFFF`, surface-tint `#F3F3F3`.
- [x] ~~Ryan fills the palette gaps: accent + status tones~~ — **resolved 8/4**, see below.
	- Mason- The orange does not survive. Yes to the badge components. Let me know if he needs to change anything for you to map this over.
	- Claude- ✅ **Nothing needs to change — the mapping is already done.** I mined his Badge variants from the file; each tone is a background + text pair, which is exactly what the code's badge classes need: **Success** `#BAF7CD` on-tone `#000000` · **Warning** `#FFE4E5` / `#B4191D` · **Danger** `#ED1C26` / `#FFFFFF` (his danger *is* the brand red — consistent with the theming doc's OQ-T1: split roles, same-ish values for KY) · **Info** `#7B7B7B` / `#FFFFFF` · **Neutral** `#F3F3F3` / `#000000`. Two *nice-to-haves*, neither blocking: add these five pairs as swatches to his Color palette frame so the palette documents itself, and register them as Figma Variables in priority #2 below. **Orange retirement is trivial:** the entire codebase uses it in exactly one place — the footer background (`bg-theme-orange`) — which gets restyled to his Footer design in Track C anyway. The only *remaining* Ryan inputs for the token layer: the neutral text grey, corner radii, and the type scale — all part of the Variables pass (priority #2 in his list below).
- [x] Us: write the token stylesheet — `--theme-*`, `--font-*`, `--radius-*` with his values as the defaults (primary `#ED1C26`, the five tone pairs above), legacy `--red`/`--orange` aliased onto it so every existing `theme-red` Tailwind usage restyles instantly. **Done 8/11/2026** — as-built below.
	- **New `ui/src/assets/styles/_tokens.scss`** — the whole token layer under `:root`: brand (`--theme-primary` = `#ED1C26`, `--theme-on-primary`; `--theme-accent` collapsed onto primary since the orange is retired), neutrals/surfaces (`--theme-ink`, `--theme-neutral`, `--theme-surface`, `--theme-surface-tint`, `--theme-background`, `--theme-border`), the five Badge tone pairs mined from his file (`--theme-success`/`--theme-on-success` … plus `--theme-muted`/`--theme-on-muted` for the neutral fill), type roles (`--font-body` / `--font-heading` / `--font-display`), and radii (`--radius-card`, `--radius-control`, `--radius-pill`). Names follow [[Tenant Theming and Branding]]'s Token Catalog, which has been updated to the as-built set.
	- **`_variables.scss` is now the deprecated alias layer** — `--red` → `--theme-primary`, `--gray` → `--theme-neutral`, `--black` → `--theme-ink`, `--white` → `--theme-surface`. Net visible change today: everything using `theme-red` moved from `#F50C22` to Ryan's `#ED1C26`. The dead Sass `$red/$orange/$gray/$black/$white` variables are gone (nothing referenced them).
	- **One deliberate exception:** `--orange` stays a literal `#FF9933` rather than aliasing onto the (now red) accent role — otherwise the footer, its only consumer, turns red before Track C restyles it. That line and the `.bg-theme-orange` usage get deleted together during Track C.
	- **The tokens are load-bearing, not decoration:** the `.din` / `.din-bold` / `.din-condensed-bold` classes used in 130+ templates now resolve through `--font-*`, `body`/`*` font-family resolves through `--font-body`, and the Material pill-button override reads `--radius-pill`. A studio's font swap is now a token change.
	- **Test:** `ui/src/app/shared/design-tokens.spec.ts` (10 specs) — every token is non-empty, the brand red and all five tone pairs match the mined values, the legacy aliases resolve through the tokens, `text-theme-red` computes to `rgb(237, 28, 38)` (proving the Tailwind → alias → token chain), the `.din*` classes match their font tokens, and `.mdc-button` takes its radius from the token. Full suite green: **2925 tests**, plus `ng build` and `ng lint` clean for the touched files.
	- **Not done here (on purpose):** the Tailwind config still only maps the legacy `theme-*` names — semantic aliases (`bg-brand`, `text-surface`) are Makeover 2.2 and get added when Track B's shared classes need them; the Material palette is still stock indigo (Makeover 2.3 / Track B); no component SCSS was touched.

**Track B — CSS consolidation (us, no Ryan dependency — the "bespoke → standard" work).** [[Website Makeover]] Phase 2, executed with today's facts (the `@honuware/ui` library is already de-Tailwinded and bans hardcoded colors, so it restyles through the variables for free):

- [ ] One shared **surface-card** class replacing the `border: 1px solid #d1d5db` duplicated across **56+ component SCSS files** (the single worst offender).
- [ ] Shared **badge/status** classes replacing the ~10 bespoke `.status-badge` / `.role-badge` / `.applied-badge` implementations, mapped onto the five Badge tones.
- [ ] Shared **page-header/back-nav**, **empty-state**, and **form-layout** patterns (each currently re-implemented per page).
- [ ] **Material theme realigned to the tokens** (makeover 2.3) — buttons, toggles, spinners pick up brand color from `--theme-primary` instead of the stock indigo.
- [ ] A **style-guide page** rendering every token + shared class — the safety net for the sweep, and later the theming doc's preview surface.
- [ ] Then the mechanical sweep replacing bespoke SCSS with the shared classes, **one layer at a time with test gates between layers**: controls/shared → public → shop → account; the back office inherits.

**Track C — Port his finished designs (after A + B — the "looking like his design" ask).**

- [ ] Atoms first, biggest leverage: Button (his three kinds), Text/Long Text inputs, Checkbox + Radio, Dropdown, Card, Badge, Avatar, Toast/Tooltip/Modal, Pagination — because they restyle through the shared layer, every page shifts at once.
	- Mason- Did you see all of these in Ryan's design? Please let me know here if there is anything that is missing.
	- Claude- ✅ **Yes — every atom on this list exists in his Foundations page.** Button (as three components, merging to one per the rename list), Text Input + Long Text Input, Checkbox, Radio ("Fill in the blank"), Dropdown ("Drop down menu"), Card ("Card / surface"), Badge (with the five tones), Avatar, Toast, Tooltip, Modal, Pagination — all present, which is why this track can start as soon as A + B land. Two caveats, not blockers: his Button set is missing the **Focus** and **Loading** states, and several components have junk variant names (both covered in the rename & cleanup list below). The atoms he *hasn't* drawn are deliberately not in this track — the nearest gaps worth adding to it when he draws them are **Empty State**, **Spinner**, and **Alert Card** (small, high-frequency); the rest live in his prioritized list.
- [ ] **Header + Footer** to his design — the largest single visual change on every page (both are 🎨 slots, so his layout + token values, no hardcoded brand).
- [ ] Per-screen layout polish for the **12 screens he has designed** (desktop): Home, About, Instructors + Detail, All Classes, Upcoming Events, Service Catalog + Booking flow, Subscription Catalog, Event Booking, Profile, Account Home. Mobile keeps today's responsive behavior until his 375 frames exist — no guessing at layouts he hasn't drawn.

**Track D — The "eventually" list (standing backlog, in order).**

- Ryan: the ✏️ rename pass → mobile frames for what exists (Home + Header/Drawer first) → the missing 32 screens + July/Delta components, per the two Quick-wins lists → Figma Variables + the fake-studio proof frame.
	- Mason- Can you create buckets of screens that you would like to see with the buckets organized from most useful to least useful. I'm sure that there are quite a few that will be totally fine as is with the global style defaults.
	- Claude- ✅ Done — three buckets, most-useful → least, inside "The prioritized list for Ryan" at the bottom of the doc. Your guess is right: **twelve** of the missing screens land in Bucket 3 ("skip — global styles cover them").
- Us: makeover Phase 3's mobile additions (Sticky Bottom Action Bar, Bottom Sheet, native Apple/Google Pay buttons), dark mode (makeover Phase 6, consuming theming D8's structure), optional visual-regression CI — and then [[Tenant Theming and Branding]] Phases 1–8 turn the whole token/content layer per-tenant.
- Dependency note: Track A's variable file is shared ground between makeover 2.1 and theming Phase 4 — whichever lands first creates it, the other consumes.

**Sequencing:** A needs Ryan (start while he's around); B can start tomorrow and runs parallel to anything; C follows A+B; D is the backlog. First visible payoff = A + the Material realignment + Header/Footer — the site reads as "his design" site-wide before a single screen is individually polished.

### 📋 The exact rename & cleanup list (paste-ready)

- Mason- Can you give detailed instructions with a list of exactly what needs to be renamed and what the current value is and a guess of what you think the rename should look like? Please make this easy for us.
- Claude- ✅ Here it is — four tables, worked top to bottom. Every "current" value is exactly as it reads in the file today (␣ marks a trailing space to delete). Where a variant rename is a guess, it's marked *(check by eye)* — the guess says which state I *think* each unnamed variant is; reassign if I guessed the order wrong.

**Table 1 — Component renames (Foundations page).** Select the component, rename, done.

| Current name | Rename to |
|---|---|
| `Event session card` | `Event Session Card` |
| `Section header` | `Section Header` |
| `Page header / Back nav` | `Page Header / Back Nav` |
| `Card / surface` | `Card / Surface` |
| `Modal` | `Modal / Dialog` |
| `Pagination` | `Pagination Controls` |
| `Photo carousel` | `Photo Carousel` |
| `Photo upload control` | `Photo Upload Control` |
| `Foreign key picker` | `Foreign-Key Picker` |
| `Seat assignment widget` | `Seat Assignment Widget` |
| `Room occupancy badge` | `Room Occupancy Badge` |
| `Book for selector` | `Book-For Selector` |
| `Payment method` | `Payment Method / Card Picker` |
| `Text input` | `Text Input` |
| `Long Text input` | `Long Text Input` |
| `Drop down menu` | `Dropdown / Select` |
| `Top navigation` | `Header / Top Nav Bar` |
| `404 error illustration` | `404 / Page Not Found` |
| `Badge` | `Badge / Pill` |
| `Card_membership` | `Membership Tier Card` |
| `Card_product` | `Card / Product` |
| `Card_account` | `Card / Account` |
| `Card_split` | `Card / Split` |
| `KnottyYoga_Logo 1` | `Logo` |
| `SafeSpace_icon 1` | `Hero Secondary Image` |
| `Fill in the blank` | `Radio Button` |
| `Week calendar` | `Date Strip / Week Navigator` |
| `Calendar` | `Date Picker` |

Already correct, no action: `Footer`, `Checkbox`, `Tooltip`, `Avatar`, `Photo carousel`'s siblings not listed here, and the Material-icon frames.

**Table 2 — Merges & deletions.**

| #   | Action                                                                                                                                                                                        |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| M1  | `Primary button` + `Secondary button` + `Tertiary button␣` → **one** `Button` component set with a `Kind` property (Primary / Secondary / Tertiary). Kills the trailing space too.            |
| M2  | `Badge pill` → merge anything unique into `Badge / Pill`, then **delete** `Badge pill`.                                                                                                       |
| M3  | `Toast` + `Snackbar` → one `Toast / Snackbar` — keep whichever looks better, delete the other.                                                                                                |
| M4  | `` `simple-text` `` → its State variants (Default/Focus/Disabled/Error) are the *correct* state set — move them into `Text Input`, then **delete** `simple-text`.                             |
| M5  | `Calendar select` + `Calendar button` → fold into `Date Picker` (variants or internals).                                                                                                      |
| M6  | `Drop down select` (a frame, not a component) → fold into `Dropdown / Select` or delete.                                                                                                      |
| M7  | `Button holder`, `Nav button_2`, `Large book now button` → fold into `Button` as variants, or rename to what they actually are.                                                               |
| D1  | Delete the duplicate `Memberships` frame (two identical copies on Screens).                                                                                                                   |
| D2  | Keep `Staff / Caleb` (rename per Table 4); **delete** `Staff / Mason`.                                                                                                                        |
| D3  | Keep one `Event details page > Intro workshop`; delete the other two — or if they're intentional states, suffix them (`… / Guest added`, etc.).                                               |
| D4  | Delete `TV - 1` (the stray play-button frame, far right of Screens) — or name it if it's real.                                                                                                |
| D5  | The **never-named frames** on Foundations (Figma auto-names). Identified from the file: `Frame 47` = a stray "BACK TO SERVICES" back-link → fold into `Page Header / Back Nav` or delete · `Frame 23` = a loose "SIGN UP" text → delete · `Frame 68` = two Primary-button instances side by side → delete, or name it if it documents a spacing rule · `Frame 49` (465×215) → open it and decide · `Frame 125` → dissolves with M6 · `Frame 198` = the "Hint text" row inside `simple-text`'s variants → rename `Helper text` (rides along in M4). |
| D6  | The five `Service calendar > …` service copies: keep the **Peak spa** chain as canonical (rename per Table 4); delete or park `Early bird spa`, `Late night spa`, `Non peak spa`, `Massage␣`. |

**Table 3 — Variant renames** *(check by eye — the guesses assume the variants are in creation order)*.

| Component | Current variants | Rename to |
|---|---|---|
| `Button` (after M1) | `Main=Default / Hover / Disabled / Down` | `State=Default / Hover / Disabled / Pressed` — then **add `Focus` and `Loading`** |
| `Checkbox` | `Checkbox=Uncheck / Checked` | `State=Unchecked / Checked` |
| `Radio Button` | `Property 1=Default / Variant2 / Variant3` | `State=Unselected / Selected / Disabled` *(check by eye)* |
| `Text Input` | `Property 1=Default / Variant2 / Variant3 / t / Variant5` | `State=Default / Focus / Filled / Error / Disabled` *(check by eye)* |
| `Long Text Input` | `Property 1=Default / Variant2 / Variant3 / Variant4 / t` | `State=Default / Focus / Filled / Error / Disabled` *(check by eye)* |
| `Dropdown / Select` | `Drop down menu=D / Select / Disabled` | `State=Default / Open / Disabled` |
| `Foreign-Key Picker` | `Foreign key picker=Default / Hover` | `State=Default / Hover` |
| `Date Picker` | `Calendar=No select / Select` | `State=Empty / Selected` |
| `Avatar` | `Property 1=Default / Variant2 / Variant3` | `State=Image / Initials / Placeholder` *(check by eye)* |
| `Payment Method / Card Picker` | `Property 1=card picker / Variant2` | `Kind=Card Picker / Saved Card` *(check by eye)* |
| `Header / Top Nav Bar` | `Property 1=Login / No login` | `State=Signed in / Signed out` |
| `Card / Account` | `Property 1=Default / Variant2` | `State=Default / Hover` *(check by eye)* |

**Table 4 — Screen renames (Screens page).**

| Current name | Rename to |
|---|---|
| `Homepage` | `Public / Home` |
| `Classes` | `Public / All Classes` |
| `Staff` | `Public / Instructors` |
| `Staff / Caleb` | `Public / Instructor Detail` |
| `Upcoming events␣` | `Public / Upcoming Events` |
| `Services` | `Shop / Service Catalog` |
| `Memberships` (the survivor of D1) | `Shop / Subscription Catalog` |
| `About Knotty Yoga` | `Public / About` |
| `Event details page > Intro workshop` (survivor of D3) | `Shop / Event Booking` |
| `Account > User information␣` | `Account / Profile (View)` |
| `Service calendar > Peak spa` | `Shop / Service Booking` |
| `Service calendar > Peak spa > Select time` | `Shop / Service Booking / Select Time` |
| `Service calendar > Peak spa > Select time > Summary` | `Shop / Service Booking / Summary` |
| `Service calendar > Peak spa > Select time > Summary > Confirmed` | `Shop / Service Booking / Confirmed` |
| `Account dashboard` | `Account / Home` |

# The prioritized list for Ryan

- Mason- Can you replace this with a bulleted list of the ordered priorities that Ryan should work on next in order based on a mix of what needs to be done first and what is most important?
- Claude- ✅ Here it is — ordered by "unblocks the most" first, then by traffic/impact. Items 1–2 unblock the engineering side (Mason and I start pulling values the moment they exist); everything after is design work in impact order.

1. **Housekeeping (~45 min).** Work the rename & cleanup tables above top to bottom (Tables 1 → 2 → 3 → 4). Purely mechanical, and it's what makes the Figma ↔ code mapping automatic from here on.
2. **Figma Variables — the token pass.** Create variables for: the palette (primary `#ED1C26`, ink `#000000`, surface `#FFFFFF`, surface-tint `#F3F3F3`, **plus the five badge tone pairs already mined from your Badge component** — promote them to the Color palette frame; **no orange**), the neutral text grey, font roles (display / heading / body), and corner radii — then rewire the components to reference the variables instead of raw values. *This is the highest-leverage hour in the whole list: the engineering token file is written straight from it.*
3. **Header / Top Nav Bar + Mobile Menu Drawer** — the flattened menu model from Delta #2, desktop **and 375**, including a mobile cart affordance. Every page on the site shows this.
4. **Mobile (375) frames for the 12 screens that already exist** — Public / Home first (fold the Delta #2 recomposition in while you're there), then the payment-adjacent flows (Shop / Event Booking, the Shop / Service Booking chain), then the rest. Mobile is the primary canvas — this is the biggest gap in the file today.
5. **The fake-studio proof frame** — duplicate Public / Home + the header, swap in an invented brand (Quick Win #6's three tasks). Quick once #2 and #3 exist, and it validates the whole token wiring.
6. **Bucket 1 — screens that genuinely need design** (unique layouts, new interaction patterns, or top traffic — in order):
	1. **Public / Our Classes** — the rebuilt schedule; top-traffic page with the new chip-in-row column.
	2. **Shop / Cart + Shop / Checkout** — the conversion funnel; mobile-first with the Sticky Bottom Action Bar + Native Payment Buttons.
	3. **Calendar / Home + the Calendar Event Chip/Card three-density set** — the single biggest component item in the inventory; the calendar is the most visually dense page.
	4. **Public / Class Detail** — viewer-aware pricing, skills, series runs.
	5. **Public / Blog + the Prose text styles** — the prose styles outlive the blog.
	6. **Shop / Series Booking** — the series checkout with summary card + coupons panel.
	7. **Public / Getting Started** — simple, but it's the funnel's front door.
	8. **Account / My Schedule** — the novel weekly-plan grid (Eligible-Slot Checkbox Cards).
	9. **Account / My Events** — the Series Rollup Panel + the multi-step Inline Cancel Flow (bottom sheet on mobile).
7. **Bucket 2 — design the pattern once, it covers the set:**
	- **Auth / Login + Register** → *the form-page pattern* (Verify inherits it).
	- **Shop / Catalog + Product Detail + Subscription Signup** → *the catalog/detail pattern*.
	- **Account / Today's Classes + Workshops & Series + Attendance History** → *the list-row + status-chip pattern*.
	- **Account / Notification Preferences** → *the preference-card pattern*.
	- Atoms to slot in as they come up: **Empty State, Spinner, Alert Card**, and the **Data Table mobile strategies** (one example of each of the three renditions).
8. **Bucket 3 — skip: global styles cover them.** Mason's instinct is right — these compose entirely from the shared blocks and pick up the new look in the engineering sweep: Account / Profile (Edit), Change Password, Purchase History, Purchase Detail, Saved Cards, My Subscriptions, Subscription Detail, Gift Permissions, My Vouchers, My Skills, Favorite Instructors, Auth / Verify Email, Public / Provider Bio (still a placeholder page), and the optional Staff / Class Check-In. Revisit one only if it looks off after the restyle lands.

## Open questions — ✅ all resolved (Mason, 8/4/2026)

- **OQ-D1 — Blog editor:** ✅ **RESOLVED — strictly 🚫.** *(Mason: "I'll go with your recommendation.")* The blog editor stays out of scope entirely; the optional-exception note has been removed from the 🚫 section above so no time gets spent there.
- **OQ-D2 — The theming document:** ✅ **RESOLVED — yes.** *(Mason: "Sure, that sounds great.")* Drafted: **[[Tenant Theming and Branding]]** — the token catalog, the content-slot inventory, the storage/endpoint design, and the admin "Site Theme" page now live there. This doc keeps only the 🎨 markers and the five design rules; when the theming doc's token catalog and Ryan's Figma Variables disagree on a name, reconcile there, not here.
- **OQ-D3 — Fake second studio frame:** ✅ **RESOLVED — yes, one Home + header frame.** *(Mason: "Sure, I'll go with your recommendation.")* Kept as rule 4 + Quick Win #6. The theming doc's final phase includes the engineering twin of this proof — a second local tenant with an invented brand — so Ryan's frame and the running site can be compared side by side.
