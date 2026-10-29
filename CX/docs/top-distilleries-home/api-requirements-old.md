# BACKEND LOGIC & FLOW SPECS

Top Distilleries – Home Page Curated List

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | Alice | 29/04/2026 |
| 0.2 | Updated access control — restricted to authenticated users only (aligned with UI spec) | Alice | 29/04/2026 |
| 0.3 | Corrected Volume and Median Price formulas; added Price Field Glossary | Alice | 29/04/2026 |
| 0.4 | Added Cold Start fallbacks (`estMarketValue`, `estMedianPrice`) using `(referencePriceMin + referencePriceMax) / 2` per variant; added TD-BR-5 and TD-BR-11 | Alice | 29/04/2026 |
| 0.5 | Removed range sub-label from card display (cross-vintage range too wide to be informative) | Alice | 29/04/2026 |
| 0.6 | Standardized on TypeScript types; replaced `completedAt` with `updatedAt`; refined Cold Start logic in Use Cases; added TD-BR-12 (Unit-level Median); removed `activeVariantCount` and Price Field Glossary sections | Alice | 01/05/2026 |

---

## I. Introduction

### 1. Purpose

> Provide a ranked leaderboard of the top 5 distilleries on the platform, ranked by their **Lifetime Trading Volume** — the total monetary value of all successfully completed cask transactions across all variant casks belonging to the distillery. A secondary tie-breaker uses active master cask count. The API also returns median pricing, 30-day delta signals, and Cold Start fallback estimation fields to give buyers quick market context per distillery.

> **Note on Data Model Naming Conventions:** Throughout this document, data field references (e.g., `transaction.transactionPrice`, `checkoutSession.originalAmount`) are written in a **conceptual, singular form**. Developers should interpret these references conceptually.

### 2. Definitions

| Term | Definition |
|------|------------|
| Distillery | A producer entity. Multiple Master Casks (parents) can belong to one Distillery |
| Master Cask (Parent) | A parent entity grouping multiple variant casks. Belongs to exactly one Distillery |
| Variant Cask (Child) | A specific vintage of a Master Cask. Transactions occur at the variant level |
| Successful Transaction | A completed trade where `checkoutSession.status = 'completed'`. Excludes `pending`, `agreement_signed`, `invoice_submitted`, `cancelled`, `failed` |
| Lifetime Volume | `SUM(cs.originalAmount)` across all completed checkout sessions linked to variant casks of a distillery. `originalAmount = transactionPrice × quantity` — the agreed market value before platform fees |
| Transaction Price (`tx.transactionPrice`) | The agreed per-unit price of a single cask in a completed transaction. Source: `transaction.transactionPrice` |
| Session Original Amount (`cs.originalAmount`) | Total agreed trade value for the checkout session: `transactionPrice × quantity`. Excludes all platform fees. Source: `checkoutSession.originalAmount` |
| Median Price | The median of `tx.transactionPrice` values (**unit-weighted per TD-BR-12**) across all completed transactions for a distillery, all time |
| Cold Start | State where a distillery has `lifetimeVolume = 0` or `null` (no completed trades). Estimation fields (`estMarketValue`, `estMedianPrice`) are returned instead |
| 30D Volume Delta | Percentage change in total transaction volume: comparing last 30 days vs. prior 30 days |
| 30D Median Price Delta | Percentage change in median transaction price: comparing last 30 days vs. prior 30 days |
| Master Cask Count | Total number of **active** Master Casks linked to the distillery. Used as a tie-breaker |

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| TD-BR-1 | Distilleries are ranked by Lifetime Volume DESC | `lifetimeVolume = SUM(cs.originalAmount)` for all `cs.status = 'completed'` sessions linked to variant casks of the distillery |
| TD-BR-2 | Tie-break: If two distilleries have equal Lifetime Volume, rank by **active** Master Cask count DESC | `masterCaskCount = COUNT(DISTINCT masterCaskId)` where `status = 'active'` |
| TD-BR-3 | Only completed checkout sessions count toward volume | `checkoutSession.status = 'completed'` only |
| TD-BR-4 | The list is capped at top 5 distilleries | Fixed limit of 5 |
| TD-BR-5 | Distilleries with zero lifetime volume CAN be included (Cold Start support) | Distilleries with `lifetimeVolume = 0` are ranked below those with volume, sorted among themselves by `masterCaskCount DESC`. Included so the section can still be shown |
| TD-BR-6 | Median Price uses `tx.transactionPrice` (per-unit agreed price) | `MEDIAN(tx.transactionPrice)` across all completed transactions, unit-weighted (**TD-BR-12**) |
| TD-BR-7 | 30D Volume Delta formula | `(volumeLast30D - volumePrev30D) / volumePrev30D × 100`. If `volumePrev30D = 0` or `null`, delta is `null` |
| TD-BR-8 | 30D Median Price Delta formula | `(medianLast30D - medianPrev30D) / medianPrev30D × 100`. If `medianPrev30D = 0` or `null`, or if `medianLast30D` is `null`, delta is `null` |
| TD-BR-9 | All-time volume and median use ALL completed transactions with no date filter | Historical data is fully included |
| TD-BR-10 | Master Cask Count includes only **active** master casks linked to the distillery | |
| TD-BR-11 | Cold Start Fallback: when `lifetimeVolume = 0` or `null`, compute estimation metrics from `referencePriceMin` and `referencePriceMax` of active variant casks | `estMarketValue = SUM((referencePriceMin + referencePriceMax) / 2)` across all active variant casks. `estMedianPrice = MEDIAN((referencePriceMin + referencePriceMax) / 2)` across all active variant casks. If no active variant casks exist, return `null` for all `est*` fields |
| TD-BR-12 | Unit-level Median Weighting | Median calculations must be unit-weighted (weighted by `tx.quantity`). Example: if a transaction covers 2 units at £10,000 each, both £10,000 values enter the median dataset. This prevents bulk-buy transactions from being under-represented versus single-unit trades |

