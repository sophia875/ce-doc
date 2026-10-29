# BACKEND LOGIC & FLOW SPECS

Top Distilleries – Home Page Curated List

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation and filled all API docs information | Alice | 01/05/2026 |
| 1.0 | Audit pass: unified `estMedianPrice` trigger to `lifetimeVolume = 0` (#1); added tertiary `distilleryId ASC` tie-break (#4); flagged `updatedAt` date-filter risk in SQL (#5); clarified `estMarketValue` variant-level SUM (#6); marked `volumeLast30D`/`volumePrev30D` as internal computation fields (#8); added `lastRefreshedAt` to payload (#9/#11); added edge case for TD-BR-4 empty distillery exclusion (#10); fixed `TD-BR-2` Tier 2 `NULL LAST` omission (#11) | Alice | 07/05/2026 |
| 1.1 | Audit & Standardize: clarified Tiered Ranking vs. Tie-break separation; consolidated/reordered BRs; hardened SCR-001/002 dependencies; removed redundant capacity/constraint rules into definitions. | Alice | 11/05/2026 |
| 1.2 | Update top distilleries count to 8 (instead of 5) | Alice | 14/05/2026 |

---

## I. Introduction

### 1. Purpose

> Provide a ranked leaderboard of the top 8 distilleries on the platform, ranked by a **Tiered Ranking Strategy** (see **TD-BR-2**) prioritizing Lifetime Trading Volume for Tier 1, with **Estimated Market Value** (`estMarketValue DESC`, **NULL LAST**) used as the primary ranking signal for Tier 2 (cold-start) entities. Alongside the ranking metric, the API returns median pricing and 30-day delta signals to give buyers quick market context per distillery. Any ties across both tiers are resolved by active master cask count and distillery ID (**TD-BR-3**).

> **Note on Data Model Naming Conventions:** Throughout this document, data field references (e.g., `transaction.transactionPrice`, `checkoutSession.originalAmount`) are written in a **conceptual, singular form**. Developers should interpret these references conceptually.

### 2. Definitions

| Term | Definition |
|------|------------|
| Distillery | A producer entity. Multiple Master Casks (parents) can belong to one Distillery |
| Master Cask (Parent) | A parent entity grouping multiple variant casks. Belongs to exactly one Distillery |
| Variant Cask (Child) | A specific vintage of a Master Cask. Transactions occur at the variant level |
| Active Ask | An ask where `ask.status = 'active'` AND `ask.expirationDate > NOW()`. Shared definition across all features (e.g. Explore Casks). |
| Successful Transaction | A completed trade where `checkoutSession.status = 'completed'`. |
| Lifetime Volume | `SUM(cs.originalAmount)` across all completed checkout sessions linked to variant casks of a distillery. |
| Transaction Price (`tx.transactionPrice`) | The agreed per-unit price of a single cask in a completed transaction. Source: `transaction.transactionPrice` |
| Session Original Amount (`cs.originalAmount`) | Total agreed trade value for the checkout session: `transactionPrice × quantity`. Source: `checkoutSession.originalAmount` |
| Median Price | The median of `tx.transactionPrice` values (**unit-weighted per TD-BR-4**) across all completed transactions for a distillery, all time. |
| 30D Volume Delta | Percentage change in total transaction volume: comparing last 30 days vs. prior 30 days. Requires **SCR-001**. |
| 30D Median Price Delta | Percentage change in median transaction price: comparing last 30 days vs. prior 30 days. Requires **SCR-001**. |
| Master Cask Count | Total number of **active** Master Casks linked to the distillery. Requires **SCR-002**. |
| Tiered Ranking Strategy | The primary sorting mechanism that prioritizes active markets (Tier 1) over estimated markets (Tier 2). |
| Deterministic Tie-break | The secondary/tertiary fallback sorting used within a tier to ensure stable results when primary metrics are equal. |

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| **TD-BR-1** | **Exclusion: Empty Brands** | Distilleries with `lifetimeVolume = 0` AND `masterCaskCount = 0` MUST be excluded from the ranking list. This ensures the leaderboard only shows distilleries with either trade history or active product inventory. |
| **TD-BR-2** | **Tiered Ranking Strategy** | If Tier 1 (`lifetimeVolume > 0`) has fewer than 8 entries, fill remaining slots with Tier 2 (`lifetimeVolume = 0`). Tier 1 is ranked by `lifetimeVolume DESC`. Tier 2 is ranked by `estMarketValue DESC` (**NULL LAST**). |
| **TD-BR-3** | **Deterministic Tie-break** | If primary metrics within a tier are equal, rank by **active Master Cask count** (**SCR-002**) DESC, then **Distillery ID** ASC. Final tie-break MUST be deterministic and not rely on database default. |
| **TD-BR-4** | **Unit-Weighted Median** | Median price calculations must be unit-weighted (weighted by `tx.quantity`). If a transaction covers 2 units at £10,000 each, both values enter the median dataset. |
| **TD-BR-5** | **Historical Integrity** | All-time metrics (`lifetimeVolume`, `medianPrice`) include ALL completed transactions. This includes transactions from distilleries/master casks that may currently be marked as `inactive` (**SCR-002**). |

---

## II. Functional Requirements

### 1. Top Distilleries Ranked List

> Returns the top 8 distilleries ranked by **Lifetime Volume** (**TD-BR-2**) using a **Tiered Ranking Strategy**. Tier 1 (traded) is prioritized by volume, with Tier 2 (cold-start) serving as fill-ins ranked by **Estimated Market Value** (**NULL LAST**). Any ties are resolved via a **Deterministic Tie-break** (**TD-BR-3**). This requires a new dedicated endpoint, as the existing `top-ranked` API currently sorts by master cask count rather than volume.

#### 1.2. Data Description

The API should return the following payload for each distillery. This includes fields directly mapped to UI components (e.g., `lifetimeVolume`, `medianPrice`), fallback metrics for cold starts (e.g., `estMedianPrice`), and underlying data points used to compute the 30-day delta badges:

| No. | Field | Data Type | Description |
|-----|-------|-----------|-------------|
| 1 | distilleryId | string | Distillery entity ID |
| 2 | distilleryName | string | Display name |
| 3 | distilleryImageUrl | string \| null | URL to distillery logo |
| 4 | masterCaskCount | number | Count of Master Casks where `status = 'active'`. (Requires **SCR-002**). |
| 5 | rank | number | Ordinal rank (1–8). |
| 6 | lifetimeVolume | number | `SUM(cs.originalAmount)` for `completed` sessions. Default `0` if no transactions. |
| 7 | medianPrice | number \| null | `MEDIAN(tx.transactionPrice)` (unit-weighted per **TD-BR-4**). `null` if no completed transactions exist. |
| 8 | estMarketValue | number \| null | **Tier 2 only.** `SUM((referencePriceMin + referencePriceMax) / 2)` across all **active variant casks** (one midpoint per variant). `null` if no active variant casks exist. |
| 9 | estMedianPrice | number \| null | **Tier 2 only.** `MEDIAN((referencePriceMin + referencePriceMax) / 2)` across all active variant casks. `null` if no active variant casks exist. |
| 10 | volumeDelta30D | number \| null | `(volumeLast30D - volumePrev30D) / volumePrev30D × 100`. Requires **SCR-001**. `null` if `volumePrev30D = 0`. |
| 11 | medianPriceDelta30D | number \| null | `(medianLast30D - medianPrev30D) / medianPrev30D × 100`. Requires **SCR-001**. `null` if `medianPrev30D = 0` or `null`, or `medianLast30D` is `null`. |
| 12 | lastRefreshedAt | datetime | Timestamp of when this distillery's stats were last recomputed. Allows FE to display data freshness indicator if needed. |

#### 1.3. Formula Reference & Examples

> [!IMPORTANT]
> **Note on Median calculation:** Median is computed at the **unit level** — if a transaction covers 2 units at £10,000 each, both £10,000 values are included in the median dataset. This prevents bulk-buy transactions from being under-represented versus single-unit trades.

> **Example of Unit-Weighted Median:**
> - **Tx 1:** 5 units @ £100
> - **Tx 2:** 1 unit @ £500
> - **Data set for Median:** `[100, 100, 100, 100, 100, 500]`
> - **Result:** **£100** (Correct unit-level) vs £300 (Incorrect transaction-level)

##### 1.3.1. Metric Implementation & Calculation Table

The following table defines the logic for all metrics returned by the API or used for internal ranking/delta computations.

| Metric | Exposed? | Calculation Formula | Dependencies |
|:---|:---:|:---|:---|
| **`lifetimeVolume`** | **Yes** (#6) | `SUM(cs.originalAmount)` | Filter: `status = 'completed'`. |
| **`medianPrice`** | **Yes** (#7) | `MEDIAN(tx.transactionPrice)` | Unit-weighted (**TD-BR-4**). |
| **`estMarketValue`** | **Yes** (#8) | `SUM((refMin + refMax) / 2)` | Tier 2 fallback. Active variants only (**SCR-002**). |
| **`estMedianPrice`** | **Yes** (#9) | `MEDIAN((refMin + refMax) / 2)` | Tier 2 fallback. Active variants only (**SCR-002**). |
| `volumeLast30D` | No | `SUM(cs.originalAmount)` | Window: `cs.completedAt` (**SCR-001**). |
| `volumePrev30D` | No | `SUM(cs.originalAmount)` | Window: `cs.completedAt` (**SCR-001**). |
| **`volumeDelta30D`** | **Yes** (#10) | `(vLast - vPrev) / vPrev × 100` | Requires **SCR-001**. |
| `medianLast30D` | No | `MEDIAN(tx.transactionPrice)` | Window: `cs.completedAt` (**SCR-001**). Unit-weighted. |
| `medianPrev30D` | No | `MEDIAN(tx.transactionPrice)` | Window: `cs.completedAt` (**SCR-001**). Unit-weighted. |
| **`medianPriceDelta30D`**| **Yes** (#11) | `(mLast - mPrev) / mPrev × 100` | Requires **SCR-001**. |
| **`lastRefreshedAt`** | **Yes** (#12) | `NOW()` | Aggregation job timestamp. |

> [!NOTE]
> **Active Variant Definition:** A variant is active if its parent `MasterCask.status = 'active'` AND its own `caskStatus = 'active'`. (**SCR-002**)



---

## III. Use Cases

### 1. Fetch Top 8 Distilleries (with Cold Start Support)

| Item | Description |
|------|-------------|
| **Objective** | Return the top 8 distilleries using the **Tiered Ranking Strategy** (TD-BR-2) and **Deterministic Tie-break** (TD-BR-3) |
| **Actor** | System |
| **Trigger** | FE requests Top Distilleries data |
| **Pre-condition(s)** | Distilleries exist in the system with active master casks |
| **Post-condition(s)** | Returns ranked list (max 8) with volume/median price or estimated values |

#### 1.1. Happy Path

| # | Scenario | Expected Result | Notes |
|---|----------|----------------|-------|
| 1 | Platform has ≥ 8 eligible distilleries with `lifetimeVolume > 0` | Return top 8 sorted by **Tier 1** (`lifetimeVolume DESC`) | Fields resolved per Section I.2 definitions |
| 2 | Platform has mix of distilleries with `lifetimeVolume > 0` and `lifetimeVolume = 0` | Return up to 8 distilleries: Tier 1 ranked by `lifetimeVolume DESC`, then Tier 2 fill-ins ranked by `estMarketValue DESC` (**NULL LAST**). (Traded distilleries will naturally appear first, while new ones are ranked among themselves by estimated value). | |

#### 1.2. Negative Path

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | No eligible distilleries exist | Return empty array `[]` | 200 OK |
| 2 | Server error | Return error | 500 Internal Server Error |

#### 1.3. Edge Cases

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|-|
| 1 | All eligible distilleries have `lifetimeVolume = 0` | Return up to 8, sorted by **Tier 2 Primary Sort** (`estMarketValue DESC`, NULL LAST), then **Universal Tie-break**: `masterCaskCount DESC`, then `distilleryId ASC`. | FE renders cards with `—` in volume and delta fields; shows estimated values instead |
| 2 | A distillery has `lifetimeVolume = 0` AND `masterCaskCount = 0` | **Exclude** from results per **TD-BR-1**. This distillery has no market signal or active product. | - |
| 3 | Fewer than 8 eligible distilleries exist | Return all eligible distilleries | FE renders available cards |
| 4 | Two distilleries have equal `lifetimeVolume` AND equal `masterCaskCount` | Apply final tie-break: `distilleryId ASC`. **Must not rely on database default.** | - |

---

## IV. Technical Dependencies

The implementation of specific metrics and sorting logic requires the following backend components and schema changes.

### 1. Schema Change Request: SCR-001

The calculation of 30-day delta metrics (Fields #10–#11) depends on the implementation of **SCR-001**.

| Item | Detail |
|------|--------|
| **Scope** | Add `completedAt` timestamp to the `CheckoutSession` entity. |
| **Status** | 🟡 Pending (Requires Backend implementation). |
| **Fallback Behavior** | Until SCR-001 is delivered, the system will use **`updatedAt`** as a temporary proxy for `completedAt` (filtered for sessions where `status = 'completed'`). Note: This may cause minor inaccuracies if a completed session is updated multiple times after completion. |
| **Reference** | See [schema-change-requests.md](../data-models/schema-change-requests.md) for full technical spec. |

### 2. Schema Change Request: SCR-002

The **`masterCaskCount`** calculation and cascading availability logic depend on **SCR-002**.

| Item | Detail |
|------|--------|
| **Scope** | Add `status` (`active` / `inactive`) to the **Master Cask** entity. |
| **Status** | 🟢 Approved (System enhancement). |
| **Logic** | **Master Status overrides Variant Status.** If a Master Cask is `inactive`, all its child variant casks must be hidden from Browse/Search lists and excluded from **Active counts** (`masterCaskCount`) and **Estimation metrics** (`est*`). **Historical metrics** (`lifetimeVolume`, `medianPrice`, and deltas) are **NOT** affected and continue to include all past completed transactions. |
| **Fallback** | Until SCR-002 is delivered, the system will use the derived logic: "Active if ≥1 child has `caskStatus = 'active'`". |

---

## V. Data Model

### Entity: Distillery — existing (reference)
- `id`: string (UUID)
- `name`: string
- `imageUrl`: string | null
- `status`: string (`active` / `inactive`)

### Entity: Master Cask — existing (reference)
- `id`: string (UUID)
- `name`: string
- `distilleryId`: string (FK)
- `status`: string (`active` / `inactive`) — **New requested field (SCR-002)**.

### Entity: CheckoutSession — existing (reference)
- `id`: string (UUID)
- `originalAmount`: number — total agreed trade value for the session. Source of `lifetimeVolume`.
- `status`: string (`completed`, `pending`, etc.) — filter to `'completed'` for volume calculations.
- `completedAt`: datetime | null — **New requested field (SCR-001)**. Timestamp recorded exactly once when status transitions to `'completed'`. Primary boundary for 30D window calculations.
- `ownershipTransferredAt`: datetime | null — timestamp of the ownership document transfer step. **No longer used** as the primary 30D window boundary.
- `updatedAt`: datetime — last modification timestamp. **Not used** for 30D window calculations.

### Entity: Transaction — existing (reference)
- `transactionPrice`: number — agreed per-unit price. Source of `medianPrice`.
- `quantity`: number — units in the transaction. Used for unit-weighted median (**TD-BR-4**).
- `status`: string (`completed`, `refunded`, etc.)
- `updatedAt`: datetime — last modification timestamp.

### Derived / Pre-computed: DistilleryVolumeStats — new (recommended)
- `distilleryId`: string (UUID)
- `lifetimeVolume`: number
- `masterCaskCount`: number
- `medianPrice`: number | null
- `estMarketValue`: number | null
- `estMedianPrice`: number | null
- `volumeDelta30D`: number | null
- `medianPriceDelta30D`: number | null
- `lastRefreshedAt`: string (ISO)

---

## VI. Permissions & Access Control
- **Load Top Distilleries List:** Restricted to `Buyer`, `Seller`, `Admin`. Anonymous access is **DENIED** (401).

---

## VII. Performance Considerations
1. **Pre-computation:** Data is served from a `DistilleryVolumeStats` table. A scheduled job refreshes the full table every 15–30 minutes.
2. **Event-Driven Refresh:** To prevent "rank flips" and stale market signals, an asynchronous refresh of a specific distillery's stats MUST be triggered upon:
   - Any session transition to `completed` (setting or updating `completedAt` per **SCR-001**).
   - Any admin update to `masterCask.status` (per **SCR-002**).
3. **Median Optimization:** Use unit-weighted pre-calculation (**TD-BR-4**) during the stats aggregation job.

---

_End of Document 2_
