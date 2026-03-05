---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 2/20/2026
Version: 0.1
tags:
---
# Overview

I need to add support for images. Please use the code base and this document for context. This is your planning document that you use for plan mode. Please make design and implementation plans here in the sections below the overview. Please do not ask me for permission to change this document.

At a high level, I want to add support for images to the database and website. I would like to support associating images with database rows.

There will be a table that lists the tables for which photo support is supported (photo_support_tables). Tables that support photos must have a 64bit integer id (their primary key).

There is a table that is already defined call photo_instances. This is the only table that stores the actual bytes of a photo (the photo field). It contains the dimensions of a photo and a type (like "jpg" or "png"). It currently has a created that is a date. I would like this changed to created_at_us and changed to a 64 bit integer and to add a last_updated_at_us that is also a 64 bit integer.

There is a table already defined called source source_photos. This table maps to an original image that has been uploaded by the user. It contains a foreign key reference to the photo_instances table to note the storage of this image. It currently contains a created that is a date. I would like this changed to created_at_us and changed to a 64 bit integer and to add a last_updated_at_us that is also a 64 bit integer. Deleting a source photo should delete the photo_instances item associated with this source photo.

There is a table already defined called source scaled_photos. This table maps to a scaled version of an image uploaded and tracked via source_photos. Basically, as we need different sizes of this image for things like thumbnails or image galleries, I'd like to keep a cache of these scaled photos. At some point, I will probably keep a hot cache of these in memory for things that have been hinted as high availability (like images on the home page). I also need to be able to have a background task as these are created to update the last used field as they are accessed and then have a periodic reaper task that runs at configurable intervals and removes older scaled photos that haven't been used within a configurable time window. The table contains a foreign key reference to the photo_instances table to note the storage of this image and a foreign key reference to the source_photos for the original for which this is an instance of. I would like a created_at_us that is a 64 bit integer and a last_used_at_us that is also a 64 bit integer. Deleting or updating a source photo should delete all the scaled_photos associated with the given image and all the photo_instances associated.

We need a table (table_item_photos) that links rows in tables to photos. It needs a BIG SERIAL primary key (id). It needs a table_name (varchar) that references a table and a table_item_id (64 bit int) that is the primary key of a given row in a table. We should have a uniqueness constraint on the table for table_name / table_item_id pairs. We also need a source_photo_id that is a foreign key reference to source_photos. This allows us to associate a photo with a row in a table marked by photo_support_tables.

We currently have support for scaling images at util/image_resize.h/cpp/test.cpp. Please take a look at this code and let me know what you think and if you have ideas for a better solution or if you think that this looks okay.

Moving up the stack from the table definitions. We need table helpers for all these tables. Please look at the current table helpers and tables and come up with a design for table helpers based on all these new tables.

Moving up to business logic, please create plans for an images business logic directory that is a peer to auth and payment. Please use a namespace Images for everything in this directory. Look at the business logic in auth and payment and create a plan for similar business logic helpers for images. Here are some workflows I can think of. Can you think of others:
- Configure a maximum width and height that we allow for source photos (and settings / secrets to configure this) and cropping attempts to upload photos above this size
- Associate / update an image with a given table / table row item
- Delete an image associated with a given table / table row item
- Lookup the source photo associated with a given table / table row item
- Fetch or create and fetch a scaled version of a given source photo and update the last used timestamp
- Delete all scaled photos older than a given time window
- List the storage space consumed by all scaled photos
- Delete oldest scaled photos until a certain size is achieved

We need to support the following workflows
- Allowing a non admin user to update the photo associated with their people entry
- Showing this photo as a thumbnail next to the user name in the dashboard instead of the generic icon if one has been uploaded
- In the admin console for editing tables, we need column metadata for if a given item has a photo and showing a thumbnail of this in the table view, a larger one in the edit view, and allowing the user to upload and update this photo
- Add an endpoint so that non admin users can request a scaled photo for a given table / table item id that is below a configurable (secret) width and height threshold.

Please fill in this document with a plan that shows (make sure you include testing needed for each step):
- Definitions of tables that are used / needed / updated
- Work to update the tables
- Work needed to update database helper
- Configuration secrets that need to be added
- The table helpers needed to support this
- The business logic
- Endpoints
- Client side changes to support admin console updates for images
- Client side changes to support photo support for user accounts
- Adding a table for home page photos with photo support (on the server)
- Adding viewing of a carousel of these home pages photos that picks a certain number of photos randomly but more skewed towards more recent photos

# Image Resize Review

The existing code at `util/image_resize.h/cpp` uses Boost.GIL with bilinear sampling and supports BMP/JPEG/PNG/TIFF. It is solid for the basic resize use case.

**Aspect-ratio-preserving resize** is already handled by `util/bounding_rect.h/cpp` which computes target dimensions that fit within a bounding box while preserving proportions. The workflow is:
1. `BoundingRect::GetClippedRect({srcWidth, srcHeight}, {maxWidth, maxHeight})` to compute target dimensions
2. `ImageResize::ResizeImage(bytes, targetWidth, targetHeight, type)` to perform the resize

No new functions are needed in `image_resize.h`. The existing `ResizeImage` + `BoundingRect::GetClippedRect` already cover all use cases (thumbnails, max-size enforcement, scaled photos).

**Note**: We restrict uploads to JPEG and PNG only (no BMP/TIFF for web use), though the resize library supports all four formats internally.

---

# Phase A: Database Schema Updates

## A.1 - Update `photo_instances` table definition
**File**: `db_schema/photos.h`, `db_schema/photos.cpp`
- Remove `created` (DATE) column constant and schema registration
- Add `kPhotoInstancesCreatedAtUs` = `"created_at_us"` (BIGINT, default `now_us()`)
- Add `kPhotoInstancesLastUpdatedAtUs` = `"last_updated_at_us"` (BIGINT, default `now_us()`)

