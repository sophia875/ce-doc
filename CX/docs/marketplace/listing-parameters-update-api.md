# Marketplace Listing Query Parameters Update API

This document defines the requirements for adding and enhancing query parameters for the marketplace listing endpoint. These parameters support advanced filtering, ownership verification, custom sorting, and pagination for the Master Cask grid layout.

---

## Endpoint Overview

### GET `/api/casks/listing`
*(Proposed New Path: `GET /api/cask-masters/listing` - see [api-change-requests.md](file:///d:/CX/docs/marketplace/api-change-requests.md))*

Returns a paginated, sorted, and filtered list of active Master Casks.

---

## Query Parameters (Inputs)

### 1. New & Enhanced Parameters

The following parameters are added or enhanced to support the updated Marketplace grid view:

| Parameter | Data Type | Required | Description |
|:---|:---|:---:|:---|
| `ownedByMe` | boolean | ❌ | Filters results to show only casks where the authenticated user owns at least one active, listed vintage. Enforces authentication (Bearer JWT). If `true` and user is unauthenticated, returns `401 Unauthorized`. |
| `buyNowOnly` | boolean | ❌ | Filters results to show only master casks that have at least one active, listed vintage with a buy-now price (`lowestAsk IS NOT NULL`). |
| `classifications` | string (comma-separated slugs) | ❌ | Filters master casks by category classifications (e.g. `single_malt,rye`). |
| `regionIds` | string (comma-separated UUIDs) | ❌ | Filters master casks by region UUIDs (e.g., Speyside, Highlands). |
| `sortBy` | enum | ❌ | Enhanced to support custom sorting options: `lifetimeVolume`, `lowestAsk`, `purchaseAvailability`, `viewCount`, `latestListingDate`, `name`, and `createdAt`. (See Sort Logic details below). |
| `size` | number | ❌ | Specifies the page size. The default value is `24` on desktop and `12` on mobile to match the responsive grid layout structure. |

### 2. Standard / Reference Parameters (Existing)

In addition to the parameters above, the endpoint continues to support existing filter parameters:
- `search` (string): Search by Master Cask name, reference, or distillery name.
- `caskTypeIds` (string, comma-separated UUIDs): Filter by cask types.
- `distilleryIds` (string, comma-separated UUIDs): Filter by distilleries.
- `minVintageYear` / `maxVintageYear` (number): Filter by vintage range.
- `minPrice` / `maxPrice` (number): Filter by effective price point range (inclusive).
- `minAbv` / `maxAbv` (number): Filter by ABV range.
- `minRla` / `maxRla` (number): Filter by RLA range.
- `minOla` / `maxOla` (number): Filter by OLA range.
- `minEstimatedBottleCount` / `maxEstimatedBottleCount` (number): Filter by estimated bottle count range.
- `sortOrder` (enum: `ASC` / `DESC`): Sort direction.
- `page` (number): Current page number (1-based).

---

## Detailed Parameter Behavior & Backend Logic

### 1. Ownership Filter (`ownedByMe`)
- **Authentication**: When `ownedByMe = true`, the request must contain a valid authentication token. If the auth header is missing or invalid, the backend must return `401 Unauthorized`.
- **Query Resolution**: Restricts results to Master Casks where at least one active, listed vintage has `ownerId` equal to the requesting user's ID.
- **Price Display**: If `ownedByMe = true`, price display logic on the returned master cask card (e.g., floor price/lowest ask) must resolve *only* using vintages owned by the user.

### 2. Buy Now Filter (`buyNowOnly`)
- Restricts the list to Master Casks that have at least one active, listed vintage child where `lowestAsk IS NOT NULL`.

### 3. Classification Filter (`classifications`)
- Accepts a comma-separated list of classification slugs (e.g., `single_malt,rye`).
- Resolves to: `masterCask.classification IN (selected_classifications)`.

### 4. Region Filter (`regionIds`)
- Accepts a comma-separated list of region UUIDs (e.g., Speyside, Highlands).
- Resolves to: `masterCask.regionId IN (selected_region_uuids)`.

### 5. Enhanced Sorting (`sortBy`) & Sort Logic
The combination of `sortBy` and `sortOrder` determines the ordering of Master Casks. The backend maps these parameters to the UI dropdown choices and database sorting logic as follows:

| UI Dropdown Option (Reference) | `sortBy` Value | `sortOrder` | Sorting Logic & Business Rules |
|:---|:---|:---|:---|
| Volume: high to low | `lifetimeVolume` | `DESC` | Sorts by the sum of transacted values (`originalAmount` in completed sessions) across all vintages. |
| Volume: low to high | `lifetimeVolume` | `ASC` | Sorts by the sum of transacted values (`originalAmount` in completed sessions) across all vintages. |
| Floor price: high to low | `lowestAsk` | `DESC` | Sorts by the pre-computed `floorPrice` DESC (resolving to `lowestAsk` DESC, falling back to `minReferencePrice` DESC if null). Nulls are sorted last. |
| Floor price: low to high | `lowestAsk` | `ASC` | Sorts by the pre-computed `floorPrice` ASC (resolving to `lowestAsk` ASC, falling back to `minReferencePrice` ASC if null). Nulls are sorted last. |
| Availability: high to low | `purchaseAvailability` | `DESC` | Sorts by the sum of quantities across active ask listings. |
| View: high to low | `viewCount` | `DESC` | Sorts by all-time master cask page views (unidirectional DESC in UI). |
| Listing date: old to new | `latestListingDate` | `ASC` | Sorts by `latestListingDate` ASC (derived from `MAX(vintage.listingDate)` across active vintages). Used for sorting only. |
| Listing date: new to old | `latestListingDate` | `DESC` | Sorts by `latestListingDate` DESC (derived from `MAX(vintage.listingDate)` across active vintages). Used for sorting only. |
| Name: A to Z | `name` | `ASC` | Sorts alphabetically by the Master Cask name field. |
| Name: Z to A | `name` | `DESC` | Sorts alphabetically by the Master Cask name field. |
| Date created: new to old | `createdAt` | `DESC` | Sorts by the Master Cask creation timestamp. |
| Date created: old to new | `createdAt` | `ASC` | Sorts by the Master Cask creation timestamp. |

#### Deterministic Fallbacks & Tie-breakers:
1. **Primary Tie-breaker**: When sorted attributes are equal, secondary sort must be applied: `minReferencePrice ASC`.
2. **Secondary Tie-breaker**: If secondary values are also equal, tertiary sort must be applied: `id ASC` (Master Cask UUID). This ensures pagination remains stable and deterministic.
3. **Null Sorting**: For sorting by `lowestAsk`, records where `lowestAsk` is `null` must always be sorted last, regardless of whether the sorting direction is `ASC` or `DESC`.

---

## Response Schema (Reference)

The endpoint returns `200 OK` with a paginated structure:

```json
{
  "totalItems": 105,
  "totalPages": 6,
  "currentPage": 1,
  "pageSize": 24,
  "items": [
    {
      "id": "master_cask_uuid_12345",
      "name": "Macallan 18 Year Old",
      "imageUrl": "https://cdn.example.com/casks/macallan-18.jpg",
      "distilleryName": "Macallan",
      "caskTypeName": "Ex-Sherry Oak",
      "regionName": "Speyside",
      "classificationLabel": "Single Malt",
      "lowestAsk": 12500,
      "floorPrice": 12500,
      "minReferencePrice": 11000,
      "lowestAskDelta30D": 4.5,
      "purchaseAvailability": 3,
      "minVintageYear": 2005,
      "maxVintageYear": 2008
    }
  ]
}
```

> **Note**: `lowestAskDelta30D` is omitted from the item response if it is `null`.
