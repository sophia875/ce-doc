# 📄 Curated List for Home – Flow Specs

**Version:** 1.0
**Last Updated:** 2026-04-08
**Status:** Draft

---

## I. Introduction

### 1. Purpose

Provide curated cask recommendation data for the Home page. The system aggregates and sorts cask data using 4 distinct strategies:

- Popularity: Based on viewCount
- Pricing opportunity: Based on lowest ask
- Market demand: Based on bid + ask activity
- Personal view history

### 2. Definitions

| Term | Definition |
|------|------------|
| Master Cask (Parent) | A parent entity grouping multiple vintages. Holds shared attributes: name, distillery, caskType, classification, region, image. |
| Cask Variant (Child) | A specific vintage of a Master Cask. Holds vintage-specific attributes: vintageYear, distillationDate, age, referencePriceMin, etc. Linked to exactly one parent. |
| viewCount | Number of times the master cask detail page has been viewed. Tracked at the master cask level |
| Ask | A sell order placed by a Seller on the marketplace for a specific cask variant |
| Bid | A buy order placed by a Buyer on the marketplace for a specific cask variant |
| lowestAsk | For variant: the minimum active ask price / For master: MIN(lowestAsk) across all variants |
| referencePriceMin | For variant: the minimum reference price / For master: MIN(referencePriceMin) across all variants |
| referencePriceMax | For variant: the maximum reference price / For master: MAX(referencePriceMax) across all variants |
| Total Activity Score | Sum of all bid quantities + all ask quantities across all variants of a master cask. Measures market liquidity |
| Recently Viewed | Record of master cask detail pages visited by an authenticated user, in reverse chronological order |

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| CL-BR-1 | High Interest sort uses viewCount DESC | Top 10 master casks sorted by master cask's own viewCount |
| CL-BR-2 | Strategic Buys uses two-tier priority sort | Priority 1: MIN(lowestAsk) across all variants ASC; Priority 2: MIN(referencePriceMin) across all variants ASC |
| CL-BR-3 | In High Demand uses Total Activity Score DESC | Score = SUM(all bid quantities) + SUM(all ask quantities) across all variants per master cask |
| CL-BR-4 | Recently Viewed returns reverse chronological order | Most recently viewed master cask first |
| CL-BR-5 | All curated lists are capped at 10 items maximum | Default limit = 10. Configurable via query param |
| CL-BR-6 | Only master casks with active variants are included in curated lists | Master cask must have at least one variant where caskStatus = 'active' |
| CL-BR-7 | Only active asks/bids are counted for sorting | Expired, cancelled, or matched asks/bids are excluded from calculations |
| CL-BR-8 | All curated endpoints return master cask data | Cards always display parent-level fields; price fields are aggregated from child variants |

---

## II. Requirements

### 1. Master Cask Display (Common Payload)

#### 1.1. Overview

> Defines the common data structure returned for master casks across all curated list endpoints. All curated lists return this standard card format with aggregated parent-level details and price indicators. Endpoints may append specific fields used for sorting.

#### 1.2. Data Description

| No. | Field | Data Type | Description |
|-----|-------|-----------|-------------|
| 1 | id | string | Master Cask ID |
| 2 | caskName | string | Display name of the master cask (e.g., "Aberlour Ex-Sherry Hogshead") |
| 3 | distilleryName | string | Name of the distillery |
| 4 | caskType | string | Type of cask |
| 5 | caskImage | string | URL to the cask product image |
| 6 | lowestAsk | number \| null | MIN(lowestAsk) across all variants. `null` if no active asks |
| 7 | referencePriceMin | number | MIN(referencePriceMin) across all variants |
| 8 | referencePriceMax | number | MAX(referencePriceMax) across all variants |
| 9 | sortPriority | number | Returned by **Strategic Buys**: 1 = has active ask, 2 = no active ask |
| 10 | totalBids | number | Returned by **In High Demand**: Sum of all active bid quantities across variants |
| 11 | totalAsks | number | Returned by **In High Demand**: Sum of all active ask quantities across variants |
| 12 | totalActivityScore | number | Returned by **In High Demand**: `totalBids + totalAsks` |
| 13 | viewedAt | string | Returned by **Recently Viewed**: Timestamp of most recent view by user |

---

### 2. High Interest Casks 🔥

#### 2.1. Overview

> Returns the top 10 most viewed master casks on the platform sorted by viewCount DESC.

#### 2.2. Use Cases

##### UC-1. Fetch Top 10 High Interest Master Casks

