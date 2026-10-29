# BACKEND LOGIC & FLOW SPECS

Notification System

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation of backend specifications for preferences and notification lifecycle | Sophia | 12/06/2026 |
| 0.2 | Update notification categories to TRANSACTION, MARKETPLACE, ACCOUNT, SYSTEM and add complete list of notification events | Antigravity | 02/07/2026 |
| 0.3 | Update notification event list, definitions, triggers, recipients, titles, and message templates | Antigravity | 03/07/2026 |
| 0.4 | Add UI-to-Backend preferences mapping, TRANSACTION/ACCOUNT bypass rules, and Phase 2 placeholders | Antigravity | 03/07/2026 |
| 0.5 | Update threshold evaluation triggers, optional thresholdPercent rule, and split Buyer/Seller notifications | Antigravity | 03/07/2026 |
| 0.6 | Document forward-looking thresholds rule | Antigravity | 03/07/2026 |

---

## I. Introduction

### 1. Purpose
This specification documents the backend business logic, validation rules, data structures, and use cases for the Cask Exchange Notification system. It covers the notification preference settings for users and the generation, lifecycle, and reading of notifications within the Notification Center.

---

### 2. Definitions

| Term | Definition |
|------|------------|
| Event Type | A specific system or user-triggered occurrence that can prompt a notification (e.g., `New Highest Offer`, `Payment Due`). |
| In-app Notification | A notification rendered inside the web application via the Notification Center bell. |
| Email Notification | An asynchronous email dispatch triggered by a system event, subject to preference settings. |
| Preference Setting | A configuration record stored per user and event type specifying whether in-app and/or email notifications are enabled. |
| Category | A logical grouping of notifications: `TRANSACTION`, `MARKETPLACE`, `ACCOUNT`, or `SYSTEM`. |
| Threshold Percent | A user-defined percentage used to filter out low-priority bids or listing notifications. |
| Read State | A boolean flag indicating whether the user has seen/interacted with the notification. |
| Item ID | ID of the related object (e.g. checkoutSessionId) used by the frontend to redirect or take action. |

---

### 3. Business Rules Summary

| Rule Code | Description | Condition |
|-----------|-------------|-----------|
| **NF-BR-1** | **Preference Channels** | Each event type supports two independent toggles: `inAppEnabled` and `emailEnabled`. |
| **NF-BR-2** | **Threshold Eligibility** | Only the event types `Offer (Bid) near your listing` (Selling) and `Listing near your offer (bid)` (Buying) support the `thresholdPercent` field. |
| **NF-BR-3** | **Threshold Validation** | `thresholdPercent` must be an integer in the range 1 to 100 (inclusive). It is optional (can be null/unvalidated) if both `inAppEnabled` and `emailEnabled` are set to `false`. |
| **NF-BR-4** | **Threshold Evaluation (Selling - Offer (Bid) near your listing)** | Evaluated per (userId, caskId) pair — trigger evaluation runs when a listing or bid is created or has its price updated. Send alert only if the incoming bid price on a given cask meets the seller's threshold percentage relative to their listing price for that same cask (see Section II.3 for formula). |
| **NF-BR-5** | **Threshold Evaluation (Buying - Listing near your offer (bid))** | Evaluated per (userId, caskId) pair — trigger evaluation runs when a listing or bid is created or has its price updated. Send alert only if the new listing price falls within the buyer's threshold percentage of their active bid price (see Section II.3 for formula). |
| **NF-BR-6** | **Category Mapping** | Every notification generated must be mapped to one of the standard categories: `TRANSACTION`, `MARKETPLACE`, `ACCOUNT`, or `SYSTEM`. |
| **NF-BR-7** | **Redirection Data** | The notification response contains type and an optional itemId (e.g., checkoutSessionId for payments, caskId for cask listings). All redirection logic — including the target path and CTA label — is resolved entirely on the FE side via an internal mapping table keyed by type. BE does not generate, store, or validate any redirect URL. |
| **NF-BR-8** | **No Auto-Expiry** | Notifications are preserved in the system and do not expire or get auto-deleted upon reading. |
| **NF-BR-9** | **Read State is Final** | Once a notification's `isRead` is set to `true`, it cannot be reverted to `false`. There is no "mark as unread" operation. |
| **NF-BR-10** | **Preference Bypass** | Events belonging to categories `TRANSACTION` and `ACCOUNT` bypass preference checks entirely. They have no toggle or threshold configuration and are always dispatched. |
| **NF-BR-11** | **Placeholder Preferences** | The eventTypes `MARKETPLACE_ANNOUNCEMENTS_COMMUNICATION`, `PROMOTIONS_OFFERS_COMMUNICATION`, and `ALLOW_PERSONALIZED_ADS_ADVERTISING` must have records persisted in the preference database in Phase 1 to allow UI settings state retention, despite having no live dispatch logic. |
| **NF-BR-12** | **Forward-Looking Preferences** | Changing `thresholdPercent` or re-enabling a preference toggle is forward-looking only. It does not retroactively trigger re-evaluation of existing active listings/bids against past events. |

