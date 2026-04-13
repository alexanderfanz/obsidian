# e-com-order-worker — Post-Purchase Order Processing Pipeline

---

## 1. order/completed Event — Full Processor Chain

### Event Ingestion

**SQS Message Format:**
```json
{
  "MessageAttributes": {
    "event_type": { "Value": "order/completed" }
  },
  "Message": "{\"account_id\":\"...\",\"store_id\":\"...\",\"entity_id\":\"...\"}"
}
```

**Parsing** (`internal/app/parser.go`):
1. SQS message body is unmarshaled to `worker.SQSMessage`
2. `MessageAttributes["event_type"].Value` is extracted as the `MessageType`
3. `Message` string is passed through as the raw event

**Predicate** (`internal/app/predicate.go`):
```go
func WhenOrderCompleted(messageType worker.Message) bool {
    return messageType == worker.OrderCompletedEventType
}
```

**Mapper** (`internal/app/mapper.go`):
The raw JSON message string is unmarshaled into:
```go
type OrderCompleteEvent struct {
    AccountID string `json:"account_id"`
    StoreID   string `json:"store_id"`
    EntityID  string `json:"entity_id"`
}
```

### Processor Chain (exact order from `internal/app/manager.go`)

#### Step 1: fill-initial-order-data (`internal/processor/fill_order_data.go`)

**Purpose:** Load the full order, assign invoice number, set status to PENDING.

**Reads:**
- Store config: `p.store.Get(ctx, util.GetKey(accountID, storeID, storeKey))` — resolves the per-store DynamoDB table name (`store.DynamoOrderTableName`)
- Order: `p.order.GetOrderByID(ctx, tableName, entityID)` — full order from DynamoDB
- Invoice counter: `p.invoice.Get(ctx, tableName)` — current invoice counter value from DynamoDB

**Writes:**
- **Order update** (DynamoDB upsert to `store.DynamoOrderTableName`):
  - `InvoiceDate` = current UTC timestamp
  - `UpdatedAt` = current UTC timestamp
  - `OrderNumber` = `int(invoice.Value)` (current counter value)
  - `Status` = `model.OrderStatusTypePending`
  - `ActivityLogs` = single entry:
    ```go
    {
        Action:    "UpdateOrderStatus",
        User:      "API",
        Status:    model.OrderStatusTypePending,
        Timestamp: now,
    }
    ```
  - **Note:** `ActivityLogs` is set as a new slice, not appended — this overwrites any prior activity logs.
- **Invoice counter increment** (DynamoDB upsert): `p.invoice.Upsert(ctx, invoice.Value+1, tableName)`

**Error behavior:** Returns error on any failure — **this is the only processor that fails the pipeline.**

**Invoice counter failure edge case:** If the order upsert succeeds but the invoice counter increment fails, the error is logged and captured to Sentry, but the processor returns `nil` (success). This means the next order could get a duplicate invoice number.

#### Step 2: upsert-account (`internal/processor/upsert_account.go`)

**Purpose:** Create or update the customer account with order data.

**Reads:**
- Store config: `p.store.Get(ctx, ...)` — for DynamoDB table name
- Order: `p.order.GetOrderByID(ctx, tableName, entityID)` — full order
- Account: `p.account.GetByEmail(ctx, *order.Email, tableName)` — existing account lookup by email

**Writes (DynamoDB upsert to `store.DynamoOrderTableName`):**

*If account exists:*
- `TotalOrderExpense` += `*order.TotalPrice` (aggregated lifetime spend in cents)
- Medical info updated via `updateAccountMedicalInfo()`

*If account does not exist (new customer):*
- New `model.Account` created:
  - `EntityID` = new UUID
  - `Key` = `"c#"` (account key prefix)
  - `FirstName` = `order.DeliveryAddress.FirstName`
  - `LastName` = `order.DeliveryAddress.LastName`
  - `Email` = `order.Email`
  - `Phone` = `order.DeliveryAddress.Phone`
  - `TotalOrderExpense` = `order.TotalPrice`
  - `HasAccount` = `false`
  - `CreatedAt` = now
  - `UpdatedAt` = now
- Medical info applied via `updateAccountMedicalInfo()`

**Medical info copy logic (`updateAccountMedicalInfo`):**
- If `order.OrderType` is set → `account.Type` = `order.OrderType`
- If `order.DateOfBirth` is set → `account.DateOfBirth` = `order.DateOfBirth`
- If `order.OrderMedicalID` is set → `account.MedicalID` = `{Number: *order.OrderMedicalID}`
  - If `order.OrderMedicalIDExpirationDate` is also set → `account.MedicalID.ExpirationDate` = that value
  - **Note:** There's a TODO comment: "check if it is ok to have a zero value for the expiration date (1970-01-01)" — when medical ID is present but expiration is nil, `ExpirationDate` defaults to zero-value (epoch).

