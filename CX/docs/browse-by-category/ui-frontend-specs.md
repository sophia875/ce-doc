# UI & FRONTEND SPECS

Browse by Category – Home Page Curated List

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | Alice | 08/06/2026 |
| 0.2 | Corrected display label field: card shows `categoryLabel` ("Single Malt"), not `classificationLabel` ("Single Malt Scotch"). Navigation uses `classification` slug. | Alice | 08/06/2026 |

---

## I. Screen Overview

The **Browse by Category** section appears on the Home page as a horizontally scrollable carousel of cask classifications (e.g., Single Malt, Irish Single Malt, Single Grain). Each card represents one classification and shows key market signals: the category label, a "Trending" badge when applicable, the median transaction price, and a 30-day price delta. This section helps buyers quickly scan classification-level market dynamics and navigate into filtered browse views per classification.

- **Screen URL**: `/` (Home page — embedded section)
- **User Role(s)**: Authenticated users (Buyer, Seller)
- **Entry Point(s)**:
  - Direct access to `/` (Home page)
  - Logo click from any page

---

## II. UI Elements Table

### 1. Section: Browse by Category

| # | Component | Type | Description | States / Variants |
|---|-----------|------|-------------|-------------------|
| 1 | Section Title | Text | Displays "Browse by Category" as a section heading | Static |
| 2 | Left Carousel Arrow | Button (Icon) | Scrolls the carousel to the left | Default, Hover, Disabled (when at start) |
| 3 | Right Carousel Arrow | Button (Icon) | Scrolls the carousel to the right | Default, Hover, Disabled (when at end) |
| 4 | Category Card | Card | Container for each classification's data. Clickable to navigate to filtered browse page | Default, Hover, Active |
| 5 | Category Image | Image | Decorative cask/whisky image representing the classification | Loaded, Fallback/Placeholder |
| 6 | Category Label | Text | Short, user-facing classification name displayed on the card (e.g., "Single Malt"). Sourced from `categoryLabel` on the Master Cask entity. | Static |
| 7 | Trending Badge | Badge | Labeled "Trending". Displayed conditionally based on the Trending metric formula | Visible, Hidden |
| 8 | Median Price Label | Text | Static label: "Med. Price" | Static |
| 9 | Median Price Value | Text | Formatted median transaction price (e.g., "£12,000"). Falls back to `estMedianPrice` if no transactions. | Has Value, Null/Empty (shows "—") |
| 10 | Price Delta Badge | Badge | Shows 30-day percentage change in median price (e.g., "+3.7% (30D)"). Colored green for positive, red for negative. | Positive (green), Negative (red), Null (hidden) |

---

### 2. Category Card — Detailed Specs

#### Layout Overview

Each card contains:

```
[ Category Image ]  [ Category Label ]  [ Trending Badge (conditional) ]
                    Med. Price
                    £12,000  +3.7% (30D)
```

#### Sub-Components Table

| Sub-Component | Source Field | Display Rule |
|---|---|---|
| Category Image | Platform-provided or classification default image | If no image available, show a generic cask placeholder |
| Category Label | `categoryLabel` | Always shown. Short display name (e.g., "Single Malt"). Not to be confused with `classificationLabel` ("Single Malt Scotch") which is the longer internal name. |
| Trending Badge | Derived from `isTrending` flag in API response | Shown only when `isTrending = true` |
| Median Price | `medianPrice` (real tx) or `estMedianPrice` (cold start) | Show `medianPrice` if not null; else show `estMedianPrice`; else "—" |
| Price Delta | `medianPriceDelta30D` | Show if not null; positive = green, negative = red, hidden if null |

#### Trending Badge Display Rules

The "Trending" badge is displayed on a category card when the API returns `isTrending = true` for that classification. The trending calculation is performed server-side. See the API Requirements document for the Trending metric formula.

#### Delta Badge Display Rules

| Condition | Display |
|---|---|
| `medianPriceDelta30D > 0` | Green badge: e.g., "+3.7% (30D)" |
| `medianPriceDelta30D < 0` | Red badge: e.g., "−2.1% (30D)" |
| `medianPriceDelta30D = 0` | Show "0.0% (30D)" in neutral color |
| `medianPriceDelta30D = null` | Badge hidden |

#### Image Fallback Rules

| Condition | Behavior |
|---|---|
| Classification has a designated image | Display classification image |
| No image provided | Display a generic cask placeholder image |
| Image URL broken / 404 | Fallback to placeholder on `onError` |

---

## III. Field Validations

This is a **read-only display screen**. No user-input fields are present. No field validations apply.

---

## IV. Cases and Flows

### IV.1. Load Browse by Classification on Home Page

| Aspect | Detail |
|--------|--------|
| **Objective** | Fetch and render the Browse by Classification section with classifications and market stats |
| **Actor** | Buyer, Seller |
| **Trigger** | User navigates to the Home page |
| **Pre-condition(s)** | User is authenticated |
| **Post-condition(s)** | Carousel is displayed with category cards |
| **API Call** | `GET /api/classifications/browse` _(new endpoint)_ |

#### Loading State

- Display skeleton cards (same width/height as real cards) in the carousel while the API call is in-flight.
- Carousel navigation arrows are disabled during loading.
- Section title "Browse by Classification" is visible.

#### Success State