---

## II. Requirements

### 1. Notification Preferences

#### 1.1. Overview
Allows users to configure which event types they want to receive notifications for, and via which delivery channels. For threshold-supported events, users can fine-tune the noise level by adjusting the threshold percentage.

##### Preference UI Grouping & Backend Dispatch Behavior
The Notification Settings UI displays toggleable preference controls grouped by UI Group and Sub-group. However, backend dispatch logic handles events based on their Backend Category. The following table specifies the mapping and behavior:

| UI Group | Sub-group | Backend Category | Has Live Event? | Toggle / Preference Behavior |
|---|---|---|---|---|
| **Buying** | Listing near your offer | `MARKETPLACE` (Listing near your offer (bid)) | Yes | User toggleable. Supports threshold. |
| **Buying** | Floor price updated | `MARKETPLACE` (Floor price updated) | Yes | User toggleable. |
| **Selling** | Offer near your listing | `MARKETPLACE` (Offer (bid) near your listing) | Yes | User toggleable. Supports threshold. |
| **Selling** | Top offer updated | `MARKETPLACE` (Top offer on your listing) | Yes | User toggleable. |
| **Cask Exchange Communication** | Product update | `SYSTEM` | Yes | User toggleable. |
| **Cask Exchange Communication** | Marketplace announcements | — | No (Phase 2) | User toggleable. Persists in database via placeholder eventType (`MARKETPLACE_ANNOUNCEMENTS_COMMUNICATION`), but has no live dispatch logic. |
| **Cask Exchange Communication** | Promotion & offer | — | No (Phase 2) | User toggleable. Persists in database via placeholder eventType (`PROMOTIONS_OFFERS_COMMUNICATION`), but has no live dispatch logic. |
| **Targeted advertising** | — | — | No (Phase 2) | User toggleable. Persists in database via placeholder eventType (`ALLOW_PERSONALIZED_ADS_ADVERTISING`), but has no live dispatch logic. |
| *(Not shown in Settings UI)* | — | `TRANSACTION` & `ACCOUNT` | Always sent | **Always dispatched.** These categories bypass the preference check entirely (no toggle, no threshold, always dispatched regardless of user settings). |

##### Placeholder eventTypes for Phase 2
To ensure settings toggles persist correctly across devices and sessions via `UC-1` and `UC-2` before the dispatch logic is implemented, the backend must define and store preferences for the following placeholder eventTypes in Phase 1:
- `MARKETPLACE_ANNOUNCEMENTS_COMMUNICATION` (Marketplace announcements)
- `PROMOTIONS_OFFERS_COMMUNICATION` (Promotion & offer)
- `ALLOW_PERSONALIZED_ADS_ADVERTISING` (Allow Personalized Ads / Targeted advertising)

#### 1.2. Data Description

