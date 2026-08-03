---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/3/2026
Version: 0.4
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I'd like to add a blog to the website. I'd like to add an author_blog permission that allows CRUD style blog post creation. Blog posts should be stored in a table called blog_posts. Besides an auto incrementing 64 bit integer id called id, they should have a name, author, body, bool draft, created_at_us, modifed_at_us, and a post_at_us. In addition, the table should be marked as having photo support.

On the client side, under the admin tab, their should be a new menu item called Blog Posts that is guarded by the author_blog permission. Going to this page should have a row of blog posts with pagination sorted by created_at_us. The user should be able to choose a year and month and have an option for All Years and All Months (choosing All Years automatically selects All Months) that let you filter posts. There should be edit and delete icons for each post.

There should be a New Post button that brings up a blog authoring screen. There should be fields for Name, Author, a bool checkbox for Draft, and a date / time picker to choose when to post and then a button for Post Now that sets the post to post immediately at the current time. Then there should be a Save Post and Cancel buttons. Below these, there should be a left and right pane for authoring. To the left we should host ngx-markdown with syntax highlighting for authoring markdown and then on the right, we should show the fully rendered markdown. Clicking Save Post should save this entry to the database. The Edit button in the grid control should bring up the same page but with the existing data from the existing post populating the controls.

There should be a new top level menu entry that required no permissions called Blog. Clicking this should bring the latest Blog posts up that shows the five most recent blog posts with next and previous posts buttons at the bottom. At the top of the page, there should be buttons for navigating year / month / day with all three only having entries for items for which there are blog posts as well as an option for All Years / All Months / All Days. Selecting All Years automatically chooses All Months / All Days. Choosing All Months automatically chooses All Days. Blog posts should be rendered with the Name as a title, then the author, the date modified last, and then the body converted to HTML to be displayed. If the current user has the author_blog permission, we should also show a button Edit Post / Delete Post. Delete Post should prompt for confirmation.

# Blog Feature with Markdown Support — Implementation Plan

> **Sections 1–4 (all server work) implemented 8/3/2026.** Gates green: **55 blog tests pass** in the Linux docker client (15 table-helper, 13 business-logic, 7 KVT, 20 endpoint), plus the honuware photo-write change with its own framework tests. Three things worth knowing, detailed inline:
> 1. 🐞 **Postgres type-inference bug caught by the tests.** A bare `$1 = 0` guard types the parameter as **int4**, so a real microsecond timestamp overflows at bind time. It only fires once a caller passes a non-zero bound — the unbounded feed tests passed while every filtered one threw. Every parameter is now cast explicitly (`$1::bigint`). See 2.1.
> 2. **`table_item_photos.table_name` is a FOREIGN KEY into `photo_support_tables`**, so nothing can be attached to a table that isn't registered for photo support. Real databases get the row from `create_database.cpp`; test databases have the tables but not the seed rows, so photo tests must register it themselves. See 3.1.
> 3. **The plan named the wrong file for table registration.** `MakeBlogPostsTable` goes in `make_app_tables.cpp` (the app half), not `make_database_info.cpp` (which just stacks framework + app). See 1.2.

> **v0.3 (8/3/2026):** Mason answered open questions 1–7 (all confirmed as planned, recorded inline) and added a requirement: **per-post photos with CRUD, displayed above each post** — captured as decision 11 and threaded through 3.1 / new 4.2 / 4.1 / 5.x / 6.1 / 7.2, the sequence, the gates, and the hand-testing steps. The photo-write investigation surfaced a real framework gap: the generic `upload_photo`/`delete_photo` endpoints hardcode **admin-only**, so a non-admin `author_blog` holder couldn't manage blog photos — fixed by a small honuware change (Phase 4.2), with a warning that `IsTableAllowed` is NOT the right check to reuse there.

> **Re-grounded against the live codebase 8/3/2026 (v0.2).** The v0.1 plan was written in March and had drifted. Material corrections in this revision — details inline in each phase:
> 1. **Angular is 21, not 19** → `ngx-markdown@^21`, not `^19` (5.1).
> 2. **Permissions are seeded and referenced by NAME, not literal id** (`PermissionIdByName` / the `Grant(role, permission)` lambda) — the v0.1 "becomes permission ID 6" hardcoding would be wrong (1.2).
> 3. **The v0.1 `web_app.cpp` anchor pattern (`auto g_X = &Endpoints::X;`) is the documented BROKEN pattern** — file-scope anchors get dead-stripped at `-O2` and every route 404s in Release. Anchors go inside `RegisterAllEndpoints()` through the `volatile` store (4.1).
> 4. **Three registration functions were missing** from 1.2: `PopulateAdminColumnFriendlyNames`, `PopulateAdminTableFriendlyNames`, `PopulateAdminTableDisplayTemplates`.
> 5. **Guards and auth types moved into `@honuware/ui/auth`** — `core/guards/auth-guards.ts` and `core/services/auth.types.ts` are re-export shims now. `AuthorBlogGuard`/`hasAuthorBlog` are app-side additions to those shim files, not library changes (5.3).
> 6. **The admin list can't use `getFilteredTableRows`** — its `FilterPair[]` is equality-only, and the year/month filter is a *range* on `created_at_us`. The admin list uses the custom list endpoint with an admin view instead (4.1, 7.1).
> 7. **A KVT can't carry an array** — `BlogPostListResultToKeyValueTable` as specced is unrepresentable. Converters produce `KeyValueTableArray`; the endpoint composes `{items, total_count}` JSON at the edge (3.1).
> 8. **The endpoint permission check is `endpointAuthHelper.RequirePermission(transaction, name, resp)`** (401 anonymous / 403 missing), not `session.ActiveUserHasPermission` (4.1).
> 9. **Spec files were missing throughout** — every new component, the mock seam, and the header menu specs now have explicit test items (house rule: tests ship in the same session as the change).
> 10. **The verification plan built C++ on Windows** — forbidden; the gate is the Linux docker client (Gates section).
> 11. **A migration system now exists** (`business_logic/migration/`) but both streams are deliberately empty pre-deploy — a new table needs **no migration**, just `create_database.cpp` + a DB recreate. Noted in 1.2 so nobody adds one out of caution.

## Context

Adding a blog to the Knotty Yoga website. Blog posts are authored by users with the `author_blog` permission via a markdown editor with live preview. Published posts are viewable by anyone on a public `/blog` page with year/month/day filtering and pagination. The blog requires a new database table, custom API endpoints, a new permission, and both admin and public Angular pages.

**Tenancy note:** `blog_posts` is tenant data — it goes through the normal `make_database_info` / `create_database.cpp` path, so every tenant DB gets it automatically (`--recreate_database` / `--create_tenant` both run this code).

