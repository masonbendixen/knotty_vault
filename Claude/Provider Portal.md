---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/26/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

 In the document Support for scheduled purchases.md, there is a section Provider Portal. I'd like to create a design and implementation for all of the items in that section. Please use this document, the code base, and these documents for context:

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

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Please create numbered subsections within each phase. Please always add tests for anything you chance for which testing is possible.

# Plan: Provider Portal

This plan implements scenarios 45–56 from Support for scheduled purchases.md. The provider portal gives service providers (massage therapists, personal trainers, etc.) a self-service view of their schedule, bookings, and the ability to manage their availability through time-off requests and shift changes.

## Current State

**Already built:**
- DB schemas: `schedule_templates`, `schedule_template_entries`, `time_off_requests`, `shift_change_requests` (tables exist, created at startup)
- DB schemas: `provider_type_assignments`, `provider_availability`, `provider_buffer_overrides` (fully implemented with table helpers)
- Staff portal shell: `/staff` route with `StaffGuard` (checks `instructor`, `admin_portal`, or `isAdmin`)
- Staff "My Sessions" view for event sessions
- Admin provider management UI: provider list, availability blocks, add/edit/delete
- Service booking flow: book_service endpoint sends confirmation email to customer (but NOT to provider)
- Booking cancellation: CancelBooking handles service bookings with refund

**Not yet built:**
- Table helpers for `schedule_templates`, `schedule_template_entries`, `time_off_requests`, `shift_change_requests`
- `provider` permission (currently only `instructor` exists for staff access)
- Provider notification email on new booking
- Provider-facing booking list (upcoming/past)
- Provider schedule view (availability + bookings overlay)
- Provider booking detail view
- Provider session cancellation
- Provider toggle for accepting bookings
- Schedule template generation (batch: template → concrete availability rows)
- Time-off request submission + approval workflow
- Shift transfer/trade request + approval workflow
- Scheduler admin UI for templates and approvals

---

## Phase 1: Foundation — Permissions, Table Helpers, and Provider Notification Email

**Goal**: Add the `provider` permission, create table helpers for the scheduling tables, send providers an email when they get booked, and register the scheduling tables in admin allowed lists.

### 1.1 Backend — Database Setup

- [ ] **Add `provider` permission** to seed data in `create_database.cpp`
  - New permission: `provider` ("Service provider — therapist, trainer, etc.")
  - New role: `Provider` with `provider` permission
  - Update `StaffGuard` in the frontend to also check for `provider` permission (add to `hasStaffAccess`)
- [ ] **Register scheduling tables** in admin allowed lists (`PopulateAdminTopLevelTables` / `PopulateAdminNestedTables`)
  - `schedule_templates`, `schedule_template_entries`, `time_off_requests`, `shift_change_requests`
- [ ] **Tests** for permission setup (verify provider permission exists after DB init)

### 1.2 Backend — Table Helpers

- [ ] **`schedule_templates` table helper**
  - `AddScheduleTemplate()`, `GetScheduleTemplate()`, `GetTemplatesForProvider()`, `GetActiveTemplatesForProvider()`, `UpdateScheduleTemplate()`, `DeleteScheduleTemplate()`
- [ ] **`schedule_template_entries` table helper**
  - `AddEntry()`, `GetEntriesForTemplate()`, `UpdateEntry()`, `DeleteEntry()`, `DeleteEntriesForTemplate()`
- [ ] **`time_off_requests` table helper**
  - `AddRequest()`, `GetRequest()`, `GetRequestsForProvider()`, `GetPendingRequests()`, `UpdateRequest()`
- [ ] **`shift_change_requests` table helper**
  - `AddRequest()`, `GetRequest()`, `GetRequestsForPerson()`, `GetPendingRequests()`, `UpdateRequest()`
- [ ] **Tests** for all four table helpers

### 1.3 Backend — Provider Booking Notification Email (Scenario 46)

- [ ] **Create `provider_booking_notification_mail.h/cpp`** in `business_logic/scheduling/`
  - `ProviderBookingNotificationData` struct: `providerFirstName`, `providerEmail`, `clientName`, `serviceName`, `variantName`, `date`, `time`, `facilityName`, `roomName`
  - `GenerateProviderBookingNotificationBody()` — HTML email template
- [ ] **Send notification email from `book_service.cpp`** after successful booking
  - Fetch provider's email from people table
  - Send email with booking details
- [ ] **Tests** for email body generation

### 1.4 Frontend — Provider Permission & Staff Guard Update

- [ ] **Update `hasStaffAccess()`** in `auth.types.ts` to include `'provider'` permission
- [ ] **Tests** for updated guard logic

