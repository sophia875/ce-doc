# UI & FRONTEND SPECS

Explore Casks — Home Page Curated List (Product Tabs)

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | Alice | 05/05/2026 |
| 0.2 | Floor Price / Est. Price label switching logic added; `publishedAt` resolved | Alice | 05/05/2026 |
| 0.3 | Est. Price changed to Est. Floor Price; publishedAt logic updated to latestListingDate | Alice | 05/05/2026 |
| 0.4 | Changed card Volume field from volume30D to lifetimeVolume | Alice | 05/05/2026 |
| 0.5 | Changed Delta Badge to use floor price delta instead of median price delta | Alice | 05/05/2026 |
| 0.6 | Fixed "View All" sort param for Top Traded tab: `?sort=volume_desc` → `?sort=lifetime_volume_desc` to align with API requirements resolved decision #4; centralized `Vintages Range` FE rendering logic in Component Details | Alice | 14/05/2026 |
| 0.7 | Switched to unified marketplace endpoint with viewport-driven limit parameter: Desktop limit=9 (3-columns x 3-rows), Tablet limit=8 (2-columns x 4-rows), Mobile limit=6 (1-column x 6-rows). Updated loading/error states accordingly. | Alice | 19/05/2026 |

---

## I. Screen Overview

> The **Explore Casks** section on the Home page showcases curated master casks organized across 4 product tabs: Top Traded, Most Watched, Strategic Buy, and New Listing. Each tab presents a scrollable grid of cask cards. A "View All" link routes the user to the /marketplace page with the corresponding sort applied.

**Screen URL**: `/` (Home page section)
**User Role(s)**: All (authenticated and anonymous)
**Entry Point(s)**: Home page, direct scroll to section

---

## II. UI Elements Table

| # | Component | Type | Description | States / Variants |
|---|-----------|------|-------------|-------------------|
| C1 | Section Title | Text | "Explore Casks" — section heading | — |
| C2 | View All Button | Button/Link | Routes to `/marketplace` with active tab's sort applied | Default, Hover |
| C3 | Tab Bar | Tab Group | 4 tabs: "Top Traded", "Most Watched", "Strategic Buy", "New Listing" | Active, Inactive, Hover |
| C4 | Cask Card Grid | Grid | Displays cask cards in a responsive grid layout | Loading (skeleton), Populated, Empty |
| C5 | Cask Card | Card | Container for a single master cask's info | Default, Hover |
| C6 | Cask Image | Image | Master cask product image | Loaded, Placeholder (no image) |
| C7 | Watchlist Icon | Icon/Button | Heart icon — add/remove master cask from watchlist | Default (empty heart), Active (filled heart), Hover |
| C8 | Cask Name | Text | Master cask display name, e.g. "Aberlour Ex-Sherry Hogshead" | — |
| C9 | Category Label | Metadata text | Classification label, e.g. "Single Malt" | — |
| C10 | Region Label | Metadata text | Region name, e.g. "Speyside" | — |
| C11 | Vintages Range | Metadata text | Year range of active variants. FE-rendered from `minVintageYear` and `maxVintageYear`. | Single year, Range, N/A |
| C12 | Price Field | Price text | Shows the effective floor price with a context-sensitive label. **"Floor Price"** when active ask exists; **"Est. Floor Price"** when no active ask (reference price only) | Floor Price (has ask), Est. Floor Price (ref only) |
| C13 | Volume | Metric text | Lifetime trading volume in abbreviated format (e.g., £120M) | — |
| C14 | Listed Count | Metric text | Number of active ask listings (e.g., "14") | "–" if no active asks |
| C15 | Delta Badge | Badge | 30D floor price delta (e.g., "▲ +3.2% (30D)") | Positive (green), Negative (red), Hidden (null) |
| C16 | Buy Now Button | CTA Button | Shown when master cask has ≥1 active ask. Shows floor price | Default, Hover |
| C17 | Make Offer Button | CTA Button | Shown when master cask has no active asks | Default, Hover |

---

## III. Component Details

### C3 — Tab Bar

- **Default active tab**: "Top Traded"
- **Behavior**: Clicking a tab reloads the card grid with the corresponding sorted data via API call
- **"View All" updates**: When tab changes, View All link updates its query params accordingly

### C5 — Cask Card

- **Layout (top → bottom)**: Image → Cask Name → [Category | Region | Vintages] → [Floor Price | Volume | Listed] → Delta Badge → CTA Button
- **Navigation**: Clicking the card (anywhere except the watchlist icon and CTA button) navigates to `/casks/{masterCaskId}`
- **Max name display**: 2 lines max, truncate with ellipsis if longer

