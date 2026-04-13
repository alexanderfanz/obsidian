# Order Lifecycle

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

The order is the central business process in the platform, touching nearly every service. This page documents the **complete flow** — every status, transition, code path, edge case, and notification.

## Order Model

Defined in [[e-com-lib]] (`pkg/model/model_gen.go`, generated from GraphQL schema). The `Order` struct has **54 pointer fields**. All prices are `*int` in **cents**.

Key field groups:
- **Identity**: `EntityID` (UUID), `Key` (`"d#"` always), `OrderNumber` (sequential, assigned post-submission)
- **Status**: `Status` (OrderStatusType), `DeclinedFromAdmin` (bool), `ErrorReason`
- **Customer**: `Email`, `DateOfBirth`, `OrderType` (ADULT/MEDICAL), `OrderMedicalID`, `BuyerAcceptsMarketing`
- **Items**: `LineItems` (products with variants), `DiscountItems` (applied promotions)
- **Pricing** (all cents): `SubtotalPrice`, `TotalLineItemsPrice`, `TotalDiscounts`, `TotalTax`, `TotalDelivery`, `TotalServiceFee`, `TotalPrice`
- **POS IDs**: `TreezOrderID`, `TreezOrderNumber`, `JaneCartID`, `JaneOrderID`, `BlazeCartID`, `DutchieOrderID`
- **Payment**: `PaymentDetails` (type + provider-specific detail struct), `TreezPayTicketIdsByStore`
- **CRM**: `AlpineiqRedemptionID`, `AlpineiqTemplateID`, `StickyTemplateID`
- **Audit**: `ActivityLogs` (append-only), `CreatedAt`, `UpdatedAt`, `InvoiceDate`, `ClientDetail`

## Status Enum — 12 Values

| Status | Description | Set by |
|--------|-------------|--------|
| `ABANDONED_CART` | Cart created, not yet submitted | [[e-com-srv]] (initial state on every UpdateCart) |
| `PENDING` | Submitted, awaiting processing | [[e-com-order-worker]] (invoice assigned) |
| `PROCESSING` | Being prepared by dispensary | [[api-srv]] (Blaze/Jane webhooks) |
| `APPROVED` | Approved for fulfillment | [[e-com-srv]] (admin action), [[e-com-app-worker]] (Treez auto-approval) |
| `ASSIGNED` | Assigned to delivery driver | [[api-srv]] (OnFleet TaskAssigned) |
| `READY` | Ready for pickup or dispatch | [[api-srv]] (Jane `ready_for_pickup`) |
| `DELIVERY_STARTED` | Driver en route | [[api-srv]] (Treez `OUT_FOR_DELIVERY`, OnFleet TaskStarted) |
| `DELIVERY_FINISHED` | Delivery route completed | [[api-srv]] (Jane `delivery_finished`, OnFleet TaskCompleted) |
| `DELIVERED` | Successfully completed | [[api-srv]] (Treez `COMPLETED`, Jane `finished`) |
| `DECLINED` | Rejected (POS, payment, admin, or delivery failure) | Multiple services |
| `VOID` | Cancelled by customer or admin | [[e-com-srv]] (`customerCancelOrder`) |
| `REFUNDED` | Fully refunded | Admin action |

**No formal state machine in code.** Transitions are enforced by each service individually. The only hard rule in [[e-com-srv]]: from `DECLINED`, only `PENDING` is allowed.

---

## Complete State Machine

