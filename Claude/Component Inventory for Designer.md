---
fileClass: Reference
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/19/2026
Version: 0.2
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