---

## II. Requirements

### 1. Top Distilleries Ranked List

#### 1.1. Overview

> Returns the top 5 distilleries ranked by their lifetime transaction volume. Includes Cold Start fallback logic to show active distilleries even if they have no trade history yet.

#### 1.2. Data Description

| No. | Field | Data Type | Description |
|-----|-------|-----------|-------------|
| 1 | distilleryId | string | Distillery entity ID |
| 2 | distilleryName | string | Display name of the distillery (e.g., "Macallan") |
| 3 | distilleryImageUrl | string \| null | URL to distillery logo/image. `null` if none set |
| 4 | masterCaskCount | number | Total count of **active** Master Casks linked to this distillery. Used as tie-breaker in ranking |
| 5 | rank | number | Ordinal rank (1–5). Computed server-side based on `lifetimeVolume` DESC, then `masterCaskCount` DESC |
| 6 | lifetimeVolume | number \| null | `SUM(cs.originalAmount)` for all `cs.status = 'completed'` sessions for this distillery, all time. `null` if no completed sessions exist |
| 7 | medianPrice | number \| null | `MEDIAN(tx.transactionPrice)` — unit-weighted per TD-BR-12, across all completed transactions, all time. `null` if no completed transactions exist |
| 8 | estMarketValue | number \| null | **Cold Start fallback only.** `SUM((referencePriceMin + referencePriceMax) / 2)` across all active variant casks. Returned when `lifetimeVolume` is `0` or `null`. `null` if no active variant casks exist |
| 9 | estMedianPrice | number \| null | **Cold Start fallback only.** `MEDIAN((referencePriceMin + referencePriceMax) / 2)` across all active variant casks. Returned when `medianPrice` is `null`. `null` if no active variant casks exist |
| 10 | volumeLast30D | number \| null | `SUM(cs.originalAmount)` for sessions with `cs.status = 'completed'` AND `cs.updatedAt >= (NOW - 30 days)`. Internal field used to compute `volumeDelta30D` |
| 11 | volumePrev30D | number \| null | `SUM(cs.originalAmount)` for sessions with `cs.status = 'completed'` AND `cs.updatedAt >= (NOW - 60 days) AND cs.updatedAt < (NOW - 30 days)`. Internal field used to compute `volumeDelta30D` |
| 12 | volumeDelta30D | number \| null | `(volumeLast30D - volumePrev30D) / volumePrev30D × 100`. Null if `volumePrev30D = 0` or `null` |
| 13 | medianLast30D | number \| null | Median of `tx.transactionPrice` (**unit-weighted per TD-BR-12**) for transactions with `cs.status = 'completed'` AND `cs.updatedAt >= (NOW - 30 days)`. Null if no transactions in window |
| 14 | medianPrev30D | number \| null | Median of `tx.transactionPrice` (**unit-weighted per TD-BR-12**) for transactions with `cs.status = 'completed'` AND `cs.updatedAt >= (NOW - 60 days) AND cs.updatedAt < (NOW - 30 days)`. Null if no transactions in window |
| 15 | medianPriceDelta30D | number \| null | `(medianLast30D - medianPrev30D) / medianPrev30D × 100`. Null if `medianPrev30D = 0` or `null`, or if `medianLast30D` is `null` |