```
                    ┌──────────────────┐
                    │  ABANDONED_CART   │  ← Every cart starts here
                    └────────┬─────────┘
                             │ SubmitCart (publishes SNS)
                             │ + order-worker processing
                             ▼
                    ┌──────────────────┐
               ┌──▶ │     PENDING      │ ◀── Only valid transition from DECLINED
               │    └──┬──────────┬────┘
               │       │          │
               │  (Admin/auto     (Admin/customer
               │   approves)       declines)
               │       │          │
               │       ▼          ▼
               │  ┌──────────┐  ┌──────────────┐
               │  │ APPROVED │  │   DECLINED   │ ◀── OnFleet TaskFailed
               │  └────┬─────┘  └──────────────┘     Treez CANCELED
               │       │                              Jane dismissed/cancelled
               │       ├──────────────────┐           Customer cancel (→ VOID)
               │       │                  │
               │       ▼                  ▼
               │  ┌──────────┐      ┌──────────┐
               │  │ ASSIGNED │      │  READY   │ ◀── Jane ready_for_pickup
               │  └────┬─────┘      └──────────┘     Treez PACKED_READY
               │       │
               │       ▼
               │  ┌───────────────────┐
               │  │ DELIVERY_STARTED  │ ◀── OnFleet TaskStarted
               │  └────────┬──────────┘     Treez OUT_FOR_DELIVERY
               │           │
               │           ▼
               │  ┌───────────────────┐
               │  │ DELIVERY_FINISHED │ ◀── OnFleet TaskCompleted
               │  └────────┬──────────┘     Jane delivery_finished
               │           │
               │           ▼
               │  ┌──────────────┐
               │  │  DELIVERED   │ ◀── Treez COMPLETED
               │  └──────────────┘     Jane finished
               │
               │  ┌──────────────┐
               │  │ PROCESSING   │ ◀── Blaze accepted, Jane with_review
               │  └──────────────┘
               │
               │  ┌──────────────┐
               │  │    VOID      │ ◀── Customer cancel
               │  └──────────────┘
               │
               │  ┌──────────────┐
               └──│  REFUNDED    │ ◀── Admin action
                  └──────────────┘
```

---

## Phase 1: Cart (ABANDONED_CART)

### Creation

Carts are created implicitly on the first `UpdateCart` call. No dedicated "create" mutation. If no `EntityID`, one is generated. `Key` is always `"d#"`. Status is always `ABANDONED_CART`.

Four POS-specific cart mutations exist:

| Mutation | POS | Price calculation | External calls |
|----------|-----|-------------------|---------------|
| `updateCart` | Generic/Blaze | Local (subtotal, tax, promotions) | Optional Blaze `FindUser` for first-time-patient check |
| `updateTreezCart` | Treez | **Treez owns prices** — `PreviewTicket` or `UpsertTicket` returns totals | Treez API (customer lookup/creation, ticket preview/draft) |
| `updateDutchieCart` | Dutchie | **Dutchie owns prices** — all totals imported | Dutchie GraphQL API |
| `updateJaneCart` | Jane | **No calculation** — passthrough | None (Jane owns prices server-side) |

### Price Calculation (Generic/Blaze path)

On every `updateCart` call in [[e-com-srv]]:

1. **Subtotal** = sum of (variant prices × quantity) for all line items
2. **Tax** = subtotal × tax percentage (or flat rate) if `store.TaxActive`
3. **Clear automatic discounts** (keeps manual/coupon discounts)
4. **Apply coupon discount** if `DiscountItems` present: `AMOUNT` → fixed cents, `PERCENTAGE` → subtotal × rate / 100
5. **First-time patient promotion** — if no discounts and Blaze configured: check if customer is new via Blaze API, apply if so
6. **Total** = subtotal + delivery + tax - discounts

### Treez Cart Flow

The Treez path (`updateTreezCart`) is significantly more complex:

1. Customer resolution: lookup by email → phone → cached TreezIds → create new if needed
2. Ticket request mapping via `MapOrderToTreezTicket()` from [[e-com-lib]]
3. **Preview mode** (TreezPay disabled): `PreviewTicket()` — validation only, no ticket created
4. **Draft ticket mode** (TreezPay enabled): `UpsertTicket()` — creates/updates draft ticket
   - Expired tickets (>20 min) are abandoned
   - `CANNOT_MODIFY_TICKET` error: removes ticket ID, creates new (duplicate ticket bug path)
5. Price reconciliation: Treez response overwrites all totals (TotalPrice, TotalTax, SubtotalPrice, fees)
6. Discount reconciliation: if Alpine IQ discounts were requested but Treez didn't apply them, they're removed from the order

### TreezPay Duplicate Ticket Bug

**Root cause**: Two concurrent `updateTreezCart` calls read stale request payload (no ticket ID yet). Both call `UpsertTicket(nil, ...)` → two tickets with same `ExternalOrderNumber`. The order is persisted AFTER ticket creation, so Request B doesn't see Request A's ticket.

**Secondary path**: `CANNOT_MODIFY_TICKET` → removes ticket ID → creates new → async ticket removal may not complete before next check.

**Status**: Not fixed as of last commit. See `docs/TREEZ_DUPLICATE_TICKET_ENTITY_ID_BUG.md` in e-com-srv.

### Payment Setup (before submission)

Payment is set up during the cart phase via separate mutations:

