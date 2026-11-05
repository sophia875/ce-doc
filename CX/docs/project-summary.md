# Cask Exchange: Master Context

Draft v0.1, 29/10/2026. Owner: Sophia (PM), handing over to the next PM. 

**Who reads this file**

- The PM / BA / QA who runs the project. Read it fully once, then use it as a reference.  
- AI tools (Antigravity, Claude). Attach or point to this file every time you ask them to write or review a spec, a test case, or a client message.

**Tag used in this file**

- `[OPEN]` \= not decided yet. Do not build on it without a decision.

---

## 0\. Rules for AI tools (read first)

1. Use only fields, statuses and endpoints that exist in section 3 or in the specs linked in section 5\. If a feature needs a new or changed field, list it clearly in its own "Data changes" part of the spec so BE can see it. Do not slip it in silently.  
2. Use the templates in `CX/templates/` (UI spec \= document 1, BE spec \= document 2).  
3. A spec usually covers: data description (type, source, when it updates), status before/after, use cases with happy path and edge cases (0, null, cancel, expire, partial match, two users at the same time), and impact on existing endpoints and screens. Pick what fits the feature; not every item applies to every spec.  
4. If something is unknown, write `[OPEN]` with the person who must answer. Never fill gaps with guesses.  
5. Business rules in section 4 are locked. If a request conflicts with one, flag the conflict, do not silently change the rule.  
6. Name the product "Cask Exchange". Never "CaskX Exchange" or "CaskX".  
7. Plain English, short sentences. Currency is GBP (£).

---

## 1\. Product in one page

**What it is.** A marketplace to buy and sell whisky casks that are still maturing. It works like a stock exchange: users place buy prices (bids) and sell prices (asks), the system matches them, then the deal goes through a multi-step checkout over several days. The market is the UK.

**Who uses it**

