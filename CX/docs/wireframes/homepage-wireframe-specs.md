# Homepage Wireframe Specification
## Mid-Fi Wireframe v1.0

**Wireframe File:** [homepage-midfi.html](file:///Users/alice/Documents/cask-exchange/docs/wireframes/homepage-midfi.html)  
**Date:** April 2026  
**Based on:** [UX Audit Report](file:///Users/alice/Documents/cask-exchange/docs/ux-audit-homepage.md) | [PRD MVP](file:///Users/alice/Documents/cask-exchange/docs/prd-mvp.md)

---

## Page Architecture Overview

The redesigned homepage is structured as **12 sections** from top to bottom, each addressing specific gaps identified in the UX audit. The page is designed for **authenticated users** (the primary use case post-MVP).

```
┌─────────────────────────────────────────────┐
│  1. Header (sticky)                         │  Navigation + Search + Profile
│     Search bar · Nav links · Notifications  │
├─────────────────────────────────────────────┤
│  2. Portfolio Snapshot Bar                   │  Personal holdings at a glance
│     Value · Casks Owned · Open Orders       │
├─────────────────────────────────────────────┤
│  3. Trust Bar                               │  Elevated from footer
│     Verified · Secure · Certificates · Bond │
├─────────────────────────────────────────────┤
│  4. Quick Filters                           │  Classification pills + 
│     Browse by Classification                │  Distillery chip carousel
│     Popular Distilleries                    │
├─────────────────────────────────────────────┤
│  5. Market Stats Bar                        │  Platform-wide aggregate data
│     247 Casks · 19 Distilleries · £2.4M     │
├─────────────────────────────────────────────┤
│  6. High Interest Casks 🔥                  │  Featured hero card + 2 stacked
│     (visual differentiation from other      │  cards (different layout)
│      curated sections)                      │
├─────────────────────────────────────────────┤
│  7. Strategic Buys 💡                       │  Standard 4-col card grid
│     Enriched cards with specs + dual price   │
├─────────────────────────────────────────────┤
│  8. In High Demand 📈                       │  Standard 4-col card grid
│     Same enriched card component             │
├─────────────────────────────────────────────┤
│  9. Market Activity                         │  Chart (left) + Trade Feed 
│     Price trend chart · Recent trades list   │  (right)
├─────────────────────────────────────────────┤
│ 10. Recently Viewed                         │  Personalised re-engagement
│     Standard 4-col card grid                 │
├─────────────────────────────────────────────┤
│ 11. How It Works                            │  4-step educational section
│     Browse → Bid/Buy → Secure → Hold/Trade   │
├─────────────────────────────────────────────┤
│ 12. Footer                                  │  Navigation, legal, support
└─────────────────────────────────────────────┘
```

---

## Section-by-Section Specification

### Section 1: Header (Sticky Navigation)

| Element | Detail |
|---------|--------|
| **Logo** | "CASK EXCHANGE" — serif font, accent colour on "EXCHANGE" |
| **Search Bar** | Centred, prominent placeholder: "Search casks, distilleries, regions..." |
| **Nav Links** | Home (active) · Marketplace · Distilleries · How It Works |
| **Right Actions** | Notification bell (icon button) · User avatar (initials) |
| **Behaviour** | Sticky on scroll, blur backdrop |

> **Audit Fix:** Adds the missing **search bar** (Weakness #4.8) and simplifies navigation labels (Weakness #4.7) — "Buying"/"Selling" replaced with "Marketplace" and "How It Works".

---

### Section 2: Portfolio Snapshot Bar

| Element | Data Source | Detail |
|---------|-------------|--------|
| **Greeting** | User profile | "Welcome back, {firstName}" + last login |
| **Portfolio Value** | SUM(estimated values) of owned casks | £124,500 with 30-day % change |
| **Casks Owned** | COUNT(owned casks) | Number + distillery count |
| **Open Orders** | COUNT(open bids + asks) | "1 bid · 1 ask" breakdown |
| **CTA** | Link | "View Portfolio →" to /portfolio |

> **Audit Fix:** Addresses Catherine's need for **homepage relevance** (Weakness #5 Persona Analysis) and gives both personas a reason to return to the homepage daily.

**Visibility Rule:** Only shown for authenticated users with ≥1 owned cask OR ≥1 open order. For brand-new users with no activity, this section is replaced by a welcome/onboarding prompt.

---

### Section 3: Trust Bar (Elevated)

| Element | Content |
|---------|---------|
| Trust Signal 1 | ✓ Verified Distillery Partners |
| Trust Signal 2 | 🔒 Secure Escrow Payments |
| Trust Signal 3 | 📜 Ownership Certificates Included |
| Trust Signal 4 | 🏦 Bonded Warehouse Storage |

> **Audit Fix:** Trust signals **elevated from page bottom** (Weakness #4.5). This slim bar immediately follows the portfolio, reinforcing credibility before the user starts browsing.

---

### Section 4: Quick Filters — Classification & Distillery

#### 4a. Browse by Classification

| Pill | Navigates to | Data |
|------|-------------|------|
| Single Malt | `/marketplace?classification=single_malt` | Count of listed casks |
| Single Grain | `/marketplace?classification=single_grain` | Count of listed casks |
| Blended Malt | `/marketplace?classification=blended_malt` | Count of listed casks |
| Bourbon | `/marketplace?classification=bourbon` | Count of listed casks |
| Irish Whiskey | `/marketplace?classification=irish_whiskey` | Count of listed casks |

**Data Model Mapping:** `classification` is a **Master Cask (Parent)** level field.

#### 4b. Popular Distilleries (horizontal scroll)

| Element | Format | Navigates to |
|---------|--------|-------------|
| Distillery Chip | Avatar initials + Name + Cask count | `/marketplace?distillery={distilleryId}` |

**Data Model Mapping:** `distillery` is a **Master Cask (Parent)** level FK. Count = number of master casks with ≥1 active, listed variant under that distillery.

> **Audit Fix:** Provides **targeted entry points** directly into the marketplace with filters pre-applied (user suggestion). Replaces the current circular image carousel with interactive filter chips that communicate utility more clearly.

---

### Section 5: Market Stats Bar

| Stat | Source | Purpose |
|------|--------|---------|
| Listed Casks | COUNT(master casks with ≥1 active variant) | Inventory size |
| Distilleries | COUNT(active distilleries) | Partner breadth |
| Total Value Traded | SUM(completed transaction values) | Trading volume |
| Trades This Month | COUNT(completed trades, current month) | Recent activity |
| Registered Investors | COUNT(active user accounts) | Social proof |

> **Audit Fix:** Addresses the **missing market data** (Weakness #4.6). These stats signal an active, liquid marketplace — critical for both personas.

---

### Section 6: Curated List — "High Interest Casks" (Featured Layout)

This section uses a **deliberately different layout** from Sections 7–8 to create visual hierarchy (Audit Weakness #4.4).

| Layout | Detail |
|--------|--------|
| **Left:** Featured Hero Card | Large horizontal card (2-column: image + full specs + pricing) |
| **Right:** 2 Stacked Cards | Standard enriched cards, vertically stacked |

The hero card shows the #1 most-viewed cask with full detail including all price points and a CTA button.

---

### Section 7–8: Curated Lists — Standard Grid

"Strategic Buys" and "In High Demand" use the standard **4-column card grid**.

---

### Enriched Cask Card Component

This is the core card redesign — compare with the current card which shows only name + single price.

#### Card Anatomy

```
┌─────────────────────────┐
│  [Cask Image]           │  ← Master: image
│  ┌─────────┐ ┌────────┐ │
│  │Classif. │ │ Region │ │  ← Badges: Master fields
│  └─────────┘ └────────┘ │
│              3 variants  │  ← COUNT(active children)
├─────────────────────────┤
│  DISTILLERY NAME         │  ← Master: distillery.name
│  Cask Full Name          │  ← Master: name (or child name)
│                          │
│  Age     ABV     RLA     │  ← Variant-level (aggregated)
│  13 yrs  68.4%   150 L   │
│─────────────────────────│
│  Lowest Ask   Highest Bid│  ← Variant: lowestAsk, highestBid
│  £10,000      £8,500     │     (aggregated across variants)
│               ▲ +2.8%    │  ← Price trend (calc from trades)
└─────────────────────────┘
```

#### Data Field Mapping

| Card Element | Field | Level | Source / Aggregation |
|-------------|-------|-------|---------------------|
| **Cask Image** | `image` | Parent | Master cask primary image |
| **Classification Badge** | `classificationLabel` | Parent | e.g. "Single Malt Scotch" |
| **Region Badge** | `region.name` | Parent | e.g. "Highland" |
| **Variant Count** | computed | — | COUNT of children where `caskStatus='active'` AND `isListed=true` |
| **Distillery** | `distillery.name` | Parent | FK to distillery |
| **Cask Name** | `name` | Parent | Master cask name (max 2 lines) |
| **Age** | `age` | Child | Display: range if multiple variants (e.g. "10–15 yrs") or single value |
| **ABV** | `abv` | Child | Display: range or single value (e.g. "63.5%") |
| **RLA** | `rla` | Child | Display: range or single value. If `rla = 0`, show `ola` with label "OLA" |
| **Cask Type** | `caskType.name` | Parent | Shown in specs row (3rd column) OR on featured card |
| **Lowest Ask** | `lowestAsk` | Child (aggregated) | MIN(lowestAsk) across active variants — green colour |
| **Highest Bid** | `highestBid` | Child (aggregated) | MAX(highestBid) across active variants — amber colour |
| **Price Trend** | computed | — | % change from last trade price vs. current price |
| **Expected Value** | `referencePriceMin` | Child (aggregated) | Shown when NO active asks exist — as "From £X" |
| **Ref. Max** | `referencePriceMax` | Child (aggregated) | Shown alongside Expected Value when no asks |

#### Price Display Logic (Updated)

```
IF any active variant has lowestAsk ≠ null:
  ├── Left:  "Lowest Ask"  →  £{MIN(lowestAsk)}  [GREEN]
  └── Right: "Highest Bid" →  £{MAX(highestBid)}  [AMBER]
              + trend indicator if trade history exists

ELSE (no active asks):
  ├── Left:  "Expected Value"  →  From £{MIN(referencePriceMin)}  [GREY]
  └── Right: "Ref. Max"        →  £{MAX(referencePriceMax)}       [GREY]
```

> **Audit Fix:** Addresses **single-price cards** (Weakness #4.2) by showing both sides of the order book. Adds **cask specifications** (Age, ABV, RLA) that are critical for investment evaluation. Adds **classification and region badges** for at-a-glance categorisation.

---

### Section 9: Market Activity

| Panel | Content | Purpose |
|-------|---------|---------|
| **Left: Price Chart** | Platform Average Cask Value index over time | Signals market direction |
| **Right: Recent Trades** | Live feed of completed BUY/SELL trades with cask name, price, time | Signals active marketplace |

**Chart Tabs:** 7D · 1M · 3M · 1Y · All — allows users to explore different timeframes.

**Trade Feed Fields:**
| Element | Detail |
|---------|--------|
| Trade Type | BUY (green) / SELL (red) |
| Cask Name | Truncated master cask name |
| Price | Transaction price in £ |
| Time | Relative timestamp ("2h ago", "1d ago") |

> **Audit Fix:** Addresses the **missing market activity signals** (Weakness #4.6) and serves Catherine's need for **market trend data**. Also builds liquidity confidence for Marcus.

---

### Section 10: Recently Viewed

Standard 4-column grid with same enriched card component. Only shown when the user has server-side view history. Hidden for first-time visitors.

---

### Section 11: How It Works

| Step | Title | Description |
|------|-------|-------------|
| 1 | Browse & Discover | Explore casks from verified distilleries. Filter by region, age, classification, and price range. |
| 2 | Place a Bid or Buy Now | Set your own price with a Bid and wait for a match — or buy instantly at the lowest available Ask price. |
| 3 | Secure Ownership | Once matched, payment is processed securely. Ownership certificates and bond documents are transferred to you. |
| 4 | Hold or Trade | Your cask matures in a bonded warehouse. Track its value over time and sell whenever you're ready. |

> **Audit Fix:** Directly addresses **missing platform education** (Weakness #4.3). Uses plain, jargon-light language aligned to Marcus's comprehension needs. Explains "Bid" and "Ask" in context without using financial jargon.

---

### Section 12: Footer

Standard footer with 4-column layout:
- **Brand column:** Logo + platform description
- **Platform:** Marketplace, Distilleries, How It Works, Pricing & Fees
- **Account:** My Portfolio, My Orders, Transaction History, Settings
- **Support:** Help Centre, Contact Us, FAQs, Report an Issue
- **Bottom bar:** Copyright + Privacy Policy, Terms of Service, Cookie Policy

---

## Audit Gap Coverage Summary

| Audit Weakness | Wireframe Section | Status |
|---------------|-------------------|--------|
| #4.1 — Hero banner / weak value prop | Replaced by Portfolio Bar + Quick Filters | ✅ Resolved |
| #4.2 — No bid/ask on cards | Enriched card with dual pricing + trend | ✅ Resolved |
| #4.3 — No "How It Works" | Section 11: 4-step educational flow | ✅ Resolved |
| #4.4 — Repetitive carousels | Featured layout (Sec 6) vs standard grid (Sec 7-8) | ✅ Resolved |
| #4.5 — Trust signals buried | Section 3: Elevated trust bar | ✅ Resolved |
| #4.6 — No market data | Section 5 (stats) + Section 9 (chart & trades) | ✅ Resolved |
| #4.7 — Navigation clarity | Simplified: Home, Marketplace, Distilleries, How It Works | ✅ Resolved |
| #4.8 — No search | Header search bar with broad scope | ✅ Resolved |
| #4.9 — Price label confusion | Updated labels + dual pricing context | ✅ Resolved |
| #4.10 — Limited personalisation | Section 2 (portfolio) + Section 10 (recently viewed) | ✅ Resolved |

---

*End of Wireframe Specification — v1.0*
