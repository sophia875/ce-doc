# Parent / Child (Master / Variant) Data Structure

## Overview
The platform uses a Parent / Child structure to manage casks. This ensures consistency for shared attributes while allowing flexibility for vintage-specific data.

## 1. Parent (Master Cask) — Global Fields
| Field | Type | Description |
|-------|------|-------------|
| name | string | Cask name (shared across all vintages) |
| distillery | string (FK) | Selected from distillery list |
| caskType | string (FK) | Selected from cask type list |
| classification | string (FK) | E.g., "Single Malt" |
| region | string (FK) | E.g., "Highlands" |
| image | string (URL) | Primary product image |
| viewCount | number | Page view counter at master cask level |

## 2. Child (Variant / Vintage) — Specific Fields
| Field | Type | Description |
|-------|------|-------------|
| vintageYear | number | Required. Year of the vintage |
| distillationDate | date | Date of distillation |
| age | number | Age of the cask |
| expectedMaturityDate | date | Expected maturity date |
| ola | number | Original Litres of Alcohol |
| rla | number | Regauged Litres of Alcohol |
| abv | number | Alcohol By Volume |
| estimatedBottleCount | number | Estimated number of bottles |
| bottleVolume | number | Volume per bottle (ml) |
| referencePriceMin | number | Required. Minimum reference price |
| referencePriceMax | number | Required. Maximum reference price |
| description | string | Variant-specific description |
| tastingNotes | string | Tasting notes |
| imageUrl | string (URL) | Variant-specific image (optional override) |
| order | number | Display order within parent |

## Key Rules
- A Parent cannot exist without at least one Child
- Each Child must be linked to exactly one Parent
- Global fields are inherited from Parent → not editable at Child level