| Provider | Mutation | Flow |
|----------|----------|------|
| **AeroPay** | `addAeropayPaymentDetailToOrder` | Fetch transaction by ID from AeroPay API, store details on order |
| **Stronghold** | `createStrongholdPayLink` | Customer resolution (3-step cascade), create PayLink with optional tipping, return URL |
| **Swifter** | `createSwifterSession` → `submitSwifterPayment` | Two-step OAuth: create session → submit order. Idempotent session creation. |
| **TreezPay** | Integrated into `updateTreezCart` | Draft ticket created during cart update. `getTreezPayToken` provides OAuth token for frontend iframe. |

### ID Verification (during checkout)

Two providers supported:

- **Berbix**: `createBerbixClientToken` → frontend modal → `getBerbixTransactionResult` → extracts DL number, DOB, expiry → persists to account + Cognito
- **IDScan**: `idScanVerification` → validates document confidence (≥70), face match (≥70), anti-spoofing (≥70), age (≥21). Failure → Cognito updated with error status.

---

## Phase 2: Submission

### Standard Checkout (`submitCart`)

`SubmitCart` is surprisingly thin — it **only publishes an SNS event**:

```
publishSNSEvent(OrderCompletedEventType, {account_id, store_id, entity_id})
```

No status change, no payment processing, no POS submission. The order stays `ABANDONED_CART` in DynamoDB. All post-submission work happens in the workers.

### Kiosk Checkout (`submitKioskCheckout`)

Unlike standard checkout, kiosk **synchronously** creates the Treez order:
1. Customer resolution (lookup/create in Treez)
2. `MapOrderToTreezTicket` + `UpsertTicket`
3. Store `TreezOrderID` and `TreezOrderNumber`
4. Publish `OrderKioskCompletedEventType` SNS event

---

## Phase 3: Post-Purchase Pipeline (async)

### order-worker: `order/completed`

[[e-com-order-worker]] processes the SNS event through a 6-step processor chain:

| Step | Processor | What it does | Error behavior |
|------|-----------|-------------|----------------|
| 1 | **fill-initial-order-data** | Load order, assign invoice number (atomic DynamoDB counter), set status to `PENDING`, append activity log | **Fatal** — only processor that fails the pipeline |
| 2 | **upsert-account** | Create or update customer [[Account]]: aggregate lifetime spend, copy medical ID/DOB | Non-fatal (returns nil) |
| 3 | **send-data-to-instrumentation** | New Relic custom event: `{EventType: "Order", Amount, AccountID, StoreID, StoreName}` | Non-fatal |
| 4 | **publish-place-order-klaviyo** | Publish SNS if Klaviyo app enabled for store | Non-fatal |
| 5 | **publish-process-treez** | Publish SNS if Treez app enabled AND `AutomaticApproval` = true | Non-fatal |
| 6 | **notify-user** | Publish SNS for status change notification (always fires) | Non-fatal |

**Invoice counter bug**: If order upsert succeeds but counter increment fails, the processor returns nil (success). Next order gets a duplicate invoice number.

**Dead code**: Blaze and Jane processors exist in the codebase but are NOT wired into the pipeline.

### order-worker: `order/kiosk-completed`

Runs only steps 1-3 (fill data, upsert account, instrumentation). Skips all integration fan-out and notifications. Kiosk orders are in-store — customer is physically present.

### app-worker: `place-order-treez`

[[e-com-app-worker]] chains 3 processors:

**PlaceOrderTreez**:
1. Load store/order/account, validate Treez app enabled + AutomaticApproval
2. Customer resolution (3-path: known → lookup → create). DOB fallback: `now - 22 years`
3. Verification photo upload: builds plan based on new vs returning customer, store feature flags (`AllowIDVerificationForDelivery`, `AllowIDVerificationForPickup`). Fetches from S3, base64-encodes, uploads to Treez document API. Non-fatal on failure.
4. Payment resolution: Stronghold charge ID, AeroPay transaction ID, TreezPay credit detection
5. `UpsertTicket` — creates or updates Treez ticket. TreezPay credit orders skip this (ticket already exists).
6. Clean up unused draft tickets from other stores
7. Set status to `APPROVED`, persist order

**AlpineIQDiscount** (chained after Treez):
- If `AlpineiqRedemptionURL` set: call `ApplyDiscountRedemption` to finalize loyalty discount
- Fatal on API/DB errors (entire chain retries, re-running PlaceOrderTreez)

