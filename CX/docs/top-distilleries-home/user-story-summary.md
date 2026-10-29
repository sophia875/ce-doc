# User Story Summary: Top Distilleries Home Section

---

## 1. Backend / API Ticket

### Context
To support brand discovery on the Cask Exchange homepage, users need to see the top-performing distilleries based on their real trading history. The platform must accurately calculate lifetime volume, unit-weighted median prices, and 30-day market trends, while also ensuring that newly listed distilleries (Cold Start) appear with estimated valuations.

### User Story
**As a** Cask Exchange User (Buyer/Seller),
**I want** the platform to accurately rank top distilleries and provide real-time market metrics,
**So that I can** discover popular brands, track their 30-day performance trends, and evaluate new distilleries based on estimated values.

### Reference Document
- [API Requirements](./api-requirements.md)

### Acceptance Criteria
1. **Endpoint & Auth:** Provide a read-only endpoint returning top distilleries, restricted to authenticated users (`Buyer`, `Seller`, `Admin`).
2. **Ranking:** Return up to 8 distilleries ordered by `lifetimeVolume` (DESC), with `masterCaskCount` (DESC) as the tie-breaker.
3. **Data Model & Metric Calculations:** Accurately compute and return core market metrics:
   - `lifetimeVolume`: Total value of all completed transactions.
   - `medianPrice`: Unit-weighted median price of completed transactions.
   - **30D Deltas:** Percentage change for volume and median price over the last 30 days compared to the prior 30-day period (handling zero/null historical data gracefully).
4. **Cold Start Support:** Include distilleries with `0` lifetime volume. Pre-calculate and return `estMarketValue` and `estMedianPrice` (using active variant midpoints) as fallbacks.
5. **Performance Strategy:** Aggregated metrics must be queried from a pre-computed stats table (refreshing every 15-30m) rather than executing heavy SQL joins in real-time.

---

## 2. Frontend / UI Ticket

### Context
Users visiting the Cask Exchange homepage need a visually engaging "Top Distilleries" section. This UI highlights the most popular distilleries, providing users with quick insights into market trends, median cask prices, and gracefully handling the display of estimated values for new distilleries without trading history.

### User Story
**As a** Cask Exchange User (Buyer/Seller),
**I want to** view a curated list of Top Distilleries on the homepage,
**So that I can** easily browse high-performing brands, quickly view their 30-day market trends, and make informed investment decisions.

### Reference Document
- [UI / Frontend Specs](./ui-frontend-specs.md)

### Acceptance Criteria
1. **Display Cards:** Render up to 8 distillery cards showing Rank (1-8), Image, Name, Cask Count, Median Price, and Volume.
2. **Cold Start UI (Fallback Labels):** 
   - If `lifetimeVolume` is `0` or null: Switch "Volume" label to "Est. Value" and render `estMarketValue` in muted grey italic text.
   - If `medianPrice` is null: Switch "Med. Price" label to "Est. Price" and render `estMedianPrice` in muted grey italic text.
3. **Delta Badges Logic:** 
   - **Positive:** Green badge `▲ +X.X% (30D)`.
   - **Negative:** Red badge `▼ -X.X% (30D)`.
   - **Zero:** Muted grey badge `— 0.0% (30D)`.
   - **Null:** Hide the delta badge entirely (do not render anything).
4. **Interactions:** Clicking a card navigates to `/distillery/{distilleryId}`. Clicking "View all" navigates to `/distilleries`.
5. **Loading & Error:** Show 8 skeleton cards during load. Silently hide the section if the API fails (5xx) or times out (>5s).
