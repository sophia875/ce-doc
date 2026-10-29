# Bug Ticket: Fix Price Warning Validation Logic for Place Bid & Place Ask

## 1. Ticket Overview
* **Component**: Backend (API & Business Logic)
* **Feature**: Place Bid & Place Ask Price Warnings
* **Impact**: Critical. Users are either not receiving warnings for extremely off-market prices, or they are seeing incorrect alerts for reasonable prices close to the 80% threshold.

---

## 2. Problem Description & Root Causes

When a user places a Bid or Ask, the backend validates the input price against the cask's **Current Market Value (MV)**. 
* A price warning should trigger if the price is **out of bounds**: `Price <= 80% MV` (Low Warning) or `Price >= 120% MV` (High Warning).
* No warning should display if the price is in the **safe range**: `80% MV < Price < 120% MV`.

Currently, this validation is failing across multiple edge cases, particularly when `MV` falls back to lower-priority indicators, or when the calculation logic uses incorrect thresholds.

### Standard MV Fallback Precedence (For Reference)
Per current specifications, `MV` must be resolved using the following order of precedence:
`LA (Lowest Ask) -> LSP (Last Sale Price) -> ASP (Average Sales Price) -> HB (Highest Bid) -> RP (Reference Price)`
*Where RP = (Min RP + Max RP) / 2*

---

## 3. Specific Bug Scenarios

### A. Place Ask Warnings Not Triggering (Low/High Ask Warnings)
1. **MV Falls Back to Average Reference Price (RP)**:
   * **Case**: Cask has no active trades or bids. `MV` is resolved to `RP = 80,000`.
   * **Issue**: Inputting prices below 80% MV or above 120% MV triggers **no warning**.
   * **Examples**:
     * Lower bound (`80%` of 80,000 = `64,000` & `63,999`) -> No warning.
     * Upper bound (`120%` of 80,000 = `96,000` & `96,001`) -> No warning.
   * **Expected**: Warn if Ask <= 64,000 or Ask >= 96,000.
2. **MV Falls Back to HB (Highest Bid)**:
   * **Case**: Cask has `HB` available but `LA` and `LSP` are missing. `MV` is resolved to `HB`.
   * **Issue**: No warnings are shown.
   * **Expected**: Warn if Ask <= 0.8 * HB or Ask >= 1.2 * HB.

### B. Incorrect Price Warning Threshold Bounds (Place Ask & Place Bid)
* **Issue**: 
  1. Inputting a price close to, but strictly within, the safe range (e.g., `85% MV`) still triggers a price warning.
  2. Inputting a price >= 120% MV does **not** trigger any warning.
* **Affected Scenarios**: This incorrect boundary logic occurs when:
  1. `MV` is resolved to `LA` (Lowest Ask).
  2. `MV` is resolved to `LSP` (Last Sale Price).
* **Expected**: Correct the boundary math. Ensure warnings only trigger when `Price <= 0.8 * MV` or `Price >= 1.2 * MV`. The safe zone `(80% MV < Price < 120% MV)` must never trigger a warning.

### C. Place Bid Warnings Not Triggering (Low/High Bid Warnings)
1. **MV Falls Back to Average Reference Price (RP)**:
   * **Case**: `MV` resolves to `RP`.
   * **Issue**: Bidding <= 80% or >= 120% of `RP` triggers no warning.
   * **Expected**: Warn if Bid <= 0.8 * RP or Bid >= 1.2 * RP.
2. **MV Falls Back to LSP (Last Sale Price)**:
   * **Case**: `LA` is unavailable, but `LSP` is available. `MV` is resolved to `LSP`.
   * **Issue**: Bids trigger no warning.
   * **Expected**: Warn if Bid <= 0.8 * LSP or Bid >= 1.2 * LSP.

### D. Missing High Bid Warning when MV = LA (Lowest Ask)
* **Case**: `MV` is resolved to `LA`.
* **Issue**: Inputting a Bid >= 120% of `LA` triggers no warning.
* **Expected**: High bid warning must trigger for Bid >= 1.2 * LA.

### E. Missing Low Bid Warning when MV = HB (Highest Bid)
* **Case**: `MV` is resolved to `HB`.
* **Issue**: Inputting a Bid <= 80% of `HB` triggers no warning.
* **Expected**: Low bid warning must trigger for Bid <= 0.8 * HB.

---

## 4. Acceptance Criteria for Backend Implementation

### AC 1: Unified MV Computation Engine
Ensure the backend helper method that resolves `MV` for a Cask evaluates the fallbacks in the exact order of precedence:
1. `LA` (Lowest Ask)
2. `LSP` (Last Sale Price)
3. `ASP` (Average Sales Price)
4. `HB` (Highest Bid)
5. `RP` (Reference Price, computed as: `(Min RP + Max RP) / 2`)

### AC 2: Consistent Warning Logic Apply to All MV Types
Regardless of which price metric `MV` resolves to, the price warning evaluation must be applied consistently:
* **Low Warning condition**: `Price <= 0.8 * MV`
* **High Warning condition**: `Price >= 1.2 * MV`
* **No Warning (Safe Zone)**: `0.8 * MV < Price < 1.2 * MV`

This evaluation must be correctly applied to:
* `GET /api/cask-asks/warning/{caskId}?askPrice=`
* `GET /api/cask-bids/warning/{caskId}?bidPrice=`
* The validate and analytics endpoints: `/api/cask-asks/validate/{caskId}`, `/api/cask-bids/validate`, `/api/cask-asks/analytics/{caskId}`, `/api/cask-bids/analytics/{caskId}`
* The final creation endpoints: `POST /api/cask-bids` and `POST /api/cask-asks`

### AC 3: API Response Consistency
For warning checks, the endpoints must return the correct warning indicators. E.g.:
```json
{
  "hasWarning": true,
  "warningType": "LOW_PRICE" // or "HIGH_PRICE"
}
```
If the price is within the safe range `(80% MV, 120% MV)`:
```json
{
  "hasWarning": false,
  "warningType": "NONE"
}
```
