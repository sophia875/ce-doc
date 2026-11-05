# Ví dụ User Story + AC mẫu cho Cask Exchange

> Bản chỉnh cho dự án **Cask Exchange**, dựa trên bộ ví dụ gốc của **Phúc NT** · BA Zone · Digital School.
>
> **Lưu ý:** số tiền, thời hạn và message trong file này chỉ để minh họa format. Khi viết story thật,
> lấy số liệu từ spec trong `docs/` và rule đã chốt ở `docs/project-summary.md` mục 4.
> Chỗ nào chưa có thì ghi `[OPEN]` kèm người cần trả lời, không tự đoán.

---

## Ví dụ 1: Đặt Bid có cảnh báo giá

### US-BID-001: Place a Bid on a vintage

**As a** signed-in Buyer who has completed KYC
**I want to** place a Bid on a vintage at my own price per unit
**So that** I can try to buy below the Lowest Ask instead of paying the current market price

**Metadata**: Epic: Buy Flow · Priority: Must · Core flow: Place Bid / Ask

### INVEST Self-check

| Tiêu chí | Đánh giá | Ghi chú |
|----------|----------|---------|
| Independent | ✅ | Login, KYC and Cask Detail already exist |
| Negotiable | ✅ | Expiry options can be discussed later |
| Valuable | ✅ | Lets price-sensitive buyers enter the market |
| Estimable | ✅ | Matching rules are already documented |
| Small | ✅ | Create only. Update and cancel are separate stories |
| Testable | ✅ | 4 AC with clear numbers |

### Acceptance Criteria

**AC1: Place a Bid within the normal price range - happy path**
- **Given** the Buyer is on the detail page of "Glen Example 2015" with a Market Value of £10,000 per unit
- **When** the Buyer places a Bid of £9,500 per unit for a target quantity of 2 and confirms the order summary
- **Then** the system creates a Bid with status Active, target quantity 2 and filled quantity 0
- **And** the Bid appears in the Buyer's Open Bids list

**AC2: Bid at 120% of Market Value - edge case (upper boundary)**
- **Given** the Market Value is £10,000 per unit
- **When** the Buyer enters a Bid of £12,000 per unit
- **Then** the system shows the price warning before the confirmation step
- **And** the Buyer can still continue and place the Bid

**AC3: Bid at 80% of Market Value - edge case (lower boundary)**
- **Given** the Market Value is £10,000 per unit
- **When** the Buyer enters a Bid of £8,000 per unit
- **Then** the system shows the price warning before the confirmation step
- **And** the Buyer can still continue and place the Bid

**AC4: Invalid price - negative path**
- **Given** the Buyer is on the Place Bid form
- **When** the Buyer enters a price of £0 and tries to continue
- **Then** the system shows a validation error on the price field
- **And** no Bid is created

### Notes
- Dependencies: matching rules (`docs/order-matching-update-specs.md`)
- Open Questions: `[OPEN, Sophia]` PRD says a Bid must be below the Lowest Ask, while matching rules say a Bid matches the Lowest Ask first. Is a Bid at or above the Lowest Ask blocked, or matched straight away?

---

## Ví dụ 2: Sell Now từ portfolio

### US-SELL-NOW-001: Sell a vintage at the Highest Bid

**As a** Seller who owns a vintage bought on Cask Exchange and has connected a Stripe account
**I want to** sell it at the current Highest Bid in one step
**So that** I can exit my position without waiting for a buyer to accept my price

**Metadata**: Epic: Sell Flow · Priority: Must · Core flow: Sell Now -> Matching

### Acceptance Criteria

**AC1: Sell Now with an Active Bid - happy path**
- **Given** the Seller owns 1 unit of "Glen Example 2015"
- **And** the Highest Bid from another user is £10,000 per unit
- **When** the Seller chooses Sell Now for 1 unit and confirms
- **Then** the confirmation screen showed a price of £10,000, a seller fee of £500 (5%) and net proceeds of £9,500
- **And** the order is matched with the Highest Bid and a Checkout Session is created with status `pending_seller`

**AC2: No Active Bid - edge case**
- **Given** the Seller owns 1 unit of "Glen Example 2015"
- **And** there is no Active Bid for this vintage
- **When** the Seller opens the vintage in the portfolio
- **Then** Sell Now is not available
- **And** the Seller can place an Ask instead

**AC3: Stripe account not connected - negative path**
- **Given** the Seller owns 1 unit of "Glen Example 2015" but has not connected a Stripe account
- **When** the Seller chooses Sell Now
- **Then** the system asks the Seller to connect a Stripe account first
- **And** no order is created

### Notes
- Open Questions: `[OPEN, Hung]` If another Seller takes the Highest Bid while this Seller is on the confirmation screen, what does the Seller see?

---

## Ví dụ 3: Thanh toán deposit trong checkout

### US-CHECKOUT-DEP-001: Pay the deposit for a matched order

**As a** Buyer whose Checkout Session is at step 2 Deposit (status `pending`)
**I want to** pay the deposit by card
**So that** the trade moves to the buyer agreement step and I keep the matched casks

**Metadata**: Epic: Checkout · Priority: Must · Core flow: Checkout step 2

### Acceptance Criteria

**AC1: Deposit paid - happy path**
- **Given** the session Subtotal is £20,000, so the Total is £21,000 (Subtotal + 5% buyer fee)
- **When** the Buyer pays the deposit by card before the deadline
- **Then** the Buyer is charged £2,100 (10% of the Total)
- **And** the session moves to `buyer_signature`
- **And** the buyer agreement deadline is set to 3 days after the deposit was paid

**AC2: Session with 2 transactions (1:N) - edge case**
- **Given** one Bid was matched with 2 Asks, so the session holds 2 transactions with a combined Total of £21,000
- **When** the Buyer pays the deposit
- **Then** the Buyer makes one card payment of £2,100 for the whole session