---

## Phase 2: Provider Schedule & Booking Views (Scenarios 45, 47, 54, 55)

**Goal**: Provider can log in and see their upcoming bookings, schedule, and booking details.

### 2.1 Backend — Provider Bookings Endpoint

- [ ] **Create `GET /api/provider/my_bookings`** endpoint
  - Returns service bookings where `provider_person_id = session.GetPersonId()`
  - Supports `status=upcoming|past` filter
  - Joins: `bookings` → `bookable_service_sessions` → `products` / `product_variants` / `people` (client name) / `facilities` / `location_rooms`
  - Response: array of provider booking objects with client name, service, variant, time, facility, room, status
- [ ] **Create `GET /api/provider/my_schedule`** endpoint
  - Returns provider's availability blocks + overlaid bookings for a date range
  - Params: `date_from`, `date_to`
  - Response: `{ availability: [...], bookings: [...] }`
- [ ] **Tests** for both endpoints

### 2.2 Frontend — Types & ServerAccess

- [ ] **Add types** to `scheduling.types.ts`: `ProviderBooking`, `ProviderScheduleResponse`
- [ ] **Add ServerAccess methods**: `getProviderBookings(status?)`, `getProviderSchedule(dateFrom, dateTo)`
- [ ] **Update** ServerAccessNetwork, ServerAccessProxy, ServerAccessMock
- [ ] **Mock spec tests**

### 2.3 Frontend — Provider Dashboard (Scenario 47)

- [ ] **Update `staff-dashboard.component`** at `/staff`
  - Add "My Service Bookings" card (links to `/staff/bookings`)
  - Add "My Schedule" card (links to `/staff/schedule`)
  - Add "Time Off Requests" card (links to `/staff/time-off`)
  - Only show service cards if user has `provider` permission
- [ ] **Tests**

### 2.4 Frontend — Provider Bookings Page (Scenarios 45, 55)

- [ ] **New component: `provider-bookings`** at `/staff/bookings`
  - Upcoming bookings: cards showing client name, service, variant, date/time, facility/room
  - Past bookings: collapsible accordion
  - Click a booking for detail view (client name, service, variant, duration, location, notes)
  - Cancel button on upcoming bookings (links to provider cancellation flow — Phase 4)
- [ ] **Tests**

### 2.5 Frontend — Provider Schedule View (Scenario 54)

- [ ] **New component: `provider-schedule`** at `/staff/schedule`
  - Week view showing availability blocks and booked sessions overlaid
  - Navigate by week (prev/next)
  - Availability shown as background blocks; bookings shown as cards within
  - Each booking card: client name, service, time
  - Color coding: available (green bg), booked (blue card), blocked (gray striped)
- [ ] **Tests**

### 2.6 Frontend — Routes

- [ ] **Add routes** to staff.routes.ts: `/staff/bookings`, `/staff/schedule`, `/staff/time-off`

---

## Phase 3: Provider Self-Service — Accepting Bookings Toggle & Time-Off Requests (Scenarios 48, 51)

**Goal**: Provider can toggle whether they're accepting bookings and submit time-off requests.

### 3.1 Backend — Provider Profile Endpoint

- [ ] **Create `GET /api/provider/my_profile`** endpoint
  - Returns the logged-in provider's `provider_type_assignments` row(s)
  - Includes: provider type name, is_accepting_bookings, max_time_hole_minutes
- [ ] **Create `POST /api/provider/toggle_accepting`** endpoint
  - Toggles `is_accepting_bookings` on the provider's assignment
  - Takes `assignment_id` in the body
- [ ] **Tests**

### 3.2 Backend — Time-Off Request Endpoints

- [ ] **Create `POST /api/provider/time_off_request`** endpoint
  - Provider submits: `requested_date_us`, `reason` (optional)
  - Validates: date is in the future, within configurable window
  - Sets `status = 'pending'`
- [ ] **Create `GET /api/provider/my_time_off_requests`** endpoint
  - Returns all time-off requests for the logged-in provider
- [ ] **Create `POST /api/admin/review_time_off/:requestId`** endpoint
  - Admin approves or denies: `{ action: "approve" | "deny", notes: "..." }`
  - On approve: blocks the availability for that date (creates a blocked `provider_availability` entry or deletes existing availability for that date)
  - On approve with existing bookings: cancels affected bookings with full refund + notification emails
- [ ] **Tests**

### 3.3 Frontend — Provider Profile Section (Scenario 48)

- [ ] **Add "Accepting Bookings" toggle** to provider dashboard or bookings page header
  - Shows current status per provider type assignment
  - Toggle calls `POST /api/provider/toggle_accepting`
