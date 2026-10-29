# UI & FRONTEND SPECS

Marketplace

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation — Added details for filters, sorting, toggles, and card actions. Removed "Top Traded" sort option per user request. | Antigravity | 2026-06-16 |
| 0.2 | Refined terminology: Replaced all references of "variant" with "vintage" per user request. | Antigravity | 2026-06-16 |

---

## I. Screen Overview

The Marketplace page serves as the primary search, discovery, and trading hub for the Cask Exchange platform. It allows users to browse casks, apply multiple overlapping filters (at both the master cask and cask vintage level), search by cask or distillery name, toggle filters for personal ownership and "Buy Now" availability, and sort listings by pricing, volume, availability, views, or name.

*   **Screen URL**: `/marketplace`
    *   Query parameters are updated in real-time to reflect the current state of search, filters, pagination, and sorting (e.g., `/marketplace?search=Aberlour&caskType=Sherry%20Butt&ownedByMe=false&buyNowOnly=true&sortBy=lowestAsk&sortOrder=DESC&page=1`).
*   **User Role(s)**: All users (authenticated and anonymous).
    *   *Note*: The "Owned by you" toggle requires authentication. If clicked by an anonymous user, they are redirected to login.
*   **Entry Point(s)**:
    *   Global Navigation Bar → "Marketplace"
    *   "View All" links on Home page curated list carousels (e.g. Volume, Most Watched, Strategic Buys, New Listing)
    *   Direct URL access (with or without query parameters)

---

## II. UI Elements Table

### Screen Layout Components

| # | Component | Type | Description | States / Vintages |
|---|-----------|------|-------------|-------------------|
| 1 | Filter Sidebar | Panel / Container | Holds all attribute range and checkbox filters. Fixed on the left on desktop; collapses into a slide-in drawer on mobile. | Default, Scrollable, Collapsed (mobile) |
| 2 | Top Header Bar | Header Group | Contains the search input, toggle switches, and the sorting dropdown. | Default |
| 3 | Search Input | Text Input | Search bar to filter casks by name, reference number, or distillery name. | Default, Filled, Focus, Loading, Empty |
| 4 | "Owned by you" Toggle | Toggle Switch | Filters list to only casks where the authenticated user owns at least one vintage. | Inactive (default), Active, Hover, Disabled (for anonymous users) |
| 5 | "Buy now available" Toggle | Toggle Switch | Filters list to only casks that have an active sell offer (lowest ask is not null on at least one vintage). | Inactive (default), Active, Hover |
| 6 | Sort By Dropdown | Dropdown | Allows selection of sorting strategies. Trigger opens a dropdown list of options. | Default (defaults to "Listing date: new to old" or "View: high to low"), Hover, Open |
| 7 | Active Filter Tags | Tag Group | Horizontal row of removable chips representing currently active filters, displayed below the header bar. | Visible, Hidden (no filters applied) |
| 8 | Clear All Button | Link Button | "Clear All" — resets all filter inputs, search, and toggles to their default states. | Default, Hover, Hidden (no filters/search active) |
| 9 | Result Count Text | Text | Displays the total number of matching casks (e.g., "Showing 24 casks"). | Default |
| 10| Cask Card Grid | Grid Container | Grid displaying the matching Master Cask Cards. Adapts column count based on viewport width. | Default, Loading (skeletons), Empty |
| 11| Pagination Bar | Pagination Group | Bottom navigation controls containing Page Numbers, Previous, and Next buttons. | Default, Hover, Active, Disabled |

### Cask Card Components

