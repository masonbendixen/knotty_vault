---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/5/2026
Version: 0.1
tags: 
---
# Overview

Please go into plan mode. Please use this document for your planning document. Do not create anything in .claude/plans. Use this file for planning and do not ask me for permission to modify this file. It is your plan file. Please leave this Overview section in tact and do your work in the sections below.

We want to add classes to the system with photo support. The classes table already exists on the server side with photo support. You should be able to fetch classes with the existing /api/get_table_rows/ endpoint and fetch scaled photos using the existing support. 

Let's modify the existing Our Classes menu so that it keeps the all classes first item but then populates the rest of the menu with the names of all of the classes. Let's have a page for all classes that has cards for each class with a photo to the left and the name and description to the right. Clicking on a class should take you to the page entry for that class in the same way that clicking the same item in the menu does. On the page for a given class, you should show the photo on top with the name and then the description below. Please be sure to add tests for any new code.

Please use the codebase and this document to generate your plan. Please create phases of implementation with check boxes next to them. Please do the layered architecture with database schema changes, CRUD table helpers, business logic changes (this could probably be added to PersonHelper), the endpoint. There shouldn't be server side work items for this change. Then the client stuff with the types, network access layer, components, and then wiring into the system.

# Add plan here

## Design Notes & Decisions

These are design questions that came up during planning, answered before writing phases.

**How to detect photo presence?**
The generic `/api/get_table_rows/classes` endpoint returns only the table's own columns (`id`, `name`, `description`) — it does not join against `table_item_photos` to compute a `has_photo` flag. The instructors pattern has a dedicated `/api/get_instructors` endpoint that does this join server-side. Since no server-side work is in scope, `ClassData` will not include `has_photo`. Instead, templates will use `<img (error)="onPhotoError($event)">` to detect missing photos client-side and reveal a placeholder. When no photo is uploaded, the `/api/get_scaled_photo/classes/{id}/{w}/{h}` endpoint returns a non-2xx response, triggering the error handler. The user experience is identical to the `has_photo` approach.

**Route shape for individual class — `:id` or `:name`?**
Using `:id` (numeric). Class names have spaces that require URL encoding, which produces ugly URLs like `/classes/Partner%20Acrobatics`. Names are unique in the DB but IDs are simpler, more conventional, and already available in the data. Route: `/classes/:id`.

**How should `mockHeaderResponse` handle classes?**
Update the hardcoded entries in `mockHeaderResponse.ts` directly — change the 6 existing `ExternalLink` fragment items (`/classes#Name`) to `InternalLink` items pointing to `/classes/{id}`. No `HeaderService` changes, no server fetch, no rename. The class list in the nav is stable for a small studio and doesn't need to be database-driven. If classes are ever added or renamed, `mockHeaderResponse.ts` is a single obvious place to update.

