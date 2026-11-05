# Cask Detail V2: Availability and Demand API Update

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Define Availability and Demand as aggregated matchable cask quantities | BA | 2026-11-05 |
| 0.2 | Add Availability and Demand to the response; exclude negative matchable quantities | BA | 2026-11-05 |

## I. Introduction

### 1. Purpose

Update the market indicators displayed on Cask Detail V2. The existing public endpoint provides market data for a selected cask vintage. `Availability` must represent the quantity currently offered for sale, and `Demand` must represent the quantity buyers are currently seeking.

### 2. Definitions

| Term | Definition |
|------|------------|
| Cask vintage | The variant identified by the endpoint's `{caskId}` path parameter. Aggregation is for this vintage only, not its parent Master Cask or sibling vintages. |
| Eligible ask | An ask for the requested vintage with `status = 'active'` and `expirationDate > NOW()`. |
| Eligible bid | A bid for the requested vintage with `status = 'active'` and `expirationDate > NOW()`. |
| `matchableQuantity` | Quantity still available for matching: `targetQuantity - filledQuantity - withheldQuantity`. |
| Availability | Sum of positive `matchableQuantity` values across eligible asks for the requested vintage. |
| Demand | Sum of positive `matchableQuantity` values across eligible bids for the requested vintage. |

## II. Requirements

### 1. Endpoint

| Method | Path | Authentication |
|--------|------|----------------|
| GET | `/api/cask-bids/market-data/{caskId}` | Public |

No request path, query parameter, or authentication changes are required. The change adds two fields to the endpoint's response.

### 2. Business Rules

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| CDM-BR-1 | Availability is an ask quantity total | `availability = SUM(ask.matchableQuantity)` for eligible asks belonging to the requested `caskId`, including only orders where `matchableQuantity > 0`. |
| CDM-BR-2 | Demand is a bid quantity total | `demand = SUM(bid.matchableQuantity)` for eligible bids belonging to the requested `caskId`, including only orders where `matchableQuantity > 0`. |
| CDM-BR-3 | Only live orders contribute | Exclude orders whose status is not `active` or whose `expirationDate <= NOW()`. Cancelled, completed, and expired orders must not contribute. |
| CDM-BR-4 | Use positive matchable quantity | Do not count the original `targetQuantity` or order count. Calculate `matchableQuantity = targetQuantity - filledQuantity - withheldQuantity`; exclude orders where this value is zero or negative. |
| CDM-BR-5 | Empty totals are zero | If no eligible orders contribute, return `0` for the corresponding indicator, not `null`. |
| CDM-BR-6 | Add response fields | Add `availability` and `demand` as numeric fields to the existing response. |
| CDM-BR-7 | Preserve existing response data | Keep all current response fields and calculations unchanged; the two new fields are additive. |

### 3. Data Changes

| Field | Type | Required | Change | Description |
|-------|------|----------|--------|-------------|
| `availability` | number | Yes | New response field | Sum of positive matchable quantities across eligible asks for the selected vintage. |
| `demand` | number | Yes | New response field | Sum of positive matchable quantities across eligible bids for the selected vintage. |

These are new fields in the response body, not request parameters. No database schema change or new endpoint is required. Add them to the existing response without removing or renaming any fields. Example of the added response fields:

```json
{
  "availability": 5,
  "demand": 7
}
```

The example shows only the new fields; the actual response must also retain all existing fields.

### 4. Use Case

#### UC-1. Get Market Data for a Cask Vintage

| Item | Description |
|------|-------------|
| **Objective** | Return market data with Availability and Demand showing current matchable cask quantities. |
| **Actor** | Public client |
| **Trigger** | Cask Detail V2 requests `GET /api/cask-bids/market-data/{caskId}` for the selected vintage. |
| **Pre-condition(s)** | The requested cask vintage exists. |
| **Post-condition(s)** | The response includes recalculated `availability` and `demand`; all other market data remains unchanged. |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Find eligible asks | CDM-BR-1, CDM-BR-3 | Select active, unexpired asks belonging to the requested vintage. |
| 2 | Sum ask quantities | CDM-BR-4, CDM-BR-5 | Sum each eligible ask's matchable quantity; use `0` when none qualify. |
| 3 | Find and sum eligible bids | CDM-BR-2, CDM-BR-3, CDM-BR-4, CDM-BR-5 | Select active, unexpired bids belonging to the requested vintage and sum their matchable quantities; use `0` when none qualify. |
| 4 | Return market data | CDM-BR-6, CDM-BR-7 | Return the existing response with the new `availability` and `demand` fields. |

### 5. Acceptance Examples

| Scenario | Eligible order quantities (`matchableQuantity`) | Expected result |
|----------|-----------------------------------------------|-----------------|
| Multiple asks and bids | Asks: `2`, `3`; bids: `4`, `3` | `availability = 5`, `demand = 7` |
| Partially filled or withheld order | An active ask has `targetQuantity = 8`, `filledQuantity = 2`, `withheldQuantity = 1` | Ask contributes `5` to `availability` |
| Negative matchable quantity | An active ask has `matchableQuantity = -2`; an active bid has `matchableQuantity = -1` | Both orders are excluded; neither contributes to the total |
| Expired or inactive order | An order has `status = 'expired'`, or `expirationDate <= NOW()` | Order contributes `0` |
| No eligible asks | No active, unexpired asks with positive matchable quantity exist | `availability = 0` |
| No eligible bids | No active, unexpired bids with positive matchable quantity exist | `demand = 0` |
| Different vintage | Active orders exist for a sibling vintage, not the requested `{caskId}` | Those orders do not contribute |

### 6. Edge Cases

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 1 | An order's `matchableQuantity` is zero or negative | Exclude the order from the corresponding sum. Do not clamp its value or allow it to reduce the total. |
| 2 | Order status or expiry changes while market data is being queried | Calculate both totals from the eligible order state visible to the request; do not include inactive or expired orders. |

## III. Access Control

| Action | Anonymous | Authenticated User | Admin |
|--------|-----------|--------------------|-------|
| View market data for a cask vintage | Yes | Yes | Yes |

_End of Document_