##### Input Preferences Data
| No. | Field | Data Type | Description |Default Value |
|-----|-------|-----------|-------------|-----|
| 1 | `eventType` | enum | Unique code representing the event type (e.g. `LISTING_NEAR_OFFER_BUYING`). |N/A |
| 2 | `inAppEnabled` | boolean | Toggle indicating if in-app alerts are enabled. |True |
| 3 | `emailEnabled` | boolean | Toggle indicating if email alerts are enabled. |True |
| 4 | `thresholdPercent` | integer / null | Threshold percentage (1-100). Only supported for `Offer (Bid) near your listing` (Selling - `OFFER_NEAR_LISTING_SELLING`) and `Listing near your offer (bid)` (Buying - `LISTING_NEAR_OFFER_BUYING`) event types. Mandatory if inAppEnabled = true OR emailEnabled = true. | For `Offer (Bid) near your listing` (Selling): 80<br>For `Listing near your offer (bid)` (Buying): 20 |


#### 1.3. Use Cases

##### UC-1. Fetch Notification Preferences
| Item | Description |
|------|-------------|
| **Objective** | Retrieve the list of all event types with their current channel toggles and threshold settings for the authenticated user. |
| **Actor** | Authenticated User |
| **Trigger** | User navigates to Notification Settings screen. |
| **Pre-condition(s)** | User session is valid and authenticated. |
| **Post-condition(s)** | Returns the user's notification preferences grouped by sections (Buying, Selling, Cask Exchange Communication, Targeted advertising). |

**Happy Path**:
| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | User has existing records in database | Returns complete list of preferences. |
| 2 | User opens settings for the first time | System auto-generates default settings for all known event types in the database and returns them (see default notes below). |

> [!NOTE]
> **Default configurations for first-time users:**
> - `inAppEnabled`: `true` 
> - `emailEnabled`: `true`
> - `thresholdPercent` for `Offer (Bid) near your listing` (`OFFER_NEAR_LISTING_SELLING`): `80`
> - `thresholdPercent` for `Listing near your offer (bid)` (`LISTING_NEAR_OFFER_BUYING`): `20`
> - All other event types (including phase-2 placeholders) do not support `thresholdPercent` (value is always `null`). The threshold input field is not applicable for these event types and should not be rendered or submitted by the client, per **NF-BR-2**.

**Negative Path**:
| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | User session is invalid | Request rejected | 401 — "Unauthorized" |

---

##### UC-2. Update Preference Setting
| Item | Description |
|------|-------------|
| **Objective** | Update the configuration (toggles and/or threshold) for a single event type. |
| **Actor** | Authenticated User |
| **Trigger** | User changes a toggle state or updates the threshold value. |
| **Pre-condition(s)** | User session is valid and authenticated. |
| **Post-condition(s)** | Database preference record updated. |

**Activity Flow & Business Rules**:
| Step | Action | BR Code | Details |
|------|--------|---------|---------|
| 1 | Validate input parameters | NF-BR-3 | Validate toggles are boolean. If event supports threshold, validate `thresholdPercent` is integer (1-100) if either toggle is true. |
| 2 | Check event support | NF-BR-2 | Ensure `thresholdPercent` is not provided/saved for event types that do not support it. |
| 3 | Save preferences | NF-BR-1 | Update or insert the database record mapping the user to the updated event configurations. |

> [!NOTE]
> **Concurrent Update Behavior:**
> Concurrent updates to the same `(userId, eventType)` record follow a last-write-wins strategy. No optimistic locking is applied. This is an intentional design decision for v1.

**Happy Path**:
| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Toggle changed for non-threshold event | Toggles updated successfully, returns updated record. |
| 2 | Threshold and toggles updated for supported event | Record updated with new toggles and `thresholdPercent`. |

**Negative Path**:
| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | `thresholdPercent` is out of range (e.g. 105) | Reject request | 400 — "Threshold must be between 1 and 100" |
| 2 | Attempting to set threshold on unsupported event | Reject request | 400 — "Threshold is not supported for this event type" |

---

### 2. Notification Center

#### 2.1. Overview
Provides users with access to their history of in-app notifications. Allows pagination, filtering by category, filtering by read/unread status, and reading status management.

#### 2.2. Data Description

