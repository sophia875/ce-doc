# 📄 Cask Filter – UI Specs

**Version**: 1.0  
**Last Updated**: 2026-04-09  
**Status**: Draft  

---

## I. Screen Overview

The Marketplace page allows users to browse and discover casks. The filter panel provides a set of criteria to narrow down the displayed results. Filters operate at the **Master Cask** level — a master cask appears in results if it matches the master-level filters (Distillery, Cask Type) AND has **at least one active, listed variant** matching the variant-level filters (Distillation Year, Price, ABV, RLA, OLA, Bottle Count).

* **Product Model**: Parent / Child (Master / Variant). See `docs/data-models/parent-child-structure.md`.
* **Screen URL**: `/marketplace`
* **User Role(s)**: All users (authenticated and anonymous).
* **Entry Point(s)**:
    * Navigation menu → Marketplace
    * "View All" from Home page curated sections
    * Direct URL access

---

## II. UI Elements Table

### Filter Panel

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Filter Panel Container | Sidebar / Drawer | Left-side filter panel on desktop, slide-in drawer on mobile | Default, Collapsed (mobile) |
| 2 | Filter Title | Heading | "Filters" — H3 heading with result count | Default |
| 3 | Clear All Button | Text Button | "Clear All" — resets all filter selections | Default, Hover, Hidden (no active filters) |

### Filter: Distillery

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 4 | Distillery Section | Collapsible Section | "Distillery" label with expand/collapse toggle | Expanded (default), Collapsed |
| 5 | Distillery Search | Text Input | Search box to filter distillery options by name | Default, Filled, Empty |
| 6 | Distillery Checkbox List | Checkbox Group | Scrollable list of distillery names with checkboxes. Multi-select enabled | Default, Selected, Scrollable |
| 7 | Selected Count Badge | Badge | Shows count of selected distilleries (e.g., "3 selected") | Default, Hidden (none selected) |

### Filter: Cask Type

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 8 | Cask Type Section | Collapsible Section | "Cask Type" label with expand/collapse toggle | Expanded (default), Collapsed |
| 9 | Cask Type Checkbox List | Checkbox Group | List of cask types with checkboxes. Multi-select | Default, Selected |
| 10 | Selected Count Badge | Badge | Count of selected cask types | Default, Hidden |

### Filter: Distillation Year

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 11 | Distillation Year Section | Collapsible Section | "Distillation Year" label with expand/collapse | Expanded, Collapsed |
| 12 | Min Year Input | Number Input | "From" — minimum distillation year | Default, Filled, Error |
| 13 | Max Year Input | Number Input | "To" — maximum distillation year | Default, Filled, Error |

### Filter: Price Range

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 14 | Price Range Section | Collapsible Section | "Price Range (£)" label with expand/collapse | Expanded, Collapsed |
| 15 | Min Price Input | Number Input | "Min" — minimum price | Default, Filled, Error |
| 16 | Max Price Input | Number Input | "Max" — maximum price | Default, Filled, Error |

### Filter: ABV

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 17 | ABV Section | Collapsible Section | "ABV (%)" label with expand/collapse | Expanded, Collapsed |
| 18 | Min ABV Input | Number Input | "Min" — minimum ABV percentage | Default, Filled, Error |
| 19 | Max ABV Input | Number Input | "Max" — maximum ABV percentage | Default, Filled, Error |

### Filter: RLA

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 20 | RLA Section | Collapsible Section | "RLA (Litres)" label with expand/collapse | Expanded, Collapsed |
| 21 | Min RLA Input | Number Input | "Min" — minimum Regauged Litres of Alcohol | Default, Filled, Error |
| 22 | Max RLA Input | Number Input | "Max" — maximum RLA | Default, Filled, Error |

### Filter: OLA

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 23 | OLA Section | Collapsible Section | "OLA (Litres)" label with expand/collapse | Expanded, Collapsed |
| 24 | Min OLA Input | Number Input | "Min" — minimum Original Litres of Alcohol | Default, Filled, Error |
| 25 | Max OLA Input | Number Input | "Max" — maximum OLA | Default, Filled, Error |

### Filter: Bottle Count

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 26 | Bottle Count Section | Collapsible Section | "Estimated Bottles" label with expand/collapse | Expanded, Collapsed |
| 27 | Min Bottle Input | Number Input | "Min" — minimum estimated bottle count | Default, Filled, Error |
| 28 | Max Bottle Input | Number Input | "Max" — maximum bottle count | Default, Filled, Error |

### Filter Actions (Mobile)

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 29 | Apply Filters Button | Primary Button | "Show Results ({count})" — applies filters and closes drawer (mobile only) | Default, Hover, Loading |
| 30 | Filter Toggle Button | Icon Button | Floating button to open filter drawer on mobile | Default, Badge (active filter count) |

### Result Area

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 31 | Active Filter Tags | Tag / Chip Group | Horizontal row of active filter tags with `×` remove action | Default, Empty |
| 32 | Result Count | Text | "Showing {count} casks" — total matching master casks | Default |
| 33 | Cask Card Grid | Card Grid | Grid of master cask cards (reusable Cask Card component) | Default, Loading, Empty |

---

## III. Field Validations