### C6 — Cask Image
- Source: master cask's primary image. Fallback: first variant's `imageUrl` (by `order` field)
- If no image: render a styled placeholder with a cask icon

### C11 — Vintages Range
- **Source**: FE-rendered from `minVintageYear` and `maxVintageYear` provided by API
- **Logic**:
  - If both are present and different: `minVintageYear – maxVintageYear` (e.g., "1999 – 2004")
  - If both are present and equal: Display single year (e.g., "2012")
  - If only one is present: Display single year
  - If both are null: Display `—`
- **Style**: Metadata text style (muted/secondary)

### C12 — Price Field (Floor Price / Est. Floor Price)
- **Value priority**: `lowestAsk` (if not null) → `referencePriceMin` (fallback)
- **Label logic** (applies across **all 4 tabs**):
  - `lowestAsk IS NOT NULL` → Label: **"Floor Price"** — a real, tradeable market ask price
  - `lowestAsk IS NULL` → Label: **"Est. Floor Price"** — the admin-set reference minimum; not directly purchasable
- **Rationale**: Displaying "Floor Price" when only a reference price is available creates cognitive dissonance — the buyer sees a price but the CTA is "Make Offer" (not "Buy Now"). The label switch signals this is an estimate, not a live ask.
- **Format**: `£{price}` formatted with locale thousands separator
- **Tooltip** *(optional, low priority)*: On "Est. Floor Price" hover/tap: "Reference price set by the platform. No active asks at this time."

### C13 — Volume
- Displays `lifetimeVolume` (all-time transacted volume)
- **Abbreviation**: `£120M`, `£1.2K`, `£3.4B` etc.
- If null or 0: display `—`

### C14 — Listed Count
- Displays `totalActiveAsks` (number of active ask orders across all variants)
- If 0 or no active asks: display `–`

### C15 — Delta Badge
- **Formula**: 30D floor price delta percent from API
- **Format**: `▲ +X.X% (30D)` or `▼ -X.X% (30D)`
- **Visibility**: Hidden entirely if delta value is null (do not render anything)
- **Colors**: Green for positive, Red for negative

### C16 — Buy Now Button
- Shown when `lowestAsk IS NOT NULL`
- Label: **Buy Now** with floor price inline: e.g., `Buy Now  £15,200`
- Style: Gold/amber background

### C17 — Make Offer Button
- Shown when `lowestAsk IS NULL` (no active asks)
- Label: **Make Offer**
- Style: Dark/black background

---

## IV. Cases and Flows

### IV.1. Initial Load — Default Tab (Top Traded)

| Aspect | Detail |
|--------|--------|
| **Trigger** | Home page loads |

#### Loading State
- Show skeleton cards (shimmer) in the grid matching the viewport-driven limit (9 placeholders on desktop, 8 on tablet, 6 on mobile)
- Tabs are rendered but not interactive until data loads

#### Success State
- Display master cask cards returned by the API (matching the viewport-driven `limit` request parameter)
- Each card renders with full data: image, name, metadata row, price, volume, listed count, delta badge, CTA button

#### Error States

| Error Condition | FE Behavior | User Message |
|-----------------|-------------|--------------|
| Network timeout | Show retry button within section | "Failed to load. Tap to retry." |
| 401 Unauthorized | Treat as anonymous (show public data) | — |
| 500 Server Error | Show error state in section | "Something went wrong. Please try again." |

#### Empty State
- If API returns empty array `[]`: hide the entire section (do not render blank grid)

---

### IV.2. Tab Switch

| Aspect | Detail |
|--------|--------|
| **Trigger** | User clicks a different tab |

#### Loading State
- Show skeleton cards within current tab content area

#### Success State
- Replace cards with data from the new tab's API response
- "View All" button URL updates to reflect new tab's sort params (see Section VI)

#### Error States

| Error Condition | FE Behavior | User Message |
|-----------------|-------------|--------------|
| API error on tab switch | Keep previous tab content; show toast error | "Failed to load tab. Please try again." |

---

### IV.3. Watchlist Toggle

| Aspect | Detail |
|--------|--------|
| **Trigger** | User clicks the heart icon on a card |

#### For Anonymous User
- Redirect to login page or show login modal

#### For Authenticated User
- Optimistic UI update: toggle heart immediately
- Call watchlist API in background
- On failure: revert icon state, show toast error

---

