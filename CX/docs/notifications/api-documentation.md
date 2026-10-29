# API Documentation: Notification System

This document provides complete specification for the Cask Exchange Notification system. It covers two primary modules: **Notification Preferences** (managing channels and threshold filters) and the **Notification Center** (retrieving and managing received alerts).

---

## 1. Overview

The Notification system on Cask Exchange provides an automated way for buyers and sellers to stay informed about marketplace events, trading status, account security, and system announcements. 
All interactions with these endpoints require **Bearer Token Authentication (JWT)** and default to secure `https`.

---

## 2. Notification Preferences

Users configure their notification options on the Settings page. Toggles are provided for In-app (push/badge) and Email delivery channels. Certain price-sensitive events support a filter threshold to prevent inbox flooding.

### 2.1 Supported Event Types & Preference Grouping

The Notification Settings UI displays toggleable preference controls grouped by UI Group / Section. However, backend dispatch logic handles events based on Backend Categories.

#### UI Group: Buying (Category: `MARKETPLACE` buyer-side)
- **Listing Near Your Offer (`LISTING_NEAR_OFFER_BUYING`)**: Informs buyer of listings near their bid. Supports threshold evaluation.
- **Floor Price Updated (`FLOOR_PRICE_UPDATED`)**: Informs buyer that the variant's floor price has changed.
- **Offer Expired (`OFFER_EXPIRED`)**: Informs buyer that their offer expired.

#### UI Group: Selling (Category: `MARKETPLACE` seller-side)
- **Offer Near Your Listing (`OFFER_NEAR_LISTING_SELLING`)**: Informs seller of offers near their asking price. Supports threshold evaluation.
- **Top Offer Updated (`TOP_OFFER_UPDATED`)**: Informs seller of a new top bid on their listed variant.
- **Listing Expired (`LISTING_EXPIRED`)**: Informs seller that their listing expired.

#### UI Group: Cask Exchange Communication
- **Product Updates (`PRODUCT_UPDATES_COMMUNICATION` / Category: `SYSTEM`)**: Notifies of platform product changes and releases.
- **Marketplace Announcements (`MARKETPLACE_ANNOUNCEMENTS_COMMUNICATION` / Category: None)**: Phase 2 placeholder eventType (toggle state persists in DB via GET/PATCH preferences APIs, but no live events are sent in Phase 1).
- **Promotion & Offers (`PROMOTIONS_OFFERS_COMMUNICATION` / Category: None)**: Phase 2 placeholder eventType (toggle state persists in DB via GET/PATCH preferences APIs, but no live events are sent in Phase 1).

#### UI Group: Targeted advertising
- **Allow Personalized Ads (`ALLOW_PERSONALIZED_ADS_ADVERTISING` / Category: None)**: Phase 2 placeholder eventType (toggle state persists in DB via GET/PATCH preferences APIs, but no live events are sent in Phase 1).

#### Not Configurable (Bypass Preference Checks entirely)
The categories `TRANSACTION` and `ACCOUNT` bypass preference settings entirely. They do not appear in the Settings UI, do not support thresholds, and are always generated and dispatched regardless of user settings.

##### Category: `ACCOUNT`
- **Welcome to CaskExchange (`WELCOME_TO_CASKEXCHANGE`)**: Welcomes user after successful account creation.
- **Password Changed (`PASSWORD_CHANGED`)**: Confirms password update.
- **Two-Factor Authentication Enabled (`TWO_FACTOR_AUTHENTICATION_ENABLED`)**: Confirms 2FA setup.
- **Two-Factor Authentication Disabled (`TWO_FACTOR_AUTHENTICATION_DISABLED`)**: Confirms 2FA removal.
- **New Login Detected (`NEW_LOGIN_DETECTED`)**: Warns of unrecognized login device/location.

##### Category: `TRANSACTION`

