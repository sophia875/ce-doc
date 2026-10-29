# BACKEND LOGIC & FLOW SPECS

Browse by Category – Home Page Curated List

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation and filled all API docs information | Alice | 08/06/2026 |
| 0.2 | Closed Trending signal decision (volume-only); added `CategoryMetadata` admin entity reference; cross-linked category-metadata-admin.md | Alice | 08/06/2026 |
| 0.3 | Corrected display label: `categoryLabel` (e.g., "Single Malt") is the card display string, NOT `classificationLabel` ("Single Malt Scotch"). Aggregation key remains `classification` slug. | Alice | 08/06/2026 |

---

## I. Introduction

### 1. Purpose

> Provide a complete list of all active cask classifications available on the platform, enriched with market signals to help buyers quickly evaluate and navigate classification-level opportunities. Each classification entry is returned with median transaction pricing (including estimated pricing for cold-start markets without transactions), 30-day price delta, lifetime volume, and a **Trending** indicator computed server-side from recent trading momentum.
> This endpoint returns **all qualified classifications** with no hard limit, sorted by the Tiered Ranking Strategy.

### 2. Definitions

| Term | Definition |
|------|------------|
| Classification | A grouping of cask variants, identified by a `classification` slug (e.g., `single_malt`). Each classification has two label fields: `categoryLabel` (short display label, e.g., "Single Malt" — used on the card) and `classificationLabel` (full label, e.g., "Single Malt" — internal). |
| Active Classification | A classification is considered active and qualified to be returned by this endpoint if it has either historical trade volume (`lifetimeVolume > 0`) OR at least one active Master Cask (`masterCaskCount > 0`). Empty classifications are excluded. |
| Classification (slug) | Internal enum value identifying the classification (e.g., `single_malt`). Used as the aggregation key and the browse filter key. Source of truth: `GET /api/cask-metadata/classifications`. |
| Category Label (`categoryLabel`) | Short, user-facing display name shown on the Browse by Category card (e.g., "Single Malt"). Lives on the Master Cask entity. All master casks within a classification share the same `categoryLabel`. |
| Classification Label (`classificationLabel`) | Longer, more specific display name (e.g., "Single Malt Scotch"). **Not displayed on the Browse by Category card.** Used internally and in other contexts (e.g., cask detail pages). |
| Master Cask (Parent) | A parent entity grouping multiple variant casks. Belongs to exactly one classification. |
| Variant Cask (Child) | A specific vintage of a Master Cask. Transactions occur at the variant level. |
| Active Ask | An ask where `ask.status = 'active'` AND `ask.expirationDate > NOW()`. |
| Successful Transaction | A completed trade where `checkoutSession.status = 'completed'`. |
| Lifetime Volume | `SUM(cs.originalAmount)` across all completed checkout sessions linked to variant casks of a classification. |
| Transaction Price (`tx.transactionPrice`) | The agreed per-unit price of a single cask in a completed transaction. |
| Session Original Amount (`cs.originalAmount`) | Total agreed trade value for the checkout session: `transactionPrice × quantity`. |
| Median Price | The median of `tx.transactionPrice` values (unit-weighted per **BC-BR-4**) across all completed transactions for a classification, all time. |
| 30D Median Price Delta | Percentage change in median transaction price: comparing last 30 days vs. prior 30 days. Requires **SCR-001**. |
| Active Master Cask Count | Total number of active Master Casks with `status = 'active'` belonging to a classification. Requires **SCR-002**. |
| Trending | A classification is considered Trending when its recent trading activity shows a significant positive momentum. See **BC-BR-6** for the exact formula. |
| Tiered Ranking Strategy | The primary sorting mechanism that prioritizes active markets (Tier 1) over estimated markets (Tier 2). |
| Deterministic Tie-break | The secondary/tertiary fallback sorting used within a tier to ensure stable results when primary metrics are equal. |

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| **BC-BR-1** | **Exclusion: Empty Classifications** | Classifications with `lifetimeVolume = 0` AND `masterCaskCount = 0` MUST be excluded. Ensures only classifications with trade history or active inventory are surfaced. |
| **BC-BR-2** | **Tiered Ranking Strategy** | Tier 1 (`lifetimeVolume > 0`) ranked by `lifetimeVolume DESC`. If Tier 1 does not cover all results, remaining slots filled from Tier 2 (`lifetimeVolume = 0`), ranked by `estMarketValue DESC` (**NULL LAST**). All qualifying classifications are returned (no hard cap). |
| **BC-BR-3** | **Deterministic Tie-break** | Within a tier, ties resolved by `masterCaskCount DESC` then `classification ASC` (alphabetical slug). Must not rely on database default ordering. |
| **BC-BR-4** | **Unit-Weighted Median** | Median price calculations are unit-weighted: if a transaction covers 2 units at £10,000 each, both values enter the median dataset. |
| **BC-BR-5** | **Historical Integrity** | All-time metrics (`lifetimeVolume`, `medianPrice`) include ALL completed transactions, including those from inactive master casks (**SCR-002**). |
| **BC-BR-6** | **Trending Metric Formula** | A classification is marked `isTrending = true` if its `volumeLast30D > 0` AND `volumeDelta30D ≥ TRENDING_THRESHOLD` (recommended threshold: **+20%** or configurable). If `volumePrev30D = 0` but `volumeLast30D > 0`, the classification is also considered Trending (new activity from zero base). Requires **SCR-001**. |