## V. Edge Cases & FE-only Logic

| # | Scenario | Expected FE Behavior |
|---|----------|---------------------|
| 1 | `lowestAsk` is null | Display `referencePriceMin` as price value. Label: **"Est. Floor Price"**. Show **"Make Offer"** button |
| 2 | `lowestAsk` is not null | Display `lowestAsk` as price value. Label: **"Floor Price"**. Show **"Buy Now £{price}"** button |
| 3 | `priceDelta30D` is null | Hide delta badge entirely — do not render anything |
| 4 | `lifetimeVolume` is null or 0 | Display `—` in Volume field |
| 5 | `totalActiveAsks` = 0 | Display `–` in Listed field. Show **"Make Offer"** button |
| 6 | Master cask has no image | Render image placeholder with cask icon |
| 7 | Cask name > ~35 chars | Truncate with ellipsis at 2 lines max |
| 8 | Only 1 active vintage year | Display single year instead of range, e.g. "2012" |
| 9 | No active variants with a vintage year | Display `—` in Vintages field |
| 10 | API returns fewer items than requested limit | Render only available cards; no empty placeholder cards |
| 11 | Delta value is very large (> 999%) | Display raw value without truncation, e.g., ▲ +1,200% (30D) |
| 12 | Master cask has no variants with a `listingDate` | Excluded from New Listing tab results at API level; no special FE handling needed |
| 13 | User navigates to `/casks/{id}` with invalid ID | Return 404 page |
| 14 | User rapidly switches tabs | Cancel previous in-flight requests using `AbortController`; only render data from the latest selected tab to prevent race conditions |
| 15 | Master cask loses all active variants mid-session | Current UI render remains visible; master cask will be excluded automatically on next tab switch or page refresh |

---

## VI. Navigation & Routing

| Action | Destination | URL / Params | Notes |
|--------|-------------|-------------|-------|
| Click "View All" on "Top Traded" tab | `/marketplace` | `?sort=lifetime_volume_desc` | Sorts marketplace by lifetime volume |
| Click "View All" on "Most Watched" tab | `/marketplace` | `?sort=view_count_desc` | Sorts by viewCount |
| Click "View All" on "Strategic Buy" tab | `/marketplace` | `?sort=floor_price_asc` | Sorts by floor price ascending |
| Click "View All" on "New Listing" tab | `/marketplace` | `?sort=listing_date_desc` | Sorts by newest listing |
| Click cask card body | `/casks/{masterCaskId}` | — | Navigate to detail page |
| Click "Buy Now" button | `/casks/{masterCaskId}#buy` | — | Navigate to detail, scroll to buy section |
| Click "Make Offer" button | `/casks/{masterCaskId}#offer` | — | Navigate to detail, scroll to offer section |
| Click heart icon (anonymous) | Login page / Login modal | — | |
| Click heart icon (authenticated) | No navigation | — | Toggles watchlist in-place |

---

## VII. Responsive Design Notes

| Breakpoint | Layout Changes | Cask Limit (`limit` param) |
|-----------|----------------|----------------------------|
| Desktop (≥ 1024px) | 3-column card grid. Full metadata row visible on each card | `limit=9` (fits clean 3x3 grid) |
| Tablet (768px – 1023px) | 2-column card grid. Metadata row may wrap | `limit=8` (fits clean 2x4 grid) |
| Mobile (< 768px) | 1-column card grid. Full card layout preserved vertically | `limit=6` (fits clean 1x6 list) |

---

## VIII. Accessibility Requirements

| # | Requirement | Importance |
|---|-------------|-----------|
| 1 | Each cask card has a unique `id` attribute: `cask-card-{masterCaskId}` | Low |
| 2 | Tab bar uses `role="tablist"` with `role="tab"` for each tab item | High |
| 3 | Active tab has `aria-selected="true"` | High |
| 4 | Cask image has `alt="{caskName} product image"` (or `alt="Cask image placeholder"` if missing) | High |
| 5 | Delta badge uses `aria-label` to convey direction: e.g., `aria-label="Up 3.2% over 30 days"` | Low |
| 6 | Buy Now / Make Offer buttons are keyboard-focusable and activatable via Enter | High |
| 7 | Heart / Watchlist icon has `aria-label="Add to watchlist"` or `aria-label="Remove from watchlist"` | Medium |
| 8 | Color contrast ratio ≥ 4.5:1 for all text elements | High |
| 9 | Cards are keyboard-navigable (Tab to focus, Enter to navigate to detail page) | High |

---

_End of Document 1_