| # | Component | Type | Description | States / Vintages |
|---|-----------|------|-------------|-------------------|
| 12| Cask Card Container | Container | Clickable card wrapper that routes the user to the Master Cask Detail page. | Default, Hover |
| 14| Cask Image | Image | Circular display of the Master Cask's product webp image. | Loaded, Fallback Barrel Placeholder |
| 15| Cask Name | Heading (H4) | Name of the master cask (e.g. "Aberlour Ex-Sherry Hogshead"). | Default |
| 16| Classification Label | Text | Category of the cask (e.g. "Single Malt"). | Default |
| 17| Floor Price Label | Text | Price title. Displays `"Floor Price"` if `lowestAsk != null`, or `"Expected value"` if `lowestAsk == null`. | Default |
| 18| Floor Price Value | Text | Displays the resolved price. E.g., `£15,200` (if `lowestAsk != null`) or `From £6,000` (if `lowestAsk == null` using `minReferencePrice`). | Default, Green Text (when lowestAsk exists) |
| 19| Delta Indicator | Badge | Growth percentage badge next to the floor price. Displays `lowestAskDelta30D` (e.g. `+2.4% LCI` or `-1.2% LCI`). | Green (positive), Red (negative), Hidden (if delta is null) |
| 20| Cask Type Detail | Text Field | Label "Cask Type" and value (e.g., "Ex-Sherry Hogshead"). | Default |
| 21| Region Detail | Text Field | Label "Region" and value (e.g., "Speyside"). | Default |
| 22| Vintages Range Detail| Text Field | Label "Vintages" and value (e.g., "1999 → 2004"). If min and max vintages are the same, displays single year (e.g., "2015"). | Default |
| 23| Availability Detail | Text Field | Label "Availability" and count of active asks/listings (e.g., "4"). | Default |
| 24| Card Action Button | Primary Button | Displayed at the bottom of the card. If `lowestAsk != null`, displays `"Buy Now"`. Otherwise, displays `"Make Offer"`. | Default, Hover, Active |

---

### Component Details

#### Sort By Dropdown — Detailed Specs
*   **Position**: Top right of the grid area.
*   **Default Value**: "Listing date: new to old" (`sortBy=latestListingDate`, `sortOrder=DESC`).
*   **Interaction**: Clicking the dropdown reveals the 9 sort options:
    1.  **Volume: high to low** (`sortBy=lifetimeVolume`, `sortOrder=DESC`)
    2.  **Volume: low to high** (`sortBy=lifetimeVolume`, `sortOrder=ASC`)
    3.  **Floor price: high to low** (`sortBy=lowestAsk`, `sortOrder=DESC`)
    4.  **Availability: high to low** (`sortBy=purchaseAvailability`, `sortOrder=DESC`)
    5.  **View: high to low** (`sortBy=viewCount`, `sortOrder=DESC`)
    6.  **Listing date: old to new** (`sortBy=latestListingDate`, `sortOrder=ASC`)
    7.  **Listing date: new to old** (`sortBy=latestListingDate`, `sortOrder=DESC`)
    8.  **Name: A to Z** (`sortBy=name`, `sortOrder=ASC`)
    9.  **Name: Z to A** (`sortBy=name`, `sortOrder=DESC`)
*   **Responsive Behavior**: Collapses to fit screen width on mobile, alignment shifts to full-width or icon trigger.

#### Card Action Button — Detailed Specs
*   **Position**: Bottom center/right of the card.
*   **Interaction**: 
    *   Clicking "Buy Now" routes the user to the checkout flow: `/checkout?caskId={caskId}&action=buy`.
    *   Clicking "Make Offer" routes the user to the bidding page: `/casks/{masterCaskId}/bid`.

---

## III. Field Validations

The filter panel contains input fields for numerical and text attributes. Validations are enforced on the client side before triggering searches.