**Error behavior:** Returns `nil` always — never fails the pipeline. Store fetch errors are logged to Sentry. Account fetch errors and upsert errors are logged but swallowed.

#### Step 3: send-order-data-to-instrumentation (`internal/processor/send_data_to_instrumentation.go`)

**Purpose:** Send a custom analytics event to New Relic for order tracking.

**Reads:**
- Store config: `p.store.Get(ctx, ...)` — for table name and store name
- Order: `p.order.GetOrderByID(ctx, tableName, entityID)` — for `TotalPrice`

**Publishes — New Relic Custom Event:**
```go
newrelic.Event{
    EventType: newrelic.EventOrder,  // "Order"
    Amount:    float64(*order.TotalPrice) / 100,  // cents → dollars
    AccountID: data.AccountID,
    StoreID:   data.StoreID,
    StoreName: *store.Name,
}
```
Sent via: `p.instrumentation.Send(ctx, []interface{}{event})`

**Attributes:**
| Attribute | Type | Description |
|-----------|------|-------------|
| `EventType` | string | Always `"Order"` (constant `newrelic.EventOrder`) |
| `Amount` | float64 | Order total in dollars (TotalPrice / 100) |
| `AccountID` | string | Tenant account ID |
| `StoreID` | string | Store ID |
| `StoreName` | string | Human-readable store name |

**Error behavior:** Returns `nil` always — store/order fetch failures and New Relic send failures are logged but swallowed.

#### Step 4: publish-place-order-klaviyo (`internal/processor/publish_place_order_klaviyo.go`)

**Purpose:** Conditionally publish an SNS event to trigger Klaviyo email marketing.

**Conditional checks:**
1. `p.app.GetByHandler(ctx, appKey, "klaviyo")` — fetch Klaviyo app config from S3
2. If app is nil → return nil (Klaviyo not configured for this store)
3. Type-assert to `model.Klaviyo` — if fails, log warning and return nil
4. If `klaviyoApp.Status` is nil or false → return nil (Klaviyo disabled)

**SNS Publish (via `publishEvent` helper in `common.go`):**
```go
&sns.PublishInput{
    Message:  `{"account_id":"...","store_id":"...","entity_id":"..."}`,
    Subject:  "order complete",
    TopicArn: config.TopicArn,
    MessageAttributes: {
        "event_type": worker.PlaceOrderKlaviyoEventType,
        "publisher":  "gapcommerce",
    },
}
```

**Error behavior:** Returns `nil` always. App fetch errors, type assertion failures, and SNS publish failures are all logged but swallowed.

#### Step 5: publish-process-treez (`internal/processor/publish_place_order_treez.go`)

**Purpose:** Conditionally publish an SNS event to trigger Treez POS integration.

**Conditional checks (3-gate):**
1. `p.app.GetByHandler(ctx, appKey, "treez")` — fetch Treez app config from S3
2. If app is nil → return nil (Treez not configured)
3. Type-assert to `model.Treez` — if fails, log warning and return nil
4. If `treezApp.Status` is nil or false → return nil (Treez disabled)
5. **If `treezApp.AutomaticApproval` is nil or false → return nil** (manual approval mode — order must be approved in dashboard before Treez sync)

**SNS Publish:**
```go
MessageAttributes: {
    "event_type": worker.ProcessTreezEventType,
    "publisher":  "gapcommerce",
}
```
Same `OrderCompleteEvent` payload as Klaviyo.

**The `AutomaticApproval` flag:** When enabled, orders are automatically forwarded to Treez on completion. When disabled (or nil), orders sit in PENDING status until an admin manually approves them in the dashboard, at which point a different code path triggers the Treez sync. This is a per-store setting.

**Error behavior:** Returns `nil` always.

#### Step 6: notify-user (`internal/processor/notify_user.go`)

**Purpose:** Publish an SNS event to trigger order status notification to the customer.

**No conditional checks** — this always fires for `order/completed`.

**SNS Publish:**
```go
MessageAttributes: {
    "event_type": worker.OrderStatusChangedNotifyEventType,
    "publisher":  "gapcommerce",
}
```
Same `OrderCompleteEvent` payload.

**Error behavior:** Returns `nil` always. SNS publish failure is logged but swallowed.

#### Final Step: SQS Message Deletion (`internal/app/deleter.go`)

After all processors complete, the SQS message is deleted:
```go
sqs.DeleteMessage(ctx, &sqs.DeleteMessageInput{
    QueueUrl:      queueName,
    ReceiptHandle: handle,
})
```

