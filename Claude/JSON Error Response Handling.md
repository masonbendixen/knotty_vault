---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 1/27/2026
Version: 0.1
tags: 
---
# Overview

Inside the document:

Payment Design Document.md

We have a section for: Migrate existing endpoints to JSON error responses (new `ErrorResponse` helper)

I'd like to put together a plan here for the work that needs to be done. Please use the Payment Design Document and code base for reference and let's put together a plan to design the support needed on the server side and schema for these JSON errors, modifying existing endpoints to generate them, and the plan for modifying the Angular client code base to consume them.

Let's start by brainstorming what should be in the JSON errors. Is there industry standard or convention to use?

# Steps
- [x] Researched RFC 7807 (Problem Details for HTTP APIs) as industry standard
- [x] Created implementation plan with phased approach
- [x] **Phase 1: Server Foundation** ✅ 2026-01-27
  - [x] Create `util/error_codes.h` with error type constants and helpers ✅ 2026-01-27
  - [x] Create `util/error_response.h` and `util/error_response.cpp` with `ErrorResponse` class ✅ 2026-01-27
  - [x] Create `util/error_response_test.cpp` with unit tests ✅ 2026-01-27
  - [x] Update `util/CMakeLists.txt` to include new files ✅ 2026-01-27
- [x] **Phase 2: Client Foundation** ✅ 2026-01-27
  - [x] Create `shared/types/ApiError.ts` with RFC 7807 `ProblemDetails` interface ✅ 2026-01-27
  - [x] Create `shared/services/error/error.service.ts` for error parsing ✅ 2026-01-27
  - [x] Create `shared/services/toast/toast.service.ts` for toast notifications ✅ 2026-01-27
  - [x] Create `shared/interceptors/error.interceptor.ts` for global HTTP error handling ✅ 2026-01-27
  - [x] Add `MatSnackBarModule` to mat-imports ✅ 2026-01-27
  - [x] Update `app.config.ts` to register error interceptor ✅ 2026-01-27
  - [x] Add toast CSS styles to `styles.scss` ✅ 2026-01-27
- [x] **Phase 3: Migrate Authentication Endpoints** ✅ 2026-01-27
  - [x] Update `login.cpp` (pattern-setting example) ✅ 2026-01-27
  - [x] Update `login_test.cpp` to expect JSON error responses ✅ 2026-01-27
  - [x] Update `logout.cpp` (no changes needed - always returns 200) ✅ 2026-01-27
  - [x] Update `register.cpp` ✅ 2026-01-27
  - [x] Update `me.cpp` and `me_test.cpp` ✅ 2026-01-27
  - [x] Update `remember.cpp` and `remember_test.cpp` ✅ 2026-01-27
  - [x] Update `account_activation.cpp` and `account_activation_test.cpp` ✅ 2026-01-27
- [x] **Phase 4: Migrate Remaining Endpoints** ✅ 2026-01-27
  - [x] Update `get_user_info.cpp` and `get_user_info_test.cpp` ✅ 2026-01-27
  - [x] Update `set_user_info.cpp` and `set_user_info_test.cpp` ✅ 2026-01-27
  - [x] Update `update_user_password.cpp` and `update_user_password_test.cpp` ✅ 2026-01-27
  - [x] Update `add_item.cpp` and `add_item_test.cpp` ✅ 2026-01-27
  - [x] Update `update_item.cpp` and `update_item_test.cpp` ✅ 2026-01-27
  - [x] Update `delete_item.cpp` and `delete_item_test.cpp` ✅ 2026-01-27
  - [x] Update `get_row.cpp` and `get_row_test.cpp` ✅ 2026-01-27
  - [x] Update `get_table_rows.cpp` and `get_table_rows_test.cpp` ✅ 2026-01-27
  - [x] Update `get_rows_by_column.cpp` and `get_rows_by_column_test.cpp` ✅ 2026-01-27
- [x] **Phase 5: Migrate Angular Components** ✅ 2026-01-27
  - [x] Replace `alert()` calls with toast notifications ✅ 2026-01-27
  - [x] Update `login.component.ts` to show toast on error ✅ 2026-01-27
  - [x] Update `register.component.ts` to show toast on error ✅ 2026-01-27
  - [x] Update `edit-db-table.component.ts` to use toast ✅ 2026-01-27
  - [x] Update `table-entry-form-dialog.component.ts` to use toast ✅ 2026-01-27

> **CRITICAL**: When migrating any endpoint, the corresponding `*_test.cpp` file MUST also be updated. Tests verify error response format (status code, response body structure). Failing to update tests will cause test failures.

# RFC 7807 Error Response Format

All error responses will follow RFC 7807 with these fields:
- `type`: Error code identifier (e.g., `"validation_error"`)
- `title`: Short human-readable summary
- `status`: HTTP status code
- `detail`: Human-readable explanation for this specific occurrence
- Extension fields: `field`, `constraint`, `provider`, `provider_code`

Example:
```json
{
  "type": "invalid_credentials",
  "title": "Invalid Credentials",
  "status": 401,
  "detail": "The email or password you entered is incorrect"
}
```