## A.2 - Update `source_photos` table definition
**File**: `db_schema/photos.h`, `db_schema/photos.cpp`
- Remove `created` (DATE) column constant and schema registration
- Add `kSourcePhotosCreatedAtUs` = `"created_at_us"` (BIGINT, default `now_us()`)
- Add `kSourcePhotosLastUpdatedAtUs` = `"last_updated_at_us"` (BIGINT, default `now_us()`)

## A.3 - Update `scaled_photos` table definition
**File**: `db_schema/photos.h`, `db_schema/photos.cpp`
- Add `kScaledPhotosCreatedAtUs` = `"created_at_us"` (BIGINT, default `now_us()`)
- Add `kScaledPhotosLastUsedAtUs` = `"last_used_at_us"` (BIGINT, default `now_us()`)

## A.4 - Create `photo_support_tables` table definition
**File**: `db_schema/photos.h`, `db_schema/photos.cpp`
- `kPhotoSupportTables` = `"photo_support_tables"`
- `kPhotoSupportTablesTableName` = `"table_name"` (STRING, PRIMARY KEY)
- Function: `MakePhotoSupportTablesTable(DatabaseInfo)`

## A.5 - Create `table_item_photos` table definition
**File**: `db_schema/photos.h`, `db_schema/photos.cpp`
- `kTableItemPhotos` = `"table_item_photos"`
- `kTableItemPhotosId` = `"id"` (BIGSERIAL, PRIMARY KEY)
- `kTableItemPhotosTableName` = `"table_name"` (STRING, FK to `photo_support_tables.table_name`)
- `kTableItemPhotosTableItemId` = `"table_item_id"` (BIGINT)
- `kTableItemPhotosSourcePhotoId` = `"source_photo_id"` (BIGINT, FK to `source_photos.source_photo_id`)
- UNIQUE constraint on (`table_name`, `table_item_id`)
- Function: `MakeTableItemPhotosTable(DatabaseInfo)`

## A.6 - Create `home_page_photos` table definition
**File**: `db_schema/home_page_photos.h`, `db_schema/home_page_photos.cpp`
- `kHomePagePhotos` = `"home_page_photos"`
- `kHomePagePhotosId` = `"id"` (BIGSERIAL, PRIMARY KEY)
- `kHomePagePhotosTitle` = `"title"` (STRING, NULLABLE)
- `kHomePagePhotosDescription` = `"description"` (STRING, NULLABLE)
- `kHomePagePhotosDisplayOrder` = `"display_order"` (INT, default 0)
- `kHomePagePhotosCreatedAtUs` = `"created_at_us"` (BIGINT, default `now_us()`)
- Function: `MakeHomePagePhotosTable(DatabaseInfo)`

## A.7 - Update `db_schema/CMakeLists.txt`
Add `home_page_photos.h` and `home_page_photos.cpp`

## A.8 - Register new tables in `db_schema/make_database_info.cpp`
- Add `MakePhotoSupportTablesTable(databaseInfo)` before photo_instances
- Add `MakeTableItemPhotosTable(databaseInfo)` after scaled_photos
- Add `MakeHomePagePhotosTable(databaseInfo)` after table_item_photos

## A.9 - Update `database_helper/create_database.cpp`
- Add CreateTable calls for `photo_support_tables`, `table_item_photos`, `home_page_photos` in correct dependency order
- In PopulateTables, add `photo_support_tables` entries for `people` and `home_page_photos`
- Add `home_page_photos` to `PopulateAdminTopLevelTables`
- Add `admin_column_data_info` entries for home_page_photos columns

**Tests**: No separate tests for schema definitions. Schema correctness is validated by table helper tests and endpoint tests that create and use these tables.

---

# Phase B: Configuration Secrets

## B.1 - Add image-related secret keys
**File**: `secrets/secret_keys.h`
```
kImageMaxSourceWidth = "image_max_source_width"
kImageMaxSourceHeight = "image_max_source_height"
kImageMaxUploadBytes = "image_max_upload_bytes"
kImageMaxPublicScaledWidth = "image_max_public_scaled_width"
kImageMaxPublicScaledHeight = "image_max_public_scaled_height"
kScaledPhotoReaperIntervalUs = "scaled_photo_reaper_interval_us"
kScaledPhotoMaxAgeUs = "scaled_photo_max_age_us"
```

## B.2 - Add default values
**File**: `secrets/secret_values.h`, `secrets/secret_values.cpp`
- `image_max_source_width` = "4096"
- `image_max_source_height` = "4096"
- `image_max_upload_bytes` = "10485760" (10MB)
- `image_max_public_scaled_width` = "800"
- `image_max_public_scaled_height` = "800"
- `scaled_photo_reaper_interval_us` = "86400000000" (24 hours in microseconds)
- `scaled_photo_max_age_us` = "2592000000000" (30 days in microseconds)

**Tests**: Secret values are tested implicitly through business logic tests that read them.

---

# Phase C: Table Helpers

## C.1 - PhotoInstances table helper
**File**: `sql_util/table_helpers/photo_instances.h`, `sql_util/table_helpers/photo_instances.cpp`
- `AddPhotoInstance(transaction, photo_bytes, type, width, height) -> int64_t` (returns id)
- `GetPhotoInstance(transaction, id) -> KeyValueTable`
- `UpdatePhotoInstance(transaction, id, photo_bytes, type, width, height)`
- `DeletePhotoInstance(transaction, id)`

