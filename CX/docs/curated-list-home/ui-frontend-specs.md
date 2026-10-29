# 📄 Curated List for Home – UI Specs

**Version**: 1.0  
**Last Updated**: 2026-04-08  
**Status**: Draft  

---

## I. Screen Overview

The Home page features a curated collection of cask recommendations displayed as horizontally-scrollable carousels. Each section highlights master casks (parent products) using a different strategy — popularity, pricing opportunity, market demand, and personal browsing history — to help users discover and evaluate investment-worthy casks quickly.

* **Product Model**: The platform uses a Parent / Child (Master / Variant) structure. All cards on this page represent master casks (parent level). When a user clicks a card, they navigate to the master cask detail page where they can browse and select specific variants.
* **Screen URL**: `/` (Home page)
* **User Role(s)**: All authenticated users.
* **Entry Point(s)**:
    * Direct URL access to `/`
    * Logo click from any page

### Data Model Context

| Level | Fields Displayed on Card | Source |
| :--- | :--- | :--- |
| **Parent (Master Cask)** | `name`, `distillery`, `image` | Master cask |
| **Parent (Master Cask)** | `lowestAsk` = MIN(lowestAsk) across all variants | Computed per master cask |
| **Parent (Master Cask)** | `referencePriceMin` = MIN(referencePriceMin) across all variants | Computed per master cask |
| **Parent (Master Cask)** | `referencePriceMax` = MAX(referencePriceMax) across all variants | Computed per master cask |

---

## II. UI Elements Table

### Section: High Interest Casks 🔥

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Section Title | Heading | "High Interest Casks" — H2 heading | Default |
| 2 | Tooltip Badge | Badge / Tooltip | "Most-watched casks by investors" | Default (visible) |
| 3 | Carousel Container | Carousel | Horizontal scrollable row (5 cards on desktop) | Default, Scrolling, Empty, Insufficient |
| 4 | Scroll Left Arrow | Icon Button | `<` arrow to scroll carousel left | Default, Hover, Disabled, Hidden |
| 5 | Scroll Right Arrow | Icon Button | `>` arrow to scroll carousel right | Default, Hover, Disabled, Hidden |
| 6 | View All Button | Button (outlined) | Navigates to full listing for this category | Default, Hover |
| 7 | Cask Card | Card | Individual cask card component | Default, Hover, Loading |

### Section: Strategic Buys 💡

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 8 | Section Title | Heading | "Strategic Buys" — H2 heading | Default |
| 9 | Tooltip Badge | Badge / Tooltip | Dark pill label: "Competitive market prices" | Default (visible) |
| 10-14 | Same as #3 - #7 | - | - | - |

### Section: In High Demand

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 15 | Section Title | Heading | "In High Demand" — H2 heading | Default |
| 16-19 | Same as #3 - #7 | - | - | - |

### Section: Recently Viewed

| # | Component | Type | Description | States / Variants |
| :--- | :--- | :--- | :--- | :--- |
| 20 | Section Title | Heading | "Recently Viewed" — H2 heading | Default |
| 21 | Carousel Container | Carousel | Data sourced from server-side view history | Default, Scrolling, Empty, Hidden |

---

## Component Details

### 1. Cask Card — Detailed Specs
Each cask card is a reusable component. Data fields come from the parent entity, while price fields are aggregated from child variants.

| # | Sub-Component | Type | Description | Data Source |
| :--- | :--- | :--- | :--- | :--- |
| **C1** | Card Container | Container | Clickable wrapper → `/marketplace/{masterCaskId}` | — |
| **C2** | Cask Image | Image | Product image of the cask barrel | Master: `image` |
| **C3** | Cask Name | Text | Full name of the master cask (Max 2 lines) | Master: `name` |
| **C4** | Price Label | Text | "Expected value" OR "Lowest ask" | Computed from variants |
| **C5** | Price Value | Text | "£{value}" or "From £{value}" | Computed from variants |

> **Price Display Logic (FE-only):**
> * **IF** master cask has any variant with an **active ask**:
>     * Label: `Lowest ask` | Value: `£{lowestAsk}`
> * **ELSE**:
>     * Label: `Expected value` | Value: `From £{referencePriceMin}`

### 2. Carousel — Detailed Specs
* **Visible Cards**: 5 per view on desktop.
* **Scroll Behavior**: Smooth horizontal scroll by 1 card width per arrow click.
* **Loop**: No loop — arrows disable at start/end of list.
* **Drag/Swipe**: Support touch swipe on mobile and tablet devices.

---

## III. Navigation & Routing

| Action | Destination | URL Change |
| :--- | :--- | :--- |
| **Click Cask Card** | Master Cask Detail Page | `/marketplace/{masterCaskId}` |
| **Click "View All" (High Interest)** | Marketplace — Popularity | `/marketplace?sortBy=popularity&sortOrder=desc` |
| **Click "View All" (Strategic Buys)** | Marketplace — Lowest Ask | `/marketplace?sortBy=lowestAsk&sortOrder=asc` |

---
_End of Document_