# Specification Update: Order Matching - Composite Match Logic

This document outlines the changes required for the Backend (BE) Order Matching engine, specifically focusing on clarifying and correcting the **Composite Match** logic when matching opposing orders.

---

## 1. Identified Issue

Under the current matching engine implementation, when the highest-priority candidate in the waitlist cannot fully satisfy an incoming opposing order's target quantity, and the waitlist contains a mixture of both **Full** and **Partial** orders, the system fails to achieve maximum fulfillment—even when a combination of other available orders could satisfy 100% of the target.

### Real-World Example (Bug Scenario)
* **Target Quantity** (Incoming Ask): `5`
* **Waitlist** (Active Bids at the same price level):
  * **Partial Bid**: `4 units`
  * **Full Bid A**: `3 units`
  * **Full Bid B**: `2 units`
* **Current Behavior**: The system matches only **4 out of 5 units** (using the Partial Bid because it has the largest quantity).
* **Expected Behavior**: The system should combine Full Bid A (3) + Full Bid B (2) to achieve a complete **5 out of 5 units** match (100% fulfillment).

---

## 2. Root Cause

The matching engine was applying the standard **Order Priority** rules (specifically prioritizing **Quantity over Fulfillment Preference**, from Section D.1.3 of the spec) to select candidates during the composite accumulation process. This resulted in the system greedily selecting the largest available order (whether Full or Partial) first, which disrupted the combination logic for subsequent orders.

However, Section D.2.2 (Composite Match) of the original specification states:
> *"For Full orders: Must find orders that sum exactly to target. Can combine: Multiple Full orders, or Full + split Partial orders."*

This implies that **Full orders must be accumulated and combined first**, while Partial orders should only act as a "split" order to fill the remaining residual quantity, rather than evaluating Full and Partial orders mixed together solely by quantity.

---

## 3. Updated Specifications & Rules

To resolve this issue, candidate priority sorting and the composite match process have been clarified.

### 3.1. Candidate Selection Priority (Section D.1.3 - Unchanged)
For reference, candidate selection sorting remains as follows:
1. **Price** (For Bids: highest first; for Asks: lowest first)
2. **Quantity** (Larger first)
3. **Fulfillment Preference** (`Full` > `Partial` as a tie-breaker when Price and Quantity are equal)
4. **Timestamp** (Earlier `priority_timestamp` first)

### 3.2. Composite Match Processing Rule (Section D.2.2 - Clarified)
When executing a Composite Match, the matching engine must split candidates into **Full** and **Partial** groups at the matching price level and execute the following two-step logic:

#### **Step 1: Accumulate Full Orders First**
1. Filter the candidate list at the matching price level to retrieve only **Full** orders.
2. Sort these Full orders in **descending order of quantity**.
3. Iterate and accumulate the Full orders sequentially:
   - For each Full order, check if adding its quantity to the current running total exceeds the target quantity.
   - If it **exceeds the target**, discard this order and proceed to the next Full order in the list.
   - If it **does not exceed the target**, add it to the running total.
4. **Stopping Condition**: Stop the accumulation loop as soon as:
   $$\text{Running Total of Full Orders} + \text{Max Partial Quantity} \ge \text{Target Quantity}$$
   *(where `Max Partial Quantity` is the remaining quantity of the highest-priority Partial order in the candidate list).*

#### **Step 2: Fill the Residual with a Partial Order**
1. Keep the accumulated Full orders from Step 1 fixed.
2. Calculate the remaining quantity required to satisfy the target:
   $$\text{Residual Quantity} = \text{Target Quantity} - \text{Running Total of Full Orders}$$
