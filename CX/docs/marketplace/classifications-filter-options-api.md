# Cask Classifications Filter Options API

This document defines the requirements for the updated category classifications metadata endpoint that supports dynamic category checkbox lists in the Marketplace sidebar.

## Endpoint Overview

### GET `/api/cask-metadata/classifications`

Returns the hierarchically structured list of category classifications (2-level hierarchy: classification and styles), dynamically filtered by distillery, cask type, or region criteria if active master casks match.

#### Query Parameters (Inputs)

| Parameter | Data Type | Required | Description |
|:---|:---|:---:|:---|
| `distilleryIds` | string (comma-separated UUIDs) | ❌ | Filter classifications by distillery. |
| `caskTypeIds` | string (comma-separated UUIDs) | ❌ | Filter classifications by cask type. |
| `regionIds` | string (comma-separated UUIDs) | ❌ | Filter classifications by region. |

#### Response Format (Outputs)

Returns `200 OK` with an array of classifications in a 2-level hierarchical structure:

```json
[
  {
    "classification": "single_malt",
    "classificationLabel": "Single Malt",
    "imageUrl": "https://...",
    "activeCount": 13,
    "styles": [
      {
        "value": "single_malt_unpeated",
        "label": "Unpeated"
      },
      {
        "value": "single_malt_lightly_peated",
        "label": "Lightly Peated"
      },
      {
        "value": "single_malt_peated",
        "label": "Peated"
      },
      {
        "value": "single_malt_heavily_peated",
        "label": "Heavily Peated / Highly"
      }
    ]
  }
]
```

#### Marketplace Filtering Rule
*   **Level 1 Extraction Only**: When used as a filter in the marketplace sidebar, only the Level 1 classification values (i.e., the parent `classification` field, e.g., `single_malt`) are extracted and displayed/applied as active filters. The Level 2 `styles` are not used as filter options for the marketplace grid view itself.

---

## Backend Notes: Dependent Filtering

This metadata endpoint must support **dependent (context-aware) filtering**. The returned list and active options/counts must dynamically reflect the currently selected filter context passed in the query parameters.

### Rules & Examples

1.  **Distillery-to-Category Mapping**:
    *   *Trigger*: A user selects **Distillery A** (`distilleryIds=distillery_A`) in the UI.
    *   *Flow*: The frontend calls `GET /api/cask-metadata/classifications?distilleryIds=distillery_A`.
    *   *Result*: The backend queries only active master casks belonging to Distillery A, and only classifications (with their styles) associated with Distillery A's active casks are returned. Any classifications with no matching active casks under this distillery are excluded from the response.
2.  **Cask Type-to-Category Mapping**:
    *   *Trigger*: A user selects **Ex-Sherry Hogshead** (`caskTypeIds=cask_type_sherry`) in the UI.
    *   *Flow*: The frontend calls `GET /api/cask-metadata/classifications?caskTypeIds=cask_type_sherry`.
    *   *Result*: The endpoint queries only master casks matching the cask type "Ex-Sherry Hogshead" and returns only classifications matching those active casks.
3.  **Count Recalculation**:
    *   For the Level 1 category objects in `/api/cask-metadata/classifications`, the count (`activeCount`) represents the number of unique **Master Casks** (not vintages) that:
        *   Have `masterCask.status = 'active'`
        *   Have at least one child vintage with `caskStatus = 'active' AND isListed = true`
        *   Satisfy the query parameters passed to the endpoint.
    *   *Result Adjustments*: When filter criteria (such as `regionIds`) are provided, the `activeCount` for each classification is dynamically recalculated to only count active casks matching those filters. Classifications with `activeCount = 0` under the selected criteria are excluded from the response.
4.  **Performance & Caching**:
    *   Since queries are dynamic and depend on multiple combinations of query parameters, results should be computed using fast lookup tables or a pre-aggregated view, caching results dynamically based on parameter hashes.