| Item | Description |
|------|-------------|
| **Objective** | Returns the top 10 most viewed master casks on the platform sorted by viewCount DESC |
| **Actor** | System (triggered by FE) |
| **Trigger** | FE requests data on Home page load |
| **Pre-condition(s)** | 1. Master casks exist with at least one active variant<br>2. At least one master cask has viewCount > 0 |
| **Post-condition(s)** | Returns up to 10 master casks sorted by viewCount DESC |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Query Master Casks | CL-1 | Query only master casks that have at least one active variant |
| 2 | Retrieve View Count | CL-2 | viewCount is the master cask's own page view counter (not aggregated from variants). Includes all-time page views |
| 3 | Resolve Lowest Ask | CL-3, CL-4 | lowestAsk = MIN(lowestAsk) across all variants. If no variant has an active ask, lowestAsk is returned as null — FE handles fallback display using referencePriceMin |
| 4 | Resolve Ref Price Min | CL-5 | referencePriceMin = MIN(referencePriceMin) across all child variants |
| 5 | Resolve Image | CL-6 | image uses the master cask's primary image. If none set, use the first variant's imageUrl (by order field) |

**Happy Path**:

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Casks qualify for Top 10 | Return up to 10 master casks sorted by viewCount DESC, status 200 |

**Negative Path**:

| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | Unexpected server error | Return error status to caller | 500 — Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|--------------|
| 1 | All master casks have viewCount = 0 | Return empty array `[]` | FE hides the section |
| 2 | Fewer than 10 qualifying master casks exist | Return all qualifying casks (< 10 items) | FE renders available cards |
| 3 | A master cask is deactivated while in the list | Next API call excludes it | List may show fewer than 10 |

---

### 3. Strategic Buys

#### 3.1. Overview

> Returns master casks sorted to highlight the best entry points for buyers using a two-tier priority sort: lowest active ask price first, then reference price for master casks where no variants have active asks.

#### 3.2. Use Cases

##### UC-2. Fetch Strategic Buys (Two-Tier Priority Sort)

| Item | Description |
|------|-------------|
| **Objective** | Return master casks sorted by best buying opportunity: first by lowest ask, then by reference price for master casks where no variants have active asks |
| **Actor** | System (triggered by FE) |
| **Trigger** | FE requests Strategic Buys Data |
| **Pre-condition(s)** | 1. Master casks exist with at least one active, listed variant<br>2. At least one master cask has at least one variant with either an active ask or a referencePriceMin |
| **Post-condition(s)** | Returns up to 10 master casks with two-tier priority sort applied |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Query Master Casks | - | Query all master casks that have at least one active, listed variant |
| 2 | Retrieve Lowest Ask | CL-10 | For each master cask, find lowestAsk = MIN(lowestAsk) across all variants. Only considers active asks: ask.status = 'Active' AND ask.expirationDate > NOW() |
| 3 | Retrieve Reference Prices | - | Find referencePriceMin = MIN(referencePriceMin) and referencePriceMax = MAX(referencePriceMax) across all variants |
| 4 | Exclude Invalid Casks | CL-11 | If a master cask has no variants with active asks AND no variants with a referencePriceMin, it is excluded from the list entirely |
| 5 | Split and Sort | CL-9 | Group A (Priority 1): lowestAsk IS NOT NULL → sort by lowestAsk ASC. Group B (Priority 2): lowestAsk IS NULL → sort by referencePriceMin ASC |
| 6 | Secondary Sort | CL-12, CL-13 | If lowestAsk tied, sort by caskName ASC. If referencePriceMin tied, sort by caskName ASC. |
| 7 | Return Response | - | Concatenate Group A + Group B, limit to top 10 results |

**Happy Path**:

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Master casks exist with asks and reference prices | Return up to 10 master casks applying two-tier logic |

**Negative Path**:

| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | Unexpected server error | Return error status to caller | 500 — Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|--------------|
| 1 | All master casks have no variants with active asks AND no variants with a reference price | Return empty array `[]` | FE hides section |
| 2 | All master casks have at least one variant with active asks (no Priority 2 group) | Return Group A only, sorted by lowestAsk ASC | Normal render |
| 3 | No master casks have any variants with active asks (no Priority 1 group) | Return Group B only, sorted by referencePriceMin ASC | All cards show "Expected value" |
| 4 | An ask expires between the time FE loads and user clicks | FE navigates to cask detail showing live data | No staleness issue for list |

---

### 4. In High Demand

#### 4.1. Overview