If any processor returns an error (only `fill-initial-order-data` can), the message is **not deleted** and will be retried by SQS based on the queue's visibility timeout and redrive policy.

---

## 2. order/kiosk-completed Event — Differences

**Predicate:**
```go
func WhenOrderKioskCompleted(messageType worker.Message) bool {
    return messageType == worker.OrderKioskCompletedEventType
}
```

**Same mapper** — uses `MapToOrderCompleted()`, same `OrderCompleteEvent` struct.

**Processor chain (3 steps vs 6):**

| Step | Processor | In order/completed? | In order/kiosk-completed? |
|------|-----------|---------------------|---------------------------|
| 1 | fill-initial-order-data | Yes | Yes |
| 2 | upsert-account | Yes | Yes |
| 3 | send-order-data-to-instrumentation | Yes | Yes |
| 4 | publish-place-order-klaviyo | Yes | **No** |
| 5 | publish-process-treez | Yes | **No** |
| 6 | notify-user | Yes | **No** |

**What's skipped and why:** Kiosk orders skip all downstream integration fan-out (Klaviyo, Treez) and customer notification. The rationale is that kiosk orders originate from an in-store kiosk — the customer is physically present, so email notifications are unnecessary, and the POS integration may be handled differently for in-store orders.

The kiosk path still performs:
- Invoice number assignment (same atomic counter)
- Status set to PENDING
- Account upsert (same lifetime spend aggregation)
- New Relic instrumentation event

---

## 3. Processor Chain Details — Per-Processor Summary

| # | Name | Reads | Writes/Publishes | Error Behavior |
|---|------|-------|------------------|----------------|
| 1 | fill-initial-order-data | Store (DynamoDB/S3), Order (DynamoDB), Invoice counter (DynamoDB) | Order upsert (DynamoDB), Invoice counter +1 (DynamoDB) | **Returns error — fails pipeline** |
| 2 | upsert-account | Store, Order, Account by email (all DynamoDB) | Account upsert (DynamoDB) | Returns nil — continues |
| 3 | send-order-data-to-instrumentation | Store, Order (DynamoDB) | New Relic custom event (HTTP) | Returns nil — continues |
| 4 | publish-place-order-klaviyo | App config "klaviyo" (S3) | SNS: `PlaceOrderKlaviyoEventType` | Returns nil — continues |
| 5 | publish-process-treez | App config "treez" (S3) | SNS: `ProcessTreezEventType` | Returns nil — continues |
| 6 | notify-user | (none) | SNS: `OrderStatusChangedNotifyEventType` | Returns nil — continues |

---

## 4. Non-Critical Failure Handling

Every processor except `fill-initial-order-data` returns `nil` on failure:

| Processor | Failure Scenario | Behavior |
|-----------|-----------------|----------|
| **upsert-account** | Store fetch fails | Error logged + Sentry, returns nil |
| **upsert-account** | Account email lookup fails | Error logged, returns nil |
| **upsert-account** | Account upsert fails | Error logged, returns nil |
| **instrumentation** | Store or order fetch fails | Error logged, returns nil |
| **instrumentation** | New Relic send fails | Error logged, returns nil |
| **publish-klaviyo** | App fetch fails | Error logged, returns nil |
| **publish-klaviyo** | Klaviyo not configured/disabled | Silent return nil |
| **publish-klaviyo** | SNS publish fails | Error logged + Sentry, returns nil |
| **publish-treez** | App fetch fails | Error logged, returns nil |
| **publish-treez** | Treez not configured/disabled/manual-approval | Silent return nil |
| **publish-treez** | SNS publish fails | Error logged + Sentry, returns nil |
| **notify-user** | SNS publish fails | Error logged + Sentry, returns nil |

**The reason:** These processors handle optional integrations. If Klaviyo is down or a New Relic send fails, the core order processing (invoice assignment, status update) has already succeeded. Re-processing the entire message from SQS would duplicate the invoice number.

**Key implication:** There is no retry mechanism for individual processor failures. If an SNS publish to Klaviyo fails, that email is permanently lost — there is no dead-letter path or compensating action.

**Special case — invoice counter failure:** Inside `fill-initial-order-data`, the invoice counter increment (`p.invoice.Upsert(ctx, nValue, tableName)`) can fail *after* the order has been upserted with the current counter value. This failure is logged and captured to Sentry, but the processor returns `nil` (success). The consequence: the counter isn't incremented, so the **next** order will receive the same invoice number. This is a subtle bug.

---

## 5. New Relic Custom Events

**Event name:** `"Order"` (constant `newrelic.EventOrder`)

