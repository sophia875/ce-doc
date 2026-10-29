# Floor Price API Specification

This document defines the requirements for adding the `floorPrice` field to the Cask Master (parent) entity responses. This field provides a resolved price point for displaying in grid cards and search results.

---

## Field Definition

| Field Name | Data Type | Logic / Computation | Description |
|:---|:---|:---|:---|
| `floorPrice` | number | `lowestAsk !== null ? lowestAsk : minReferencePrice` | The lowest active ask price if available; otherwise falls back to the minimum reference price of the cask. |

---

## Affected Endpoints

The `floorPrice` field must be added to all endpoints returning the Master Cask (parent) object, specifically:

- `GET /api/cask-masters/{id}` (Get detail of a single master cask)
- `GET /api/cask-masters/listing` (Get paginated master casks with filtering/sorting)
- `GET /api/cask-masters/search` (Search master casks)
- `GET /api/cask-masters/recently-viewed` (Get recently viewed master casks)
- `GET /api/cask-masters/{id}/similar` (Get similar master casks)

---

## Response Schema Example

### Example 1: Floor Price falls back to Reference Price (`lowestAsk` is null)

If `lowestAsk` is `null`, `floorPrice` resolves to `minReferencePrice` (`7214` in the sample below):

```json
{
  "id": "cask_master_051d845b-f828-4c12-8da0-f06ec40fddca",
  "name": "Dalmore Ex-Sherry Hogshead",
  "imageUrl": "https://caskx-exchange-static-assets.s3.us-east-1.amazonaws.com/cask-masters/cask_master_051d845b-f828-4c12-8da0-f06ec40fddca/Dalmore Ex-Sherry Hogshead.webp",
  "distilleryId": "distillery_9c9d3817-f02a-4aa8-bb48-52b2ea6b44c1",
  "distillery": {
    "id": "distillery_9c9d3817-f02a-4aa8-bb48-52b2ea6b44c1",
    "name": "Bunnahabhain",
    "summary": "Unique unpeated Islay producer with elegant, maritime character",
    "description": "Founded in 1881 on northeast coast of Islay. Owned by Distell/Heineken. Known for UNPEATED Islay style (rare). Water from Margadale River. Produces elegant, coastal, briny character. Also produces 'Staoisha' - heavily peated expression for independent bottlers (named after Loch Staoisha, 2km from distillery). Maritime, gentle style when unpeated. Capacity: 2.5 million litres.",
    "region": "Islay",
    "country": "Scotland",
    "establishedYear": 1881,
    "company": "Distell (Heineken)",
    "masterCaskCount": 11,
    "rank": 1,
    "website": "https://bunnahabhain.com",
    "imageUrl": "https://caskx-exchange-static-assets.s3.us-east-1.amazonaws.com/distilleries/distillery_9c9d3817-f02a-4aa8-bb48-52b2ea6b44c1/ad890357-d1c4-478b-813c-b582b6b52c08.webp",
    "isVerified": false,
    "status": "active",
    "createdAt": "2026-04-10T06:06:29.522Z",
    "updatedAt": "2026-04-10T07:05:17.323Z"
  },
  "distilleryName": "Bunnahabhain",
  "caskTypeId": "cask_type_b8ef36cf-ffd2-444f-b70d-d68e0f274e17",
  "caskType": {
    "id": "cask_type_b8ef36cf-ffd2-444f-b70d-d68e0f274e17",
    "name": "Ex-Sherry Hogshead",
    "typicalCapacityLiters": null,
    "description": null
  },
  "regionId": "region_157319aa-4c0a-4a46-b07a-a07c9290c28c",
  "region": {
    "id": "region_157319aa-4c0a-4a46-b07a-a07c9290c28c",
    "name": "Highland",
    "description": null,
    "country": "Scotland",
    "createdAt": "2026-04-10T06:06:29.522Z",
    "updatedAt": "2026-04-10T06:06:29.522Z"
  },
  "categoryLabel": "Single Malt",
  "classification": "single_malt_scotch",
  "classificationLabel": "Single Malt Scotch",
  "status": "active",
  "peatLevels": null,
  "lowestAsk": null,
  "floorPrice": 7214,
  "highestBid": 12100,
  "minReferencePrice": 7214,
  "maxReferencePrice": 25000,
  "viewCount": 394,
  "createdAt": "2026-04-10T06:06:29.522Z",
  "updatedAt": "2026-07-08T05:06:24.570Z",
  "lowestAskPrev30D": null,
  "lowestAskDelta30D": null,
  "lifetimeVolume": "11000.00",
  "volumeDelta30D": "-100.00",
  "medianPrice": "11000.00",
  "medianPriceDelta30D": null,
  "activeBidsCount": "2.00",
  "salesCount30D": "0.00",
  "demandScore": "1.20",
  "price": 7214,
  "children": [
    {
      "id": "cask_ac0aceb8-c96c-4188-8e88-409430288fa3",
      "name": "2005",
      "masterId": "cask_master_051d845b-f828-4c12-8da0-f06ec40fddca",
      "distilleryId": "distillery_9c9d3817-f02a-4aa8-bb48-52b2ea6b44c1",
      "distillationDate": "2005-09-01",
      "vintageYear": 2005,
      "age": 21,
      "expectedMaturityDate": null,
      "ola": null,
      "rla": "85.10",
      "abv": null,
      "estimatedBottleCount": 121,
      "priceReference": "20000.00",
      "referencePriceMin": "15000.00",
      "referencePriceMax": "25000.00",
      "readyToSell": true,
      "bottleVolume": 700,
      "order": 0,
      "lowestAsk": null,
      "highestBid": 12100,
      "caskStatus": "active",
      "description": "A Highland single malt from Dalmore in Alness, founded 1839. Owned by Whyte & Mackay, featuring iconic 12-point Royal Stag emblem. Known for multi-cask maturation using bourbon and exclusive aged sherry casks from González Byass. Master Distiller Richard Paterson selects each cask. Produces rich, sherried Highland whisky with legendary finesse.",
      "tastingNotes": "20-year Highland with dark chocolate, orange marmalade, dried fruit, vanilla, warm spice, sherry richness, and long elegant finish.",
      "storageLocation": null,
      "imageUrl": "https://caskx-exchange-static-assets.s3.us-east-1.amazonaws.com/casks/cask_ac0aceb8-c96c-4188-8e88-409430288fa3/63d0d153-8bb6-463b-a875-6028288c46c4.webp",
      "caskTypeId": "cask_type_b8ef36cf-ffd2-444f-b70d-d68e0f274e17",
      "classification": "single_malt_scotch",
      "classificationLabel": "Single Malt Scotch",
      "regionId": "region_157319aa-4c0a-4a46-b07a-a07c9290c28c",
      "spiritType": null,
      "spiritTypeId": null,
      "createdAt": "2026-04-10T06:06:29.522Z",
      "updatedAt": "2026-07-07T08:08:22.687Z",
      "listingDate": "2026-04-10T06:06:29.262Z",
      "activeAt": "2026-07-07T08:08:22.686Z",
      "verificationDate": null,
      "isListed": true,
      "isVerified": false,
      "viewCount": 0
    }
  ]
}
```

### Example 2: Floor Price resolves to Lowest Ask (`lowestAsk` is present)

If `lowestAsk` is present, `floorPrice` resolves to `lowestAsk` (`12000` in the sample below):

```json
{
  "id": "cask_master_051d845b-f828-4c12-8da0-f06ec40fddca",
  "name": "Dalmore Ex-Sherry Hogshead",
  "lowestAsk": 12000,
  "floorPrice": 12000,
  "highestBid": 12100,
  "minReferencePrice": 7214,
  "maxReferencePrice": 25000,
  ...
}
```
