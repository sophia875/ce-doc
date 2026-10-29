# 📄 Cask Filter – Backend Logic & Flows Specs

**Version**: 1.0  
**Last Updated**: 2026-04-09  
**Status**: Draft  

---

## I. Introduction

### 1. Purpose

> Provide a filterable cask browsing experience on the Marketplace page. The system accepts a combination of filter parameters — some targeting master-level attributes (Distillery, Cask Type) and others targeting variant-level attributes (Distillation Year, Price, ABV, RLA, OLA, Bottle Count) — and returns master casks that match ALL specified criteria. A master cask qualifies if it matches master-level filters AND has **at least one active, listed variant** satisfying all variant-level filters.

### 2. Definitions

| Term | Definition |
|------|-----------|
| Master Cask (Parent) | A parent cask entity grouping multiple variants. Holds shared attributes: `name`, `distillery`, `caskType`, `classification`, `region`, `image` |
| Cask Variant (Child) | A specific vintage of a Master Cask. Holds vintage-specific attributes: `vintageYear`, `distillationDate`, `abv`, `rla`, `ola`, `estimatedBottleCount`, `referencePriceMin`, `referencePriceMax`, etc. |
| Master-level filter | A filter applied to attributes that belong to the Master Cask entity (e.g., Distillery, Cask Type) |
| Variant-level filter | A filter applied to attributes that belong to individual Variant entities (e.g., ABV, RLA, OLA, Distillation Year, Price, Bottle Count). A master cask passes if **at least one** of its variants satisfies the condition |
| Active variant | A variant where `caskStatus = 'active'` AND `isListed = true` |
| lowestAsk | For variant: the minimum active ask price. For master: MIN(lowestAsk) across all variants |
| highestBid | For variant: the maximum active bid price. For master: MAX(highestBid) across all variants |
| referencePriceMin | For variant: the minimum reference price. For master: MIN(referencePriceMin) across all variants |
| referencePriceMax | For variant: the maximum reference price. For master: MAX(referencePriceMax) across all variants |
| Effective price range | For a variant, the full set of price points considered for filtering: all individual active ask prices + all individual active bid prices + `referencePriceMin` + `referencePriceMax`. A variant matches the price filter if **any** of these price points falls within [minPrice, maxPrice] |

### 3. Filter Level Matrix

> This matrix defines which entity level each filter targets. This is critical because all variant-level filters use "at least one matching variant" logic.

| Filter | Target Level | Field(s) | Match Logic |
|--------|-------------|----------|-------------|
| Distillery | **Master** | `master.distilleryId` | Master cask's distillery matches any of the selected distillery IDs |
| Cask Type | **Master** | `master.caskTypeId` | Master cask's cask type matches any of the selected cask type IDs |
| Distillation Year | **Variant** | `variant.vintageYear` | Master cask has ≥ 1 active variant where `vintageYear` is within [min, max] |
| Price Range | **Variant** | All active ask prices, all active bid prices, `referencePriceMin`, `referencePriceMax` | Master cask has ≥ 1 active variant where **any** price point (individual ask price, bid price, referencePriceMin, or referencePriceMax) falls within [min, max] |
| ABV | **Variant** | `variant.abv` | Master cask has ≥ 1 active variant where `abv` is within [min, max] |
| RLA | **Variant** | `variant.rla` | Master cask has ≥ 1 active variant where `rla` is within [min, max] |
| OLA | **Variant** | `variant.ola` | Master cask has ≥ 1 active variant where `ola` is within [min, max] |
| Bottle Count | **Variant** | `variant.estimatedBottleCount` | Master cask has ≥ 1 active variant where `estimatedBottleCount` is within [min, max] |

### 4. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| CF-BR-1 | All filters use AND logic between different filter types | If user filters by Distillery AND ABV, master cask must match distillery AND have ≥ 1 variant matching ABV range |
| CF-BR-2 | Checkbox filters use OR logic within the same type | If user selects Distillery A and Distillery B, master cask matches if distillery is A OR B |
| CF-BR-3 | Variant-level filters use "at least one" match logic | A master cask passes a variant-level filter if ANY of its active, listed variants satisfy the condition |
| CF-BR-4 | Only active, listed variants are considered | Variants with `caskStatus != 'active'` or `isListed = false` are excluded from filter evaluation |
| CF-BR-5 | Range filters are inclusive on both ends | `minABV = 40, maxABV = 60` matches variants with `40 ≤ abv ≤ 60` |
| CF-BR-6 | Partial range filters are valid | User can provide only min or only max — the missing bound is unbounded |
| CF-BR-7 | Price filter checks ALL price points on a variant | A variant matches the price filter if ANY of the following falls within [minPrice, maxPrice]: each individual active ask price, each individual active bid price, `referencePriceMin`, `referencePriceMax` |
| CF-BR-8 | Results are paginated | Default page size = 20 |

