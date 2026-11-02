# CE Docs

A documentation repository for the Cask Exchange platform, covering product requirements, UX specifications, data models, business rules, and API requirements for marketplace and trading features.

This repo is intentionally documentation-first: it contains working specs, feature briefs, design assumptions, and reusable templates for product and engineering teams rather than application code.

## Overview

Cask Exchange is a whisky and premium spirits cask marketplace where users can browse casks, compare market data, place bids/asks, complete transactions, and manage their portfolio. The documents in this repository define the product behavior, information architecture, and business logic needed to shape the platform experience.

The material in this repo is organized around:

- Product requirements and MVP scope
- UX and frontend specifications
- Backend / API requirements and business rules
- Data model definitions and examples
- Reusable document templates for consistent planning

## Repository structure

```text
ce-doc/
├── CX/
│   ├── docs/
│   │   ├── data-models/
│   │   │   ├── parent-child-structure.md
│   │   │   ├── cask-variant-structure.json
│   │   │   ├── checkout-session-structure.json
│   │   │   ├── schema-change-requests.md
│   │   │   └── ...
│   │   ├── browse-by-category/
│   │   │   ├── ui-frontend-specs.md
│   │   │   └── api-requirements.md
│   │   ├── cask-filter/
│   │   │   ├── ui-frontend-specs.md
│   │   │   └── api-requirements.md
│   │   ├── curated-list-home/
│   │   │   ├── ui-frontend-specs.md
│   │   │   └── api-requirements.md
│   │   ├── explore-casks-home/
│   │   │   ├── ui-frontend-specs.md
│   │   │   └── api-requirements.md
│   │   ├── marketplace/
│   │   │   ├── ui-frontend-specs.md
│   │   │   ├── api-requirements.md
│   │   │   └── ...
│   │   ├── notifications/
│   │   │   ├── ui-frontend-specs.md
│   │   │   ├── api-requirements.md
│   │   │   └── api-documentation.md
│   │   ├── top-distilleries-home/
│   │   │   ├── ui-frontend-specs.md
│   │   │   ├── api-requirements.md
│   │   │   └── user-story-*.md
│   │   ├── top-cask/
│   │   │   ├── ui-frontend-specs.md
│   │   │   └── api-requirements.md
│   │   ├── project-summary.md
│   │   ├── prd-mvp.md
│   │   ├── api-reference.md
│   │   ├── ux-audit-homepage.md
│   │   ├── order-matching-update-specs.md
│   │   ├── payout-trigger-webhook-specs.md
│   │   └── ...
│   ├── templates/
│   │   ├── document-1-ui-frontend-specs.md
│   │   └── document-2-api-requirements.md
│   └── README.md (if present in nested project folder)
└── README.md
