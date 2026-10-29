# API Requirements: Market Activities

This document provides the complete API specification for the **Market Activities** feature on Cask Exchange.

---

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Initial draft for Market Activities API endpoint supporting active listings & offers, filtering, sorting, and pagination. | System Architect | 10/09/2026 |
| 0.2 | Refined specs per review: restricted to authenticated users, fixed sort to `createdAt` DESC, removed status & expiry calculation fields. | System Architect | 10/09/2026 |
| 0.3 | Added formal Use Case section (UC-MA-01) detailing Main Flow, Pre/Post Conditions, and Exception Flows for BA/Dev/QA alignment. | Business Analyst | 10/09/2026 |

---

## 1. Introduction

### 1.1 Purpose
The **Market Activities** API endpoint provides a real-time list of all active listings (asks) and offers (bids) across all casks in the marketplace for logged-in users. This feed allows authenticated buyers and sellers to monitor recent market actions and price trends.

### 1.2 Key Objectives
- Retrieve active marketplace activities (both **Listings** and **Offers**) for all casks.
- Restrict endpoint access to authenticated users only.
- Support activity type filtering: Show all activities (`ALL`), show listings only (`LISTING`), or show offers only (`OFFER`).
- Sort activities by default by creation date (`createdAt` DESC — newest to oldest).
- Support standard offset-based pagination (`page`, `limit`).

---

## 2. Definitions & Terminology

| Term | Definition |
| --- | --- |
| **Master Cask** | The parent entity representing a distillery's cask product line (e.g., Benriach Ex-Bourbon Cask). |
| **Variant Cask** | A specific vintage / batch instance of a Master Cask where trading actually occurs. |
| **Listing (Ask)** | A seller's sell order stating an asking price to sell a cask variant. |
| **Offer (Bid)** | A buyer's buy order stating an offer price to purchase a cask variant. |
| **Active Listing** | A listing where `status = 'active'`, `expirationDate > NOW()`, and `variant.caskStatus = 'active'`. |
| **Active Offer** | An offer where `status = 'active'`, `expirationDate > NOW()`, and `variant.caskStatus = 'active'`. |

---

## 3. Use Case & Business Rules

### 3.1 Use Case Specifications

#### UC-MA-01: Fetch Market Activities Feed

- **Objective**: Retrieve a paginated, real-time list of active marketplace listings and offers across all casks.
- **Primary Actor**: Authenticated User (Buyer / Seller / Admin).
- **Trigger**: User opens the Market Activities section, changes the activity type filter (`ALL`, `LISTING`, `OFFER`), or navigates pages.
- **Pre-conditions**:
  1. The user must be authenticated with a valid Bearer JWT Token.
- **Post-conditions**:
  1. The system returns an HTTP 200 OK response with the filtered, paginated list of active activities sorted by `createdAt DESC`.

#### Main Activity Flow
1. **Frontend Request**: The client sends a `GET /api/market-activities` request with optional parameters (`type`, `page`, `limit`, `sortBy`, `sortOrder`) and the `Authorization` header containing the JWT token.
2. **Authentication Verification**: The backend validates the JWT token (**MA-BR-1**).
3. **Parameter Validation**: The backend validates query parameters (`type` in `[ALL, LISTING, OFFER]`, `page >= 1`, `1 <= limit <= 100`).
4. **Data Querying & Filtering**:
   - Query listings table for `status = 'active'` AND `expirationDate > NOW()` AND `variant.caskStatus = 'active'`.
   - Query offers table for `status = 'active'` AND `expirationDate > NOW()` AND `variant.caskStatus = 'active'`.
   - Combine results based on `type` filter (**MA-BR-2**, **MA-BR-3**).
5. **Sorting & Pagination**: Sort combined dataset by `createdAt DESC` (**MA-BR-4**), apply offset pagination (`page`, `limit`) (**MA-BR-5**).
6. **Response Generation**: Construct payload returning `data` array and `meta.pagination` object.

