# BACKEND LOGIC & FLOW SPECS

Explore Casks — Home Page Curated List (Product Tabs)

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | Alice | 05/05/2026 |
| 0.2 | Resolved open questions: `publishedAt` added to MasterCask; Floor Price label switching logic clarified | Alice | 05/05/2026 |
| 1.0 | Consolidated production API alignment: aligned old documentation with current production API. | Alice | 19/05/2026 |
| 1.1 | Updated Strategic Buy sorting definition to match Tiered Buy Opportunity; added lowestAskPrev30D formula and snapshot mechanism. | Alice | 26/05/2026 |

---

## I. Introduction

### 1. Purpose

> Provide sorted master cask recommendation data for the **Explore Casks** section on the Home page. The section displays 4 product tabs, each using a distinct sorting strategy to surface relevant master casks for buyers:
>
> - **Top Traded**: Sorted by `lifetimeVolume DESC` — casks with the highest all-time transacted value
> - **Most Watched**: Sorted by `viewCount DESC` — casks with the most page views
> - **Strategic Buy**: Sorted by Tiered Buy Opportunity (`lowestAsk ASC`, then fallback to `minReferencePrice ASC`) — lowest-priced active market entry points
> - **New Listing**: Sorted by `latestListingDate DESC` — most recently listed casks. `latestListingDate` is derived from `MAX(variant.listingDate)` across all active variants.
> Each tab returns the requested number of master casks (based on the `size` query parameter) with a standardized card response structure. The View All CTA links to `/marketplace` with the matching sort.
> **Note on Data Model Naming Conventions:** Field references (e.g., `checkoutSession.originalAmount`, `variant.lowestAsk`) are written in conceptual, singular form. Developers should map these to actual table/field names accordingly.

---

### 2. Definitions

| Term | Definition |
|------|------------|
| Master Cask (Parent) | A parent entity grouping multiple variant casks. Belongs to exactly one Distillery. Holds shared attributes: name, distillery, caskType, classification, region. |
| Variant Cask (Child) | A specific vintage of a Master Cask. Transactions occur at the variant level. Holds vintage-specific data: vintageYear, referencePriceMin, lowestAsk, etc. |
| Successful Transaction | A completed trade where `checkoutSession.status = 'completed'`. |
| Lifetime Volume | `SUM(cs.originalAmount)` for `completed` sessions across all variants of that master cask. Default `0` if no transactions. Mirrors Top Distilleries logic. |
| lowestAsk | The `masterCask.lowestAsk` field. `null` if no active asks exist on any variant. Represents the real market minimum purchase price. |
| minReferencePrice | The `masterCask.minReferencePrice` field. Admin-set reference minimum — always present. Used as fallback when `lowestAsk` is null. |
| `askCount` | The count of distinct active ask orders across all variants of a master cask. Sourced from the production API response field `askCount`. |
| `purchaseAvailability` | The SUM of all active ask quantities (available casks) across all variants of a master cask. Displayed as "Listed" count on the frontend card. **New requested field**. |
| `lowestAskDelta30D` | Percentage change in the `lowestAsk` over the last 30 days. `null` if no snapshot exists. |
| viewCount | The master cask's own page view counter — tracked at master cask level. |
| minVintageYear | `MIN(variant.vintageYear)` across all **active** variants of the master cask. |
| maxVintageYear | `MAX(variant.vintageYear)` across all **active** variants of the master cask. |
| Active Ask | An ask where `ask.status = 'active'` AND `ask.expirationDate > NOW()`. Shared definition across all features. |
| latestListingDate | Derived from `MAX(variant.listingDate)` across all active variants of the master cask. Used for New Listing sort. |
| Tiered Ranking Strategy | The primary sorting mechanism that prioritizes active markets (Tier 1) over estimated markets (Tier 2). Used in Strategic Buy tab. |
| Deterministic Tie-break | The secondary/tertiary fallback sorting used within a tier to ensure stable results when primary metrics are equal. |