**`ClassData.id` — `string` or `number`?**
`number`. The `DataResults` format from `get_table_rows` returns all values as strings (since it's a generic 2D string array), so the network layer parses `id` to `number` at the mapping step. This matches the backend `BIGSERIAL` type and is consistent with other domain types in the codebase.

---

## Server-Side Status (No Work Required)

The backend already has everything this feature needs:

- ✅ `classes` table exists with `id` (BIGSERIAL PK), `name` (STRING, unique), `description` (STRING)
- ✅ `classes` is registered in `photo_support_tables` — photo upload/retrieval is live
- ✅ `GET /api/get_scaled_photo/classes/{id}/{width}/{height}` — public, no auth, returns cached scaled image
- ✅ `POST /api/upload_photo/classes/{id}/jpeg|png` — admin auth, stores photo
- ✅ `POST /api/get_table_rows/classes` — returns all classes as `DataResults`
- ✅ `GET /api/get_row/classes/id/{id}` — returns a single class by id as `DataResults`

The instructors feature is the reference pattern for everything below.

---

## Phase 1: Types & ServerAccess Layer

*Establish the shared type and wire it through all layers of the network stack before touching any UI.*

### 1a. Relocate and Update `ClassData` Type
- [x] Move `ClassData` from `pages/public/services/classes/class-data.ts` to `shared/types/class.types.ts`
- [x] Update the type:
  ```ts
  export interface ClassData {
    id: number;       // was string; parse from DataResults in network layer
    name: string;
    description: string;
    // no has_photo — detected via <img (error)> fallback (see Design Notes)
  }
  ```
- [x] Update the import in `class-info.component.ts` to point to the new path (old import will be removed in Phase 6 cleanup anyway)
  - Also updated `classes.service.ts` to import from new path and changed id values to numbers

### 1b. Extend the `ServerAccess` Interface
- [x] Add to `shared/types/ServerAccess.ts`:
  ```ts
  getClasses(): Observable<ClassData[]>;
  getClass(id: number): Observable<ClassData>;
  ```

### 1c. Implement in `ServerAccessNetwork`
- [x] Implement `getClasses()`:
  - GET `/api/get_table_rows/classes`, receive `DataResults`
  - Map `sortedColumnNames` + `dataTable` rows into `ClassData[]` (columns are alphabetically sorted: `description`, `id`, `name`)
- [x] Implement `getClass(id)`:
  - GET `/api/get_row/classes/id/{id}`, receive `DataResults`
  - Map single row to `ClassData`, throw a 404-style error if `dataTable` is empty

### 1d. Update `ServerAccess.ts` Proxy
- [x] Add `getClasses()` and `getClass(id)` delegating through `serialize()` to the underlying implementation

### 1e. Implement in `ServerAccessMock`
- [x] Implement `getClasses()` — reads from `this.tables` (classes table), maps rows to `ClassData[]`
- [x] Implement `getClass(id)` — find by id, return the class or `throwError` with status 404 if not found

### 1f. Tests for Mock
- [x] Add to `ServerAccess.mock.spec.ts`:
  - `getClasses()` returns all 6 classes
  - Each class has correct `id` (number), `name`, and `description`
  - `getClass(id)` returns the correct class for a valid id
  - `getClass(id)` returns a 404 error for an unknown id

---

## Phase 2: Update the All-Classes Page (`ClassInfoComponent`)

*Replace the hardcoded service with a real server call and rebuild the layout to match the design: photo left, name + description right, card is clickable.*

### 2a. Component Logic
- [x] Update `class-info.component.ts`:
  - Remove `CLASSES_SOURCE_SERVICE_TOKEN` injection
  - Inject `@Inject(SERVER_ACCESS_TOKEN) private serverAccess: ServerAccess`
  - Add `loading = true` and `error = false` fields
  - In `ngOnInit()`, call `this.serverAccess.getClasses().subscribe({ next, error })` — set `loading = false` in both handlers
  - Add `getPhotoUrl(c: ClassData): string` → `` `/api/get_scaled_photo/classes/${c.id}/256/256` ``
  - Add `onPhotoError(event: Event)` — hide the `<img>` element and show its sibling placeholder `<div>`

### 2b. Template
- [x] Rewrite `class-info.component.html`:
  - `<mat-spinner>` shown while `loading` is true
  - Empty-state `<p>` shown when not loading and list is empty
  - `*ngFor` over `classInfoList` of `mat-card` elements, each with:
    - `[routerLink]="['/classes', classInfo.id]"` on the card (entire card is a link)
    - Left side (fixed width): `<img [src]="getPhotoUrl(classInfo)" (error)="onPhotoError($event)" class="class-photo">` + `<div class="photo-placeholder">` sibling (hidden by default, revealed on error)
    - Right side (flex-fill): `<h2>` for name, `<p>` for description
  - Add border to `mat-card` per project convention (`border: 1px solid #d1d5db`)

### 2c. Styles
- [x] Update `class-info.component.scss`:
  - Card uses flex row layout
  - Photo side: fixed width (180px), photo fills container with `object-fit: cover`
  - Placeholder: same dimensions as photo, grey background with centered icon
  - Content side: flex-grow, padding

### 2d. Tests
- [x] Write/update `class-info.component.spec.ts` using the `createComponent(overrides)` factory pattern from `instructors.component.spec.ts`:
  - Shows loading spinner before data arrives
  - Renders one card per class after data loads
  - Each card's `routerLink` points to `/classes/{id}`
  - Photo `src` attribute matches expected URL pattern
  - Empty-state message shown when `getClasses()` returns `[]`
  - Loading stops and no crash when `getClasses()` errors

---

## Phase 3: Individual Class Detail Page (New)

*New route and component for a single class. Photo on top, name, description below.*

### 3a. Route
- [x] Add to `public.routes.ts`:
  ```ts
  { path: 'classes/:id', component: ClassDetailComponent }
  ```
  *(Place after the existing `classes` route so both resolve correctly)*

### 3b. Component
- [x] Create directory `pages/public/class-detail/`
- [x] Create `class-detail.component.ts`:
  - Inject `ActivatedRoute` and `@Inject(SERVER_ACCESS_TOKEN) private serverAccess: ServerAccess`
  - In `ngOnInit()`, read `route.snapshot.paramMap.get('id')`, call `getClass(+id)`
  - Fields: `classData: ClassData | null = null`, `loading = true`, `notFound = false`
  - Add `getPhotoUrl(): string` → `/api/get_scaled_photo/classes/${this.classData!.id}/600/400` (larger size for detail view)
  - Add `onPhotoError(event: Event)` — same hide/show pattern as Phase 2

### 3c. Template
- [x] Create `class-detail.component.html`:
  - Loading spinner while `loading`
  - Not-found message when `notFound`
  - When data is loaded:
    - Full-width photo block at top: `<img>` with `(error)` fallback to placeholder
    - `<h1>` class name below photo
    - `<p>` description below name
    - Back link: `<a [routerLink]="['/classes']">← All Classes</a>`

### 3d. Styles
- [x] Create `class-detail.component.scss`:
  - Photo block: full width, max-height 350px, `object-fit: cover`
  - Content area: centered, max-width 800px, padding

### 3e. Tests
- [x] Create `class-detail.component.spec.ts`:
  - Shows loading spinner initially
  - Renders class name and description after fetch
  - Photo `src` matches expected URL
  - Shows not-found state when `getClass()` returns 404 error
  - Back link points to `/classes`

---

## Phase 4: Update Navigation Menu

*Update the hardcoded "Our Classes" dropdown to link to the new class detail pages. No HeaderService changes needed.*

- [x] In `mockHeaderResponse.ts`, replace the 6 `ExternalLink` items in `classDropdown.menu` with `InternalLink` items using the correct `/classes/:id` routes:
  ```ts
  { title: 'Knotty Yoga',                kind: HeaderButtonKind.InternalLink, goTo: '/classes/1' },
  { title: 'Therapeutic Knotty Yoga',    kind: HeaderButtonKind.InternalLink, goTo: '/classes/2' },
  { title: 'Partner Acrobatics',         kind: HeaderButtonKind.InternalLink, goTo: '/classes/3' },
  { title: 'Tumbling',                   kind: HeaderButtonKind.InternalLink, goTo: '/classes/4' },
  { title: 'Handstands',                 kind: HeaderButtonKind.InternalLink, goTo: '/classes/5' },
  { title: 'Aerial Fabric',              kind: HeaderButtonKind.InternalLink, goTo: '/classes/6' },
  ```
  *(IDs 1–6 match the seeded database rows. "All Classes" first item is unchanged.)*

---

## Phase 5: Cleanup

*Remove dead code left behind by the refactor.*

- [x] Delete `pages/public/services/classes/classes.service.ts` (the `ClassesSourceService` / `ClassesSourceServiceImpl` / `CLASSES_SOURCE_SERVICE_TOKEN`)
- [x] Delete `pages/public/services/classes/class-data.ts` (type moved to `shared/types/class.types.ts` in Phase 1)
- [x] Delete `pages/public/services/classes/classes.service.spec.ts`
- [x] Delete `pages/public/services/classes/` directory (now empty)
- [x] Confirmed `angular.json` has no `fileReplacements` referencing the deleted files
- [x] Searched codebase — no remaining imports of `CLASSES_SOURCE_SERVICE_TOKEN` or the old `class-data` path