## C.2 - SourcePhotos table helper
**File**: `sql_util/table_helpers/source_photos.h`, `sql_util/table_helpers/source_photos.cpp`
- `AddSourcePhoto(transaction, photo_instance_id) -> int64_t`
- `GetSourcePhoto(transaction, id) -> KeyValueTable`
- `GetSourcePhotoWithInstance(transaction, id) -> KeyValueTable` (JOIN query returning photo bytes too)
- `DeleteSourcePhoto(transaction, id)` (also deletes associated photo_instance)
- `UpdateSourcePhotoTimestamp(transaction, id)`

## C.3 - ScaledPhotos table helper
**File**: `sql_util/table_helpers/scaled_photos.h`, `sql_util/table_helpers/scaled_photos.cpp`
- `AddScaledPhoto(transaction, source_photo_id, photo_instance_id, width, height) -> int64_t`
- `GetScaledPhoto(transaction, id) -> KeyValueTable`
- `GetScaledPhotoBySourceAndDimensions(transaction, source_photo_id, width, height) -> KeyValueTable` (for cache lookup)
- `GetScaledPhotosBySourcePhotoId(transaction, source_photo_id) -> KeyValueTableArray`
- `UpdateLastUsedAtUs(transaction, id)`
- `DeleteScaledPhoto(transaction, id)`
- `DeleteScaledPhotosBySourcePhotoId(transaction, source_photo_id)`
- `GetScaledPhotosOlderThan(transaction, cutoff_us) -> KeyValueTableArray`
- `GetTotalScaledPhotoStorageBytes(transaction) -> int64_t` (SUM of photo bytes length)
- `GetScaledPhotosOrderedByLastUsed(transaction, limit) -> KeyValueTableArray` (oldest first)

## C.4 - PhotoSupportTables table helper
**File**: `sql_util/table_helpers/photo_support_tables.h`, `sql_util/table_helpers/photo_support_tables.cpp`
- `AddPhotoSupportTable(transaction, table_name)`
- `IsPhotoSupportedForTable(transaction, table_name) -> bool`
- `GetAllPhotoSupportTables(transaction) -> KeyValueTableArray`
- `DeletePhotoSupportTable(transaction, table_name)`

## C.5 - TableItemPhotos table helper
**File**: `sql_util/table_helpers/table_item_photos.h`, `sql_util/table_helpers/table_item_photos.cpp`
- `AddTableItemPhoto(transaction, table_name, table_item_id, source_photo_id) -> int64_t`
- `GetTableItemPhoto(transaction, table_name, table_item_id) -> KeyValueTable` (by unique constraint)
- `GetTableItemPhotoById(transaction, id) -> KeyValueTable`
- `UpdateTableItemPhoto(transaction, table_name, table_item_id, source_photo_id)`
- `DeleteTableItemPhoto(transaction, table_name, table_item_id)`
- `HasPhoto(transaction, table_name, table_item_id) -> bool`

## C.6 - HomePagePhotos table helper
**File**: `sql_util/table_helpers/home_page_photos.h`, `sql_util/table_helpers/home_page_photos.cpp`
- `AddHomePagePhoto(transaction, title, description, display_order) -> int64_t`
- `GetHomePagePhoto(transaction, id) -> KeyValueTable`
- `GetAllHomePagePhotos(transaction) -> KeyValueTableArray`
- `UpdateHomePagePhoto(transaction, id, title, description, display_order)`
- `DeleteHomePagePhoto(transaction, id)`
- `GetHomePagePhotosForCarousel(transaction, count) -> KeyValueTableArray` (weighted random, favoring recent)

## C.7 - Update `sql_util/table_helpers/CMakeLists.txt`
Add all new .h and .cpp files.

## C.8 - Table helper tests
**Files**: One test file per helper:
- `sql_util/table_helpers/photo_instances_test.cpp`
- `sql_util/table_helpers/source_photos_test.cpp`
- `sql_util/table_helpers/scaled_photos_test.cpp`
- `sql_util/table_helpers/photo_support_tables_test.cpp`
- `sql_util/table_helpers/table_item_photos_test.cpp`
- `sql_util/table_helpers/home_page_photos_test.cpp`

Each test file: self-contained tests (no fixtures), create tables in transaction, test all CRUD operations. Follow the stored procedures setup pattern from CLAUDE.md.

## C.9 - Update test CMakeLists.txt
Add all test .cpp files.

## C.10 - Update `test/src/util/payment_table_test_helper.cpp`
Add photo table creation to `MakePaymentTables` so endpoint tests have photo tables available. Create in order: `photo_instances` -> `source_photos` -> `scaled_photos` -> `photo_support_tables` -> `table_item_photos` -> `home_page_photos`.

---

# Phase D: Business Logic (Images namespace)

## D.1 - Domain structs
**File**: `images/image_info.h`
```cpp
namespace Images {
    struct SourcePhotoInfo {
        int64_t id = 0;
        int64_t photoInstanceId = 0;
        std::string type;         // "jpeg", "png", etc.
        int width = 0;
        int height = 0;
        int64_t createdAtUs = 0;
        int64_t lastUpdatedAtUs = 0;
    };

    struct ScaledPhotoInfo {
        int64_t id = 0;
        int64_t sourcePhotoId = 0;
        int64_t photoInstanceId = 0;
        std::string type;
        int width = 0;
        int height = 0;
        int64_t createdAtUs = 0;
        int64_t lastUsedAtUs = 0;
    };

    struct PhotoData {
        std::vector<char> bytes;
        std::string type;
        int width = 0;
        int height = 0;
    };

    struct UploadResult {
        bool success = false;
        std::string errorMessage;
        SourcePhotoInfo sourcePhoto;
    };

    struct ScaledPhotoResult {
        bool success = false;
        std::string errorMessage;
        PhotoData photo;
    };

    struct StorageStats {
        int64_t totalScaledPhotos = 0;
        int64_t totalStorageBytes = 0;
    };
}
```