#### Exception & Edge Flows
- **EF-1: Unauthenticated User**: If JWT token is missing or invalid, return `401 Unauthorized` (**EF-MA-1**).
- **EF-2: Invalid Filter Parameter**: If `type` parameter is not in `ALL`, `LISTING`, `OFFER`, return `400 Bad Request` with `INVALID_PARAMETER` error code (**EF-MA-2**).
- **EF-3: Empty Result Set**: If no active activities match the criteria, return `200 OK` with an empty `data: []` array and `total: 0` (**EF-MA-3**).

---

### 3.2 Business Rules Table

| Rule Code | Description | Logic / Condition |
| --- | --- | --- |
| **MA-BR-1** | **Authentication Required** | Access is strictly restricted to logged-in users. Calls without a valid Bearer JWT Token return `401 Unauthorized`. |
| **MA-BR-2** | **Active Status Eligibility** | Only active items are returned (`status = 'active'` AND `expirationDate > NOW()` AND `variant.caskStatus = 'active'`). Inactive, expired, cancelled, or completed listings/offers are automatically filtered out by backend query logic. |
| **MA-BR-3** | **Activity Type Filter** | The `type` query parameter filters items:<br>- `ALL` (default): Combines active listings and offers.<br>- `LISTING`: Returns active listings (asks) only.<br>- `OFFER`: Returns active offers (bids) only. |
| **MA-BR-4** | **Default Sorting** | Results are strictly sorted by creation date from newest to oldest (`createdAt DESC`). Fallback tie-breaker: `id DESC`. |
| **MA-BR-5** | **Pagination & Range Limits** | Standard offset pagination via `page` (default `1`, min `1`) and `limit` (default `10`, min `1`, max `100`). Total count (`total`) and total pages (`totalPages`) are included in pagination metadata. |

---

## 4. Endpoint Specifications

### 4.1 Endpoint Details

- **HTTP Method**: `GET`
- **Route**: `/api/market-activities`
- **Authentication**: **Required** (`Bearer JWT Token`). Public/Anonymous access is forbidden.

---

### 4.2 Query Parameters

| Parameter | Type | Required | Default | Allowed Values | Description |
| --- | --- | --- | --- | --- | --- |
| `type` | String | No | `ALL` | `ALL`, `LISTING`, `OFFER` | Filter activities by type. `ALL` shows both listings and offers, `LISTING` shows listings only, `OFFER` shows offers only. |
| `page` | Integer | No | `1` | `>= 1` | Page number for pagination. |
| `limit` | Integer | No | `10` | `1..100` | Number of items per page. |
| `sortBy` | String | No | `createdAt` | `createdAt`, `price` | Field to sort results by. Defaults to `createdAt`. |
| `sortOrder` | String | No | `desc` | `desc`, `asc` | Sort direction. Defaults to `desc` (newest to oldest). |

---

### 4.3 Response Structure

#### 4.3.1 Success Response (200 OK)

```json
{
  "success": true,
  "data": [
    {
      "id": "act_10928374",
      "activityType": "LISTING",
      "cask": {
        "id": "cask_v789012",
        "masterCaskId": "mc_123456",
        "name": "Benriach Ex-Bourbon Cask 2012",
        "distilleryName": "Benriach",
        "caskType": "Ex-Bourbon Barrel",
        "vintageYear": 2012,
        "imageUrl": "https://cdn.caskexchange.com/casks/benriach-2012.jpg"
      },
      "price": 12200.00,
      "currency": "GBP",
      "createdAt": "2026-09-10T14:30:00Z",
      "expirationDate": "2026-09-13T14:30:00Z"
    },
    {
      "id": "act_10928375",
      "activityType": "OFFER",
      "cask": {
        "id": "cask_v345678",
        "masterCaskId": "mc_654321",
        "name": "Caol Ila Ex-Bourbon Cask 2015",
        "distilleryName": "Caol Ila",
        "caskType": "Ex-Bourbon Hogshead",
        "vintageYear": 2015,
        "imageUrl": "https://cdn.caskexchange.com/casks/caol-ila-2015.jpg"
      },
      "price": 11400.00,
      "currency": "GBP",
      "createdAt": "2026-09-10T12:00:00Z",
      "expirationDate": "2026-09-11T00:00:00Z"
    }
  ],
  "meta": {
    "pagination": {
      "total": 142,
      "page": 1,
      "limit": 10,
      "totalPages": 15,
      "hasNextPage": true,
      "hasPrevPage": false
    },
    "filters": {
      "type": "ALL",
      "sortBy": "createdAt",
      "sortOrder": "desc"
    }
  }
}
```

