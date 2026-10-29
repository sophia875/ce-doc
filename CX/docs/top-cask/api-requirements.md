# BACKEND LOGIC & FLOW SPECS

Top Cask Curated List (Scored Recommendations)

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Initial creation of Backend Logic & Flow specs | Alice | 11/06/2026 |

---

## I. Introduction

### 1. Purpose

Provide a sorted and scoring-based master cask recommendation list for the **Top Casks** section on the homepage and marketplace. This section surfaces master casks ranked based on a custom hybrid liquidity score that balances buyer demand and recent market transaction velocity.

- **Primary Sorting**: Ranked by `demandScore DESC`
- **Endpoint**: Uses the unified `GET /api/cask-masters` endpoint (the same endpoint used by the Explore Casks tabs), differentiated by supplying the `sortBy=demandScore` parameter.

### 2. Definitions

| Term | Definition |
|------|------------|
| Master Cask (Parent) | A parent entity grouping multiple vintages/variants. Holds shared attributes (e.g., name, distillery, caskType, classification, region, image). |
| Variant Cask (Child) | A specific vintage of a Master Cask. Transactions and listings occur at the variant level. |
| `activeBidsCount` | The total quantity of active buy orders (Bids) across all variants of a master cask where `bid.status = 'active'` and `bid.expirationDate > NOW()` (UK timezone). |
| `salesCount30D` | The total quantity of casks transacted in completed checkout sessions (`status = 'completed'`) within the last 30 days. |
| `lowestAsk` | The minimum active ask price across all active variants of a master cask. Sourced from `MIN(variant.lowestAsk)`. `null` if no variants have active asks. |
| `minReferencePrice` | The minimum reference price across all active variants of a master cask. Sourced from `MIN(variant.referencePriceMin)`. |
| `demandScore` | The calculated score of a master cask used to rank and recommend high-liquidity casks. |

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| **TC-BR-1** | **Master Cask Eligibility** | Only master casks where `masterCask.status = 'active'` AND with at least 1 variant where `caskStatus = 'active'` and `isListed = true` are eligible for ranking. |
| **TC-BR-2** | **Deterministic Tie-break** | When scores are equal, apply the following secondary sort: 1. `COALESCE(lowestAsk, minReferencePrice) ASC` (active floor price, falling back to reference price). If both are null, the cask is sorted after all priced casks (`NULLS LAST`). 2. `id ASC` (alphanumeric UUID sort). |
| **TC-BR-3** | **Query Capping** | The default result set size returned to the frontend is capped, driven by the `size` query parameter. |

---

## II. Requirements

### 1. Endpoint Strategy

> The Top Casks feature leverages the unified **`GET /api/cask-masters`** endpoint (shared with the Explore Casks and Market Movers sections). The frontend retrieves the Top Casks by making a request to this endpoint with the query parameter `sortBy=demandScore`.

### 2. Master Cask Response Structure

#### 2.1. Overview

The endpoint returns the same base card response structure as Explore Casks, augmented with the specific fields needed to render the Top Casks.

#### 2.2. Metric Implementation & Calculation Table

The following table defines the logic for the new master cask metrics defined by the API. These metrics are calculated and joined at the master cask level.

The endpoint **returns all fields currently provided by `/api/cask-masters`**. The fields with **"Included in Response? = ✅"** must be added to the response to support Top Casks:

| # | Metric | Included in Response? | Data Type | Calculation Formula | Notes / Dependencies |
|---|--------|-----------------------|-----------|---------------------|----------------------|
| 1 | `activeBidsCount` | ❌ | number | `SUM(bid.quantity)` | Sourced from `cask_bids` table.<br>Filter: `bid.status = 'active'` AND `bid.expirationDate > NOW()` (UK timezone). |
| 2 | `salesCount30D` | ❌ | number | `SUM(session.quantity)` | Sourced from `checkout_sessions` table.<br>Window: `session.completedAt >= NOW() - 30 days` (UK timezone). Filter: `session.status = 'completed'`. |
| 3 | `demandScore` | ❌ | number | `(0.6 * activeBidsCount) + (0.4 * salesCount30D)` | Pre-computed hybrid scoring value used for primary sorting. |

#### 2.3. Use Cases

##### UC-1. Fetch Top Casks

| Item | Description |
|------|-------------|
| **Objective** | Return the highest-scoring eligible master casks sorted by `demandScore` DESC. |
| **Actor** | System (triggered by Frontend) |
| **Trigger** | Frontend requests the unified endpoint: `GET /api/cask-masters?sortBy=demandScore`. |
| **Pre-condition(s)** | Master casks and variant casks exist in database. |
| **Post-condition(s)** | Returns a sorted JSON array of master cask card objects. |

**Happy Path**:

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Valid request with standard database state | Returns JSON list of top master casks matching the size, sorted descending by score. |

**Negative Path**:

| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | Unexpected database connection failure | Returns server error payload. | 500 — "Internal Server Error" |
| 2 | No eligible master casks in database | Returns empty JSON list. | 200 — `[]` |

#### 2.4. Edge Cases (feature-level)

| # | Scenario | Related UC(s) | Expected Behavior | Side Effects |
|---|----------|---------------|-------------------|-------------|
| 1 | A transaction completes or a bid is placed mid-session | UC-1 | The score remains at the last pre-computed value. It will be updated automatically during the next scheduler interval. | None |
| 2 | All master casks have a score of 0.0 (e.g., brand-new platform) | UC-1 | Casks are sorted entirely by their tie-breakers (`lowestAsk ASC`, then `id ASC`). | List remains populated using price-based sorting. |

---

## III. Performance Considerations

1. **Refresh Schedule**:
   - A background CRON worker should run every **30 minutes** to recalculate metrics in `MasterCaskMarketStats` for all active master casks.
   - When a new transaction completes (`CheckoutSession` status transitions to `completed`), the target master cask's `salesCount30D` and `demandScore` can be updated deferred to the next cron interval.
