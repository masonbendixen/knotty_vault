---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/16/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I'd like to add a blog to the website. I'd like to add an author_blog permission that allows CRUD style blog post creation. Blog posts should be stored in a table called blog_posts. Besides an auto incrementing 64 bit integer id called id, they should have a name, author, body, bool draft, created_at_us, modifed_at_us, and a post_at_us. In addition, the table should be marked as having photo support.

On the client side, under the admin tab, their should be a new menu item called Blog Posts that is guarded by the author_blog permission. Going to this page should have a row of blog posts with pagination sorted by created_at_us. The user should be able to choose a year and month and have an option for All Years and All Months (choosing All Years automatically selects All Months) that let you filter posts. There should be edit and delete icons for each post.

There should be a New Post button that brings up a blog authoring screen. There should be fields for Name, Author, a bool checkbox for Draft, and a date / time picker to choose when to post and then a button for Post Now that sets the post to post immediately at the current time. Then there should be a Save Post and Cancel buttons. Below these, there should be a left and right pane for authoring. To the left we should host ngx-markdown with syntax highlighting for authoring markdown and then on the right, we should show the fully rendered markdown. Clicking Save Post should save this entry to the database. The Edit button in the grid control should bring up the same page but with the existing data from the existing post populating the controls.

There should be a new top level menu entry that required no permissions called Blog. Clicking this should bring the latest Blog posts up that shows the five most recent blog posts with next and previous posts buttons at the bottom. At the top of the page, there should be buttons for navigating year / month / day with all three only having entries for items for which there are blog posts as well as an option for All Years / All Months / All Days. Selecting All Years automatically chooses All Months / All Days. Choosing All Months automatically chooses All Days. Blog posts should be rendered with the Name as a title, then the author, the date modified last, and then the body converted to HTML to be displayed. If the current user has the author_blog permission, we should also show a button Edit Post / Delete Post. Delete Post should prompt for confirmation.

# Blog Feature with Markdown Support — Implementation Plan

## Context

Adding a blog to the Knotty Yoga website. Blog posts are authored by users with the `author_blog` permission via a markdown editor with live preview. Published posts are viewable by anyone on a public `/blog` page with year/month/day filtering and pagination. The blog requires a new database table, custom API endpoints, a new permission, and both admin and public Angular pages.

---

## Open Questions and Decisions

### Resolved

1. **Generic CRUD vs custom endpoints?** — **Custom endpoints.** The public blog needs pagination with date filtering, draft exclusion, and a "get available dates" query that generic CRUD cannot handle. The admin list can use `getFilteredTableRows` for the table view, but create/update/delete should use custom endpoints.

2. **Admin route location?** — **New top-level route `/blog-admin`** with `[AuthGuard, AuthorBlogGuard]`. The `/manage` route requires `ManageProductsGuard` which is a different permission — blog authors shouldn't need `manage_products`.

3. **Author field type?** — **Free-text string.** The spec defines `author` as a string. Authors may want pen names. Default to logged-in user's name in the editor.

4. **Markdown editor approach?** — **Plain `<textarea>` on left, `<markdown>` preview on right using ngx-markdown.** No heavyweight CodeMirror/Monaco. PrismJS (optional dependency of ngx-markdown, installed separately) provides syntax highlighting in the rendered preview's code blocks.

5. **Draft handling?** — Filter `draft=true` posts out of all public queries via SQL. Drafts visible only in admin list.

6. **"Post Now" behavior?** — Sets `post_at_us = now` AND `draft = false`. Convenience shortcut for "publish immediately."

7. **Column name typo?** — Spec says "modifed_at_us" — use correct spelling `modified_at_us`.

### Resolved (User-Confirmed)

8. **Day-level filtering on public blog:** — **Client-side computation.** Derive available days from the posts loaded for the selected year/month. The `GetAvailableDates` endpoint returns year/month pairs only. Day options are extracted from `post_at_us` of currently loaded posts.