**StickyDiscount** (chained after AlpineIQ):
- Checks `StickyCustomerID` on account, parses discount ID from rewards ID format
- **Bug**: `isValidStickyApp` returns `false` unconditionally — processor is effectively disabled for all stores

### app-worker: `place-order-blaze`

1. Verify stock via Blaze API for each line item. If all out of stock: publish `OrderOutOfStockNotifyEventType`, return.
2. Customer management: find/register/login in Blaze (hardcoded password: `5Tv6b4ZfFT`)
3. Build Blaze cart with line items, payment mapping, delivery type, promotions
4. `SubmitCart` to Blaze
5. Save `BlazeOrder` mapping record (key: `blaze#`)

### app-worker: `place-order-jane`

1. Save `JaneOrder` mapping record (key: `j#`)
2. If no `JaneOrderID`: set `TimeWindow = "NOT SET"`, return
3. Retrieve Jane cart, map discounts (promo code + CRM redemptions), copy store/notes data
4. If AeroPay preauthorization exists on Jane cart: map to order payment details
5. Persist enriched order

### app-worker: `place-order-klaviyo`

Maps order to Klaviyo "Placed Order" event: profile info, line items, discounts, brands, categories. Sends via Klaviyo API.

---

## Phase 4: Status Updates (Webhooks)

### Treez Webhooks (`POST /treez/order-updated`)

[[api-srv]] processes Treez status changes:

**Filters** (any failing → silent 200 OK):
- Empty `ExternalOrderNumber`
- Revenue source not `e-commerce`
- No `TICKET_STATUS` in event types
- Order in `ABANDONED_CART` status
- `DeclinedFromAdmin` flag set (clears flag and returns)

**Status mapping**:

| Treez Status | Internal Status | Notify (SNS) | LisTrack Email |
|---|---|---|---|
| `VERIFICATION_PENDING` | *(unchanged)* | No | — |
| `AWAITING_PROCESSING` | *(unchanged)* | No | — |
| `IN_PROCESS` | *(unchanged)* | No | — |
| `PACKED_READY` | `APPROVED` | Yes | — |
| `OUT_FOR_DELIVERY` | `DELIVERY_STARTED` | Yes | — |
| `COMPLETED` | `DELIVERED` | Yes | `OrderCompletedPickup` |
| `CANCELED` | `DECLINED` | Yes | `OrderRejectedPickup` |

**Side effects per status**:
- Activity log appended (user: "API")
- LisTrack email (delivery orders only, non-fatal)
- Alpine IQ SMS (if configured): confirmation, ready, cancelled, completed messages
- Alpine IQ discount reversion (on `CANCELED` if `AlpineiqRedemptionID` exists)
- SNS `OrderStatusChangedNotify` event (if `notify = true`)

### Jane Webhooks (`POST /jane/order-updated/`)

Bearer token auth. 11 status values:

| Jane Status | Internal Status | LisTrack (Pickup) | LisTrack (Delivery) |
|---|---|---|---|
| `verification` | `PENDING` | — | — |
| `pending` | `PENDING` | — | — |
| `dismissed` | `DECLINED` | `OrderRejectedPickup` | `OrderRejectedDelivery` |
| `cancelled` | `DECLINED` | `OrderRejectedPickup` | `OrderRejectedDelivery` |
| `finished` | `DELIVERED` | `OrderCompletedPickup` | `OrderCompletedDelivery` |
| `finished_without_review` | `DELIVERED` | `OrderCompletedPickup` | `OrderCompletedDelivery` |
| `ready_for_pickup` | `READY` | `OrderApprovedPickup` | `OrderApprovedPickup` |
| `delivery_started` | `DELIVERY_STARTED` | `OrderOnRoute` | `OrderOnRoute` |
| `delivery_finished` | `DELIVERY_FINISHED` | `OrderCompletedDelivery` | `OrderCompletedDelivery` |
| `with_review` | `PROCESSING` | — | — |
| `staff_member_review` | `PROCESSING` | — | — |

Delivery timestamps enriched from Jane API (non-fatal). LisTrack email failure is **fatal** (returns 500).

### Blaze Webhooks (`POST /blaze/order-updated`)

Simplest handler. Maps 3 statuses:
- `Accepted` → `PROCESSING`
- `Canceled` → `DECLINED`
- *(default)* → `DELIVERED`

**No side effects** — only DynamoDB update. No email, SMS, SNS, or activity log.

### OnFleet Delivery Webhooks (6 events)

All return 200 to OnFleet regardless of errors (errors logged to Sentry only).

