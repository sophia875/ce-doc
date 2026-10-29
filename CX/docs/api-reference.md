# API Reference — Cask Exchange Backend

> Source: `http://3.235.138.204:3000/api/docs`  
> Last synced: 06/05/2026  
> Base URL: `http://3.235.138.204:3000`  
> Auth: Bearer JWT token (unless noted as public)

---

## Authentication (`/api/auth`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/auth/whoami` | Get current user profile | 🔒 |
| POST | `/api/auth/signup` | Create new user account | Public |
| POST | `/api/auth/signin` | Sign in, returns JWT + refresh token | Public |
| POST | `/api/auth/signout` | Invalidate current session | Public |
| POST | `/api/auth/verify-user` | Verify email with token | Public |
| POST | `/api/auth/resend-verification` | Resend verification email | Public |
| POST | `/api/auth/forgot-password` | Request password reset email | Public |
| POST | `/api/auth/reset-password` | Reset password using token | Public |
| POST | `/api/auth/change-password` | Change password (requires old password) | 🔒 |
| POST | `/api/auth/check-reset-password` | Validate reset password token | Public |
| POST | `/api/auth/verify-password` | Check if password is correct | Public |
| POST | `/api/auth/refresh-token` | Generate new access token | Public |
| POST | `/api/auth/revoke-token` | Revoke a refresh token | 🔒 |
| POST | `/api/auth/revoke-all-tokens` | Revoke all refresh tokens | 🔒 |
| POST | `/api/auth/emergency-logout` | Clear session with expired token | Public |
| GET | `/api/auth/ttb-license-types` | Get all TTB license types | Public |
| POST | `/api/auth/verify-2fa` | Complete sign-in with TOTP 2FA | Public |
| POST | `/api/auth/verify-sms-2fa` | Complete sign-in with SMS 2FA | Public |
| POST | `/api/auth/initiate-2fa-method` | Choose SMS or App 2FA during login | Public |

---

## Two-Factor Authentication (`/api/auth/2fa`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/auth/2fa/status` | Get 2FA status for current user | 🔒 |
| GET | `/api/auth/2fa/devices` | List registered 2FA devices | 🔒 |
| POST | `/api/auth/2fa/devices` | Register a new authenticator device | 🔒 |
| POST | `/api/auth/2fa/devices/verify` | Verify & enable a newly registered device | 🔒 |
| PATCH | `/api/auth/2fa/devices/{deviceId}` | Update device name | 🔒 |
| DELETE | `/api/auth/2fa/devices/{deviceId}` | Remove a registered device | 🔒 |
| DELETE | `/api/auth/2fa/disable` | Disable 2FA and delete all devices | 🔒 |
| POST | `/api/auth/2fa/recovery-codes/regenerate` | Regenerate recovery codes | 🔒 |

## SMS 2FA (`/api/auth/sms-2fa`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| POST | `/api/auth/sms-2fa/start-challenge` | Send SMS OTP | 🔒 |
| POST | `/api/auth/sms-2fa/verify` | Verify SMS OTP | Public |
| POST | `/api/auth/sms-2fa/enable` | Enable SMS 2FA | 🔒 |
| POST | `/api/auth/sms-2fa/disable` | Disable SMS 2FA | 🔒 |
| GET | `/api/auth/sms-2fa/status/{userId}` | Get SMS 2FA status for user | 🔒 |

---

## Security (`/api/security`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/security/sessions` | Get all active sessions | 🔒 |
| POST | `/api/security/sessions/revoke` | Revoke multiple sessions | 🔒 |
| POST | `/api/security/sessions/revoke-single` | Revoke a single session | 🔒 |
| POST | `/api/security/sessions/revoke-all` | Revoke all sessions except current | 🔒 |
| POST | `/api/security/logout` | Log out and invalidate current session | 🔒 |

---

## Casks (`/api/casks`)

