# Specification Update: Suggested Asks Mechanism

This document outlines the changes required to update the **Suggested Asks** calculation logic. 

## 1. Overview of Changes

* **Old Mechanism**: The system calculated and assigned the three suggested asks (Ask 1, Ask 2, Ask 3) statically based on the matched scenario in a formula table. This could result in duplicate suggestions or suggested asks that are lower than or equal to the Lowest Ask (LA).
* **New Mechanism**: The system first computes a set of "candidate values" using the same scenario formulas. It then processes the candidate array dynamically (removing duplicates, filtering out values <= LA, and sorting in ascending order). If the resulting array contains fewer than 3 values, fallback rules are applied. Finally, the values are mapped to the three suggested ask levels.

---

## 2. Terminology & Core Definitions

Please preserve the following standard terms in the codebase:
* **LA (Lowest Ask)**: The lowest price currently offered by a seller for the product.
* **HB (Highest Bid)**: The highest price currently offered by a buyer for the product.
* **LSP (Last Sale Price)**: The price of the most recent completed transaction for the product.
* **ASP (Average Sales Price)**: The average price of all past transactions.
* **RP (Reference Price)**: A platform-defined default price (e.g., manufacturer's suggested retail price). Used when no market data (LA, HB, LSP, ASP) is available. 
Calculated as the average of the Min and Max reference price set for the product:  
RP = (Min RP + Max RP) / 2
* **MV (Current Market Value)**: Ascertained via the priority order: LA > LSP > ASP > HB > RP.

---

## 3. The 4-Step Calculation Flow

### Step 1: Calculate Candidate Values
Based on the availability of market indicators, determine the current scenario and calculate three initial candidate values:

```text
Candidate Array = [Candidate 1, Candidate 2, Candidate 3]
```

| **Availability Scenario** | **Candidate 1** | **Candidate 2** | **Candidate 3** |
| :--- | :--- | :--- | :--- |
| Both LA and HB are available | `1.05 * LA` | `1.1 * LA` | `1.02 * HB` |
| LA and LSP are available, HB unavailable | `1.05 * LA` | `1.1 * LA` | `1.1 * LSP` |
| HB and LSP are available, LA unavailable | `1.05 * HB` | `1.1 * HB` | `1.1 * LSP` |
| Only LA is available, HB & LSP unavailable | `1.05 * LA` | `1.1 * LA` | `1.15 * LA` |
| Only HB is available, LA & LSP unavailable | `1.02 * HB` | `1.05 * HB` | `1.1 * HB` |
| Only LSP is available, HB & LA unavailable | `1.1 * LSP` | `1.15 * LSP` | `1.25 * LSP` |
| RP is available, LA, HB, LSP unavailable | `1.1 * RP` | `1.15 * RP` | `1.25 * RP` |

*Note: If no market data is available, fall back to Reference Price (RP).*

### Step 2: Process the Array
Apply the following operations in order to the calculated Candidate Array:
1. **Deduplication**: Remove duplicate values from the array.
2. **Filtering by Lowest Ask (LA)**:
   * If **LA is available**, remove any candidate value that is **less than or equal to LA** (`<= LA`). Suggested asks must always be strictly higher than LA.
   * If **LA is unavailable**, skip this filter.
3. **Sorting**: Sort the remaining values in **ascending order** to produce the processed array:
   ```text
   [V1, V2, V3, ...] where V1 < V2 < V3
   ```

### Step 3: Assign Suggested Asks
Map the processed values to the three suggested ask levels:
* **SA1 ("Conservative")** = `V1`
* **SA2 ("Moderate")** = `V2`
* **SA3 ("Aggressive")** = `V3`

### Step 4: Apply Fallback Rules (If array has < 3 elements after processing)
If the processed array has fewer than 3 elements after Step 2, apply the following rules:

* **Case A: 2 values remaining `[V1, V2]`**
  * `SA1 = V1`
  * `SA2 = V2`
  * `SA3 = V2 * 1.05` (Generates an aggressive ask by adding a 5% premium to the moderate ask)
* **Case B: 1 value remaining `[V1]`**
  * `SA1 = V1`
  * `SA2 = V1 * 1.05` (Generates a moderate ask with a 5% premium)
  * `SA3 = V1 * 1.10` (Generates an aggressive ask with a 10% premium)

---

## 4. Execution Examples

### Example 1: Standard Case (No fallback needed)
* **Market Inputs**:
  * Lowest Ask (LA) = 100
  * Highest Bid (HB) = 120
  * Last Sale Price (LSP) = 110
* **Step 1: Calculate Candidate Values**
  * Scenario matched: **Both LA and HB are available**.
  * `Candidate 1 = 1.05 * LA = 1.05 * 100 = 105`
  * `Candidate 2 = 1.10 * LA = 1.10 * 100 = 110`
  * `Candidate 3 = 1.02 * HB = 1.02 * 120 = 122.40`
  * `Candidate Array = [105, 110, 122.40]`
* **Step 2: Process the Array**
  * Deduplicate: No duplicates. `[105, 110, 122.40]`
  * Filter `<= LA` (100): All candidates are `> 100`. `[105, 110, 122.40]`
  * Sort: Already sorted. `[105, 110, 122.40]`
  * Result: `V1 = 105`, `V2 = 110`, `V3 = 122.40`
* **Step 3 & 4: Assignment**
  * **SA1 (Conservative)** = 105.00
  * **SA2 (Moderate)** = 110.00
  * **SA3 (Aggressive)** = 122.40

### Example 2: Fallback Case - 2 values remaining (Filtering removes 1 value)
* **Market Inputs**:
  * Lowest Ask (LA) = 100
  * Highest Bid (HB) = 95
  * Last Sale Price (LSP) is unavailable.
* **Step 1: Calculate Candidate Values**
  * Scenario matched: **Both LA and HB are available**.
  * `Candidate 1 = 1.05 * LA = 1.05 * 100 = 105`
  * `Candidate 2 = 1.10 * LA = 1.10 * 100 = 110`
  * `Candidate 3 = 1.02 * HB = 1.02 * 95 = 96.90`
  * `Candidate Array = [105, 110, 96.90]`
* **Step 2: Process the Array**
  * Deduplicate: No duplicates. `[105, 110, 96.90]`
  * Filter `<= LA` (100): 96.90 is less than 100, so it is filtered out. Remaining: `[105, 110]`
  * Sort: Already sorted. `[105, 110]`
  * Result: `V1 = 105`, `V2 = 110`
* **Step 3 & 4: Assignment (Case A Fallback)**
  * **SA1 (Conservative)** = `V1 = 105.00`
  * **SA2 (Moderate)** = `V2 = 110.00`
  * **SA3 (Aggressive)** = `V2 * 1.05 = 110 * 1.05 = 115.50`

### Example 3: Fallback Case - 1 value remaining (Hypothetical edge case)
*Note: While the current scenario formulas mathematically guarantee at least 2 unique candidates when values are positive, this example demonstrates the fallback behavior for 1 value (which handles future formula updates, manual admin pricing overrides, or cases where certain candidate formulas are disabled/null).*
* **Market Inputs**:
  * Lowest Ask (LA) = 100
  * Hypothetical Candidate Array (after calculation or override): `[105, 105, 95]`
* **Step 2: Process the Array**
  * Deduplicate: `[105, 95]`
  * Filter `<= LA` (100): 95 is less than 100, so it is filtered out. Remaining: `[105]`
  * Sort: `[105]`
  * Result: `V1 = 105`
* **Step 3 & 4: Assignment (Case B Fallback)**
  * **SA1 (Conservative)** = `V1 = 105.00`
  * **SA2 (Moderate)** = `V1 * 1.05 = 105 * 1.05 = 110.25`
  * **SA3 (Aggressive)** = `V1 * 1.10 = 105 * 1.10 = 115.50`
