# Backend Specification: Expiration & Cascade Cancellation Handling (Partial Reversion Version)

## 1. Overview
This document specifies the technical design for managing expiration checks, cascading status updates, and quantity release logic for **Bids**, **Asks**, **Payments**, and **Payouts** in the backend.

The system must run periodic checks to expire stale records, handle cancellation dependencies between payments and payouts, and correctly adjust marketplace inventory when transactions fall through.

### 1.1 Background & Existing Limitations
Currently, the backend schedules several daily cron jobs for expiration checks:
- **00:00 (Midnight):** `handleOrderExpiry()` runs to check order expiration.
- **01:00 / 02:00:** `handleExpiredOrdersUpdate()` and `handleDailyCleanup()` (annotated with `@Cron(CronExpression.EVERY_DAY_AT_2AM)`) run daily cleanups and automated status transitions for orders past their expiration.

**Key Issues with the Current Implementation:**
1. **Lack of Real-time Precision:** A once-daily run is insufficient for time-sensitive expiration, causing delayed status updates that are not reflected on the Frontend in a timely manner.
2. **Missing Cascade Logic:** No mechanism exists to propagate status changes across linked entities (e.g., canceling a parent Payment when a linked Payout expires, or vice versa).
3. **No Automatic Inventory Release:** There is no flow to release locked quantities back to the marketplace when payments or payouts are canceled/rejected.

---

## 2. Cron Jobs & Scheduling Logic

To handle record expiration, the backend implements a unified status-transition service that can be triggered via two scheduling approaches: **EOD (End of Day) Cron** and **High-Frequency Polling or Dynamic Scheduling**.

### 2.1 EOD Expiration Cron
- **Schedule:** Run daily at the end of the day (e.g., `0 0 * * *` or `23:59:00`).
- **Target:** All active `Bids`, `Asks`, `Payments`, and `Payouts` that have reached their expiration timestamp.
- **Action:** Transition their status to `EXPIRED`.

### 2.2 Short-Window Expiration Handling
For records that are scheduled to expire **on the following day** (within the next 24 hours), we need higher precision. The backend can implement one of the two options below:

#### Option A: High-Frequency Polling (Cron)
- **Schedule:** Run every 5 minutes (`*/5 * * * *`).
- **Target:** Active records whose expiration time falls within the next 24 hours and is less than or equal to the current time.
- **Action:** Call the unified expiration handler to mark them as `EXPIRED`.

#### Option B: Dynamic Scheduling / Task Queuing (Recommended)
Instead of polling database tables every 5 minutes, we schedule a one-off delayed job (e.g., via Redis, BullMQ, or a cloud-native task scheduler) targeting the exact expiration timestamp ($T_{\text{expire}}$).

- **Execution Flow:**
  1. When a record is created or updated, schedule a task at $T_{\text{expire}}$.
  2. At execution time $T_{\text{expire}}$, fetch the current record from the database.
  3. **Verification Check:** Compare the scheduled time with the record's current expiration time.
     - **If the expiration time has NOT changed (not extended):** Mark the record as `EXPIRED` and execute the status transition logic.
     - **If the expiration time has changed (extended or modified):** Do nothing (cancel this execution). A new job will have been scheduled for the new timestamp.

```mermaid
flowchart TD
    Start([Job Triggered at T_expire]) --> GetRecord[Fetch current record from DB]
    GetRecord --> CheckTimestamp{Is current_expire_time == T_expire?}
    CheckTimestamp -- Yes --> RunTransition[Execute Unified Expiration Handler]
    CheckTimestamp -- No (Extended) --> DoNothing([Do Nothing & Terminate])
    RunTransition --> End([Done])
```

> [!IMPORTANT]
> Both the EOD Cron and the short-window scheduler must invoke the **Unified Expiration Handler** to guarantee consistency in status transitions and business logic.

---

## 3. Cascading Cancellations (Interdependent Status Changes)

There is a **1-to-Many (1:N)** relationship between **Payments** and **Payouts**. 
- A single **Payment** is associated with one or more **Payouts** (e.g., `Payment A` is linked to `Payout B`, `Payout C`, and `Payout D`).

When a Payment or a Payout expires, is canceled, or is rejected, the status change must cascade through the linked entities.

### 3.1 Payout Expiration / Cancellation Cascade
If a specific **Payout** expires, is canceled, or is rejected:
1. The associated **Payment** is transitioned to `CANCELED`.
2. **All other Payouts** linked to that same Payment are also transitioned to `CANCELED`.