- [ ] **Tests**

### 3.4 Frontend — Time-Off Request UI (Scenario 51)

- [ ] **New component: `provider-time-off`** at `/staff/time-off`
  - Submit form: date picker, optional reason text
  - List of submitted requests with status (pending, approved, denied)
  - Status badges with color coding
- [ ] **Tests**

### 3.5 Frontend — Admin Time-Off Review

- [ ] **New component: `time-off-review`** at `/manage/time-off`
  - List of pending time-off requests across all providers
  - Approve/Deny buttons with optional notes
  - Shows warning if approving would affect existing bookings
- [ ] **Add link** to manage dashboard
- [ ] **Tests**

---

## Phase 4: Provider Session Cancellation (Scenario 56)

**Goal**: Provider can cancel an upcoming service session they're assigned to. Client is refunded and notified.

### 4.1 Backend — Provider Cancel Session

- [ ] **Create `POST /api/provider/cancel_session/:sessionId`** endpoint
  - Validates: provider is the assigned provider for this session
  - Cancels the service session (sets status to `cancelled`)
  - Cancels the associated booking
  - Processes full refund (ignoring cancellation policy — provider-initiated)
  - Sends cancellation email to client with note that provider cancelled
- [ ] **Create provider cancellation email template** (`provider_cancelled_session_mail.h/cpp`)
  - Different from client-initiated cancellation — message should explain "Your provider has cancelled this appointment" and mention full refund
- [ ] **Tests**

### 4.2 Frontend — Cancel Button on Provider Bookings

- [ ] **Add cancel action** to provider bookings page
  - Confirmation dialog: "Cancel this appointment? The client will be fully refunded."
  - On confirm: call `POST /api/provider/cancel_session/:sessionId`
  - Show success/error result
- [ ] **Tests**

---

## Phase 5: Schedule Templates & Generation (Scenarios 49, 50)

**Goal**: Schedulers can create recurring weekly templates for providers and generate concrete availability from them. Day-level overrides are supported.

### 5.1 Backend — Schedule Template Business Logic

- [ ] **Create `schedule_template_helper.h/cpp`** in `business_logic/scheduling/`
  - `CreateTemplate()` — creates a schedule template with entries
  - `GenerateAvailability()` — generates `provider_availability` rows from a template for a date range
    - For each day in range: check if day_of_week matches a template entry
    - Create `provider_availability` row with `source = 'template'` and `schedule_template_id` FK
    - Skip dates that already have manual overrides (source = 'manual') or approved time-off
  - `RegenerateAvailability()` — deletes template-generated rows and regenerates
- [ ] **Create `POST /api/admin/schedule_template`** endpoint
  - Create a new template with weekly entries
- [ ] **Create `GET /api/admin/schedule_templates/:providerId`** endpoint
  - List templates for a provider
- [ ] **Create `POST /api/admin/generate_availability`** endpoint
  - Generate concrete availability rows from a template for a date range
  - Params: `template_id`, `date_from_us`, `date_to_us`, `facility_id`
- [ ] **Create `POST /api/admin/override_availability`** endpoint (Scenario 50)
  - Add/modify a specific day's availability for a provider
  - Creates `provider_availability` with `source = 'manual'`
  - Manual entries take precedence over template-generated ones
- [ ] **Tests**

### 5.2 Frontend — Schedule Template Admin UI

- [ ] **New component: `schedule-template-editor`** at `/manage/providers/:personId/templates`
  - Create/edit templates: name, effective date range
  - Weekly grid: checkboxes for each day, start/end time inputs
  - "Generate Availability" button: date range picker + facility selector → calls generate endpoint
  - List existing templates with active/inactive status
- [ ] **Link from provider list** → "Templates" action button
- [ ] **Tests**

### 5.3 Frontend — Day Override UI

- [ ] **Add override capability** to existing provider availability admin page
  - "Override Day" button that creates a manual availability entry
  - Visual indicator for template-generated vs manual entries
  - Warning when overriding a template-generated day
- [ ] **Tests**

---

## Phase 6: Shift Transfers & Trades (Scenarios 52, 53)

**Goal**: Providers can request to give a shift to another provider (transfer) or swap shifts (trade). Requires target provider acceptance and scheduler approval.

### 6.1 Backend — Shift Change Request Logic

- [ ] **Create `shift_change_helper.h/cpp`** in `business_logic/scheduling/`
  - `CreateTransferRequest()` — provider offers a shift to another provider
  - `CreateTradeRequest()` — provider proposes swapping shifts
  - `RespondToRequest()` — target provider accepts/declines
  - `ReviewRequest()` — scheduler approves/denies
  - On final approval:
    - Transfer: reassign availability block to new provider, update any existing bookings' `provider_person_id`, email affected clients
    - Trade: swap both availability blocks, update bookings, email clients