##### Notification Record Data
| No. | Field | Data Type | Description |
|-----|-------|-----------|-------------|
| 1 | `id` | UUID | Unique identifier of the notification. |
| 2 | `category` | enum | Category: `TRANSACTION`, `MARKETPLACE`, `ACCOUNT`, `SYSTEM` (**NF-BR-6**). |
| 3 | `type` | string | Subtype matching trigger event. |
| 4 | `title` | string | Headline of the notification. |
| 5 | `message` | string | Body text. |
| 6 | `itemId` | string / null | Related resource ID (e.g., checkoutSessionId, caskId) for FE-controlled redirection (**NF-BR-7**). |
| 7 | `isRead` | boolean | Current status of the notification. |
| 8 | `createdAt` | datetime | Generation timestamp. |

#### 2.3. Use Cases

##### UC-3. Retrieve Notifications List
| Item | Description |
|------|-------------|
| **Objective** | List in-app notifications with category/read filters using pagination. |
| **Actor** | Authenticated User |
| **Trigger** | User views the Notification Center. |
| **Pre-condition(s)** | User session is valid. |
| **Post-condition(s)** | Returns a paginated list of notifications matching filters, along with overall unread badge count. |
[!NOTE]
Pagination: Backend is free to choose the pagination mechanism (cursor-based, offset-based, or keyset-based) and default/max page size, as long as it supports infinite-scroll-style consumption on FE. No specific implementation is mandated by this spec.
Filtering: Supports filtering by category and by read/unread status.
---

##### UC-4. Mark Notification as Read
| Item | Description |
|------|-------------|
| **Objective** | Mark a single specific notification as read. |
| **Actor** | Authenticated User |
| **Trigger** | User hovers and clicks checkmark, or clicks the notification card. |
| **Pre-condition(s)** | Notification exists and belongs to the active user. |
| **Post-condition(s)** | Notification status `isRead` transitions to `true`. |

**Happy Path**:
| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | Target notification is unread | Marks as read, updates total unread counts. |
| 2 | Target notification is already read | No-op, succeeds with 200 OK. |

**Negative Path**:
| # | Scenario | Expected Result | Error Code / Message |
|---|----------|----------------|---------------------|
| 1 | Notification ID does not exist | Return not found | 404 — "Notification not found" |
| 2 | Notification belongs to another user | Access denied | 403 — "Forbidden" |

---

##### UC-5. Mark All Notifications as Read
| Item | Description |
|------|-------------|
| **Objective** | Mark all notifications (optionally filtered by category) as read in a single batch. |
| **Actor** | Authenticated User |
| **Trigger** | User clicks "Mark all read" link. |
| **Pre-condition(s)** | User is logged in. |
| **Post-condition(s)** | All matching unread notifications are updated to `isRead = true` (**NF-BR-9**). |

**Happy Path**:
| # | Scenario | Expected Result |
|---|----------|----------------|
| 1 | No category specified | All user's unread notifications are marked read. |
| 2 | Category specified | Only notifications in that category are marked read. |

---

##### UC-6. Fetch Unread Badge Count
| Item | Description |
|------|-------------|
| **Objective** | Retrieve the current total unread badge count to render in the global header bell. |
| **Actor** | Authenticated User / System |
| **Trigger** | Initial app load or header lifecycle polling. |
| **Pre-condition(s)** | User is authenticated. |
| **Post-condition(s)** | Returns count of notifications where `isRead = false`. |

> [!NOTE]
> **Badge Count Delivery Mechanism:**
> The unread badge count is delivered to the client via Server-Sent Events (SSE). The server pushes an updated count whenever a new notification is created or a notification is marked as read. A REST fallback endpoint remains available for initial page load.

### 3. Threshold Evaluation Logic

#### 3.1. Overview
When a system event occurs that supports user threshold filters, the backend must evaluate whether the event satisfies the user's custom `thresholdPercent` setting before dispatching the notification (In-app or Email).