> Returns master casks with the highest market activity, measured by aggregating all bid and ask quantities across all variants. Indicates market liquidity and interest.

#### 4.2. Use Cases

##### UC-3. Fetch In High Demand Casks

| Item | Description |
|------|-------------|
| **Objective** | Return master casks with the highest total market activity (bids + asks) to indicate demand and liquidity |
| **Actor** | System (triggered by FE) |
| **Trigger** | FE requests High Demand Data |
| **Pre-condition(s)** | 1. Master casks exist with at least one active, listed variant<br>2. At least one variant has active bids or asks |
| **Post-condition(s)** | Returns up to 10 master casks sorted by totalActivityScore DESC |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Query Master Casks | - | Query all master casks that have at least one active, listed variant |
| 2 | Calculate Total Bids | CL-14, CL-16 | Calculate totalBids = SUM(bid quantities) across all variants where bid is active. Sums quantities, not count of records |
| 3 | Calculate Total Asks | CL-15, CL-16 | Calculate totalAsks = SUM(ask quantities) across all variants where ask is active. Sums quantities, not count of records |
| 4 | Calculate Score | CL-17 | Calculate totalActivityScore = totalBids + totalAsks. Exclude master casks where totalActivityScore = 0 |
| 5 | Sort | CL-18 | Sort by totalActivityScore DESC. If tied, secondary sort by viewCount DESC |
| 6 | Return Response | - | Limit to top 10, resolve additional fields, return response |

**Happy Path**:

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Valid Casks exist | Return up to 10 master casks sorted by score |

**Negative Path**:

| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | Unexpected server error | Return error status to caller | 500 — Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|--------------|
| 1 | No master casks have variants with any active bids or asks | Return empty array `[]` | FE hides section |
| 2 | A bid/ask is matched/cancelled after list computed | Stale data is acceptable until next page load | None |
| 3 | A master cask has bids but no asks (or vice versa) | Still included. Score = bids + asks (one side can be 0) | None |

---

### 5. Recently Viewed

#### 5.1. Overview

> Returns casks that the authenticated user has recently visited, ordered by most recent view. This provides a personalized browsing history on the Home page.

#### 5.2. Use Cases

##### UC-4. Fetch Recently Viewed Master Casks

| Item | Description |
|------|-------------|
| **Objective** | Return the user's most recently viewed master casks for personalized Home page display |
| **Actor** | Authenticated User (via FE) |
| **Trigger** | FE requests Recently Viewed Data |
| **Pre-condition(s)** | 1. User is authenticated (valid Bearer token)<br>2. View history records exist for this user |
| **Post-condition(s)** | Returns up to 10 master casks sorted by `viewedAt` DESC |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Authenticate | - | Verify Bearer token. Anonymous users receive 401 |
| 2 | Query History | CL-19 | Query view history records server-side for this user at master cask level |
| 3 | Sort and Deduplicate | CL-20 | Sort by viewedAt DESC. Keep only the most recent entry if the same master cask was viewed multiple times |
| 4 | Filter Inactive Casks | CL-21 | Master casks that no longer have any active, listed variants are excluded from the recently viewed list |
| 5 | Resolve Fields | CL-22 | Map Parent fields. Map lowestAsk and referencePriceMin which reflect current (live) data |
| 6 | Return Response | - | Limit to top 10 results, return response |

**Happy Path**:

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | User has valid view history | Returns up to 10 master casks sorted by recently viewed |

**Negative Path**:

| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | User is not authenticated | API returns 401 | 401 — Unauthorized |
| 2 | Unexpected server error | Return error status to caller | 500 — Internal Server Error |

**Edge Cases**:

| # | Scenario | Expected Behavior | Side Effects |
|---|----------|-------------------|--------------|
| 1 | User has no view history | Return empty array `[]` | FE hides section |
| 2 | A previously viewed master cask loses all active variants | Excluded from results | None |

##### UC-5. Record Master Cask View Event

| Item | Description |
|------|-------------|
| **Objective** | Record when a user visits a master cask detail page, to populate their recently viewed list and increment viewCount |
| **Actor** | System (triggered when user navigates to master cask detail page) |
| **Trigger** | User loads a master cask detail page (`/casks/{masterCaskId}`) |
| **Pre-condition(s)** | Master cask has at least one active, listed variant |
| **Post-condition(s)** | 1. If user is authenticated: view event is recorded/updated<br>2. Master cask `viewCount` is incremented |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Receive Event | CL-23 | FE sends event; processing is async/fire-and-forget. Must not block detail page rendering |
| 2 | Increment View Count | CL-25 | Increment masterCask.viewCount by 1 for both authenticated and anonymous users |
| 3 | Record History (Auth) | CL-24 | If authenticated: upsert view history. To prevent view inflation from refreshes, same user viewing same master cask within 5 mins only counts as 1 view |