###### Transaction Events - Buyer-Facing
- **Payment Created (`PAYMENT_CREATED`)**: Notifies buyer that payment setup is complete.
- **Seller Confirmation Failed (`SELLER_CONFIRMATION_FAILED`)**: Notifies buyer that confirmation deadline was missed.
- **Deposit Payment Required (`DEPOSIT_PAYMENT_REQUIRED`)**: Informs buyer to make a deposit.
- **Deposit Received (`DEPOSIT_RECEIVED`)**: Confirms receipt of buyer's deposit.
- **Deposit Expired (`DEPOSIT_EXPIRED_BUYER`)**: Sent when buyer does not complete deposit before deadline.
- **Agreement Ready to Sign (`AGREEMENT_READY_TO_SIGN`)**: Prompts buyer to sign transaction agreement.
- **Agreement Requires Update (`AGREEMENT_REQUIRES_UPDATE`)**: Prompts buyer to update the agreement.
- **Payment Agreement Admin Rejected (`PAYMENT_AGREEMENT_ADMIN_REJECTED_BUYER`)**: Sent when admin rejects buyer agreement.
- **Remaining Payment Required (`REMAINING_PAYMENT_REQUIRED`)**: Prompts buyer to complete remaining payment.
- **Remaining Payment Received (`REMAINING_PAYMENT_RECEIVED`)**: Confirms receipt of remaining payment.
- **Remaining Payment Proof Submitted (`REMAINING_PAYMENT_RECEIVED_BUYER`)**: Sent when buyer chooses bank transfer for remaining payment and submits proof.
- **Remaining Payment Proof Rejected (`REMAINING_PAYMENT_PROOF_UPDATE_BUYER`)**: Sent when admin rejects bank transfer proof.
- **Ownership Transfer Completed (`OWNERSHIP_TRANSFER_COMPLETED`)**: Confirms successful transfer of ownership.

###### Transaction Events - Seller-Facing
- **Payout Created (`PAYOUT_CREATED`)**: Notifies seller payout is setup and agreement is ready.
- **Payout Agreement Finalized (`PAYOUT_AGREEMENT_FINALIZED`)**: Confirms payout agreement has been signed by both parties.
- **Payout Agreement Requires Update (`PAYOUT_AGREEMENT_REQUIRES_UPDATE`)**: Informs seller of payout agreement revision requests.
- **Payout Agreement Admin Rejected (`PAYOUT_AGREEMENT_ADMIN_REJECTED_SELLER`)**: Sent when admin rejects seller agreement.
- **Payout Completed (`PAYOUT_COMPLETED`)**: Confirms payout was sent to seller.

###### Transaction Expiration & Expiry Events
- **Agreement Expired (`AGREEMENT_EXPIRED`)**: Informs that agreement signing deadline expired.
- **Payment/Payout Cancelled (`PAYMENT_PAYOUT_CANCELLED_EXPIRATION`)**: Informs that payment deadline expired and order cancelled.

---

### 2.2 Business Rules

1. **Dual Channel Configuration**: Every event type operates with independent boolean controls for `inAppEnabled` and `emailEnabled`.
2. **Threshold Display Rule**: The `thresholdPercent` input is visible when `inAppEnabled == true` OR `emailEnabled == true`. It is hidden only when both toggles are `false`.
3. **Threshold Scope**: `thresholdPercent` is stored globally at the event level (rather than separate values for in-app or email channels).
4. **No Auto-Deletion**: Notifications are preserved permanently unless specifically cleaned up by backend archiving policies (no 30-day auto-expiry).
5. **Preference Bypass**: Events belonging to categories `TRANSACTION` and `ACCOUNT` bypass preference settings entirely. They are always dispatched regardless of settings, have no toggles, and do not appear in the Settings UI.
6. **Phase 2 Placeholders**: The eventTypes `MARKETPLACE_ANNOUNCEMENTS_COMMUNICATION`, `PROMOTIONS_OFFERS_COMMUNICATION`, and `ALLOW_PERSONALIZED_ADS_ADVERTISING` represent placeholder preferences in Phase 1. Toggles are managed in Settings UI and persisted via preferences database tables, but no actual events are generated.

---

### 2.3 Threshold Rules

Threshold calculations prevent users from receiving excessive alerts for insignificant price updates.

> [!NOTE]
> **Forward-Looking Thresholds:**
> Threshold evaluation is forward-looking only. Changing `thresholdPercent` or re-enabling a preference toggle does not retroactively trigger re-evaluation of existing active listings/bids against past events — it only applies to future create/update events.

#### Selling Event Threshold Rule (`OFFER_NEAR_LISTING_SELLING`)
Calculated using the seller's active list (ask) price. A notification is triggered if and only if:
`Bid Price >= (thresholdPercent / 100) * Listing Price`