9. **ngx-markdown Angular 19 compatibility:** — **Confirmed compatible.** Use `ngx-markdown@^19.0.0` (latest v19.1.1) with `marked@^15.0.0`. The library's major version aligns with Angular's major version. Uses `provideMarkdown()` for standalone app setup. No fallback needed.

---

## Section 1: Backend — Database Schema & Registration

### Phase 1.1: Blog Posts Table Definition

**Create:**
- `server/knottyyoga_server/src/db_schema/blog_posts.h`
- `server/knottyyoga_server/src/db_schema/blog_posts.cpp`

**blog_posts.h** — Constants:
- `kBlogPostsTable = "blog_posts"`
- `kBlogPostsId = "id"` (BIGSERIAL PK)
- `kBlogPostsName = "name"` (STRING, not null)
- `kBlogPostsAuthor = "author"` (STRING, not null)
- `kBlogPostsBody = "body"` (STRING, not null — maps to PG TEXT)
- `kBlogPostsDraft = "draft"` (BOOL, default TRUE)
- `kBlogPostsCreatedAtUs = "created_at_us"` (BIGINT, default `kDatabaseInfoDefaultNow`)
- `kBlogPostsModifiedAtUs = "modified_at_us"` (BIGINT, default `kDatabaseInfoDefaultNow`)
- `kBlogPostsPostAtUs = "post_at_us"` (BIGINT, nullable — null = not scheduled)
- Declare `void MakeBlogPostsTable(DatabaseInfo databaseInfo);`

**blog_posts.cpp** — Use `AddColumnPrimaryKey`, `AddColumnSimple` (name, author, body), `AddColumnNotNullableWithDefault` (draft → `kDatabaseInfoDefaultFalse`... actually wait, default TRUE), `AddColumnNotNullableWithDefault` (created_at_us, modified_at_us → `kDatabaseInfoDefaultNow`), `AddColumnNullable` (post_at_us).

**Pattern reference:** `db_schema/home_page_photos.h/.cpp`

### Phase 1.2: Three-Place Registration + Seed Data

**Modify:** `server/knottyyoga_server/src/db_schema/make_database_info.cpp`
- Add `#include "blog_posts.h"`
- Add `MakeBlogPostsTable(databaseInfo);` in `MakeDatabaseInfo()` (after gift permissions, under `// Blog tables` comment)

**Modify:** `server/knottyyoga_server/src/database_helper/create_database.cpp`
1. Add `#include "db_schema/blog_posts.h"` to includes
2. `CreateTables()`: Add `CreateTable(DbSchema::kBlogPostsTable);` under `// Blog tables`
3. `PopulatePhotoSupportTables()`: Add `AddRow(DbSchema::kBlogPostsTable);`
4. `PopulatePermissions()`: Add `AddRow("author_blog", "Permission to author and manage blog posts.");` — becomes permission ID 6
5. `PopulateRolePermissions()`: Add `const int kAuthorBlogPermissionId = 6;` and `AddRow(kAdminRoleId, kAuthorBlogPermissionId);` so admins get blog access
6. `PopulateAdminTopLevelTables()`: Add `AddRow(DbSchema::kBlogPostsTable);`
7. `PopulateAdminTablePermissions()`: Add `AddRow(DbSchema::kBlogPostsTable, 6);` (author_blog permission ID)
8. `PopulateAdminColumnDataInfo()`: Add entries for all blog_posts columns:
   - `name` → label "Title", hint "Blog post title", type "text", required "true"
   - `author` → label "Author", hint "Post author name", type "text", required "true"
   - `body` → label "Body", hint "Blog post content (Markdown)", type "long-text", required "true"
   - `draft` → label "Draft", hint "Whether this is a draft", type "checkbox", required "false"
   - `created_at_us` → label "Created", hint "When created", type "date", required "false", readonly "true"
   - `modified_at_us` → label "Modified", hint "When last modified", type "date", required "false", readonly "true"
   - `post_at_us` → label "Post Date", hint "When published", type "date", required "false"

