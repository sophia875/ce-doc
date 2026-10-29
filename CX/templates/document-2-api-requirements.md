# BACKEND LOGIC & FLOW SPECS

[FEATURE_NAME]

  

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation | [AUTHOR] | [DATE] |

---

## I. Introduction

### 1. Purpose
> [Brief description of what this feature/module achieves from a system perspective]

### 2. Definitions

| Term | Definition |
|------|-----------|
| [Term 1] | [Definition] |
| [Term 2] | [Definition] |

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| BR-1 | [Rule description] | [When/if condition] |

---

## II. Requirements

### 1. [Feature Section Name]

#### 1.1. Overview

> [Brief description of the feature / section — what problem it solves, the overall flow idea, and the expected outcome. This is a high-level summary. Detailed triggers, pre-conditions, and post-conditions belong in each Use Case below.]

#### 1.2. Data Description

| No. | Field | Data Type | Description |
|-----|-------|-----------|-------------|
| 1 | [field_name] | string / number / datetime / boolean / enum | [What it represents, format, computation logic] |
| 2 | | | |

#### 1.3. Use Cases

##### UC-1. [Use Case Name]

| Item | Description |
|------|-------------|
| **Objective** | [What this use case achieves] |
| **Actor** | [System / Buyer / Seller / Admin] |
| **Trigger** | [What starts this use case] |
| **Pre-condition(s)** | [Required state before execution] |
| **Post-condition(s)** | [Expected state after execution] |

**Activity Flow & Business Rules**:

> Each step is a short action name. The corresponding BR code describes the detailed logic, validations, transformations, and side effects for that step.

| Step | Action | BR Code | Details |
|------|--------|---------|---------
| 1 | [Step name — e.g., Validate request] | [XX-1] | [Detailed rule for this step — data transformations, validations, conditions, side effects] |
| 2 | [Step name — e.g., Query data] | [XX-2] | [Detailed rule for this step] |
| 3 | [Step name — e.g., Return response] | [XX-3] | [Detailed rule for this step] |

**Happy Path**:

> The expected, successful execution path where all conditions are met and no errors occur.

| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | [e.g., Valid request with all required data] | [e.g., Data is created/returned successfully, status 200] |
| 2 | [e.g., User has correct permissions] | [e.g., Action completes, response includes expected data] |

**Negative Path**:

> Scenarios where the request is invalid, the user lacks permissions, or required preconditions are not met. The system should reject or handle gracefully.

| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | [e.g., Missing required field] | [e.g., Return validation error] | [e.g., 400 — "Field X is required"] |
| 2 | [e.g., User not authenticated] | [e.g., Reject request] | [e.g., 401 — "Unauthorized"] |
| 3 | [e.g., Resource not found] | [e.g., Return not found] | [e.g., 404 — "Cask not found"] |

**Edge Cases** _(optional — at UC level if specific to this use case)_:

> Unusual but valid scenarios, boundary conditions, or race conditions. If edge cases span multiple use cases, move them to the feature-level Edge Cases section (1.4).

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 1 | [e.g., Concurrent modification of same record] | [e.g., Last write wins, return latest state] |

##### UC-2. [Next Use Case]

_(Repeat UC structure: Objective → Activity Flow & BR → Happy Path → Negative Path → Edge Cases)_

#### 1.4. Edge Cases _(feature-level)_

> Use this section for edge cases that span multiple use cases within this feature, or broader system-level edge cases that don't belong to a single UC.

| # | Scenario | Related UC(s) | Expected Behavior | Side Effects |
|---|----------|---------------|-------------------|-------------|
| 1 | [e.g., Data changes between two sequential UC calls] | UC-1, UC-2 | [Steps taken by system] | [Side effects if any] |

> **Guidance: UC-level vs Feature-level**  
> - **UC-level**: Use when edge cases are specific to one use case and don't interact with other flows.  
> - **Feature-level (1.4)**: Use when edge cases involve interactions between multiple UCs, system-wide conditions, or cross-cutting concerns (e.g., caching, concurrency, data consistency).

---

### 2. [Next Feature Section]

_(Repeat the same structure: Overview → Data Description → Use Cases (with Happy/Negative/Edge per UC) → Feature-level Edge Cases)_

---

## III. Data Model

### Entity: [Entity Name]

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | ✅ | auto-generated | Primary key |
| status | enum | ✅ | [default] | [Possible values and transitions] |
| createdAt | datetime | ✅ | NOW() | Record creation timestamp |
| updatedAt | datetime | ✅ | NOW() | Last update timestamp |

### Status Flow / State Machine

```
[Status A] → [Status B] → [Status C]
                ↓
           [Status D (Failed)]
```

---

## IV. Notifications / Side Effects

| # | Trigger Event | Channel | Template | Recipient | Description |
|---|--------------|---------|----------|-----------|-------------|
| 1 | [e.g., Match created] | Email | [template-name.html] | [Seller / Buyer / Admin] | [What it communicates] |
| 2 | | Push | | | |

---

## V. Permissions & Access Control

> **Authentication required by default.** All endpoints are authenticated. Add an `Anonymous` column only when an action explicitly allows unauthenticated access, and document the rationale.

| Use Case / Action | Anonymous | Buyer | Seller | Admin |
|-------------------|-----------|-------|--------|-------|
| [Action Name e.g., View Data] | ❌ | ✅ (own data) | ✅ (own data) | ✅ (all) |
| [Action Name e.g., Create Record] | ❌ | ✅ | ❌ | ✅ |
| [Action Name e.g., Edit Record] | ❌ | ❌ | ❌ | ✅ |
| [Action Name e.g., Delete Record] | ❌ | ❌ | ❌ | ✅ |

---

_End of Document 2_
