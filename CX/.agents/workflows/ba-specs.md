---
description: Generate UI/Frontend Specs (Document 1) and API Requirements (Document 2) from feature descriptions
---

# BA Specs Generation Workflow

When the user provides a feature description, screen spec table, or raw requirement notes, generate two structured documents following the templates in `/templates/`.

## Steps

1. **Understand the Feature**
   - Read the raw input (feature description, screen specs, requirement notes)
   - Identify: screen name, user roles, entry points, related entities

2. **Generate Document 1: UI & Frontend Specs**
   - Follow the template at `templates/document-1-ui-frontend-specs.md`
   - Fill in ALL sections:
     - **Document Version Control table**: list version 0.1 with creation date and author
     - **Screen Overview**: purpose, URL, user roles, entry points
     - **UI Elements Table**: every visible component with type, description, states
     - **Field Validations**: per-field rules (required, format, length, custom)
     - **FE Integration Cases**: for each API call — trigger, loading, success, error states (per error code)
     - **Edge Cases & FE-only Logic**: offline, timeout, resize, permission, empty state scenarios
     - **Navigation & Routing**: where each action leads
     - **Responsive Design Notes**: layout changes per breakpoint
     - **Accessibility Requirements**: WCAG compliance checklist

3. **Generate Document 2: API Requirements (Backend Specs)**
   - Follow the template at `templates/document-2-api-requirements.md`
   - Follow the structure from the reference Google Doc: https://docs.google.com/document/d/1vxRACJZrhgMSd-6qXVbakXbZ7c9LeTcq0l__EH2gBfg
   - Fill in ALL sections:
     - **Document Version Control table**: list version 0.1 with creation date and author
     - **Introduction**: purpose, definitions, business rules
     - **Requirements**: per section — overview (trigger, pre/post conditions), data description, use cases (UC format with BR codes), edge cases
     - **Data Model**: entity fields, types, status flow / state machine
     - **Notifications / Side Effects**: emails, push notifications triggered
     - **Permissions & Access Control**: role-based access matrix. Default to authenticated users only (❌ for Anonymous). Only add Anonymous ✅ if the feature explicitly allows public access — document the rationale

4. **Output**
   - Save Document 1 as: `docs/[feature-name]/ui-frontend-specs.md`
   - Save Document 2 as: `docs/[feature-name]/api-requirements.md`
   - Present a summary to the user with key decisions and open questions

## Reference Document Structure (from Google Docs)

The API Requirements document follows the Cask Exchange documentation pattern:
- **Use Case (UC) Format**: Objective → Actor → Trigger → Pre-conditions → Post-conditions → **Activity Flow & Business Rules** *(only when the UC involves user interaction; omit for fully system-triggered/automated flows)* → Happy/Negative/Edge Cases
- **Business Rules (BR)**: Coded as `[SECTION]-[NUMBER]` (e.g., SA-1, SA-2)
- **Edge Cases**: Trigger → Flow → Side Effects (cancel objects, release orders, send emails)
- **Data Description Tables**: No. → Field → Data Type → Description
- **Status/State Machines**: Clear status transitions with conditions

## Notes
- Write in English unless the original requirement is in Vietnamese
- Use consistent naming conventions matching the Cask Exchange domain
- Cross-reference related documents (e.g., Payment Flow, Sign Agreement)
- Include examples where business logic is complex (e.g., matching, fee calculations)

## Existing API Reference

Before writing API Requirements, **always check** `docs/api-reference.md` for existing endpoints.
Key rules derived from the live API:

| Topic | Rule |
|-------|------|
| **Master Cask** | No dedicated `/api/master-casks` endpoint. Master data comes embedded in variant (`/api/casks`) responses via `master` field. New features targeting master casks need new endpoints. |
| **Region name** | `region` is an **object** (`{ id, name, ... }`), NOT a flat string. Expose as `regionName` (mapped from `region.name`) in new endpoint payloads. |
| **Classification** | Backend has both `classification` (slug, e.g. `single_malt_scotch`) and `classificationLabel` (display, e.g. `Single Malt Scotch`). Always use `classificationLabel` for display fields. Use `GET /api/cask-metadata/classifications` as source of truth. |
| **viewCount** | Tracked at variant level via `POST /api/casks/{id}/view`. Listing supports `sortBy=viewCount`. |
| **Top Distilleries** | `GET /api/distilleries/top-ranked` sorts by cask count, NOT volume. A new endpoint is required for volume-based Top Distilleries features. |
| **Explore Casks** | No existing curated-tab endpoints. All 4 tabs (Top Traded, Most Watched, Strategic Buy, New Listing) require new endpoints with a pre-computed stats table. |
| **Authentication** | Default to 🔒 (authenticated). Only mark Public if the feature explicitly requires anonymous access — document the rationale in Permissions section. |
