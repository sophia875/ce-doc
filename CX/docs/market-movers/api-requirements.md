# BACKEND LOGIC & FLOW SPECS

Market Movers — Home Page Curated List

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | Antigravity | 08/06/2026 |

---

## I. Introduction

### 1. Purpose

> Provide a sorted list of master casks for the **Market Movers** section on the Home page. This section surfaces master casks that are experiencing the highest positive momentum in trading volume over the last 30 days.
> 
> - **Primary Sorting**: Ranked by `volumeDelta30D DESC`.
> - **Endpoint**: Uses the unified `GET /api/cask-masters` endpoint (the same endpoint used by the Explore Casks tabs), differentiated by supplying the `sortBy=volumeDelta30D` parameter.

---

### 2. Definitions

| Term | Definition |
|------|------------|
| Master Cask (Parent) | A parent entity grouping multiple variant casks. |
| volumeLast30D | Total volume (`SUM(cs.originalAmount)`) of completed checkout sessions in the last 30 days across all variants of the master cask. |
| volumePrev30D | Total volume of completed checkout sessions in the prior 30-day window (days 31 to 60 ago). |
| volumeDelta30D | Percentage change in trading volume: `((volumeLast30D - volumePrev30D) / volumePrev30D) * 100`. |
| medianPrice | Unit-weighted median transaction price of all completed transactions for this master cask, all time. |
| medianPriceDelta30D | Percentage change in median transaction price: last 30 days vs. prior 30 days. |
| Live Floor (`lowestAsk`) | The current minimum active ask price across all active variants of the master cask. Represents the real market entry point. |
| Deterministic Tie-break | The fallback sorting used to ensure stable results when primary metrics are equal. |

---

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| **MM-BR-1** | **Eligibility: Active Status** | Only master casks where `masterCask.status = 'active'` AND with ≥ 1 variant where `caskStatus = 'active'` AND `isListed = true` are eligible (matches Explore Casks logic). |
| **MM-BR-2** | **Ranking** | Eligible casks are ranked by `volumeDelta30D DESC`. |
| **MM-BR-3** | **Result Cap** | Returns the number of casks specified by the `size` request parameter from the FE. |
| **MM-BR-4** | **Deterministic Tie-break** | When `volumeDelta30D` values are equal, fallback tie-break is `lifetimeVolume DESC`, then `id ASC`. |
| **MM-BR-5** | **Delta Calculation & Cold Start** | If `volumePrev30D = 0`, `volumeDelta30D` is considered `null` (division by zero). Cold start is not supported for Market Movers; casks with `volumeDelta30D = null` OR `lifetimeVolume = 0` must be excluded by the BE. If any are returned, the FE must hide the row. |

---

## II. Functional Requirements

### 1. Endpoint Strategy

> The Market Movers section uses the **`/api/cask-masters` endpoint** with `sortBy=volumeDelta30D` and `sortOrder=desc`.
>
> | Sorting Intent (`sortBy` field) | Sort Direction | Fallback Tie-breakers (Deterministic) |
> |---|---|---|
> | `volumeDelta30D` | DESC | `lifetimeVolume DESC`, then `id ASC` |

---

### 2. Master Cask Response Structure

#### 2.1. Overview

> The endpoint returns the same base card response structure as Explore Casks, augmented with the specific fields needed to render the Market Movers table.

#### 2.2. Formula Reference & Examples

> [!IMPORTANT]
> **Note on Median calculation:** Median is computed at the **unit level** — if a transaction covers 2 units at £10,000 each, both £10,000 values are included in the median dataset. This prevents bulk-buy transactions from being under-represented versus single-unit trades.

> **Example of Unit-Weighted Median:**
> - **Tx 1:** 5 units @ £100
> - **Tx 2:** 1 unit @ £500
> - **Data set for Median:** `[100, 100, 100, 100, 100, 500]`
> - **Result:** **£100** (Correct unit-level) vs £300 (Incorrect transaction-level)

##### 2.2.1. Metric Implementation & Calculation Table

The following table defines the logic for all metrics returned by the API or used for internal ranking/delta computations.