**Sent via:** New Relic Events API (`p.instrumentation.Send(ctx, []interface{}{event})`)

**Attributes:**

| Field | Type | Source | Example |
|-------|------|--------|---------|
| `EventType` | string | Constant | `"Order"` |
| `Amount` | *float64 | `order.TotalPrice / 100` | `42.50` |
| `AccountID` | string | Event payload | `"acc_123"` |
| `StoreID` | string | Event payload | `"store_456"` |
| `StoreName` | string | Store config | `"Berkeley Dispensary"` |

**Notes:**
- Amount is converted from cents (int) to dollars (float64)
- If `order.TotalPrice` is nil, amount defaults to `0.0`
- One event per order — emitted for both `order/completed` and `order/kiosk-completed`
- No integration outcome tracking (Blaze success/fail, Treez success/fail, etc.)

---

## 6. Unused/Dead Code — Processors Not Wired Into Pipeline

The following processors exist in the codebase with full implementations and factory methods on `App`, but are **not referenced in `WorkerManager()`**:

| Processor | Factory Method | File |
|-----------|---------------|------|
| `publishBlazeProcessor` | `App.PublishBlazeProcessor()` | `publish_place_order_blaze.go` |
| `publishJaneProcessor` | `App.JaneProcessor()` | `place_order_jane.go` |
| `listrackEmailProcessor` | `App.ListrackEmailProcessor()` | `send_listrack_email.go` |

**Blaze processor** (`publish_place_order_blaze.go`): Checks `apps.json` for a `"blaze"` handler with `Status == true`, then publishes `worker.ProcessBlazeEventType` to SNS. Identical pattern to Klaviyo/Treez. Returns nil on all errors.

**Jane processor** (`place_order_jane.go`): Much more complex than the others. Rather than just publishing an SNS event, it:
1. Checks `apps.json` for a `"jane"` handler with `Status == true`
2. Loads the store and order from DynamoDB
3. If `order.JaneCartID` is nil → returns nil (not a Jane order)
4. Saves a `JaneOrder` record to DynamoDB (key: `"j#"`, entity_id: JaneCartID, order_id: order EntityID)
5. If `order.JaneOrderID` is nil → sets `DeliveryDetails.TimeWindow = "NOT SET"` and returns
6. Creates a Jane API client using the app config credentials
7. Calls `janeSrv.RetrieveCart(ctx, *order.JaneOrderID)` to fetch the Jane cart
8. Maps Jane discounts onto the order:
   - Promo code discounts: iterates Jane cart products, calculates price difference × count, creates `OrderDiscountItem` with type `ProductDiscount`
   - CRM redemptions: sums up reward amounts, creates `OrderDiscountItem` with type `AmountOffOrder`
9. Copies `JaneStoreID`, `JaneStoreName`, `Notes` from the Jane cart
10. If AeroPay preauthorization exists on the Jane cart, maps it to `order.PaymentDetails`
11. Upserts the enriched order back to DynamoDB
Returns actual errors for store/order fetch failures and DynamoDB write failures (would fail the pipeline if wired in).

**LisTrack processor** (`send_listrack_email.go`): Loads the order and determines whether it's delivery or pickup based on `order.DeliveryDetails.Delivery`, then sends a transactional email via the LisTrack service using event type `LisTrackEventTypeOrderNewPickup` or `LisTrackEventTypeOrderNewDelivery`. Returns nil on all errors.

These may have been active in a previous version and subsequently removed from the pipeline, or they may be intended for future use.

---

## 7. SNS Event Summary

All SNS events use the same shared helper (`common.go:publishEvent`):

```go
Subject:  "order complete"
TopicArn: config.TopicArn  // single shared SNS topic
MessageAttributes:
    "event_type": <event-type string>
    "publisher":  "gapcommerce"
Message: JSON-serialized OrderCompleteEvent {account_id, store_id, entity_id}
```

| Event Type Constant | Condition | Downstream Consumer |
|---------------------|-----------|-------------------|
| `worker.PlaceOrderKlaviyoEventType` | Klaviyo app enabled for store | e-com-app-worker (Klaviyo processor) |
| `worker.ProcessTreezEventType` | Treez app enabled + AutomaticApproval=true | e-com-app-worker (Treez processor) |
| `worker.OrderStatusChangedNotifyEventType` | Always (order/completed only) | e-com-notification-worker |

**Not published by this worker** (despite having processors in codebase):
| Event Type | Reason |
|-----------|--------|
| `worker.ProcessBlazeEventType` | Blaze processor exists but not wired into pipeline |
| (Jane has no SNS event) | Jane processor does direct API calls + DynamoDB, not SNS fan-out |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-order-worker (master @ c55ec05)*
