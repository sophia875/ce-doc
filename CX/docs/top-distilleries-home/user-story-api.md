# User Story: Top Distilleries — API

### Context
The Homepage requires a "Top Distilleries" section. The backend must expose an authenticated endpoint that returns ranked distillery data, including Cold Start fallback metrics for distilleries with no completed trades.

### User Story
**As a** Frontend Client (authenticated),
**I want to** fetch a ranked list of the top 8 distilleries via a single API call,
**So that I can** render the Top Distilleries section on the homepage without additional round-trips.

### Reference Document
- [API Requirements](./api-requirements.md)

### Acceptance Criteria
1. **Endpoint:** `GET /api/top-distilleries` — restricted to authenticated roles (Buyer, Seller, Admin). Returns 401 if unauthenticated.
2. **Ranking:** Results are sorted descending by `lifetimeVolume`; tie-broken by `masterCaskCount` descending. Returns up to 8 distilleries.
3. **Cold Start:** Distilleries with `lifetimeVolume = 0` are included. Their response must contain `estMarketValue` and `estMedianPrice` (midpoint of `referencePriceMin` + `referencePriceMax` / 2).
4. **Delta Metrics:**
   - `volumeDelta30D` = `(volumeLast30D − volumePrev30D) / volumePrev30D × 100`. Returns `null` if `volumePrev30D = 0` or `null`.
   - `medianPriceDelta30D` = `(medianLast30D − medianPrev30D) / medianPrev30D × 100`. Returns `null` if `medianPrev30D = 0`, `null`, or `medianLast30D` is `null`.
5. **Median Calculation:** Median price must be unit-weighted (a transaction with `quantity = N` contributes N price entries).
6. **Performance:** Data is served from a pre-computed `DistilleryVolumeStats` table, refreshed every 15–30 minutes. Real-time joins are not acceptable.
