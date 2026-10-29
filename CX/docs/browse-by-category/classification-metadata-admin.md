# BACKEND LOGIC & FLOW SPECS

Classification Metadata — Admin Management

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | Alice | 08/06/2026 |

---

## I. Introduction

### 1. Purpose

> The `ClassificationMetadata` entity provides admin-managed display configuration for each classification category shown in the **Browse by Category** section. Because category classifications (`single_malt`, `irish_single_malt`, etc.) are derived from the `classification` field on Master Casks — and are not standalone DB records — there is no existing mechanism to attach display assets (images, visibility toggles, label overrides) to a category.
>
> This new entity fills that gap, allowing admins to:
> - Upload a representative image per classification (shown on the Browse by Category card)
> - Override the display name if needed (falls back to `MasterCask.categoryLabel` if not set)
> - Toggle category visibility in the Browse by Category section without deleting data
>
> The `classificationImageUrl` from this entity is consumed by the `GET /api/categories/browse` endpoint (see [api-requirements.md](./api-requirements.md)).

### 2. Definitions

| Term | Definition |
|------|------------|
| Classification (slug) | The enum-like string identifying a category (e.g., `single_malt`). Derived from `MasterCask.classification`. Source of truth: `GET /api/cask-metadata/classifications`. |
| `categoryLabel` | Short, user-facing display name for the category (e.g., "Single Malt"). Lives on the `MasterCask` entity. All master casks sharing a classification have the same `categoryLabel`. |
| `classificationLabel` | Longer, more specific display name (e.g., "Single Malt Scotch"). Not displayed on the Browse by Category card; used in other product contexts. |
| Classification Metadata | Admin-managed configuration record for a classification, keyed by the `classification` slug. |
| `isVisible` | A flag that allows admins to suppress a category from the Browse by Category section without affecting cask data or market stats. |

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| **CM-BR-1** | **1:1 mapping** | One `ClassificationMetadata` record per `classification` slug. The slug is the primary key; duplicates are rejected. |
| **CM-BR-2** | **displayName fallback** | If `displayName` is `null`, the Browse by Category API falls back to `MasterCask.categoryLabel` as the card label. |
| **CM-BR-3** | **Visibility gate** | If `isVisible = false` for a classification, it MUST be excluded from the `GET /api/categories/browse` response, regardless of its market stats. |
| **CM-BR-4** | **Image storage** | `classificationImageUrl` must be an S3-hosted URL (same bucket pattern as distillery images). Images are uploaded via multipart form; the backend handles S3 upload and stores the resulting URL. |
| **CM-BR-5** | **Slug validation** | On create/update, the `classification` slug MUST exist in the active classification enum (`GET /api/cask-metadata/classifications`). Reject unknown slugs with 400. |
| **CM-BR-6** | **Stats refresh on metadata change** | Any update to `ClassificationMetadata` (image, visibility, displayName) MUST trigger an async refresh of the `ClassificationVolumeStats` record for that classification, so the Browse by Category API serves fresh data immediately. |

---

## II. Functional Requirements

### 1. Admin: Manage Classification Metadata

> Admins can create or update the display configuration for any classification category. The typical workflow is: after a new `classification` appears on the platform (because a new Master Cask with that classification was listed), an admin visits the Classification Metadata admin panel and uploads an image + reviews the display config.

#### 1.1. Data Description

| No. | Field | Data Type | Required | Description |
|-----|-------|-----------|----------|-------------|
| 1 | `classification` | string | ✅ | Classification slug (e.g., `single_malt`). Primary key. Must exist in the classification enum (**CM-BR-5**). |
| 2 | `classificationImageUrl` | string \| null | ❌ | S3-hosted URL of the category's representative image. Set by the BE after admin uploads via multipart. `null` if no image has been uploaded. |
| 3 | `displayName` | string \| null | ❌ | Admin override for the card label. If `null`, the Browse by Category API uses `MasterCask.categoryLabel` (**CM-BR-2**). Max 60 chars. |
| 4 | `isVisible` | boolean | ✅ | Controls visibility in the Browse by Category section (**CM-BR-3**). Default: `true`. |
| 5 | `createdAt` | datetime | ✅ (auto) | Record creation timestamp. |
| 6 | `updatedAt` | datetime | ✅ (auto) | Last update timestamp. |

#### 1.2. Use Cases

##### UC-1. Create Classification Metadata Record