---

## Open Questions and Decisions

### Resolved

1. **Generic CRUD vs custom endpoints?** — **Custom endpoints for everything user-facing.** The public blog needs pagination with date filtering, draft exclusion, and a "get available dates" query that generic CRUD cannot handle; the admin list needs a *range* filter on `created_at_us`, which `getFilteredTableRows`' equality-only `FilterPair[]` cannot express either (v0.1 got this wrong). One list endpoint serves both via a `view=admin` flag. The generic CRUD registration still happens (1.2) so the Manage Data debug editor and the photo endpoints work against the table.
	- Mason- I want custom end points / UI.

2. **Admin route location?** — **New top-level route `/blog-admin`** with `[AuthGuard, AuthorBlogGuard]`. The `/manage` route requires `ManageProductsGuard` which is a different permission — blog authors shouldn't need `manage_products`.
	- Mason- Add a new permission.

3. **Author field type?** — **Free-text string.** The spec defines `author` as a string. Authors may want pen names. Default to logged-in user's name in the editor.
	- Mason- This sounds fine.

4. **Markdown editor approach?** — **Plain `<textarea>` on left, `<markdown>` preview on right using ngx-markdown.** No heavyweight CodeMirror/Monaco. No PrismJS — blog posts don't need code block syntax highlighting. ngx-markdown's HTML sanitization stays **on** (the default); author_blog holders are trusted staff, but there's no reason to disable it.
	- Mason- This sounds fine.

5. **Draft handling?** — Filter `draft=true` posts out of all public queries via SQL. Drafts visible only in the admin list. **Also unpublished-by-omission:** a post whose `post_at_us` is NULL (never scheduled) or in the future is likewise excluded from public queries — "published" = `draft = false AND post_at_us IS NOT NULL AND post_at_us <= now_us()`.
	- Mason- This sounds like a good idea.

6. **"Post Now" behavior?** — Sets `post_at_us = now` AND `draft = false`, then saves. Convenience shortcut for "publish immediately."
	- Mason- This sounds reasonable.

7. **Column name typo?** — Spec says "modifed_at_us" — use correct spelling `modified_at_us`.
	- Mason- Yes, please correct this typo.

### Resolved (User-Confirmed)

8. **Day-level filtering on public blog:** — **Client-side computation.** Derive available days from the posts loaded for the selected year/month. The dates endpoint returns year/month pairs only. Day options are extracted from `post_at_us` of currently loaded posts. *(Known limit, accepted: with more than one page in a month, the day dropdown only offers days present on the loaded page.)*

9. **ngx-markdown compatibility:** — **The app is Angular 21.2 today** (v0.1 said 19 — stale). ngx-markdown's major tracks Angular's major: install `ngx-markdown@^21.0.0` plus whatever `marked` major its peerDependencies declare (npm reports it at install). Uses `provideMarkdown()` for standalone app setup. If the pending Angular 22 upgrade lands before this plan executes, use `^22` instead.

10. **Timezone for year/month bucketing:** — **UTC on both sides.** The server extracts year/month from `post_at_us` in UTC; the client derives day options and computes filter ranges with the `Date` UTC accessors. (Posts are stamped with real instants — `Date.now()*1000` — so a post published late evening Pacific can bucket into the next UTC day/month. Accepted: consistency matters more than the edge, and the alternative drags studio-timezone plumbing into a blog.)

11. **Blog post photos (Mason, 8/3/2026):** — *"Please update the design to add being able to upload a photo for each blog post (have CRUD functionality for the photos) and then display the photo above each blog post."* — **Design: one photo per post through the existing generic photo system** (`table_item_photos` keyed by `(blog_posts, id)`), which the table's photo-support registration (1.2) already plugs into. No new storage, no new tables.
    - **Create/Update (replace):** the editor hosts the library `hw-photo-upload` control (`@honuware/ui/photos`) — live against the post id in edit mode; in create mode with `[deferUpload]="true"`, flushing via `uploadPendingPhoto('blog_posts', id)` after the row is created (the exact pattern `manage/instructors` already uses).
    - **Delete:** the control has no remove affordance (verified in its d.ts), so the editor adds a **Remove photo** button → confirm → `deletePhoto('blog_posts', id)`.
    - **Read/display:** `BlogPostInfo` carries `has_photo` (resolved via `TableHelpers::TableItemPhotos::HasPhoto`, the `SeriesInfo.classHasPhoto` precedent); the public page renders `/api/get_scaled_photo/blog_posts/{id}/...` above the post when set. Photo **reads** are already public (public pages use `get_scaled_photo` in `<img>` tags today).
    - **The catch — photo WRITES are admin-only today:** honuware's `upload_photo.cpp` hardcodes `session.IsAdmin` for general-table uploads and `delete_photo.cpp` allows only admin (or your own `people` photo). A non-admin `author_blog` holder couldn't manage blog photos. Fixed in **Phase 4.2** (small honuware change aligning photo-write auth with the `admin_table_permissions` model the CRUD endpoints use).

---

## Section 1: Backend — Database Schema & Registration

### Phase 1.1: Blog Posts Table Definition — **DONE (8/3/2026)**

- [x] Create `server/knottyyoga_server/src/db_schema/blog_posts.h`
- [x] Create `server/knottyyoga_server/src/db_schema/blog_posts.cpp`
- [x] Add both to `db_schema/CMakeLists.txt` (`target_sources(knotty_yoga_core PRIVATE ...)`, header before cpp)

