# Specification Update: `/api/cask-metadata/sort` (MVP)

This document defines the update requirements for the sort metadata endpoint (`GET /api/cask-metadata/sort`) for the MVP version.

---

## 1. Overview of Changes

* **Primary Change**: Update the **"Ask: lowest first"** sort option:
  * **Previous**: `value: "lowestAsk"` — sorted purely by lowest ask price (`lowestAsk ASC`).
  * **Updated**: `value: "strategicBuys"` — sorted using the **Strategic Buys** two-tier priority logic (Two-Tier Priority Sort).
* **Purpose**: Ensures Master Casks without active Ask listings (`lowestAsk IS NULL`) are still ordered logically below active Master Casks (sorted by minimum reference price `minReferencePrice ASC`), rather than being pushed to the end via undefined null sorting.

---

## 2. Endpoint Update Details

### `GET /api/cask-metadata/sort`

* **Authentication**: Public
* **Query Parameters**: None.

#### Response Payload (`200 OK`)

```json
{
  "sortOptions": [
    {
      "name": "ABV: highest first",
      "value": "abv",
      "defaultOrder": "DESC",
      "description": "Sort by the highest ABV across all variants in a master cask"
    },
    {
      "name": "RLA: highest first",
      "value": "rla",
      "defaultOrder": "DESC",
      "description": "Sort by the highest RLA across all variants in a master cask"
    },
    {
      "name": "Most popular",
      "value": "popularity",
      "defaultOrder": "DESC",
      "description": "Sort by master cask view count"
    },
    {
      "name": "Ask: lowest first",
      "value": "strategicBuys",
      "defaultOrder": "ASC",
      "description": "Sort by Strategic Buys priority logic (Tier 1: active lowest ask ASC, Tier 2: min reference price ASC)"
    },
    {
      "name": "Bid: highest first",
      "value": "highestBid",
      "defaultOrder": "DESC",
      "description": "Sort by the highest active bid across all variants in a master cask"
    }
  ]
}
```

---

## 3. Backend Handling Logic for `sortBy=strategicBuys`

When the user selects the **"Ask: lowest first"** option, the Frontend passes query parameters `sortBy=strategicBuys&sortOrder=ASC` to the cask listing API (`GET /api/casks/listing`).

The Backend applies the **Two-Tier Priority Sort** algorithm as follows:

### Tiered Ranking Strategy:

1. **Tier 1 (Priority 1 — Active Market Entries)**:
   * **Condition**: Master Casks with at least 1 active variant currently having an active Ask listing (`lowestAsk IS NOT NULL`).
   * **Sort Order**: Sorted ascending by lowest ask price (`lowestAsk ASC`).
2. **Tier 2 (Priority 2 — Estimated Market Entries)**:
   * **Condition**: Master Casks with no active Ask listings (`lowestAsk IS NULL`).
   * **Sort Order**: Sorted ascending by minimum reference price (`minReferencePrice ASC`).
3. **Tier Ordering**:
   * All Master Casks in **Tier 1** strictly precede Master Casks in **Tier 2**.

### Tie-breaker Handling (Deterministic Fallback):

When 2 or more Master Casks have identical values within the same Tier:
* **Tie-breaker**: Sorted ascending by Master Cask ID (`id ASC`).
* Ensures stable pagination behavior without duplicate or shifting items across pages.

---

## 4. Impact on Listing API (`GET /api/casks/listing`)

### Parameter Mapping Table for `sortBy` (MVP):

| Option UI Name | `sortBy` Value | `sortOrder` | Backend Sorting Logic |
|:---|:---|:---|:---|
| ABV: highest first | `abv` | `DESC` | `MAX(variant.abv) DESC`, tie-breaker: `id ASC` |
| RLA: highest first | `rla` | `DESC` | `MAX(variant.rla) DESC`, tie-breaker: `id ASC` |
| Most popular | `popularity` | `DESC` | `masterCask.viewCount DESC`, tie-breaker: `id ASC` |
| **Ask: lowest first** | **`strategicBuys`** | **`ASC`** | **Tier 1: `lowestAsk ASC` -> Tier 2: `minReferencePrice ASC`, tie-breaker: `id ASC`** |
| Bid: highest first | `highestBid` | `DESC` | `MAX(variant.highestBid) DESC` (nulls last), tie-breaker: `id ASC` |

---

## 5. Summary of Changes for FE & BE

* **Backend (BE)**:
  1. Update the response payload for `GET /api/cask-metadata/sort`: change the `value` for `"Ask: lowest first"` from `"lowestAsk"` to `"strategicBuys"`.
  2. Implement handling for `sortBy=strategicBuys` in `GET /api/casks/listing` using the two-tier algorithm (Tier 1: `lowestAsk ASC`, Tier 2: `minReferencePrice ASC`, Tie-breaker: `id ASC`).
* **Frontend (FE)**:
  1. Dynamically fetch sort options via `GET /api/cask-metadata/sort`.
  2. When the user selects "Ask: lowest first", automatically pass the returned `value` (`strategicBuys`) to the URL parameter `?sortBy=strategicBuys&sortOrder=ASC`.