| # | Field Name | Required | Min Value | Max Value | Format / Pattern | Custom Rules | Error Message |
|---|------------|----------|-----------|-----------|-----------------|--------------|---------------|
| 1 | Search Bar | ❌ | — | 100 characters | Text | Sanitized to remove script/special HTML characters. | "Search query is too long" |
| 2 | Min Price | ❌ | 0 | — | Integer (£) | Must be ≤ Max Price if Max Price is filled. | "Min price cannot exceed max price" |
| 3 | Max Price | ❌ | 0 | — | Integer (£) | Must be ≥ Min Price if Min Price is filled. | "Max price must be greater than min price" |
| 4 | Min Year | ❌ | 1800 | Current Year | 4-digit Integer | Must be ≤ Max Year if Max Year is filled. | "Enter a valid year range" |
| 5 | Max Year | ❌ | 1800 | Current Year | 4-digit Integer | Must be ≥ Min Year if Min Year is filled. | "Enter a valid year range" |
| 6 | Min ABV | ❌ | 0 | 100 | Decimal (%) | Must be ≤ Max ABV if Max ABV is filled. | "ABV must be between 0 and 100" |
| 7 | Max ABV | ❌ | 0 | 100 | Decimal (%) | Must be ≥ Min ABV if Min ABV is filled. | "ABV must be between 0 and 100" |
| 8 | Min RLA | ❌ | 0 | — | Decimal (litres) | Must be ≤ Max RLA if Max RLA is filled. | "RLA must be a positive number" |
| 9 | Max RLA | ❌ | 0 | — | Decimal (litres) | Must be ≥ Min RLA if Min RLA is filled. | "RLA must be a positive number" |
| 10| Min OLA | ❌ | 0 | — | Decimal (litres) | Must be ≤ Max OLA if Max OLA is filled. | "OLA must be a positive number" |
| 11| Max OLA | ❌ | 0 | — | Decimal (litres) | Must be ≥ Min OLA if Min OLA is filled. | "OLA must be a positive number" |
| 12| Min Bottles | ❌ | 0 | — | Integer | Must be ≤ Max Bottles if Max Bottles is filled. | "Bottle count must be positive" |
| 13| Max Bottles | ❌ | 0 | — | Integer | Must be ≥ Min Bottles if Min Bottles is filled. | "Bottle count must be positive" |

---

## IV. Cases and Flows

### IV.1. Load Marketplace Casks
| Aspect | Detail |
|--------|--------|
| **Trigger** | Page loads or user navigates to `/marketplace` |

#### Loading State
- The cask grid shows 6 to 9 card skeleton cards with pulsing animations.
- Filter sidebar options show simplified loading skeletons.
- All interactive controls (Toggles, Dropdown, Filters) are temporarily disabled.

#### Success State
- Grid is populated with Master Cask Cards matching the current filters/URL parameters.
- Page counts and total results count text update.
- URL parameters are updated to match search/filter/sort selections.
- Pagination controls render active state. Previous/Next buttons are disabled if on first/last page respectively.

#### Error States
| Error Condition | FE Behavior | User Message |
|-----------------|-------------|--------------|
| Server Error (500) | Renders error illustration in grid area, with a "Retry" button. | "Something went wrong on our server. Please try again." |
| Network Timeout | Renders timeout message with "Retry" button. | "Connection timed out. Please check your internet connection." |
| Unauthorized (401) | *Only if `ownedByMe=true` is set.* Uncheck the toggle, reset query parameter, reload list, and show toast alert. | "Please log in to view casks owned by you." |

#### Empty State
- Renders an empty state barrel illustration in the grid area with the text *"No casks match your filters"* and a primary button *"Clear Filters"*.

---

### IV.2. Apply Filters and Search
| Aspect | Detail |
|--------|--------|
| **Trigger** | User types in search bar, changes range fields, selects checkboxes, or toggles "Owned by you" / "Buy now available" |

#### Interaction Flow
1. **Debounce Inputs**: 
   - Checkbox options, Toggles, and Sort dropdowns apply immediately (0ms debounce).
   - Text search and numeric range inputs are debounced by **500ms** after the last keypress to prevent rapid API spam.
2. **Validate**: Client validates min/max rules. If invalid, display inline error underneath the specific input range and abort query.
3. **URL Update**: Replace browser state URL with updated query string representing all active filters.
4. **Fetch**: Trigger the grid loading state and request data from `GET /api/casks/listing` with updated parameters.
5. **Tags**: Render matching Active Filter tags with `×` icons below the top header bar.

