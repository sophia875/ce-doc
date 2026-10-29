# Schema Change Requests

This document tracks requested additions or modifications to the backend data model. Each entry includes the rationale, the exact field spec, and which feature(s) depend on it.

---

## SCR-001 — Add `completedAt` to `CheckoutSession`

| Item | Detail |
|------|--------|
| **Status** | 🟡 Pending BE confirmation |
| **Requested by** | Alice |
| **Date** | 07/05/2026 |
| **Required for** | Top Distilleries (home page), and potentially any future feature computing 30-day volume windows |

### Problem

`CheckoutSession` currently has no reliable single timestamp that marks the moment a session transitions to `status = 'completed'`. The closest existing fields are:

| Field | Why it's unsuitable |
|-------|---------------------|
| `updatedAt` | Updated on **any** status change — refunds, admin edits, retries — not just completion. Would produce incorrect 30D window boundaries. |
| `ownershipTransferredAt` | Represents a specific step (ownership document transfer), which can be `null` even on sessions with `status = 'completed'`. Not guaranteed to be set at completion time. |

### Requested Change

Add a new field `completedAt` to the `CheckoutSession` entity:

```
checkoutSession.completedAt: datetime | null
```

| Property | Spec |
|----------|------|
| **Type** | `datetime` (ISO 8601) |
| **Nullable** | Yes — `null` for all sessions not yet in `completed` status |
| **Set when** | Whenever `checkoutSession.status` transitions **to** `'completed'` from any other status. If a session is reverted (e.g., moved from `completed` back to `agreement_signed` for a fix) and then re-completed, this timestamp must be updated to the latest completion time. |
| **Mutability** | **Mutable only** during a transition to `'completed'`. Must NOT be overwritten or cleared by other status changes (e.g., moving to `refunded`) or subsequent admin metadata updates. |
| **Index** | Recommended: `INDEX(completedAt)` to support efficient 30-day range queries in the `DistilleryVolumeStats` pre-computation job. |

### Usage

Once available, this field will be used as the 30D window boundary in all volume and median calculations:

```sql
-- 30D Volume Window
volumeLast30D = SUM(cs.originalAmount)
  WHERE cs.completedAt >= (NOW - 30 days)
  AND cs.status = 'completed'

-- 30D Median Price Window (Unit-Weighted)
medianLast30D = MEDIAN(tx.transactionPrice)
  JOIN checkout_sessions cs ON tx.sessionId = cs.id
  WHERE cs.completedAt >= (NOW - 30 days)
  AND cs.status = 'completed'
```

This replaces the interim workaround of using `cs.ownershipTransferredAt`.

### Impact

| Scope | Change |
|-------|--------|
| DB Schema | Add `completed_at` column to `checkout_sessions` table |
| Backfill | Existing completed sessions require `completedAt` backfilling. **Primary Source:** Timestamp from the `status = 'completed'` audit log entry. **Fallback:** If logs are unavailable, use `updatedAt` but flag as "High Risk/Best Effort" as it may capture post-completion edits (admin notes, etc.) rather than the true completion date. |
| **Side Effect** | Setting or updating `completedAt` MUST trigger an asynchronous refresh of the `DistilleryVolumeStats` record for the affected distillery to ensure ranking deltas are reflected immediately. |
| Application Code | Set `completedAt = NOW()` whenever `status` is set to `'completed'` |
| Docs to update | `docs/top-distilleries-home/api-requirements.md` (already references `cs.completedAt` pending this change) |

---

## SCR-002 — Add `status` to `MasterCask`

| Item | Detail |
|------|--------|
| **Status** | 🟢 Approved |
| **Requested by** | Alice |
| **Date** | 07/05/2026 |
| **Required for** | Top Distilleries (ranking logic), and Admin "Unpublish" feature |

### Problem

`MasterCask` (Parent) currently lacks an explicit status field. Availability is only managed at the `VariantCask` (Child) level. This makes it difficult to:
1.  Perform efficient ranking queries (requires complex joins to check for active children).
2.  Unpublish an entire product line (requires updating every single variant).
3.  **Correct Cask Count tracking:** The current logic is prone to counting individual variants rather than unique Master Casks (parents), leading to inflated or inaccurate brand metrics.

### Requested Change

Add a new field `status` to the `MasterCask` entity:

```
masterCask.status: string ('active' | 'inactive')
```

| Property | Spec |
|----------|------|
| **Type** | `string` |
| **Values** | `'active'`, `'inactive'` |
| **Default** | `'active'` |
| **Logic** | **Master Status overrides Variant Status.** If `masterCask.status = 'inactive'`, all its variant children must be hidden from Browse/Search lists and excluded from **Active counts** (`masterCaskCount`) and **Estimation metrics** (`estMarketValue`, `estMedianPrice`). **Historical data** (`lifetimeVolume`, `medianPrice`, and delta computations) must **NOT** be excluded; they continue to use all completed transactions to maintain the integrity of the distillery's brand success signals. |

### Usage

This field will simplify the `masterCaskCount` calculation:

```sql
-- New simplified count: ignores historical-only (inactive) products
masterCaskCount = COUNT(id) WHERE distillery_id = ? AND status = 'active'
```

### Impact

| Scope | Change |
|-------|--------|
| DB Schema | Add `status` column to `master_casks` table |
| Application Code | Update visibility queries (Explore/Search) to check `master.status` AND `child.caskStatus` |
| Ranking Logic | Ensure `lifetimeVolume` continues to SUM all completed trades, even if the parent `MasterCask` is now `inactive` |
| Admin Panel | Add "Unpublish/Deactivate" button to Master Cask edit page |
| **Side Effect** | Any change to `masterCask.status` MUST trigger an asynchronous refresh of the `DistilleryVolumeStats` record for the affected distillery. This prevents "rank flips" caused by stale tie-breaker data (masterCaskCount). |
| **Backfill** | Existing Master Casks should be initialized to `'active'`. Recommendation: Set to `'inactive'` if a record has zero child variants with `caskStatus = 'active'`. |

---