> [!NOTE]
> **Forward-Looking Thresholds:**
> Threshold evaluation is forward-looking only. Changing `thresholdPercent` or re-enabling a preference toggle does not retroactively trigger re-evaluation of existing active listings/bids against past events — it only applies to future create/update events.

#### 3.2. Evaluation Formulas

##### Selling - Offer (Bid) near your listing
Evaluated per (userId, caskId) pair — each active listing a user holds for a given cask is checked independently against incoming bids for that same cask. Trigger evaluation runs only when a bid is newly created or has its price updated. Changes to the user's own listing price do not retroactively trigger evaluation against existing bids — they only apply as the new reference point for future bid events.
Alert is sent if and only if:
`Bid Price(X) >= (thresholdPercent / 100) * Listing Price(X)`
Where X = the specific cask the user's listing is for.
If a user holds active listings on multiple casks (e.g. Cask A and Cask B), each listing is evaluated independently against bids for its respective cask. There is no single global "listing price" per user.

##### Buying - Listing near your offer (bid)
Evaluated per (userId, caskId) pair — each active bid a user holds on a given cask is checked independently against incoming listings for that same cask. Trigger evaluation runs only when a listing is newly created or has its price updated. Changes to the user's own bid price do not retroactively trigger evaluation against existing listings — they only apply as the new reference point for future listing events.
Alert is sent if and only if:
`Bid Price(X) < Listing Price(X) <= Bid Price(X) * (1 + thresholdPercent / 100)`
Where X = the specific cask the user's bid is placed on.
If a user holds active bids on multiple casks (e.g. Cask A and Cask B), each bid is evaluated independently against listings for its respective cask. There is no single global "active bid price" per user.
---

## III. Data Model

### Entity: `NotificationPreference`

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| userId | UUID | ✅ | N/A | Foreign key to User entity |
| eventType | varchar | ✅ | N/A | Unique event code string |
| inAppEnabled | boolean | ✅ | true | Flag for in-app deliveries |
| emailEnabled | boolean | ✅ | true | Flag for email dispatches |
| thresholdPercent | integer | ❌ | null | Filter threshold (1-100) |
| updatedAt | datetime | ✅ | NOW() | Timestamp of last change |

- **Primary Key**: `(userId, eventType)` composite key.

### Entity: `Notification`

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | ✅ | auto-generated | Primary Key |
| userId | UUID | ✅ | N/A | Recipient user ID |
| category | varchar | ✅ | N/A | `TRANSACTION`, `MARKETPLACE`, `ACCOUNT`, `SYSTEM` |
| type | varchar | ✅ | N/A | Event type code string |
| title | varchar | ✅ | N/A | Notification title text |
| message | text | ✅ | N/A | Notification body text |
| itemId | varchar | ❌ | null | Related resource ID for FE-controlled redirection (NF-BR-7) |
| isRead | boolean | ✅ | false | Read state status flag |
| createdAt | datetime | ✅ | NOW() | Record creation timestamp |

> [!NOTE]
> **Data Retention Policy:**
> Notifications are retained indefinitely in v1 per **NF-BR-8**. A data archiving strategy (e.g. soft-delete or cold storage migration for records older than 12 months) should be defined before production scale.

### Status Flow / State Machine

```
   [System Event Triggered]
              │
              ▼
   (Evaluate Preferences)
              │
     ┌────────┴────────┐
     │ (Channel: InApp)│ (Channel: Email)
     ▼                 ▼
[UNREAD]           [Email Dispatched]
     │
     ├─────────────────┐
     │ (User clicks    │ (User clicks
     │  checkmark)     │  card body)
     ▼                 ▼
  [READ] ---------> [READ + Redirect]
```

> [!IMPORTANT]
> **Terminal State Transition:**
> The `READ` state is a terminal state. Once a notification's status transitions to read, it cannot be reverted back to unread (there is no reverse transition or "mark as unread" action), per **NF-BR-9**.

---

## IV. Notifications / Side Effects

The table below lists all supported notification events on the Cask Exchange platform, grouped by their categories (`ACCOUNT`, `TRANSACTION`, `MARKETPLACE`, `SYSTEM`).