| Event | Status Change | Activity Log | LisTrack | Treez Sync |
|-------|-------------|-------------|----------|-----------|
| **TaskStarted** | → `DELIVERY_STARTED` | Yes | `OrderOnRoute` | Update ticket: `OutOfDelivery` |
| **TaskArriving** | Nested: `→ Arriving` | No | — | — |
| **TaskCompleted** | → `DELIVERY_FINISHED` | Yes | `OrderCompletedDelivery` | Update ticket: `Completed` + payment details |
| **TaskFailed** | → `DECLINED` | Yes | `OrderRejectedDelivery` | **RemoveTicket** (full removal) |
| **TaskDelayed** | Nested: `→ Delayed` | No | — | — |
| **TaskAssigned** | → `ASSIGNED` | Yes | — | — |

**TaskFailed is the most destructive** — multi-step revert:
1. Set `DECLINED`, clear OnFleet reference, append activity log, persist
2. Remove Treez ticket via `RemoveTicket()`, clear `TreezOrderID`/`TreezOrderNumber`, persist again
3. Send rejection email (non-fatal)

**TaskCompleted payment mapping**:
- `Check` → `PaymentMethodCheck`
- `CreditCard` → `PaymentMethodCredit`
- `DebitCard` → `PaymentMethodDebit`
- `Aeropay` → `PaymentMethodAeropay`
- *(default)* → `PaymentMethodCash`

---

## Phase 5: Admin Actions

### UpdateOrderStatus (via [[gap-dashboard]])

Admin transitions in [[e-com-srv]] with guards:
- Same-status rejected
- From `DECLINED`: only `PENDING` allowed

| Transition | What happens |
|------------|-------------|
| → `APPROVED` | Barcode validation (unless bypassed), create Treez ticket + OnFleet task (if delivery). OnFleet failure triggers Treez rollback. |
| → `DECLINED` | Set `DeclinedFromAdmin = true`, remove Treez ticket, remove OnFleet task |
| → `DELIVERED` / `DELIVERY_FINISHED` | Complete Treez ticket with payment details, send LisTrack email |

All transitions: append activity log (user = admin email), publish SNS notification, persist.

### UpdateOrderDetails (Pending orders only)

Editable: delivery address, payment method, promo code, notes, line items, time window. Changes sent to Treez via `PreviewTicket` for validation; Treez prices overwrite local totals.

### Customer Cancellation

`customerCancelOrder`: cancels Jane order → removes Treez ticket → removes OnFleet task → sets `VOID`. Each step independent.

### Admin Dashboard UI

[[gap-dashboard]] shows orders in Kanban (by status lanes) or table view. Order detail has 4 tabs: Data (editable in PENDING), Additional Data (POS/driver info), Activity Logs, Raw JSON. Status buttons shown based on current state.

---

## Phase 6: Notifications

[[e-com-notification-worker]] handles 5 event types:

| Event | Template key | Subject | Recipient |
|-------|-------------|---------|-----------|
| `order/confirm_notify` | `order/confirm_notify` | `"{Store} Order #{N}: Confirmed"` | Customer |
| `order/delivered_notify` | `order/delivered_notify` | `"{Store} Order #{N}: Delivered"` | Customer |
| `order/out_of_stock_notify` | `order/out_of_stock_notify` | `"{Store} Order #{N}: Declined"` | Customer |
| `order/status_changed` | **Dynamic** (see below) | `"{Store} Order #{N}: {Action}"` | Customer |
| `contact/business_owner` | `contact/business_owner` | `"{Name} want to contact..."` | **Merchant** |

### Status → Template Mapping (`status_changed`)

| Order Status | Template Key | Subject Action |
|---|---|---|
| `PENDING` | `order/confirm_notify` | "Confirmed" |
| `DECLINED` | `order/declined_notify` | "Cancelled" |
| `DELIVERED` | `order/completed_notify` | "Completed" |
| `APPROVED` | `order/ready_notify` | "Ready" |
| `DELIVERY_STARTED` | `order/on_route_notify` | "On Route" |
| *(other)* | *(no email)* | — |

### Known Bugs in Notifications

