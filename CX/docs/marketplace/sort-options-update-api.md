# Sort Options Update API

This document defines the requirements for the updated sort metadata endpoint that lists available sorting configurations for the Marketplace grid view.

## Endpoint Overview

### GET `/api/cask-metadata/sort`

Returns the list of available sort options, their internal `value` names, the default sorting directions (`defaultOrder`), and user-friendly descriptions.

#### Query Parameters (Inputs)
None.

#### Response Format (Outputs)

Returns `200 OK` with an object containing the array of updated sort options:

```json
{
  "sortOptions": [
    {
      "name": "Volume: high to low",
      "value": "lifetimeVolume",
      "defaultOrder": "DESC",
      "description": "Sorts by the sum of transacted values across all vintages"
    },
    {
      "name": "Volume: low to high",
      "value": "lifetimeVolume",
      "defaultOrder": "ASC",
      "description": "Sorts by the sum of transacted values across all vintages"
    },
    {
      "name": "Ask: lowest first",
      "value": "strategicBuys",
      "defaultOrder": "ASC",
      "description": "Sorts by Strategic Buys priority logic (Tier 1: active lowest ask ASC, Tier 2: min reference price ASC)"
    },
    {
      "name": "Availability: high to low",
      "value": "purchaseAvailability",
      "defaultOrder": "DESC",
      "description": "Sorts by the sum of quantities across active ask listings"
    },
    {
      "name": "View: high to low",
      "value": "viewCount",
      "defaultOrder": "DESC",
      "description": "Sorts by all-time master cask page views"
    },
    {
      "name": "Listing date: old to new",
      "value": "latestListingDate",
      "defaultOrder": "ASC",
      "description": "Sorts by listing date of the latest active vintage"
    },
    {
      "name": "Listing date: new to old",
      "value": "latestListingDate",
      "defaultOrder": "DESC",
      "description": "Sorts by listing date of the latest active vintage"
    },
    {
      "name": "Name: A to Z",
      "value": "name",
      "defaultOrder": "ASC",
      "description": "Sorts alphabetically by the Master Cask name"
    },
    {
      "name": "Name: Z to A",
      "value": "name",
      "defaultOrder": "DESC",
      "description": "Sorts alphabetically by the Master Cask name"
    }
  ]
}
```

---

## Mapping Details
For more information on the exact sorting logic, deterministic fallbacks, and backend tie-breakers associated with each options key, please refer to the sorting logic details in [Marketplace Listing Query Parameters Update API](file:///d:/CX/docs/marketplace/listing-parameters-update-api.md).
