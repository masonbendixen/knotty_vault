---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/4/2026
Version: 0.1
tags: 
---
# Overview

Please go into plan mode and use this document as your planning document. Do not use .claude/plans at all. Please leave the overview in tact and do your work in the sections below. Please use the code base and this document for context.

The server code base is a layered architecture with each lower layer only being dependent on itself and the layers below. Here are the layers:

- Database definition layer
- Util layer (database CRUD wrappers and helpers for sending email, configuration, wrapping square, etc)
- The business logic layer
- Endpoints for the HTTP server

The code base has many of these at the top level of the directory structure. I would like these top level directories:

- Database definition layer (db_schema/)
- Util layer (database CRUD wrappers and helpers for sending email, configuration, wrapping square, etc) (database stuff is under sql_util/ and other utilities and helper code is under util/)
- The business logic layer (under a new directory business_logic/)
- Endpoints for the HTTP server (endpoints/)

Of these, db_schema/, sql_util/, util/, and endpoints/ exist and are pretty fine as is. Here are the big changes:

- business_logic/ is new. Please create it and move auth, images, payment, and scheduling underneath it.
- secrets/ and square/ should move underneath util/
- Please fix up all the paths to take the new locations into account.

Please create a plan that we can implement.

# Server Codebase Refactor Plan

## Current State

```
src/
├── auth/                  (22 files) → business_logic/auth/
├── database_helper/       (stays)
├── db_schema/             (stays)
├── endpoints/             (stays)
├── images/                (8 files)  → business_logic/images/
├── payment/               (22 files) → business_logic/payment/
├── scheduling/            (12 files) → business_logic/scheduling/
├── secrets/               (9 files)  → util/secrets/
├── sql_util/              (stays)
├── square/                (8 files)  → util/square/
├── util/                  (stays, gains secrets/ and square/)
└── main.cpp
```

## Target State

```
src/
├── business_logic/
│   ├── auth/
│   ├── images/
│   ├── payment/
│   └── scheduling/
├── database_helper/
├── db_schema/
├── endpoints/
├── sql_util/
├── util/
│   ├── http/
│   ├── mail/
│   ├── secrets/
│   └── square/
└── main.cpp
```

## Impact Analysis

### Include Path Updates Required

All includes are relative to `src/` (set by `target_include_directories(knotty_yoga_core PUBLIC ${CMAKE_CURRENT_LIST_DIR})` in `src/CMakeLists.txt`). Every `#include` referencing a moved directory must be updated.

| Old Prefix | New Prefix | References | Unique Files |
|---|---|---|---|
| `"auth/` | `"business_logic/auth/` | 138 | 53 |
| `"images/` | `"business_logic/images/` | 13 | 11 |
| `"payment/` | `"business_logic/payment/` | 34 | 23 |
| `"scheduling/` | `"business_logic/scheduling/` | 20 | 14 |
| `"secrets/` | `"util/secrets/` | 90 | 54 |
| `"square/` | `"util/square/` | 9 | 8 |
| **Total** | | **~304** | |

### Cross-Dependencies Between Moved Directories

These are includes *within* the moved directories that reference *other* moved directories. These all need double-updating (old prefix replaced with new prefix).

| From | Includes | Files |
|---|---|---|
| auth/ | secrets/ | person.cpp, person_test.cpp, server_config.h, session.h, person_verify_mail_test_util |
| images/ | secrets/ | image_helper.cpp, image_helper.h, image_helper_test.cpp |
| payment/ | secrets/ | payment_helper.cpp, payment_helper.h, payment_helper_test.cpp |
| payment/ | square/ | payment_helper.cpp, payment_helper.h, payment_helper_test.cpp |
| payment/ | scheduling/ | payment_helper.cpp (booking_confirmation_mail) |
| scheduling/ | payment/ | booking_helper.h, event_session_helper.h (catalog_helper, purchase_helper) |