**Happy Path**:

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Authenticated User view event | Updates history and increments viewCount asynchronously |
| 2 | Anonymous User view event | Increments viewCount only asynchronously |

**Negative Path**:

| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | Failed async recording | Fail silently. Do not retry or block FE | N/A |

**(No specific edge cases for UC-5)**

## III. Data Model

### Entity: MasterCask (Parent) — existing, updated

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | ✅ | auto-generated | Primary key |
| name | string | ✅ | — | Cask name (global, shared across all variants) |
| distillery | string (FK) | ✅ | — | FK to Distillery |
| caskType | string (FK) | ✅ | — | FK to CaskType |
| classification | string (FK) | ✅ | — | FK to Classification (e.g., "Single Malt") |
| region | string (FK) | ✅ | — | FK to Region (e.g., "Highlands") |
| image | string (URL) | ✅ | — | Primary product image |
| viewCount | number | ✅ | 0 | **Page view counter** — incremented when master cask detail page is viewed. Used for High Interest sort |
| createdAt | datetime | ✅ | NOW() | Record creation timestamp |
| updatedAt | datetime | ✅ | NOW() | Last update timestamp |

### Entity: CaskVariant (Child) — existing, for reference

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | ✅ | auto-generated | Primary key |
| masterCaskId | UUID (FK) | ✅ | — | FK to MasterCask. Each child linked to exactly one parent |
| vintageYear | number | ✅ | — | Year of the vintage |
| referencePriceMin | number | ✅ | — | Minimum reference price (used for Strategic Buys fallback) |
| referencePriceMax | number | ✅ | — | Maximum reference price |
| order | number | ❌ | 0 | Display order within parent |
| ... | ... | ... | ... | Other variant-specific fields (distillationDate, abv, rla, ola, etc.) |

**Constraint**: A MasterCask must have ≥ 1 CaskVariant.

### Entity: CaskViewHistory — new

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | ✅ | auto-generated | Primary key |
| userId | UUID (FK) | ✅ | — | FK to User. The user who viewed the cask |
| masterCaskId | UUID (FK) | ✅ | — | FK to MasterCask. The master cask that was viewed |
| viewedAt | datetime | ✅ | NOW() | Timestamp of the most recent view by this user |
| createdAt | datetime | ✅ | NOW() | Record creation timestamp |
| updatedAt | datetime | ✅ | NOW() | Last update timestamp |

**Unique Constraint**: `(userId, masterCaskId)` — one record per user per master cask, updated on repeated views.

---

## IV. Notifications / Side Effects

| # | Trigger Event | Channel | Recipient | Description |
|---|--------------|---------|-----------|-------------|
| — | N/A | — | — | No notifications are triggered by curated list APIs. These are read-only endpoints with no user-facing side effects |

> Note: Recording a view event has a side effect of incrementing `viewCount`, but no notifications are sent.

---

## V. Permissions & Access Control

| Action / Flow | Anonymous | Buyer | Seller | Admin |
|---------------|-----------|-------|--------|-------|
| Load High Interest | ✅ | ✅ | ✅ | ✅ |
| Load Strategic Buys | ✅ | ✅ | ✅ | ✅ |
| Load In High Demand | ✅ | ✅ | ✅ | ✅ |
| Load Recently Viewed | ❌ (Denied) | ✅ (own data) | ✅ (own data) | ✅ (own data) |
| Record View Event | ✅ (viewCount only) | ✅ (viewCount + history) | ✅ (viewCount + history) | ✅ |

---

## VI. Performance Considerations

| # | Concern | Recommendation |
|---|---------|---------------|
| 1 | Curated list queries involve aggregations across variants, bids, and asks | Pre-compute and cache curated lists. Refresh every 5–15 minutes via background job (cron) |
| 2 | viewCount increment is high-frequency | Use async queue (e.g., Redis) to batch-write viewCount updates. Do not block API response |
| 3 | Strategic Buys requires JOIN with asks table | Create materialized view or denormalized `lowestAsk` field on MasterCask, updated when asks change |
| 4 | In High Demand requires SUM across bids and asks | Pre-compute `totalActivityScore` as a denormalized field, updated on bid/ask create/cancel events |

---

_End of Document 2_
