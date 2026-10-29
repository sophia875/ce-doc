# BACKEND LOGIC & FLOW SPECS

Marketplace Listing

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | Sophia | 2026-06-16 |
| 0.2 | Fix default page size; add sort label mapping table; add NULL sort handling for `lowestAsk`; add `lowestAskDelta30D` null cross-reference; update `isWatched` auth note; add cache lag documentation; clarify `latestListingDate` as sort-only | Sophia | 2026-06-16 |
| 0.3 | Rename title to Marketplace Listing; remove watchlist features and `isWatched` output field; add Data Model and Notifications sections; renumber use cases. | Antigravity | 2026-06-16 |
| 0.4 | Split filter options API specs into separate classifications and regions files. | Antigravity | 2026-06-18 |
| 0.5 | Add sort options update reference link. | Antigravity | 2026-06-18 |
| 0.6 | Split listing endpoint query parameter updates into separate spec file. | Antigravity | 2026-06-19 |

---

## I. Introduction

### 1. Purpose

The Marketplace backend system manages search, filter, sorting, and pagination for spirits and whisky casks. The system supports multi-layered queries on both parent level (Master Cask) and child level (Cask Vintage) fields. A Master Cask is returned in the list if it is active and has at least one active, listed vintage matching all vintage-level criteria.

### 2. Definitions

> [!NOTE]
> **Data Mapping Rule**: Conceptual terms like **Cask Vintage** (or **vintage**) are used in these requirements to represent the child entities. These correspond directly to the database `CaskVariant` (or `variant`) tables and the `/api/casks` endpoints in the physical production backend implementation.

| Term | Definition |
|------|------------|
| Master Cask (Parent) | A parent cask entity representing a product line. Groups multiple vintages under one name. Holds shared attributes: `name`, `distilleryId`, `caskTypeId`, `classification` (slug), `regionId`, `imageUrl`, and `viewCount`. |
| Cask Vintage (Child) | A physical cask vintage of a Master Cask. Holds vintage-specific data: `vintageYear`, `referencePriceMin`, `referencePriceMax`, `abv`, `rla`, `ola`, `estimatedBottleCount`, `lowestAsk`, `highestBid`, `caskStatus`, `isListed`, and `ownerId`. |
| Active Cask Vintage | A vintage where `caskStatus = 'active'` AND `isListed = true`. |
| Active Master Cask | A master cask where `masterCask.status = 'active'` AND has at least one active, listed vintage child. |

---

### 3. Filter Level Matrix

This matrix defines which entity level each filter targets. This is critical because all vintage-level filters use "at least one matching vintage" logic.

| Filter | Target Level | Field(s) | Match Logic |
|:---|:---|:---|:---|
| Distillery | Master | `master.distilleryId` | Master cask's distillery matches any of the selected distillery IDs. |
| Cask Type | Master | `master.caskTypeId` | Master cask's cask type matches any of the selected cask type IDs. |
| Region | Master | `master.regionId` | Master cask's region matches any of the selected region IDs. |
| Category (Classification) | Master | `master.classification` | Master cask's classification matches any of the selected classifications. |
| Distillation Year | Vintage | `vintage.vintageYear` | Master cask has ≥ 1 active vintage where `vintageYear` is within [min, max]. |
| Price Range | Vintage | All active ask prices, all active bid prices, `referencePriceMin`, `referencePriceMax` | Master cask has ≥ 1 active vintage where any price point falls within [min, max]. |
| ABV | Vintage | `vintage.abv` | Master cask has ≥ 1 active vintage where `abv` is within [min, max]. |
| RLA | Vintage | `vintage.rla` | Master cask has ≥ 1 active vintage where `rla` is within [min, max]. |
| OLA | Vintage | `vintage.ola` | Master cask has ≥ 1 active vintage where `ola` is within [min, max]. |
| Bottle Count | Vintage | `vintage.estimatedBottleCount` | Master cask has ≥ 1 active vintage where `estimatedBottleCount` is within [min, max]. |