*Example: If Listing Price = £10,000, and threshold = 80%, notifications will only fire for bids of £8,000 or higher.*

#### Buying Event Threshold Rule (`LISTING_NEAR_OFFER_BUYING`)
Calculated using the buyer's active offer (bid) price. A notification is triggered if and only if:
`Bid Price < Listing Price <= Bid Price * (1 + thresholdPercent / 100)`

*Example: If the user bid £1,000 and the threshold is 20%, they will only be notified of new listings priced between £1,001 and £1,200.*

---

### 2.4 APIs

#### 2.4.1 GET /api/notifications/preferences
Retrieves all notification preferences for the logged-in user, grouped by UI section.

- **Method**: `GET`
- **Path**: `/api/notifications/preferences`
- **Response Headers**: `Content-Type: application/json`

##### Request Schema
No request body or query parameters required.

##### Response Schema (200 OK)
```json
{
  "sections": [
    {
      "sectionName": "Buying",
      "preferences": [
        {
          "eventType": "LISTING_NEAR_OFFER_BUYING",
          "label": "Listing near your offer (bid)",
          "description": "Sent when a new listing is published within your notification threshold for a cask variant you have an active Offer on.",
          "inAppEnabled": true,
          "emailEnabled": true,
          "supportsThreshold": true,
          "thresholdPercent": 20
        },
        {
          "eventType": "FLOOR_PRICE_UPDATED",
          "label": "Floor Price Updated",
          "inAppEnabled": true,
          "emailEnabled": false,
          "supportsThreshold": false,
          "thresholdPercent": null
        }
      ]
    },
    {
      "sectionName": "Selling",
      "preferences": [
        {
          "eventType": "OFFER_NEAR_LISTING_SELLING",
          "label": "Offer (Bid) near your listing",
          "inAppEnabled": true,
          "emailEnabled": true,
          "supportsThreshold": true,
          "thresholdPercent": 80
        },
        {
          "eventType": "TOP_OFFER_UPDATED",
          "label": "Top Offer Updated",
          "inAppEnabled": true,
          "emailEnabled": true,
          "supportsThreshold": false,
          "thresholdPercent": null
        }
      ]
    },
    {
      "sectionName": "Cask Exchange Communication",
      "preferences": [
        {
          "eventType": "PRODUCT_UPDATES_COMMUNICATION",
          "label": "Product updates",
          "inAppEnabled": true,
          "emailEnabled": true,
          "supportsThreshold": false,
          "thresholdPercent": null
        },
        {
          "eventType": "MARKETPLACE_ANNOUNCEMENTS_COMMUNICATION",
          "label": "Marketplace announcements",
          "inAppEnabled": true,
          "emailEnabled": true,
          "supportsThreshold": false,
          "thresholdPercent": null
        },
        {
          "eventType": "PROMOTIONS_OFFERS_COMMUNICATION",
          "label": "Promotions & offers",
          "inAppEnabled": true,
          "emailEnabled": false,
          "supportsThreshold": false,
          "thresholdPercent": null
        }
      ]
    },
    {
      "sectionName": "Targeted advertising",
      "preferences": [
        {
          "eventType": "ALLOW_PERSONALIZED_ADS_ADVERTISING",
          "label": "Allow Personalized Ads",
          "inAppEnabled": false,
          "emailEnabled": false,
          "supportsThreshold": false,
          "thresholdPercent": null
        }
      ]
    }
  ]
}
```

##### Field Descriptions
- `sectionName` (string): The logical UI category group.
- `eventType` (string): Unique identifier slug for the notification trigger type.
- `inAppEnabled` (boolean): Flag indicating whether in-app push alerts are active.
- `emailEnabled` (boolean): Flag indicating whether email alerts are active.
- `supportsThreshold` (boolean): Read-only flag indicating if this event supports percentage filtering.
- `thresholdPercent` (integer/null): Selected alert filter percentage. Null if not supported.

---

#### 2.4.2 PATCH /api/notifications/preferences
Updates preferences and threshold details for a single specified event type.

- **Method**: `PATCH`
- **Path**: `/api/notifications/preferences`
- **Request Headers**: `Content-Type: application/json`

