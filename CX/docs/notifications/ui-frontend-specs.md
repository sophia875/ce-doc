# UI & FRONTEND SPECS

Notification System

## Document Version Control

| Version | Summary of changes | Updated by | Date |
| --- | --- | --- | --- |
| 0.1 | Creation of UI/Frontend specifications for Notification Settings and Center | Antigravity | 12/06/2026 |
| 0.2 | Update notification categories to TRANSACTION, MARKETPLACE, ACCOUNT, SYSTEM and update event names | Antigravity | 03/07/2026 |

---

## I. Screen Overview

The notification system consists of two primary screens:

### 1. Notification Settings Screen
- **Purpose**: Allows users to manage their notification preferences, toggling In-app and Email alerts for various marketplace events, and set price thresholds for high-priority alerts.
- **Screen URL**: `/settings/notifications`
- **User Role(s)**: Authenticated Users (Buyer / Seller)
- **Entry Point(s)**: 
  - User Header Avatar Dropdown -> Settings -> "Notifications" sidebar tab.
  - Gear Icon on the Notification Center page.

### 2. Notification Center Screen
- **Purpose**: Displays received notifications, categorizing them, allowing filtering by read/unread status, and providing quick access to call-to-actions (CTAs) for key marketplace activities (e.g., making payments, reviewing offers).
- **Screen URL**: `/notifications`
- **User Role(s)**: Authenticated Users (Buyer / Seller)
- **Entry Point(s)**: 
  - Notification Bell Icon in the global header (displays unread badge count).

---

## II. UI Elements Table

### 1. Notification Settings Screen (`/settings/notifications`)

| # | Component | Type | Description | States / Variants |
|---|-----------|------|-------------|-------------------|
| 1 | Settings Sidebar | List / Links | Navigation menu to switch settings screens (Personal Information, Security and Privacy, Notifications). | Default, Active (Notifications selected), Hover |
| 2 | Screen Header | Title & Subtitle | Displays "Notifications" and "Manage how you receive marketplace updates, offers, and trading activity." | Default |
| 3 | Section Container | Card / Group | Groups related event types (Buying, Selling, Cask Exchange Communication, Targeted Advertising). | Default |
| 4 | Event Item Row | Row Layout | Individual event row containing description, category, and channel toggles. | Default, Hover |
| 5 | Channel Toggle (In-app) | Toggle Switch | Enables/disables in-app notifications for the event type. | Checked, Unchecked, Hover, Disabled, Loading |
| 6 | Channel Toggle (Email) | Toggle Switch | Enables/disables email notifications for the event type. | Checked, Unchecked, Hover, Disabled, Loading |
| 7 | Threshold Percent Input | Numeric Input | Input box to specify the percentage threshold. Only visible if In-app OR Email toggle is enabled for supported events. | Default, Focus, Error, Disabled |
| 8 | Threshold Inline Label | Text | Describes the threshold logic (e.g. "Notify Me when the Offer reaches at least [32]% of my Listing price" or "Notify Me when the Listing reaches within [32]% of my Offer price"). | Default |

### 2. Notification Center Screen (`/notifications`)

| # | Component | Type | Description | States / Variants |
|---|-----------|------|-------------|-------------------|
| 1 | Category Sidebar | List / Links | Filters notifications by category: All, Transaction, Marketplace, Account, System. Shows count next to each. | Default, Active, Hover |
| 2 | Header Title | Title & Subtitle | Displays category name (e.g., "Market") and dynamic description of the category. | Default |
| 3 | "Unread only" Toggle | Toggle Switch | Filters the notification list to only show unread notifications. | Checked, Unchecked, Hover |
| 4 | "Mark all read" Link | Button / Link | Marks all notifications in the current view/category as read. | Default, Hover, Active, Disabled (when no unread items) |
| 5 | Settings Gear Icon | Icon Button | Navigates back to the `/settings/notifications` settings screen. | Default, Hover, Focus |
| 6 | Time Block Header | Text Header | Subdivides list into time periods: Today, Yesterday, Earlier This Week, Earlier This Month. | Default |
| 7 | Notification Card | Row Card | Individual notification details (icon, title, message, relative time, status indicator, CTA). | Unread, Read, Hover (reveals inline actions) |
| 8 | Category Icon | Icon | Visual indicator of notification category on the left side of the card. | Default (custom per category) |
| 9 | Unread Indicator | Circle Badge | A green dot (`#22C55E`) on the right side of the card signifying unread status. | Visible (Unread), Hidden (Read) |
| 10 | CTA / Action Button | Button | A button inside the notification card triggering a specific redirection flow (e.g. "Make Payment" for DEPOSIT_PAYMENT_REQUIRED). Renders based on notification type and presence of `itemId`. | Default, Hover, Active, Focus, Hidden (if null) |
| 11 | Inline "Mark as Read" Button | Icon Button | Checkmark icon appearing on hover on unread cards, enabling quick read management. | Default, Hover, Hidden (if already read) |
| 12 | Empty State Illustration | Image & Text | Shown when there are no notifications in the current filter/category. | Default |
| 13 | Load More Button / Indicator | Button / Spinner | For paginated lists, dynamically requests the next set of items. | Default, Loading, Hidden (no more items) |

