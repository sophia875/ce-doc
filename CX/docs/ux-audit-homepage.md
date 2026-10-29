# UX/UI Audit Report — Homepage
## Cask Exchange MVP

**Version:** 1.0  
**Date:** April 2026  
**Auditor:** UX Specialist  
**Scope:** Homepage (`/`) — Authenticated User View  
**Reference Documents:** PRD v1.0, Curated List Home UI Specs  
**Competitor Benchmarks:** StockX (bid/ask marketplace), OpenSea (digital asset marketplace)

---

## Table of Contents

1. [Audit Objectives & Methodology](#1-audit-objectives--methodology)
2. [Homepage Section Breakdown](#2-homepage-section-breakdown)
3. [Strengths](#3-strengths)
4. [Weaknesses & Opportunities](#4-weaknesses--opportunities)
5. [Persona Alignment Analysis](#5-persona-alignment-analysis)
6. [Competitor Benchmark Comparison](#6-competitor-benchmark-comparison)
7. [Recommendations Summary](#7-recommendations-summary)

---

## 1. Audit Objectives & Methodology

### Audit Goals
This audit evaluates the current Cask Exchange homepage against the PRD's stated objectives, user personas, and competitive benchmarks. The focus areas are:

- **Goal Alignment:** Does the homepage support the platform's objective of being a *trusted, transparent marketplace*?
- **Discovery Effectiveness:** Does the layout enable efficient Browse → Interest → Action flow for both persona types?
- **Content Richness vs. Accessibility:** Does the design strike the right balance between data density (for Catherine/Active Traders) and approachability (for Marcus/Enthusiast Investors)?
- **Brand & Tone:** Does the visual identity match the stakeholder requirement of "classic and content-rich, not too techy"?

### Methodology
- Visual analysis of the homepage UI screenshot
- Cross-reference with PRD user personas, user stories, and feature requirements
- Benchmarking against StockX (bid/ask marketplace model) and OpenSea (asset marketplace model)
- Evaluation via heuristic framework (Nielsen's 10 Usability Heuristics + marketplace-specific criteria)

---

## 2. Homepage Section Breakdown

The homepage is structured as a vertically-stacked page with the following sections (top to bottom):

| # | Section | Content | Purpose |
|---|---------|---------|---------|
| 1 | **Navigation Bar** | Logo, Marketplace, Distillery, Buying, Selling, Admin links + Profile avatar | Global navigation |
| 2 | **Hero Banner** | "Introducing our latest Cask release" + Explore CTA | Promotional showcase |
| 3 | **Popular Distilleries** | Horizontal carousel of distillery cards (circular images) | Distillery-led discovery |
| 4 | **High Interest Casks 🔥** | Carousel of 5 cask cards with tooltip "Most-watched casks by investors" | Popularity-based curation |
| 5 | **Strategic Buys 💡** | Carousel of 5 cask cards with tooltip "Competitive market prices" | Value-based curation |
| 6 | **In High Demand** | Carousel of 5 cask cards | Demand-based curation |
| 7 | **Recently Viewed** | Carousel of previously viewed cask cards | Personalised re-engagement |
| 8 | **Trust & Transparency** | 3-column trust badges (Verified, Secure Transactions, Provenance) | Credibility signals |
| 9 | **Footer** | Logo, copyright, legal links, partner badges (Comodo, Mastercard, etc.) | Standard footer |

---

## 3. Strengths

### 3.1 ✅ Strong Curated Discovery Model
The homepage uses multiple curated carousels (High Interest, Strategic Buys, In High Demand, Recently Viewed) that mirror proven marketplace patterns from StockX's category-driven homepage. Each section serves a distinct discovery strategy:
- **High Interest Casks** → social proof / popularity signal
- **Strategic Buys** → value-oriented (appeals to price-conscious Enthusiast Investor)
- **In High Demand** → scarcity / urgency signal
- **Recently Viewed** → personalised re-engagement

> **PRD Alignment:** Directly supports **US-01** (Browse all available casks) and addresses Marcus's behaviour of *"browsing the listing page extensively before committing."*

### 3.2 ✅ Classic, Warm Visual Tone — Appropriate to Product
The dark background with warm amber/gold accents creates a premium, whisky-adjacent aesthetic that avoids the "crypto/fintech" look of competitors like OpenSea. This directly fulfils the stakeholder directive:

> *"The client doesn't want the platform to be too 'techy' or modern. The product is barrels so it needs to be friendly to our TA who are classic and old school."*

**What works well:**
- Dark theme with amber CTA buttons evokes barrel-aged spirits
- Real cask/barrel photography in cards grounds the experience in the physical product
- Distillery cards use rounded images, giving a softer, more editorial feel
- The hero banner uses an atmospheric whisky lifestyle image

### 3.3 ✅ Trust Section Present at Page Bottom
The three-column trust section ("Cask Factory verified", "Secure & Transparent Transactions", "Provenance & Authenticity Guaranteed") addresses a critical concern for both personas — especially Marcus, whose pain points include being *"concerned about payment security"* and *"unsure how to verify cask quality or authenticity."*

> **PRD Alignment:** Supports the business objective of establishing Cask Exchange as a *"trusted, transparent marketplace."*

### 3.4 ✅ Consistent Card Layout
Cask cards across all sections use a consistent component pattern:
- Cask barrel image
- Cask name (with distillery)
- Price information (Lowest Ask or Expected Value)

This consistency reduces cognitive load and makes scanning across carousels intuitive.

### 3.5 ✅ Distillery-Led Discovery
The "Popular Distilleries" row provides an alternative discovery path centred on producer reputation — directly supporting **US-03** (*"View distillery profiles so I understand the producer and their reputation"*). This is especially important because:
- Both personas evaluate casks partly based on distillery reputation
- Catherine focuses on *"provenance, distillery reputation, and market data"*

### 3.6 ✅ Clear Primary Navigation
The top navigation clearly segments the platform's main areas (Marketplace, Distillery, Buying, Selling, Admin) — matching the Information Architecture defined in the PRD. The persistent header gives users consistent orientation.

---

## 4. Weaknesses & Opportunities

### 4.1 ⚠️ Hero Banner — Weak Value Proposition & Low Information Density

**Issue:** The hero banner ("Introducing our latest Cask release") is generic and promotional. It reads like a whisky brand launch announcement, not a marketplace entry point. The subheading text is barely legible at the current contrast ratio.

**Impact:**
- **Marcus (Enthusiast):** Gains no understanding of *what* Cask Exchange does or *how* it works. PRD identifies he *"needs clear guidance"* and *"jargon-light explanations of trading mechanics."* The hero provides none.
- **Catherine (Active Trader):** Sees no actionable market data. Her focus is on *"price charts and order book depth"* — the hero wastes prime viewport real estate.
- **New visitors:** No onboarding hook. The hero assumes the user already understands the platform.

**Competitor Reference:**
- **StockX** uses the hero to spotlight trending items with real pricing data, making it immediately actionable.
- **OpenSea** features trending collections with volume/floor price metrics directly in the hero section.

**Recommendation:**
> Replace the generic promotional hero with a content-rich, value-driven section. Options:
> - A short platform explainer tagline ("Buy & sell maturing whisky casks. Like the stock exchange, for barrels.") + a featured cask with live pricing
> - A market snapshot: total casks traded, total value, number of distilleries — to build credibility
> - Or rotate between editorial (latest release) and onboarding (how it works)

---

### 4.2 ⚠️ No Bid/Ask Market Data Visible on Cards

**Issue:** Cask cards display only a single price metric ("Lowest ask: £X" or "Expected value: From £X"). There is **no Bid price, no spread, no price change indicator** on any card.

**Impact:**
- **Catherine (Active Trader):** Cannot assess market opportunity at a glance. She needs to see both sides of the order book (Bid and Ask) to gauge spread and decide whether to act. Per PRD, she *"focuses heavily on price charts and order book depth per cask."*
- **Marcus (Enthusiast):** The single price with no context may feel opaque. He doesn't know if the price is high, low, or trending.

**Competitor Reference:**
- **StockX** shows both the current Ask and last sale price on every card, plus a percentage price change indicator (e.g., ▲ +8%). This immediately communicates market movement and relative value.

**Recommendation:**
> Add a secondary price data point to each card — at minimum:
> - Show both **Lowest Ask** and **Highest Bid** (or at least a Bid indicator if one exists)
> - Add a **price trend indicator** (e.g., ▲ +5% last 30d) to signal market movement
> - This aligns the card with the PRD's bid/ask exchange model without adding excessive complexity

---

### 4.3 ⚠️ No Clear Explanation of Platform Mechanics

**Issue:** The homepage provides zero education on how the bid/ask model works. There is no "How it Works" section, no explainer, no onboarding prompt.

**Impact:**
- **Marcus (Enthusiast):** The PRD explicitly states he is *"unfamiliar with Bid/Ask order mechanics — needs clear guidance."* The current homepage offers none. This creates a significant comprehension barrier before the user even reaches a cask detail page.
- First-time visitors from a whisky enthusiast background (not trading) will have no context for terms like "Ask," "Bid," or the transaction model.

**Competitor Reference:**
- **StockX** has a persistent "How It Works" section in the footer and onboarding tooltips.
- **OpenSea** uses contextual education (inline tooltips, hover explanations) throughout.

**Recommendation:**
> Add a lightweight "How It Works" section between the curated lists and the trust section. A simple 3-step visual:
> 1. **Browse** — Explore casks from renowned distilleries
> 2. **Place a Bid or Buy Now** — Set your price or buy instantly
> 3. **Own & Trade** — Your cask matures while you hold or trade it
>
> Keep it visual, jargon-light, and aligned to Marcus's mental model. This doesn't need to be complex — a single illustrated row would suffice.

---

### 4.4 ⚠️ Carousel Sections Are Visually Repetitive

**Issue:** The four curated carousel sections (High Interest, Strategic Buys, In High Demand, Recently Viewed) use **identical card layouts, identical carousel behaviour, and very similar cask imagery** (same barrel photos across sections). The visual differentiation between sections is limited to the section title and a small tooltip badge.

**Impact:**
- **Cognitive overload:** As a user scrolls, the sections blur together. The user cannot quickly distinguish whether "Strategic Buys" offers a meaningfully different selection from "High Interest" without reading every card.
- **Reduced engagement:** When sections look the same, users may skip later sections (In High Demand, Recently Viewed) assuming they've already seen the content.
- **Scroll fatigue:** Four horizontally-scrollable carousels stacked vertically demands significant scroll effort. Without clear visual breaks, the page feels monotonous.

**Competitor Reference:**
- **StockX** uses varied visual treatments for different sections: hero-sized featured items, grid layouts for trending categories, and smaller cards for recently viewed. This creates visual rhythm and maintains engagement.
- **OpenSea** alternates between large collection banners, grid-based trending items, and category pills — each section feels distinct.

**Recommendation:**
> Introduce visual variation between sections:
> - **Feature one section differently** — e.g., the "High Interest Casks" section could use larger cards or a highlighted/featured card as the first item
> - **Alternate layout styles** — consider a grid for one section instead of a carousel
> - **Add distinct visual markers** per section — subtle background colour shifts, icons, or differing card sizes
> - **Consider reducing to 3 curated sections** and use the freed space for educational or market-data content

---

### 4.5 ⚠️ Trust Section Is Too Generic and Buried

**Issue:** The "Ensuring Trust & Transparency in Cask Investments" section at the bottom of the page features three generic icon+text cards. The copy is largely boilerplate and the section is positioned below the fold — most users may never scroll to see it.

**Impact:**
- **Marcus (Enthusiast):** Trust is his #1 blocker. He is *"concerned about payment security and fund handling"* and *"unsure how to verify cask quality or authenticity."* Burying the trust signals at page bottom means they fail to influence his critical early-stage browsing.
- **Catherine (Active Trader):** While she needs less convincing on trust, visible credibility signals (regulated entity, verified distilleries) reinforce her confidence in the platform.

**Recommendation:**
> - **Elevate trust signals higher on the page** — consider a slim trust bar below the hero or between the first two sections (e.g., "✓ Verified Distilleries • ✓ Secure Payments • ✓ Ownership Certificates")
> - **Make the trust section content more specific** — replace generic language with concrete proof points: "120+ verified casks", "Partnered with 15 distilleries", "All casks bonded & insured"
> - **Add social proof elements** — testimonials, partner logos (distilleries), or trading volume stats

---

### 4.6 ⚠️ No Market Overview / Platform Data

**Issue:** The homepage presents zero aggregate market data — no total trading volume, no number of listed casks, no count of active distilleries, no recent activity feed.

**Impact:**
- **Both personas** need assurance that this is an active marketplace with liquidity. A silent homepage with no activity indicators can trigger *"is anyone else using this?"* anxiety, which is lethal for marketplace adoption.
- The PRD objective to *"enable liquidity in an otherwise illiquid asset class"* is undermined if users cannot see evidence of active trading.

**Competitor Reference:**
- **OpenSea** prominently displays volume, floor price, and number of items for collections.
- **StockX** shows last sale data and number of asks on every product, communicating a live, active market.

**Recommendation:**
> Add a **market snapshot section** or **inline stats** near the top of the page:
> - Total casks listed | Total value traded | Active distilleries
> - This can be a simple horizontal stats bar (similar to a financial dashboard)

---

### 4.7 ⚠️ Navigation Label Clarity

**Issue:** The navigation includes separate "Buying" and "Selling" menu items. For a user who hasn't made a purchase yet, the distinction between "Marketplace" and "Buying"/"Selling" is unclear. Are Buying/Selling action modes, or are they information pages?

**Impact:**
- **Marcus (Enthusiast):** May be confused about where to start. Is "Marketplace" for browsing and "Buying" for purchasing? Or does "Marketplace" already contain buying functionality?
- This creates a navigation ambiguity that the PRD's IA doesn't reflect — the PRD groups "Browse" separately from "Authenticated" actions.

**Recommendation:**
> - Consider consolidating navigation: "Marketplace" for all browse/buy, "My Portfolio" or "My Orders" for authenticated sell/manage actions
> - Alternatively, clarify the Buying/Selling links as **"How Buying Works"** and **"How Selling Works"** if they are educational pages
> - The navigation should clearly separate *discovery* from *transaction management*

---

### 4.8 ⚠️ No Search Functionality Visible

**Issue:** There is no search bar or search icon in the navigation or anywhere on the homepage.

**Impact:**
- **Catherine (Active Trader):** Likely arrives knowing exactly what distillery or cask type she wants. Without search, she must navigate through the Marketplace listing and apply filters — adding friction to her *"fast, minimal-click"* workflow requirement.
- **Marcus (Enthusiast):** May want to search for a distillery he's heard about. No search means he must visually scan the Popular Distilleries carousel or navigate to a separate page.

**Competitor Reference:**
- **StockX** has a prominent search bar centred in the navigation — search is the primary discovery mechanism.
- **OpenSea** similarly features a search bar as the dominant navigation element.

**Recommendation:**
> Add a **search bar** (or at minimum a search icon with expandable input) to the main navigation. This is a critical marketplace pattern.

---

### 4.9 ⚠️ Cask Card Price Labels May Confuse New Users

**Issue:** Cards use two different price labels — "Lowest ask" and "Expected value" — depending on whether the cask has active asks. The term "Expected value" is ambiguous and could be interpreted in multiple ways (predicted future value? appraised value? list price?).

**Impact:**
- **Marcus (Enthusiast):** Does not understand "Ask" terminology per the PRD. A card labelled "Lowest ask: £45,700" may mean nothing to him. Further, "Expected value: From £X" on another card with no ask creates an inconsistency that needs explanation.

**Recommendation:**
> - Consider user-testing the label "Lowest ask" vs. simpler alternatives like "Starting from" or "Buy from £X" for the retail-oriented audience
> - Add a small info tooltip (ℹ️) next to the price label on cards to explain: *"This is the lowest price at which a cask owner is willing to sell"*
> - Ensure "Expected value" is clearly defined somewhere — consider "Estimated value" or "Guide price" as clearer alternatives

---

### 4.10 ⚠️ No Personalisation Beyond "Recently Viewed"

**Issue:** The only personalised section is "Recently Viewed." There are no recommendations based on:
- Portfolio composition (what they already own)
- Watched/favourited casks
- Distillery preferences

**Impact:**
- **Catherine (Active Trader):** Who *"checks portfolio and open orders regularly"* would benefit from a "Related to your portfolio" section showing casks from the same distilleries she owns.
- **Marcus (Enthusiast):** Would benefit from a "Recommended for you" section after his first browse session.

**Recommendation:**
> This is a post-MVP consideration, but flag for the roadmap. Add a "Recommended for You" or "From Distilleries You Follow" section as the personalisation strategy matures.

---

## 5. Persona Alignment Analysis

### Marcus — "The Enthusiast Investor" (Retail)

| PRD Need | Homepage Support | Verdict |
|----------|-----------------|---------|
| Clear, jargon-light explanations of trading mechanics | ❌ No explanation of Bid/Ask model anywhere on page | **Gap** |
| Confident, trustworthy visual design | ✅ Dark/gold aesthetic is premium and trust-inducing | **Met** |
| Transparent pricing and fee information | ⚠️ Single price shown with no context (no fees, no comparisons) | **Partial** |
| Browse extensively before committing | ✅ Four curated sections enable extensive browsing | **Met** |
| Understand value and provenance | ⚠️ Distillery section exists but cards lack provenance detail | **Partial** |

**Overall for Marcus:** The homepage *looks* trustworthy but doesn't *explain* anything. Marcus can browse, but will be confused by pricing labels and lack of context on how to act. **Risk of early drop-off due to comprehension gap.**

---

### Catherine — "The Active Trader" (High-Net-Worth)

| PRD Need | Homepage Support | Verdict |
|----------|-----------------|---------|
| Dense, data-rich content | ❌ Cards show minimal data — no bid price, no spread, no trend | **Gap** |
| Fast, minimal-click transaction flows | ⚠️ No shortcut actions from homepage (no quick bid/buy CTA) | **Partial** |
| Market data and order book depth | ❌ No market overview, no aggregate trading data | **Gap** |
| Monitor order status efficiently | ❌ No portfolio summary or order status widget on homepage | **Gap** |

**Overall for Catherine:** The homepage is heavily skewed toward browse/discovery and provides almost no investment decision-support data. Catherine will skip the homepage entirely and go straight to specific cask detail pages — meaning the homepage adds minimal value for this high-value persona. **Missed opportunity to surface actionable market intelligence.**

---

## 6. Competitor Benchmark Comparison

### vs. StockX (Bid/Ask Marketplace)

| Element | StockX | Cask Exchange | Gap |
|---------|--------|---------------|-----|
| **Search** | Prominent, centred search bar | Not present | 🔴 Critical |
| **Card pricing** | Ask price + Last Sale + % Change | Single price only | 🔴 Significant |
| **Market activity signals** | Number of asks, last sale data | None | 🟡 Moderate |
| **Navigation clarity** | Simple (Browse, Sell, About) | Potentially confusing (Marketplace vs. Buying vs. Selling) | 🟡 Moderate |
| **Hero content** | Trending items with live data | Generic promotional banner | 🟡 Moderate |
| **How It Works** | Footer + onboarding flow | Not present | 🔴 Significant |
| **Visual tone** | Modern, clean, minimal | Warm, premium, whisky-themed | ✅ Intentionally different — appropriate |

### vs. OpenSea (Digital Asset Marketplace)

| Element | OpenSea | Cask Exchange | Gap |
|---------|---------|---------------|-----|
| **Volume/activity metrics** | Prominently shown per collection | Not present | 🔴 Significant |
| **Section visual variety** | Mix of banners, grids, carousels | All carousels, same card style | 🟡 Moderate |
| **Category discovery** | Category pills, trending filters | Distillery carousel only | 🟡 Moderate |
| **Personalisation** | Watchlist, recommendations | Recently Viewed only | 🟡 Moderate |

---

## 7. Recommendations Summary

Prioritised by impact and implementation effort:

### High Priority (Address Before Launch)

| # | Recommendation | Reason | Effort |
|---|----------------|--------|--------|
| 1 | **Add search bar to navigation** | Critical marketplace pattern; blocks Catherine's primary workflow | Low |
| 2 | **Add Bid price and/or trend indicator to cask cards** | Without this, cards don't communicate the platform's core bid/ask model | Medium |
| 3 | **Replace or enhance hero banner** with value proposition + market data | Current hero wastes prime viewport on a generic message | Medium |
| 4 | **Add "How It Works" section** | Marcus cannot understand the platform without explicit education | Low–Medium |

### Medium Priority (Strong UX Improvement)

| # | Recommendation | Reason | Effort |
|---|----------------|--------|--------|
| 5 | **Elevate trust signals** (slim trust bar near top of page) | Trust is buried; Marcus needs it before he browses | Low |
| 6 | **Add market overview stats** (total casks, volume, distilleries) | Communicates an active, liquid marketplace | Low–Medium |
| 7 | **Differentiate carousel sections visually** | Reduce repetition fatigue and improve section scannability | Medium |
| 8 | **Clarify "Buying" and "Selling" navigation labels** | Current labels create ambiguity about their purpose | Low |

### Lower Priority (Post-MVP / Iterative)

| # | Recommendation | Reason | Effort |
|---|----------------|--------|--------|
| 9 | **Revisit price label terminology** ("Lowest ask" → consider user testing) | May confuse non-trading users; needs validation | Low |
| 10 | **Add personalised recommendations** beyond Recently Viewed | Increases engagement for returning users | High |
| 11 | **Add portfolio summary widget** for logged-in users on homepage | Gives Catherine a reason to visit the homepage | Medium |

---

## Appendix: Design Tone Assessment

### Stakeholder Directive
> *"The client doesn't want the platform to be too 'techy' or modern. The product is barrels — it needs to be friendly to our TA who are classic and old school, but also content-rich."*

### Assessment
The current design **successfully avoids a 'techy' look** — the dark/amber palette, barrel imagery, and editorial-style distillery cards create a classic, premium feel that is appropriate for the product category. However, the design **fails the 'content-rich' requirement.** The homepage is visually appealing but information-sparse:
- No market data
- No bid/ask context on cards
- No educational content
- No activity signals

**The challenge ahead:** Increase information density (market data, pricing context, trading education) without sacrificing the warm, classic aesthetic. The model here should be a **premium financial publication** (think: Financial Times, Wealth Management portals) rather than a fintech dashboard — data-rich but editorially presented.

---

*End of Audit Report — v1.0*