---

## II. Requirements

### 1. Cask Filter & Search

#### 1.1. Overview

> The cask filter system enables users to narrow down the Marketplace listing by combining master-level filters (Distillery, Cask Type) with variant-level filters (Distillation Year, Price, ABV, RLA, OLA, Bottle Count). All filter types are combined with AND logic. Within checkbox filters, selected values use OR logic. For variant-level filters, a master cask qualifies if at least one of its active variants matches. Results are returned as a paginated list of master cask cards.

#### 1.2. Data Description

##### Filter Parameters (Input)

| No. | Parameter | Data Type | Level | Required | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | distilleryIds | string[] | Master | ❌ | Array of distillery IDs to filter by (OR logic within) |
| 2 | caskTypeIds | string[] | Master | ❌ | Array of cask type IDs to filter by (OR logic within) |
| 3 | minVintageYear | number | Variant | ❌ | Minimum distillation year (inclusive) |
| 4 | maxVintageYear | number | Variant | ❌ | Maximum distillation year (inclusive) |
| 5 | minPrice | number | Variant | ❌ | Minimum price — matches against all active ask/bid prices and reference prices on a variant (inclusive) |
| 6 | maxPrice | number | Variant | ❌ | Maximum price (inclusive) |
| 7 | minABV | number | Variant | ❌ | Minimum ABV percentage (inclusive) |
| 8 | maxABV | number | Variant | ❌ | Maximum ABV percentage (inclusive) |
| 9 | minRLA | number | Variant | ❌ | Minimum Regauged Litres of Alcohol (inclusive) |
| 10 | maxRLA | number | Variant | ❌ | Maximum RLA (inclusive) |
| 11 | minOLA | number | Variant | ❌ | Minimum Original Litres of Alcohol (inclusive) |
| 12 | maxOLA | number | Variant | ❌ | Maximum OLA (inclusive) |
| 13 | minBottles | number | Variant | ❌ | Minimum estimated bottle count (inclusive) |
| 14 | maxBottles | number | Variant | ❌ | Maximum estimated bottle count (inclusive) |
| 15 | page | number | — | ❌ | Page number (default: 1) |
| 16 | pageSize | number | — | ❌ | Items per page (default: 20, max: 50) |
| 17 | sortBy | string | — | ❌ | Sort field: `popularity`, `lowestAsk`, `newest`, `name` (default: `newest`) |
| 18 | sortOrder | string | — | ❌ | `asc` or `desc` (default: `desc`) |

##### Response Fields (Output — per Master Cask)

| No. | Field | Data Type | Source | Description |
| :--- | :--- | :--- | :--- | :--- |
| 1 | id | string | Master | Master Cask ID |
| 2 | name | string | Master | Display name of the master cask |
| 3 | distillery | string | Master | Distillery name |
| 4 | caskType | string | Master | Cask type name |
| 5 | image | string | Master | Master cask product image URL |
| 6 | lowestAsk | number \| null | Computed | MIN(lowestAsk) across all active variants |
| 7 | referencePriceMin | number | Computed | MIN(referencePriceMin) across all active variants |
| 8 | referencePriceMax | number | Computed | MAX(referencePriceMax) across all active variants |
| 9 | variantCount | number | Computed | Count of active, listed variants for this master cask |
| 10 | viewCount | number | Master | Page view counter |

##### Pagination Metadata (Output)

| Field | Data Type | Description |
| :--- | :--- | :--- |
| totalItems | number | Total count of matching master casks |
| totalPages | number | Calculated: ceil(totalItems / pageSize) |
| currentPage | number | Current page number |
| pageSize | number | Items per page |

#### 1.3. Use Cases

##### UC-1. Filter Casks with Combined Criteria