**Modify:** `server/knottyyoga_server/src/db_schema/CMakeLists.txt`
- Add `blog_posts.h` and `blog_posts.cpp` to `target_sources(knotty_yoga_core PRIVATE ...)`

**No blog seed data needed** — blog starts empty.

---

## Section 2: Backend — Table Helper

### Phase 2.1: Blog Posts Table Helper + Tests

**Create:**
- `server/knottyyoga_server/src/sql_util/table_helpers/blog_posts.h`
- `server/knottyyoga_server/src/sql_util/table_helpers/blog_posts.cpp`
- `server/knottyyoga_server/src/sql_util/table_helpers/blog_posts_test.cpp`

**Class:** `TableHelpers::BlogPosts`

**Methods:**
```
BlogPosts(DatabaseHelper databaseHelper)
int64_t AddBlogPost(transaction, name, author, body, draft, postAtUs)
KeyValueTable GetBlogPost(transaction, id)
void UpdateBlogPost(transaction, id, name, author, body, draft, modifiedAtUs, postAtUs)
void DeleteBlogPost(transaction, id)
KeyValueTableArray GetPublishedPosts(transaction, pageSize, page)
int64_t GetPublishedPostCount(transaction)
KeyValueTableArray GetPublishedPostsByDateRange(transaction, startUs, endUs, pageSize, page)
int64_t GetPublishedPostCountByDateRange(transaction, startUs, endUs)
KeyValueTableArray GetAvailableDates(transaction)
```

**Key SQL patterns:**
- Published = `draft = false AND post_at_us <= now_us()`
- Pagination = `ORDER BY post_at_us DESC LIMIT $1 OFFSET $2`
- Available dates = `SELECT DISTINCT EXTRACT(YEAR FROM to_timestamp(post_at_us / 1000000.0))::int, EXTRACT(MONTH FROM to_timestamp(post_at_us / 1000000.0))::int FROM blog_posts WHERE draft = false AND post_at_us <= now_us() ORDER BY year DESC, month DESC`

**Use:** `DbCrud::AddRowToTableFetchInt64PrimaryKey`, `DbCrud::GetRow`, `DbCrud::UpdateRow`, `DbCrud::DeleteRow` for standard CRUD. Custom SQL via `transaction.Execute()` for the published/date queries.

**Tests:** Use `TestDatabaseUtil` + `RunInTransaction`. No table creation needed. Test all methods.

**Modify:** `server/knottyyoga_server/src/sql_util/table_helpers/CMakeLists.txt`
- Add `blog_posts.h`, `blog_posts.cpp` to core, `blog_posts_test.cpp` to tests

---

## Section 3: Backend — Business Logic Layer

### Phase 3.1: Blog Helper + Key Value Table + Tests

**Create directory:** `server/knottyyoga_server/src/business_logic/blog/`

**Create files:**
- `business_logic/blog/CMakeLists.txt`
- `business_logic/blog/blog_helper.h` and `.cpp`
- `business_logic/blog/blog_helper_test.cpp`
- `business_logic/blog/blog_key_value_table.h` and `.cpp`
- `business_logic/blog/blog_key_value_table_test.cpp`

**Domain structs** (in `blog_helper.h`):
```cpp
namespace Blog {
    struct BlogPostInfo {
        int64_t id;
        std::string name;
        std::string author;
        std::string body;
        bool draft;
        int64_t createdAtUs;
        int64_t modifiedAtUs;
        int64_t postAtUs;  // 0 if not scheduled
    };

    struct BlogPostListResult {
        std::vector<BlogPostInfo> posts;
        int64_t totalCount;
    };

    struct AvailableDate {
        int year;
        int month;
    };
}
```

**BlogHelper class:** Wraps `TableHelpers::BlogPosts`, converts `KeyValueTable` → `BlogPostInfo` structs. Methods mirror table helper but return domain structs.