- [ ] **Shift change email templates**
  - `shift_request_notification_mail` — notify target provider of incoming request
  - `shift_approved_client_mail` — notify clients their provider changed
- [ ] **Endpoints**:
  - `POST /api/provider/shift_change_request` — create transfer or trade
  - `GET /api/provider/my_shift_requests` — list own requests (sent and received)
  - `POST /api/provider/respond_shift_request/:id` — accept/decline
  - `POST /api/admin/review_shift_request/:id` — approve/deny
  - `GET /api/admin/pending_shift_requests` — list all pending for scheduler review
- [ ] **Tests**

### 6.2 Frontend — Provider Shift Request UI

- [ ] **New component: `shift-requests`** at `/staff/shift-requests`
  - "Request Transfer" form: select shift (from own availability), select target provider
  - "Request Trade" form: select own shift, select target provider + their shift
  - List of pending/completed requests with status flow
- [ ] **Tests**

### 6.3 Frontend — Admin Shift Request Review

- [ ] **New component: `shift-request-review`** at `/manage/shift-requests`
  - List of pending shift requests (transfers and trades)
  - Show both providers, shifts involved, any affected bookings
  - Approve/Deny buttons
  - Warning about client notifications
- [ ] **Add link** to manage dashboard
- [ ] **Tests**

---

## Dependencies & Ordering

```
Phase 1 (Foundation) — Must be first. Provider permission + table helpers + notification email.
Phase 2 (Schedule & Booking Views) — Depends on Phase 1. Provider-facing read views.
Phase 3 (Self-Service) — Depends on Phase 2. Toggle + time-off requests.
Phase 4 (Provider Cancellation) — Depends on Phase 2. Provider cancels a session.
Phase 5 (Templates & Generation) — Depends on Phase 1. Scheduler creates recurring patterns.
Phase 6 (Shift Changes) — Depends on Phases 2 & 5. Transfer/trade requests.
```

Phases 3, 4, and 5 can be worked on in parallel after Phase 2 is complete.

---

## Open Questions

1. **Provider permission assignment**: When an admin adds someone as a provider via the provider list page, should the system automatically assign them the `provider` role/permission so they can access the staff portal? Or should that be a separate manual step? (Currently adding a `provider_type_assignment` row does not grant any roles/permissions.)
	- Mason- Yes, this sounds great.

2. **Schedule template facility**: Template entries define day-of-week + start/end time but no facility. When generating concrete availability, the facility is specified. Should a provider's template be tied to a single facility, or should the generation step always require specifying which facility (supporting providers who work at multiple locations on different days)?
	- Mason- I like the flexibility of allowing providers working at multiple locations on different days.

3. **Time-off request window**: The scenario mentions "configurable window (e.g., min 2 weeks out, max 12 weeks out)." Should this be a server-configurable secret (like auth session duration), or is a hardcoded reasonable default acceptable for the initial implementation?
	- Mason- No reason not to make this a configurable secret.

4. **Shift change with existing bookings**: Scenario 52/53 note that if there are bookings during the shifted time, clients must be notified and given the option to cancel with full refund. Should the shift change be auto-approved only when no bookings exist, with admin override required when bookings are affected? Or should the admin always approve regardless?
	- Mason- Let's do auto approval if no bookings and then admin override needed if bookings are affected. I figure that the schedule will be posted for a bit before bookings are allowed and it would be nice to let the providers swap shifts easily until there are bookings.

5. **Provider cancellation vs admin cancellation**: Scenario 56 says provider can cancel their own session. Should this be limited to sessions that haven't started yet (like the client cancellation check), or should providers be able to cancel sessions at any time? Also, should provider cancellation always result in a full refund (as stated in the scenario), or should it follow the product's cancellation policy?
	- Mason- There should be a configurable window in which providers can cancel their own sessions. Once that window is past, the provider cancellation should result in a high priority notification to admins to do manual follow up like trying to call clients to let them know that their session has been cancelled. Client should definitely get a full refund for provider cancellation.

6. **Notification preferences**: Should the system support provider notification preferences (email vs no-email, or digest vs individual)? Or is immediate email on every booking sufficient for the initial implementation?
	- Mason- They should get email for every booking.

7. **Existing manual availability vs template generation**: When generating availability from a template, what happens to existing manually-created availability blocks for the same dates? Should they be preserved (taking precedence), deleted, or should generation skip those dates entirely?
	- Mason- Let's skip template generation for days that manually have availability blocks. We also should have a w