---

### Component Details

#### Threshold Percent Input (Settings)
- **Position**: Placed immediately below the description text of supported event rows (`Buying - Listing near your offer (bid)`, `Selling - Offer (Bid) near your listing`).
- **Default Value**: `20` (Buying) / `80` (Selling) (or whatever value is retrieved from preferences API, falling back to database default).
- **Interaction**: 
  - Shown when either `In-app` or `Email` toggle is enabled (`true`).
  - Slide out / Fade in animation when transitioning from hidden to visible.
  - Hidden and saved preferences updated when both toggles are disabled (`false`).
- **Responsive Behavior**: Stacked vertically below the toggles on screens under 768px.

#### Notification Card (Center)
- **Position**: In the main layout stream, stacked vertically inside time groups.
- **Interaction**:
  - Clicking the card body: Redirects the user to the FE-constructed destination route (determined by notification type and `itemId`) and simultaneously sends a patch request to mark it as read.
  - Clicking the CTA button: Triggers direct navigation to the FE-constructed destination route and marks notification as read.
  - Hovering on an unread card: Reveals a checkmark button (`Mark as Read`) on the right side. Clicking it marks the notification as read without navigating.

---

## III. Field Validations

| # | Field Name | Required | Min Length | Max Length | Format / Pattern | Custom Rules | Error Message |
|---|------------|----------|-----------|-----------|-----------------|--------------|---------------|
| 1 | `thresholdPercent` | ✅ (when threshold input is visible) | N/A | N/A | Integer (1–100) | Must be between 1 and 100 inclusive. Non-numeric or decimals rejected on input. | "Threshold must be a whole number between 1 and 100." |

---

## IV. Cases and Flows

### IV.1. Load Notification Preferences
| Aspect | Detail |
|--------|--------|
| **Trigger** | On `/settings/notifications` page mount. |

#### Loading State
- Displays skeleton loaders for settings groups.
- Toggles are grayed out and non-interactive.

#### Success State
- Toggles mapped to current values from `GET /api/notifications/preferences`.
- If a supported event has In-App or Email toggle = `true`, the threshold percent input field is rendered with its stored value.

#### Error States
| Error Condition | FE Behavior | User Message |
|-----------------|-------------|--------------|
| Unauthorized / Session Expired | Redirect to login page | "Session expired. Please log in again." |
| Server Error / Timeout | Show retry button on empty settings canvas | "Failed to load notification settings. Please try again." |

---

### IV.2. Toggle Channel Preferences
| Aspect | Detail |
|--------|--------|
| **Trigger** | Clicking the In-app or Email toggle switch for any event. |

#### Loading State
- The targeted toggle switch displays a subtle spinner or enters a semi-transparent state.
- Interaction with the specific row toggles is disabled during the API call.

#### Success State
- Toggle changes state successfully.
- If toggling ON a supported event row (`Buying - Listing near your offer (bid)`, `Selling - Offer (Bid) near your listing`) and both were previously OFF, display the threshold input field with its default value (20% for buying, 80% for selling).
- If toggling OFF a toggle and both toggles are now OFF, hide the threshold input field.