---

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| **EC-BR-1** | **Eligibility: Active Master Cask and Variant Required** | Only master casks where `masterCask.status = 'active'` (**SCR-002**) AND with ≥ 1 variant where `caskStatus = 'active'` AND `isListed = true` are included across all 4 tabs. |
| **EC-BR-2** | **Result Cap** | Each tab returns the number of casks specified by the `size` request parameter, sent by the FE based on the current viewport layout. |
| **EC-BR-3** | **Top Traded Tab — Ranking** | Eligible casks are ranked by `lifetimeVolume DESC`. Casks with zero lifetime volume remain eligible and fall naturally after all traded casks. |
| **EC-BR-4** | **Most Watched Tab — Ranking** | Eligible casks are ranked by `viewCount DESC`. |
| **EC-BR-5** | **Strategic Buy Tab — Tiered Ranking** | Casks are divided into two tiers ranked separately. Tier 1 (`lowestAsk IS NOT NULL`) is ranked by `lowestAsk ASC`. Tier 2 (`lowestAsk IS NULL`) is ranked by `minReferencePrice ASC`. Tier 1 always precedes Tier 2. |
| **EC-BR-6** | **New Listing Tab — Ranking** | Eligible casks are ranked by `latestListingDate DESC`. `latestListingDate` is defined in Section I.2. |
| **EC-BR-7** | **Deterministic Tie-break** | When primary sort values are equal within a sort group or tier, apply a secondary sort. For **Top Traded**: secondary sort is `minReferencePrice ASC` (keeps the tab distinct from Most Watched in cold-start), final tie-break is `id ASC`. For **Strategic Buy** (within each tier), **Most Watched**, and **New Listing**: tie-break is `id ASC` only. The final tie-break **MUST be deterministic** and must not rely on database default ordering. |

---

## II. Functional Requirements

### 0. Endpoint Strategy

> The Explore Casks section uses the **`/api/cask-masters` endpoint** — one call per tab, with sort and pagination params supplied at call time.
>
> The table below describes the **sorting intent and deterministic tie-breakers** for each tab. Subelement parameters (e.g., `sortBy` and `sortOrder`) are to be aligned with the backend's production API:
>
> | # | Tab | Primary Sorting Intent (`sortBy` field) | Sort Direction | Fallback Tie-breakers (Deterministic) |
> |---|---|---|---|---|
> | 1 | Top Traded | `lifetimeVolume` | DESC | `minReferencePrice ASC`, then `id ASC` |
> | 2 | Most Watched | `viewCount` (or `popularity`) | DESC | `id ASC` |
> | 3 | Strategic Buy | Tiered Buy Opportunity (`lowestAsk`, then `minReferencePrice`) | ASC | `id ASC` (applied within each tier) |
> | 4 | New Listing | `latestListingDate` (or `createdAt`) | DESC | `id ASC` |
>
> The number of results per call is viewport-driven and supplied by the FE via the `size` query parameter (e.g., 6 on mobile, 8 on tablet, 9 on desktop), with `page=1`.
>
> **Dependency Note:** The "View All" CTA navigates to the marketplace. The FE must map each tab's sorting intent to the correct marketplace URL sort params. Exact values to be agreed between BE and FE when implementing the Marketplace sort feature.

---

### 1. Common Master Cask Card Response Structure

#### 1.1. Overview

> Defines the standard data structure returned by the unified `/api/cask-masters` endpoint. The endpoint returns the same base card response structure for all 4 Explore Casks tabs.

#### 1.2. Data Description

The endpoint returns **all fields currently provided by `/api/cask-masters`** (see the production API schema for the full field list). No existing fields are removed.

In addition, the following fields are **required by this feature** and must be included in the response. BE to confirm which are already available and which need to be added:

| No. | Field | Data Type | Description |
|-----|-------|-----------|-------------|
| 1 | `lifetimeVolume` | number | **All tabs.** `SUM(cs.originalAmount)` for `completed` sessions across all variants of that master cask. Default `0` if no transactions. Primary sort key for Top Traded tab. |
| 2 | `lowestAskDelta30D` | number \| null | **All tabs.** 30D lowest ask delta in percent: `((lowestAsk - lowestAskPrev30D) / lowestAskPrev30D) * 100`. `null` if `lowestAskPrev30D` is null or if `lowestAsk` is null. |
| 3 | `purchaseAvailability` | number | **All tabs.** SUM of active ask quantities (available casks) across all variants. **New requested field** (e.g., if Variant A has 2 active asks for 5 casks each, and Variant B has 1 active ask for 3 casks, `purchaseAvailability = 13` total casks available, whereas `askCount = 3` distinct ask orders). |
| 4 | `latestListingDate` | datetime \| null | **New Listing only.** `MAX(variant.listingDate)` across active variants. Primary sort key for New Listing tab. |
| 5 | `distilleryName` | string | **All tabs.** Distillery display name (e.g., "Aberlour"). Sourced from `masterCask.distillery.name`. |
| 6 | `minVintageYear` | number \| null | **All tabs.** `MIN(variant.vintageYear)` across active variants. |
| 7 | `maxVintageYear` | number \| null | **All tabs.** `MAX(variant.vintageYear)` across active variants. |
| 8 | `lowestAskPrev30D` | number \| null | **All tabs.** The lowest active ask price recorded 30 days ago (historical benchmark). `null` if no snapshot exists. **New requested field**. |