---

### IV.3. Clear Filters
| Aspect | Detail |
|--------|--------|
| **Trigger** | User clicks "Clear All" or clicks the `×` on an Active Filter tag |

#### Success State
- **Clicking `×` on Tag**: Clears the corresponding filter (for checkboxes, unchecks the selected item; for range inputs, empties both Min and Max fields).
- **Clicking "Clear All"**: Resets all checkboxes (Category, Cask Type, Region, Distillery), clears all text and range inputs, sets toggles to `false`, and resets Sort By to default.
- URL is reset to `/marketplace` (default parameters).
- Grid reloads with default list.

---

## V. Edge Cases & FE-only Logic

| # | Scenario | Expected FE Behavior |
|---|----------|---------------------|
| 1 | User opens a direct URL with invalid/out-of-range parameters (e.g. `minPrice=abc`) | Ignore invalid parameters, log console warning, and strip the invalid values from the URL query string before loading the default page. |
| 2 | Slow network connection with overlapping requests | Implement an request abort mechanism (e.g. using `AbortController` in fetch). If a new filter request is sent before the previous one returns, abort the previous request to avoid race conditions and out-of-order renders. |
| 3 | Authenticated user logs out in another tab | If an "Owned by you" toggle request returns a `401 Unauthorized` response, redirect to login, clear local storage user data, and reset toggles. |
| 4 | Distillery search matches nothing | If the user types a search string into the Distillery scrollable search input that matches no distilleries, display: *"No distilleries found"* within that panel section. |
| 5 | Screen resize mid-filter | If mobile drawer is open and screen is resized to desktop width (≥ 1024px), close mobile drawer and render the static left sidebar panel seamlessly. |

---

## VI. Navigation & Routing

| Action | Destination | URL Change | Notes |
|--------|-------------|-----------|-------|
| Click Cask Card (General Area) | Master Cask Detail Page | `/marketplace/{masterCaskId}` | Navigates to details of the clicked master cask. |
| Click "Buy Now" Button | Checkout Page | `/checkout?caskId={caskId}&action=buy` | Redirects directly to Stripe checkout page. |
| Click "Make Offer" Button | Bidding Page | `/casks/{masterCaskId}/bid` | Opens bidding interface. |
| Clear All Filters | Same page (unfiltered) | `/marketplace` | Resets url parameters. |

---

## VII. Responsive Design Notes

| Breakpoint | Layout Changes |
|-----------|---------------|
| **Desktop (≥ 1024px)** | - Left sidebar is fixed (width: 280px) and scrollable.<br>- Right cask grid displays in a **3-column layout** (or 4-column on ultra-wide screens).<br>- All filter sections expanded by default. |
| **Tablet (768px – 1023px)** | - Left sidebar collapses into a slide-in bottom/left drawer.<br>- Right cask grid displays in a **2-column layout**.<br>- "Filters" toggle button visible in the top bar to open drawer.<br>- Toggles and search bar adjust to fit screen width. |
| **Mobile (< 768px)** | - Filter sidebar collapses into a full-screen slide-in drawer.<br>- Cask grid displays in a **1-column layout**.<br>- Top header bar places Search Input, Toggles, and Sort dropdown in a vertical stack to prevent horizontal overflow. |

---

## VIII. Accessibility Requirements

- [ ] Keyboard navigation: Users can tab through the sidebar filter items, toggles, search input, sort dropdown, card grid, and pagination. 
- [ ] All clickable items trigger action on `Space` or `Enter` keys.
- [ ] Focus indicators are clearly visible (using outline borders on focus).
- [ ] Screen readers: Checkboxes use `<label>` tags linked via `for` attribute. Active filter tags use `aria-live="polite"` to announce results changes.
- [ ] WCAG AA compliance: Text contrast ratio is ≥ 4.5:1. Active text/badges use strong high-contrast color palettes.

---

_End of Document 1_