**blog_key_value_table.h:** Conversion functions:
- `BlogPostInfoToKeyValueTable(const BlogPostInfo&) → KeyValueTable`
- `BlogPostListResultToKeyValueTable(const BlogPostListResult&) → KeyValueTable` (includes `items` array + `total_count`)
- `AvailableDateToKeyValueTable(const AvailableDate&) → KeyValueTable`
- `AvailableDatesToKeyValueTable(const std::vector<AvailableDate>&) → KeyValueTable` (wraps in `dates` array)

**Modify:** `server/knottyyoga_server/src/business_logic/CMakeLists.txt`
- Add `add_subdirectory(blog)`

---

## Section 4: Backend — Endpoints

### Phase 4.1: Blog Endpoints + Tests

**Create:**
- `server/knottyyoga_server/src/endpoints/blog_posts.h`
- `server/knottyyoga_server/src/endpoints/blog_posts.cpp`
- `server/knottyyoga_server/src/endpoints/blog_posts_test.cpp`

**Routes to register:**

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | `/api/blog_posts` | None | Published posts with pagination. Query params: `page`, `page_size`, `start_us`, `end_us` |
| GET | `/api/blog_posts/dates` | None | Available year/month pairs for published posts |
| GET | `/api/blog_posts/<int>` | Optional | Single post. Public: non-draft only. With `author_blog`: all posts |
| POST | `/api/blog_posts` | `author_blog` | Create blog post |
| PUT | `/api/blog_posts/<int>` | `author_blog` | Update blog post (also updates `modified_at_us`) |
| DELETE | `/api/blog_posts/<int>` | `author_blog` | Delete blog post |

**Endpoint structure:** Follow `admin_duplicate_product.cpp` pattern:
- `HandleGet`/`HandlePost`/`HandlePut`/`HandleDelete` HTTP handlers
- `SetupRouting` class with `RoutingBase`
- Exported functions: `GetBlogPosts`, `GetBlogPostDates`, `GetBlogPost`, `PostBlogPost`, `PutBlogPost`, `DeleteBlogPost`
- Permission check: `session.ActiveUserHasPermission(transaction, "author_blog")`
- Response: `SqlUtil::KeyValueTableToJson(Blog::BlogPostInfoToKeyValueTable(result))`
- Query param parsing: `req.url_params.get("page")` etc.

**Modify:** `server/knottyyoga_server/src/endpoints/web_app.cpp`
- Add `#include "blog_posts.h"`
- Add `auto g_GetBlogPosts = &Endpoints::GetBlogPosts;` (and similar for each exported function)

**Modify:** `server/knottyyoga_server/src/endpoints/CMakeLists.txt`
- Add `blog_posts.h`, `blog_posts.cpp` to core target
- Add `blog_posts_test.cpp` to tests target

---

## Section 5: Frontend — Dependencies & Types

### Phase 5.1: Install ngx-markdown + Blog Types

**Run:** `cd ui && npm install ngx-markdown@^19.0.0 marked@^15.0.0 prismjs@^1.30.0 --save`
- ngx-markdown v19.x is confirmed compatible with Angular 19 (major versions align)
- `marked@^15.0.0` is the required peer dependency for ngx-markdown v19
- `prismjs` is optional but needed for syntax highlighting in code blocks

**Modify:** `ui/angular.json`
- Add to `styles` array: `"node_modules/prismjs/themes/prism-okaidia.css"`
- Add to `scripts` array:
  ```json
  "node_modules/prismjs/prism.js",
  "node_modules/prismjs/components/prism-typescript.min.js",
  "node_modules/prismjs/components/prism-javascript.min.js",
  "node_modules/prismjs/components/prism-css.min.js"
  ```
  (Add additional `prism-*.min.js` entries for other languages as needed)