## D.2 - KeyValueTable conversion
**File**: `images/image_key_value_table.h`, `images/image_key_value_table.cpp`
- `SourcePhotoInfoToKeyValueTable(const SourcePhotoInfo&) -> KeyValueTable`
- `ScaledPhotoInfoToKeyValueTable(const ScaledPhotoInfo&) -> KeyValueTable`
- `SourcePhotoInfoFromKeyValueTable(const KeyValueTable&) -> SourcePhotoInfo`
- `ScaledPhotoInfoFromKeyValueTable(const KeyValueTable&) -> ScaledPhotoInfo`

## D.3 - ImageHelper class
**File**: `images/image_helper.h`, `images/image_helper.cpp`
```cpp
namespace Images {
class ImageHelper {
public:
    ImageHelper(DatabaseHelper databaseHelper);
    ImageHelper(DatabaseHelper databaseHelper,
                Secrets::SecretsHelperPtr secretsHelper);

    // Upload & associate a photo with a table item
    // - Validates table is in photo_support_tables
    // - Enforces max source dimensions (crop if needed)
    // - Creates photo_instance, source_photo, table_item_photo
    // - If item already has a photo, replaces it (deletes old)
    UploadResult UploadAndAssociatePhoto(
        Transaction& transaction,
        std::string_view tableName,
        int64_t tableItemId,
        const std::vector<char>& imageBytes,
        std::string_view imageType);

    // Delete photo associated with a table item
    // - Deletes table_item_photo, all scaled_photos + instances,
    //   source_photo + instance
    bool DeletePhotoForItem(
        Transaction& transaction,
        std::string_view tableName,
        int64_t tableItemId);

    // Lookup source photo info for a table item
    std::optional<SourcePhotoInfo> GetSourcePhotoForItem(
        Transaction& transaction,
        std::string_view tableName,
        int64_t tableItemId);

    // Get or create a scaled version of a source photo
    // - Checks scaled_photos cache first
    // - If not found, fetches source, resizes, stores as new scaled_photo
    // - Updates last_used_at_us
    ScaledPhotoResult GetOrCreateScaledPhoto(
        Transaction& transaction,
        int64_t sourcePhotoId,
        int width,
        int height);

    // Get scaled photo for a table item (convenience: lookup + scale)
    ScaledPhotoResult GetScaledPhotoForItem(
        Transaction& transaction,
        std::string_view tableName,
        int64_t tableItemId,
        int width,
        int height);

    // Check if a table item has a photo
    bool HasPhoto(
        Transaction& transaction,
        std::string_view tableName,
        int64_t tableItemId);

    // Reaper: delete scaled photos not used within the given window
    int64_t DeleteScaledPhotosOlderThan(
        Transaction& transaction,
        int64_t maxAgeUs);

    // Storage stats
    StorageStats GetScaledPhotoStorageStats(Transaction& transaction);

    // Delete oldest scaled photos until storage is at or below targetBytes
    int64_t DeleteOldestScaledPhotosUntilSize(
        Transaction& transaction,
        int64_t targetBytes);

private:
    DatabaseHelper databaseHelper_;
    Secrets::SecretsHelperPtr secretsHelper_;
    TableHelpers::PhotoInstances photoInstances_;
    TableHelpers::SourcePhotos sourcePhotos_;
    TableHelpers::ScaledPhotos scaledPhotos_;
    TableHelpers::PhotoSupportTables photoSupportTables_;
    TableHelpers::TableItemPhotos tableItemPhotos_;

    // Internal: delete a source photo and all its scaled photos + instances
    void CascadeDeleteSourcePhoto(
        Transaction& transaction, int64_t sourcePhotoId);

    // Internal: enforce max dimensions via resize (preserving aspect ratio)
    // Uses BoundingRect::GetClippedRect to compute target dimensions,
    // then ImageResize::ResizeImage to perform the resize
    PhotoData EnforceMaxDimensions(
        Transaction& transaction,
        const std::vector<char>& imageBytes,
        std::string_view imageType,
        int width,
        int height);
};
}
```

## D.4 - Create `images/CMakeLists.txt`
List all .h and .cpp files, link to `knotty_yoga_core`.

## D.5 - Update parent `src/CMakeLists.txt`
Add `add_subdirectory(images)`.

## D.6 - ImageHelper tests
**File**: `images/image_helper_test.cpp`
- Test UploadAndAssociatePhoto (valid table, invalid table, oversized image gets cropped)
- Test DeletePhotoForItem (cascades correctly, returns false if no photo)
- Test GetSourcePhotoForItem (found, not found)
- Test GetOrCreateScaledPhoto (creates on first call, returns cached on second)
- Test HasPhoto
- Test DeleteScaledPhotosOlderThan
- Test GetScaledPhotoStorageStats
- Test DeleteOldestScaledPhotosUntilSize
- Test replacing a photo (upload when one already exists)

## D.7 - KeyValueTable conversion tests
**File**: `images/image_key_value_table_test.cpp`
- Round-trip tests for SourcePhotoInfo and ScaledPhotoInfo

---

# Phase E: Endpoints

## E.1 - Upload photo endpoint
**File**: `endpoints/upload_photo.h`, `endpoints/upload_photo.cpp`
- `POST /api/upload_photo/<string:table>/<int:item_id>?type=jpeg`
- Request body: raw image bytes (`Content-Type: application/octet-stream`)
- Image type from `req.url_params.get("type")` - must be "jpeg" or "png"
- Auth: admin required (for general table uploads)
- Validates: file size <= `image_max_upload_bytes` secret, type is jpeg/png
- Calls `ImageHelper::UploadAndAssociatePhoto`
- Returns: JSON with source photo info