#### Error States
| Error Condition | FE Behavior | User Message |
|-----------------|-------------|--------------|
| API Validation Error | Revert toggle to previous position | "Invalid setting value." |
| Server Error / Network Timeout | Revert toggle to previous position | "Failed to save settings. Please check your connection." |

---

### IV.3. Load Notifications list (Notification Center)
| Aspect | Detail |
|--------|--------|
| **Trigger** | On page mount or when selecting a new sidebar category filter or toggling "Unread only". |

#### Loading State
- Sidebar counts show skeleton pulse.
- Main notification list displays skeleton rows mimicking notification cards.

#### Success State
- Maps returned items to card elements grouped by date headers (Today, Yesterday, etc.).
- Active categories highlight in the sidebar.
- Updates the total unread badge count next to the header bell and the selected category menu.
- Stores the pagination state/token returned by the backend for pagination.

#### Empty State
- Renders an empty state layout with a bell icon, message "You're all caught up!", and a button "Back to Marketplace".

---

### IV.4. Mark Notification as Read
| Aspect | Detail |
|--------|--------|
| **Trigger** | Click on inline checkmark icon (Mark as Read) OR click on card body / CTA. |

#### Success State
- The green dot indicator is faded out.
- The card background changes to standard "Read" background color.
- Sidebar unread counts and global header badge count are decremented by 1.
- Inline checkmark button fades out.

---

### IV.5. Mark All Notifications as Read
| Aspect | Detail |
|--------|--------|
| **Trigger** | Clicking "Mark all read" link button. |

#### Success State
- All unread green dots fade out.
- Sidebar counts for current active category (or all if "All" is active) are set to 0.
- Header unread bell badge is updated dynamically.

---

## V. Edge Cases & FE-only Logic

| # | Scenario | Expected FE Behavior |
|---|----------|---------------------|
| 1 | Loss of Internet Connection mid-page | Show floating banner "Offline: Changes may not be saved." Disable preference toggles. |
| 2 | Extremely long notification message | Truncate message body to a maximum of 3 lines with ellipsis `...`. Add a "Show more" link to expand the text inline if message exceeds 3 lines. |
| 3 | Rapid toggle clicking (Debounce) | Apply debouncing (300ms) to toggle triggers to avoid flooding backend with sequential preferences updates. |
| 4 | Notification with missing or non-redirectable `itemId` / type | Do not render any pointer cursor or hover click shadow on the notification card body. Omit the CTA button entirely. |

---

## VI. Navigation & Routing

| Action | Destination | URL Change | Notes |
|--------|-------------|-----------|-------|
| Click notification card (with redirectable `itemId`) | Targeted action target | `/checkout/...` or `/marketplace/...` etc. | Redirect occurs after PATCH read request is sent. Route is dynamically constructed by FE using type and `itemId`. |
| Click Settings Gear Icon | Notification Settings Page | `/settings/notifications` | Opens settings view. |
| Click Sidebar category link | Filter list view | `/notifications?category=categoryName` | Updates query parameters. |

---

## VII. Responsive Design Notes

| Breakpoint | Layout Changes |
|-----------|---------------|
| Desktop (≥ 1024px) | Two-column layout: Sidebar navigations on left (fixed width 280px), main contents on right. |
| Tablet (768px – 1023px) | Sidebar collapses into a top horizontal filter bar (scrollable on overflow). Settings sidebar transitions to a top dropdown trigger. |
| Mobile (< 768px) | Full-width single-column layout. Notification cards are padded tightly. Toggles stack vertically under the text details if necessary. CTA buttons expand to full-width buttons at bottom of cards. |

---

## VIII. Accessibility Requirements

- [ ] All interactive elements (toggles, input fields, category tabs, and buttons) have unique `id` attributes.
- [ ] Notification preferences toggles have proper `aria-label` identifying the event and channel (e.g. `aria-label="In-app notification for New Highest Offer"`).
- [ ] Keyboard navigation support (Tab key correctly iterates through setting toggles and notification items; pressing Enter/Space clicks toggles/buttons).
- [ ] Focus indicators visible (vibrant ring around focused elements).
- [ ] Color contrast ratio ≥ 4.5:1 for all text elements (including muted timestamps and helper descriptions).
- [ ] Screen readers read out unread/read state dynamically using `aria-live` when state updates occur.

---

_End of Document 1_
