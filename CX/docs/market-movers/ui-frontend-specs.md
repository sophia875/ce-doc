# UI & FRONTEND SPECS

Market Movers — Home Page Section

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | Antigravity | 08/06/2026 |

---

## I. Screen Overview

The **Market Movers** section appears on the Home page as a ranked list of master casks, sorted by their 30-day volume momentum (`volumeDelta30D`). It highlights casks that are experiencing significant surges in trading activity. Each row displays the cask's rank, name, 30-day volume with its delta, median price with its delta, and the live floor price.

- **Screen URL**: `/` (Home page — embedded section)
- **User Role(s)**: Authenticated users (Buyer, Seller)
- **Entry Point(s)**:
  - Direct access to `/` (Home page)
  - Logo click from any page

---

## II. UI Elements Table

### 1. Section: Market Movers

| # | Component | Type | Description | States / Variants |
|---|-----------|------|-------------|-------------------|
| 1 | Section Title | Text | Displays "Market Movers" as a section heading | Static |
| 2 | Market Movers Table | Table | Container for the ranked list of casks | Loading (Skeleton), Populated, Empty |
| 3 | Rank Column Header | Text | Displays "Rank" | Static |
| 4 | Cask Column Header | Text | Displays "Cask" | Static |
| 5 | Volume Column Header | Text | Displays "Volume" | Static |
| 6 | Med. Price Column Header | Text | Displays "Med. Price" | Static |
| 7 | Live Floor Column Header | Text | Displays "Live Floor" | Static |
| 8 | Rank Value | Badge | Numeric rank (1, 2, 3...). Top 3 may have distinct visual styling (e.g., gold, silver, bronze medals). | 1, 2, 3, 4+ |
| 9 | Cask Name | Text | Name of the master cask (e.g., "Aberlour Ex-Sherry Hogshead"). Clickable to navigate to cask detail page. | Default, Hover |
| 10 | Volume Value | Text | Formatted 30-day volume (e.g., "£490,000") | Has Value, "—" |
| 11 | Volume Delta Badge | Badge | 30-day percentage change in volume (e.g., "+3.2% (30D)"). Colored green for positive, red for negative. | Positive (green), Negative (red), Hidden if null |
| 12 | Median Price Value | Text | Formatted median price (e.g., "£32,000") | Has Value, "—" |
| 13 | Median Price Delta Badge | Badge | 30-day percentage change in median price | Positive (green), Negative (red), Hidden if null |
| 14 | Live Floor Value | Text | Formatted live floor price (`lowestAsk`), e.g., "£33,000" | Has Value, "—" |

---

## III. Field Validations

This is a **read-only display screen**. No user-input fields are present. No field validations apply.

---

## IV. Cases and Flows

### IV.1. Load Market Movers on Home Page

| Aspect | Detail |
|--------|--------|
| **Objective** | Fetch and render the Market Movers section with ranked master casks and market stats |
| **Actor** | Buyer, Seller |
| **Trigger** | User navigates to the Home page |
| **Pre-condition(s)** | User is authenticated |
| **Post-condition(s)** | Table is displayed with ranked casks |
| **API Call** | `GET /api/cask-masters` with `sortBy=volumeDelta30D` and `sortOrder=desc` |

#### Loading State

- Display a skeleton loader matching the table row structure.
- Section title "Market Movers" is visible.

#### Success State

| # | Action | Success State |
|---|---|---|
| 1 | Render Table | FE filters out any master casks where `volumeDelta30D` is null (representing cold start / no trading activity). For each remaining master cask:<br>1. Render Rank based on its position in the filtered list (1-indexed).<br>2. Display `name`.<br>3. Format and display `volumeLast30D`.<br>4. Render Volume Delta Badge (`volumeDelta30D`).<br>5. Format and display `medianPrice`.<br>6. Render Median Price Delta Badge (`medianPriceDelta30D`).<br>7. Format and display `lowestAsk` as Live Floor. | UI renders rows in order |

#### Error States

| Error Condition | FE Behavior | User Message |
|---|---|---|
| 401 Unauthorized / Session Expired | Redirect to login page | "Session expired. Please log in again." |
| 500 Server Error | Hide section or show inline error message | "Unable to load market movers. Please refresh the page." |
| Network timeout | Show retry button within section | "Something went wrong. Please try again." |

#### Empty State

- If no master casks are returned (e.g. `[]`), hide the entire Market Movers section from the Home page.

---

## V. Edge Cases & FE-only Logic

| # | Scenario | Expected FE Behavior |
|---|---|---|
| 1 | `lowestAsk` is null | Display "—" for the Live Floor value |
| 2 | `volumeDelta30D` is null (Cold Start) | Hide the entire row (do not render the cask). Ranks of other visible rows are recalculated sequentially. |
| 3 | `medianPriceDelta30D` is null | Hide the median price delta badge entirely |
| 4 | Long Cask Name | Truncate with ellipsis ("…") to fit column width; show full name on hover tooltip |
| 5 | Network Timeout | Skeleton remains until timeout threshold; then show error state with retry CTA |

---

## VI. Navigation & Routing

| Action | Destination |
|---|---|
| Click on Cask Row / Name | Navigate to Master Cask Detail page (`/casks/[masterCaskId]`) |

---

## VII. Responsive Design Notes

- **Desktop/Tablet**: Display as a standard data table.
- **Mobile**: Transform rows into stacked cards to ensure readability of all metrics without horizontal scrolling.

---

## VIII. Accessibility Requirements

- **Semantic HTML**: Use proper `<table>`, `<th>`, `<tr>`, and `<td>` elements for the desktop view.
- **Aria Labels**: Delta badges must have `aria-label` clearly stating "Increase of X percent" or "Decrease of X percent" since color (green/red) alone is not sufficient for screen readers.
- **Contrast**: Ensure text contrast ratios meet WCAG AA standards, especially for colored text (red/green deltas).

---

_End of Document 1_
