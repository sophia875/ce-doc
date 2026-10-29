# BACKEND LOGIC & FLOW SPECS

Similar Cask Master API Specification

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation of Similar Cask Master API specs | Antigravity | 2026-07-24 |

---

## I. Introduction

### 1. Purpose

This document defines the requirements for a new backend endpoint to fetch similar casks at the **Master Cask (parent) level**. This endpoint is consumed by the **Cask Detail / Similar Cask Section** in the UI, replacing the previous vintage-centric similar casks endpoint. Similarity is determined by a weighted scoring system based on Master Cask attributes and the pre-computed floor price.

### 2. Definitions

| Term | Definition |
|------|------------|
| Source Master Cask | The Master Cask currently being viewed by the user. Its attributes are used as the reference point for calculations. |
| Candidate Master Cask | An active Master Cask being evaluated for similarity to the Source Master Cask. |
| Active Master Cask | A parent cask with `status = 'active'` that contains at least one active, listed child vintage (`caskStatus = 'active' AND isListed = true`). |
| Floor Price | The pre-computed minimum price of a Master Cask. Logic: `lowestAsk !== null ? lowestAsk : minReferencePrice`. |

### 3. Business Rules Summary

| Rule Code | Description | Condition / Example |
|-----------|-------------|---------------------|
| SC-BR-1 | Exclude Source Cask | The Source Master Cask itself must always be excluded from similarity evaluation. |
| SC-BR-2 | Active Candidates Only | Only active Master Casks can be considered as candidates. |
| SC-BR-3 | Scoring Weighting | Candidates are scored up to `1.0` based on match weights: Distillery (0.4), Floor Price (0.3), Region (0.1), Cask Type (0.1), and Classification (0.1). |
| SC-BR-4 | Floor Price Match Margin | A candidate receives `0.3` points if its `floorPrice` is within `±10%` (inclusive) of the source's `floorPrice`. Formula: `0.9 * source.floorPrice <= candidate.floorPrice <= 1.1 * source.floorPrice`. |
| SC-BR-5 | Score Greater than Zero | If all candidate casks score `0.0`, the similar casks section is hidden. The API returns an empty array `[]`. |
| SC-BR-6 | Selection Limit | A maximum of 4 candidate Master Casks with the highest scores are returned. |
| SC-BR-7 | Sorting & Tie-breakers | Results are sorted by score descending. For tie scores, follow tie-breaker order: `purchaseAvailability` DESC -> `activeBidsCount` DESC -> `lifetimeVolume` DESC -> alphabetical `name` (A-Z, case-insensitive). |

---

## II. Requirements

### 1. Similar Casks at Master Cask Level

#### 1.1. Overview
Retrieve a list of up to 4 similar active Master Casks based on scoring similarity weights. Used to populate the "Similar Casks" carousel/section on the Cask Detail screen.

#### 1.2. Data Description

##### Inputs (Path Parameter)

| Parameter | Data Type | Required | Description |
|:---|:---|:---:|:---|
| `id` | UUID | ✅ | The UUID of the source Master Cask. |

##### Outputs (Array of Card-Friendly Master Cask Objects)

| Field | Data Type | Required | Description |
|---|---|---|---|
| `id` | UUID | ✅ | Master Cask ID |
| `name` | string | ✅ | Master Cask display name |
| `imageUrl` | string \| null | ❌ | Image URL of the Master Cask |
| `distilleryName` | string | ✅ | Associated distillery name |
| `caskTypeName` | string | ✅ | Associated cask type name |
| `regionName` | string | ✅ | Associated region name |
| `classificationLabel`| string | ✅ | Display label for classification (e.g. "Single Malt Scotch") |
| `lowestAsk` | number \| null | ❌ | Lowest active ask price across child vintages |
| `floorPrice` | number | ✅ | Pre-computed floor price (`lowestAsk !== null ? lowestAsk : minReferencePrice`) |
| `minReferencePrice` | number | ✅ | Minimum reference price across child vintages |
| `lowestAskDelta30D` | number \| null | ❌ | Percent change in lowest ask over last 30 days (omitted if null) |
| `purchaseAvailability` | number | ✅ | Sum of active ask quantities |
| `minVintageYear` | number | ✅ | Earliest vintage year among active child vintages |
| `maxVintageYear` | number | ✅ | Latest vintage year among active child vintages |
| `similarityScore` | number | ✅ | Mapped similarity score (0.0 to 1.0) |