- **Delivery processor S3 key bug**: Uses `{storeID}-{storeKey}-{notificationKey}` instead of `{accountID}-{storeID}-{notificationKey}`. Templates likely never load for delivery notifications.
- **Missing template functions**: `formatPrice` and `inc` only registered for ConfirmOrder and StatusChanged. Delivery and OutOfStock templates using these will **panic at runtime**.
- **Inconsistent nil handling**: ConfirmOrder/StatusChanged treat nil `EmailNotificationActive` as "skip email". Delivery/OutOfStock treat nil as "proceed with email".
- **Out-of-stock date format**: Uses `"2006"` (year only) instead of full date — likely a bug.
- **Nil pointer risks**: OutOfStock dereferences `*order.Email` directly (panics on nil). All processors access `Variants[0]` without bounds checking.

---

## Payment Types

| Type | Enum | Provider Detail Struct | Key Fields |
|------|------|----------------------|------------|
| Cash | `CASH` | — | — |
| Debit | `DEBIT_CARD` | — | `CardType`, `LastFour` |
| Credit | `CREDIT_CARD` | — | `CardType`, `LastFour` |
| Check | `CHECK` | — | — |
| AeroPay | `AEROPAY` | `AeroPayPaymentDetail` | `TransactionID`, `Amount`, `MerchantName` |
| Swifter | `SWIFTER` | `SwifterPaymentDetail` | `SessionID`, `OrderID`, `Status` (STARTED/COMPLETED) |
| Stronghold | `STRONGHOLD` | `StrongholdPaymentDetail` | `PaylinkID`, `ChargeID`, `Status` (CREATED/USED/CANCELED) |
| TreezPay | `TREEZPAY` | `TreezPayPaymentDetail` | `ProcessorName`, `PaymentMethod`, `PaymentID` |

---

## Event Flow Summary

```
CUSTOMER CHECKOUT
  treez-tlp → updateTreezCart (GraphQL) → e-com-srv
                                              │
  treez-tlp → submitCart (GraphQL) ──────────▶ SNS: OrderCompleted
                                                    │
ASYNC PIPELINE                                      ▼
  e-com-order-worker ── fill data + invoice ── SNS fan-out:
    ├──▶ e-com-app-worker: place-order-treez/blaze/jane/klaviyo
    └──▶ e-com-notification-worker: status_changed

POS WEBHOOKS
  Treez/Jane/Blaze/OnFleet ──▶ api-srv ── DynamoDB update
                                    ├──▶ Alpine IQ SMS
                                    ├──▶ LisTrack email
                                    └──▶ SNS: OrderStatusChangedNotify
                                              │
                                              ▼
                                    e-com-notification-worker

ADMIN ACTIONS
  gap-dashboard → updateOrderStatus (GraphQL) → e-com-srv
                                                    ├──▶ Treez ticket create/remove
                                                    ├──▶ OnFleet task create/remove
                                                    ├──▶ LisTrack email
                                                    └──▶ SNS: OrderStatusChangedNotify
```

---

## Known Issues & Risks

| Issue | Severity | Location |
|-------|----------|----------|
| TreezPay duplicate ticket race condition | High | [[e-com-srv]] `updateTreezCart` |
| Sticky discount processor always returns false (disabled) | High | [[e-com-app-worker]] `isValidStickyApp` |
| Invoice counter can produce duplicates on failure | Med | [[e-com-order-worker]] `fill-initial-order-data` |
| Delivery notification S3 key construction bug | Med | [[e-com-notification-worker]] `delivery.go` |
| Missing `formatPrice`/`inc` in Delivery/OutOfStock templates | Med | [[e-com-notification-worker]] |
| Inconsistent `EmailNotificationActive` nil handling | Med | [[e-com-notification-worker]] |
| No retry for failed CRM subscriptions | Med | [[e-com-srv]] `SubscribeToCrm` |
| DynamoDB upsert failure after Treez+OnFleet creation = inconsistent state | Med | [[e-com-srv]] `UpdateOrderStatus` |
| Hardcoded Blaze password for all customers | Low | [[e-com-app-worker]] |
| All operations use 5-minute global timeout, no per-op tuning | Low | [[e-com-srv]] |

---

*Sources: [[raw/order-lifecycle/e-com-lib-order.md]], [[raw/order-lifecycle/e-com-srv-order.md]], [[raw/order-lifecycle/api-srv-order.md]], [[raw/order-lifecycle/e-com-order-worker-order.md]], [[raw/order-lifecycle/e-com-app-worker-order.md]], [[raw/order-lifecycle/e-com-notification-worker-order.md]], [[raw/order-lifecycle/treez-tlp-order.md]], [[raw/order-lifecycle/gap-dashboard-order.md]]*