> **Note:** The `cask` entity here refers to **variant casks** (children). Master casks are managed via separate fields within the variant response.

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/casks` | Get all casks (requires `status` + `isListed`) | 🔒 |
| POST | `/api/casks` | Create a new cask (multipart with optional image) | 🔒 |
| GET | `/api/casks/listing` | Get paginated casks with full filtering & sorting | 🔒 |
| GET | `/api/casks/search?query=` | Search casks by name/reference/distillery | 🔒 |
| GET | `/api/casks/recently-viewed?limit=` | Get current user's recently viewed casks | 🔒 |
| GET | `/api/casks/{id}` | Get a single cask by ID | 🔒 |
| PUT | `/api/casks/{id}` | Update a cask (multipart with optional image) | 🔒 |
| DELETE | `/api/casks/{id}` | Delete a cask | 🔒 |
| GET | `/api/casks/reference/{reference}` | Get cask by reference number | 🔒 |
| POST | `/api/casks/{id}/view` | Increment view count | 🔒 |
| GET | `/api/casks/{id}/similar` | Get similar casks (scored at vintage level) | 🔒 |
| GET | `/api/casks/{id}/similar-v2` | Get similar casks (optimized V2 at vintage level) | 🔒 |
| GET | `/api/cask-masters/{id}/similar` | Get similar master casks (scored at master level) | 🔒 |
| GET | `/api/casks/filter-options/distilleries` | Get distillery filter options with counts | 🔒 |
| GET | `/api/casks/filter-options/cask-types` | Get cask type filter options with counts | 🔒 |

### `/api/casks/listing` — Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | number | Default: 1 |
| `size` | number | Default: 10 |
| `status` | `active` \| `inactive` \| `discontinued` | Cask status |
| `isListed` | boolean | Whether listed for sale |
| `search` | string | Search by name, reference, or distillery |
| `minVintageYear` / `maxVintageYear` | number | Vintage year range |
| `minPrice` / `maxPrice` | number | Price range |
| `minAbv` / `maxAbv` | number | ABV range |
| `minRla` / `maxRla` | number | RLA range |
| `minOla` / `maxOla` | number | OLA range |
| `minEstimatedBottleCount` / `maxEstimatedBottleCount` | number | Bottle count range |
| `distilleryIds` | string[] | Comma-separated distillery IDs |
| `caskTypeIds` | string[] | Comma-separated cask type IDs |
| `sortBy` | enum | `averageGrowth`, `abv`, `rla`, `ola`, `popularity`, `marketInterest`, `lowestAsk`, `strategicBuys`, `highestBid`, `vintageYear`, `estimatedBottleCount`, `price`, `createdAt`, `viewCount` |
| `sortOrder` | `ASC` \| `DESC` | Sort direction |

---

## Cask Metadata (`/api/cask-metadata`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/cask-metadata/ranges` | Get min/max ranges for cask attributes | Public |
| GET | `/api/cask-metadata/sort` | Get available sort options | Public |
| GET | `/api/cask-metadata/classifications` | Get supported cask classifications (2-level hierarchy, filterable) | 🔒 |

> **Key insight for specs**: `/api/cask-metadata/classifications` returns a 2-level structure (`classification`, `classificationLabel`, `imageUrl`, Level 1 `activeCount`, and nested `styles`). It accepts query parameters for both the **Marketplace page** (`distilleryIds`, `caskTypeIds`, `regionIds`) and the **Distillery Listing page** (`countries`, `statuses`, `companies`, `regionIds`) to dynamically filter categories and recalculate counts based on context. When used as a filter in the marketplace sidebar, only the Level 1 classification values (and their dynamic `activeCount`) are extracted.

---