| Item | Description |
|------|-------------|
| **Objective** | Create a new `ClassificationMetadata` record for a classification that does not yet have one |
| **Actor** | Admin |
| **Trigger** | Admin submits the Create Classification Metadata form |
| **Pre-condition(s)** | The `classification` slug exists in the enum; no existing record for this slug |
| **Post-condition(s)** | New `ClassificationMetadata` record created; async stats refresh triggered (**CM-BR-6**) |

**Activity Flow & Business Rules:**

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Validate slug | CM-BR-5 | Verify `classification` exists in `GET /api/cask-metadata/classifications`. Return 400 if unknown. |
| 2 | Check uniqueness | CM-BR-1 | Reject if a record already exists for this slug (409 Conflict). |
| 3 | Upload image (if provided) | CM-BR-4 | Upload image to S3; store resulting URL as `classificationImageUrl`. |
| 4 | Persist record | — | Insert `ClassificationMetadata` with provided fields. Default `isVisible = true` if not set. |
| 5 | Trigger stats refresh | CM-BR-6 | Async refresh of `ClassificationVolumeStats` for this classification. |
| 6 | Return created record | — | 201 Created with the full `ClassificationMetadata` payload. |

**Happy Path:**

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Valid slug + image uploaded | Record created; image uploaded to S3; URL stored; 201 returned |
| 2 | Valid slug, no image | Record created with `classificationImageUrl = null`; 201 returned |

**Negative Path:**

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | Unknown `classification` slug | Reject | 400 — "Unknown classification" |
| 2 | Duplicate slug | Reject | 409 — "Category metadata already exists for this classification. Use PUT to update." |
| 3 | Unauthenticated | Reject | 401 Unauthorized |
| 4 | Non-admin role | Reject | 403 Forbidden |
| 5 | Image upload fails (S3 error) | Reject with error | 500 — "Image upload failed. Please try again." |
| 6 | `displayName` exceeds 60 chars | Reject | 400 — "displayName must not exceed 60 characters" |

---

##### UC-2. Update Classification Metadata Record

| Item | Description |
|------|-------------|
| **Objective** | Update the image, display name, or visibility of an existing `ClassificationMetadata` record |
| **Actor** | Admin |
| **Trigger** | Admin submits the Edit Classification Metadata form |
| **Pre-condition(s)** | A `ClassificationMetadata` record exists for the given `classification` slug |
| **Post-condition(s)** | Record updated; async stats refresh triggered (**CM-BR-6**) |

**Activity Flow & Business Rules:**

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Look up record | — | Find `ClassificationMetadata` by `classification` slug. Return 404 if not found. |
| 2 | Upload new image (if provided) | CM-BR-4 | If a new image is included in the request, upload to S3; update `classificationImageUrl`. If no image in request, keep existing URL. |
| 3 | Apply field updates | CM-BR-2, CM-BR-3 | Update `displayName`, `isVisible` as provided. Null `displayName` clears the override (reverts to `categoryLabel` fallback). |
| 4 | Persist changes | — | Update record; set `updatedAt = NOW()`. |
| 5 | Trigger stats refresh | CM-BR-6 | Async refresh of `ClassificationVolumeStats` for this classification. |
| 6 | Return updated record | — | 200 OK with the full updated `ClassificationMetadata` payload. |

**Happy Path:**

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Update `isVisible = false` | Category suppressed from Browse by Category on next API call |
| 2 | Upload a new image | Old S3 object retained or cleaned up (BE decision); new URL stored |
| 3 | Clear `displayName` (set to null) | Card reverts to showing `MasterCask.categoryLabel` |

**Negative Path:**

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | Record not found | Return not found | 404 — "Category metadata not found" |
| 2 | Unauthenticated | Reject | 401 |
| 3 | Non-admin | Reject | 403 |

---

##### UC-3. List All Classification Metadata Records

| Item | Description |
|------|-------------|
| **Objective** | Return all `ClassificationMetadata` records for the admin panel overview |
| **Actor** | Admin |
| **Trigger** | Admin navigates to the Classification Metadata management page |
| **Pre-condition(s)** | Admin is authenticated |
| **Post-condition(s)** | Returns all existing records |

**Happy Path:**

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Records exist | Return array of all `ClassificationMetadata` records |
| 2 | No records exist yet | Return empty array `[]` |

> [!NOTE]
> This list does NOT need to match the Browse by Category public API. It shows ALL configured records, including those with `isVisible = false`. The public Browse API applies its own filtering separately.

---

##### UC-4. Get Single Classification Metadata Record

| Item | Description |
|------|-------------|
| **Objective** | Fetch a single `ClassificationMetadata` record by classification slug |
| **Actor** | Admin |
| **Trigger** | Admin clicks "Edit" on a classification metadata record |
| **Pre-condition(s)** | Admin is authenticated |