| Item | Description |
|------|-------------|
| **Objective** | Return a paginated list of master casks matching all provided filter criteria |
| **Actor** | Any user (authenticated or anonymous) |
| **Trigger** | FE sends filter request (on filter change, debounced) |
| **Pre-condition(s)** | At least one master cask with an active, listed variant exists in the system |
| **Post-condition(s)** | Returns a paginated list of matching master casks with computed price aggregations |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------
| 1 | Validate filter parameters | CF-1 | Validate all input parameters: range min ≤ max, `minVintageYear` / `maxVintageYear` are valid years, `minABV` / `maxABV` are between 0–100, `pageSize` ≤ 50. If any validation fails, return 400 error with field-specific messages |
| 2 | Apply master-level filters | CF-2 | Start with all master casks. If `distilleryIds` provided: filter to master casks where `distilleryId IN (distilleryIds)`. If `caskTypeIds` provided: filter to master casks where `caskTypeId IN (caskTypeIds)`. These are AND-combined if both are provided |
| 3 | Apply variant-level filters | CF-3 | For each remaining master cask, check its active, listed variants (`caskStatus = 'active'` AND `isListed = true`). A master cask is included if **at least one** of its active variants satisfies ALL of the following (when filter is provided): `vintageYear` within [minVintageYear, maxVintageYear], price match (see CF-4), `abv` within [minABV, maxABV], `rla` within [minRLA, maxRLA], `ola` within [minOLA, maxOLA], `estimatedBottleCount` within [minBottles, maxBottles] |
| 4 | Resolve price for filtering | CF-4 | For price range filter evaluation at the **variant** level, collect ALL price points for the variant: (1) every individual active ask price on this variant, (2) every individual active bid price on this variant, (3) `referencePriceMin`, (4) `referencePriceMax`. The variant matches if **any one** of these price points falls within [minPrice, maxPrice]. This ensures a cask is discoverable whether a user is searching at ask, bid, or reference valuation levels |
| 5 | Compute aggregations per master cask | CF-5 | For each qualifying master cask, compute: `lowestAsk` = MIN(lowestAsk) across all active variants, `referencePriceMin` = MIN(referencePriceMin) across all active variants, `referencePriceMax` = MAX(referencePriceMax) across all active variants, `variantCount` = COUNT of active, listed variants |
| 6 | Apply sorting | CF-6 | Sort results by the `sortBy` parameter: `popularity` → `viewCount` DESC, `lowestAsk` → effective price ASC (lowestAsk with referencePriceMin fallback), `newest` → `createdAt` DESC, `name` → alphabetical. Apply `sortOrder` override if provided |
| 7 | Apply pagination | CF-7 | Slice results by `page` and `pageSize`. Calculate `totalItems`, `totalPages`. Default: page=1, pageSize=20 |
| 8 | Return response | CF-8 | Return paginated master cask list with computed fields and pagination metadata |

##### UC-2. Fetch Filter Options (Distilleries & Cask Types)

| Item | Description |
|------|-------------|
| **Objective** | Return available filter options for checkbox filters so FE can render the filter lists |
| **Actor** | System (triggered by FE on Marketplace page load) |
| **Trigger** | FE requests available filter options |
| **Pre-condition(s)** | Distillery and CaskType reference tables populated |
| **Post-condition(s)** | Returns lists of available distilleries and cask types that have at least one active, listed cask variant |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------
| 1 | Query active distilleries | CF-9 | Return all distilleries that have at least one master cask with an active, listed variant. Include `id`, `name`, and count of associated active master casks. Sort alphabetically by name |
| 2 | Query active cask types | CF-10 | Return all cask types that have at least one master cask with an active, listed variant. Include `id`, `name`, and count of associated active master casks. Sort alphabetically by name |
| 3 | Return filter options | CF-11 | Return both lists in a single response to minimize network round-trips |

#### 1.4. Edge Cases

| # | Trigger | Flow | Side Effects |
|---|---------|------|-------------|
| 1 | No filters applied | Return all master casks with active, listed variants, paginated | Default sorted by `newest` |
| 2 | All variant-level filters provided, but no master cask has a single variant matching ALL of them simultaneously | Return empty result `{ items: [], totalItems: 0 }` | FE shows empty state |
| 3 | A variant satisfies ABV filter but NOT price filter — only other variant satisfies price but NOT ABV | Master cask is **excluded**. The same single variant must satisfy all variant-level filters | — |
| 4 | User provides minPrice only, no maxPrice | Filter price ≥ minPrice with no upper bound | — |
| 5 | User provides maxPrice only, no minPrice | Filter price ≤ maxPrice with no lower bound | — |
| 6 | Variant has no active asks, no active bids, and `referencePriceMin = null` and `referencePriceMax = null` | Variant has zero price points — excluded from price filter evaluation. If all variants lack any pricing data, master cask is excluded when price filter is active | — |
| 7 | URL query params contain invalid values (e.g., non-numeric minPrice) | Return 400 with validation error | FE displays validation message |
| 8 | pageSize exceeds maximum (50) | Cap at 50 silently | — |

---

### 2. Master Cask Card Display

#### 2.1. Overview

> Each result in the Marketplace listing is rendered as a Master Cask Card. This section defines the data fields displayed on each card and the price display logic that determines which price label and value to show.