---

#### 1.3. Use Cases

##### UC-1. Fetch Similar Master Casks

| Item | Description |
|------|-------------|
| **Objective** | Retrieve up to 4 similar, active master casks scored and ordered based on similarity. |
| **Actor** | System (triggered by Frontend Cask Detail page load) |
| **Trigger** | Frontend requests similar casks for the current master cask ID. |
| **Pre-condition(s)** | The requesting user is authenticated. The source Master Cask ID is valid. |
| **Post-condition(s)** | Returns a JSON array of up to 4 scored Master Cask summary cards. |

**Activity Flow & Business Rules**:

| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Validate path param & authentication | SC-UC1-1 | If Bearer token is missing, return `401 Unauthorized`. If `id` does not exist in the database, return `404 Not Found`. |
| 2 | Query candidate master casks | SC-UC1-2 | Select all Master Casks where `status = 'active'`, excluding the source Master Cask (`id != sourceId`). Each candidate must have at least one active, listed vintage (`caskStatus = 'active' AND isListed = true`). |
| 3 | Apply scoring criteria | SC-UC1-3 | For each candidate, initialize `score = 0.0`. Accumulate points based on rules:<br>• Match `distilleryId`: `+0.4`<br>• `floorPrice` within `±10%` of source `floorPrice`: `+0.3`<br>• Match `regionId`: `+0.1`<br>• Match `caskTypeId`: `+0.1`<br>• Match `classification`: `+0.1` |
| 4 | Filter zero-score candidates | SC-UC1-4 | Exclude any candidate where `score == 0.0`. If all candidate casks have a score of `0.0`, return `200 OK` with an empty array `[]`. |
| 5 | Order by score & tie-breakers | SC-UC1-5 | Order candidate list by:<br>1. `score` DESC<br>2. `purchaseAvailability` DESC<br>3. `activeBidsCount` DESC<br>4. `lifetimeVolume` DESC<br>5. `name` ASC (case-insensitive alphabetical) |
| 6 | Limit results | SC-UC1-6 | Take the top 4 candidates from the sorted list. |
| 7 | Return response | SC-UC1-7 | Return `200 OK` with the JSON array of mapped card-friendly Master Cask objects. |

**Happy Path**:
- Source cask detail page loaded. Similar casks endpoint returns `200 OK` with an array containing 1 to 4 scored similar Master Cask card-friendly objects.

**Negative Path**:
- Invalid / missing JWT token -> Returns `401 Unauthorized`.
- Non-existent Master Cask ID -> Returns `404 Not Found` with message `"Cask Master not found"`.

**Edge Cases**:
- **Source Cask has no similar items (all score 0)**: Returns `200 OK` with `[]` payload. The frontend will hide the "Similar Casks" section.

---

## III. Data Model

Uses the existing `MasterCask`, `CaskVariant` and `MasterCaskMarketStats` models. Refer to the Marketplace Listing spec ([api-requirements.md](file:///d:/CX/docs/marketplace/api-requirements.md)) for the detailed fields.

---

## IV. Notifications / Side Effects

None. This is a read-only query endpoint.

---

## V. Permissions & Access Control

| Use Case / Action | Anonymous | Buyer | Seller | Admin |
|-------------------|-----------|-------|--------|-------|
| Fetch Similar Master Casks | ❌ | ✅ | ✅ | ✅ |

_End of Document 2_
