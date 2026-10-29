# Specification Update: Suggested Bids Mechanism

This document outlines the changes required to update the **Suggested Bids** calculation logic.

## 1. Overview of Changes

* **Old Mechanism**: The system calculated and assigned the three suggested bids (Bid 1, Bid 2, Bid 3) statically based on the matched scenario in a formula table. This could result in duplicate suggestions, or suggested bids that are higher than or equal to the Lowest Ask (LA).
* **New Mechanism**: The system first computes a set of "candidate values" using the same scenario formulas. It then processes the candidate array dynamically (removing duplicates, filtering out values >= LA, and sorting in ascending order). If the resulting array contains fewer than 3 values, fallback rules are applied. Finally, the values are mapped to the three suggested bid levels.

---

## 2. Terminology & Core Definitions

Please preserve the following standard terms in the codebase:
* **LA (Lowest Ask)**: The lowest price currently offered by a seller for the product.
* **HB (Highest Bid)**: The highest price currently offered by a buyer for the product.
* **LSP (Last Sale Price)**: The price of the most recent completed transaction for the product.
* **ASP (Average Sales Price)**: The average price of all past transactions.
* **RP (Reference Price)**: A platform-defined default price (e.g., manufacturer's suggested retail price).
  * **Note**: The Reference Price (RP) used for calculations must be computed as the average of the minimum and maximum reference prices:
    $$RP = \frac{RP_{min} + RP_{max}}{2}$$
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
| Both LA and HB are available | `0.95 * LA` | `0.90 * LA` | `0.98 * HB` |
| LA and LSP are available, HB unavailable | `0.95 * LA` | `0.90 * LA` | `0.90 * LSP` |
| HB and LSP are available, LA unavailable | `0.95 * HB` | `0.90 * HB` | `0.90 * LSP` |
| Only LA is available, HB & LSP unavailable | `0.95 * LA` | `0.90 * LA` | `0.85 * LA` |
| Only HB is available, LA & LSP unavailable | `0.98 * HB` | `0.95 * HB` | `0.90 * HB` |
| Only LSP is available, HB & LA unavailable | `0.90 * LSP` | `0.85 * LSP` | `0.75 * LSP` |
| RP is available, LA, HB, LSP unavailable | `0.90 * RP` | `0.85 * RP` | `0.75 * RP` |

*Note: If no market data is available, fall back to the calculated Reference Price (RP).*

### Step 2: Process the Array
Apply the following operations in order to the calculated Candidate Array:
1. **Deduplication**: Remove duplicate values from the array.
2. **Filtering by Lowest Ask (LA)**:
   * If **LA is available**, remove any candidate value that is **greater than or equal to LA** (`>= LA`). Suggested bids must always be strictly less than LA.
   * If **LA is unavailable**, skip this filter.
3. **Sorting**: Sort the remaining values in **ascending order** to produce the processed array:
   ```text
   [V1, V2, V3, ...] where V1 < V2 < V3
   ```

### Step 3: Assign Suggested Bids
Map the processed values to the three suggested bid levels:
* **SB1 ("Good Bid")** = `V1`
* **SB2 ("Better Bid")** = `V2`
* **SB3 ("Buy Faster")** = `V3`

### Step 4: Apply Fallback Rules (If array has < 3 elements after processing)
If the processed array has fewer than 3 elements after Step 2, apply the following rules:

* **Case A: 2 values remaining `[V1, V2]`**
  * `SB1 = V1 * 0.95` (Generates a lower bid by applying a 5% discount to the lower remaining value)
  * `SB2 = V1`
  * `SB3 = V2`
* **Case B: 1 value remaining `[V1]`**
  * `SB1 = V1 * 0.90` (Generates a lower bid by applying a 10% discount)
  * `SB2 = V1 * 0.95` (Generates a medium bid by applying a 5% discount)
  * `SB3 = V1`
* **Case C: 0 values remaining** *(Safety fallback if all candidates are filtered out)*
  * `SB1 = LA * 0.90`
  * `SB2 = LA * 0.95`
  * `SB3 = LA * 0.98`

---

## 4. Execution Examples

### Example 1: Standard Case (No fallback needed)
* **Market Inputs**:
  - Lowest Ask (LA) = 500
  - Highest Bid (HB) = 480
  - Last Sale Price (LSP) = 460
* **Step 1: Calculate Candidate Values**
  - Scenario matched: **Both LA and HB are available**.
  - `Candidate 1 = 0.95 * LA = 0.95 * 500 = 475.00`
  - `Candidate 2 = 0.90 * LA = 0.90 * 500 = 450.00`
  - `Candidate 3 = 0.98 * HB = 0.98 * 480 = 470.40`
  - `Candidate Array = [475.00, 450.00, 470.40]`
* **Step 2: Process the Array**
  - Deduplicate: No duplicates. `[475.00, 450.00, 470.40]`
  - Filter `>= LA` (500): All values are < 500. Remaining: `[475.00, 450.00, 470.40]`
  - Sort (ascending): `[450.00, 470.40, 475.00]`
  - Result: `V1 = 450.00`, `V2 = 470.40`, `V3 = 475.00`
* **Step 3 & 4: Assignment**
  - **SB1 (Good Bid)** = 450.00
  - **SB2 (Better Bid)** = 470.40
  - **SB3 (Buy Faster)** = 475.00

### Example 2: Fallback Case - 2 values remaining (Filtering removes 1 value)
* **Market Inputs**:
  - Lowest Ask (LA) = 500
  - Highest Bid (HB) = 515 (Anomalous case where HB >= LA)
  - Last Sale Price (LSP) is unavailable.
* **Step 1: Calculate Candidate Values**
  - Scenario matched: **Both LA and HB are available**.
  - `Candidate 1 = 0.95 * LA = 0.95 * 500 = 475.00`
  - `Candidate 2 = 0.90 * LA = 0.90 * 500 = 450.00`
  - `Candidate 3 = 0.98 * HB = 0.98 * 515 = 504.70`
  - `Candidate Array = [475.00, 450.00, 504.70]`
* **Step 2: Process the Array**
  - Deduplicate: No duplicates. `[475.00, 450.00, 504.70]`
  - Filter `>= LA` (500): 504.70 is `>= 500`, so it is filtered out. Remaining: `[475.00, 450.00]`
  - Sort (ascending): `[450.00, 475.00]`
  - Result: `V1 = 450.00`, `V2 = 475.00`
* **Step 3 & 4: Assignment (Case A Fallback)**
  - **SB1 (Good Bid)** = `V1 * 0.95` = `450.00 * 0.95` = 427.50
  - **SB2 (Better Bid)** = `V1` = 450.00
  - **SB3 (Buy Faster)** = `V2` = 475.00

### Example 3: Fallback Case - 1 value remaining (Hypothetical edge case)
*Note: This demonstrates the fallback behavior for 1 value (which handles future formula updates, manual pricing overrides, or edge cases).*
* **Market Inputs**:
  - Lowest Ask (LA) = 100
  - Hypothetical Candidate Array (after calculation or override): `[95, 95, 105]`
* **Step 2: Process the Array**
  - Deduplicate: `[95, 105]`
  - Filter `>= LA` (100): 105 is filtered out because it is >= 100. Remaining: `[95]`
  - Sort: `[95]`
  - Result: `V1 = 95`
* **Step 3 & 4: Assignment (Case B Fallback)**
  - **SB1 (Good Bid)** = `V1 * 0.90` = `95.00 * 0.90` = 85.50
  - **SB2 (Better Bid)** = `V1 * 0.95` = `95.00 * 0.95` = 90.25
  - **SB3 (Buy Faster)** = `V1` = 95.00