## E.2 - Upload user photo endpoint
**File**: `endpoints/upload_user_photo.h`, `endpoints/upload_user_photo.cpp`
- `POST /api/upload_user_photo?type=jpeg`
- Request body: raw image bytes (`Content-Type: application/octet-stream`)
- Image type from `req.url_params.get("type")` - must be "jpeg" or "png"
- Auth: logged in (uses session person_id, table="people")
- Validates: file size <= `image_max_upload_bytes`, type is jpeg/png
- Calls `ImageHelper::UploadAndAssociatePhoto` with table="people" and user's own person_id
- Returns: JSON with source photo info

## E.3 - Get photo endpoint (binary response)
**File**: `endpoints/get_photo.h`, `endpoints/get_photo.cpp`
- `GET /api/get_photo/<string:table>/<int:item_id>`
- Auth: logged in
- Fetches source photo bytes via ImageHelper
- Returns: binary image data with `Content-Type: image/{type}`
- Sets `Cache-Control: public, max-age=86400` and `ETag` based on photo_instance_id + last_updated_at_us

## E.4 - Get scaled photo endpoint (binary response)
**File**: `endpoints/get_scaled_photo.h`, `endpoints/get_scaled_photo.cpp`
- `GET /api/get_scaled_photo/<string:table>/<int:item_id>/<int:width>/<int:height>`
- Auth: logged in
- Non-admin users: enforces max public scaled dimensions from secrets
- Uses `BoundingRect::GetClippedRect` to compute aspect-ratio-preserving dimensions
- Calls `ImageHelper::GetScaledPhotoForItem`
- Returns: binary image data with `Content-Type: image/{type}`
- Sets `Cache-Control: public, max-age=86400` and `ETag`

## E.5 - Delete photo endpoint
**File**: `endpoints/delete_photo.h`, `endpoints/delete_photo.cpp`
- `DELETE /api/delete_photo/<string:table>/<int:item_id>`
- Auth: admin required (or user deleting their own people photo)
- Calls `ImageHelper::DeletePhotoForItem`
- Returns: 200 OK

## E.6 - Has photo endpoint
**File**: `endpoints/has_photo.h`, `endpoints/has_photo.cpp`
- `GET /api/has_photo/<string:table>/<int:item_id>`
- Auth: logged in
- Returns: JSON `{ has_photo: true/false }`

## E.7 - Home page photos endpoint
**File**: `endpoints/home_page_photos.h`, `endpoints/home_page_photos.cpp`
- `GET /api/home_page_photos/<int:count>`
- Auth: none (public)
- Queries `home_page_photos` with recency-weighted random selection
- Returns: JSON array of `{ id, title, description, photo_url }` where photo_url is the get_scaled_photo URL

## E.10 - Update `delete_item` endpoint for photo cleanup
**File**: `endpoints/delete_item.cpp`
- Before deleting a row, check if the table is in `photo_support_tables`
- If so, call `ImageHelper::DeletePhotoForItem` first to clean up associated photos
- This handles cascade cleanup at the application layer (see Open Question 4)

## E.8 - Update `endpoints/CMakeLists.txt`
Add all new endpoint .h and .cpp files.

## E.9 - Endpoint tests
**Files**:
- `endpoints/upload_photo_test.cpp`
- `endpoints/upload_user_photo_test.cpp`
- `endpoints/get_photo_test.cpp`
- `endpoints/get_scaled_photo_test.cpp`
- `endpoints/delete_photo_test.cpp`
- `endpoints/has_photo_test.cpp`
- `endpoints/home_page_photos_test.cpp`

Each test: use `TestUtil::MakePaymentTables` (which now includes photo tables), test auth requirements, test happy path, test error cases.

---

# Phase F: Client-side Admin Image Support

## F.1 - Add photo-related methods to ServerAccess interface
**File**: `ui/src/app/shared/types/ServerAccess.ts`
```typescript
uploadPhoto(tableName: string, tableItemId: number, imageData: string, imageType: string): Observable<SourcePhotoInfo>;
uploadUserPhoto(imageData: string, imageType: string): Observable<SourcePhotoInfo>;
deletePhoto(tableName: string, tableItemId: number): Observable<void>;
hasPhoto(tableName: string, tableItemId: number): Observable<{ has_photo: boolean }>;
getHomePagePhotos(count: number): Observable<HomePagePhotoInfo[]>;
```

## F.2 - Add photo types
**File**: `ui/src/app/shared/types/photo.types.ts` (NEW)
```typescript
export interface SourcePhotoInfo {
  id: number;
  type: string;
  width: number;
  height: number;
  created_at_us: number;
}

export interface HomePagePhotoInfo {
  id: number;
  title?: string;
  description?: string;
  photo_url: string;
}
```

## F.3 - Implement in ServerAccessNetwork
**File**: `ui/src/app/shared/services/network/ServerAccessNetwork.ts`
- `uploadPhoto`: POST to `/api/upload_photo` with JSON body
- `uploadUserPhoto`: POST to `/api/upload_user_photo`
- `deletePhoto`: DELETE to `/api/delete_photo/{table}/{id}`
- `hasPhoto`: GET to `/api/has_photo/{table}/{id}`
- `getHomePagePhotos`: GET to `/api/home_page_photos/{count}`

## F.4 - Implement in ServerAccessMock
**File**: `ui/src/app/shared/services/network/ServerAccess.mock.ts`
- In-memory photo state
- Mock responses

## F.5 - Update ServerAccessProxy
**File**: `ui/src/app/shared/services/network/ServerAccess.ts`
- Delegate new methods