**AC3: Card declined before the deadline - edge case**
- **Given** the deposit deadline has not passed
- **When** the Buyer's card is declined
- **Then** the session stays `pending` and the Buyer can try again
- **And** no email is sent

**AC4: Deadline passes without payment - negative path**
- **Given** the Buyer has not paid the deposit
- **When** the deposit deadline (3 days after the last seller agreement was signed by Admin) passes
- **Then** the session becomes `expired` and the Buyer's matched quantity is cancelled
- **And** the Seller's quantity goes back to the market with its original priority
- **And** the cancellation email does not mention refunds

### Notes
- Release rules: `docs/expiration-handling-specs-partial-reversion.md` section 4
- On staging each step expires after 5 minutes, so QA can test AC4

---

## Ví dụ 4: Hủy Bid đã khớp một phần

### US-BID-CANCEL-001: Cancel the unmatched part of a Bid

**As a** Buyer with an Active Bid on a vintage
**I want to** cancel the part of my Bid that has not been matched
**So that** I stop buying more units while keeping the deal that is already in checkout

### Acceptance Criteria

**AC1: Cancel a Bid with nothing matched - happy path**
- **Given** the Buyer has an Active Bid with target quantity 5 and filled quantity 0
- **When** the Buyer cancels the Bid and confirms
- **Then** the Bid status becomes Cancelled
- **And** the Bid can no longer be matched

**AC2: Cancel a partly matched Bid - edge case**
- **Given** the Buyer has an Active Bid with target quantity 10 and filled quantity 4, and the 4 units are in a Checkout Session
- **When** the Buyer cancels the Bid and confirms
- **Then** the remaining 6 units are closed and can no longer be matched
- **And** the Checkout Session for the 4 matched units does not change

**AC3: Bid already closed - negative path**
- **Given** the Buyer's Bid is Completed, Expired or Cancelled
- **When** the Buyer views the Bid
- **Then** the cancel action is not available
- **And** the Bid does not change

### Notes
- Open Questions: `[OPEN, Sophia]` After AC2, which status does the Bid show: Cancelled or Completed?

---

## Ví dụ 5: Admin hoàn tất chuyển quyền sở hữu

### US-ADMIN-OWN-001: Complete ownership transfer

**As an** Admin (client ops team) handling a Checkout Session with status `invoice_paid`
**I want to** upload the ownership documents and mark the transfer as complete
**So that** the Buyer officially owns the vintage and the Seller gets paid

### Acceptance Criteria

**AC1: Complete the transfer - happy path**
- **Given** the session is `invoice_paid` and has 1 transaction worth £10,000
- **When** the Admin uploads the ownership documents and completes the transfer
- **Then** the session becomes `completed`
- **And** the payout for the Seller moves from `not_ready` to `processing` with an amount of £9,500 (match value minus 5% fee)

**AC2: Session with 2 Sellers (1:N) - edge case**
- **Given** the session is `invoice_paid` with 2 transactions: £12,000 from Seller A and £8,000 from Seller B
- **When** the Admin completes the transfer
- **Then** Seller A gets a payout of £11,400 and Seller B gets a payout of £7,600
- **And** both payouts move to `processing`

**AC3: Final payment not confirmed yet - negative path**
- **Given** the session is `invoice_submitted` (bank transfer proof waiting for review)
- **When** the Admin opens the session
- **Then** the Admin cannot complete the ownership transfer

### Notes
- Payout failure and retry belong to a separate story (see Ví dụ 6, split by business rule)

---

## Ví dụ 6: Story quá lớn cần split

❌ **Quá lớn:**
"As a Buyer, I want to manage my Bids (view, change price, change quantity, cancel)"
(Có chữ "manage", gộp nhiều thao tác, mỗi thao tác có rule priority/matching khác nhau)

✅ **Split thành:**
- **US-BID-VIEW-001**: View my Open Bids
- **US-BID-PRICE-001**: Change the price of an Active Bid (price change resets priority and re-runs matching)
- **US-BID-QTY-001**: Change the target quantity of an Active Bid (increase resets priority and re-runs matching; decrease does not; cannot go below the filled quantity)
- **US-BID-CANCEL-001**: Cancel the unmatched part of a Bid (Ví dụ 4)

---

## Patterns rút ra từ các ví dụ trên

1. **Persona có trạng thái**: Buyer / Seller là vai trò trong 1 giao dịch. Luôn kèm điều kiện: đã KYC, đã connect Stripe, đang ở bước checkout nào.
2. **Ghi vị trí trên core flow** trong Metadata: Browse -> Order -> Matching -> Checkout (5 bước) -> Payout.
3. **Edge case đặc thù**: 0, null, cancel, expire, partial match, 1:N session, hai user cùng lúc. Chọn case phù hợp, không cần đủ hết.
4. **Tiền có số cụ thể** bằng £: fee 5% mỗi bên, deposit 10% của Total, payout = match value trừ 5%.
5. **Quantity nói rõ loại**: target / filled / remaining, không ghi chung chung.
6. **Status dùng đúng tên** có sẵn (`pending_seller`, `pending`, `buyer_signature`, Active, Cancelled...).
7. **Không biết thì `[OPEN]`** kèm người cần trả lời. Mâu thuẫn với rule đã chốt thì ghi rõ conflict.
8. **Split** theo thao tác (view / change price / change quantity / cancel), theo bước checkout, hoặc theo business rule.

---
*Ví dụ gốc: **Phúc NT** · BA Zone · Digital School. Chỉnh cho Cask Exchange.*
*Vui lòng giữ nguyên attribution khi chia sẻ hoặc fork.*