| Category | Notification | Definition | Trigger Condition | Recipient | Title | Message Template |
|---|---|---|---|---|---|---|
| **ACCOUNT** | Welcome to CaskExchange | Welcome user after successful account creation | Email verification completed | User | Welcome to CaskExchange | Your account has been successfully verified. You're all set to start exploring the marketplace. |
| **ACCOUNT** | Password changed | Password changed confirmation | Password updated successfully | User | Password Updated | Your password has been changed successfully. If you didn't make this change, please contact support immediately. |
| **ACCOUNT** | Two-Factor Authentication Enabled | User enabled their first 2FA method | User enabled their first 2FA method | User | Two-Factor Authentication Enabled | Your account is now protected with two-factor authentication. |
| **ACCOUNT** | Two-Factor Authentication Disabled | The user no longer has any active 2FA methods | The user no longer has any active 2FA methods | User | Two-Factor Authentication Disabled | Your account is no longer protected by 2FA. |
| **ACCOUNT** | New login detected | New login notification | Login from new device/location | User | New Login Detected | A new login was detected from [browser] on [device] in [location]. If this wasn't you, please secure your account. |
| **TRANSACTION** | Payment created | Payment created | Buyer finishes "Buy Now" flow, OR buyer's offer matches a listing | Buyer | Payment Created | Your payment for [cask name + variant] ([Transaction Amount]) has been created. Waiting for seller confirmation. |
| **TRANSACTION** | Seller confirmation failed | Sale confirmation not completed within the deadline | Sale confirmation not completed within the deadline | Buyer | Confirmation Deadline Missed | Seller confirmation for [cask name + variant] ([Transaction Amount]) wasn't completed within the required timeframe. This transaction has been cancelled. |
| **TRANSACTION** | Deposit payment required | Deposit payment required | Payment status = Awaiting Deposit | Buyer | Deposit Payment Required | Complete your deposit of [deposit amount] for [cask name + variant]. |
| **TRANSACTION** | Deposit received | Deposit received confirmation | Deposit payment successful | Buyer | Deposit Received | We've received your deposit of [deposit amount] for [cask name + variant]. |
| **TRANSACTION** | Agreement ready to sign | Agreement ready to sign | Admin uploads agreement | Buyer | Agreement Ready to Sign | The agreement for [cask name + variant]([Transaction Amount]) is ready for your signature. |
| **TRANSACTION** | Agreement requires update | Agreement requires update or re-signing | Admin requests update | Buyer | Agreement Requires Your Attention | Your agreement for [cask name + variant] ([Transaction Amount]) requires review and updates. |
| **TRANSACTION** | Payment Agreement Expired | Payment agreement not signed within the deadline | Payment agreement not signed within the deadline | Buyer | Payment Agreement Expired | The payment agreement for [cask name + variant] ([Transaction Amount]) wasn't signed within the required timeframe. This transaction has been cancelled. |
| **TRANSACTION** | Remaining payment required | Remaining payment required | Deposit verified & agreement completed | Buyer | Remaining Payment Required | Complete your remaining payment of [remaining amount] for [cask name + variant]. |
| **TRANSACTION** | Remaining payment received | Remaining payment received | Final payment successful | Buyer | Remaining Payment Received | We've received your remaining payment of [remaining amount] for [cask name + variant]. |
| **TRANSACTION** | Ownership transfer completed | Cask ownership transfer completed | Ownership transfer completed | Seller | Ownership Transfer Completed | Ownership of [cask name + variant] ([Transaction Amount]) has been successfully transferred. Your transaction is now complete. |
| **TRANSACTION** | Payout Created | Payout created => agreement ready to sign | Seller finishes "Sell Now" flow, or seller's listing matches an offer | Seller | Transaction Matched | Your payout for [cask name + variant] ([Transaction Amount]) has been created. Please sign the agreement to confirm. |
| **TRANSACTION** | Payout Agreement Finalized | Seller/Admin signs agreement | Seller/Admin signs agreement | Seller | Agreement Finalized | The payout agreement for [cask name + variant] ([Transaction Amount]) has been finalized. |
| **TRANSACTION** | Payout agreement requires update | Payout agreement requires update or re-signing | Admin requests update | Seller | Payout Agreement Requires Update | Your payout agreement for [cask name + variant] ([Transaction Amount]) requires review and updates. |
| **TRANSACTION** | Payout completed | Payout completed | Admin marks payout completed | Seller | Payout Completed | Payoutcompleted for [cask name + variant] ([Transaction Amount]). |
| **TRANSACTION** | Payout Agreement Expired | Payout agreement not signed within the deadline | Payout agreement not signed within the deadline | Seller | Payout Agreement Expired | The payout agreement for [cask name + variant] ([Transaction Amount]) wasn't signed within the required timeframe. This transaction has been cancelled. |
| **TRANSACTION** | Payment failed due to payment expiration | Payment cancelled due to payment expiration | Deposit/remaining payment not completed within the deadline | Buyer | Payment Cancelled | Payment for [cask name + variant] couldn't be completed within the required timeframe. This transaction has been cancelled. |
| **TRANSACTION** | Payout failed due to payment failure | Payout cancelled due to payment failure | Related payment status changed to "Failed" | Seller | Payout Cancelled | Payout for [cask name + variant] cannot be completed due to related payment failure. |
| **MARKETPLACE** | Listing near your offer (bid) | Evaluated per (userId, caskId) pair — each active listing a user holds for a given cask is checked independently against incoming bids for that same cask. | Alert is sent if and only if:<br>Bid Price(X) < Listing Price(X) <= Bid Price(X) * (1 + thresholdPercent / 100)<br>Where X = the specific cask the user's bid is placed on. | Buyer | Listing Matches Your Offer Range | A listing for [cask name + variant] at [listing price] is within your notification threshold. |
| **MARKETPLACE** | Offer (Bid) near your listing | Evaluated per (userId, caskId) pair — each active bid a user holds on a given cask is checked independently against new listings for that same cask. | Alert is sent if and only if:<br>Bid Price(X) >= (thresholdPercent / 100) * Listing Price(X)<br>Where X = the specific cask the user's listing is for. | Seller | Offer Near Your Listing Price | An offer of [offer price] for [cask name + variant] is within your notification threshold. |
| **MARKETPLACE** | Floor Price Updated | Sent when a new lowest Listing (floor price) is published for a cask user currently has an active Offer on. | A newly created or updated Listing becomes the floor price for the cask. | Buyer | Floor Price Updated | The floor price for [cask name + variant] is now [floor price]. |
| **MARKETPLACE** | Top Offer Updated | Sent when a new highest Offer is placed on a cask user currently has listed for sale. | A newly created or updated Offer becomes the top offer for the cask. | Seller | Top Offer Updated | The top offer for [cask name + variant] is now [offer price]. |
| **MARKETPLACE** | Listing expired | Listing expired | Listing reaches expiration date | User | Listing Expired | Your listing of [cask name + variant] at [listing price] has expired. |
| **MARKETPLACE** | Offer expired | Offer expired | Offer reaches expiration date | User | Offer Expired | Your offer of [offer price] for [cask name + variant] has expired. |
| **SYSTEM** | Scheduled maintenance | Scheduled maintenance notification | Admin schedules maintenance | User | Scheduled Maintenance | CaskX will undergo scheduled maintenance. Some services may be temporarily unavailable. |
| **SYSTEM** | Maintenance completed | Maintenance completed | Maintenance finished | User | Maintenance Completed | Scheduled maintenance has been completed. All services are now available. |
| **SYSTEM** | Terms & policies updated | Terms & policies updated | Admin publishes new terms | User | Terms & Policies Updated | We've updated our Terms and Policies. Please review the latest version. |

> [!NOTE]
> **Email Delivery & Thresholding:**
> Email delivery is asynchronous and may be subject to a short delay after the triggering event. If a threshold check fails for a supported event type, no Notification record is created and no email is dispatched.

_End of Document 2_