## F.6 - Add `has_photo_support` to table schema metadata
**File**: `ui/src/app/shared/types/TableSchema.ts`
- Add `has_photo_support?: boolean` to TableSchema interface

On the server side, `DatabaseMetadata` should include `has_photo_support: true` for tables in `photo_support_tables`.

**Server file**: `sql_util/json/database_rest_helper.cpp`
- In `GenerateTableMetadata`, check if table is in `photo_support_tables` and add `has_photo_support` to the table JSON.
- Thread `PhotoSupportTables` table helper through the metadata generation functions.

## F.7 - Photo thumbnail in table-view-control
**File**: `ui/src/app/controls/table-view-control/`
- If table has photo support, add a "Photo" column showing a small thumbnail (`<img>` tag pointing to `/api/get_scaled_photo/{table}/{id}/50/50`)
- Show a placeholder icon if no photo

## F.8 - Photo display and upload in edit view (composite-row-control)
**File**: `ui/src/app/controls/composite-row-control/`
- If table has photo support, show current photo (larger, e.g., 300x300) above the form
- Add file input for photo upload
- On file select: read file, convert to base64, call `uploadPhoto`
- Add delete photo button
- Create a new `photo-upload` control component for this

## F.9 - Photo upload control component
**Files**: `ui/src/app/controls/photo-upload/photo-upload.component.ts/html/scss`
- Standalone component
- Input: `tableName`, `tableItemId`, `hasPhotoSupport`
- Displays current photo thumbnail from `/api/get_scaled_photo/...`
- File input with drag-and-drop area
- Upload button, delete button
- Loading states

## F.10 - Tests for client-side admin
- `photo-upload.component.spec.ts` - Component tests
- `ServerAccess.mock.spec.ts` - Add tests for new mock methods
- Update `table-view-control` and `composite-row-control` specs

---

# Phase G: Client-side User Account Photo ✅ COMPLETE

## G.1 - Photo display on account page ✅
**File**: `ui/src/app/pages/account/user_information/`
- Show user photo (or placeholder) at top of account info card
- Photo from `/api/get_scaled_photo/people/{person_id}/200/200`
- **Implemented**: Added `PhotoUploadComponent` with `userMode=true` to user_information page, centered above the info fields with a border separator

