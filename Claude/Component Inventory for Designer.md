---
fileClass: Reference
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 5/19/2026
Version: 0.1
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

### Navigation and layout

- **★ Header / Top Nav Bar** — the persistent top bar across every page. Logo on the left; menu on the right. Variants needed: logged-out (login + register), logged-in (account dropdown with avatar), with-cart-badge (shows a red circle on the cart icon when there are items), mobile (hamburger collapsed). On mobile the header should also accommodate a safe-area inset for the iPhone notch. *Code name: `header.component`.*
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

### Error and not-found

- **404 / Page Not Found** — the screen shown when the user hits a route that doesn't exist. *Code name: `page-not-found`.*

---

## Part 2 — Screens (design as Figma Frames)

Naming convention: use the **bold Figma frame name** from each entry below. For each screen, design at least a **Desktop (1280)** and **Mobile (375)** frame. Tablet (768) is helpful for the heavier data screens (calendar, tables) but optional elsewhere.

### Public website (no login required)

These are the screens an unauthenticated visitor sees. Highest visibility — they're the marketing surface — so probably worth the most design care.

| Figma frame name | URL | What it shows |
|---|---|---|
| **Public / Home** | `/` | Hero carousel, brand value-prop, next-upcoming-event card, link to events list. |
| **Public / About** | `/about` | About text and "how to join" section. Currently mostly placeholder. |
| **Public / Classes** | `/classes` | List of class types the studio teaches. |
| **Public / Class Detail** | `/classes/:id` | Detail page for a single class type. |
| **Public / Instructors** | `/instructors` | Grid of instructor photos with names. |
| **Public / Provider Bio** | `/providers/:personId` | A single instructor's biography page. |
| **Public / Upcoming Events** | `/events` | List of bookable upcoming events with dates and prices. |

(There is also a stub `/staff` page that currently just renders "staff works!" — flagged as an open question in the plan; ignore until that's resolved.)

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
| **Shop / Event Booking** | `/shop/event/:sessionId` | Book a seat at a specific scheduled event. |
| **Shop / Cart** | `/shop/cart` | Items in cart, voucher code field, upsell suggestions, totals, "proceed to checkout" button. |
| **Shop / Checkout** | `/shop/checkout/:productId` | Payment form (Square card input), order summary, final "pay" button. |

### My account (logged-in user pages)

| Figma frame name | URL | What it shows |
|---|---|---|
| **Account / Home** | `/my/account` | Landing page with cards linking to each account section. |
| **Account / Profile (View)** | `/my/user-information` | Photo, name, email, phone — read-only summary. |
| **Account / Profile (Edit)** | `/my/update-user-info` | Same fields, editable. |
| **Account / Change Password** | `/my/update-user-password` | Current password + new password + confirm. |
| **Account / Purchase History** | `/my/purchases` | List of past orders. |
| **Account / Purchase Detail** | `/my/purchases/:id` | Single order: line items, payment, receipt. |
| **Account / My Events** | `/my/events` | Upcoming events the user is booked into. |
| **Account / Saved Cards** | `/my/cards` | Saved credit cards with add/remove. |
| **Account / My Subscriptions** | `/my/subscriptions` | Active membership(s). |
| **Account / Subscription Detail** | `/my/subscriptions/:id` | Single subscription with billing history, pause/cancel actions. |
| **Account / Gift Permissions** | `/my/gift-permissions` | Manage who can book on your behalf (and who you can book for). |
| **Account / My Vouchers** | `/my/vouchers` | Vouchers/credits available to the user. |

### Studio operations — Manage portal (for the studio owner / managers)

There are a lot of these. The good news: they almost all follow one of three patterns: **list of things**, **form to create/edit a thing**, **dashboard with summary cards**. Designing one beautiful instance of each pattern is enough — the engineer can apply that pattern to all the rest.

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

### Studio operations — Staff portal (for instructors)

Same "list of things" / "form" / "dashboard" patterns.

| Figma frame name | URL | Pattern |
|---|---|---|
| **Staff / Dashboard** | `/staff` | Dashboard |
| **Staff / My Sessions** | `/staff/sessions` | List of sessions I'm teaching |
| **Staff / Bookings** | `/staff/bookings` | List of who's booked in to my sessions |
| **Staff / Schedule** | `/staff/schedule` | Calendar |
| **Staff / Time-Off** | `/staff/time-off` | List + form |
| **Staff / Shift Requests** | `/staff/shift-requests` | List + form |
| **Staff / Check-In** | `/staff/check-in` | Roster with check-in toggles |
| **Staff / Preferences** | `/staff/preferences` | Form |

### Calendar

| Figma frame name | URL | What it shows |
|---|---|---|
| **Calendar / Home** | `/calendar` | Month/week/day view of all events. Most visually-dense page in the app — likely needs its own treatment. |

### Admin (low priority — database editor)

This is the catch-all database editor used by the studio owner for things that don't have a dedicated `/manage` page yet. The styling here is the most boilerplate-y in the app and probably the lowest-priority for a designer pass. A "good enough" treatment that comes for free from the building blocks above is fine.

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

Everything else can come later.
