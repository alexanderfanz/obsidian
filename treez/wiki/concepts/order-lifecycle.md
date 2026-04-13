# Order Lifecycle

The order is the central business process in the platform, touching nearly every service.

## State machine

```
             ┌──────────────┐
             │ Shopping Cart │  (key="d#", status=draft)
             │  (UpdateCart) │
             └──────┬───────┘
                    │ SubmitCart
                    ▼
             ┌──────────────┐
             │   Pending    │  (order-worker assigns invoice, fans out)
             └──────┬───────┘
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
    ┌──────────┐ ┌────────┐ ┌────────────┐
    │ Approved │ │Rejected│ │AbandonedCart│
    └────┬─────┘ └────────┘ └────────────┘
         │
         ▼
    ┌──────────┐
    │Completed │
    └──────────┘
```

## Service flow

### 1. Cart phase (synchronous)

**[[e-com-srv]]** handles `UpdateCart`:
- Recalculates line item prices, promotion discounts, taxes, and totals on every call
- May sync to multiple POS systems simultaneously (Treez + Jane, etc.)
- Evaluates automatic promotions; applies coupon codes via `ApplyPromotionCode`

### 2. Checkout (synchronous → async)

**[[e-com-srv]]** handles `SubmitCart`:
- Pushes order to connected POS system via handler files
- Initiates payment flow (AeroPay, Stronghold, Swifter, or TreezPay)
- Publishes SNS events for downstream processing

### 3. Post-purchase pipeline (async)

**[[e-com-order-worker]]** picks up `order/completed` SQS event:
- Assigns invoice number (atomic DynamoDB counter)
- Sets status to PENDING
- Upserts customer [[Account]] (aggregates lifetime spend, copies medical ID/DOB)
- Conditionally publishes: `place_order_klaviyo`, `process_blaze`, `process_jane`, `process_treez`, `order_status_changed_notify`

**[[e-com-app-worker]]** picks up integration events:
- Submits order to Treez/Blaze/Jane POS with provider-specific API calls
- Uploads verification photos for Treez (age compliance)
- Redeems AlpineIQ/Sticky loyalty discounts (chained after Treez placement)

**[[e-com-notification-worker]]** picks up notification events:
- Sends templated HTML emails via SES (confirmation, delivery, status change)

### 4. Status updates (webhook-driven)

**[[api-srv]]** receives inbound webhooks from POS/delivery systems:
- Treez: maps 7 statuses → internal status. Triggers Alpine IQ SMS, LisTrack email, SNS notification.
- Jane: maps 11 statuses, distinguishes delivery vs. pickup paths.
- OnFleet: 6 delivery task events. TaskFailed reverts to Declined; TaskCompleted updates Treez ticket with payment details.
- Blaze: order status change processing.

### 5. Kiosk orders

`order/kiosk-completed` events run a shorter pipeline in [[e-com-order-worker]] — enrichment only, no notifications or integration fan-out. Prevents duplicate downstream flows for in-store orders.

## Event flow diagram

```
e-com-srv (SubmitCart)
    │ SNS
    ▼
e-com-order-worker
    │ SNS (fan-out)
    ├──▶ e-com-app-worker (place-order-treez/blaze/jane/klaviyo)
    └──▶ e-com-notification-worker (order_status_changed_notify)

External POS/Delivery
    │ HTTP webhook
    ▼
api-srv
    │ SNS
    └──▶ e-com-notification-worker (status change emails)
```