#### 1.3. Formula Reference & Examples

##### 1.3.1. Metric Implementation & Calculation Table

The following table defines the logic for all metrics returned by the API or used for internal ranking/delta computations.

| Metric | Exposed? | Calculation Formula | Dependencies / Notes |
|:---|:---:|:---|:---|
| **`lowestAsk`** | **Yes** (Master field) | `MIN(variant.lowestAsk)` | Current lowest active ask across all active variants. |
| **`lowestAskPrev30D`** | **Yes** (#8) | `MIN(ask.askPrice)` on date `NOW() - 30 days` | Requires daily snapshot table (`master_cask_lowest_ask_snapshots`) recording `lowestAsk` of active asks at the end of each day. `null` if no snapshot exists. |
| **`lowestAskDelta30D`** | **Yes** (#2) | `((lowestAsk - lowestAskPrev30D) / lowestAskPrev30D) * 100` | Percent change in lowest ask over the last 30 days. `null` if `lowestAskPrev30D` is null or if `lowestAsk` is null. |
| **`purchaseAvailability`**| **Yes** (#3) | `SUM(ask.quantity)` | Sum of quantities of all active asks across all active variants. Active ask filter: `ask.status = 'active' AND ask.expirationDate > NOW()`. |
| **`latestListingDate`** | **Yes** (#4) | `MAX(variant.listingDate)` | Max listing date of active variants. |

---

### 2. Top Traded Tab

#### 2.1. Overview

> Returns master casks with the highest all-time transacted volume (GBP), sorted DESC.

#### 2.2. Use Cases

##### UC-1. Fetch Top Traded Master Casks

| Item | Description |
|------|-------------|
| **Objective** | Return the requested number of master casks ranked by all-time completed transaction volume (GBP) |
| **Actor** | System (triggered by FE) |
| **Trigger** | FE requests Explore Casks data (default tab on Home page load) |
| **Pre-condition(s)** | Master casks exist with ≥ 1 active variant |
| **Post-condition(s)** | Returns ranked list matching the requested size, sorted by `lifetimeVolume DESC` (**EC-BR-3**) |

**Happy Path**:

| # | Scenario | Expected Result | Notes |
|---|----------|----------------|-------|
| 1 | Platform has ≥ requested size eligible master casks with lifetimeVolume > 0 | Return results sorted by `lifetimeVolume DESC` (**EC-BR-3**) | Standard card fields resolved per Section I.2 definitions |
| 2 | Platform has mix of master casks with lifetimeVolume > 0 and lifetimeVolume = 0 | Return all eligible master casks sorted by `lifetimeVolume DESC`. Any casks with equal volume (including zero-volume casks) are tie-broken by `minReferencePrice ASC`, then `id ASC` (**EC-BR-7**). | |

**Negative Path**:

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | No eligible master casks exist | Return empty array `[]` | 200 OK |
| 2 | Server error | Return error | 500 Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|-|
| 1 | All eligible master casks have `lifetimeVolume = 0` (platform cold start) | Return up to the requested size, sorted by `minReferencePrice ASC`, then `id ASC`. **Note:** This intentionally differs from Most Watched (which sorts by `viewCount DESC`) to keep the two tabs distinct during cold start. | FE renders cards with `—` in volume field |
| 2 | Fewer than requested size eligible master casks exist | Return all eligible master casks | FE renders available cards |

---

### 3. Most Watched Tab

#### 3.1. Overview

> Returns the most viewed master casks on the platform, sorted by viewCount DESC.

#### 3.2. Use Cases

##### UC-2. Fetch Most Watched Master Casks

| Item | Description |
|------|-------------|
| **Objective** | Return the requested number of master casks with the highest all-time page view count |
| **Actor** | System (triggered by FE) |
| **Trigger** | FE clicks "Most Watched" tab |
| **Pre-condition(s)** | Master casks exist with ≥ 1 active variant |
| **Post-condition(s)** | Returns ranked list matching the requested size, sorted by `viewCount DESC` (**EC-BR-4**) |

**Happy Path**:

| # | Scenario | Expected Result | Notes |
|---|----------|----------------|-------|
| 1 | Platform has ≥ requested size eligible master casks with viewCount > 0 | Return results sorted by `viewCount DESC` | Standard card fields resolved per Section I.2 definitions |

**Negative Path**:

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | No eligible master casks exist | Return empty array `[]` | 200 OK |
| 2 | Server error | Return error | 500 Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|-|
| 1 | All eligible master casks have viewCount = 0 | Return up to the requested size, sorted by `id ASC` (deterministic fallback per **EC-BR-7**). **Must not rely on database default.** | - |
| 2 | Fewer than requested size eligible master casks exist | Return all eligible master casks | FE renders available cards |

---

### 4. Strategic Buy Tab

#### 4.1. Overview

> Returns master casks sorted to highlight the best entry-point buying opportunities using a two-tier sort. This requirement formalizes the "Strategic Buys" logic into a new dedicated endpoint.

#### 4.2. Use Cases

##### UC-3. Fetch Strategic Buy Master Casks

| Item | Description |
|------|-------------|
| **Objective** | Return master casks sorted by best buying opportunity |
| **Actor** | System (triggered by FE) |
| **Trigger** | FE clicks "Strategic Buy" tab |
| **Pre-condition(s)** | Master casks exist with ≥ 1 active variant. |
| **Post-condition(s)** | Returns ranked list matching the requested size, with two-tier sort applied (**EC-BR-5**) |

**Happy Path**:

| # | Scenario | Expected Result | Notes |
|---|----------|----------------|-------|
| 1 | Platform has ≥ requested size eligible master casks with lowestAsk IS NOT NULL | Return results sorted by `lowestAsk ASC` (Tier 1). Tie-break: `id ASC`. (**EC-BR-5**, **EC-BR-7**) | Standard card fields resolved per Section I.2 definitions |
| 2 | Platform has mix of master casks with lowestAsk IS NOT NULL and lowestAsk IS NULL | Return all eligible master casks: Tier 1 (`lowestAsk IS NOT NULL`) sorted by `lowestAsk ASC`, followed by Tier 2 (`lowestAsk IS NULL`) sorted by `minReferencePrice ASC`. Within each tier — Tie-break: `id ASC`. (**EC-BR-5**, **EC-BR-7**) | |

**Negative Path**:

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | No eligible master casks exist | Return empty array `[]` | 200 OK |
| 2 | Server error | Return error | 500 Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|-|
| 1 | All eligible master casks lack active asks | Return up to the requested size, sorted by `minReferencePrice ASC` (Tier 2 only). Tie-break: `id ASC`. (**EC-BR-5**, **EC-BR-7**) | FE renders cards with "Est. Floor Price" reference price label |
| 2 | Fewer than requested size eligible master casks exist | Return all eligible master casks | FE renders available cards |
| 3 | An ask expires between list render and user click | FE navigates to detail page with live data; no list-level handling needed | - |

---

### 5. New Listing Tab

#### 5.1. Overview

> Returns the most recently listed master casks, sorted by `latestListingDate DESC`. Allows buyers to discover fresh inventory. `latestListingDate` is defined in Section I.2.

#### 5.2. Use Cases

##### UC-4. Fetch New Listing Master Casks

| Item | Description |
|------|-------------|
| **Objective** | Return the requested number of master casks sorted by most recently listed variant |
| **Actor** | System (triggered by FE) |
| **Trigger** | FE clicks "New Listing" tab |
| **Pre-condition(s)** | Master casks exist with ≥ 1 active variant |
| **Post-condition(s)** | Returns ranked list matching the requested size, sorted by `latestListingDate DESC` |

**Happy Path**:

| # | Scenario | Expected Result | Notes |
|---|----------|----------------|-------|
| 1 | Platform has ≥ requested size eligible master casks | Return results sorted by `latestListingDate DESC` (**EC-BR-6**) | Standard card fields resolved per Section I.2 definitions |

**Negative Path**:

| # | Scenario | Expected Result | Error Code |
|---|----------|----------------|------------|
| 1 | No eligible master casks exist | Return empty array `[]` | 200 OK |
| 2 | Server error | Return error | 500 Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|-|
| 1 | Multiple master casks have same latestListingDate | Tie-break: `id ASC` (deterministic fallback per **EC-BR-7**). **Must not rely on database default.** | - |
| 2 | Fewer than requested size eligible master casks exist | Return all eligible master casks | FE renders available cards |

---

## III. Data Model

### Entity: MasterCask (Parent) — existing, updated

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | ✅ | auto-generated | Primary key |
| status | string | ✅ | active | Lifecycle status: `'active'` or `'inactive'`. **New requested field (SCR-002)**. If `'inactive'`, the master cask is excluded from all tab results, regardless of variant status. |
| name | string | ✅ | — | Cask display name |
| distilleryId | UUID (FK) | ✅ | — | FK to Distillery |
| caskTypeId | UUID (FK) | ✅ | — | FK to CaskType |
| classification | string | ✅ | — | e.g., `single_malt` |
| regionId | UUID (FK) | ✅ | — | FK to Region |
| imageUrl | string \| null | ❌ | null | Master cask primary image |
| viewCount | number | ✅ | 0 | All-time page view counter. Used for Most Watched sort |
| lowestAsk | number \| null | ❌ | null | Pre-computed minimum active ask price across variants |
| minReferencePrice | number | ✅ | — | Pre-computed minimum reference price across variants |
| maxReferencePrice | number | ✅ | — | Pre-computed maximum reference price across variants |
| price `[INTERNAL]` | number | ✅ | — | Effective floor price (`lowestAsk` fallback to `minReferencePrice`). **Internal use only — not exposed in API response** (removed in v0.7). FE computes this natively from `lowestAsk` and `minReferencePrice`. |
| createdAt | datetime | ✅ | NOW() | Record creation timestamp |
| updatedAt | datetime | ✅ | NOW() | Last update timestamp |

### Entity: CaskVariant (Child) — existing, reference

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | ✅ | auto-generated | Primary key |
| masterCaskId | UUID (FK) | ✅ | — | FK to MasterCask |
| vintageYear | number | ✅ | — | Year of vintage |
| referencePriceMin | number | ✅ | — | Minimum reference price |
| referencePriceMax | number | ✅ | — | Maximum reference price |
| lowestAsk | number \| null | ❌ | null | Current minimum active ask price |
| highestBid | number \| null | ❌ | null | Current maximum active bid price |
| caskStatus | enum | ✅ | — | `active`, `sold`, `inactive` |
| isListed | boolean | ✅ | false | Whether the variant is publicly listed. Used in conjunction with `caskStatus = 'active'` to determine variant eligibility. |
| listingDate | datetime | ✅ | — | Date this specific variant was listed on the marketplace |
| createdAt | datetime | ✅ | NOW() | Record creation timestamp |
| updatedAt | datetime | ✅ | NOW() | Last update timestamp |

### Derived / Pre-computed: MasterCaskMarketStats — new (recommended)

| Field | Type | Description |
|-------|------|-------------|
| masterCaskId | UUID | FK to MasterCask |
| lifetimeVolume | number | SUM of completed session amounts across all variants |
| lowestAskPrev30D | number \| null | The lowest ask 30 days ago (benchmark price) |
| lowestAskDelta30D | number \| null | Percent change in lowest ask over the last 30 days |
| `askCount` | number | Number of active ask orders across all variants |
| `purchaseAvailability` | number | SUM of active ask quantities (available casks) across all variants |
| minVintageYear | number \| null | MIN(variant.vintageYear) for active variants |
| maxVintageYear | number \| null | MAX(variant.vintageYear) for active variants |
| latestListingDate | datetime \| null | MAX(variant.listingDate) for active variants |
| lastRefreshedAt | datetime | Last time stats were recomputed |

---

## IV. Technical Dependencies

### 1. Schema Change Request: SCR-002

The **Master Cask eligibility filter** (**EC-BR-1**) depends on the implementation of **SCR-002**.

| Item | Detail |
|------|--------|
| **Scope** | Add `status` (`active` / `inactive`) to the **MasterCask** entity. |
| **Status** | 🟢 Approved (System enhancement). |
| **Impact on EC-BR-1** | The eligibility filter must check `masterCask.status = 'active'` in addition to `variant.caskStatus = 'active'`. A master cask flagged as `inactive` must be excluded from all 4 tab results, regardless of variant status. |
| **Reference** | See [schema-change-requests.md](../data-models/schema-change-requests.md) for full technical spec. |

### 2. SCR-001 — Not Applicable

**SCR-001** (adding `completedAt` to `CheckoutSession`) is **not required** for this feature:
- `lowestAskDelta30D` is computed from **lowest ask snapshots** (`MasterCaskMarketStats.lowestAskPrev30D`), not from a 30-day checkout session window.
- `lifetimeVolume` uses a simple `status = 'completed'` filter with no time-window dependency.

No fallback behavior is needed for this feature in relation to SCR-001.

---

## V. Notifications / Side Effects

| # | Trigger Event | Channel | Recipient | Description |
|---|--------------|---------|-----------|-------------|
| — | N/A | — | — | No notifications triggered by these read-only curated list APIs |

---

## VI. Permissions & Access Control

| Action | Anonymous | Buyer | Seller | Admin |
|--------|-----------|-------|--------|-------|
| Load Top Traded | ✅ | ✅ | ✅ | ✅ |
| Load Most Watched | ✅ | ✅ | ✅ | ✅ |
| Load Strategic Buy | ✅ | ✅ | ✅ | ✅ |
| Load New Listing | ✅ | ✅ | ✅ | ✅ |

> All 4 Explore Casks tabs are publicly accessible. No authentication required for read access.
>
> **Rate limiting**: Anonymous access is subject to the platform's standard API rate-limiting policy (implementation detail for BE). This spec does not define specific rate limits, but they should be applied to prevent abuse of unauthenticated endpoints.

---

## VII. Resolved Decisions

| # | Decision | Resolution |
|---|----------|------------|
| 1 | **Sorting field for New Listing tab** | New Listing tab sorts by the most recently listed/created master cask (DESC). The conceptual `latestListingDate` (derived from `MAX(variant.listingDate)`) is the preferred business logic; `masterCask.createdAt` is a candidate fallback. BE to confirm the exact field and sort param used in production. |
| 2 | **Floor Price label — universal or context-sensitive?** | **Two distinct labels** used across all tabs: **"Floor Price"** when `lowestAsk IS NOT NULL` (real market ask exists); **"Est. Floor Price"** when `lowestAsk IS NULL` (reference price only). This prevents buyer confusion between a tradeable price and an estimated reference. |
| 3 | **Unified endpoint vs 4 separate endpoints** | **Resolved: Use the unified `/api/cask-masters` endpoint** — one call per tab with sort and pagination params (see Section II.0). Maintaining separate home-page endpoints would duplicate sort logic and create a sync risk. Pagination size is viewport-driven (FE responsibility); exact param name to be confirmed with BE. |
| 4 | **View All CTA → Marketplace sort param mapping** | Each tab's "View All" CTA links to `/marketplace` with a sort param matching the tab's sorting intent (e.g., volume DESC, view count DESC, strategic buy order, newest first). Exact param names and values to be agreed between BE and FE when implementing the Marketplace sort feature. |
| 5 | **Exact sort param names not locked in this spec** | Sort param names and enum values (e.g., `sortBy`, `sort`, `order`) are intentionally left to BE to define based on the production API. This spec documents sorting **intent** only. FE must align with BE on final param names before implementation. |

---

## VIII. Performance Considerations

| # | Concern | Recommendation |
|---|---------|---------------|
| 1 | All 4 tabs involve aggregations across variants, asks, and checkout sessions | Pre-compute `MasterCaskMarketStats` table, refreshed every 5–15 minutes via background job |
| 2 | `lowestAskDelta30D` requires tracking historical lowest ask prices | Pre-compute in background job; store `lowestAskPrev30D` in stats table |
| 3 | `askCount` and `purchaseAvailability` change frequently as orders are placed/expired | Update denormalized fields reactively on ask create/cancel/expire events, OR refresh via cron |
| 4 | `lifetimeVolume` grows monotonically and changes only on completed transactions | Update on checkout session completion event |
| 5 | `latestListingDate`, `minVintageYear`, `maxVintageYear` change when a variant's `caskStatus` changes | Refresh `MasterCaskMarketStats` via cron, OR update reactively on variant `caskStatus` change event |

---

_End of Document 2_