---

## II. Functional Requirements

### 1. Browse by Category List

> Returns all qualified classifications on the platform, ranked by the **Tiered Ranking Strategy** (**BC-BR-2**). Tier 1 (traded) is prioritized by lifetime volume; Tier 2 (cold-start) serves as fill-in ranked by Estimated Market Value (**NULL LAST**). All ties are resolved via **Deterministic Tie-break** (**BC-BR-3**). Each classification entry includes median pricing, 30-day delta signals, and a **Trending** flag computed from volume momentum (**BC-BR-6**).
>
> This requires a **new dedicated endpoint**. The existing `GET /api/cask-metadata/classifications` returns classification labels but does not include market metrics.

#### 1.1. Query Parameters

None. The endpoint returns all qualifying classifications without pagination or limiting, as the total number of classifications on the platform is naturally small (expected < 20).

#### 1.2. Data Description

The API should return the following payload for each classification:

| No. | Field | Data Type | Description |
|-----|-------|-----------|-------------|
| 1 | `classification` | string | Classification slug (e.g., `single_malt`). Aggregation key and navigation filter key. |
| 2 | `categoryLabel` | string | Short display label shown on the card (e.g., `"Single Malt"`). Sourced from `MasterCask.categoryLabel`. All master casks in a classification share the same value. |
| 3 | `classificationImageUrl` | string \| null | URL to a representative image for this classification. Sourced from the `ClassificationMetadata` admin-managed entity. `null` if not configured. See [classification-metadata-admin.md](./classification-metadata-admin.md). |
| 4 | `masterCaskCount` | number | Count of Master Casks where `status = 'active'` belonging to this classification. Requires **SCR-002**. |
| 5 | `lifetimeVolume` | number | `SUM(cs.originalAmount)` for `completed` sessions. Default `0` if no transactions. |
| 6 | `medianPrice` | number \| null | `MEDIAN(tx.transactionPrice)` (unit-weighted per **BC-BR-4**). `null` if no completed transactions. |
| 7 | `estMarketValue` | number \| null | **Tier 2 only.** `SUM((referencePriceMin + referencePriceMax) / 2)` across all active variant casks. `null` if no active variants. |
| 8 | `estMedianPrice` | number \| null | **Tier 2 fallback.** `MEDIAN((referencePriceMin + referencePriceMax) / 2)` across all active variant casks. `null` if no active variants. FE displays this when `medianPrice` is `null`. |
| 9 | `volumeDelta30D` | number \| null | `(volumeLast30D - volumePrev30D) / volumePrev30D × 100`. Requires **SCR-001**. `null` if `volumePrev30D = 0` (unless handled by Trending logic). |
| 10 | `medianPriceDelta30D` | number \| null | `(medianLast30D - medianPrev30D) / medianPrev30D × 100`. Requires **SCR-001**. `null` if `medianPrev30D = 0` or `null`, or `medianLast30D` is `null`. |
| 11 | `isTrending` | boolean | `true` if category meets the Trending threshold per **BC-BR-6**. `false` otherwise. |
| 12 | `lastRefreshedAt` | datetime | Timestamp of when this classification's stats were last recomputed. Allows FE to display data freshness if needed. |