**blog_posts.h** — constants (pattern: `db_schema/home_page_photos.h`):
- `kBlogPostsTable = "blog_posts"`
- `kBlogPostsId = "id"`, `kBlogPostsName = "name"`, `kBlogPostsAuthor = "author"`, `kBlogPostsBody = "body"`, `kBlogPostsDraft = "draft"`, `kBlogPostsCreatedAtUs = "created_at_us"`, `kBlogPostsModifiedAtUs = "modified_at_us"`, `kBlogPostsPostAtUs = "post_at_us"`
- `kPermissionAuthorBlog = "author_blog"` — app-side permission name constant. (The framework permission constants live in honuware's `components/platform/db_schema/permissions.h`; `author_blog` is app-specific, so its constant lives here with the table it gates rather than requiring a honuware pin bump.)
- Declare `void MakeBlogPostsTable(DatabaseInfo databaseInfo);`

**blog_posts.cpp** — verified against `database_info.h`:
- `AddTable(kBlogPostsTable)`
- `AddColumnPrimaryKey(..., DB_TYPE_BIGSERIAL)` — id
- `AddColumnSimple(..., DB_TYPE_STRING)` — name, author, body (not-null; STRING maps to PG TEXT, fine for the body)
- `AddColumnNotNullableWithDefault(..., DB_TYPE_BOOL, kDatabaseInfoDefaultTrue)` — draft (new posts default to draft)
- `AddColumnNotNullableWithDefault(..., DB_TYPE_BIGINT, kDatabaseInfoDefaultNow)` — created_at_us, modified_at_us
- `AddColumnNullable(..., DB_TYPE_BIGINT)` — post_at_us (NULL = not scheduled)

### Phase 1.2: Registration in create_database.cpp — **DONE (8/3/2026)**

> **Plan correction:** the table builder goes in **`db_schema/make_app_tables.cpp`**, not `make_database_info.cpp`. The latter is just the composition root that stacks `MakeFrameworkTables` then `MakeAppTables`; the app's per-table calls all live in the app builder.

All in `server/knottyyoga_server/src/database_helper/create_database.cpp` unless noted. **Permissions and roles are referenced BY NAME** (`PermissionIdByName` / `RoleIdByName`) — ids renumber per tenant, never hardcode one.

- [x] `db_schema/make_app_tables.cpp` *(not `make_database_info.cpp` — see above)*: `#include "blog_posts.h"` + `MakeBlogPostsTable(databaseInfo);` under a `// Blog tables` comment. No FK dependencies, so it sits last.
- [x] `create_database.cpp` includes: `#include "db_schema/blog_posts.h"`
- [x] `CreateTables()`: `CreateTable(DbSchema::kBlogPostsTable);`
- [x] `PopulatePhotoSupportTables()`: `AddRow(DbSchema::kBlogPostsTable);` — this is what "photo support" means; the generic photo endpoints (`upload_photo` / `get_photo` / `get_scaled_photo` / `has_photo`) then accept `blog_posts` rows
- [x] `PopulatePermissions()`: `AddRow(DbSchema::kPermissionAuthorBlog, "Permission to author and manage blog posts.");` — NOT pricing-eligible (omit the third arg)
- [x] `PopulateRolePermissions()`: `Grant(DbSchema::kRoleNameAdmin, DbSchema::kPermissionAuthorBlog);` — admins can author. (Studio Manager: leave out; grant later if wanted.)
- [x] `PopulateAdminTopLevelTables()`: `AddRow(DbSchema::kBlogPostsTable);` — required or the generic CRUD/photo endpoints reject the table ("Table is not an allowed table")
- [x] `PopulateAdminTablePermissions()`: `AddRow(DbSchema::kBlogPostsTable, std::stoi(PermissionIdByName(transaction, DbSchema::kPermissionAuthorBlog)));` — follows the `kManageProductsPermissionId` lookup pattern already in that function. **This row does double duty:** it grants non-admin `author_blog` holders the generic CRUD surface for the table AND (after Phase 4.2) photo upload/delete against `blog_posts` rows.
- [x] `PopulateAdminColumnDataInfo()` — signature is `(table, column, label, hint, htmlInputType, required, hidden="", readonly_="")`:
  - name → "Title", "Blog post title", "text", "true"
  - author → "Author", "Post author name", "text", "true"
  - body → "Body", "Blog post content (Markdown)", "text", "true"
  - draft → "Draft", "Hidden from the public blog while checked", "checkbox", "false"
  - created_at_us → "Created", "When created", "date", "false", "", "true" (readonly)
  - modified_at_us → "Modified", "When last modified", "date", "false", "", "true" (readonly)
  - post_at_us → "Post Date", "When the post goes public", "date", "false"
- [x] `PopulateAdminColumnFriendlyNames()` — *(missing from v0.1)* grid headers: Title, Author, Draft, Created, Modified, Post Date
- [x] `PopulateAdminTableFriendlyNames()` — *(missing from v0.1)* `AddRow(DbSchema::kBlogPostsTable, "Blog Posts", "Blog posts authored via the Blog Posts admin page. Debug view; author via the blog editor.");`
- [x] `PopulateAdminTableDisplayTemplates()` — *(missing from v0.1)* `AddRow(DbSchema::kBlogPostsTable, "{name}");`
- [x] **No migration.** `business_logic/migration/` exists but both streams are deliberately empty pre-deploy (`all_migrations.cpp`: "Both are empty pre-deploy"). New table = create_database + recreate the dev DB with `knottyyoga_database_helper --recreate_database` (needs `HONUWARE_ALLOW_DESTRUCTIVE=1`).
- [x] No blog seed data — the blog starts empty.

---

## Section 2: Backend — Table Helper

### Phase 2.1: Blog Posts Table Helper + Tests — **DONE (8/3/2026, 15 tests)**

- [x] Create `sql_util/table_helpers/blog_posts.h` / `.cpp` — class `TableHelpers::BlogPosts`
- [x] Create `sql_util/table_helpers/blog_posts_test.cpp` — suite `BlogPostsTest`
- [x] Add all three to `sql_util/table_helpers/CMakeLists.txt` (h+cpp to core, test to tests)

> 🐞 **Postgres typed the range parameters as int4 and every filtered query threw.** The window guard was written `($1 = 0 OR post_at_us >= $1)`; Postgres infers a parameter's type from its **first** use, so the literal `0` typed `$1` as **int4** and a real microsecond timestamp (~1.8e15) overflowed at bind time. The failure is invisible until a caller passes a non-zero bound — the three unbounded tests passed while all three filtered ones threw. Fixed by casting **every** parameter explicitly (`$1::bigint`, `LIMIT NULLIF($3::bigint, 0) OFFSET $4::bigint`). Worth remembering for any future `param = 0` sentinel against a BIGINT column.
>
> **Also:** `UpdateBlogPost` writes `now_us()` and `NULL` as raw SQL, which needs `DbCrud::UpdateRow`'s `allowedSqlKeywords` set (`{ "now_us()", "NULL" }`) — without it they bind as the literal strings and clearing a post date silently fails. A test pins that (`UpdateWithZeroPostAtClearsTheDate`).

**Methods:**
```
BlogPosts(DatabaseHelper databaseHelper)
int64_t AddBlogPost(tx, name, author, body, draft, postAtUs)        // postAtUs 0 → NULL
KeyValueTable GetBlogPost(tx, id)
void UpdateBlogPost(tx, id, name, author, body, draft, postAtUs)    // also stamps modified_at_us = now_us()
void DeleteBlogPost(tx, id)
// Public feed — published = draft=false AND post_at_us IS NOT NULL AND post_at_us <= now_us();
// ORDER BY post_at_us DESC. startUs/endUs of 0 = unbounded.
KeyValueTableArray GetPublishedPosts(tx, startUs, endUs, pageSize, page)
int64_t GetPublishedPostCount(tx, startUs, endUs)
// Admin list — every post incl. drafts; range + sort on created_at_us DESC.
KeyValueTableArray GetAllPosts(tx, startUs, endUs, pageSize, page)
int64_t GetAllPostCount(tx, startUs, endUs)
// Distinct UTC (year, month) pairs, newest first. forAdmin=false → published posts
// bucketed on post_at_us; forAdmin=true → all posts bucketed on created_at_us.
KeyValueTableArray GetAvailableDates(tx, bool forAdmin)
```

- [x] Standard CRUD through `DbCrud::*` (`AddRowToTableFetchInt64PrimaryKey`, `GetRow`, `UpdateRow`, `DeleteRow`); the published/admin/date queries are custom SQL — follow the `kSql...` constant + `$1/$2` parameter pattern in `sql_util/table_helpers/price_schedules.cpp` (incl. the `COUNT(*) OVER()`/`LIMIT NULLIF($n, 0)` idioms if convenient)
- [x] Available-dates SQL: `SELECT DISTINCT EXTRACT(YEAR FROM to_timestamp(post_at_us / 1000000.0) AT TIME ZONE 'UTC')::int AS year, EXTRACT(MONTH FROM ...)::int AS month FROM blog_posts WHERE <published> ORDER BY 1 DESC, 2 DESC` (UTC per decision 10)
- [x] Tests — 15 cases: add/get round-trip, NULL post date, missing id, update rewrite + modified advancing, clearing the date unpublishes, delete, the three-condition published rule, newest-first order, pagination without overlap, page-size 0, half-open range, unbounded zero bounds, admin range on created_at_us, distinct date pairs, public dates ignoring unpublished. *(Bool columns come back as `"t"`/`"f"` from Postgres, not `"true"`/`"false"` — the tests use a tolerant `BoolField` helper rather than pinning the driver's spelling.)*

---

## Section 3: Backend — Business Logic Layer

### Phase 3.1: Blog Helper + Key Value Table + Tests — **DONE (8/3/2026, 13 + 7 tests)**

- [x] Create directory `business_logic/blog/` with `CMakeLists.txt`
- [x] `business_logic/CMakeLists.txt`: `add_subdirectory(blog)`
- [x] Create `blog_helper.h` / `.cpp` + `blog_helper_test.cpp` (suite `BlogHelperTest`)
- [x] Create `blog_key_value_table.h` / `.cpp` + `blog_key_value_table_test.cpp` (suite `BlogKeyValueTableTest`)
- [x] **Added beyond the plan:** a free `Blog::IsPublished(post, nowUs)` stating the three-condition rule in domain terms, so the single-post endpoint can gate an unpublished post without re-deriving it. `SaveBlogPostRequest` (create and update take the same shape) so the two endpoints share one parse. `DeletePost` also drops the post's `table_item_photos` row — an orphan would otherwise be handed to whatever post later reuses the id.

> ⚠️ **`table_item_photos.table_name` is a FOREIGN KEY into `photo_support_tables`.** Nothing can be attached to a table that isn't registered for photo support. Real databases get the row from `PopulatePhotoSupportTables`; a **test** database has the tables but not the seed rows, so the two photo tests call `TableHelpers::PhotoSupportTables::AddPhotoSupportTable(tx, "blog_posts")` first. Both failed with an opaque `unknown file: Failure` until that was added — worth knowing before writing any other photo test.

**Domain structs** (in `blog_helper.h`, `namespace Blog`):
```cpp
struct BlogPostInfo {
    int64_t id = 0;
    std::string name, author, body;
    bool draft = true;
    bool hasPhoto = false;  // decision 11 — resolved from table_item_photos
    int64_t createdAtUs = 0, modifiedAtUs = 0;
    int64_t postAtUs = 0;   // 0 = not scheduled (NULL in the DB)
};
struct BlogPostListResult { std::vector<BlogPostInfo> posts; int64_t totalCount = 0; };
struct AvailableDate { int year = 0; int month = 0; };
```

**BlogHelper** wraps `TableHelpers::BlogPosts`, converts rows → structs. Methods mirror the table helper (`GetPublishedPosts` / `GetAllPosts` returning `BlogPostListResult`, `GetAvailableDates(forAdmin)`, CRUD returning `BlogPostInfo`). Resolves `hasPhoto` per post via `TableHelpers::TableItemPhotos::HasPhoto(tx, kBlogPostsTable, id)` — the `SeriesInfo.classHasPhoto` precedent (`class_series_helper.cpp`); helper test covers with/without a photo row.

**blog_key_value_table** — *(v0.1 correction: a KVT is a flat string map and cannot hold an array; list responses are composed at the endpoint edge like every other list in the codebase)*:
- `KeyValueTable BlogPostInfoToKeyValueTable(const BlogPostInfo&)` — emits every column plus `has_photo`; booleans as `"true"/"false"`, `post_at_us` as `""` when 0
- `KeyValueTableArray BlogPostsToKeyValueTableArray(const std::vector<BlogPostInfo>&)`
- `KeyValueTable AvailableDateToKeyValueTable(const AvailableDate&)`
- `KeyValueTableArray AvailableDatesToKeyValueTableArray(const std::vector<AvailableDate>&)`
- [x] KVT tests — 7 cases: every field, booleans as strings, the `""`-for-unscheduled rule, defaults, post-array order, available-date conversion + order

---

## Section 4: Backend — Endpoints

### Phase 4.1: Blog Endpoints + Tests

- [ ] Create `endpoints/blog_posts.h` / `.cpp` — one file, multiple routes in one `SetupRouting::AddRoute` (pattern: `endpoints/admin_coupons.cpp`, which registers 4 routes)
- [ ] Create `endpoints/blog_posts_test.cpp` — suite `BlogPostsEndpointTest`
- [ ] Add h/cpp to core + test to tests in `endpoints/CMakeLists.txt` — **then prove the test suite actually runs**: `--gtest_filter=BlogPostsEndpointTest.*` must report a non-zero count (`0 tests` still exits 0 — this repo has already shipped a never-registered endpoint test once)

**Routes:**

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | `/api/blog_posts` | none / `author_blog` for `view=admin` | List. Params: `page`, `page_size` (cap ≤ 50), `start_us`, `end_us`, `view=admin`. Public: published only, `post_at_us` DESC, range on `post_at_us`. Admin: drafts included, `created_at_us` DESC, range on `created_at_us`. |
| GET | `/api/blog_posts/dates` | none / `author_blog` for `view=admin` | Distinct year/month pairs (`{dates: [...]}`). Public buckets published `post_at_us`; admin buckets all `created_at_us`. |
| GET | `/api/blog_posts/<int>` | none for published | Single post. If the row is a draft (or unscheduled/future), gate with `RequirePermission` — published rows are public. Crow matches `dates` vs `<int>` by type, so the literal route doesn't collide. |
| POST | `/api/blog_posts` | `author_blog` | Create; returns the created post. |
| PUT | `/api/blog_posts/<int>` | `author_blog` | Update (stamps `modified_at_us`); returns the updated post. |
| DELETE | `/api/blog_posts/<int>` | `author_blog` | Delete. |

**Structure** (per `endpoints/CLAUDE.md` — handlers + `SetupRouting` + exported functions only, no helpers in the endpoint file):
- Exported: `GetBlogPosts`, `GetBlogPostDates`, `GetBlogPost`, `PostBlogPost`, `PutBlogPost`, `DeleteBlogPost`
- Permission check: `endpointAuthHelper.RequirePermission(transaction, DbSchema::kPermissionAuthorBlog, resp)` — 401 anonymous / 403 missing, writes resp itself *(v0.1's `session.ActiveUserHasPermission` doesn't exist)*
- List composition at the edge (pattern: `admin_attendance_templates.cpp` / the payments example): `Json::JsonObject{{"items", SqlUtil::KeyValueTableArrayToJson(Blog::BlogPostsToKeyValueTableArray(r.posts))}, {"total_count", ...}}`
- [ ] `endpoints/web_app.cpp`: `#include "blog_posts.h"` + **anchors INSIDE `RegisterAllEndpoints()`**: `anchor = reinterpret_cast<AnchorFunc>(&Endpoints::GetBlogPosts);` (one per exported function, following the file's convention). *(v0.1's file-scope `auto g_X = &...;` is the exact pattern the codebase banned: dead-stripped at `-O2`, every route 404s in Release.)*
- [ ] Endpoint tests (drive over HTTP via `EndpointTestHelper` + `handle_full`; query params via `req.url_params = crow::query_string(...)`, never `req.url`): public list excludes drafts/unscheduled/future + paginates + range-filters; admin view 401 anonymous, 403 without the permission, includes drafts for a holder (grant via the permissions table helper, as `admin_attendance_templates_test.cpp` does); dates both views; single GET public-vs-draft gating; POST/PUT/DELETE happy paths + 401/403; PUT advances `modified_at_us`; `has_photo` false by default and true after a `table_item_photos` row is seeded

### Phase 4.2: Honuware — photo WRITE endpoints honor table permissions (decision 11)

The generic photo write endpoints are **admin-only today** (verified 8/3/2026): `upload_photo.cpp` hardcodes `session.IsAdmin(transaction)` for general-table uploads ("Admin required for general table uploads"), and `delete_photo.cpp` allows only admin or your own `people` photo. Without this phase, a non-admin `author_blog` holder can author posts but not photos. Reads (`get_photo` / `get_scaled_photo` / `has_photo`) are public and stay untouched.

> ⚠️ **Do NOT "fix" this by swapping `IsAdmin` for `IsTableAllowed`.** `GetAllowedTables` (which backs `IsTableAllowed`) is the union of the app's **base public allow-list** and the per-permission grants — reusing it for writes would let ANY logged-in user upload photos to every base-allowed public table. The write check must be the *grants half only*.

All changes in the **server_components** repo (`components/platform/`); co-dev via `-DFETCHCONTENT_SOURCE_DIR_HONUWARE=...`, finish by pushing honuware and bumping the pinned `GIT_TAG` SHA in the app's top-level CMakeLists.

- [ ] `endpoints/endpoint_auth_helper.h/.cpp`: new `bool RequireTableWriteAccess(Transaction&, std::string_view tableName, crow::response& resp)` — 401 when not logged in; true for admin; for non-admins true iff the table is granted to them via `admin_table_permissions` (factor the grants lookup out of `GetAllowedTables` rather than duplicating it); else 403. Fail closed on lookup errors, like `RequirePermission` does.
- [ ] `endpoints/upload_photo.cpp`: general-table path uses `RequireTableWriteAccess` instead of the `IsAdmin` check (people self-upload path / `upload_user_photo` untouched).
- [ ] `endpoints/delete_photo.cpp`: same swap, keeping the "non-admin may delete their own `people` photo" carve-out.
- [ ] Framework tests (`honuware_tests`): non-admin with a table grant can upload + delete on that table; same user 403 on an ungranted table; plain logged-in user 403 on a base-allow-listed table (the `IsTableAllowed` trap, pinned by a test); anonymous 401; admin unchanged; people self-photo carve-out unchanged.
- [ ] Gate: honuware component suite green in ITS docker client, then the app suite green against the co-dev tree (both commands in the Gates section).

---

## Section 5: Frontend — Dependencies & Types

### Phase 5.1: Install ngx-markdown + Blog Types

- [ ] From `ui/`: `npm install ngx-markdown@^21.0.0 marked --save` — **Angular is 21.2 in package.json** *(v0.1 said 19 — stale)*; ngx-markdown's major must match. Take the `marked` major ngx-markdown's peerDependencies ask for (npm surfaces it). If the Angular 22 upgrade has landed by execution time, use `^22`.
- [ ] `ui/src/app/app.config.ts`: `import { provideMarkdown } from 'ngx-markdown';` + `provideMarkdown()` in the `providers` array (no loader config — we never fetch remote `.md` files)
- [ ] Create `ui/src/app/shared/types/blog.types.ts`:

```typescript
export interface BlogPost {
  id: number;
  name: string;
  author: string;
  body: string;
  draft: boolean;
  has_photo: boolean;          // decision 11 — drives the photo above the post
  created_at_us: number;
  modified_at_us: number;
  post_at_us: number | null;   // null = not scheduled
}
export interface BlogPostListResponse { items: BlogPost[]; total_count: number; }
export interface AvailableDate { year: number; month: number; }
export interface BlogDatesResponse { dates: AvailableDate[]; }
export interface SaveBlogPostRequest {   // create + update take the same shape
  name: string;
  author: string;
  body: string;
  draft: boolean;
  post_at_us: number | null;
}
```

### Phase 5.2: Server Access Interface + All Implementations

The seam is **five** files, not four — the mock's spec is part of the seam *(missing from v0.1)*.

- [ ] `shared/types/ServerAccess.ts` — import blog types, add 6 methods:
  - `getBlogPosts(page, pageSize, opts?: {startUs?, endUs?, adminView?}): Observable<BlogPostListResponse>`
  - `getBlogPost(id): Observable<BlogPost>`
  - `getBlogDates(adminView?: boolean): Observable<BlogDatesResponse>`
  - `createBlogPost(request: SaveBlogPostRequest): Observable<BlogPost>`
  - `updateBlogPost(id, request: SaveBlogPostRequest): Observable<BlogPost>`
  - `deleteBlogPost(id): Observable<void>`
- [ ] `shared/services/network/ServerAccessNetwork.ts` — implement against `/api/blog_posts*` with a private `normalizeBlogPost`: KVT→JSON serializes booleans as the strings `"true"/"false"` and the unscheduled `post_at_us` as `""` — coerce `draft` + `has_photo` via the existing `coerceBool`, `"" → null` for `post_at_us`, `Number(...)` the timestamps (same pattern as `normalizeSignupOffering`)
- [ ] `shared/services/network/ServerAccess.mock.ts` — `blogPosts: BlogPost[]` seeded with 3–5 posts **including one draft, one future-dated post, and one with `has_photo: true`** so local mode exercises the exclusion rules and the photo banner; in-memory implementations of all 6 (mock honors published-vs-admin filtering, sorting, pagination). The mock's existing `PhotoAccess` methods (`uploadPhoto`/`deletePhoto`/`hasPhoto`) should flip the matching post's `has_photo` when called with `blog_posts`, so the editor's photo flow works in local mode
- [ ] `shared/services/network/ServerAccess.ts` (proxy) — 6 `this.serialize(() => this.impl.method(...))` passthroughs
- [ ] `ServerAccess.mock.spec.ts` — cases: public list excludes the draft + the future post and sorts `post_at_us` DESC; admin view includes drafts sorted `created_at_us` DESC; pagination + `total_count`; date pairs; single get found/404; create/update/delete round-trip; update stamps `modified_at_us`; `has_photo` flips through the photo methods

### Phase 5.3: Auth Types & Guard

*(v0.1 correction: `core/services/auth.types.ts` and `core/guards/auth-guards.ts` are now one-line re-export shims over `@honuware/ui/auth`, where `hasPermission`, `hasManageProducts`, and all guards live. `author_blog` is app-specific, so its helper + guard are **app-side additions to the shim files** — no honuware release needed.)*

- [ ] `core/services/auth.types.ts`: add (alongside the re-export)
  ```typescript
  export function hasAuthorBlog(authData: AuthData): boolean {
    return (authData.isAuth && authData.isAdmin) || hasPermission(authData, 'author_blog');
  }
  ```
- [ ] `core/guards/auth-guards.ts`: add a functional `AuthorBlogGuard` (`CanActivateFn`) that injects `AuthService`, checks `hasAuthorBlog`, and redirects to `/` on failure — mirror the library's `ManageProductsGuard` behavior
- [ ] Spec coverage for both (a small `auth-guards.spec.ts` if none exists app-side): admin passes, `author_blog` holder passes, plain user redirected, anonymous redirected

---

## Section 6: Frontend — Public Blog View

### Phase 6.1: Public Blog List Page

- [ ] Create `pages/public/blog/blog-list/blog-list.component.ts` / `.html` / `.scss` / `.spec.ts` *(spec was missing from v0.1)*
- [ ] Standalone component; imports `SharedModule` + ngx-markdown's standalone `MarkdownComponent`; injects `SERVER_ACCESS_TOKEN`, `AuthService`, `MatDialog`, `Router`
- [ ] On init: `getBlogDates()` + `getBlogPosts(0, 5)`; page size 5 per the Overview
- [ ] **Date navigation** at top:
  - Year dropdown from the dates response + "All Years"; Month dropdown limited to months present for the chosen year + "All Months"; Day dropdown derived client-side from loaded posts + "All Days"
  - "All Years" forces "All Months" + "All Days"; "All Months" forces "All Days"
  - Selection computes a UTC `[startUs, endUs)` range (decision 10) and reloads page 0
- [ ] **Post rendering** per post: **the photo above everything when `has_photo`** — `<img src="/api/get_scaled_photo/blog_posts/{{post.id}}/1200/675">`, full content width, `max-width: 100%` (decision 11; reads are public, no auth needed); then name as title; "By {author} | {modified date}"; `<markdown [data]="post.body">` (sanitization on — default)
- [ ] **Author controls:** when `hasAuthorBlog(authData)` — Edit Post → `/blog-admin/edit/:id`; Delete Post → `ConfirmDialogComponent` from **`@honuware/ui/foundation`** *(v0.1 path `shared/components/confirm-dialog` no longer exists)* → `deleteBlogPost` → reload
- [ ] **Pagination:** Previous/Next from `total_count`, disabled at the bounds
- [ ] Styling: card borders `1px solid #d1d5db` per house rule
- [ ] Spec: renders posts as markdown (assert rendered HTML, e.g. `**bold**` → `<strong>`); photo `<img>` present with the scaled URL only when `has_photo`; date dropdown cascade rules; range passed to the fetch; pagination bounds; Edit/Delete only for `hasAuthorBlog`; delete confirms then reloads; empty state

### Phase 6.2: Blog Routes & Menu

- [ ] `pages/public/public.routes.ts`: `{ path: 'blog', component: BlogListComponent }`
- [ ] `shared/services/header/mockHeaderResponse.ts`: add a `blogButton` (`title: 'Blog'`, InternalLink, `goTo: '/blog'`) to the top-level menu between `eventsButton` and `membershipsButton` *(v0.1's "shopButton, around line 179" is stale — the Phase 1 menu rework renamed and reordered everything)*
- [ ] Update `mockHeaderResponse.spec.ts` + `header.service.spec.ts` — both assert menu contents and will need the new entry *(missing from v0.1)*

---

## Section 7: Frontend — Admin Blog Management

### Phase 7.1: Admin Blog List Page

- [ ] Create `pages/blog-admin/blog-list/blog-admin-list.component.ts` / `.html` / `.scss` / `.spec.ts` *(spec was missing from v0.1)*
- [ ] Data source: **`getBlogPosts(page, pageSize, {startUs, endUs, adminView: true})` + `getBlogDates(true)`** — *not* `getFilteredTableRows` as v0.1 planned: its `FilterPair[]` is equality-only and the year/month filter is a range on `created_at_us`; the custom endpoint also keeps drafts and the right sort
- [ ] Year/Month dropdowns with "All Years" / "All Months" ("All Years" forces "All Months"), computing a UTC `created_at_us` range
- [ ] `mat-table`: Title, Author, Draft (chip), Created, Post Date; edit icon → `/blog-admin/edit/:id`; delete icon → `ConfirmDialogComponent` → `deleteBlogPost` → reload
- [ ] "New Post" button → `/blog-admin/new`; paginator wired to `total_count`
- [ ] Spec: lists rows incl. drafts; filter range passed through; edit/delete/new navigation; delete confirm; pagination

### Phase 7.2: Blog Editor Page

- [ ] Create `pages/blog-admin/blog-editor/blog-editor.component.ts` / `.html` / `.scss` / `.spec.ts` *(spec was missing from v0.1)*
- [ ] Standalone; `SharedModule`, `MarkdownComponent`, `ReactiveFormsModule`, `PhotoUploadComponent` (`@honuware/ui/photos`)
- [ ] Form fields: Name (required), Author (required, defaults to `firstName + ' ' + lastName` from `authData` in create mode), Draft checkbox (defaults checked), Post At = **Material datepicker + `<input matInput type="time">`** — the established pair from `manage/events/event-create` *(house rule: dates use date pickers, times use hour pickers — no free-text)*; converts to/from microseconds, empty → `post_at_us: null`
- [ ] **Photo section (decision 11)** — `hw-photo-upload` with `[tableName]="'blog_posts'"`:
  - Edit mode: `[tableItemId]="postId"` — upload/replace live against the row.
  - Create mode: `[tableItemId]="0" [deferUpload]="true"`; after `createBlogPost` returns the id, flush via `@ViewChild(PhotoUploadComponent)` → `if (photoUpload?.hasPendingFile) photoUpload.uploadPendingPhoto('blog_posts', id)` before navigating — the exact `manage/instructors` create-flow pattern (`instructors-admin.component.ts`).
  - **Remove photo** button (edit mode, shown when the post has a photo) → `ConfirmDialogComponent` → `deletePhoto('blog_posts', postId)` — the control has no built-in remove (verified in the photos d.ts).
- [ ] Buttons: **Post Now** (sets Draft unchecked + Post At = now, then saves), **Save Post**, **Cancel** (→ `/blog-admin`, no save)
- [ ] Split pane: left `<textarea formControlName="body">`, right `<markdown [data]="...">` live preview; `display: flex; gap: 1rem;` panes `flex: 1`; preview scrolls independently
- [ ] Edit mode (`:id` param): load via `getBlogPost(id)`, populate; create mode (`new`): empty except author default
- [ ] Save → `createBlogPost` / `updateBlogPost` → navigate to `/blog-admin`
- [ ] Spec: create defaults (draft checked, author prefilled); edit populates from the fetch; Post Now flips draft + stamps now + saves; save maps date+time → microseconds and empty → null; cancel navigates without saving; preview renders the textarea content; create-with-pending-photo flushes `uploadPendingPhoto` with the new id; Remove photo confirms then calls `deletePhoto`. *(Mock `hasPhoto` in the spec's ServerAccess stub — `PhotoUploadComponent` calls it on init, as the instructors-admin spec notes.)*

### Phase 7.3: Admin Blog Routes & Menu

- [ ] Create `pages/blog-admin/blog-admin.routes.ts`:
  ```typescript
  const routes: Routes = [
    { path: '', component: BlogAdminListComponent },
    { path: 'new', component: BlogEditorComponent },
    { path: 'edit/:id', component: BlogEditorComponent },
  ];
  export default routes;
  ```
- [ ] `app.routes.ts`: after the `manage` block —
  ```typescript
  { path: 'blog-admin', loadChildren: () => import('@pages/blog-admin/blog-admin.routes'), canActivate: [AuthGuard, AuthorBlogGuard] },
  ```
- [ ] `mockHeaderResponse.ts`: import `hasAuthorBlog`; widen the Admin dropdown condition to `authData.isAuth && (authData.isAdmin || hasManageProducts(authData) || hasAuthorBlog(authData))`; push a "Blog Posts" item (`goTo: '/blog-admin'`) into `adminMenu` when `hasAuthorBlog(authData)` — note this makes the Admin dropdown appear for a user whose ONLY elevated permission is `author_blog`, containing just Blog Posts (intended)
- [ ] Update `mockHeaderResponse.spec.ts`: Blog Posts present for admin and for an `author_blog`-only user, absent otherwise; Admin dropdown appears for the `author_blog`-only user

---

## Implementation Sequence

Phase-level checklist (dependency order; mark off with the granular boxes above):

- [ ] **1.1** Blog posts table schema
- [ ] **1.2** create_database registration (needs 1.1)
- [ ] **2.1** Table helper + tests (needs 1.2)
- [ ] **3.1** Business logic + KVT + tests (needs 2.1)
- [ ] **4.1** Endpoints + tests + web_app anchors (needs 3.1)
- [ ] **4.2** Honuware: photo-write auth via table permissions (needs 1.2 to be meaningful; independent of 2.1–4.1 — admins can exercise the photo UX without it, non-admin authors need it)
- [ ] **5.1** ngx-markdown + blog types
- [ ] **5.2** ServerAccess seam ×5 files (needs 5.1)
- [ ] **5.3** hasAuthorBlog + AuthorBlogGuard
- [ ] **6.1** Public blog list page (needs 5.2)
- [ ] **6.2** Public route + Blog menu item (needs 6.1)
- [ ] **7.1** Admin blog list page (needs 5.2)
- [ ] **7.2** Blog editor page incl. the photo section (needs 5.2; non-admin photo writes need 4.2)
- [ ] **7.3** Admin routes + Admin-dropdown menu item (needs 5.3, 7.1, 7.2)

Backend (1.1–4.1) completes before frontend; 5.1/5.3 can start any time.

---

## Critical File Reference

### CREATE (Backend)
- `db_schema/blog_posts.h` + `.cpp`
- `sql_util/table_helpers/blog_posts.h` + `.cpp` + `_test.cpp`
- `business_logic/blog/CMakeLists.txt`, `blog_helper.h/.cpp/_test.cpp`, `blog_key_value_table.h/.cpp/_test.cpp`
- `endpoints/blog_posts.h` + `.cpp` + `_test.cpp`

### CREATE (Frontend)
- `shared/types/blog.types.ts`
- `pages/public/blog/blog-list/blog-list.component.{ts,html,scss,spec.ts}`
- `pages/blog-admin/blog-admin.routes.ts`
- `pages/blog-admin/blog-list/blog-admin-list.component.{ts,html,scss,spec.ts}`
- `pages/blog-admin/blog-editor/blog-editor.component.{ts,html,scss,spec.ts}`

### MODIFY (Backend)
- `db_schema/make_database_info.cpp`, `db_schema/CMakeLists.txt`
- `database_helper/create_database.cpp` — the 11 registration points in Phase 1.2
- `sql_util/table_helpers/CMakeLists.txt`, `business_logic/CMakeLists.txt`, `endpoints/CMakeLists.txt`
- `endpoints/web_app.cpp` — include + `RegisterAllEndpoints()` anchors

### MODIFY (Honuware — server_components repo, Phase 4.2)
- `components/platform/endpoints/endpoint_auth_helper.h/.cpp` — new `RequireTableWriteAccess`
- `components/platform/endpoints/upload_photo.cpp` + `delete_photo.cpp` — auth swap (+ their `_test.cpp` files)
- App top-level `CMakeLists.txt` — honuware `GIT_TAG` pin bump once pushed

### MODIFY (Frontend)
- `shared/types/ServerAccess.ts`, `shared/services/network/ServerAccessNetwork.ts`, `ServerAccess.mock.ts`, `ServerAccess.mock.spec.ts`, `ServerAccess.ts` (proxy)
- `core/services/auth.types.ts` (+ `hasAuthorBlog`), `core/guards/auth-guards.ts` (+ `AuthorBlogGuard`)
- `shared/services/header/mockHeaderResponse.ts` + `mockHeaderResponse.spec.ts` + `header.service.spec.ts`
- `app.routes.ts`, `pages/public/public.routes.ts`, `app.config.ts` (`provideMarkdown()`)
- `package.json` (ngx-markdown + marked); `angular.json` untouched

### Key existing pieces to reuse (verified 8/3/2026)
- `endpointAuthHelper.RequirePermission(transaction, name, resp)` — honuware `endpoint_auth_helper.h`
- `DbCrud::*` — `sql_util/database_access/database_crud_helpers.h`
- `SqlUtil::KeyValueTableToJson` / `KeyValueTableArrayToJson` — `sql_util/json/key_value_table_json.h`
- `PermissionIdByName` / `Grant` lambdas — already in `create_database.cpp`
- `hasPermission(authData, name)` — `@honuware/ui/auth` (via the `core/services/auth.types.ts` shim)
- `ConfirmDialogComponent` — `@honuware/ui/foundation`
- `coerceBool` — private helper already in `ServerAccessNetwork.ts`
- Date + time inputs — `pages/manage/events/event-create` (Material datepicker + `matInput type="time"`)
- `PhotoUploadComponent` (`hw-photo-upload`) — `@honuware/ui/photos`; defer-then-flush create pattern in `pages/manage/instructors/instructors-admin.component.ts`
- `TableHelpers::TableItemPhotos::HasPhoto` — the `has_photo` resolution (see `class_series_helper.cpp`)

---

## Gates & Verification

### Automated gates
- [ ] **C++ (the only build/test path — never build on Windows):** full Linux docker suite green —
  `MSYS_NO_PATHCONV=1 docker run --rm --network knotty-net -v "C:/Users/mason/source/repos/knottyyoga/server:/src" -v "C:/Users/mason/source/repos/server_components:/honuware" -v honuware-conan2:/root/.conan2 -v knottyyoga-linux-build:/build -e HONUWARE_SRC_DIR=/honuware -e HONUWARE_DB_SSLMODE=disable -w /src knottyyoga_build:latest bash docker_project/build_and_test.sh`
  During development, filter with `"--gtest_filter=BlogPostsTest.*:BlogHelperTest.*:BlogKeyValueTableTest.*:BlogPostsEndpointTest.*"` — and confirm the filtered run reports a **non-zero** test count (a test file missing from CMakeLists still exits 0).
- [ ] **Honuware (Phase 4.2 only):** the server_components suite green in its own docker client (`server_components/docker/build_and_test.sh` — exact `docker run` invocation in the `reference_linux_docker_build_clients` memory), then the app suite green against the co-dev tree before pushing + bumping the pin.
- [ ] **Angular** (bare commands from the `ui/` working directory): `npx tsc --noEmit -p tsconfig.app.json` + `-p tsconfig.spec.json` clean; `npx ng test --watch=false --browsers=ChromeHeadless` full suite green; `npx ng build` clean; `npx ng lint` no new findings vs baseline
- [ ] Database recreate succeeds: `knottyyoga_database_helper --recreate_database` (with `HONUWARE_ALLOW_DESTRUCTIVE=1`) — blog_posts created, author_blog present, photo support registered

### Live hand-testing (blank DB + real create_database seed data, via the web UI)
1. Sign in as the admin. The **Admin** dropdown now contains **Blog Posts**. The top-level menu shows **Blog** between **Upcoming Events** and **Memberships** — visible signed out too.
2. **Admin → Blog Posts** → **New Post**. Fill **Name** "Welcome to Knotty Yoga", leave **Author** prefilled with your name, leave **Draft** checked, no post date. Type markdown with a heading, bold, and a list in the left pane — the right pane renders it live. **Save Post** → back on the list, the row shows a **Draft** chip.
3. Open **Blog** (top menu): the page is empty — drafts don't show.
4. Edit the post: uncheck **Draft**, click **Post Now**, save. **Blog** now shows the post — title, "By {author}", the modified date, and the body rendered as HTML (heading/bold/list, not raw markdown).
5. Create a second post with **Draft** unchecked and a **Post At** date one month in the future. It appears in the admin list but **not** on **Blog** (scheduled, not yet published).
6. Create six more published posts (Post Now). **Blog** shows the **5** most recent with **Previous/Next** at the bottom; **Next** shows the older ones; **Previous** returns.
7. On **Blog**, pick a **Year** and **Month** from the dropdowns — only year/months that actually have posts are offered; the list filters. Pick a **Day** — the day list only offers days from the loaded posts. Choose **All Years** — Month and Day snap back to **All Months** / **All Days** and everything returns.
8. In the admin list, use **Year/Month** to filter by creation date; **All Years** forces **All Months**.
9. Signed in as admin on **Blog**, each post shows **Edit Post** / **Delete Post**. **Delete Post** prompts for confirmation; confirming removes it. Sign out — the buttons are gone.
10. **Photos (decision 11).** **Admin → Blog Posts → New Post**: fill in Name/body, choose a photo in the **Photo** control, **Post Now**, save — the photo uploads as part of the save (deferred until the row exists). Open **Blog**: the photo renders **above** that post's title, scaled to the content width. Posts without a photo show no image and no gap.
11. **Edit** the same post: the Photo control shows the current photo. Choose a different file — the photo is replaced (confirm on **Blog** after a reload). Click **Remove photo**, confirm — the control empties and the post on **Blog** renders with no image.
12. Via the test-helper app (or Manage Data), create a second user **without** `author_blog`: no **Blog Posts** menu item, `/blog-admin` redirects away, no Edit/Delete on **Blog**. Grant them `author_blog` (no other elevated permission): the **Admin** dropdown appears containing just **Blog Posts**, and authoring works — **including uploading and removing a blog photo while NOT an admin** (this is the Phase 4.2 change; before it, non-admin photo writes 403).
13. **Manage Data → Blog Posts** (debug editor): the row opens, and a photo can be uploaded against it (photo support registered).
