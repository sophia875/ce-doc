# Related Distilleries API Update Specification

This document defines the backend updates for the related distilleries endpoint (`GET /api/distilleries/{id}/related`). It highlights only the changes to field naming, counting logic, and similarity calculation.

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation of Related Distilleries API updates spec | Antigravity | 2026-08-06 |

---

## Required Updates

### 1. Field Rename (`caskCount` → `masterCaskCount`)
The field in the response payload indicating the quantity of casks for a distillery is renamed from `caskCount` to `masterCaskCount` to align with the parent-child product architecture (SCR-002).

- **Old Field**: `caskCount` (number)
- **New Field**: `masterCaskCount` (number)

---

### 2. Count Logic Update
The backend logic to calculate this count must switch from counting individual child vintages to counting parent Master Casks.

- **Old Logic**: Count of all active `CaskVariant` (child vintage) records associated with the distillery.
- **New Logic**: Count of unique, active `MasterCask` (parent) records associated with the distillery.
  ```sql
  masterCaskCount = COUNT(id) WHERE distillery_id = ? AND status = 'active'
  ```

---

### 3. Similarity Scoring & Matching Rules
The similarity score is calculated by summing the weights of all matching tags/criteria. The maximum possible score is `1.00`.

> [!NOTE]
> **Scoring System Update**: The scoring system has transitioned from the legacy integer-based model (weights: Company 12, Country 10, Region 9, Founding Year 8, Number of Casks 7, Status 4; max total score of 50) to a normalized decimal-based model (max total score of `1.00`). This ensures the scoring is consistent with other similarity matching endpoints (e.g., similar casks) while preserving the exact relative importance and proportions of each criterion (normalized by dividing each legacy weight by 50).

#### Tag Matching Criteria & Weights:

| Tag | Weight | Matching Rule |
| :--- | :---: | :--- |
| **Company** | **0.24** | Exact string match on parent company name (`candidate.company == source.company`). |
| **Country** | **0.20** | Exact string match on country (`candidate.country == source.country`). |
| **Region** | **0.18** | Exact string match on region (`candidate.region == source.region`). |
| **Founding Year** | **0.16** | Within `±3` years (inclusive) of the source distillery's established year (`source.establishedYear - 3 <= candidate.establishedYear <= source.establishedYear + 3`). |
| **Number of Casks** | **0.14** | Active inventory scale (`masterCaskCount`) is within `±15%` (inclusive) of the source distillery's `masterCaskCount`. Formula: `0.85 * source.masterCaskCount <= candidate.masterCaskCount <= 1.15 * source.masterCaskCount`. If source has `masterCaskCount = 0`, candidates with `masterCaskCount = 0` match. |
| **Status** | **0.08** | Candidate's status is active (`candidate.status == "active"`). |

#### Filter, Return, and Sorting Logic:
- **Filtering**: Only candidate distilleries with **at least one matching tag** (score > 0) are eligible to be returned. The source distillery is always excluded.
- **Limit**: Display up to **10** related distilleries.
- **Sorting**: Eligible candidates are ordered by:
  1. Similarity score descending (Highest score first).
  2. Status: `Active > Inactive` (active distilleries sort before inactive ones if score is tied).


## Changed Response Fields

The returned distillery payload is updated as follows:

| Field | Data Type | Required | Description |
|---|---|---|---|
| `masterCaskCount` | number | ✅ | Count of active Master Casks (`status = 'active'`). *Replaces `caskCount`.* |