| Metric | Exposed? | Calculation Formula | Dependencies / Filters / Windows |
|:---|:---:|:---|:---|
| `volumeLast30D` | No (internal) | `SUM(checkoutSession.originalAmount)` | Window: `completedAt` within the last 30 days (`checkoutSession.completedAt >= NOW - 30 days`). Filter: session status = `completed`. |
| `volumePrev30D` | No (internal) | `SUM(checkoutSession.originalAmount)` | Window: `completedAt` between -60 and -30 days ago (`checkoutSession.completedAt >= NOW - 60 days AND checkoutSession.completedAt < NOW - 30 days`). Filter: session status = `completed`. |
| `volumeDelta30D` | **Yes (#1)** | `((volumeLast30D - volumePrev30D) / volumePrev30D) * 100` | Returns `null` if `volumePrev30D` is `0` (cold start not supported). |
| `medianPrice` | **Yes (#2)** | `MEDIAN(tx.transactionPrice)` | Unit-weighted. Filter: `tx.status = 'completed'` (all-time). |
| `medianLast30D` | No (internal) | `MEDIAN(tx.transactionPrice)` | Window: `completedAt` within last 30 days. Unit-weighted. Filter: `tx.status = 'completed'`. |
| `medianPrev30D` | No (internal) | `MEDIAN(tx.transactionPrice)` | Window: `completedAt` between -60 and -30 days. Unit-weighted. Filter: `tx.status = 'completed'`. |
| `medianPriceDelta30D` | **Yes (#3)** | `((medianLast30D - medianPrev30D) / medianPrev30D) * 100` | Returns `null` if `medianPrev30D` is `0` or `null`, or if `medianLast30D` is `null`. |
| `lowestAsk` | No (existing base field) | `MIN(ask.price)` | Current minimum active ask price across active variants. |

---

#### 2.3. Response

The existing `/api/cask-masters` endpoint already returns standard master cask properties (including `lowestAsk`, `volumeLast30D`, etc.). The following fields must be **added** to the response to support Market Movers:

| No. | Field | Data Type | Description |
|-----|-------|-----------|-------------|
| 1 | `volumeDelta30D` | number \| null | Percentage change in 30-day volume. Primary sort key. FE hides the entire row if `null` is returned. |
| 2 | `medianPrice` | number \| null | Unit-weighted median transaction price (all-time). |
| 3 | `medianPriceDelta30D` | number \| null | Percentage change in median transaction price over the last 30 days. |

---

### 3. Use Cases

##### UC-1. Fetch Market Movers

| Item | Description |
|------|-------------|
| **Objective** | Return the requested number of master casks ranked by 30-day volume momentum |
| **Actor** | System (triggered by FE) |
| **Trigger** | FE requests Market Movers data on Home page load |
| **Pre-condition(s)** | Master casks exist with active variants and completed transactions |
| **Post-condition(s)** | Returns ranked list matching the requested size, sorted by `volumeDelta30D DESC` (**MM-BR-2**) |

**Happy Path**:

| # | Scenario | Expected Result | Notes |
|---|----------|----------------|-------|
| 1 | Platform has eligible master casks with positive `volumeDelta30D` | Return results sorted by `volumeDelta30D DESC` | Fields resolved per definitions |

**Negative Path**:

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | No eligible master casks exist | Return empty array `[]` | 200 OK |
| 2 | Server error | Return error | 500 Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|-|
| 1 | `volumePrev30D = 0` (resulting in `volumeDelta30D = null`) | Cask is excluded from Market Movers as cold start is not supported. | FE hides the entire cask row if a null delta is returned |
| 2 | Casks have equal `volumeDelta30D` | Apply final tie-break: `lifetimeVolume DESC`, then `id ASC` (**MM-BR-4**). | — |

---

## III. Data Model

### Derived / Pre-computed: MasterCaskMarketStats — Updates

To support the Market Movers sorting on the existing `/api/cask-masters` endpoint, the `MasterCaskMarketStats` derived table (introduced in the Explore Casks feature) must be extended to include 30-day volume and median price metrics.

**New fields required in `MasterCaskMarketStats`:**
- `volumeLast30D`: number
- `volumePrev30D`: number
- `volumeDelta30D`: number | null
- `medianPrice`: number | null
- `medianLast30D`: number | null
- `medianPrev30D`: number | null
- `medianPriceDelta30D`: number | null

---

## IV. Technical Dependencies

### 1. Schema Change Request: SCR-001

The calculation of 30-day volume and median price windows relies on the `completedAt` timestamp on the `CheckoutSession` entity (**SCR-001**). Until this is delivered, `updatedAt` is used as a proxy for completed sessions.

---

## V. Permissions & Access Control

| Action | Anonymous | Buyer | Seller | Admin |
|--------|-----------|-------|--------|-------|
| Load Market Movers | ✅ | ✅ | ✅ | ✅ |

> Publicly accessible by default, aligned with the Explore Casks Home Page endpoints.

---

_End of Document 2_