#### 2.2. Data Description (Data Displayed on Master Cask Card)

| No. | Field | Data Type | Source | Description |
| :--- | :--- | :--- | :--- | :--- |
| 1 | name | string | Master | Display name of the master cask (e.g., "Aberlour Ex-Sherry Hogshead") |
| 2 | distillery | string | Master | Name of the distillery |
| 3 | image | string (URL) | Master | Master cask product image URL |
| 4 | lowestAsk | number \| null | Computed (Children) | MIN(lowestAsk) across all active variants. `null` if no variant has an active ask |
| 5 | referencePriceMin | number | Computed (Children) | MIN(referencePriceMin) across all active variants |

#### 2.3. Use Cases

##### UC-3. Resolve Master Cask Card Display Data

| Item | Description |
|------|-------------|
| **Objective** | Provide the display data for a single master cask card in the Marketplace listing |
| **Actor** | System (triggered per master cask in response) |
| **Trigger** | After filter/search query returns qualifying master casks (UC-1) |
| **Pre-condition(s)** | Master cask has at least one active, listed variant |
| **Post-condition(s)** | Card data object is populated with parent fields and computed price fields |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------
| 1 | Resolve parent fields | CF-12 | Read `name`, `distillery`, `image` directly from the MasterCask entity |
| 2 | Compute lowestAsk | CF-13 | Query all active variants of this master cask. For each variant, get its `lowestAsk` (minimum active ask price). Compute `MIN(lowestAsk)` across all variants. If no variant has an active ask, return `null` |
| 3 | Compute referencePriceMin | CF-14 | Compute `MIN(referencePriceMin)` across all active variants of this master cask |
| 4 | Determine price label and value | CF-15 | **IF** `lowestAsk` is not null: label = `"Lowest ask"`, value = `£{lowestAsk}`. **ELSE**: label = `"Expected value"`, value = `"From £{referencePriceMin}"` |

#### 2.4. Edge Cases

| # | Trigger | Flow | Side Effects |
|---|---------|------|-------------|
| 1 | Master cask has no active asks across any variant | Display "Expected value" with `referencePriceMin` | — |
| 2 | Master cask image is null | FE shows a fallback barrel placeholder image | — |
| 3 | Master cask name exceeds 2 lines | FE truncates with ellipsis after line 2 | — |
| 4 | `lowestAsk` = 0 or negative | Do not display price. Show "Contact us" or hide price field | — |
| 5 | `referencePriceMin` is null and `lowestAsk` is null | Hide price section entirely on the card | — |

---

## III. Data Model

> No new entities required. This feature uses existing entities:
> - **MasterCask** — see `docs/data-models/parent-child-structure.md`
> - **CaskVariant** — see `docs/data-models/parent-child-structure.md`
> - **Distillery** — reference table
> - **CaskType** — reference table

### Recommended Indexes

| Table | Index Fields | Purpose |
|-------|-------------|---------|
| CaskVariant | `(masterCaskId, caskStatus, isListed)` | Fast lookup of active variants per master cask |
| CaskVariant | `(vintageYear)` | Distillation year range filter |
| CaskVariant | `(abv)` | ABV range filter |
| CaskVariant | `(rla)` | RLA range filter |
| CaskVariant | `(ola)` | OLA range filter |
| CaskVariant | `(estimatedBottleCount)` | Bottle count range filter |
| MasterCask | `(distilleryId)` | Distillery filter |
| MasterCask | `(caskTypeId)` | Cask type filter |

---

## IV. Notifications / Side Effects

| # | Trigger Event | Channel | Recipient | Description |
|---|--------------|---------|-----------|-------------|
| — | N/A | — | — | No notifications. This is a read-only filter/search feature with no side effects |

---

## V. Permissions & Access Control

| Action / Flow | Anonymous | Buyer | Seller | Admin |
|---------------|-----------|-------|--------|-------|
| Filter/Search Casks | ✅ | ✅ | ✅ | ✅ |
| Fetch Filter Options | ✅ | ✅ | ✅ | ✅ |

---

## VI. Performance Considerations

| # | Concern | Recommendation |
|---|---------|---------------|
| 1 | Variant-level filters require checking across all variants per master cask | Use SQL `EXISTS` subquery instead of JOINing all variants. Only check for existence of at least one matching variant |
| 2 | Multiple range filters on variant fields | Consider a composite query approach: single subquery with all variant-level conditions combined |
| 3 | Distillery and Cask Type option lists | Cache filter options. Refresh every 5–15 minutes or on cask create/update events |
| 4 | Pagination with complex filters | Use `COUNT(*) OVER()` for total count to avoid a separate counting query |

---

_End of Document_