**Modify:** `ui/src/app/app.config.ts`
- Import: `import { provideMarkdown } from 'ngx-markdown';`
- Add `provideMarkdown()` to `providers` array
  - Note: If remote `.md` file loading via `[src]` is needed later, pass `{ loader: HttpClient }` and ensure `provideHttpClient()` is also present

**Create:** `ui/src/app/shared/types/blog.types.ts`
```typescript
export interface BlogPost {
  id: number;
  name: string;
  author: string;
  body: string;
  draft: boolean;
  created_at_us: number;
  modified_at_us: number;
  post_at_us: number | null;
}
export interface BlogPostListResponse {
  items: BlogPost[];
  total_count: number;
}
export interface AvailableDate {
  year: number;
  month: number;
}
export interface BlogDatesResponse {
  dates: AvailableDate[];
}
export interface CreateBlogPostRequest {
  name: string;
  author: string;
  body: string;
  draft: boolean;
  post_at_us: number | null;
}
export interface UpdateBlogPostRequest {
  name: string;
  author: string;
  body: string;
  draft: boolean;
  post_at_us: number | null;
}
```

### Phase 5.2: Server Access Interface + All Implementations

**Modify:** `ui/src/app/shared/types/ServerAccess.ts`
- Import blog types
- Add 6 blog methods to `ServerAccess` interface:
  - `getBlogPosts(page, pageSize, startUs?, endUs?): Observable<BlogPostListResponse>`
  - `getBlogPost(id): Observable<BlogPost>`
  - `getBlogDates(): Observable<BlogDatesResponse>`
  - `createBlogPost(request): Observable<BlogPost>`
  - `updateBlogPost(id, request): Observable<BlogPost>`
  - `deleteBlogPost(id): Observable<void>`
- Add re-exports for blog types

**Modify:** `ui/src/app/shared/services/network/ServerAccessNetwork.ts`
- Implement all 6 methods with HTTP calls to `/api/blog_posts*`

**Modify:** `ui/src/app/shared/services/network/ServerAccess.mock.ts`
- Add `blogPosts: BlogPost[]` array with 3-5 sample posts
- Implement all 6 methods with in-memory operations

**Modify:** `ui/src/app/shared/services/network/ServerAccess.ts` (Proxy)
- Add all 6 methods using `this.serialize(() => this.impl.method(...))`

### Phase 5.3: Auth Types & Guard

**Modify:** `ui/src/app/core/services/auth.types.ts`
- Add:
  ```typescript
  export function hasAuthorBlog(authData: AuthData): boolean {
    return (authData.isAuth && authData.isAdmin) || hasPermission(authData, 'author_blog');
  }
  ```

**Modify:** `ui/src/app/core/guards/auth-guards.ts`
- Import `hasAuthorBlog`
- Add `AuthorBlogGuard` following `ManageProductsGuard` pattern

---

## Section 6: Frontend — Public Blog View

### Phase 6.1: Public Blog List Page

**Create:**
- `ui/src/app/pages/public/blog/blog-list/blog-list.component.ts`
- `ui/src/app/pages/public/blog/blog-list/blog-list.component.html`
- `ui/src/app/pages/public/blog/blog-list/blog-list.component.scss`

**Features:**
- Standalone component importing `SharedModule`, `MarkdownModule` (from `ngx-markdown`)
- Injects `SERVER_ACCESS_TOKEN` for API calls, `AuthService` for permission checks
- On init: calls `getBlogDates()` and `getBlogPosts(0, 5)`
- **Date navigation** at top:
  - Year dropdown: from `dates` response + "All Years"
  - Month dropdown: filtered by selected year + "All Months"
  - Day dropdown: derived from loaded posts + "All Days"
  - Selecting "All Years" auto-selects "All Months" / "All Days"
  - Date selection computes `startUs`/`endUs` and reloads posts