| # | Field Name | Required | Min Value | Max Value | Format / Pattern | Custom Rules | Error Message |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Min Year | ❌ | 1800 | Current Year | Integer (4 digits) | Must be ≤ Max Year if both provided | "Enter a valid year" |
| 2 | Max Year | ❌ | 1800 | Current Year | Integer (4 digits) | Must be ≥ Min Year if both provided | "Enter a valid year" |
| 3 | Min Price | ❌ | 0 | — | Number (£) | Must be ≤ Max Price if both provided | "Enter a valid price" |
| 4 | Max Price | ❌ | 0 | — | Number (£) | Must be ≥ Min Price if both provided | "Enter a valid price" |
| 5 | Min ABV | ❌ | 0 | 100 | Number (%) | Must be ≤ Max ABV if both provided | "Enter a valid ABV (0–100)" |
| 6 | Max ABV | ❌ | 0 | 100 | Number (%) | Must be ≥ Min ABV if both provided | "Enter a valid ABV (0–100)" |
| 7 | Min RLA | ❌ | 0 | — | Number (litres) | Must be ≤ Max RLA if both provided | "Enter a valid RLA" |
| 8 | Max RLA | ❌ | 0 | — | Number (litres) | Must be ≥ Min RLA if both provided | "Enter a valid RLA" |
| 9 | Min OLA | ❌ | 0 | — | Number (litres) | Must be ≤ Max OLA if both provided | "Enter a valid OLA" |
| 10 | Max OLA | ❌ | 0 | — | Number (litres) | Must be ≥ Min OLA if both provided | "Enter a valid OLA" |
| 11 | Min Bottles | ❌ | 0 | — | Integer | Must be ≤ Max Bottles if both provided | "Enter a valid number" |
| 12 | Max Bottles | ❌ | 0 | — | Integer | Must be ≥ Min Bottles if both provided | "Enter a valid number" |

---

## IV. Cases and Flows

### IV.1. Apply Filters

| Aspect | Detail |
| :--- | :--- |
| **Trigger** | User checks/unchecks a checkbox or changes a range input value (debounced — 500ms after last input change) |

#### Loading State
- Cask card grid shows a loading overlay or skeleton state.
- Active filter tags update immediately to reflect the user's current selections.
- On mobile, the "Show Results" button shows a spinner and the estimated count updates.

#### Success State
- Cask card grid re-renders with filtered master cask results.
- Result count text updates (e.g., "Showing 23 casks").
- URL query parameters update to reflect current filter state (enables shareable links).
- Active filter tags render below the search bar for quick visibility.

#### Error States

| Error Condition | FE Behavior | User Message |
| :--- | :--- | :--- |
| Min > Max on any range filter | Inline validation error, prevent request | "Min value cannot exceed max value" |
| Server Error | Show error state in result area | "Something went wrong. Please try again." |
| Network Timeout | Show retry option | "Connection timed out. Please try again." |

#### Empty State
- If no master casks match: Show empty state illustration with message "No casks match your filters" and a "Clear Filters" button.

---

### IV.2. Clear Individual Filter

| Aspect | Detail |
| :--- | :--- |
| **Trigger** | User clicks `×` on an active filter tag, or unchecks a checkbox |

#### Success State
- The specific filter is removed from the active filters.
- Results re-query with updated filter set.
- If removing a range filter (e.g., removing "ABV: 40–60"), both min and max for that range are cleared.

---

### IV.3. Clear All Filters

| Aspect | Detail |
| :--- | :--- |
| **Trigger** | User clicks "Clear All" button |

#### Success State
- All filter inputs reset to default (unchecked, empty).
- Active filter tags are removed.
- Results re-query without any filters (full unfiltered list).
- URL query parameters are cleared.

---

## V. Edge Cases & FE-only Logic

| # | Scenario | Expected FE Behavior |
| :--- | :--- | :--- |
| 1 | User enters only Min value in a range filter | Filter with ≥ Min, no upper bound restriction |
| 2 | User enters only Max value in a range filter | Filter with ≤ Max, no lower bound restriction |
| 3 | User enters same value for Min and Max | Filter for exact match |
| 4 | User enters Min > Max | Show inline validation error. Do not submit the filter |
| 5 | User types rapidly in range inputs | Debounce 500ms — filters apply only after user stops typing |
| 6 | User navigates to URL with filter query params | Pre-populate filter inputs from URL params on page load |
| 7 | User selects many distilleries (long checkbox list) | Distillery section is scrollable (max-height). Search input helps narrow options |
| 8 | Distillery search matches nothing | Show "No distilleries found" message within the section |
| 9 | User clicks browser back after applying filters | Restore previous filter state from URL history |
| 10 | User applies filters then navigates to cask detail, then returns | Restore filter state from URL (persisted in URL params) |

---

## VI. Navigation & Routing

| Action | Destination | URL Change |
| :--- | :--- | :--- |
| **Apply any filter** | Same page — results update | `/marketplace?distillery=X&caskType=Y&minPrice=1000&...` |
| **Click Cask Card** | Master Cask Detail Page | `/marketplace/{masterCaskId}` |
| **Clear All Filters** | Same page — full results | `/marketplace` |

---

_End of Document_