3. Select the single highest-priority **Partial** order from the candidate list.
4. Fulfill the residual quantity using this Partial order (i.e., deduct only the required `Residual Quantity` from the Partial order's remaining quantity; do not consume its entire quantity unless it exactly matches the residual).

---

## 4. Comparison: Legacy vs. Updated Logic

### Example Dataset
* **Target Quantity**: `25`
* **Max Partial**: `9` (Highest-priority Partial order)
* **Full Orders**: `A = 8`, `B = 7`, `C = 6`, `D = 5`, `E = 3` (Sorted descending by quantity)

### Legacy Logic (Mixed sorting by quantity first)

| Step | Candidate Evaluated | Accumulated Quantity | Action / Result |
| :--- | :--- | :--- | :--- |
| **1** | Partial (9) | 9 | **Add** (9) |
| **2** | Full A (8) | 17 | **Add** (8) |
| **3** | Full B (7) | 24 | **Add** (7) |
| **4** | Full C (6) | 30 > 25 | **Discard** (Exceeds Target) |
| **5** | Full D (5) | 29 > 25 | **Discard** (Exceeds Target) |
| **6** | Full E (3) | 27 > 25 | **Discard** (Exceeds Target) |

* **Total Matched**: **24 / 25** (Unfulfilled by 1 unit, despite candidates C, D, and E remaining on the waitlist).
* **Cause of Failure**: The Partial order of 9 was matched first due to its larger quantity, causing subsequent Full orders to overshoot the target quantity.

---

### Updated Logic (Accumulate Full Orders First, then Fill Residual)

**Step 1: Accumulate Full Orders**

| Step | Full Candidate Evaluated | Running Total of Full Orders | Stop Condition Check (`Running Total + Max Partial (9) >= 25`) | Action |
| :--- | :--- | :--- | :--- | :--- |
| **1** | Full A (8) | 8 | `8 + 9 = 17 < 25` | **Accumulate** |
| **2** | Full B (7) | 15 | `15 + 9 = 24 < 25` | **Accumulate** |
| **3** | Full C (6) | 21 | `21 + 9 = 30 >= 25` | **Accumulate & STOP** |

**Step 2: Fill Residual**
* **Selected Full Orders**: `A (8) + B (7) + C (6) = 21`
* **Residual Quantity**: `25 - 21 = 4`
* **Fulfillment from Partial**: Take exactly **4 units** from the Partial order (since 4 <= 9).

* **Total Matched**: **25 / 25** (100% Fulfill!).
* **Result**: `A(8) + B(7) + C(6) + Partial(4/9) = 25`.

---

### Side-by-Side Comparison Summary

| Metric | Legacy Logic | Updated Logic |
| :--- | :--- | :--- |
| **Fulfillment Rate** | 24 / 25 (Incomplete) | **25 / 25 (100% matched)** |
| **Number of Orders Used** | 3 | **4** |
| **Core Characteristics** | Partial order blocks subsequent Full orders, causing overshoot and partial failure. | Full orders accumulate freely; Partial order acts as a flexible plug for the remaining balance. |

---

## 5. Validation Against Original Spec Examples

The updated logic is verified to be 100% backward-compatible with the existing examples in Section D of the original specification document:

* **Example 1 (All Full: A=50, B=40, C=10, Target=60)**:
  * Full orders: `A(50)`, `B(40)`, `C(10)`. Max Partial: None.
  * Accumulation:
    - Add A (50) -> Running total = 50.
    - Check B (40) -> 50 + 40 = 90 > 60 -> Discard B.
    - Check C (10) -> 50 + 10 = 60 -> Add C.
  * **Result**: A(50) + C(10) = 60. Matches original spec example.

* **Example 2 (Full A=50, Partial B=15, Full C=5, Target=60)**:
  * Full orders: `A(50)`, `C(5)`. Max Partial: `B(15)`.
  * Accumulate Full orders:
    - Add A (50) -> Running total = 50.
    - Check stop condition: `50 + 15 = 65 >= 60`. Stop condition met. Stop accumulation (C is not evaluated/used).
  - Fill residual: `60 - 50 = 10`. Take 10 units from Partial B.
  - **Result**: A(50) + B(10/15) = 60. Matches original spec example (Bid C is not used).

---

## 6. Complexity & Implementation Effort

* **Linear Complexity ($\mathcal{O}(N)$)**: This is a single-pass iteration ("loop once, skip if overshoot") over the filtered Full orders list. It does not require calculating combinations (subset-sum permutations), maintaining excellent performance.
* **Low Implementation Effort**: Backend developers can reuse the existing accumulation logic (already implemented for Example 1) with minor modifications to group Full and Partial orders separately and perform a simple subtraction to allocate the residual quantity to the partial order.

---

## 7. Known Limitation (Deferred for MVP)

* **Subset-Sum Permutations with Only Full Orders**: When the waitlist contains **only Full orders** (meaning no Partial order is available to fill the residual), and sequential accumulation based on priority fails because a higher-priority order blocks a perfect lower-priority combination.
  * **Example Scenario**:
    - Target = 10.
    - Waitlist contains Full orders: `A = 6`, `B = 5`, `C = 5` (Priority: A > B > C).
    - Sequential accumulation: Add A (6) -> try B (5) (6+5=11 > 10, discard) -> try C (5) (6+5=11 > 10, discard) -> Matches only 6/10.
    - However, combining `B (5) + C (5) = 10` would achieve 100% matching, but it requires skipping the higher-priority order A.
* **Decision**: This edge case is deferred to the backlog / V2 because:
  - It requires a confluence of rare conditions (multiple Full orders at the same price level, zero Partial orders, specific quantities causing overshoot).
  - The impact is low: it only results in slightly suboptimal immediate fulfillment (the remaining quantities remain active on the waitlist for future matches), without causing any financial or transactional discrepancies.
  - We will re-evaluate in V2 when liquidity increases and real-world trading data indicates a significant impact on fulfillment rates.
