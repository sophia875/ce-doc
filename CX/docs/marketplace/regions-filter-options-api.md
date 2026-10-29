# Regions Filter Options API

This document defines the requirements for the new regions filter metadata endpoint that supports dynamic region checkbox lists in the Marketplace sidebar.

## Endpoint Overview

### GET `/api/cask-masters/filter-options/regions`

Returns a list of regions (e.g. Speyside, Highlands) along with the count of active master casks in each region, dynamically adjusted by active filter criteria.

#### Query Parameters (Inputs)

| Parameter | Data Type | Required | Description |
|:---|:---|:---:|:---|
| `distilleryIds` | string (comma-separated UUIDs) | ❌ | Filter counts/options by distillery. |
| `caskTypeIds` | string (comma-separated UUIDs) | ❌ | Filter counts/options by cask type. |
| `classifications` | string (comma-separated slugs) | ❌ | Filter counts/options by classification. |

#### Response Format (Outputs)

Returns `200 OK` with an array of region options:

```json
[
  {
    "id": "region_a2e3fc09-edda-46a9-a701-71a72c65b86c",
    "name": "Speyside",
    "activeCount": 13
  },
  {
    "id": "region_b1234c09-edda-46a9-a701-71a72c65b99f",
    "name": "Highlands",
    "activeCount": 3
  }
]
```

---

## Backend Notes: Dependent Filtering

This metadata endpoint must support **dependent (context-aware) filtering**. The returned list and active options/counts must dynamically reflect the currently selected filter context passed in the query parameters.

### Rules & Examples

1.  **Cask Type-to-Region Mapping**:
    *   *Trigger*: A user selects **Ex-Sherry Hogshead** (`caskTypeIds=cask_type_sherry`) in the UI.
    *   *Flow*: The frontend calls `GET /api/cask-masters/filter-options/regions?caskTypeIds=cask_type_sherry`.
    *   *Result*: The endpoint queries only master casks matching the cask type "Ex-Sherry Hogshead" and returns only the relevant regions matching those active casks.
2.  **Count Recalculation**:
    *   For `GET /api/cask-masters/filter-options/regions`, the count (`activeCount`) represents the number of unique **Master Casks** (not vintages) that:
        *   Have `masterCask.status = 'active'`
        *   Have at least one child vintage with `caskStatus = 'active' AND isListed = true`
        *   Satisfy the query parameters passed to the endpoint.
    *   *Result Adjustments*: When filter criteria (such as `classifications` or `distilleryIds`) are provided, the `activeCount` for each region is dynamically recalculated to only count active casks matching those filters. Regions with `activeCount = 0` under the selected criteria are excluded from the response.
3.  **Performance & Caching**:
    *   Since queries are dynamic and depend on multiple combinations of query parameters, results should be computed using fast lookup tables or a pre-aggregated view, caching results dynamically based on parameter hashes.