| Role | What they do |
| :---- | :---- |
| User | One account can both buy and sell. "Buyer" and "Seller" are roles in a single trade, not account types. |
| Admin (client's ops team) | Lists casks, reviews and signs agreements, confirms bank transfers, completes ownership transfer, retries payouts. |
| System | Matching, expiry, cancellation cascade, emails. |

**Who can sell.** At launch only Admin sells: outside casks are not allowed, so Admin owns all casks at the start. Once a user buys a cask from Admin, that user can sell it again (Place Ask / Sell Now). There is no flow to sell a cask from outside the platform, or a cask the user owns but that is not on Cask Exchange.

**Two tracks**

- **MVP (v1):** live on production since 16 to 19/01/2026, but production has not been deployed since 04/2026. Phase 2 of MVP targets 31/10/2026.  
- **v2 (Cask Exchange 2.0):** redesign. FE is done. Remaining work is BE, FE integration, QA, deploy. Plan: 2/11 to 31/12/2026. BE capacity about 15h/week (plan with 12h). Scope is not locked; cut scope to fit capacity and remove the matching UI for anything cut.

**Environments**

| Env | URL | Notes |
| :---- | :---- | :---- |
| Production | [https\://platform.caskexchange.com](https://platform.caskexchange.com) | Real users and data. Never test here. |
| MVP staging | [http\://3.235.138.204](http://3.235.138.204) | HTTP only, so Stripe embedded UI and KYC camera do not work properly. Swagger: port 3000, `/api/docs`. |
| v2 staging | [https\://cask-exchange-bp.vercel.app](https://cask-exchange-bp.vercel.app) | v2 FE, includes how-to-buy guide. |
| Marketing site | [https\://www\.caskexchange.com](https://www.caskexchange.com) | Webflow. |

**Tech stack**

> Note: the FE part below comes from an AI summary of the FE repo ([FE technical overview](https://docs.google.com/document/d/1-iTC5QT6ptkcoE7O8MR4Giw0oxVOMk5w04QTORZZrVw/edit?tab=t.l5sn68txiq1e)). It may be incomplete or not fully accurate. A PM does not need this level of detail day to day; use it to know who owns what and where to ask.

| Layer | What | Owner |
| :---- | :---- | :---- |
| FE (web app) | Next.js 15, React 19, TypeScript. UI: Tailwind, shadcn/Radix. Data: TanStack Query, Zustand. Forms: React Hook Form \+ Zod. Login: NextAuth (JWT). | Bearplus (Kelvin) |
| Admin dashboard | Same FE repo, staff-only routes. BE enforces permissions. | Bearplus |
| Resources (guides, FAQs) | Payload CMS 3 inside the FE repo, with its own PostgreSQL and S3 media. Editors manage content at `/cms-admin`. | Bearplus |
| BE (commerce API) | NestJS. Owns accounts, casks, orders, matching, checkout, payments, payouts, ownership. It is the source of truth; FE only shows its state. | Namtech |
| Timed jobs | Redis \+ BullMQ (expiry). FE has no scheduler. | Namtech |
| Realtime | Notification stream (SSE) for the inbox; Socket.IO for live market data. | BE sends, FE shows |
| Integrations | Stripe (payments) and Stripe Connect (payouts); DocuSign (agreements; field positions live in BE code); Twilio (SMS 2FA); AWS SES (email from [support@caskexchange.com](mailto:support@caskexchange.com)). Stripe and DocuSign webhooks are handled by BE, not FE. | Namtech |
| Infra | AWS: EC2 \+ Postgres, CloudFront, S3 (PDF files). | Namtech / client |

**Deploy**

- BE: branches `develop`, `staging`, `production`. Merging `staging` into `production` triggers CI/CD. Namtech merges to production.  
- FE: push to `staging` deploys to EC2 (Docker Compose); push to `production` builds an image to ECR and updates ECS. CMS database migrations are not part of the deploy and must be run separately.  
- Repos live in the client's GitHub org (`caskxofficial`).

**People** (details in the handover doc)

| Who | Role | Contact for |
| :---- | :---- | :---- |
| Steve Scheidecker | Client owner / CEO | Business rules, legal, money decisions. Slow to answer: ask early, chase weekly. |
| Jon Itzen | Client, day-to-day reviewer, is a developer | Reviews Figma, staging, UAT. Owns DNS, Stripe, DocuSign, Twilio, GitHub invites. |
| Barrett | Bearplus partner | Commercial, scope, extra hours, escalation with vendor. |
| Kelvin | Bearplus FE dev | FE status, integration. |
| Hung (Namtech) | BE dev, part-time | BE estimate and build. Wants bug reports with ID \+ request detail. |
| Matt (Namtech) | Vendor QC / coordinator | ETA, routing BE questions. |

---

## 2\. The core flow (the spine of the product)

Every feature sits somewhere on this line. When you write or review a spec, first say where on this line it sits.

```
Browse -> Place Bid / Ask, Buy Now / Sell Now -> Matching -> Checkout Session
  -> 1. Seller agreement -> 2. Deposit -> 3. Buyer agreement -> 4. Final payment
  -> 5. Ownership transfer -> Payout to seller
```

### 2.1 Ordering

- **Bid** \= buy order. **Ask** \= sell order.  
- **Buy Now** \= buy at the Lowest Ask. **Sell Now** \= sell at the Highest Bid (only casks you own on the platform, i.e. bought through Cask Exchange).  
- **Fulfillment type:** `Full` \= must fill the exact quantity in one go. `Partial` \= can fill in parts.  
- Unfilled quantity waits in the **waitlist** (the order book).

### 2.2 Matching

- Priority: **Price \> Quantity \> Fulfillment (Full before Partial) \> Time** (`priority_timestamp`).  
- Bids match the lowest ask first. Asks match the highest bid first.  
- A user's own bid and ask never match each other.  
- If no 1:1 match, the system tries a **composite match** (1 order vs several). Updated rule: accumulate Full orders first, then fill the rest. See `CX/docs/order-matching-update-specs.md`.

### 2.3 Checkout session (after a match)

- One match event \= one **Checkout Session**. It holds one or more **Transactions** (one per matched ask). 1 bid matched to 2 asks \= 1 session, 2 transactions ("1:N").  
- **Settlement type:** Total \<= £30,000 \= **Indirect** (Cask Exchange sits in the middle, admin countersigns). Total \> £30,000 \= **Direct** (buyer and seller sign one agreement per transaction, admin approves). Threshold is admin-configurable.

| Step | Session status | Who acts | Deadline (`expiryDate`) |
| :---- | :---- | :---- | :---- |
| 1\. Seller agreement | `pending_seller` | Seller signs (DocuSign), Admin signs/approves | `null` (no timer while waiting for sellers) |
| 2\. Deposit | `pending` | Buyer pays by card (Stripe) | MAX(all `sellerAgreementAdminSignedAt`) \+ 3 days |
| 3\. Buyer agreement | `buyer_signature` | Buyer signs, Admin signs/approves | `depositPaidAt` \+ 3 days |
| 4a. Final payment, card | `invoice_payment` \-\> `invoice_paid` | Buyer pays, Stripe webhook confirms | MAX(all `buyerAgreementAdminSignedAt`) \+ 3 days |
| 4b. Final payment, bank transfer | `invoice_payment` \-\> `invoice_submitted` | Buyer uploads proof, then Admin reviews it | Two windows: (1) buyer has 3 days to upload proof (same base as 4a); (2) Admin has 10 days to review, counted from the first upload. Re-upload does NOT reset the 10 days. |
| 5\. Ownership transfer | `invoice_paid` \-\> `completed` | Admin uploads documents and completes | `null` (no deadline) |

- Deposit status is `pending` (not `pending_deposit`; some specs still use the old name).  
- Steps are gated: opening another step by URL redirects to the current step.  
- Admin has no signing deadline. Admin can request seller agreement revisions with no cap; each request resets the seller deadline to now \+ 3 days.  
- On staging, deadlines are shortened so QA can test expiry: 5 minutes per step, and 10 minutes for Admin to review bank transfer proof (4b).  
- Source: `CX/docs/checkout-session-pending-seller-specs.md`.

### 2.4 Payout (seller gets paid)

- Triggered automatically when Admin completes ownership transfer (bank-transfer orders too).  
- `payoutStatus`: `not_ready` \-\> `processing` \-\> `completed` (Stripe `payment.created` on the connected account) or `failed` (`transfer.reversed`).  
- Retry: Admin via Swagger, `PATCH /api/admin/settlements/{transactionId}`.  
- Amount: match value minus 5% fee, in GBP (converted to USD if the seller account does not accept GBP).  
- Seller must connect a Stripe account to sell.

### 2.5 Cancel and expiry

- One transaction cancelled \= the whole checkout session is cancelled.  
- Release is **fault-based**: the party at fault is cancelled, the other party's quantity goes back to the market. Rules matrix: `CX/docs/expiration-handling-specs-partial-reversion.md` section 4\.  
- Returned quantity keeps its original `priority_timestamp` and waits for the next order event (it is not re-matched immediately).  
- Expiry runs on BullMQ jobs scheduled when `expiryDate` is set and removed when it becomes `null`. States with `expiryDate = null` must never be expired.  
- DocuSign envelopes are voided on cancel.

---

## 3\. Data model and glossary

### 3.1 Cask structure

```
Distillery -> Master Cask (parent, one product) -> Variant (child, one vintage)
```

- Trades happen at **variant** level.  
- **Naming:** "Variant" is the name in specs, code and API. When talking to the client, the team or users, say **"vintage"**.  
- Master Cask: `name`, `distillery`, `caskType`, `classification`, `region` (object: use `region.name`), `image`, `viewCount`, `status`.  
- Variant: `vintageYear` (required), `distillationDate`, `age`, `ola`, `rla`, `abv`, `estimatedBottleCount`, `referencePriceMin/Max` (required), `caskStatus` (`active`/`inactive`), `isListed`, `readyToSell`, `lowestAsk`, `highestBid`.  
- Display name format: `[Master Cask Name] [Vintage Year]`. Every cask has a vintage year.  
- Barrel ID was removed (03/2026).

### 3.2 Quantity fields (the most common source of bugs)

| Field | Meaning |
| :---- | :---- |
| `targetQuantity` | What the user asked for |
| `filledQuantity` | Sum of quantity in non-cancelled transactions. Never written directly. |
| `remainingQuantity` | `target - filled` |
| `withheldQuantity` | Reserved, not available to match |
| `matchableQuantity` | `remaining - withheld` |

The same word "quantity" means different things in different endpoints. Always name the exact field in a spec.

### 3.3 Statuses

| Object | Values |
| :---- | :---- |
| Bid | Active, Completed, Expired, Cancelled |
| Ask | Active, Completed, Expired, Cancelled (no Pending) |
| Checkout session | `pending_seller`, `pending`, `buyer_signature`, `invoice_payment`, `invoice_submitted`, `invoice_paid`, `completed`, `cancelled`, `expired` |
| Seller agreement | `awaiting_signature` \-\> `seller_signed` \-\> `signed` or `rejected` |
| Manual payment | `pending_review` from first proof upload |
| Payout | `not_ready`, `processing`, `completed`, `failed` |

### 3.4 Glossary

| Term | Meaning |
| :---- | :---- |
| LA / HB | Lowest Ask / Highest Bid |
| LSP / ASP / RP | Last Sale Price / Average Sale Price / Reference Price (admin default) |
| MV (Market Value) | First available of LA \> LSP \> ASP \> HB \> RP |
| Transaction | 1 bid matched with 1 ask |
| Checkout Session | Buyer's payment session for one match event, holds 1..N transactions |
| Payment / Payout | Money in from buyer / money out to seller. 1 Payment : N Payouts |

---

## 4\. Locked business rules

| Rule | Value | Decided |
| :---- | :---- | :---- |
| Fee | 5% buyer \+ 5% seller, configurable | 25/08/2025 |
| Checkout Total | Subtotal \+ 5% buyer fee |  |
| Deposit | 10% of Total (Total includes the fee), card only, no minimum | 01/10/2026, Sophia |
| Currency | GBP (£) | Changed from \$ on 24 to 25/12/2025 |
| Settlement | \<= £30,000 Indirect, \> £30,000 Direct |  |
| Price warning | Bid/Ask \>= 120% or \<= 80% of MV shows a warning. User can still continue (also on update). |  |
| Discount codes | Removed |  |
| Refunds, warehouse fees | Handled off-platform |  |
| Ownership transfer | Manual: Admin uploads documents |  |
| Wire transfer | Client's accounting confirms manually |  |
| Payout reversal | Only sets status `failed`. Stripe balance can go negative; client tops it up. |  |
| Invoice / receipt links | Public S3 links, no login, no expiry, must be non-guessable | 01/10/2026, Sophia |

**Order update rules** (source: `Bid Update & Cancel - Data Model & Business Rules`)

| Update | Reset priority? | Re-run matching? |
| :---- | :---- | :---- |
| Price up or down | Yes | Yes |
| Quantity increase | Yes | Yes |
| Quantity decrease | No | No |
| Fulfillment change (only if filled \= 0\) | Yes | Yes |
| Expiry change | No | No |

- Partially matched order: quantity min \= filled, fulfillment locked. Fully matched, expired or cancelled: everything locked.  
- Cancel only closes the unmatched part. Existing checkout sessions are not touched.

**Email rules**

- No email when a seller or buyer signs (waiting for admin).  
- No email on card decline before deadline, or on Stripe payout failure.  
- Cancellation emails do not mention refunds.

**Open conflicts that may change flows**

- `[OPEN, Steve]` Supplier Terms still have blank `[●]` values: days for seller to accept/decline, business days funds are held before payout.  
- `[OPEN, Steve]` 3 days per step vs contract saying invoice due in 14 days; UK/EU buyers' legal right to cancel/refund (raised 22/06/2026).  
- `[OPEN]` Deposit refund when a session is cancelled after deposit is paid.  
- `[OPEN]` Buyer wires real money but the 10-day window expires: no reconcile or refund process.

---

## 5\. Where things live (source of truth map)

| Need | Go to |
| :---- | :---- |
| Client, team, history, commercial rules | Handover doc, tab "Cask Exchange" |
| v2 scope and weekly plan | Google Sheet "CaskExchange workspace", Sheet 8 (backlog v2) |
| BE estimates, ETA, ClickUp links | Google Sheet "CaskExchange \- BE backlog" |
| Latest API / BE specs for 2.0 | Google Doc "CE 2.0 APIs & BE Specs" |
| Spec drafts (markdown) | GitHub `sophia875/ce-doc`, folder `CX/docs/` |
| Spec templates | `CX/templates/` |
| Test cases and client bugs | QA Google Sheets (MVP Phase 1, MVP Phase 2, v2). Orange \= ready to test, green \= partly blocked, white \= not tested. |
| BE tickets | ClickUp folder "Cask Exchange P2" |
| Pending client requests (warehouse, refund) | Google Doc "Cask Exchange \- New requests" |
| Legacy requirements | Old Google Doc \+ ClickUp doc. Reference only, partly outdated. |
| Live API contract | Swagger on MVP staging (`:3000/api/docs`) |
| FE technical overview (AI summary) | [Google Doc](https://docs.google.com/document/d/1-iTC5QT6ptkcoE7O8MR4Giw0oxVOMk5w04QTORZZrVw/edit?tab=t.l5sn68txiq1e). For devs mostly; may be incomplete. |

**When sources disagree, this order wins:**

1. A dated decision with an owner (meeting recap, client email, ticket comment).  
2. The ClickUp ticket description (agreed with Hung, 11/09/2026: ticket beats old Slack threads).  
3. The newest spec (Google Doc "CE 2.0 APIs & BE Specs", then the repo).  
4. Swagger, for what BE actually built.  
5. Legacy docs.

For transaction emails, the diagram `Email in transaction flow.pdf` wins over `Transaction_Flow_post match.md`.

---

## 6\. How work flows (process \+ quality gates)

```
Need -> Spec (AI draft) -> Spec review (AI + human) -> Ready gate -> BE estimate
     -> BE build -> FE integrate -> QA (Swagger / UI) -> Bug fix -> Update this file
```

&nbsp;

| Step | What to do | Gate before moving on |
| :---- | :---- | :---- |
| 1\. Need | Collect the request, Figma link, client context. | You can write one sentence: who does what, on which screen, to get what. You know where it sits on the core flow (section 2). |
| 2\. Spec draft | Ask Antigravity, with this file \+ the template. | Uses only known fields; new or changed fields are listed clearly. |
| 3\. Spec review | Ask Claude to review against the checklist below. Fix, repeat. | All checklist items pass. |
| 4\. Ready gate | Read the spec yourself and explain it out loud. If FE must provide an HTML template (email, PDF), attach it. | You can explain every part. Open questions have an owner. |
| 5\. BE estimate | Send to Namtech in batches (1 to 2 weeks of work). | Note: estimating uses BE hours. Do not send tasks one by one. |
| 6\. BE build | Ticket description is the source of truth. Any answer given in Slack must be copied into the doc/ticket. |  |
| 7\. QA | Test with Swagger or on staging after FE integrates. Generate test cases from the spec with AI (specs only have use cases), review them, then test. | Bug report has: ID, request, response, expected result. |
| 8\. Close | Update this file if a rule, field or status changed. | Section 3 and 4 still true. |

**Spec checklist (gate 3 and 4\)**

Use it as a guide, not a fixed rule. Pick the items that fit the feature; a small change does not need all of them.

1. Every new field has: type, where the value comes from, when it updates.  
2. Every status change: before \-\> after, and who or what triggers it.  
3. Edge cases: 0, null, cancel, expire, partial match, 1:N session, two users at the same time.  
4. Impact: which existing endpoints and screens change.  
5. Use cases: happy path and edge cases with expected results. (Test cases come later: generate them from the spec with AI.)  
6. No conflict with section 4\. Conflicts are flagged, not hidden.  
7. Open questions marked `[OPEN]` with an owner.

**Ticket flow in ClickUp:** Waiting ETA \-\> Backlog \-\> To-do \-\> Ready for integration. A large ticket has FE, BE and integration subtasks; move to QA only when all three are done. **Google Doc tab colours:** red \= drafting, yellow \= ready for dev, green \= done. **Meetings:** client call Monday 9 AM US Central (about 21:00 to 22:00 VN; check daylight-saving changes in early November and March). Recap every meeting in Slack.

---

## 7\. Traps we already fell into

- **Testing on production.** Production has real data. Client and team test on staging only.  
- **Editing expiry in the database.** Do not. Expiry is driven by code / BullMQ jobs.  
- **Wrong Stripe sandbox.** Staging pointed to the wrong sandbox until 29/09/2026. Check before testing payments.  
- **Removing someone from GitHub.** Removing Khoi broke the staging deploy key (07/2026). Check deploy keys before offboarding anyone.  
- **Stripe webhook testing.** `Stripe-Signature` is generated per delivery; you cannot type it into Swagger. Use a real test event or Stripe CLI.  
- **Bank transfers do not fund the Stripe balance.** Low balance makes payouts fail.  
- **"Quantity" ambiguity.** See 3.2. Specs that just say "quantity" cause bugs.  
- **Big deploy.** Dev is \~200 commits ahead of production (since 04/2026), with new Redis/BullMQ, possible schema drift, Terraform issues, and live users. Deploy needs its own plan and checklist from Namtech.  
- **Wireframes read as final UI.** Steve comments on wireframes as if they were design. Label them "wireframe" and keep them black, grey and white.  
- **Client asks for new features mid-UAT.** Write it down in a doc/ticket, estimate separately, wait for client confirmation. MVP launch comes first.  
- **Invoice PDF page breaks.** HTML-to-PDF does not break pages like Google Docs; current fix covers simple transactions only.

---

## 8\. Known gaps in the product (not promised)

From the Phase 2 known limitations (14/10/2026) and backlog:

- Price warning does not follow the rule yet (fix estimated 8.5h).  
- Search only matches vintage year (fix: search by Master Cask Name \+ Vintage, 2.5h).  
- Admin cannot delete a distillery cleanly; classification data in Manage Cask is wrong; admin view does not refresh after rejecting payment proof.  
- Seller portfolio / inventory view: on hold until after launch (Jon, 15/10/2026). In v2 backlog, may be cut.  
- Warehouse flow, Refund & Cancellation: spec exists with many open questions; not scheduled.