#### 1.3. Formula Reference & Examples

> [!IMPORTANT]
> **Note on Median calculation:** Median is computed at the **unit level** — if a transaction covers 2 units at £10,000 each, both £10,000 values are included in the median dataset. This prevents bulk-buy transactions from being under-represented versus single-unit trades.

> **Example of Unit-Weighted Median:**
> - **Tx 1:** 5 units @ £100
> - **Tx 2:** 1 unit @ £500
> - **Data set for Median:** `[100, 100, 100, 100, 100, 500]`
> - **Result:** **£100** (Correct unit-level) vs £300 (Incorrect transaction-level)

##### 1.3.1. Logic Reference (Internal Calculations)

```sql
-- 1. All-time Volume (lifetimeVolume)
SUM(cs.originalAmount) WHERE cs.status = 'completed'

-- 2. All-time Median (medianPrice) — unit-weighted per TD-BR-12
MEDIAN(tx.transactionPrice) WHERE cs.status = 'completed'

-- 3. 30D Volume Windows
volumeLast30D = SUM(cs.originalAmount) WHERE cs.status = 'completed' AND cs.updatedAt >= (NOW - 30 days)
volumePrev30D = SUM(cs.originalAmount) WHERE cs.status = 'completed' AND cs.updatedAt >= (NOW - 60 days) AND cs.updatedAt < (NOW - 30 days)

-- 4. 30D Median Price Windows (Unit-Weighted per TD-BR-12)
medianLast30D = MEDIAN(tx.transactionPrice) WHERE cs.status = 'completed' AND cs.updatedAt >= (NOW - 30 days)
medianPrev30D = MEDIAN(tx.transactionPrice) WHERE cs.status = 'completed' AND cs.updatedAt >= (NOW - 60 days) AND cs.updatedAt < (NOW - 30 days)

-- 5. Cold Start Fallbacks (when lifetimeVolume = 0 or null)
estMarketValue = SUM((referencePriceMin + referencePriceMax) / 2) WHERE variantCask.caskStatus = 'active'
estMedianPrice = MEDIAN((referencePriceMin + referencePriceMax) / 2) WHERE variantCask.caskStatus = 'active'
-- If no active variant casks exist, return null for all est* fields
```

##### 1.3.2. Delta Formulas

- `volumeDelta30D` = `(volumeLast30D - volumePrev30D) / volumePrev30D × 100` — Null if `volumePrev30D = 0` or `null`
- `medianPriceDelta30D` = `(medianLast30D - medianPrev30D) / medianPrev30D × 100` — Null if `medianPrev30D = 0` or `null`, or if `medianLast30D` is `null`