> [!NOTE]
> **Fields #9 (`volumeDelta30D`) is included in the payload even though FE currently only displays `medianPriceDelta30D` and `isTrending`.** Volume delta is included for future use and for the Trending metric computation. FE may choose to ignore `volumeDelta30D` in the initial UI iteration.

#### 1.3. Formula Reference & Examples

> [!IMPORTANT]
> **Note on Median calculation:** Median is computed at the **unit level** — if a transaction covers 2 units at £10,000 each, both £10,000 values are included in the median dataset. This prevents bulk-buy transactions from being under-represented versus single-unit trades.

> **Example of Unit-Weighted Median:**
> - **Tx 1:** 5 units @ £100
> - **Tx 2:** 1 unit @ £500
> - **Dataset for Median:** `[100, 100, 100, 100, 100, 500]`
> - **Result:** **£100** (Correct unit-level) vs £300 (Incorrect transaction-level)

##### 1.3.1. Metric Implementation & Calculation Table

| Metric | Exposed? | Calculation Formula | Dependencies |
|:---|:---:|:---|:---|
| **`lifetimeVolume`** | **Yes** (#5) | `SUM(cs.originalAmount)` | Filter: `status = 'completed'`. |
| **`medianPrice`** | **Yes** (#6) | `MEDIAN(tx.transactionPrice)` | Unit-weighted (**BC-BR-4**). |
| **`estMarketValue`** | **Yes** (#7) | `SUM((refMin + refMax) / 2)` | Tier 2 fallback. Active variants only (**SCR-002**). |
| **`estMedianPrice`** | **Yes** (#8) | `MEDIAN((refMin + refMax) / 2)` | Tier 2 fallback. Active variants only (**SCR-002**). |
| `volumeLast30D` | No (internal) | `SUM(cs.originalAmount)` | Window: `cs.completedAt` in last 30 days (**SCR-001**). |
| `volumePrev30D` | No (internal) | `SUM(cs.originalAmount)` | Window: `cs.completedAt` in prior 30-day window (**SCR-001**). |
| **`volumeDelta30D`** | **Yes** (#9) | `(vLast - vPrev) / vPrev × 100` | Requires **SCR-001**. `null` if `vPrev = 0`. |
| `medianLast30D` | No (internal) | `MEDIAN(tx.transactionPrice)` | Window: `cs.completedAt` in last 30 days (**SCR-001**). Unit-weighted. |
| `medianPrev30D` | No (internal) | `MEDIAN(tx.transactionPrice)` | Window: `cs.completedAt` in prior 30-day window (**SCR-001**). Unit-weighted. |
| **`medianPriceDelta30D`** | **Yes** (#10) | `(mLast - mPrev) / mPrev × 100` | Requires **SCR-001**. |
| **`isTrending`** | **Yes** (#11) | See **BC-BR-6** | `volumeLast30D > 0` AND (`volumeDelta30D ≥ threshold` OR `volumePrev30D = 0`). |
| **`lastRefreshedAt`** | **Yes** (#12) | `NOW()` at job runtime | Aggregation job timestamp. |

##### 1.3.2. Trending Metric — Detailed Formula

**BC-BR-6 — Trending Computation:**

```
isTrending = true  IF:
  (volumeLast30D > 0 AND volumePrev30D = 0)          -- new activity from zero
  OR
  (volumeDelta30D >= TRENDING_THRESHOLD)              -- sustained growth above threshold
  
WHERE:
  TRENDING_THRESHOLD = +20 (i.e., +20% change)       -- recommended default, configurable
  volumeDelta30D     = (volumeLast30D - volumePrev30D) / volumePrev30D × 100

isTrending = false  IF:
  volumeLast30D = 0     -- no activity in last 30D
  OR  volumeDelta30D < TRENDING_THRESHOLD
```

> [!NOTE]
> The `TRENDING_THRESHOLD` should be a configurable backend constant (not hardcoded), so it can be tuned without a code change. Recommended starting value: `+20%`.

> [!NOTE]
> **Decision — Volume-only Trending:** `isTrending` is intentionally based on **volume momentum only** (`volumeDelta30D`), not price momentum (`medianPriceDelta30D`). Rationale: "Trending" communicates *activity interest* (more people are trading this classification), which volume directly measures. Price movement is already surfaced separately via the `medianPriceDelta30D` delta badge on the card. Mixing price into the Trending signal would conflate two distinct market signals — a classification could appear "Trending" purely because prices dropped, driving opportunistic buyers, which is misleading for the intended UX meaning of "Trending".

> [!NOTE]
> **Active Variant Definition:** A variant is active if its parent `MasterCask.status = 'active'` AND its own `caskStatus = 'active'`. (**SCR-002**)

---

## III. Use Cases

### 1. Fetch All Qualifying Classifications

| Item | Description |
|------|-------------|
| **Objective** | Return all qualifying classifications with market signals, ranked by the **Tiered Ranking Strategy** (**BC-BR-2**) and **Deterministic Tie-break** (**BC-BR-3**) |
| **Actor** | System |
| **Trigger** | FE requests Browse by Category data (e.g., on Home page load) |
| **Pre-condition(s)** | Classifications exist in the system with at least one active master cask or historical transactions |
| **Post-condition(s)** | Returns all qualifying classifications (excluding empty ones per **BC-BR-1**) with computed stats |

**Activity Flow & Business Rules:**

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Load all classifications | — | Fetch full classification list from the source of truth (`cask-metadata/classifications`). |
| 2 | Join with stats | BC-BR-4, BC-BR-5 | For each classification, aggregate `lifetimeVolume`, `medianPrice` (unit-weighted), `masterCaskCount`, `estMarketValue`, `estMedianPrice` from the pre-computed `ClassificationVolumeStats` table. |
| 3 | Compute 30D windows | SCR-001 | Compute `volumeLast30D`, `volumePrev30D`, `medianLast30D`, `medianPrev30D` using `cs.completedAt` date windows. |
| 4 | Compute deltas | — | Derive `volumeDelta30D` and `medianPriceDelta30D` from 30D windows. Set `null` where denominator is 0. |
| 5 | Compute Trending flag | BC-BR-6 | Derive `isTrending` per the Trending formula. |
| 6 | Exclude empty classifications | BC-BR-1 | Remove any classification where `lifetimeVolume = 0` AND `masterCaskCount = 0`. |
| 7 | Apply Tiered Ranking | BC-BR-2 | Sort: Tier 1 (`lifetimeVolume > 0`) by `lifetimeVolume DESC`, then Tier 2 by `estMarketValue DESC` (**NULL LAST**). |
| 8 | Apply Tie-break | BC-BR-3 | Within each tier, break ties by `masterCaskCount DESC`, then `classification ASC`. |
| 9 | Return response | — | Serialize and return the payload per Section 1.2. |

**Happy Path:**

| # | Scenario | Expected Result | Notes |
|---|----------|----------------|-------|
| 1 | Platform has multiple classifications with `lifetimeVolume > 0` | Return all Tier 1 classifications sorted by `lifetimeVolume DESC`, followed by Tier 2 fill-ins | Fields resolved per Section 1.2 definitions |
| 2 | Platform has mix of traded and non-traded classifications | Return Tier 1 first, then Tier 2 by `estMarketValue DESC` (**NULL LAST**) | |

**Negative Path:**

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | No qualifying classifications exist | Return empty array `[]` | 200 OK |
| 2 | Server error | Return error | 500 Internal Server Error |
| 3 | Unauthenticated request | Reject request | 401 Unauthorized |

**Edge Cases:**

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|-|
| 1 | All classifications have `lifetimeVolume = 0` | Return all non-empty classifications sorted by Tier 2 ranking (`estMarketValue DESC`, NULL LAST), then tie-break | FE renders cards with `null` medianPrice (shows `estMedianPrice` fallback or "—") |
| 2 | A classification has `lifetimeVolume = 0` AND `masterCaskCount = 0` | Exclude per **BC-BR-1** | — |
| 3 | Two classifications have equal `lifetimeVolume` AND equal `masterCaskCount` | Apply final tie-break: `classification ASC`. Must not rely on database default. | — |
| 4 | `volumePrev30D = 0` but `volumeLast30D > 0` | `volumeDelta30D = null` (division by zero); `isTrending = true` (new activity from zero base per **BC-BR-6**) | — |
| 5 | `medianPrev30D = null` or `0` | `medianPriceDelta30D = null` | FE hides delta badge |
| 6 | `referencePriceMax = 0` for a variant (data quality issue) | Exclude that variant from `estMarketValue` / `estMedianPrice` calculations (treat as invalid reference data) | Log warning for data team |

---

## IV. Technical Dependencies

### 1. Schema Change Request: SCR-001

The calculation of 30-day delta metrics (Fields #9–#10) and the **Trending metric** (**BC-BR-6**) depend on the implementation of **SCR-001**.

| Item | Detail |
|------|--------|
| **Scope** | Add `completedAt` timestamp to the `CheckoutSession` entity. |
| **Status** | 🟡 Pending (Requires Backend implementation). |
| **Fallback Behavior** | Until SCR-001 is delivered, use `updatedAt` as a temporary proxy for `completedAt` (filtered for sessions where `status = 'completed'`). Note: This may cause minor inaccuracies if a completed session is updated after completion. |
| **Reference** | See [schema-change-requests.md](../data-models/schema-change-requests.md) for full technical spec. |

### 2. Schema Change Request: SCR-002

The **`masterCaskCount`** calculation and active variant filtering depend on **SCR-002**.

| Item | Detail |
|------|--------|
| **Scope** | Add `status` (`active` / `inactive`) to the **Master Cask** entity. |
| **Status** | 🟢 Approved (System enhancement). |
| **Logic** | **Master Status overrides Variant Status.** If a Master Cask is `inactive`, all its child variant casks must be excluded from active counts (`masterCaskCount`) and estimation metrics (`est*`). Historical metrics continue to include all past completed transactions. |
| **Fallback** | Until SCR-002 is delivered, use derived logic: "Active if ≥1 child has `caskStatus = 'active'`". |

---

## V. Data Model

### Entity: Classification — existing (reference)

Managed via `GET /api/cask-metadata/classifications`. Returns a hierarchically structured list of category classifications (2-level hierarchy: classification and styles).

- `classification`: Level 1 slug (e.g., `single_malt`)
- `classificationLabel`: Level 1 display name (e.g., `Single Malt`)
- `imageUrl`: S3-hosted category cover image URL
- `activeCount`: Level 1 count of active master casks matching query criteria
- `styles`: Array of Level 2 styles under the classification (each has `value` and `label`, e.g., `single_malt_unpeated`, `Unpeated`)

### Entity: ClassificationMetadata — new (admin-managed)

A new admin-managed entity to store per-classification display configuration (image, label overrides, visibility). Full spec in [classification-metadata-admin.md](./classification-metadata-admin.md).

- `classification`: string (slug — primary key, FK to classification enum)
- `classificationImageUrl`: string | null — S3-hosted image URL for this classification card
- `displayName`: string | null — optional admin override for `categoryLabel` display (falls back to `categoryLabel` from master cask data if null)
- `isVisible`: boolean — if `false`, classification is suppressed from the Browse by Category section even if it has data
- `createdAt` / `updatedAt`: datetime

### Entity: Master Cask — existing (reference)

- `id`: string (UUID)
- `name`: string
- `classification`: string (FK reference to classification slug)
- `classificationLabel`: string
- `status`: string (`active` / `inactive`) — **New requested field (SCR-002)**.

### Entity: CheckoutSession — existing (reference)

- `id`: string (UUID)
- `originalAmount`: number — total agreed trade value for the session.
- `status`: string (`completed`, `pending`, etc.)
- `completedAt`: datetime | null — **New requested field (SCR-001)**. Primary boundary for 30D window calculations.
- `updatedAt`: datetime — **fallback proxy** until SCR-001 is delivered.

### Entity: Transaction — existing (reference)

- `transactionPrice`: number — agreed per-unit price.
- `quantity`: number — units in the transaction. Used for unit-weighted median (**BC-BR-4**).
- `status`: string (`completed`, `refunded`, etc.)

### Derived / Pre-computed: ClassificationVolumeStats — new (recommended)

| Field | Type | Description |
|-------|------|-------------|
| `classification` | string | Classification slug — primary key |
| `categoryLabel` | string | Short display label for the card (e.g., "Single Malt"). Sourced from `MasterCask.categoryLabel`. |
| `classificationImageUrl` | string \| null | Sourced from `ClassificationMetadata.classificationImageUrl`. `null` if admin has not configured an image. |
| `lifetimeVolume` | number | SUM of all completed `cs.originalAmount` |
| `masterCaskCount` | number | Count of active Master Casks in this classification |
| `medianPrice` | number \| null | Unit-weighted median transaction price (all time) |
| `estMarketValue` | number \| null | SUM of mid-points of active variant reference prices |
| `estMedianPrice` | number \| null | Median of mid-points of active variant reference prices |
| `volumeDelta30D` | number \| null | 30-day volume percentage change |
| `medianPriceDelta30D` | number \| null | 30-day median price percentage change |
| `isTrending` | boolean | Whether classification meets the Trending threshold (**volume-only** per **BC-BR-6**) |
| `lastRefreshedAt` | datetime (ISO) | Timestamp of last stats refresh |

---

## VI. Permissions & Access Control

| Use Case / Action | Anonymous | Buyer | Seller | Admin |
|---|---|---|---|---|
| Fetch Browse by Category list | ❌ | ✅ | ✅ | ✅ |

> **Rationale for no anonymous access:** Classification market data (median prices, volumes, deltas) is proprietary market intelligence. Access is restricted to authenticated users only, consistent with the platform's authentication-first policy.

---

## VII. Performance Considerations

1. **Pre-computation:** Data is served from a `ClassificationVolumeStats` table, refreshed by a scheduled aggregation job every **15–30 minutes**.
2. **Event-Driven Refresh:** An asynchronous refresh of a specific classification's stats MUST be triggered upon:
   - Any checkout session transitioning to `completed` (setting or updating `completedAt` per **SCR-001**).
   - Any admin update to `masterCask.status` (per **SCR-002**).
   - Any admin update to `ClassificationMetadata` (image, visibility, display name) — see [classification-metadata-admin.md](./classification-metadata-admin.md).
3. **Median Optimization:** Use unit-weighted pre-calculation (**BC-BR-4**) during the stats aggregation job, not at query time.
4. **No Pagination Needed:** Since the number of classifications is small (expected < 20 on the platform), no pagination is required. All classifications are returned in a single response.

---

_End of Document 2_