| # | Action | Success State |
|---|---|---|
| 1 | Render Carousel | FE iterates over the response array. For each classification:<br>1. Render a **Category Card**.<br>2. Set image `src` to `classificationImageUrl` (or fallback).<br>3. Display `categoryLabel`.<br>4. Display Trending badge if `isTrending = true`.<br>5. Format and display `medianPrice` (or `estMedianPrice`).<br>6. Render Price Delta Badge. | UI renders cards in order |

- Cards are ordered as returned by the API (server determines sort order — by `lifetimeVolume DESC`, cold-start fill-in by `estMarketValue DESC`).
- Carousel arrows are enabled if content overflows the visible area.
- If all cards fit without scrolling, arrows are hidden or remain in disabled state (design decision).

**Data mapping:**

| API Field | UI Component |
|---|---|
| `categoryLabel` | Category Label (card display name) |
| `isTrending` | Trending Badge visibility |
| `medianPrice` (non-null) | Median Price Value |
| `estMedianPrice` (fallback) | Median Price Value (when `medianPrice` is null) |
| `medianPriceDelta30D` | Price Delta Badge |
| Category image URL | Category Image |

#### Error States

| Error Condition | FE Behavior | User Message |
|---|---|---|
| 401 Unauthorized / Session Expired | Redirect to login page | "Session expired. Please log in again." |
| 500 Server Error | Hide section or show inline error message | "Unable to load classifications. Please refresh the page." |
| Network timeout | Show retry button within section | "Something went wrong. Please try again." |

#### Empty State

- If the API returns an empty array `[]`, hide the entire section from the home page.
- No "empty state" illustration is needed inside the carousel — just omit the section.

---

### IV.2. User Interaction — Click Category Card

| Aspect | Detail |
|--------|--------|
| **Trigger** | User clicks on any Category Card |

#### Behavior

- Navigate to the Cask Browse/Listing page pre-filtered by the selected classification.
- **URL**: `/casks?classification={classificationSlug}` (or equivalent browse URL pattern)
- The classification filter is passed as a query parameter using the `classification` slug (e.g., `single_malt`).
- No loading indicator needed on the card itself; browser page navigation handles it.

---

### IV.3. User Interaction — Carousel Navigation

| Aspect | Detail |
|--------|--------|
| **Trigger** | User clicks Left or Right arrow buttons |

#### Behavior

- Scrolls the carousel horizontally by approximately one card width (or a defined scroll step).
- Left arrow is disabled/hidden when already at the leftmost position.
- Right arrow is disabled/hidden when already at the rightmost position.
- Smooth scroll animation applied.
- Keyboard navigation (Arrow keys) supported when carousel is focused.

---

## V. Edge Cases & FE-only Logic

| # | Scenario | Expected FE Behavior |
|---|---|---|
| 1 | `medianPrice` is null (cold-start classification) | Display `estMedianPrice` value instead; if also null, display "—" |
| 2 | `medianPriceDelta30D` is null | Hide the price delta badge entirely; do not show "—" in its place |
| 3 | `isTrending = false` for all classifications | No Trending badges shown; render cards normally |
| 4 | Only 1–2 classifications returned | Carousel arrows hidden; cards render left-aligned inline |
| 5 | `classificationImageUrl` is `null` | Render card; use fallback generic cask image |
| 6 | Category image fails to load | Fallback to generic cask placeholder via `onError` handler |
| 7 | Long `classificationLabel` text | Truncate with ellipsis ("…") to fit card width; show full label on hover tooltip |
| 8 | User resizes window | Carousel re-evaluates overflow and shows/hides arrows accordingly |
| 9 | Slow network / API timeout | Skeleton remains until timeout threshold; then show error state with retry CTA |
| 10 | User navigates back from Browse page | Home page reload; classifications reload fresh from API |

---

## VI. Navigation & Routing

| Action | Destination | URL Change | Notes |
|---|---|---|---|
| Click Category Card | Cask Browse / Listing page | `/casks?classification={slug}` | `slug` is the `classification` value (e.g., `single_malt`) |
| Click Home Logo | Home page | `/` | Section reloads |

---

## VII. Responsive Design Notes

| Breakpoint | Layout Changes |
|---|---|
| Desktop (≥ 1024px) | Show ~4 cards visible; horizontal carousel with arrow navigation |
| Tablet (768px – 1023px) | Show ~2–3 cards; carousel with touch swipe support; arrows still visible |
| Mobile (< 768px) | Show ~1–2 cards; horizontal scroll (touch-drag); arrows may be hidden in favor of swipe |

> Touch swipe gesture should be supported on all breakpoints below desktop.

---

## VIII. Accessibility Requirements

- [ ] All interactive elements (carousel arrows, category cards) have unique `id` attributes
- [ ] Carousel arrows have `aria-label` (e.g., `aria-label="Scroll left"`, `aria-label="Scroll right"`)
- [ ] Category cards are keyboard-navigable (Tab + Enter to activate)
- [ ] Trending Badge has `aria-label="Trending"` for screen readers
- [ ] Delta badge color alone does not convey meaning — include `+` or `−` prefix in text
- [ ] Image has descriptive `alt` text (e.g., `alt="Single Malt category"`)
- [ ] Color contrast ratio ≥ 4.5:1 for all text elements
- [ ] Focus indicators visible on card and arrow elements
- [ ] Skeleton loaders use `aria-busy="true"` on the container during loading

---

_End of Document 1_
