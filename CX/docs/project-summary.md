# CaskX Exchange — BA Documentation Workspace Summary

## 1. Project Overview

**Project**: CaskX Exchange — a whisky cask investment marketplace.  
**Role**: Business Analyst assistant generating structured UI/Frontend and Backend specifications for features.  
**Workspace**: `/Users/alice/Documents/cask-exchange`

---

## 2. Directory Structure

```
cask-exchange/
├── templates/                          # Reusable document templates
│   ├── document-1-ui-frontend-specs.md # UI & Frontend Specs template
│   └── document-2-api-requirements.md  # Backend Logic & Flows template
├── docs/
│   ├── data-models/                    # Shared data model references
│   │   ├── parent-child-structure.md   # Master/Variant field definitions
│   │   └── cask-variant-structure.json # Real API response sample (Old Pulteney)
│   ├── curated-list-home/              # Feature: Curated List for Home
│   │   ├── ui-frontend-specs.md
│   │   └── api-requirements.md
│   └── cask-filter/                    # Feature: Cask Filter (Marketplace)
│       ├── ui-frontend-specs.md
│       └── api-requirements.md
└── .agents/workflows/
    └── ba-specs.md                     # Workflow for generating BA docs
```

---

## 3. Core Data Model: Parent / Child (Master / Variant)

### Key Architecture Decision
The platform uses a **Parent / Child** structure for casks:

| Level | Entity | Purpose |
|-------|--------|---------|
| **Parent (Master Cask)** | Groups vintages under one product | Holds shared attributes |
| **Child (Variant)** | A specific vintage/barrel | Holds vintage-specific attributes |

### Master Cask (Parent) Fields
`name`, `distillery` (FK), `caskType` (FK), `classification` (FK), `region` (FK), `image` (URL), `viewCount`

> **IMPORTANT**: Master Cask does **NOT** have a `status` field or `readyToSell`. These belong to the Variant.

### Variant (Child) Fields
`vintageYear`, `distillationDate`, `age`, `expectedMaturityDate`, `ola`, `rla`, `abv`, `estimatedBottleCount`, `bottleVolume`, `referencePriceMin`, `referencePriceMax`, `description`, `tastingNotes`, `imageUrl`, `order`

**Variant-only lifecycle fields**: `readyToSell`, `caskStatus` (`active`/`inactive`), `isListed`, `lowestAsk`, `highestBid`

### Active Master Cask Definition
A master cask is considered "active" if it has **at least one variant** where `caskStatus = 'active'` AND `isListed = true`. There is no master-level status flag.

### Price Aggregation Rules (Master level)
| Field | Aggregation | Description |
|-------|------------|-------------|
| `lowestAsk` | MIN(lowestAsk) across all active variants | Lowest active ask price |
| `referencePriceMin` | MIN(referencePriceMin) across all active variants | Lowest reference price |
| `referencePriceMax` | MAX(referencePriceMax) across all active variants | Highest reference price |

### viewCount
Tracked at the **Master Cask level** (not variant). Incremented when a user visits the master cask detail page. Variant does NOT have its own viewCount in the master object (though the API response currently returns viewCount on the variant, this is the master's value passed through).

---

## 4. Documentation Standards & Templates

### Document 1: UI & Frontend Specs
**Template**: `templates/document-1-ui-frontend-specs.md`  
**Sections**: Screen Overview → UI Elements Table → Field Validations → Cases and Flows → Edge Cases → Navigation & Routing

### Document 2: Backend Logic & Flows Specs  
**Template**: `templates/document-2-api-requirements.md`  
**Key structural decisions**:

1. **Overview (per feature section)**: Brief high-level description of the feature — NOT a use-case-level spec table. Detailed triggers/preconditions belong in each UC.

2. **Activity Flow & Business Rules**: Merged into a single table where each row maps a step to its BR code:
   ```
   | Step | Action | BR Code | Details |
   ```
   BR codes describe the detailed logic for each step (not separate from the flow).

3. **3-Group Analysis per Use Case**:
   - **Happy Path**: Expected successful flows
   - **Negative Path**: Invalid requests, missing permissions, failed preconditions (with Error Code/Message)
   - **Edge Cases**: Boundary conditions, race conditions (optional at UC level)
   - **Feature-level Edge Cases (section 1.4)**: Cross-UC edge cases, system-wide concerns

4. **No specific API endpoints**: Documents describe WHAT the system should do, not HOW the API is structured. No HTTP methods, URLs, or request/response schemas. Developers decide implementation.

---

## 5. Completed Feature Docs

### Feature 1: Curated List for Home
**Location**: `docs/curated-list-home/`  
**Key rules**:
- All cards represent **Master Casks**
- `viewCount` is on Master level
- Navigation: Click card → `/marketplace/{masterCaskId}`

### Feature 2: Cask Filter (Marketplace)
**Location**: `docs/cask-filter/`  
**Key rules**:
- **Filter Level Matrix**: Distillery & Cask Type are Master-level; all others are Variant-level
- **Variant-level filters**: Master cask passes if ≥ 1 active variant satisfies ALL variant-level filters simultaneously
- **Price filter**: Checks ALL price points — every active ask price, every active bid price, `referencePriceMin`, `referencePriceMax`.
- Results are paginated (default 20).

---

## 6. Key Design Decisions Log

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | No `status` field on Master Cask | Active state derived from children: ≥ 1 variant with `caskStatus = 'active'` AND `isListed = true` |
| 2 | `viewCount` on Master level only | Users view the master cask detail page, not individual variants |
| 3 | `readyToSell`, `caskStatus`, `isListed` on Variant only | These are lifecycle/trade properties of physical stock items |
| 4 | Price filter uses ALL bid/ask prices | Ensures casks are discoverable at any active market price point |
| 5 | Shared Price display logic | Card shows "Lowest ask: £X" or "Expected value: From £Y" |