##### Request Schema
```json
{
  "eventType": "LISTING_NEAR_OFFER_BUYING",
  "inAppEnabled": true,
  "emailEnabled": false,
  "thresholdPercent": 35
}
```

##### Request Validations
| Field | Required | Type | Validation Rules | Error Code / Message |
|-------|----------|------|------------------|----------------------|
| `eventType` | Yes | String | Must match a supported event type code. | 400 — "Invalid or unsupported eventType" |
| `inAppEnabled` | Yes | Boolean | Must be `true` or `false`. | 400 — "inAppEnabled must be a boolean" |
| `emailEnabled` | Yes | Boolean | Must be `true` or `false`. | 400 — "emailEnabled must be a boolean" |
| `thresholdPercent` | No | Integer | Mandatory if event type supports threshold AND (`inAppEnabled` OR `emailEnabled` is true). Must be >= 1 and <= 100. Not required/validated if both `inAppEnabled` and `emailEnabled` are `false`. | 400 — "Threshold percent must be an integer between 1 and 100" |

##### Response Schema (200 OK)
```json
{
  "eventType": "LISTING_NEAR_OFFER_BUYING",
  "inAppEnabled": true,
  "emailEnabled": false,
  "thresholdPercent": 35,
  "updatedAt": "2026-06-12T10:11:42Z"
}
```

---

## 3. Notification Center

The Notification Center displays past notifications, categorized by type, and tracks read/unread states.

### 3.1 Notification Categories

All incoming notifications belong to exactly one of the following categories:
- **`TRANSACTION`**: Escrow status updates, payouts, agreement signatures, and transaction payment notifications.
- **`MARKETPLACE`**: Status updates on listings, offers, floor price or top offer changes, and threshold-based bid/listing notifications.
- **`ACCOUNT`**: Security alerts, password modifications, 2FA activations, and login events.
- **`SYSTEM`**: Platform announcements, terms and policy updates, or notifications about scheduled maintenance.

---

### 3.2 Notification Lifecycle

A notification has a simple status cycle:
1. **Created**: Saved with status `isRead = false`. Unread counts are incremented.
2. **Read**: The user interacts with the alert (hover checkmark click or full card click). Status transitions to `isRead = true`. Counts are decremented.
3. **Archived / Non-Active** (Backend-only): Preserved in user archives, matching `NF-BR-8` (no auto-expiry).

---

### 3.3 Read/Unread Logic

- **Unread indicators**: Marked in the UI with a distinct green dot and highlighted text.
- **Interaction**: Clicking a notification card or its CTA button triggers redirection using the FE-constructed route based on notification type and `itemId`, and concurrently dispatches a background request to mark the notification as read.
- **Badge count**: Updates dynamically in the header notification bell count whenever status updates occur.

---

### 3.4 APIs

#### 3.4.1 GET /api/notifications
Retrieves list of notifications with pagination and filtering.

> [!NOTE]
> **Pagination Reference:**
> The pagination parameters shown below (e.g. `cursor` and `nextCursor`) represent a reference implementation designed for infinite scrolling. The backend team is free to define the final pagination solution (e.g., offset-based, keyset-based, or cursor-based) as required by the technical architecture.

- **Method**: `GET`
- **Path**: `/api/notifications`
- **Query Parameters**:
  - `category` (string, optional): Filter by category (`TRANSACTION`, `MARKETPLACE`, `ACCOUNT`, `SYSTEM`).
  - `isRead` (boolean, optional): Filter by read status (`true`/`false`).
  - `limit` (integer, optional): Page size limit (default `10`, maximum `100`).
  - `cursor` (string, optional): Base64 encoded string indicating target page offset. Contains the `createdAt` and unique `id` of the last record from the previous list.

##### Response Schema (200 OK)
```json
{
  "items": [
    {
      "id": "2d192f15-ea83-4a18-8f81-5fe5c8621a60",
      "category": "TRANSACTION",
      "type": "PAYMENT_CREATED",
      "title": "Payment Created",
      "message": "Your payment for Macallan 25 Years (18000 USD) has been created. Waiting for seller confirmation.",
      "itemId": "session-12345",
      "isRead": false,
      "createdAt": "2026-06-12T16:00:00Z"
    }
  ],
  "nextCursor": "ZXlKaGNtVmhkR1ZrUVhRaU9pSXlTURUyTFRBMkxURXlWRFE2TURBNk1EQmFJaXdpYVdRaU9pSXlaREV4T1RKaFpUVXRaRUU0TXkwaE1qZ3hMVFp1WlRWall6WXlNV1kyTUNJc0luMCUzRD0=",
  "unreadCount": 3
}
```

