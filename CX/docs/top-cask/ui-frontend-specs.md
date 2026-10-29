# UI & FRONTEND SPECS

Top Cask — Homepage Curated Section

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation of UI/Frontend Specifications for Top Cask feature | Alice | 11/06/2026 |

---

## I. Screen Overview

The **Top Cask** feature displays a curated showcase of the highest-scoring master casks on the homepage. Scoring is computed as `60% Active Bids + 40% Casks Sold (30 Days)`. The section features a grid of distinct, highly visual vertical cards designed to drive immediate user interaction.

**Screen URL**: `/` (Home page section) and `/marketplace?sort=demandScore`
**User Role(s)**: All (authenticated and anonymous)
**Entry Point(s)**: Home page scroll section, or navigation link/button redirecting to this section.

---

## II. UI Elements Table

| # | Component | Type | Description | States / Variants |
|---|-----------|------|-------------|-------------------|
| C1 | Section Title | Text | "Top Casks" - section header | Default |
| C2 | View All Button | Button/Link | Routes to the Marketplace with the Demand Score sorting applied | Default, Hover |
| C3 | Card Grid Container | Grid | Responsive grid layout containing Top Cask cards | Loading (skeleton), Populated, Empty |
| C4 | Top Cask Card | Card | Container for a single master cask's information and CTAs | Default, Hover |
| C5 | Circular Cask Badge | Image | Crop of cask head image with dynamic overlay fallback for distillery and age | Loaded (custom image), Fallback (rendered dynamic cask head), Loading |
| C6 | Watchlist Icon | Button/Icon | Star icon in the top right to add/remove the cask from the user's watchlist | Default (empty star), Active (filled star), Hover |
| C7 | Cask Title | Text | Master cask name (e.g., "Glenburgie Sherry Butt") | Default |
| C8 | Floor Price Field | Text | Displays the lowest active ask price or fallback reference price | Floor Price (active asks), Est. Floor Price (fallback) |
| C9 | Price Delta Badge | Badge | 30-day floor price delta (e.g., "+1.5% (30D)") | Positive (green text), Negative (red text), Hidden (null delta) |
| C10 | Category Label | Metadata text | Master cask classification (e.g., "Single Malt") | Default |
| C11 | Cask Type Label | Metadata text | Master cask cask type (e.g., "Ex-Bourbon Hogshead") | Default |
| C12 | Region Label | Metadata text | Master cask region (e.g., "Highland") | Default |
| C13 | Vintages Range | Metadata text | Shows min and max vintage years of active variants (e.g., "1999 – 2004" or "2012") | Default |
| C14 | Buy Now Button | CTA Button | Triggers direct purchase of the lowest-priced active variant | Default (active), Disabled (greyed out if no active asks), Hover |
| C15 | Make Offer Button | CTA Button | Opens bidding interface or detail page to place an offer (bid) | Default, Hover |

### Component Details

#### C5 — Circular Cask Badge — Detailed Specs
- **Position**: Top-left of the card.
- **Aesthetic**: Rendered inside a circular frame with a double border.
- **Data Logic**:
  - If `masterCask.image` is set, crop it to a circle.
  - If `masterCask.image` is null, load a generic wooden cask head graphic. Overlay the **Distillery Name** (upper half arch) and **Youngest Variant Age / Vintage Year** (center, e.g., "8Y" or "2009") using high-contrast, serif typography to simulate a branded cask stamp.
- **Interaction**: Clicking the badge navigates to `/casks/{masterCaskId}`.

#### C6 — Watchlist Icon (Star) — Detailed Specs
- **Position**: Top-right of the card.
- **States**: 
  - *Default (Not Watched)*: Grey outline star.
  - *Active (Watched)*: Gold filled star.
- **Interaction**: Toggle state immediately (optimistic UI update). If user is anonymous, open sign-in modal.

#### C8 & C9 — Floor Price and Price Delta Badge — Detailed Specs
- **Label switching**: 
  - If `lowestAsk` is not null: Render label **"Floor Price"** and value `£{lowestAsk}` (formatted with thousands separator).
  - If `lowestAsk` is null: Render label **"Est. Floor Price"** and value `From £{minReferencePrice}`.
- **Delta Badge position**: Displayed inline next to the price value.
- **Delta Badge format**: `+{delta}% (30D)` or `-{delta}% (30D)`. Green for positive, red for negative. Hidden entirely if `lowestAskDelta30D` is null.

#### C13 — Vintages Range — Detailed Specs
- **Logic**: FE-rendered from `minVintageYear` and `maxVintageYear` (provided by backend in standard master cask payload).
  - If both are present and different: `minVintageYear – maxVintageYear` (e.g., "2000 – 2009").
  - If both are present and equal: Display single year (e.g., "2012").
  - If only one is present: Display single year.
  - If both are null: Display `—`.