- **Post rendering:** For each post:
  - `<h2>{{ post.name }}</h2>` (title)
  - `<p>By {{ post.author }} | {{ formatDate(post.modified_at_us) }}</p>`
  - `<markdown [data]="post.body"></markdown>` (rendered body)
  - If `hasAuthorBlog(authData)`: Edit Post button (links to `/blog-admin/edit/:id`) and Delete Post button (confirmation dialog)
- **Pagination:** "Previous" / "Next" buttons, page tracking, disable at bounds
- Styling: SCSS with `mat-card` borders, consistent with project conventions

### Phase 6.2: Blog Routes & Menu

**Modify:** `ui/src/app/pages/public/public.routes.ts`
- Import `BlogListComponent`
- Add: `{ path: 'blog', component: BlogListComponent }`

**Modify:** `ui/src/app/shared/services/header/mockHeaderResponse.ts`
- Add `blogButton` definition:
  ```typescript
  const blogButton: HeaderButton = {
    title: 'Blog',
    kind: HeaderButtonKind.InternalLink,
    goTo: '/blog',
  };
  ```
- Insert into `headerData.menu` array (between `eventsButton` and `shopButton`, around line 179)

---

## Section 7: Frontend — Admin Blog Management

### Phase 7.1: Admin Blog List Page

**Create:**
- `ui/src/app/pages/blog-admin/blog-list/blog-admin-list.component.ts`
- `ui/src/app/pages/blog-admin/blog-list/blog-admin-list.component.html`
- `ui/src/app/pages/blog-admin/blog-list/blog-admin-list.component.scss`

**Features:**
- Standalone component with `SharedModule` imports
- Uses `getFilteredTableRows('blog_posts', 'created_at_us', false, pageSize, page, filterPairs)` for paginated list
- Year/Month filter dropdowns with "All Years" / "All Months"
- Material table (`mat-table`) with columns: Name, Author, Draft (chip/badge), Created Date
- Edit icon per row → navigates to `/blog-admin/edit/:id`
- Delete icon per row → confirmation dialog → `deleteBlogPost(id)` → reload
- "New Post" button → navigates to `/blog-admin/new`
- Pagination controls (page size, page number, total count)

### Phase 7.2: Blog Editor Page

**Create:**
- `ui/src/app/pages/blog-admin/blog-editor/blog-editor.component.ts`
- `ui/src/app/pages/blog-admin/blog-editor/blog-editor.component.html`
- `ui/src/app/pages/blog-admin/blog-editor/blog-editor.component.scss`

