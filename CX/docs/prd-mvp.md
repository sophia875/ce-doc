# Product Requirements Document
## Cask Exchange — MVP Platform

**Version:** 1.0  
**Date:** April 2026  
**Author:** Product Team  
**Status:** Draft — Foundation for UX Audit  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Product Objectives](#2-product-objectives)
3. [Target Audience](#3-target-audience)
4. [User Personas](#4-user-personas)
5. [User Stories](#5-user-stories)
6. [Feature Requirements](#6-feature-requirements)
7. [Information Architecture](#7-information-architecture)
8. [Key User Flows](#8-key-user-flows)
9. [Non-Functional Requirements](#9-non-functional-requirements)
10. [Glossary](#10-glossary)

---

## 1. Executive Summary

Cask Exchange is a marketplace platform for buying and selling maturing whisky and premium spirit casks. Users can trade casks as investment assets — similar to a stock exchange but for physical barrels stored at distilleries.

The platform enables two core transaction modes: placing orders at desired prices (Bid/Ask) and executing immediate trades at current market prices (Buy Now / Sell Now). The MVP targets both retail enthusiasts and high-net-worth investors seeking alternative assets with long-term appreciation potential.

This PRD serves as the foundation for the UX audit of the MVP platform. It defines intended user behaviour, feature scope, and success criteria to evaluate whether the current UX supports user goals effectively.

---

## 2. Product Objectives

### 2.1 Business Objectives

- Establish Cask Exchange as a trusted, transparent marketplace for cask trading
- Facilitate efficient price discovery through a bid/ask order book mechanism
- Enable liquidity in an otherwise illiquid alternative asset class
- Build a scalable admin-operated platform with minimal friction for end users

### 2.2 UX Audit Goals (Context for This PRD)

This document provides the intended product experience so the audit can evaluate:

- Whether current UX supports user goals at each stage of the funnel
- Where friction, confusion, or drop-off may occur in key flows (Browse → Buy / Sell → Manage)
- Whether information architecture and nomenclature match user mental models
- Admin-side efficiency and content management usability

---

## 3. Target Audience

Cask Exchange serves two primary audience groups, with both equally important to the platform's liquidity and growth.

### 3.1 Retail Investors & Enthusiasts

Individuals with a personal or financial interest in whisky and premium spirits who view casks as an accessible alternative investment. They may be first-time cask buyers who need education alongside the transaction experience.

**Characteristics:**
- Age range: 28–55
- Moderate to high disposable income
- Familiar with online investing or trading apps (e.g., stocks, crypto)
- Whisky hobbyists or collectors who want deeper market participation
- May require onboarding support to understand Bid/Ask mechanics

### 3.2 High-Net-Worth & Experienced Investors

Sophisticated investors who actively manage alternative asset portfolios and treat casks as a store of value or speculative investment. They are comfortable with order book interfaces and expect detailed data.

**Characteristics:**
- Age range: 35–65
- High net worth; treat casks as part of a diversified portfolio
- Experienced with trading platforms; expect professional-grade tools
- Focus on provenance, distillery reputation, and market data
- Low tolerance for friction or unclear pricing information

---

## 4. User Personas

### Persona 1 — "The Enthusiast Investor" (Retail)

> **Name:** Marcus, 38, Marketing Director  
> **Location:** Ho Chi Minh City, Vietnam  
> **Experience:** Casual whisky collector; recently started investing in stocks  

**Goals:**
- Buy a cask from a reputable distillery as a long-term investment
- Understand the value and provenance of what he's buying
- Eventually sell at a profit when the time is right

**Pain Points:**
- Unfamiliar with Bid/Ask order mechanics — needs clear guidance
- Unsure how to verify cask quality or authenticity
- Concerned about payment security and fund handling

**Behaviours:**
- Browses the listing page extensively before committing
- Reads cask detail pages carefully, especially distillery info and price history
- Likely to use Buy Now rather than placing a Bid on first purchase
- Returns to the platform periodically to check cask value

**UX Needs:**
- Clear, jargon-light explanations of trading mechanics
- Confident, trustworthy visual design
- Transparent pricing and fee information
- Simple portfolio view showing owned casks and current estimated value

---

### Persona 2 — "The Active Trader" (High-Net-Worth)

> **Name:** Catherine, 52, Private Equity Partner  
> **Location:** Singapore  
> **Experience:** Experienced investor; owns multiple casks across distilleries  

**Goals:**
- Actively manage a portfolio of casks — buying low, selling at peak
- Place precise Bid and Ask orders to optimise entry and exit prices
- Monitor order status and market trends efficiently

**Pain Points:**
- Slow or unclear order management UI wastes time
- Wants richer market data (price history, order depth) to make decisions
- Finds it frustrating when transaction confirmation flows are unnecessarily verbose

**Behaviours:**
- Uses both Bid/Ask and Buy Now/Sell Now depending on market conditions
- Checks portfolio and open orders regularly
- Focuses heavily on price charts and order book depth per cask
- May hold multiple open Bids and Asks simultaneously

**UX Needs:**
- Dense, data-rich cask detail and order management pages
- Fast, minimal-click transaction flows
- Clear order status states (open, matched, completed, cancelled)
- Reliable notifications for order matches

---

### Persona 3 — "The Platform Admin" (Internal Operations)

> **Name:** Linh, 29, Operations Manager  
> **Location:** Internal team  

**Goals:**
- List new casks accurately and efficiently
- Review and process transactions flagged for review
- Maintain data integrity of casks, contracts, and distillery records

**Pain Points:**
- Manual data entry errors when listing new casks
- Unclear audit trail when disputes arise on a transaction
- Difficulty tracking which casks have pending documentation vs. approved

**Behaviours:**
- Logs in daily to check pending transactions and new listing tasks
- Manages batch cask uploads during distillery onboarding
- Investigates flagged trades and coordinates with legal/operations teams

**UX Needs:**
- Structured cask creation form with validation
- Clear transaction status pipeline with filter and search
- Document/contract management linked to each cask
- Role-appropriate access controls

---

## 5. User Stories

### Browse & Discovery

| ID | As a… | I want to… | So that… |
|----|-------|------------|----------|
| US-01 | User | Browse all available casks with filters | I can narrow down options matching my preferences |
| US-02 | User | View detailed information on a specific cask | I can assess its quality, provenance, and value before buying |
| US-03 | User | View distillery profiles | I understand the producer and their reputation |
| US-04 | User | See market data (bids, asks, sales...) for a cask | I can gauge market trends and fair value |

### Buy Flow

| ID | As a… | I want to… | So that… |
|----|-------|------------|----------|
| US-05 | Buyer | Place a Bid at my desired price | I can potentially buy below the current Ask price |
| US-06 | Buyer | Buy Now at the lowest available Ask price | I can complete a purchase immediately without waiting |
| US-07 | Buyer | Confirm order details before finalising | I can review my purchase before committing |
| US-08 | Buyer | Complete payment securely | The transaction is confirmed and the cask is transferred to me |

### Sell Flow

| ID | As a… | I want to… | So that… |
|----|-------|------------|----------|
| US-09 | Seller | Place an Ask at my desired price | I can potentially sell above the current Bid price |
| US-10 | Seller | Sell Now at the highest available Bid price | I can exit a position immediately |
| US-11 | Seller | Only sell casks I own | Accidental or fraudulent listings are prevented |
| US-12 | Seller | Confirm sale details before finalising | I can review the transaction before committing |

### Order & Portfolio Management

| ID | As a… | I want to… | So that… |
|----|-------|------------|----------|
| US-13 | User | View all my open Bids | I know which buy orders are still pending |
| US-14 | User | View all my open Asks | I know which sell orders are still pending |
| US-15 | User | View my transaction history | I have a record of all completed trades |
| US-16 | User | View casks I currently own | I can manage my portfolio and decide when to sell |

### Account & Settings

| ID | As a… | I want to… | So that… |
|----|-------|------------|----------|
| US-17 | User | Update my profile information | My account details stay current |
| US-18 | User | Change my password | My account remains secure |

### Admin

| ID | As an… | I want to… | So that… |
|----|--------|------------|----------|
| US-19 | Admin | Create and publish new cask listings | Inventory is available for users to trade |
| US-20 | Admin | Edit existing cask details | Inaccurate information can be corrected |
| US-21 | Admin | Monitor all transactions | Irregularities can be flagged and investigated |
| US-22 | Admin | Manage contracts linked to casks | Legal documentation stays organised and accessible |

---

## 6. Feature Requirements
**The infomation below is for reference only, you can based on the user personas and behavior to suggest more approriate features and details. Or the features could be changed during working process

### 6.1 Cask Listing & Discovery

**Cask Listing Page**
- Grid or list view of all casks available on the exchange
- Filter options: distillery, year of distillation, price range, cask size, cask type, abv, rla... (could be added more)
- Sort options: price (asc/desc), newest listings, popular (could be added more)

**Cask Detail Page**
- Full cask specification
- Market data tables: current lowest Ask, current highest Bid, last trade prices
- Price history chart (at minimum: last trade prices over time)
- Distillery information summary with link to distillery profile
- Supporting documents (e.g., certificate of ownership, bond certificate)
- CTA: Buy Now / Place Bid

**Distillery Pages**
- Distillery listing: overview cards for all distilleries on platform
- Distillery detail: background, region, spirit style, casks currently available

### 6.2 Buy Flow

**Place Bid**
- Input: price per unit, quantity (if applicable)
- System validates input is below current lowest Ask
- Confirmation screen showing order summary
- Order submitted → stored as open Bid in user's account
- Auto-matched when a matching Ask is placed or an existing Ask meets the Bid price

**Buy Now**
- Triggered from Cask Detail page
- Automatically selects the lowest current Ask price
- Confirmation screen with price, fees (if applicable), and total
- On confirm: payment initiated → order matched → cask ownership transferred

### 6.3 Sell Flow

**Place Ask**
- Available only for casks user owns
- Input: price at which user wants to sell
- Confirmation screen showing order summary
- Order submitted → stored as open Ask
- Auto-matched when a matching Bid meets the Ask price

**Sell Now**
- Triggered from user's portfolio / owned casks view
- Automatically matches to the highest current Bid
- Confirmation screen with price, fees (if applicable), and net proceeds
- On confirm: order matched → ownership transferred → seller receives proceeds

### 6.4 Order Management

- **Manage Bids:** Table of all open buy orders — cask, bid price, date placed, status. Option to cancel open Bids.
- **Manage Asks:** Table of all open sell orders — cask, ask price, date placed, status. Option to cancel open Asks.
- **Transaction History:** Completed trades with date, cask, buy/sell side, price, and status.

### 6.5 Portfolio

- List of all casks currently owned by the user
- For each cask: name, distillery, date acquired, purchase price, current market estimate (if available)
- CTA: Sell Now or Place Ask directly from portfolio view

### 6.6 Account & Settings

- Profile: name, email, contact details — editable
- Security: password change, optional 2FA (post-MVP consideration)
- Notification preferences (post-MVP consideration)

### 6.7 Admin Panel

- **Cask Management:** Create, edit, publish, and unpublish cask listings. Required fields: all cask metadata, distillery link, warehouse details.
- **Transaction Monitoring:** View all platform transactions with status filters (pending, matched, completed, disputed). Ability to flag or escalate transactions.
- **Contract Management:** Upload and link legal documents (ownership certificates, bond documents) to individual cask records.
- **Distillery Management:** Create and edit distillery profiles.

---

## 7. Information Architecture

```
Cask Exchange
├── Login/Register/Forgot Password
├
├── Browse - User
│   ├── Cask Listing (with filters & sorts)
│   │   └── Cask Detail
│   ├── Distillery Listing
│       └── Distillery Detail  
│
├── Authenticated — User
│   ├── Dashboard / Home
│   ├── My Portfolio (owned casks)
│   │   └── Sell Now / Place Ask (from cask)
│   ├── My Orders
│   │   ├── Open Bids
│   │   └── Open Asks
│   ├── Transaction History
│   └── Account Settings
│       ├── Profile (integrated with Stripe KYC)
│       └── Security
│       └── Notifications
│
└── Authenticated — Admin
    ├── Cask Management
    │   ├── All Casks
    │   └── Create / Edit / Delete Cask
    ├── Transaction Monitoring
    ├── Contract Management
    └── Distillery Management
```

## 8. Glossary

| Term | Definition |
|------|------------|
| **Cask** | A barrel of maturing spirits — the primary tradeable asset on the platform |
| **Distillery** | The production facility where casks are stored and matured |
| **Bid** | A standing buy order — the buyer proposes a price and waits for a match |
| **Ask** | A standing sell order — the seller proposes a price and waits for a match |
| **Buy Now** | Immediate purchase at the lowest currently available Ask price |
| **Sell Now** | Immediate sale at the highest currently available Bid price |
| **Order Book** | The collection of all open Bids and Asks for a given cask |
| **Matched Order** | A Bid and Ask that have met on price — triggering a completed transaction |
| **Portfolio** | The user's collection of casks they currently own |
| **Admin** | Internal operations team member with elevated platform permissions |

---

*Document version 1.0 — to be reviewed and updated as the UX audit findings are incorporated.*