#### C14 & C15 — Buy Now & Make Offer CTA Buttons — Detailed Specs
- **Buy Now Button**:
  - **Style**: Gold/amber background with dark bold text.
  - **Behavior**: If `lowestAsk` is null (no active asks), disable the button (grey background, muted text) and display a tooltip on hover: *"No active asks at this time. Place an offer instead."*
  - **Action**: Clicking navigates to `/casks/{masterCaskId}#buy` or directly launches checkout for the lowest ask variant.
- **Make Offer Button**:
  - **Style**: Transparent/white background with a thin dark border and dark text.
  - **Behavior**: Always active.
  - **Action**: Navigates to `/casks/{masterCaskId}#offer` or launches the bidding modal.

---

## III. Field Validations

This section is read-only. No input validation is performed directly on the card. 

---

## IV. Cases and Flows

### IV.1. Load Top Casks Grid

| Aspect | Detail |
|--------|--------|
| **Trigger** | On homepage load / scroll into viewport |

#### Loading State
- Render shimmer skeleton templates for cards (based on current viewport capacity: 4 for desktop, 3 for tablet, 2 for mobile).
- Buttons and star icons are disabled during loading.

#### Success State
- Cards render with correct parent metadata and aggregated variant details.
- Cards are ordered from left to right by their pre-computed `demandScore` descending.

#### Error States

| Error Condition | FE Behavior | User Message |
|-----------------|-------------|--------------|
| Request Timeout | Render retry placeholder in container | "Unable to load Top Casks. Tap to retry." |
| Server Error (500) | Hide container or show generic error state | "Something went wrong. Please try again later." |
| Anonymous User | Allow page load normally (displays public data) | — |

#### Empty State
- If the endpoint returns `[]` (no qualifying master casks), hide the entire "Top Casks" section from the homepage.

---

### IV.2. Toggle Watchlist (Star Button)

| Aspect | Detail |
|--------|--------|
| **Trigger** | User clicks the star icon |

#### Success State (Authenticated)
- Toggle star style immediately (optimistic UI update).
- Background API call to watchlist toggle. If it fails, revert UI state and show a toast error message: *"Failed to update watchlist. Please try again."*

#### Success State (Anonymous)
- Intercept click, show login/signup modal with return path back to homepage.

---

## V. Edge Cases & FE-only Logic

| # | Scenario | Expected FE Behavior |
|---|----------|---------------------|
| 1 | Cask name is exceptionally long | Truncate title (C7) at 2 lines max with ellipsis. |
| 2 | Master Cask variants have different vintage years | Render as a range: `minVintageYear – maxVintageYear` (e.g., "1999 – 2004"). |
| 3 | Master Cask has only 1 active vintage variant (or min and max are equal) | Render single vintage year (e.g., "2012"). |
| 4 | Master Cask has no variants with active asks | Floor Price value set to `minReferencePrice` prefixed with "From". Label changes to "Est. Floor Price". "Buy Now" button becomes disabled. |
| 5 | Delta value is zero | Display `0.0% (30D)` in muted grey text. |
| 6 | Dynamic overlay text on cask head is long | Auto-scale font size (min 10px) to prevent text overflow from circular crop. |

---

## VI. Navigation & Routing

| Action | Destination | URL Change | Notes |
|--------|-------------|-----------|-------|
| Click card title or badge | Master Cask Detail Page | `/casks/{masterCaskId}` | None |
| Click "View All" button | Marketplace | `/marketplace?sort=demandScore` | Passes sorting parameters |
| Click active "Buy Now" | Checkout / Detail Page Buy section | `/casks/{masterCaskId}#buy` | Navigates directly to buy option |
| Click "Make Offer" | Detail Page Bidding section | `/casks/{masterCaskId}#offer` | Navigates directly to bidding option |

---

## VII. Responsive Design Notes

| Breakpoint | Layout Changes | Card Count (`size` param) |
|-----------|----------------|----------------------------|
| Desktop (≥ 1024px) | 4-column card grid, full card structure visible. | `size=4` (fits clean 4x1 grid) |
| Tablet (768px – 1023px) | 3-column card grid, 4th card wraps or is omitted. | `size=3` (fits clean 3x1 grid) |
| Mobile (< 768px) | 2-column card grid, metadata text wraps. | `size=2` (fits clean 2x1 grid) |

---

## VIII. Accessibility Requirements

- [ ] All interactive elements (Star, CTA buttons) have unique `id` attributes: e.g., `top-cask-star-{masterCaskId}`, `top-cask-buy-{masterCaskId}`.
- [ ] Proper `aria-label` for watchlist star: `aria-label="Add {caskName} to watchlist"` or `aria-label="Remove {caskName} from watchlist"`.
- [ ] Keyboard navigation support: Tab index order covers Star → Card Link → Buy Now → Make Offer.
- [ ] Color contrast ratio >= 4.5:1 for price, metadata, and buttons.
- [ ] Cask head crop image has descriptive `alt` tag: `alt="{distilleryName} cask head crop badge"`.

---

_End of Document 1_