##### Field Descriptions
- `items` (array): List of notification records.
- `items[].itemId` (string | null): Related resource ID (e.g., `checkoutSessionId` for payments, `caskId` for listings) to construct redirection paths on the frontend.
- `nextCursor` (string | null): Encoded cursor string for paginating next batch. Returns `null` if no additional older records exist.
- `unreadCount` (integer): Total number of unread notifications matching the active filters (or overall inbox if no filters).

---

#### 3.4.2 PATCH /api/notifications/{id}/read
Marks a single notification as read.

- **Method**: `PATCH`
- **Path**: `/api/notifications/{id}/read`
- **Path Parameters**:
  - `id` (string/UUID): Unique ID of the notification.

##### Response Schema (200 OK)
```json
{
  "id": "2d192f15-ea83-4a18-8f81-5fe5c8621a60",
  "isRead": true,
  "readAt": "2026-06-12T17:11:42Z"
}
```

##### Errors Table
| HTTP Status | Error Code | Message | Description |
|-------------|------------|---------|-------------|
| 403 Forbidden | `ACCESS_DENIED` | "You do not own this notification" | Authenticated user is not the recipient of target notification. |
| 404 Not Found | `NOT_FOUND` | "Notification not found" | Notification ID does not exist in database. |

---

#### 3.4.3 PATCH /api/notifications/read-all
Marks all unread notifications as read. Supports optional category filtering.

- **Method**: `PATCH`
- **Path**: `/api/notifications/read-all`
- **Query Parameters**:
  - `category` (string, optional): If provided, marks only notifications of this category as read.

##### Response Schema (200 OK)
```json
{
  "markedCount": 8,
  "category": "MARKET"
}
```

##### Field Descriptions
- `markedCount` (integer): Number of records modified during execution.
- `category` (string | null): Filter scope applied to command. Null if all inbox marked read.

---

#### 3.4.4 GET /api/notifications/unread-count
Fast-path endpoint to return the current overall unread count badge. Used for global header bell layout updates.

> [!NOTE]
> **SSE real-time updates:**
> In addition to this REST fallback endpoint used for initial page load, the server pushes the unread badge count dynamically to the client via Server-Sent Events (SSE) (endpoint `GET /api/notifications/unread-count/stream`) whenever new notifications are created or read states change.

- **Method**: `GET`
- **Path**: `/api/notifications/unread-count`

##### Response Schema (200 OK)
```json
{
  "unreadCount": 3
}
```

---

## 4. Event Trigger Matrix

This table maps out the notification settings configuration, routing, and trigger mechanics for all known events in Cask Exchange:

| Event Type Code | Category | Trigger Condition | Supports Threshold | Default Configuration |
|---|---|---|---|---|
| `WELCOME_TO_CASKEXCHANGE` | `ACCOUNT` | Email verification completed (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PASSWORD_CHANGED` | `ACCOUNT` | Password updated successfully (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `TWO_FACTOR_AUTHENTICATION_ENABLED` | `ACCOUNT` | User enabled their first 2FA method (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `TWO_FACTOR_AUTHENTICATION_DISABLED` | `ACCOUNT` | The user no longer has any active 2FA methods (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `NEW_LOGIN_DETECTED` | `ACCOUNT` | Login from new device/location (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PAYMENT_CREATED` | `TRANSACTION` | Buyer finishes "Buy Now" flow, OR buyer's offer matches a listing (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `SELLER_CONFIRMATION_FAILED` | `TRANSACTION` | Sale confirmation not completed within the deadline (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `DEPOSIT_PAYMENT_REQUIRED` | `TRANSACTION` | Payment status = Awaiting Deposit (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `DEPOSIT_RECEIVED` | `TRANSACTION` | Deposit payment successful (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `DEPOSIT_EXPIRED_BUYER` | `TRANSACTION` | Buyer does not complete deposit before deadline (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `AGREEMENT_READY_TO_SIGN` | `TRANSACTION` | Admin uploads agreement (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `AGREEMENT_REQUIRES_UPDATE` | `TRANSACTION` | Admin requests update (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PAYMENT_AGREEMENT_ADMIN_REJECTED_BUYER` | `TRANSACTION` | Admin rejects buyer agreement (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `REMAINING_PAYMENT_REQUIRED` | `TRANSACTION` | Deposit verified & agreement completed (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `REMAINING_PAYMENT_RECEIVED` | `TRANSACTION` | Final payment successful (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `REMAINING_PAYMENT_RECEIVED_BUYER` | `TRANSACTION` | Buyer chooses bank transfer for final payment and submits proof (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `REMAINING_PAYMENT_PROOF_UPDATE_BUYER` | `TRANSACTION` | Admin rejects bank transfer proof (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `OWNERSHIP_TRANSFER_COMPLETED` | `TRANSACTION` | Ownership transfer completed (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PAYOUT_CREATED` | `TRANSACTION` | Seller finishes "Sell Now" flow, or seller's listing matches an offer (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PAYOUT_AGREEMENT_FINALIZED` | `TRANSACTION` | Seller/Admin signs agreement (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PAYOUT_AGREEMENT_REQUIRES_UPDATE` | `TRANSACTION` | Admin requests update (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PAYOUT_AGREEMENT_ADMIN_REJECTED_SELLER` | `TRANSACTION` | Admin rejects seller agreement (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PAYOUT_COMPLETED` | `TRANSACTION` | Admin marks payout completed (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `AGREEMENT_EXPIRED` | `TRANSACTION` | Payment/payout agreement not signed within the deadline (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `PAYMENT_PAYOUT_CANCELLED_EXPIRATION` | `TRANSACTION` | Deposit/remaining payment not completed within the deadline (Bypasses preferences - always sent) | No | In-App: `true`<br>Email: `true` (non-configurable) |
| `LISTING_NEAR_OFFER_BUYING` | `MARKETPLACE` | Alert is sent if new listing falls within threshold of active bid | **Yes** | In-App: `true`<br>Email: `true`<br>Threshold: `20` |
| `OFFER_NEAR_LISTING_SELLING` | `MARKETPLACE` | Alert is sent if incoming bid meets seller's listing price threshold | **Yes** | In-App: `true`<br>Email: `true`<br>Threshold: `80` |
| `FLOOR_PRICE_UPDATED` | `MARKETPLACE` | A newly created or updated Listing becomes the floor price for the cask | No | In-App: `true`<br>Email: `true` |
| `TOP_OFFER_UPDATED` | `MARKETPLACE` | A newly created or updated Offer becomes the top offer for the cask | No | In-App: `true`<br>Email: `true` |
| `LISTING_EXPIRED` | `MARKETPLACE` | Listing reaches expiration date | No | In-App: `true`<br>Email: `true` |
| `OFFER_EXPIRED` | `MARKETPLACE` | Offer reaches expiration date | No | In-App: `true`<br>Email: `true` |
| `PRODUCT_UPDATES_COMMUNICATION` | `SYSTEM` | Product updates published | No | In-App: `true`<br>Email: `true` |
| `MARKETPLACE_ANNOUNCEMENTS_COMMUNICATION` | None | Phase 2 placeholder (Marketplace announcements) - saved in preferences but no live events | No | In-App: `true`<br>Email: `true` |
| `PROMOTIONS_OFFERS_COMMUNICATION` | None | Phase 2 placeholder (Promotion & offer) - saved in preferences but no live events | No | In-App: `true`<br>Email: `true` |
| `ALLOW_PERSONALIZED_ADS_ADVERTISING` | None | Phase 2 placeholder (Targeted advertising) - saved in preferences but no live events | No | In-App: `false`<br>Email: `false` |
| `SCHEDULED_MAINTENANCE` | `SYSTEM` | Admin schedules maintenance | No | In-App: `true`<br>Email: `true` |
| `MAINTENANCE_COMPLETED` | `SYSTEM` | Maintenance finished | No | In-App: `true`<br>Email: `true` |
| `TERMS_POLICIES_UPDATED` | `SYSTEM` | Admin publishes new terms | No | In-App: `true`<br>Email: `true` |