## Distilleries (`/api/distilleries`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/distilleries` | Get all distilleries | Public |
| POST | `/api/distilleries` | Create a distillery (multipart + optional image) | 🔒 |
| GET | `/api/distilleries/list` | Get paginated distilleries (basic) | Public |
| GET | `/api/distilleries/listing` | Get paginated distilleries with full filtering & sorting | 🔒 |
| GET | `/api/distilleries/{id}` | Get distillery detail with cask masters | Public |
| PUT | `/api/distilleries/{id}` | Update a distillery | 🔒 |
| DELETE | `/api/distilleries/{id}` | Delete a distillery | 🔒 |
| GET | `/api/distilleries/search/{query}` | Search distilleries by name | Public |
| GET | `/api/distilleries/top-ranked?limit=` | Get top distilleries by master cask count (not volume) | Public |
| GET | `/api/distilleries/countries` | Get distinct countries with counts (dynamic filtering) | Public |
| GET | `/api/distilleries/regions` | Get distinct regions with counts (dynamic filtering) | Public |
| GET | `/api/distilleries/companies` | Get distinct companies with counts (dynamic filtering) | Public |
| GET | `/api/distilleries/statuses` | Get distinct statuses with counts (dynamic filtering) | Public |
| GET | `/api/distilleries/{id}/related` | Get related distilleries (scored - see [related-distilleries-api.md](file:///d:/CX/docs/marketplace/related-distilleries-api.md)) | 🔒 |

> ⚠️ **Important for Top Distilleries specs**: The existing `GET /api/distilleries/top-ranked` sorts by **master cask count (DESC)**, NOT by lifetime trading volume. A **new dedicated endpoint** will be needed for the Top Distilleries feature as specified.

### `/api/distilleries/listing` — Sort Options

| `sortBy` value | Description |
|----------------|-------------|
| `establishedYear` | Sort by founding year |
| `masterCaskCount` | Sort by number of active master casks |
| `name` | Sort by name |

---

## Regions (`/api/regions`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/regions` | Get regions (filterable, returns activeCount) | Public |
| POST | `/api/regions` | Create a region | 🔒 Admin |
| GET | `/api/regions/list` | Get paginated regions | Public |
| GET | `/api/regions/{id}` | Get region by ID | Public |
| PUT | `/api/regions/{id}` | Update a region | 🔒 Admin |
| DELETE | `/api/regions/{id}` | Delete a region | 🔒 Admin |

> **Key insight for specs**: `GET /api/regions` returns all fields for each region (`id`, `name`, `country`, `description`, `createdAt`, `updatedAt`) along with a dynamic `activeCount` field representing matching active entities. It accepts query parameters for both the **Marketplace page** (`distilleryIds`, `caskTypeIds`, `classifications`) and the **Distillery Listing page** (`countries`, `statuses`, `companies`, `classifications`) to dynamically filter regions and recalculate counts based on context. Regions with `activeCount = 0` under the selected criteria are excluded from the response.

---

## Cask Bids (`/api/cask-bids`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/cask-bids` | Get all bids (optional `caskId`, `active` filter) | Public |
| POST | `/api/cask-bids` | Create a new bid | 🔒 |
| GET | `/api/cask-bids/my-bids` | Get current user's bids | 🔒 |
| GET | `/api/cask-bids/my-bids/filtered` | Get my bids with pagination, search, sorting | 🔒 |
| GET | `/api/cask-bids/my-bids/status-counts` | Get my bid counts by status | 🔒 |
| GET | `/api/cask-bids/matching-asks?minAskAmount=&desiredQuantity=&caskId=` | Find bids matching an ask price | Public |
| GET | `/api/cask-bids/highest/{caskId}` | Get highest bid for a cask | Public |
| GET | `/api/cask-bids/{id}` | Get bid by ID | Public |
| PUT | `/api/cask-bids/{id}` | Update a bid | 🔒 |
| DELETE | `/api/cask-bids/{id}` | [ADMIN] Delete a bid | 🔒 Admin |
| DELETE | `/api/cask-bids/{id}/cancel` | Cancel a bid (owner only) | 🔒 |
| GET | `/api/cask-bids/suggested/{caskId}` | Get suggested bid amounts (Good/Better/Buy Faster) | Public |
| GET | `/api/cask-bids/market-data/{caskId}` | Get comprehensive market data | Public |
| POST | `/api/cask-bids/validate` | Validate a bid amount | Public |
| POST | `/api/cask-bids/calculate-payment` | Calculate total buyer payment with fees | Public |
| GET | `/api/cask-bids/analytics/{caskId}?bidPrice=&quantity=` | Get bid analytics (warnings, cost) | Public |
| GET | `/api/cask-bids/warning/{caskId}?bidPrice=` | Check for high/low bid warnings | Public |
| GET | `/api/cask-bids/market-depth/{caskId}` | Get bid market depth analysis | Public |
| POST | `/api/cask-bids/calculate-checkout` | Calculate checkout totals for bid orders | 🔒 |
| POST | `/api/cask-bids/validate-discount` | Validate a discount code for bid checkout | 🔒 |

---

## Cask Asks (`/api/cask-asks`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/cask-asks` | Get all asks (optional `caskId`, `active` filter) | Public |
| POST | `/api/cask-asks` | Create a new ask | 🔒 |
| GET | `/api/cask-asks/my-asks` | Get current user's asks | 🔒 |
| GET | `/api/cask-asks/my-asks/filtered` | Get my asks with pagination, search, sorting | 🔒 |
| GET | `/api/cask-asks/my-asks/status-counts` | Get my ask counts by status | 🔒 |
| GET | `/api/cask-asks/lowest/{caskId}` | Get lowest ask for a cask | Public |
| GET | `/api/cask-asks/matching-bids?maxBidAmount=&desiredQuantity=&caskId=` | Find asks matching a bid budget | 🔒 |
| GET | `/api/cask-asks/{id}` | Get ask by ID | Public |
| PUT | `/api/cask-asks/{id}` | Update an ask | 🔒 |
| DELETE | `/api/cask-asks/{id}` | [ADMIN] Delete an ask | 🔒 Admin |
| DELETE | `/api/cask-asks/{id}/cancel` | Cancel an ask (owner only) | 🔒 |
| GET | `/api/cask-asks/analytics/{caskId}?askPrice=&quantity=` | Get ask analytics (warnings, payout) | Public |
| GET | `/api/cask-asks/payout/calculate?askPrice=&quantity=` | Calculate net payout for an ask | Public |
| GET | `/api/cask-asks/warning/{caskId}?askPrice=` | Check for low ask warnings | Public |
| GET | `/api/cask-asks/market-depth/{caskId}` | Get ask market depth analysis | Public |
| GET | `/api/cask-asks/suggested/{caskId}` | Get suggested ask prices (conservative/moderate/aggressive) | Public |
| GET | `/api/cask-asks/validate/{caskId}?askPrice=` | Validate an ask price | Public |
| GET | `/api/cask-asks/quantity/{caskId}?askPrice=` | Get quantity available at a specific price | Public |
| POST | `/api/cask-asks/calculate-checkout` | Calculate checkout totals for purchase | 🔒 |
| POST | `/api/cask-asks/validate-discount` | Validate a discount code for checkout | 🔒 |

---

## Market Orders (`/api/market-orders`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| POST | `/api/market-orders/buy-now` | Buy Now — bid at current lowest ask | 🔒 |
| POST | `/api/market-orders/sell-now` | Sell Now — sell at highest bid | 🔒 |
| GET | `/api/market-orders/current-prices/{caskId}` | Get current `lowestAsk` + `highestBid` for a cask | 🔒 |
| GET | `/api/market-orders/fulfillment/buy/{caskId}?quantity=&budget=` | Analyze buy fulfillment options | 🔒 |
| GET | `/api/market-orders/fulfillment/sell/{caskId}?quantity=&minimumPrice=` | Analyze sell fulfillment options | 🔒 |
| GET | `/api/market-orders/fulfillment/probability/{caskId}?orderType=&price=&quantity=` | Calculate fulfillment probability | 🔒 |

---

## Market Data (`/api/market-data`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/market-data/view` | Get market data overview | Public |
| GET | `/api/market-data/view/cask/{caskId}` | Get market data for a cask | Public |
| GET | `/api/market-data/best-investments?limit=&sortBy=` | Get best investment casks | Public |
| GET | `/api/market-data/cask/{caskId}` | Get market data for a cask | Public |
| GET | `/api/market-data/cask/{caskId}/asks-view` | Get asks view for a cask | Public |
| GET | `/api/market-data/cask/{caskId}/bids-view` | Get bids view for a cask | Public |
| GET | `/api/market-data/asks` | Get all available asks | Public |
| GET | `/api/market-data/bids` | Get all available bids | Public |
| GET | `/api/market-data/sales?caskId=` | Get completed sales for a cask | Public |

---

## Checkout (`/api/checkout`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| POST | `/api/checkout/deposit-payment` | Step 1: Process 10% deposit | 🔒 |
| POST | `/api/checkout/confirm-deposit/{paymentIntentId}` | Confirm deposit payment | 🔒 |
| POST | `/api/checkout/sign-agreement` | Step 2: Get DocuSign link for buyer | 🔒 |
| POST | `/api/checkout/regenerate-agreement` | Regenerate buyer DocuSign envelope | 🔒 |
| POST | `/api/checkout/invoice-payment` | Step 3: Process invoice payment | 🔒 |
| POST | `/api/checkout/confirm-invoice/{paymentIntentId}` | Confirm invoice payment | 🔒 |
| POST | `/api/checkout/complete-transfer/{sessionId}` | Step 4: Complete ownership transfer | 🔒 |
| POST | `/api/checkout/manual-payment-evidence` | Upload manual transfer evidence | 🔒 |
| POST | `/api/checkout/manual-payment-evidence/resubmit` | Resubmit corrected evidence | 🔒 |
| POST | `/api/checkout/admin/manual-payment/{sessionId}/approve` | [ADMIN] Approve manual payment | 🔒 Admin |
| POST | `/api/checkout/admin/manual-payment/{sessionId}/reject` | [ADMIN] Reject manual payment | 🔒 Admin |
| GET | `/api/checkout/session/{sessionId}` | Get checkout session details | 🔒 |
| GET | `/api/checkout/session/{sessionId}/status` | Get checkout session status | 🔒 |
| GET | `/api/checkout/progress/{sessionId}` | Get checkout progress | 🔒 |
| GET | `/api/checkout/session/{sessionId}/invoice-receipt` | Download invoice or receipt | 🔒 |
| GET | `/api/checkout/session/{sessionId}/ownership-transfer-document` | Get ownership transfer document | 🔒 |

---

## Transactions (`/api/transactions`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/transactions/ongoing` | Get ongoing transactions (paginated, filterable) | 🔒 |
| GET | `/api/transactions/ongoing/status-counts` | Get ongoing transaction counts by step | 🔒 |
| GET | `/api/transactions/history` | Get completed/failed transaction history | 🔒 |
| GET | `/api/transactions/history/status-counts` | Get history status counts | 🔒 |
| GET | `/api/transactions/checkout-sessions?bidId=` | Get checkout sessions for a bid | 🔒 |
| GET | `/api/transactions/checkout-sessions/{sessionId}/manual-payment-evidence` | Get manual payment evidence URL | 🔒 |
| GET | `/api/transactions/checkout-sessions/{sessionId}/documents` | Get all documents for a session | 🔒 |
| GET | `/api/transactions/{transactionId}` | Get transaction detail | 🔒 |
| GET | `/api/transactions/asks/{askId}` | Get ask with its transactions | 🔒 |

## Bids (`/api/bids`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| GET | `/api/bids/{bidId}` | Get bid detail with transactions | 🔒 |
| GET | `/api/bids/{bidId}/transactions` | Get all transactions for a bid | 🔒 |

---

## Discount Codes (`/api/discount`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| POST | `/api/discount/apply` | Apply a discount code (2-min reservation) | 🔒 |
| DELETE | `/api/discount/remove` | Remove active discount reservation | 🔒 |
| POST | `/api/discount/confirm` | Confirm discount after successful payment | 🔒 |
| GET | `/api/discount/active` | Get current user's active reservation | 🔒 |
| GET | `/api/discount/reservations` | Get all user discount reservations | 🔒 |
| GET | `/api/discount/health` | Health check | 🔒 |

---

## Stripe Connect (`/api/stripe-connect`)

| Method | Path | Summary | Auth |
|--------|------|---------|------|
| POST | `/api/stripe-connect/accounts` | Create Stripe Connect account | Public |
| GET | `/api/stripe-connect/profile` | Get current user's Stripe profile | 🔒 |
| PUT | `/api/stripe-connect/profile` | Update current user's Stripe account | 🔒 |
| GET | `/api/stripe-connect/payouts` | Get current user's payout details | 🔒 |
| GET | `/api/stripe-connect/payouts/{payoutId}` | Get specific payout | 🔒 |
| POST | `/api/stripe-connect/dashboard-login` | Create Stripe Express Dashboard login link | 🔒 |
| GET | `/api/stripe-connect/health` | Health check | Public |
| POST | `/api/stripe-connect/accounts/persons` | Create a person on Stripe account | 🔒 |
| GET | `/api/stripe-connect/accounts/persons` | Get persons on Stripe account | 🔒 |
| GET | `/api/stripe-connect/accounts/persons/{personId}` | Get a person | 🔒 |
| DELETE | `/api/stripe-connect/accounts/persons/{personId}` | Delete a person | 🔒 |

---

## Key Notes for BA Specs Writers

1.  **Master Cask vs. Variant Cask:**
   - The `/api/casks` group deals with variant casks (children).
   - Master cask data is embedded in the variant response under `master` / `children`.
   - There is **no dedicated `/api/cask-masters` endpoint** — master cask interactions happen via variant endpoints, except for the master-level similar casks endpoint (`GET /api/cask-masters/{id}/similar`).

2. **`classificationLabel` mapping & hierarchy:**
   - Use `GET /api/cask-metadata/classifications` to fetch the hierarchical 2-level structure (classification and nested styles).
   - The backend returns `classification` (Level 1 slug), `classificationLabel` (Level 1 display text), `imageUrl` (image assets), `activeCount` (Level 1 active master cask count), and `styles` (Level 2 enum-like values and labels).
   - In the marketplace filter sidebar, only the Level 1 classification values (and their dynamic `activeCount`) are extracted and applied as active filters.

3. **`region` field:**
   - Region is an **object** (`{ id, name, country, ... }`), not a flat string.
   - When specs reference "region name for display", map it from `region.name` → expose as `regionName` in new endpoints.

4. **Top Distilleries — No existing endpoint:**
   - `GET /api/distilleries/top-ranked` sorts by **master cask count**, not volume.
   - A **new endpoint** is required for the Top Distilleries Home feature.

5. **Explore Casks — No existing endpoint:**
   - No current endpoint handles the curated tab structure (Top Traded, Most Watched, etc.).
   - All 4 tabs require new endpoints backed by a **pre-computed stats table**.

6. **`viewCount` tracking:**
   - Tracked at the **variant cask level** via `POST /api/casks/{id}/view`.
   - The `GET /api/casks/listing` supports `sortBy=viewCount` for the Most Watched tab.