#### 1.4. Use Cases

##### UC-1. Fetch Top 5 Distilleries (with Cold Start Support)

| Item | Description |
|------|-------------|
| **Objective** | Return the top 5 distilleries ranked by lifetime volume, with `est*` fallback fields for distilleries with no completed trade history |
| **Actor** | System (triggered by FE on Home page load) |
| **Trigger** | FE requests Top Distilleries data on Home page load |
| **Pre-condition(s)** | At least one distillery exists in the system with active master casks |
| **Post-condition(s)** | Returns up to 5 distilleries with rank, volume/median price (or `est*` fallback values), and 30D delta fields |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Aggregate Lifetime Volume per Distillery | TD-BR-1, TD-BR-3 | Compute `lifetimeVolume = SUM(cs.originalAmount)` for all `cs.status = 'completed'` sessions. Distilleries with 0 volume are **included** (not excluded) |
| 2 | Count Active Master Casks per Distillery | TD-BR-2, TD-BR-10 | Count `masterCaskCount = COUNT(DISTINCT masterCaskId)` where `status = 'active'` for each distillery |
| 3 | Rank and Limit | TD-BR-4, TD-BR-2, TD-BR-5 | Sort by `lifetimeVolume DESC`, then by `masterCaskCount DESC`. Distilleries with `lifetimeVolume = 0` rank below those with volume but are still included. Take `LIMIT 5` |
| 4 | Compute Median Price | TD-BR-6, TD-BR-12 | Compute `medianPrice` across all completed transactions (unit-weighted). `null` if no completed transactions |
| 5 | Compute Cold Start Fallbacks | TD-BR-11 | If `lifetimeVolume = 0` or `null`: compute `estMarketValue` and `estMedianPrice` from active variant casks. If no active variants exist, return `null` for all `est*` fields |
| 6 | Compute 30D Window Metrics | TD-BR-7, TD-BR-8 | Compute `volumeLast30D`, `volumePrev30D`, `volumeDelta30D`, `medianLast30D`, `medianPrev30D`, `medianPriceDelta30D` |
| 7 | Assemble Response | — | Map all fields to response payload. Distilleries with 0 volume must return `est*` fields (not `null`) if active variants exist |

**Happy Path**:

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Platform has ≥ 5 distilleries with completed transactions | Return exactly 5 distilleries, ranked 1–5 by lifetime volume |
| 2 | Platform has < 5 distilleries with completed transactions | Return all qualifying distilleries (< 5 items) |
| 3 | Some distilleries have 0 volume (new/Cold Start) | Distilleries with volume ranked first, then Cold Start distilleries by `masterCaskCount DESC`; `est*` fields populated |
| 4 | New platform — no completed transactions exist | Return top 5 by `masterCaskCount DESC`; all entries show only `est*` fields |

**Negative Path**:

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | No distilleries exist in the system at all | Return empty array `[]` | 200 OK — FE hides section |
| 2 | Distilleries exist but have no active master casks | Return empty array `[]` | 200 OK — FE hides section |
| 3 | Unexpected server error during aggregation | Return error status | 500 — Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 1 | Two or more distilleries have identical `lifetimeVolume` | Tie-break by `masterCaskCount DESC` |
| 2 | A distillery has `lifetimeVolume > 0` but `volumePrev30D = 0` | `volumeDelta30D = null`. FE displays `— (30D)` |
| 3 | A distillery has only 1 completed transaction | `medianPrice` = that single transaction price. Delta fields may be null if insufficient window data |
| 4 | Cold Start distillery has no active variant casks | Return `null` for all `est*` fields |
| 5 | A transaction is refunded after completion | Exclude from all aggregations if `status` changes from `completed` to `refunded` |

---

## III. Data Model