## G.2 - Photo upload on account page ✅
**File**: `ui/src/app/pages/account/user_information/`
- Reuse `photo-upload` component
- Calls `uploadUserPhoto` endpoint (doesn't need admin)
- **Implemented**: `photo-upload` component's `@Input() userMode` switches between `uploadPhoto` (admin) and `uploadUserPhoto` (user) endpoints

## G.3 - Photo thumbnail in dashboard header ✅
**File**: `ui/src/app/shared/components/header/`
- When user is logged in, show small photo thumbnail next to name
- Photo from `/api/get_scaled_photo/people/{person_id}/32/32`
- Fall back to existing icon if no photo
- **Implemented**: Added `avatarUrl` to `BaseHeaderButton` type, rendered as round 32x32 `<img>` in dropdown trigger, with `onAvatarError` fallback to mat-icon

## G.4 - Tests ✅
- Update `user_information` specs - added photo-upload rendering tests (auth/non-auth)
- Update header component specs - added `onAvatarError` fallback test
- All 348 tests passing

### Prerequisites completed:
- Server: `get_user_info` now returns `person_id`
- Types: `UserInfo.person_id`, `AuthData.personId` added
- Auth service: maps `person_id` to `personId`
- Mock: includes `person_id: 1`
- Header types: `avatarUrl` on `BaseHeaderButton`
- Mock header response: constructs avatar URL from `authData.personId`

---

# Phase H: Home Page Photos & Carousel ✅ COMPLETE

## H.1 - Recency-weighted random selection query ✅
The `GetHomePagePhotosForCarousel` table helper was already implemented with weighted random selection (`ORDER BY random() * created_at_us DESC`).
- **Fixed**: `HomePagePhotoInfo` type was mismatched (had `table_name`/`table_item_id` which don't exist in the response). Updated to match actual endpoint response: `id`, `title`, `description`, `display_order`, `created_at_us`.
- **Fixed**: `ServerAccessNetwork.getHomePagePhotos` now unwraps the `{ items: [...] }` wrapper from the endpoint.

## H.2 - Home page carousel component ✅
**Files**: `ui/src/app/pages/public/home-page/photo-carousel/photo-carousel.component.ts/html/scss`
- Standalone component fetching photos via `getHomePagePhotos(6)`
- Crossfade transitions between slides (0.6s ease-in-out)
- Auto-rotate every 5 seconds with pause on hover
- Navigation arrows (left/right) and dot indicators
- Title/description overlay with gradient background
- 16:9 aspect ratio viewport, responsive
- Photo URLs constructed as `/api/get_scaled_photo/home_page_photos/{id}/800/450`

## H.3 - Integrate carousel into home page ✅
- Replaced the gray placeholder squares with `<app-photo-carousel>` at the top of the home page
- Removed the old `graySquare` hero div and placeholder gallery sections

## H.4 - Tests ✅
- Created `photo-carousel.component.spec.ts` with 10 tests (creation, photo loading, navigation, wrapping, dots/arrows rendering, pause/resume, URL generation)
- Updated `home-page.component.spec.ts` with `SERVER_ACCESS_TOKEN` provider
- Updated `ServerAccess.mock.spec.ts` test for non-empty photo data
- Added 6 mock home page photos to `ServerAccessMock`
- All 359 tests passing

---

# Phase I: Periodic Task Scheduler (Thread Pool Extension)

## I.1 - Add PeriodicTask support to ThreadPool
**File**: `util/thread_pool.h`, `util/thread_pool.cpp`

Extend the existing ThreadPool (which uses `boost::asio::thread_pool`) to support periodic task execution with clean cancellation.

```cpp
class PeriodicTaskHandle {
public:
    void Cancel();        // Request cancellation
    bool IsCancelled() const;
    void Join();          // Block until the task has stopped
private:
    std::shared_ptr<std::atomic<bool>> cancelled_;
    std::shared_ptr<std::promise<void>> completion_;
};

// New method on ThreadPool:
PeriodicTaskHandle QueuePeriodic(
    std::function<void()> fn,
    std::chrono::microseconds interval);
```

Implementation approach using `boost::asio::steady_timer`:
- Create a `steady_timer` on the thread pool's io_context
- On each timer expiry, execute the callback, then reschedule
- `Cancel()` sets the atomic flag and cancels the timer
- `Join()` waits on a future that is fulfilled when the task loop exits
- `ThreadPool::Shutdown()` cancels all periodic tasks before joining

## I.2 - Integrate reaper task into server startup
**File**: `main.cpp` (or a new `images/reaper_task.h/cpp`)
- On server startup, read `scaled_photo_reaper_interval_us` and `scaled_photo_max_age_us` from secrets
- Start a periodic task that calls `ImageHelper::DeleteScaledPhotosOlderThan`
- Store the `PeriodicTaskHandle` for clean shutdown

## I.3 - Tests
**File**: `util/thread_pool_test.cpp` (extend existing)
- Test `QueuePeriodic` executes callback multiple times
- Test `Cancel` stops future executions
- Test `Join` blocks until cancelled task stops
- Test periodic task survives callback exceptions
- Test `Shutdown` cancels all periodic tasks

---

# Implementation Order Summary

1. Image resize enhancements (prerequisite for business logic)
2. Phase A: Database schema updates
3. Phase B: Configuration secrets
4. Phase C: Table helpers + tests
5. Phase D: Business logic + tests
6. Phase E: Endpoints + tests
7. Phase F: Client admin image support
8. Phase G: Client user account photo
9. Phase H: Home page carousel
10. Phase I: Periodic task scheduler + reaper integration

---

# Open Questions (Resolved)

## 1. Photo upload format
**Decision: Raw binary POST body with URL parameters.**

Crow fully supports POST routes with URL parameters where the entire body is raw binary image data. This is already used in the codebase (e.g., `purchase_pay_card.cpp` uses `"/api/purchase_pay_card/<int>"`). The approach:

- `POST /api/upload_photo/<string:table>/<int:item_id>?type=jpeg` with raw image bytes as body
- `POST /api/upload_user_photo?type=jpeg` with raw image bytes as body
- Access body via `req.body` (a `std::string` that holds binary data)
- Access image type via `req.url_params.get("type")`
- On the client side, send with `Content-Type: application/octet-stream`

This avoids the 33% overhead of base64 encoding and is simpler than multipart. Crow also supports multipart via `crow::multipart::message_view` if we ever need it later, but raw body is cleaner for single-file uploads.

**Plan updates**: Endpoints E.1 and E.2 updated to use this pattern instead of JSON with base64.

## 2. Max file size
**Decision: Add `image_max_upload_bytes` configurable secret, default 10MB (10485760 bytes).**

10MB is sensible because: high-quality JPEG photos from modern phones are typically 3-8MB, and PNG screenshots rarely exceed 5MB. This provides headroom while preventing abuse.

**Plan update**: Added to Phase B secrets.

## 3. Image type restriction
**Decision: Restrict to JPEG and PNG only.** BMP and TIFF are uncommon for web use. The upload endpoint will validate the `type` parameter and reject anything other than `"jpeg"` or `"png"`.

## 4. Cascade delete on table row deletion
**Decision: Option A - Application-level cascade in the `delete_item` endpoint.**

**Background:** All foreign keys in the codebase use `ON DELETE CASCADE` (hardcoded in `db_and_table_operations.cpp`). However, `table_item_photos` uses a generic pattern (`table_name` VARCHAR + `table_item_id` BIGINT) rather than a proper FK to each supported table, so PostgreSQL CASCADE won't trigger when a parent row (e.g., `people`) is deleted.

**Chosen approach:** When the `delete_item` endpoint deletes a row, it checks if the table is in `photo_support_tables` and calls `ImageHelper::DeletePhotoForItem` first. This keeps all logic in the application layer, consistent with our architecture where business logic lives in C++ and everything in SQL is represented in metadata.

**Alternatives considered (may revisit later if needed):**
- **Option B - PostgreSQL trigger per-table**: A `BEFORE DELETE` trigger on each photo-supported table that cleans up `table_item_photos`. Would require extending `DatabaseInfo` metadata with a new concept (e.g., `AddDeleteTrigger`) to represent triggers in code. More robust if rows are deleted outside the application, but adds SQL complexity not represented in current metadata.
- **Option C - Stored procedure + trigger**: A shared `cleanup_table_item_photo()` stored procedure called via per-table triggers, similar to the `now_us()` pattern. Same trade-offs as Option B but with shared logic.

If we find that rows in photo-supported tables are being deleted through paths other than `delete_item` (direct SQL, other endpoints), we may need to revisit and implement Option B or C to guarantee cleanup at the database level.

## 5. Home page photo management
**Decision: Use the generic admin table editor.** `home_page_photos` will be in `admin_top_level_tables` and managed like any other table. No bespoke UI needed.

## 6. Carousel library
**Decision: Custom CSS scroll-snap implementation (zero dependencies).**

Uses native browser scrolling with `scroll-snap-type: x mandatory`. Zero bundle impact, best performance and accessibility, works with our existing TailwindCSS. We implement auto-rotate timer, prev/next buttons, and navigation dots ourselves in the Angular component.

**Upgrade path**: If we later need complex effects (parallax, 3D transitions, advanced touch gestures), **Swiper** (v12.1.2, ~31KB gzipped) is the recommended upgrade. It uses web components (`<swiper-container>`, `<swiper-slide>`) and is framework-agnostic, so it works with Angular 19 via `CUSTOM_ELEMENTS_SCHEMA`. Install with `npm install swiper`.

**Other options evaluated and rejected:**
- **ngx-owl-carousel-o** (~50-60KB): True Angular component but carries jQuery/Owl Carousel legacy. Larger bundle for similar features.
- **Angular CDK DragDrop** (already installed): Not designed for carousels, poor touch/swipe UX.

## 7. Photo caching headers
**Decision: Yes, use HTTP cache headers.** The `get_scaled_photo` and `get_photo` endpoints will set:
- `Cache-Control: public, max-age=86400` (24 hours)
- `ETag` based on photo_instance_id + last_updated_at_us for cache validation

**Plan update**: Added to Phase E endpoint implementations.

## 8. Aspect ratio on resize
**Decision: Preserve aspect ratio using `BoundingRect::GetClippedRect`.**

The existing `util/bounding_rect.h/cpp` already implements exactly this! `GetClippedRect(inputRect, clippedRect)` scales proportionally to fit within the bounding box:
- If input fits, returns as-is
- If only one dimension exceeds, scales proportionally
- If both exceed, scales to fit the tighter constraint

The image resize flow becomes:
1. `BoundingRect::GetClippedRect({srcWidth, srcHeight}, {requestedWidth, requestedHeight})` to compute target dimensions
2. `ImageResize::ResizeImage(bytes, targetWidth, targetHeight, type)` to perform the resize

**Plan update**: Phase D's ImageHelper will use BoundingRect. No new resize functions needed in image_resize.h.

## 9. Background reaper task
**Decision: Extend ThreadPool with periodic execution support. This is a separate Phase (Phase I).**

The existing `util/thread_pool.h/cpp` uses `boost::asio::thread_pool` with 8 threads. The cleanest extension is to add a `boost::asio::steady_timer` based periodic task that:
- Runs a callback at a configurable interval
- Returns a cancellation handle (or uses `std::atomic<bool>`)
- Supports clean shutdown via `ThreadPool::Shutdown()`

See new **Phase I** below.

## 10. One photo per item
**Decision: Keep one photo per row (UNIQUE constraint on table_name/table_item_id).** For future multi-photo needs, create a linking table with FK to the multi-photo table and multiple entries in `table_item_photos` via that linked table.

---

# Phase J: Carousel Redesign - Overlay Captions, Scroll Support, Remove display_order

## Context
The carousel currently shows captions in a black bar below each image, only supports button navigation (no scroll/drag), and the `home_page_photos` table has a `display_order` column the user doesn't want. The server already returns photos randomly weighted by recency via `ORDER BY random() * created_at_us DESC`, so no SQL logic changes are needed for that.

## J.1 - Frontend: Caption Overlay & Scroll Support
**Files:**
- `ui/src/app/pages/public/home-page/photo-carousel/photo-carousel.component.html`
- `ui/src/app/pages/public/home-page/photo-carousel/photo-carousel.component.scss`
- `ui/src/app/pages/public/home-page/photo-carousel/photo-carousel.component.ts`
- `ui/src/app/pages/public/home-page/photo-carousel/photo-carousel.component.spec.ts`

**Caption**: Move `.card-caption` inside the image area as an absolute-positioned overlay on the bottom-left with a subtle semi-transparent background. Remove the dark background bar.

**Scroll/drag**: Add mouse drag and touch drag support to the carousel track. Track `mousedown`/`mousemove`/`mouseup` and `touchstart`/`touchmove`/`touchend` to allow dragging the carousel. Keep the auto-slide and button navigation.

## J.2 - Frontend: Remove display_order from types and mock
**Files:**
- `ui/src/app/shared/types/photo.types.ts` - Remove `display_order` from `HomePagePhotoInfo`
- `ui/src/app/shared/services/network/ServerAccess.mock.ts` - Remove `display_order` from mock data

## J.3 - Backend: Remove display_order from schema
**Files:**
- `server/knottyyoga_server/src/db_schema/home_page_photos.h` - Remove `kHomePagePhotosDisplayOrder` constant
- `server/knottyyoga_server/src/db_schema/home_page_photos.cpp` - Remove display_order column definition

## J.4 - Backend: Remove display_order from table helpers
**Files:**
- `server/knottyyoga_server/src/sql_util/table_helpers/home_page_photos.h` - Remove `displayOrder` params from `AddHomePagePhoto` and `UpdateHomePagePhoto`
- `server/knottyyoga_server/src/sql_util/table_helpers/home_page_photos.cpp` - Remove display_order from SQL INSERT/UPDATE

## J.5 - Backend: Remove display_order from database initialization
**File:** `server/knottyyoga_server/src/database_helper/create_database.cpp`
- Remove the admin metadata rows that reference `display_order` for the home_page_photos table

## J.6 - Backend: Update tests
**Files:**
- `server/knottyyoga_server/src/sql_util/table_helpers/home_page_photos_test.cpp` - Remove display_order from test calls and assertions
- `server/knottyyoga_server/src/endpoints/home_page_photos_test.cpp` - Remove display_order from test calls

## J.7 - Database migration
User will need to run: `ALTER TABLE home_page_photos DROP COLUMN display_order;`

## Verification
- Run `ng test` - all Angular tests pass
- Rebuild C++ server and run `knottyyoga_tests` (or at least the home_page_photos tests)
- Visual check: carousel shows images with caption overlaid, supports mouse drag scrolling, auto-slides