Key observation: **payment/ ↔ scheduling/** have bidirectional dependencies. Both move under `business_logic/`, so these become `business_logic/payment/` ↔ `business_logic/scheduling/` — still valid since they're in the same layer.

### CMake Changes

1. **src/CMakeLists.txt** — Replace 4 individual `add_subdirectory()` calls with one `add_subdirectory(business_logic)`. Remove `add_subdirectory(secrets)` and `add_subdirectory(square)`.
2. **New: src/business_logic/CMakeLists.txt** — Passthrough to auth/, images/, payment/, scheduling/.
3. **src/util/CMakeLists.txt** — Add `add_subdirectory(secrets)` and `add_subdirectory(square)`.
4. Individual CMakeLists.txt files inside moved directories do NOT need changes (they reference local filenames, not paths).

### Documentation Updates

- **Root CLAUDE.md** — Update layering diagram paths and all references to `auth/`, `payment/`, `secrets/`, `square/`
- **payment/CLAUDE.md** — Update any path references (stays with payment/ after move)
- **secrets/CLAUDE.md** — Update any path references (stays with secrets/ after move)
- **endpoints/CLAUDE.md** — Update include path examples

---

## Phase 1: Create Directory Structure and Move Files

Use `git mv` to move directories. This preserves git history.

- [x] Create `src/business_logic/` directory
- [x] `git mv src/auth src/business_logic/auth`
- [x] `git mv src/images src/business_logic/images`
- [x] `git mv src/payment src/business_logic/payment`
- [x] `git mv src/scheduling src/business_logic/scheduling`
- [x] `git mv src/secrets src/util/secrets`
- [x] `git mv src/square src/util/square`

## Phase 2: Update CMakeLists.txt Files

- [x] Update `src/CMakeLists.txt`:
  - Remove: `add_subdirectory(auth)`, `add_subdirectory(images)`, `add_subdirectory(payment)`, `add_subdirectory(scheduling)`, `add_subdirectory(secrets)`, `add_subdirectory(square)`
  - Add: `add_subdirectory(business_logic)`
- [x] Create `src/business_logic/CMakeLists.txt`:
  ```cmake
  add_subdirectory(auth)
  add_subdirectory(images)
  add_subdirectory(payment)
  add_subdirectory(scheduling)
  ```
- [x] Update `src/util/CMakeLists.txt`:
  - Add: `add_subdirectory(secrets)` and `add_subdirectory(square)`

## Phase 3: Update Include Paths — Business Logic Directories

Update all `#include` statements for the 4 business logic directories. Process one directory at a time to keep changes reviewable.

### 3a: auth/ → business_logic/auth/ (138 references, 53 files)

- [x] Update includes in `src/business_logic/auth/` internal files (self-references like `#include "auth/person.h"` → `#include "business_logic/auth/person.h"`)
- [x] Update includes in `src/endpoints/` (~45 files reference auth/)
- [x] Update includes in `src/main.cpp`
- [x] Update includes in `src/business_logic/payment/` (payment_helper references auth? — verify)

### 3b: images/ → business_logic/images/ (13 references, 11 files)

- [x] Update includes in `src/endpoints/` (11 files reference images/)

### 3c: payment/ → business_logic/payment/ (34 references, 23 files)

- [x] Update includes in `src/endpoints/` (~10 files reference payment/)
- [x] Update includes in `src/business_logic/payment/` internal files
- [x] Update includes in `src/business_logic/scheduling/` (booking_helper.h, event_session_helper.h reference payment/)

### 3d: scheduling/ → business_logic/scheduling/ (20 references, 14 files)

- [x] Update includes in `src/endpoints/` (~7 files reference scheduling/)
- [x] Update includes in `src/business_logic/payment/` (payment_helper.cpp references scheduling/)
- [x] Update includes in `src/business_logic/scheduling/` internal files

## Phase 4: Update Include Paths — Util Directories

### 4a: secrets/ → util/secrets/ (90 references, 54 files)

- [x] Update includes in `src/business_logic/auth/` (5 files reference secrets/)
- [x] Update includes in `src/business_logic/images/` (3 files reference secrets/)
- [x] Update includes in `src/business_logic/payment/` (3 files reference secrets/)
- [x] Update includes in `src/endpoints/` (~30 files reference secrets/)
- [x] Update includes in `src/database_helper/` (create_database.cpp references secrets/)
- [x] Update includes in `src/sql_util/table_helpers/` (email_verifications references secrets/)
- [x] Update includes in `src/util/mail/` (mail_helper references secrets/)
- [x] Update includes in `src/main.cpp`

### 4b: square/ → util/square/ (9 references, 8 files)

- [x] Update includes in `src/business_logic/payment/` (3 files reference square/)
- [x] Update includes in `src/endpoints/` (4 files reference square/)
- [x] Update includes in `src/main.cpp`

## Phase 5: Update Documentation

- [x] Update root `CLAUDE.md`:
  - Layering diagram: `auth/` → `business_logic/auth/`, `payment/` → `business_logic/payment/`, add `images/` and `scheduling/`
  - Low-level services: `secrets/` → `util/secrets/`, `square/` → `util/square/`
  - Backend structure descriptions
  - Example code paths (e.g., `payment/payment_key_value_table.h/cpp` → `business_logic/payment/...`)
  - NormalizeCrLf reference to `auth/person_verify_mail.cpp` → `business_logic/auth/person_verify_mail.cpp`
- [x] Update `business_logic/payment/CLAUDE.md` path references (no external paths found)
- [x] Update `util/secrets/CLAUDE.md` path references (no external paths found)
- [x] Update `endpoints/CLAUDE.md` include path examples
- [x] Update memory file `MEMORY.md` if it references old paths

## Phase 6: Build and Test

- [ ] Build the server (Windows: Visual Studio / CMake)
- [ ] Build and run C++ tests (`knottyyoga_tests`)
- [ ] Build the database helper (`knottyyoga_database_helper`)
- [ ] Fix any remaining include errors from the build

## Execution Notes

**Order matters**: Phases 1-4 must be done together before building. A partial move will break the build. The safest approach is to do all moves and all include updates in one batch, then build.

**Mechanical find-and-replace**: The include path updates (Phases 3-4) are purely mechanical. For each moved directory, it's a global find-and-replace:
- `#include "auth/` → `#include "business_logic/auth/`
- `#include "images/` → `#include "business_logic/images/`
- `#include "payment/` → `#include "business_logic/payment/`
- `#include "scheduling/` → `#include "business_logic/scheduling/`
- `#include "secrets/` → `#include "util/secrets/`
- `#include "square/` → `#include "util/square/`

These are safe because the prefixes are unique — no file has an include like `#include "auth_something/..."` that could be falsely matched.

**Risk**: Low. This is a pure file reorganization with no logic changes. Every change is a path update. If the build succeeds and tests pass, the refactor is correct.