### Entity: Distillery — existing (reference)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | string (UUID) | ✅ | auto-generated | Primary key |
| name | string | ✅ | — | Distillery display name |
| imageUrl | string \| null | ❌ | null | Distillery logo/image |
| status | string | ✅ | `active` | `active` / `inactive` |
| createdAt | string (ISO) | ✅ | NOW() | Record creation timestamp |
| updatedAt | string (ISO) | ✅ | NOW() | Last update timestamp |

### Entity: Transaction — existing (reference)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | string (UUID) | ✅ | Primary key |
| variantCaskId | string (UUID) | ✅ | FK to VariantCask |
| transactionPrice | number | ✅ | Per-unit agreed price. Used for `medianPrice` calculations |
| quantity | number | ✅ | Number of units. Used for unit-weighted median per **TD-BR-12** |
| status | string | ✅ | `pending` / `completed` / `cancelled` / `failed` / `refunded` |
| updatedAt | string (ISO) | ✅ | Used for 30D window calculations |

> **Key join path**: `Transaction → VariantCask (via variantCaskId) → MasterCask (via masterId) → Distillery (via distilleryId)`

### Entity: CheckoutSession — existing (reference)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | string (UUID) | ✅ | Primary key |
| status | string | ✅ | `pending` / `agreement_signed` / `invoice_submitted` / `completed` / `cancelled` / `failed` |
| originalAmount | number | ✅ | Agreed trade value before fees (`transactionPrice × quantity`). Used for **volume** calculations |
| updatedAt | string (ISO) | ✅ | Timestamp when session status was last updated. Used for 30D window calculations |

### Derived / Pre-computed: DistilleryVolumeStats — new (recommended)

To avoid expensive real-time aggregation, a pre-computed stats table (or materialized view) is recommended:

| Field | Type | Description |
|-------|------|-------------|
| distilleryId | string (UUID) | FK to Distillery |
| lifetimeVolume | number \| null | Pre-computed `SUM(cs.originalAmount)` for all completed sessions |
| masterCaskCount | number | Pre-computed COUNT of active master casks |
| medianPrice | number \| null | Pre-computed unit-weighted median transaction price (all time) |
| estMarketValue | number \| null | Cold Start: `SUM((referencePriceMin + referencePriceMax) / 2)` across active variants |
| estMedianPrice | number \| null | Cold Start: `MEDIAN((referencePriceMin + referencePriceMax) / 2)` across active variants |
| volumeLast30D | number \| null | Volume in last 30 days |
| volumePrev30D | number \| null | Volume in prior 30-day window |
| volumeDelta30D | number \| null | Pre-computed delta |
| medianLast30D | number \| null | Unit-weighted median in last 30 days |
| medianPrev30D | number \| null | Unit-weighted median in prior 30-day window |
| medianPriceDelta30D | number \| null | Pre-computed delta |
| lastRefreshedAt | string (ISO) | Timestamp of last aggregation refresh |

---

## IV. Permissions & Access Control

| Action / Flow | Anonymous | Buyer | Seller | Admin |
|---------------|-----------|-------|--------|-------|
| Load Top Distilleries List | ❌ | ✅ | ✅ | ✅ |

> **Authentication required.** Only authenticated users (Buyer, Seller, Admin) may load the Top Distilleries list. Anonymous/unauthenticated requests must be rejected (401).

---

## V. Performance Considerations

| # | Concern | Recommendation |
|---|---------|----------------|
| 1 | Lifetime volume aggregation joins 4 entities across potentially millions of transaction rows | **Pre-compute** and store in a `DistilleryVolumeStats` table or materialized view. Refresh every 15–30 minutes via background job |
| 2 | Unit-weighted median calculation is expensive at scale | Pre-compute and store in `DistilleryVolumeStats` |
| 3 | 30D window deltas require time-bounded aggregation | Pre-compute all window fields in the background refresh job |
| 4 | On new transaction `completed` event | Trigger async update to `DistilleryVolumeStats` for the affected distillery |
| 5 | On transaction status change from `completed` → `refunded` | Trigger recomputation for the affected distillery's stats record |

---

_End of Document 2_