---

### 4.4 Data Field Definitions

| Field Path | Data Type | Description | Example |
| --- | --- | --- | --- |
| `id` | String | Unique identifier of the listing or offer record | `"act_10928374"` |
| `activityType` | Enum | Type of activity (`LISTING` or `OFFER`) | `"LISTING"` |
| `cask.id` | String | Unique identifier of the variant cask | `"cask_v789012"` |
| `cask.masterCaskId` | String | Parent master cask ID | `"mc_123456"` |
| `cask.name` | String | Full display name of the cask | `"Benriach Ex-Bourbon Cask 2012"` |
| `cask.distilleryName` | String | Name of the distillery | `"Benriach"` |
| `cask.caskType` | String | Classification or cask type description | `"Ex-Bourbon Barrel"` |
| `cask.vintageYear` | Integer | Vintage year of distillation | `2012` |
| `cask.imageUrl` | String | Product image URL | `"https://cdn..."` |
| `price` | Number (Decimal) | Listing asking price or offer bid price in decimal | `12200.00` |
| `currency` | String | Currency code (ISO 4217 standard) | `"GBP"` |
| `createdAt` | String (ISO 8601) | Timestamp when the activity was created | `"2026-09-10T14:30:00Z"` |
| `expirationDate` | String (ISO 8601) | Expiration timestamp of the listing/offer | `"2026-09-13T14:30:00Z"` |
| `meta.pagination.total` | Integer | Total count of active activities matching filter criteria | `142` |
| `meta.pagination.page` | Integer | Current page number | `1` |
| `meta.pagination.limit` | Integer | Number of items requested per page | `10` |
| `meta.pagination.totalPages` | Integer | Total computed pages | `15` |

---

## 5. Error Handling & HTTP Status Codes

| HTTP Code | Error Code | Scenario | Response Payload |
| --- | --- | --- | --- |
| `401` | `UNAUTHORIZED` | Bearer token is missing or invalid | `{"success": false, "error": {"code": "UNAUTHORIZED", "message": "Authentication token is required to access market activities"}}` |
| `400` | `INVALID_PARAMETER` | Invalid `type` parameter (e.g., `type=INVALID`) | `{"success": false, "error": {"code": "INVALID_PARAMETER", "message": "Invalid type parameter. Allowed values: ALL, LISTING, OFFER"}}` |
| `400` | `INVALID_PAGINATION` | `page < 1` or `limit > 100` | `{"success": false, "error": {"code": "INVALID_PAGINATION", "message": "Limit must be between 1 and 100"}}` |
| `500` | `INTERNAL_SERVER_ERROR` | Database query failure or unexpected server error | `{"success": false, "error": {"code": "INTERNAL_SERVER_ERROR", "message": "An unexpected error occurred while fetching market activities"}}` |

---

## 6. Permissions & Security

| User Role | Access Allowed | Description |
| --- | --- | --- |
| **Anonymous (Guest)** | ❌ No | Access forbidden (`401 Unauthorized`). |
| **Authenticated Buyer / Seller** | ✅ Yes | Full access to view market activities. |
| **Admin** | ✅ Yes | Full access. |

---

## 7. Performance & Caching Recommendations

1. **Database Indexing**:
   - Composite index on Listing table: `(status, expiration_date, created_at DESC)`
   - Composite index on Offer table: `(status, expiration_date, created_at DESC)`
2. **Caching Strategy**:
   - User-scoped or short TTL caching (e.g., TTL = 10 seconds) for paginated requests.