**Features:**
- Standalone component with `SharedModule`, `MarkdownModule` (from `ngx-markdown`), `ReactiveFormsModule`
- **Reactive form fields:**
  - `name` (mat-form-field, text input, required)
  - `author` (mat-form-field, text input, required, defaults to user's `firstName + " " + lastName`)
  - `draft` (mat-checkbox, defaults to true)
  - `postAtUs` (Material datepicker + timepicker combo, converts to/from microseconds)
  - `body` (textarea, required)
- **Buttons:**
  - "Post Now" → sets `draft=false`, `post_at_us=Date.now()*1000`, then saves
  - "Save Post" → saves with current form values
  - "Cancel" → navigates to `/blog-admin`
- **Split-pane layout:**
  - Left: `<textarea formControlName="body" class="markdown-editor">` (full height)
  - Right: `<markdown [data]="form.get('body').value">` (live preview, scrollable)
  - CSS: `display: flex; gap: 1rem;` with each pane at `flex: 1`
- **Edit mode:** When route has `:id` param, load via `getBlogPost(id)`, populate form
- **Create mode:** When route is `new`, form starts empty except author default
- **On save:** `createBlogPost()` or `updateBlogPost()` → navigate to `/blog-admin`

### Phase 7.3: Admin Blog Routes & Menu

**Create:** `ui/src/app/pages/blog-admin/blog-admin.routes.ts`
```typescript
const routes: Routes = [
  { path: '', component: BlogAdminListComponent },
  { path: 'new', component: BlogEditorComponent },
  { path: 'edit/:id', component: BlogEditorComponent },
];
```

**Modify:** `ui/src/app/app.routes.ts`
- Import `AuthorBlogGuard`
- Add route block after `manage`:
  ```typescript
  {
    path: 'blog-admin',
    loadChildren: () => import('@pages/blog-admin/blog-admin.routes'),
    canActivate: [AuthGuard, AuthorBlogGuard],
  },
  ```

**Modify:** `ui/src/app/shared/services/header/mockHeaderResponse.ts`
- Import `hasAuthorBlog` from `auth.types`
- Update admin dropdown visibility condition (line ~200):
  ```typescript
  if (authData.isAuth && (authData.isAdmin || hasManageProducts(authData) || hasAuthorBlog(authData))) {
  ```
- Add "Blog Posts" to `adminMenu` conditionally:
  ```typescript
  if (hasAuthorBlog(authData)) {
    adminMenu.push({
      title: 'Blog Posts',
      kind: HeaderButtonKind.InternalLink,
      goTo: '/blog-admin',
    });
  }
  ```

---

## Implementation Sequence

Execute phases in this order (dependency-aware):

| Step | Phase | Description | Depends On |
|------|-------|-------------|------------|
| 1 | 1.1 | Blog posts table schema | — |
| 2 | 1.2 | Three-place registration + seed data | 1.1 |
| 3 | 2.1 | Table helper + tests | 1.1, 1.2 |
| 4 | 3.1 | Business logic helper + KVT + tests | 2.1 |
| 5 | 4.1 | Blog endpoints + tests | 3.1 |
| 6 | 5.1 | Install ngx-markdown + blog types | — |
| 7 | 5.2 | Server access interface + implementations | 5.1 |
| 8 | 5.3 | Auth types & guard | — |
| 9 | 6.1 | Public blog list page | 5.1, 5.2 |
| 10 | 6.2 | Public blog routes & menu | 6.1 |
| 11 | 7.1 | Admin blog list page | 5.2 |
| 12 | 7.2 | Blog editor page | 5.1, 5.2 |
| 13 | 7.3 | Admin blog routes & menu | 5.3, 7.1, 7.2 |

Backend (steps 1-5) can be completed fully before frontend. Frontend steps 6-8 can be started as soon as types are defined.

---

## Critical File Reference

### Files to CREATE (Backend)
- `server/knottyyoga_server/src/db_schema/blog_posts.h` + `.cpp`
- `server/knottyyoga_server/src/sql_util/table_helpers/blog_posts.h` + `.cpp` + `_test.cpp`
- `server/knottyyoga_server/src/business_logic/blog/CMakeLists.txt`
- `server/knottyyoga_server/src/business_logic/blog/blog_helper.h` + `.cpp` + `_test.cpp`
- `server/knottyyoga_server/src/business_logic/blog/blog_key_value_table.h` + `.cpp` + `_test.cpp`
- `server/knottyyoga_server/src/endpoints/blog_posts.h` + `.cpp` + `_test.cpp`

### Files to CREATE (Frontend)
- `ui/src/app/shared/types/blog.types.ts`
- `ui/src/app/pages/public/blog/blog-list/blog-list.component.ts` + `.html` + `.scss`
- `ui/src/app/pages/blog-admin/blog-admin.routes.ts`
- `ui/src/app/pages/blog-admin/blog-list/blog-admin-list.component.ts` + `.html` + `.scss`
- `ui/src/app/pages/blog-admin/blog-editor/blog-editor.component.ts` + `.html` + `.scss`

### Files to MODIFY (Backend)
- `server/.../db_schema/make_database_info.cpp` — add `MakeBlogPostsTable`
- `server/.../database_helper/create_database.cpp` — 8 functions to modify (CreateTables, PopulatePhotoSupport, PopulatePermissions, PopulateRolePermissions, PopulateAdminTopLevelTables, PopulateAdminTablePermissions, PopulateAdminColumnDataInfo)
- `server/.../db_schema/CMakeLists.txt` — add blog_posts files
- `server/.../sql_util/table_helpers/CMakeLists.txt` — add blog_posts files
- `server/.../business_logic/CMakeLists.txt` — add blog subdirectory
- `server/.../endpoints/CMakeLists.txt` — add blog_posts files
- `server/.../endpoints/web_app.cpp` — register blog endpoint references

### Files to MODIFY (Frontend)
- `ui/src/app/shared/types/ServerAccess.ts` — add 6 blog methods + re-exports
- `ui/src/app/shared/services/network/ServerAccessNetwork.ts` — implement blog HTTP calls
- `ui/src/app/shared/services/network/ServerAccess.mock.ts` — implement blog mock
- `ui/src/app/shared/services/network/ServerAccess.ts` — proxy blog methods
- `ui/src/app/core/services/auth.types.ts` — add `hasAuthorBlog()`
- `ui/src/app/core/guards/auth-guards.ts` — add `AuthorBlogGuard`
- `ui/src/app/shared/services/header/mockHeaderResponse.ts` — add Blog menu + Blog Posts admin
- `ui/src/app/app.routes.ts` — add blog-admin route
- `ui/src/app/pages/public/public.routes.ts` — add /blog route
- `ui/angular.json` — add PrismJS styles
- `ui/src/app/app.config.ts` — add markdown provider

### Key Existing Functions to Reuse
- `DbCrud::AddRowToTableFetchInt64PrimaryKey` — `sql_util/database_access/database_crud_helpers.h`
- `SqlUtil::KeyValueTableToJson` — `sql_util/json/key_value_table_json.h`
- `ErrorResponse::NotAuthenticated/NotAuthorized/BadRequest` — `util/error_response.h`
- `session.ActiveUserHasPermission(transaction, "author_blog")` — `business_logic/auth/session.h`
- `microsToDate` / `dateToMicros` — `ui/src/app/shared/utils/DateFormatting.ts`
- `hasPermission(authData, permission)` — `ui/src/app/core/services/auth.types.ts`
- `ConfirmDialogComponent` — `ui/src/app/shared/components/confirm-dialog/`

---

## Verification Plan

### Backend
1. **Build:** `cd server/knottyyoga_server/build && cmake .. && make` — verify clean compilation
2. **Unit tests:** `bin/knottyyoga_tests --gtest_filter=BlogPosts*` — all table helper tests pass
3. **Unit tests:** `bin/knottyyoga_tests --gtest_filter=BlogHelper*` — all business logic tests pass
4. **Unit tests:** `bin/knottyyoga_tests --gtest_filter=BlogKeyValueTable*` — all KVT tests pass
5. **Unit tests:** `bin/knottyyoga_tests --gtest_filter=BlogEndpoint*` — all endpoint tests pass
6. **Database reset:** Run `knottyyoga_database_helper` to verify table creation with new `blog_posts` table, `author_blog` permission, and photo support registration

### Frontend
7. **Build:** `cd ui && ng build` — verify clean compilation
8. **Unit tests:** `ng test` — all tests pass
9. **Dev server:** `ng serve` — verify:
   - `/blog` route loads public blog page (empty initially)
   - "Blog" menu item appears in header
   - Admin dropdown shows "Blog Posts" when logged in as admin
   - `/blog-admin` route loads admin list
   - `/blog-admin/new` loads editor with split-pane markdown
   - Creating a post and viewing it on `/blog` works end-to-end

### Integration (requires backend running)
10. Create a blog post via admin, verify it appears on public blog
11. Test draft filtering — draft posts hidden from public view
12. Test date filtering — year/month navigation works
13. Test pagination — next/prev buttons work with >5 posts
14. Test photo upload via admin data viewer (photo support registered)
15. Test permission enforcement — non-admin without `author_blog` cannot access `/blog-admin`