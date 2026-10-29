# Specification Update: `/api/cask-metadata/classifications`

This document defines the update requirements for the cask classifications metadata endpoint (`GET /api/cask-metadata/classifications`).

---

## 1. Overview of Changes

* **Primary Objective**: Streamline and update the classifications metadata list returned to the frontend by renaming specific labels and purging unused/deprecated classification categories.
* **Key Updates**:
  1. **Rename Labels**:
     * `"Single Malt Scotch"` ➔ **`"Single Malt"`**
     * `"Single Grain Scotch"` ➔ **`"Single Grain"`**
     * `"Irish Whiskey"` ➔ **`"Irish Single Malt"`**
  2. **Remove Deprecated Options**:
     * Remove all 10 remaining classification options (`blended_scotch`, `blended_malt_scotch`, `single_pot_still_irish_whiskey`, `bourbon`, `tennessee_whiskey`, `rye_whiskey`, `canadian_whisky`, `japanese_whisky`, `indian_whisky`, `australian_whisky`).
  3. **Resulting Active Options**: Exactly **3** active classifications will be returned.

---

## 2. API Specification

### Endpoint: `GET /api/cask-metadata/classifications`

* **HTTP Method**: `GET`
* **Authentication**: Public
* **Query Parameters**: None (or optional filter context parameters if dynamic dependent filtering is enabled)

#### Response Payload (`200 OK`)

```json
{
  "classifications": [
    {
      "value": "single_malt",
      "label": "Single Malt",
      "imageUrl": "https://caskx-exchange-static-assets.s3.us-east-1.amazonaws.com/classifications/single_malt_scotch/633227bd-e1ef-4e72-8710-fac99fcc087b.png"
    },
    {
      "value": "single_grain",
      "label": "Single Grain",
      "imageUrl": null
    },
    {
      "value": "irish_single_malt",
      "label": "Irish Single Malt",
      "imageUrl": null
    }
  ]
}
```

> **Note on `value` Key Mapping**:
> * Standard updated key values are `single_malt`, `single_grain`, and `irish_single_malt`.
> * If backward compatibility with legacy database values (`single_malt_scotch`, `single_grain_scotch`, `irish_whiskey`) is required without database migration, the `value` field can retain the legacy identifier while serving the updated `label`. See Section 3 for mapping details.

---

## 3. Classifications Mapping Table

| Previous Label | Previous Value | Updated Label | Recommended Value (`value`) | Status | `imageUrl` |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Single Malt Scotch | `single_malt_scotch` | **Single Malt** | `single_malt` *(or `single_malt_scotch`)* | ✅ **Kept & Renamed** | Asset URL / Preserved |
| Single Grain Scotch | `single_grain_scotch` | **Single Grain** | `single_grain` *(or `single_grain_scotch`)* | ✅ **Kept & Renamed** | `null` |
| Irish Whiskey | `irish_whiskey` | **Irish Single Malt** | `irish_single_malt` *(or `irish_whiskey`)* | ✅ **Kept & Renamed** | `null` |
| Blended Scotch | `blended_scotch` | - | - | ❌ **Removed** | N/A |
| Blended Malt Scotch | `blended_malt_scotch` | - | - | ❌ **Removed** | N/A |
| Single Pot Still Irish Whiskey | `single_pot_still_irish_whiskey` | - | - | ❌ **Removed** | N/A |
| Bourbon | `bourbon` | - | - | ❌ **Removed** | N/A |
| Tennessee Whiskey | `tennessee_whiskey` | - | - | ❌ **Removed** | N/A |
| Rye Whiskey | `rye_whiskey` | - | - | ❌ **Removed** | N/A |
| Canadian Whisky | `canadian_whisky` | - | - | ❌ **Removed** | N/A |
| Japanese Whisky | `japanese_whisky` | - | - | ❌ **Removed** | N/A |
| Indian Whisky | `indian_whisky` | - | - | ❌ **Removed** | N/A |
| Australian Whisky | `australian_whisky` | - | - | ❌ **Removed** | N/A |

---

## 4. Backend Implementation Guidelines

1. **Metadata Master List Filter**:
   * Update the classification metadata seed/service configuration to only output the 3 supported classifications.
2. **Database Query / Filtering Logic**:
   * Ensure `GET /api/casks/listing` filter queries accept the updated `value` keys (`single_malt`, `single_grain`, `irish_single_malt`).
   * If existing cask records in the database use legacy strings (e.g. `single_malt_scotch`), either:
     * **Option A (DB Data Migration)**: Update existing DB rows to use new values (`single_malt`, `single_grain`, `irish_single_malt`).
     * **Option B (BE Alias Mapping)**: Map incoming query filter values in backend repository layer (`single_malt` ➔ DB value `single_malt_scotch`).
3. **Response Payload Construction**:
   * Ensure `imageUrl` is preserved for Single Malt (or updated if new branding assets are uploaded).

---

## 5. Frontend Integration Impact

* **Sidebar Category Filter**: Only 3 options will display in the classification filter checkbox list.
* **Cask Cards & Detail Badges**: Category badges will display `"Single Malt"`, `"Single Grain"`, and `"Irish Single Malt"`.
* **Query Params**: Frontend sends the updated `value` parameter when users select category filters in Marketplace.