*Example Scenario:*
- `Payment A` $\rightarrow$ `Payout B`, `Payout C`, `Payout D`.
- `Payout B` expires $\rightarrow$ Status of `Payment A` becomes `CANCELED` $\rightarrow$ Status of `Payout C` and `Payout D` become `CANCELED`.

### 3.2 Payment Expiration / Cancellation Cascade
If a **Payment** expires, is canceled, or is rejected:
1. **All Payouts** linked to that Payment are transitioned to `CANCELED`.

*Example Scenario:*
- `Payment A` $\rightarrow$ `Payout B`, `Payout C`, `Payout D`.
- `Payment A` expires $\rightarrow$ Status of `Payout B`, `Payout C`, and `Payout D` become `CANCELED`.

```mermaid
graph TD
    subgraph Scenario 1: Payout B Expires/Fails
        PayoutB[Payout B: EXPIRED] -->|Trigger Cascade| PaymentA_C[Payment A: CANCELED]
        PaymentA_C --> PayoutC_C[Payout C: CANCELED]
        PaymentA_C --> PayoutD_C[Payout D: CANCELED]
    end

    subgraph Scenario 2: Payment A Expires/Fails
        PaymentA_Exp[Payment A: EXPIRED] -->|Trigger Cascade| PayoutB_C[Payout B: CANCELED]
        PaymentA_Exp --> PayoutC_C2[Payout C: CANCELED]
        PaymentA_Exp --> PayoutD_C2[Payout D: CANCELED]
    end
```

### 3.3 DocuSign Agreement Cancellation
When a Payout is transitioned to `CANCELED` (either directly due to its own expiration/failure or indirectly via cascade from a canceled Payment), the associated DocuSign agreement for that Payout must also be canceled/voided.
- **Rule:** If a Payout transitions to `CANCELED` status, the backend must trigger the necessary API call to void/cancel the corresponding DocuSign agreement package to prevent outstanding signing requests.

---

## 4. Quantity Release & Inventory Reversion (Fault-Based Rules)

In this version, quantity release is determined by which entity caused the failure (the **Triggering Entity** vs the **Cascaded Entities**).

### 4.1 Definitions
- **Triggering Entity:** The specific Payment or Payout that directly expired, failed, was rejected, or was manually canceled.
- **Cascaded Entity:** Any associated Payment or Payout that was automatically canceled as a consequence of the triggering entity's status change.

### 4.2 Reversion Rules Matrix

| Entity Type | Expiration / Failure Cause | Reversion Action | Detail |
| :--- | :--- | :--- | :--- |
| **Triggering Entity** | Direct expiration, rejection, or cancellation (Faulty Party) | **No Action** | The quantity associated with this specific entity is **not** released back to the market. |
| **Cascaded Entity** | Programmatic cancellation due to cascade (Innocent Party) | **Revert Quantity** | 1. If its associated Bid/Ask has **expired**: Do nothing.<br>2. If its associated Bid/Ask is **active**: Decrease `filled_qty` and increase `remaining_qty`. |

### 4.3 Examples of Quantity Release Behavior

#### Example 1: Payment A Expires / Fails
- **Relation:** `Payment A` (10 units) $\rightarrow$ `Payout B` (4 units), `Payout C` (6 units).
- **Event:** `Payment A` expires (becomes the **Triggering Entity**).
- **Result:**
  - `Payment A` is the triggering entity: **No units** from `Payment A` are returned to the market.
  - `Payout B` and `Payout C` are cascaded entities:
    - If their associated Asks are still **active**, their units (4 and 6 units respectively) are returned to the market.
    - If their associated Asks have **expired**, no action is taken.

#### Example 2: Payout B Expires / Fails
- **Relation:** `Payment A` (10 units) $\rightarrow$ `Payout B` (4 units), `Payout C` (6 units).
- **Event:** `Payout B` expires (becomes the **Triggering Entity**).
- **Result:**
  - `Payout B` is the triggering entity: **No units** from `Payout B` are returned to the market.
  - `Payment A` and `Payout C` are cascaded entities:
    - If `Payment A`'s associated Bid is still **active**, its units (10 units) are returned to the market.
    - If `Payout C`'s associated Ask is still **active**, its units (6 units) are returned to the market.

---

## 5. Implementation Guidelines
Database transactions must be used for these status transitions and quantity updates to prevent race conditions and ensure data consistency. Use row locking (`SELECT FOR UPDATE`) on the Bid/Ask record during updating.

### 5.1 Reversion Formula (for Cascaded Entities only)
- `new_filled_qty = max(0, old_filled_qty - released_qty)`
- `new_remaining_qty = old_remaining_qty + released_qty`