---

### 4. Business Rules Summary

| Rule Code | Description | Condition / Example |
|-----------|-------------|---------------------|
| MP-BR-1 | All filters use AND logic between different filter types | If user filters by Distillery AND ABV, master cask must match distillery AND have ≥ 1 vintage matching ABV range. |
| MP-BR-2 | Checkbox filters use OR logic within the same type | If user selects Distillery A and Distillery B, master cask matches if distillery is A OR B. |
| MP-BR-3 | Vintage-level filters use "at least one" match logic | A master cask passes a vintage-level filter if ANY of its active vintages satisfy the condition. |
| MP-BR-4 | Only active, listed vintages are considered | Vintages with `caskStatus != 'active'` or `isListed = false` are excluded from filter evaluation. |
| MP-BR-5 | Range filters are inclusive on both ends | `minABV = 40, maxABV = 60` matches vintages with `40 ≤ abv ≤ 60`. |
| MP-BR-6 | Partial range filters are valid | User can provide only min or only max, the missing bound is unbounded. |
| MP-BR-7 | Price filter checks ALL price points on a vintage | A vintage matches the price filter if ANY of the following falls within [minPrice, maxPrice]: each individual active ask price, each individual active bid price, `referencePriceMin`, `referencePriceMax`. |
| MP-BR-8 | Results are paginated | Default page size: 24 on desktop, 12 on mobile. |
| MP-BR-9 | Owned By You Filter | If `ownedByMe = true`, filters to master casks where at least one vintage has `ownerId = currentUser.id` (requires authentication). |
| MP-BR-10 | Buy Now Only Filter | If `buyNowOnly = true`, filters to master casks where at least one active vintage has `lowestAsk != null`. |
| MP-BR-11 | Deterministic Tie-breakers | If sorting values are equal, the system applies a deterministic fallback: secondary sort is `minReferencePrice ASC`, and tertiary sort is `id ASC`. When sorting by `lowestAsk`, records where `lowestAsk` is `null` are always sorted last, regardless of `sortOrder`. |

---

### 5. API Reference Documents

Detailed requirements for the API changes, renames, and dynamic filters are split into dedicated files:

1.  **Endpoint Renames**: [api-change-requests.md](file:///d:/CX/docs/marketplace/api-change-requests.md) maps variant-centric endpoints to parent-centric (`/cask-masters`) paths.
2.  **Classifications Filter Options**: [classifications-filter-options-api.md](file:///d:/CX/docs/marketplace/classifications-filter-options-api.md) defines classifications query endpoint and context-aware dependent filtering business logic.
3.  **Regions Filter Options**: [regions-filter-options-api.md](file:///d:/CX/docs/marketplace/regions-filter-options-api.md) defines regions filter options query endpoint and context-aware dependent filtering business logic.
4.  **Sort Options Update**: [sort-options-update-api.md](file:///d:/CX/docs/marketplace/sort-options-update-api.md) defines the updated sort options endpoint.
5.  **Listing Parameters Update**: [listing-parameters-update-api.md](file:///d:/CX/docs/marketplace/listing-parameters-update-api.md) defines query parameters, authentication, and sorting rules for the marketplace listing.
6.  **Floor Price API**: [floor-price-api.md](file:///d:/CX/docs/marketplace/floor-price-api.md) defines the backend rules and response schema for the new `floorPrice` field.

---

## II. Requirements

### 1. Marketplace Filter & Search List

#### 1.1. Overview
Returns a paginated, sorted, and filtered list of active Master Casks. The query parameters filter at both master and vintage levels. Dynamic aggregations are pre-computed for performance.

#### 1.2. Data Description (Query Parameters & Response Fields)

##### Inputs (Query Parameters)

**Existing / Reference Parameters:**  
Same as standard filter parameters from `GET /api/casks/listing` (specifically: `search`, `caskTypeIds`, `distilleryIds`, `minVintageYear` / `maxVintageYear`, `minPrice` / `maxPrice`, `minAbv` / `maxAbv`, `minRla` / `maxRla`, `minOla` / `maxOla`, `minEstimatedBottleCount` / `maxEstimatedBottleCount`, `sortOrder`, and `page`).

**New / Enhanced Parameters (To Be Created):**  

Please refer to the dedicated [listing-parameters-update-api.md](file:///d:/CX/docs/marketplace/listing-parameters-update-api.md) for full details on the newly added/enhanced parameters (`ownedByMe`, `buyNowOnly`, `classifications`, `regionIds`, `sortBy`, and `size`) and detailed sort mapping rules.

##### Outputs (Master Cask Card Object Fields)

**Existing / Reference Fields:**  
Same as active master cask listing/details response fields (specifically: `id`, `name`, `imageUrl`, `distilleryName`, `caskTypeName`, `regionName`, `categoryLabel` / `classificationLabel`, `lowestAsk`, `minReferencePrice`, `lowestAskDelta30D` [field is omitted when null], `purchaseAvailability`, `minVintageYear`, and `maxVintageYear`).

**New / Enhanced Fields (To Be Created):**  
- `floorPrice` (number): The resolved floor price for the cask master (resolves to `lowestAsk` if not null; otherwise falls back to `minReferencePrice`).

> **Note:** `latestListingDate` is used for sorting only and is not returned in the response payload.

---

#### 1.3. Use Cases

##### UC-1. Fetch Marketplace Cask List

| Item | Description |
|------|-------------|
| **Objective** | Fetch a paginated list of filtered and sorted master casks, resolving floor price labels. |
| **Actor** | System (triggered by Frontend) |
| **Trigger** | Frontend sends list request on page load or filter changes. |
| **Pre-condition(s)** | Database has active master casks and vintages. |
| **Post-condition(s)** | Returns paginated payload with computed prices and growth metrics. |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Validate filters and page params | MP-UC1-1 | Enforce range boundaries (min ≤ max). If `ownedByMe = true` and auth token is missing, throw `401 Unauthorized`. |
| 2 | Filter master-level fields | MP-UC1-2 | Filter MasterCask table: status must be `'active'`. Match search query against name or distillery. Filter by `distilleryId`, `caskTypeId`, `regionId`, and `classification` (if parameters provided). |
| 3 | Filter vintage-level fields | MP-UC1-3 | Check active vintages (`caskStatus = 'active' AND isListed = true`). Match against vintage, ABV, RLA, OLA, and bottle count ranges. Match price using the effective price points definition (MP-BR-7). |
| 4 | Apply Ownership & Buy Now filters | MP-UC1-4 | If `ownedByMe = true`, restrict vintages to `ownerId = currentUserId`. If `buyNowOnly = true`, restrict to vintages where `lowestAsk IS NOT NULL`. |
| 5 | Sort results | MP-UC1-5 | Sort using requested `sortBy` parameter: `lifetimeVolume` (completed transaction amount), `lowestAsk` (resolved min ask), `purchaseAvailability` (active ask quantity), `viewCount` (master page views), `latestListingDate` (max listing date), `name` (alphabetical). Apply `sortOrder`. See Sort Label Mapping table (Section I.5). |
| 6 | Deterministic fallback | MP-UC1-6 | When sorted attributes are equal, apply secondary sort `minReferencePrice ASC` followed by `id ASC` to prevent random records from swapping positions during paging. When sorting by `lowestAsk`, null values are always sorted last regardless of `sortOrder`. |
| 7 | Return paginated response | MP-UC1-7 | Paginate matching parents using standard offsets. Return active list and pagination headers (`totalItems`, `totalPages`, `currentPage`, `pageSize`). |

**Happy Path**:
- Valid filters and sort parameters return 200 OK with the array of mapped cask card data and pagination headers.

**Negative Path**:
- Invalid format parameter (e.g. `minPrice=abc`) → Returns 400 Bad Request.
- `ownedByMe = true` requested by unauthenticated user → Returns 401 Unauthorized.

---

##### UC-2. Fetch Marketplace Filter Options

| Item | Description |
|------|-------------|
| **Objective** | Retrieve the dynamic list of classifications, regions, distilleries, and cask types along with the number of associated active parent casks. |
| **Actor** | System (triggered by Frontend) |
| **Trigger** | Page loads for the first time. |
| **Pre-condition(s)** | Metadata tables populated. |
| **Post-condition(s)** | Returns arrays of available options with counts. |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Fetch category classifications | MP-UC2-1 | Fetch the hierarchical classification categories using `GET /api/cask-metadata/classifications`. The frontend extracts Level 1 classifications (`classification`, `classificationLabel`, and `activeCount`) to display as filter options with counts. The classifications and counts dynamically adjust based on active filters (e.g. `regionIds`). Categories with `activeCount = 0` under the selected criteria are excluded. |
| 2 | Fetch cask type counts | MP-UC2-2 | Query counts of active, listed master casks grouped by `caskTypeId`. Returns ID, name, and count. |
| 3 | Fetch region counts | MP-UC2-3 | Fetch the region list using `GET /api/regions`. The frontend receives the full region details along with the dynamic `activeCount`. The regions and counts dynamically adjust based on active filters (e.g. `distilleryIds`, `caskTypeIds`, `classifications`). Regions with `activeCount = 0` under the selected criteria are excluded. |
| 4 | Fetch distillery counts | MP-UC2-4 | Query counts of active, listed master casks grouped by `distilleryId`. Returns ID, name, and count. |
| 5 | Return filter options | MP-UC2-5 | Return the respective filter lists to the frontend. Depending on client implementation, these may be queried individually. |

---

#### 1.4. Feature-level Edge Cases

| # | Scenario | Related UC(s) | Expected Behavior | Side Effects |
|---|----------|---------------|-------------------|-------------|
| 1 | Vintage price drops or is sold out while user is viewing the page. | UC-1 | The list reflects changes on next refresh. If a user clicks "Buy Now" on a sold-out item, the Checkout API will return 400/409 (out of stock) and reject the checkout session. | Prevents duplicate purchases of the same physical cask. |
| 2 | Caching of filter options and counts. | UC-2 | Filter options (including classifications) and counts are cached for 5 minutes for performance. Option availability and counts may lag behind real-time data by up to 5 minutes after cask status changes (e.g. a newly activated cask may not appear in category options immediately). No manual cache invalidation is implemented in the current scope — cache expires naturally. |
| 3 | Multiple vintages match the same query, but the user owns only one. | UC-1 | If `ownedByMe = true`, the master cask is shown, but standard price display logic resolves only using vintages owned by the user. | Keeps ownership lists clean and accurate. |

---

## III. Data Model

### Entity: MasterCask (Parent) — existing, reference

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | UUID | ✅ | auto-generated | Primary key |
| `status` | string | ✅ | `'active'` | Lifecycle status: `'active'` or `'inactive'`. If `'inactive'`, the master cask is excluded from search/listing. |
| `name` | string | ✅ | — | Cask display name |
| `distilleryId` | UUID (FK) | ✅ | — | FK to Distillery |
| `caskTypeId` | UUID (FK) | ✅ | — | FK to CaskType |
| `classification` | string | ✅ | — | Category classification slug (e.g., `single_malt`) |
| `regionId` | UUID (FK) | ✅ | — | FK to Region |
| `imageUrl` | string \| null | ❌ | null | Master cask primary image |
| `viewCount` | number | ✅ | `0` | All-time page view counter. |
| `lowestAsk` | number \| null | ❌ | null | Pre-computed minimum active ask price across active child vintages. |
| `floorPrice` | number | ✅ | — | Pre-computed floor price (resolves to `lowestAsk` if not null, otherwise `minReferencePrice`). |
| `minReferencePrice` | number | ✅ | — | Pre-computed minimum reference price across child vintages. |
| `maxReferencePrice` | number | ✅ | — | Pre-computed maximum reference price across child vintages. |
| `createdAt` | datetime | ✅ | `NOW()` | Record creation timestamp |
| `updatedAt` | datetime | ✅ | `NOW()` | Last update timestamp |

### Entity: CaskVariant (Child) — existing, reference

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | UUID | ✅ | auto-generated | Primary key |
| `masterCaskId` | UUID (FK) | ✅ | — | FK to MasterCask |
| `vintageYear` | number | ✅ | — | Year of vintage |
| `referencePriceMin` | number | ✅ | — | Minimum reference price |
| `referencePriceMax` | number | ✅ | — | Maximum reference price |
| `lowestAsk` | number \| null | ❌ | null | Current minimum active ask price |
| `highestBid` | number \| null | ❌ | null | Current maximum active bid price |
| `caskStatus` | enum | ✅ | — | Status of the physical cask vintage: `'active'`, `'sold'`, `'inactive'` |
| `isListed` | boolean | ✅ | `false` | Whether the vintage is listed. Used with `caskStatus = 'active'` for eligibility. |
| `listingDate` | datetime | ✅ | — | Date this specific variant/vintage was listed on the marketplace |
| `ownerId` | UUID \| null | ❌ | null | Owner user ID (used for `ownedByMe` query resolution) |
| `createdAt` | datetime | ✅ | `NOW()` | Record creation timestamp |
| `updatedAt` | datetime | ✅ | `NOW()` | Last update timestamp |

### Derived / Pre-computed: MasterCaskMarketStats — existing, reference

| Field | Type | Description |
|-------|------|-------------|
| `masterCaskId` | UUID | FK to MasterCask |
| `lifetimeVolume` | number | SUM of completed session amounts across all variants/vintages |
| `lowestAskPrev30D` | number \| null | The lowest ask 30 days ago (benchmark price) |
| `lowestAskDelta30D` | number \| null | Percent change in lowest ask over the last 30 days |
| `askCount` | number | Number of active ask orders across all variants/vintages |
| `purchaseAvailability` | number | SUM of active ask quantities (available casks) across all active variants/vintages |
| `minVintageYear` | number \| null | MIN(vintageYear) for active variants/vintages |
| `maxVintageYear` | number \| null | MAX(vintageYear) for active variants/vintages |
| `latestListingDate` | datetime \| null | MAX(listingDate) for active variants/vintages |
| `lastRefreshedAt` | datetime | Last time stats were recomputed |

### Status Flow / State Machine

```
   [active] ──(Cask vintage sold)──> [sold]
      │
(Admin deactivates)
      ▼
  [inactive]
```

---

## IV. Notifications / Side Effects

| # | Trigger Event | Channel | Template | Recipient | Description |
|---|--------------|---------|----------|-----------|-------------|
| 1 | Cask view count increments | DB Update | N/A | System / DB | Increments `viewCount` on `MasterCask` dynamically when user views details. |
| 2 | Completed checkout session | DB Update | N/A | System / DB | Triggers update of `lifetimeVolume` in `MasterCaskMarketStats`. |
| 3 | Background job scheduler | DB Update | N/A | System / DB | Recalculates metrics (e.g. `lowestAskDelta30D`, `purchaseAvailability`, etc.) in `MasterCaskMarketStats` every 5-15 mins. |

> **Note on Notifications**: No emails, SMS, or push notifications are directly triggered by the read-only search and filter listing APIs. All side effects are confined to database updates, cache refresh, and analytical pre-computations.

_End of Document 2_