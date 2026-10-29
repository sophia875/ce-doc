# API Change Requests: Endpoint Renames

This document outlines the required renames for existing cask-related endpoints. These renames align the routing structure with the parent-child product architecture, shifting endpoints from a variant-centric `/api/casks` base path to a master-centric `/api/cask-masters` base path.

## Endpoint Renames

| Existing Path | Proposed New Path | Description |
|:---|:---|:---|
| `GET /api/casks` | `GET /api/cask-masters` | Get all master casks. |
| `GET /api/casks/listing` | `GET /api/cask-masters/listing` | Get paginated master casks with filtering and sorting. |
| `GET /api/casks/search` | `GET /api/cask-masters/search` | Search master casks by name, reference, or distillery. |
| `GET /api/casks/recently-viewed` | `GET /api/cask-masters/recently-viewed` | Get current user's recently viewed master casks. |
| `GET /api/casks/{id}` | `GET /api/cask-masters/{id}` | Get detail of a single master cask. |
| `PUT /api/casks/{id}` | `PUT /api/cask-masters/{id}` | Update master cask details. |
| `DELETE /api/casks/{id}` | `DELETE /api/cask-masters/{id}` | Delete a master cask. |
| `GET /api/casks/filter-options/distilleries` | `GET /api/cask-masters/filter-options/distilleries` | Get distillery filter options with active counts. |
| `GET /api/casks/filter-options/cask-types` | `GET /api/cask-masters/filter-options/cask-types` | Get cask type filter options with active counts. |
| `POST /api/casks/{id}/view` | `POST /api/cask-masters/{id}/view` | Increment master cask view count. |
| `GET /api/casks/{id}/similar` | `GET /api/cask-masters/{id}/similar` | Get similar master casks (scored at master level - see [similar-casks-api.md](file:///d:/CX/docs/marketplace/similar-casks-api.md)). |

## Rationale
Since the marketplace and listing layouts operate primarily at the **Master Cask** (Parent) level, grouping the vintages (children) dynamically, the base query routing should reflect `/cask-masters` instead of `/casks` to prevent developer confusion and maintain clean semantic paths.