**Negative Path:**

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | Record not found | Return not found | 404 |

---

#### 1.3. Edge Cases

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|-|
| 1 | Admin sets `isVisible = false` for a classification that has active casks | Category disappears from Browse by Category section. Cask data and market stats are unaffected. | Stats refresh triggered; next Browse API call excludes this category |
| 2 | A new classification appears on platform (new master cask listed) but no `ClassificationMetadata` exists | `GET /api/categories/browse` includes the category with `classificationImageUrl = null` and displays `categoryLabel` as the label. Admins can create a metadata record later. | No metadata required for category to appear |
| 3 | Admin deletes (or clears) `classificationImageUrl` | FE falls back to the generic cask placeholder image | Stats refresh triggered |
| 4 | Admin uploads a non-image file | Reject at validation layer | 400 — "Only image files are accepted (jpg, png, webp)" |

---

## III. API Endpoints — New (Required)

> No existing endpoint manages `ClassificationMetadata`. The following new endpoints are required.

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| `GET` | `/api/classification-metadata` | List all classification metadata records | 🔒 Admin |
| `GET` | `/api/classification-metadata/{classification}` | Get metadata for one classification | 🔒 Admin |
| `POST` | `/api/classification-metadata` | Create classification metadata (multipart, optional image) | 🔒 Admin |
| `PUT` | `/api/classification-metadata/{classification}` | Update classification metadata (multipart, optional new image) | 🔒 Admin |

> [!NOTE]
> A `DELETE` endpoint is intentionally **not specified** at this stage. If admins need to "remove" metadata, they should use `isVisible = false` to suppress display, or clear `classificationImageUrl` and `displayName` fields via PUT. Hard deletion risks orphaning the category display configuration if the same classification reappears later.

### Request Format (POST / PUT)

`Content-Type: multipart/form-data`

| Field | Type | Description |
|---|---|---|
| `classification` | string | (POST only) Classification slug |
| `image` | file \| null | Optional image file. Accepted: `jpg`, `png`, `webp`. Max size: 5 MB. |
| `displayName` | string \| null | Optional display name override |
| `isVisible` | boolean | Visibility flag |

### Response Format (single record)

```json
{
  "classification": "single_malt",
  "classificationImageUrl": "https://caskx-exchange-static-assets.s3.us-east-1.amazonaws.com/category-metadata/single_malt/cover.webp",
  "displayName": null,
  "isVisible": true,
  "createdAt": "2026-06-08T04:00:00.000Z",
  "updatedAt": "2026-06-08T04:00:00.000Z"
}
```

---

## IV. Data Model

### Entity: ClassificationMetadata — new

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `classification` | string | ✅ | — | Classification slug. Primary key. FK to classification enum. |
| `classificationImageUrl` | string \| null | ❌ | `null` | S3 URL of the classification image. Set by BE after admin uploads. |
| `displayName` | string \| null | ❌ | `null` | Admin override for the card label. Falls back to `MasterCask.categoryLabel` if null (**CM-BR-2**). Max 60 chars. |
| `isVisible` | boolean | ✅ | `true` | Visibility gate for Browse by Category section (**CM-BR-3**). |
| `createdAt` | datetime | ✅ | `NOW()` | Record creation timestamp. |
| `updatedAt` | datetime | ✅ | `NOW()` | Last update timestamp. |

### S3 Key Convention

```
category-metadata/{classification}/{filename}.{ext}
```

Example:
```
category-metadata/single_malt/cover.webp
```

---

## V. Notifications / Side Effects

| # | Trigger Event | Effect | Description |
|---|---|---|---|
| 1 | `ClassificationMetadata` created or updated | Async `ClassificationVolumeStats` refresh (**CM-BR-6**) | Ensures `classificationImageUrl`, `displayName`, and `isVisible` changes are reflected immediately in the public Browse by Category API |

---

## VI. Permissions & Access Control

| Use Case / Action | Anonymous | Buyer | Seller | Admin |
|---|---|---|---|---|
| List all classification metadata | ❌ | ❌ | ❌ | ✅ |
| Get single classification metadata | ❌ | ❌ | ❌ | ✅ |
| Create classification metadata | ❌ | ❌ | ❌ | ✅ |
| Update classification metadata | ❌ | ❌ | ❌ | ✅ |

> All endpoints are Admin-only. Buyers and Sellers never interact with this entity directly — they only see the derived output via the public `GET /api/categories/browse` endpoint.

---

_End of Document_
