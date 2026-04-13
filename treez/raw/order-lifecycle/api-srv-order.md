# api-srv — Order Status Changes from External Systems

> This document covers every code path in `api-srv` that receives an order status change from an external system (Treez, Jane, Blaze, OnFleet), maps it to an internal `OrderStatusType`, and fires side effects (DynamoDB update, SMS, email, SNS event, Treez ticket sync).

---

## 1. Treez Order Webhook

**Endpoint:** `POST /treez/order-updated?account_id=X&store_id=Y`
**Handler:** `internal/app/treez.go` — `OrderUpdated` (line 64)
**Service:** `internal/service/treez/service.go` — `ProcessOrder` (line 83)

### 1.1 Handler Flow

1. **Request parsing** — `NewTreezOrderRequestBuilder(r).Build()` extracts `account_id` and `store_id` from query params, decodes JSON body via custom `UnmarshalJSON`. Returns 400 on failure.
2. **Empty external order number** — If `req.Body.Data.ExternalOrderNumber` is empty, logs warning and returns 200 OK (silent skip).
3. **Revenue source filtering** — Reads `req.Body.Data.RevenueSource` (nullable). If the value doesn't equal the `treez.TreezSource` constant (the Treez e-commerce source identifier), logs warning and returns 200 OK (silent skip). Only e-commerce orders are processed.
4. **Event type filtering** — Calls `containsTreezEventType(req.Body.EventTypes, TreezEventTypeTicket)`. If `"TICKET_STATUS"` is not present in the array, logs warning and returns 200 OK (silent skip).
5. **Service call** — `h.treezService.ProcessOrder(ctx, accountID, storeID, req.Body.Data)`.
6. **Error handling**:
   - `order.ErrNotFound` for specific staging stores (`o3d1w4kbx6`, GoldFlora `ct9kgkrd9ghc711b7l1g`, Airfield `ct9grhstk77s711ohs10`): silent 200 OK.
   - `order.ErrNotFound` otherwise: logged + Sentry + 500.
   - Any other error: logged + Sentry (with accountID, storeID, orderID context) + 500.
7. **Success** — 201 Created.

### 1.2 Polymorphic `event_type` Handling

`TreezOrderRequestData.UnmarshalJSON` (`internal/model/treez.go`):

- Reads `event_type` as `json.RawMessage`.
- **Try string first:** If `event_type` is a JSON string, sets `EventType = value` and `EventTypes = [value]`.
- **Try array second:** If `event_type` is a JSON array, sets `EventTypes = values` and `EventType = values[0]`.
- **Neither:** Returns error `"decoding event_type: must be string or array"`.
- Also unmarshals the optional `test` string field (present in test payloads) and the `data` object.

### 1.3 Service Processing (`ProcessOrder`)

**Step 1: Store retrieval** — `s.store.Get(ctx, storeKey)` fetches the store from S3. Error wraps with accountID/storeID/orderID context.

**Step 2: Order retrieval** — `s.order.GetOrderByID(ctx, *store.DynamoOrderTableName, data.ExternalOrderNumber)`. Returns `inerror.EntityNotFound` if the order is nil.

**Step 3: Abandoned cart check** — If `order.Status == OrderStatusTypeAbandonedCart`, returns nil immediately. Rationale: TreezPay creates an order on the Treez side, but if the customer abandons the cart, Treez sends a cancel event that should be ignored.

**Step 4: DeclinedFromAdmin flag reset** — If `order.DeclinedFromAdmin` is non-nil and true:
- Sets `order.DeclinedFromAdmin = nil`.
- Upserts order to DynamoDB.
- Returns nil immediately (no further processing).

**Step 5: Status mapping** — `mapOrderStatus(data.OrderStatus, *order.Status)` returns `(status, lisTrackEvent, notify bool)`.

**Step 6: Update order status** — `order.Status = &status`.

**Step 7: Resolve notes** — If `data.TicketNote` is non-nil, formats as `"Treez Notes: {note}"` and appends to existing `order.Notes` with a newline separator.

**Step 8: Activity log** — Appends to `order.ActivityLogs`:
```
Action:    "UpdateOrderStatus"
User:      "API"
Status:    <mapped status>
Timestamp: <UTC unix timestamp>
```
Also sets `order.UpdatedAt`.

**Step 9: Persist** — Upserts to DynamoDB.

**Step 10: LisTrack email (delivery orders only)** — If `utils.IsDelivery(order.DeliveryDetails)` is true, sends a transactional email via `s.listrack.SendTransactionalEmail()` with the mapped event type. Failure is logged as a warning but does not fail the operation. Pickup orders are not emailed here — they are notified when manually moved in the admin dashboard.

**Step 11: Alpine IQ handling** — `s.handleAlpineIQ(ctx, accountID, storeID, order, store, data)`. See section 1.6.

**Step 12: SNS notification** — If `notify == true`, publishes `OrderStatusChangedNotifyEventType` to SNS. See section 1.7.

### 1.4 Full Status Mapping

`mapOrderStatus()` in `internal/service/treez/service.go`:

| TreezOrderStatus | String Value | Internal OrderStatusType | LisTrack Event | Notify (SNS) |
|---|---|---|---|---|
| `VERIFICATION_PENDING` | `"VERIFICATION_PENDING"` | *(unchanged — falls to default)* | `OrderNew` | `false` |
| `AWAITING_PROCESSING` | `"AWAITING_PROCESSING"` | *(unchanged — falls to default)* | `OrderNew` | `false` |
| `IN_PROCESS` | `"IN_PROCESS"` | *(unchanged — falls to default)* | `OrderNew` | `false` |
| `PACKED_READY` | `"PACKED_READY"` | `OrderStatusTypeApproved` | `OrderNew` | `true` |
| `OUT_FOR_DELIVERY` | `"OUT_FOR_DELIVERY"` | `OrderStatusTypeDeliveryStarted` | `OrderNew` | `true` |
| `COMPLETED` | `"COMPLETED"` | `OrderStatusTypeDelivered` | `OrderCompletedPickup` | `true` |
| `CANCELED` | `"CANCELED"` | `OrderStatusTypeDeclined` | `OrderRejectedPickup` | `true` |

Default case (any unrecognised status): status unchanged, event = `OrderNew`, notify = `false`.

### 1.5 Abandoned Cart & DeclinedFromAdmin

- **Abandoned cart:** If the order is already in `OrderStatusTypeAbandonedCart`, the entire webhook is silently ignored. No status update, no side effects.
- **DeclinedFromAdmin:** If the admin had previously declined the order (`DeclinedFromAdmin == true`), the flag is cleared and the order is saved. No further processing occurs — the Treez status update is effectively consumed just to clear the flag.

### 1.6 Alpine IQ Handling

`handleAlpineIQ()` in `internal/service/treez/service.go`:

**Config resolution:**
1. Fetches Alpine IQ app config via `s.getAlpineIQApp(ctx, accountID, storeID)`.
2. If app not found, not configured, or inactive: returns nil silently.
3. Validates `UserID` and `APIKey` are non-nil. If nil: logs error, captures to Sentry, returns nil.

**Discount reversion (on decline):**
- **Condition:** `order.Status == OrderStatusTypeDeclined AND order.AlpineiqRedemptionID != nil`.
- Fetches account by email: `s.account.GetByEmail(ctx, *order.Email, tableName)`.
- Calls `aiqService.RevertDiscountRedemption()` with UserID, ContactID (from account), TemplateID, RedemptionID.
- Failure: logged + Sentry, but execution continues.

**SMS notification:**
- **Condition:** `aiqApp.NotifyBySms` must be non-nil and true.
- Resolves phone number: tries `order.DeliveryAddress.Phone` first, falls back to `account.Phone`. If both nil, no SMS sent.
- Resolves message via `resolveSMSMessage()`:

| TreezOrderStatus | Active Flag Required | Message Template |
|---|---|---|
| `AWAITING_PROCESSING` | `OrderConfirmSmsMessageActive` | `"{StoreName} - Your order #{TreezOrderNumber} has been confirmed.\n\nReply STOP to opt out."` |
| `PACKED_READY` | `OrderReadySmsMessageActive` | `"{StoreName} - Your order #{TreezOrderNumber} is ready.\n\nReply STOP to opt out."` |
| `OUT_FOR_DELIVERY` | `OrderReadySmsMessageActive` | `"{StoreName} - Your order #{TreezOrderNumber} is ready.\n\nReply STOP to opt out."` |
| `COMPLETED` | `OrderCompletedSmsMessageActive` | `"{StoreName} - Your order #{TreezOrderNumber} has been completed.\n\nReply STOP to opt out."` |
| `CANCELED` | `OrderCanceledSmsMessageActive` | `"{StoreName} - Your order #{TreezOrderNumber} has been canceled.\n\nReply STOP to opt out."` |
| `VERIFICATION_PENDING`, `IN_PROCESS` | — | No SMS |

- Sends via `client.SendSMS()` with `CampaignID` from app config and `BillingUserID` from app UserID.
- Failure: logged + Sentry, but execution continues.

### 1.7 SNS Event Publishing

`publishSNSEvent()` in `internal/service/treez/service.go`:

**Payload:**
```json
{
  "account_id": "<accountID>",
  "store_id": "<storeID>",
  "entity_id": "<orderEntityID>"
}
```

**SNS message attributes:**
- `event_type` = `worker.OrderStatusChangedNotifyEventType`
- `publisher` = `"gapcommerce"`

**Subject:** `"order complete"`
**Topic:** `s.config.TopicArn` (from `aws_sns_topic` env var).

### 1.8 Treez Enum Values

```go
// TreezEventType
TreezEventTypeTicket  = "TICKET_STATUS"
TreezEventTypeProduct = "PRODUCT"

// TreezOrderStatus
TreezOrderStatusVerificationPending = "VERIFICATION_PENDING"
TreezOrderStatusAwaitingProcessing  = "AWAITING_PROCESSING"
TreezOrderStatusInProcess           = "IN_PROCESS"
TreezOrderStatusPackedReady         = "PACKED_READY"
TreezOrderStatusOutForDelivery      = "OUT_FOR_DELIVERY"
TreezOrderStatusCompleted           = "COMPLETED"
TreezOrderStatusCanceled            = "CANCELED"
```

---

## 2. Jane Order Webhook

**Endpoint:** `POST /jane/order-updated/?account_id=X&store_id=Y`
**Header:** `Authorization: Bearer <token>`
**Handler:** `internal/app/jane.go` — `OrderUpdated` (line 55)
**Service:** `internal/service/jane/service.go` — `ProcessOrder` (line 52)

### 2.1 Handler Flow

1. **Request parsing** — `NewJaneRequestBuilder(r, h.config).Build()`.
2. **Auth validation** — Splits `Authorization` header on `"Bearer "`. If not exactly 2 parts, returns 400 `"incorrect jane header"`. Compares extracted token against `config.JaneVerificationToken`. If mismatch, returns 401 `inerror.Unauthorized`.
3. **Query param validation** — Extracts `account_id` and `store_id`. Returns 400 if missing.
4. **Body decode** — JSON decodes into `JaneRequestData` (contains `Cart` with `ID`, `Status`, `Mode`, `CustomerName`, `CustomerEmail`).
5. **Service call** — `h.janeService.ProcessOrder(ctx, accountID, storeID, body)`.
6. **Error handling:**
   - `inerror.EntityNotFound`: 404.
   - Other: logged + Sentry + 500.
7. **Success** — 200 OK with `{"status": "OK"}`.

### 2.2 Service Processing (`ProcessOrder`)

**Step 1: Store retrieval** — Fetches store from S3.

**Step 2: Jane order lookup** — `s.repository.Get(ctx, fmt.Sprint(request.Cart.ID), JaneOrderKey, tableName)`. Looks up the `JaneOrder` record in DynamoDB by Jane cart ID (key prefix `j#`) to get the internal `OrderID`.

**Step 3: Order retrieval** — `s.order.GetOrderByID(ctx, tableName, janeOrder.OrderID)`. Returns `inerror.EntityNotFound` if nil.

**Step 4: Delivery determination** — `isDelivery := utils.IsDelivery(order.DeliveryDetails)`.

**Step 5: Status mapping** — `s.mapStatus(request.Cart.Status, isDelivery)` returns `(status, *eventType)`.

**Step 6: Update order status** — `order.Status = &status`.

**Step 7: Delivery timestamp enrichment** — If the new status is anything other than `OrderStatusTypePending`, calls `s.getJaneDeliveryPickUpDates()`:
- Fetches Jane app config from S3 (`apps.json`).
- Initialises Jane API service with credentials (AppID, APIKey, ClientID, ClientSecret, etc.).
- Calls `srv.RetrieveCart(ctx, *order.JaneOrderID)`.
- Copies `cart.DeliveryStartedAt` → `order.DeliveryDetails.StartedAt` and `cart.DeliveryFinishedAt` → `order.DeliveryDetails.FinishedAt` (as `scalar.Timestamp`).
- **Non-fatal:** Failure is logged + Sentry, but ProcessOrder continues.

**Step 8: Persist** — Upserts order to DynamoDB. Failure returns error (500).

**Step 9: LisTrack email** — If `eventType != nil`, sends transactional email. **Failure is fatal** — returns error (500).

### 2.3 Full Status Mapping

`mapStatus()` in `internal/service/jane/service.go`:

| JaneOrderStatus | String Value | Internal OrderStatusType | LisTrack Event (Pickup) | LisTrack Event (Delivery) |
|---|---|---|---|---|
| `verification` | `"verification"` | `OrderStatusTypePending` | nil (no email) | nil (no email) |
| `pending` | `"pending"` | `OrderStatusTypePending` | nil (no email) | nil (no email) |
| `dismissed` | `"dismissed"` | `OrderStatusTypeDeclined` | `OrderRejectedPickup` | `OrderRejectedDelivery` |
| `cancelled` | `"cancelled"` | `OrderStatusTypeDeclined` | `OrderRejectedPickup` | `OrderRejectedDelivery` |
| `finished` | `"finished"` | `OrderStatusTypeDelivered` | `OrderCompletedPickup` | `OrderCompletedDelivery` |
| `finished_without_review` | `"finished_without_review"` | `OrderStatusTypeDelivered` | `OrderCompletedPickup` | `OrderCompletedDelivery` |
| `ready_for_pickup` | `"ready_for_pickup"` | `OrderStatusTypeReady` | `OrderApprovedPickup` | `OrderApprovedPickup` * |
| `delivery_started` | `"delivery_started"` | `OrderStatusTypeDeliveryStarted` | `OrderOnRoute` | `OrderOnRoute` * |
| `delivery_finished` | `"delivery_finished"` | `OrderStatusTypeDeliveryFinished` | `OrderCompletedDelivery` | `OrderCompletedDelivery` * |
| `with_review` | `"with_review"` | `OrderStatusTypeProcessing` | nil (no email) | nil (no email) |
| `staff_member_review` | `"staff_member_review"` | `OrderStatusTypeProcessing` | nil (no email) | nil (no email) |

\* `ready_for_pickup` has no delivery variant (always `OrderApprovedPickup`). `delivery_started` and `delivery_finished` have no pickup variant (always delivery event types).

### 2.4 Delivery vs. Pickup Differences

The `isDelivery` flag only affects which LisTrack event type is selected for statuses that have both pickup and delivery variants:
- **Dismissed/Cancelled:** `OrderRejectedPickup` vs `OrderRejectedDelivery`
- **Finished/FinishedWithoutReview:** `OrderCompletedPickup` vs `OrderCompletedDelivery`

### 2.5 Jane Enum Values

```go
JaneOrderStatusDismissed             = "dismissed"
JaneOrderStatusCancelled             = "cancelled"
JaneOrderStatusFinished              = "finished"
JaneOrderStatusFinishedWithoutReview = "finished_without_review"
JaneOrderStatusVerification          = "verification"
JaneOrderStatusPending               = "pending"
JaneOrderStatusReadyForPickup        = "ready_for_pickup"
JaneOrderStatusDeliveryFinished      = "delivery_finished"
JaneOrderStatusDeliveryStarted       = "delivery_started"
JaneOrderStatusReview                = "with_review"
JaneOrderStatusStaffReview           = "staff_member_review"
```

DynamoDB key: `JaneOrderKey = "j#"`

---

## 3. Blaze Order Webhook

**Endpoint:** `POST /blaze/order-updated?account_id=X&store_id=Y`
**Handler:** `internal/app/blaze.go` — `OrderUpdated` (line 91)
**Service:** `internal/service/blaze/service.go` — `ProcessOrder` (line 112)

### 3.1 Handler Flow

1. **Request parsing** — `NewBlazeRequestBuilder[blaze.Order](r).Build()`. Extracts query params and decodes body as `blaze.Order`.
2. **Service call** — `h.blazeService.ProcessOrder(ctx, accountID, storeID, blazeOrder)`.
3. **Error handling:**
   - `inerror.EntityNotFound`: 404.
   - Other: logged + Sentry + 500.
4. **Success** — 200 OK.

### 3.2 Service Processing (`ProcessOrder`)

1. **Store retrieval** — Fetches store from S3.
2. **Blaze order lookup** — `s.repository.Get(ctx, bOrder.Cart.ID, BlazeOrderKey, tableName)`. Looks up the `BlazeOrder` record in DynamoDB by Blaze cart ID (key prefix `blaze#`) to get the internal `OrderID`.
3. **Order retrieval** — Gets the full order by OrderID.
4. **Status mapping** — `mapBlazeOrderStatusToStatus(bOrder.OrderStatus)`.
5. **Update and persist** — Sets `order.Status` and upserts to DynamoDB.

### 3.3 Full Status Mapping

`mapBlazeOrderStatusToStatus()`:

| Blaze Status | Internal OrderStatusType |
|---|---|
| `blaze.AcceptedOrderStatus` | `OrderStatusTypeProcessing` |
| `blaze.CanceledOrderStatus` | `OrderStatusTypeDeclined` |
| *(any other / default)* | `OrderStatusTypeDelivered` |

### 3.4 Side Effects

**None.** The Blaze order webhook only updates the order status in DynamoDB. No email, no SMS, no SNS events, no activity log entries.

### 3.5 Blaze Model

```go
type BlazeOrder struct {
    EntityID  string           `dynamodbav:"entity_id"`
    Key       string           `dynamodbav:"key"`         // "blaze#{cartID}"
    OrderID   string           `dynamodbav:"order_id"`
    CreatedAt scalar.Timestamp `dynamodbav:"created_at"`
}
```

DynamoDB key: `BlazeOrderKey = "blaze#"`

---

## 4. OnFleet Delivery Webhooks

**Endpoints:** 6 task event endpoints, all under `/onfleet/`
**Handler:** `internal/app/onfleet.go`
**Service:** `internal/service/onfleet/service.go`

All 6 handlers share the same `processTask` pattern:
- **GET requests** return the `check` query parameter (OnFleet webhook verification handshake).
- **POST requests** decode the `OnFleetRequest`, call the service method, and **always return 200 OK** — errors are logged and sent to Sentry but never returned to OnFleet.

### 4.1 Request Builder Enrichment

`OnFleetRequestBuilder.Build()` (`internal/model/onfleet.go`) does significant work:

1. Decodes the `OnFleetTask` JSON body and extracts the `onfleet.Task`.
2. Iterates through `task.Metadata` array to extract `account_id`, `store_id`, and `entity_id` (the internal order ID).
3. Fetches the full `Store` from S3 using the account/store key.
4. Fetches the full `Order` from DynamoDB using entity_id and `store.DynamoOrderTableName`.
5. Validates that `order.DeliveryDetails.OnFleet.ID` matches the incoming `task.ID`.
6. Returns `OnFleetRequest` containing `BaseRequest`, `Order`, `Store`, and `Task`.

The handler receives a fully enriched request with the complete Order and Store already loaded.

### 4.2 TaskStarted

**Endpoint:** `GET/POST /onfleet/task-started`
**Service method:** `ProcessTaskStarted`

**Order status transition:** `Order.Status` → `OrderStatusTypeDeliveryStarted`

**DynamoDB update:**
- `Order.Status = OrderStatusTypeDeliveryStarted`
- `Order.ActivityLogs` += `{Action: "UpdateOrderStatus", User: "API", Status: OrderStatusTypeDeliveryStarted, Timestamp: now}`
- `Order.UpdatedAt = now`

**LisTrack email:** `LisTrackEventTypeOrderOnRoute`
- Non-fatal: failure logged + Sentry, execution continues.

**Treez ticket sync (if `order.TreezOrderID` exists):**
1. Gets Treez service credentials from app config.
2. Looks up customer by email (lowercased) via `treezService.GetCustomerByEmail()`.
3. Updates ticket: `Status = treez.OutOfDeliveryOrderStatusType`, `CustomerID` from lookup.
4. If Treez app not configured: returns nil silently (skips).

### 4.3 TaskArriving

**Endpoint:** `GET/POST /onfleet/task-arriving`
**Service method:** `ProcessTaskArriving`

**Status transition:** `Order.DeliveryDetails.OnFleet.Status` → `DeliveryStatusArriving`

Note: Updates the **nested delivery status**, not the main `Order.Status`.

**DynamoDB update:**
- `Order.DeliveryDetails.OnFleet.Status = DeliveryStatusArriving`

**LisTrack email:** None
**Treez ticket sync:** None
**Activity log:** None

### 4.4 TaskCompleted

**Endpoint:** `GET/POST /onfleet/task-completed`
**Service method:** `ProcessTaskCompleted`

**Order status transition:** `Order.Status` → `OrderStatusTypeDeliveryFinished`

**DynamoDB update:**
- `Order.Status = OrderStatusTypeDeliveryFinished`
- `Order.ActivityLogs` += `{Action: "UpdateOrderStatus", User: "API", Status: OrderStatusTypeDeliveryFinished, Timestamp: now}`
- `Order.UpdatedAt = now`

**LisTrack email:** `LisTrackEventTypeOrderCompletedDelivery`
- Non-fatal: failure logged + Sentry, execution continues.

**Treez ticket sync with payment details (if `order.TreezOrderID` exists):**
1. Gets Treez service credentials.
2. Looks up customer by email.
3. Fetches full ticket from Treez via `treezService.GetTicket()`.
4. Maps payment method from internal order to Treez:

| Internal Payment Type | Treez Payment Method |
|---|---|
| `PaymentTypeCheck` | `PaymentMethodCheck` |
| `PaymentTypeCreditCard` | `PaymentMethodCredit` |
| `PaymentTypeDebitCard` | `PaymentMethodDebit` |
| `PaymentTypeAeropay` | `PaymentMethodAeropay` |
| nil / unknown (default) | `PaymentMethodCash` |

5. Updates ticket with:
   - `Payments`: single entry with `PaymentSource = EccommerceOrderSourceType`, mapped `PaymentMethod`, `AmountPaid = ticket total`
   - `Status = treez.CompletedOrderStatusType`
   - `CashDrawerName`: worker name from OnFleet task

### 4.5 TaskFailed — Full Revert Flow

**Endpoint:** `GET/POST /onfleet/task-failed`
**Service method:** `ProcessTaskFailed`

**Order status transition:** `Order.Status` → `OrderStatusTypeDeclined`

This is the most destructive handler. It performs a multi-step revert:

**Part 1: Decline the order**
1. Sets `Order.Status = OrderStatusTypeDeclined`.
2. **Clears OnFleet reference:** `Order.DeliveryDetails.OnFleet = nil`.
3. Appends activity log: `{Action: "UpdateOrderStatus", User: "API", Status: OrderStatusTypeDeclined, Timestamp: now}`.
4. Sets `Order.UpdatedAt = now`.
5. **First DynamoDB upsert** — persists declined status and cleared delivery reference.

**Part 2: Remove Treez ticket (if `order.TreezOrderID` exists)**
1. Gets Treez service credentials.
2. **Calls `treezService.RemoveTicket(ctx, TreezOrderID)`** — completely removes/cancels the ticket from Treez.
3. Clears internal references: `Order.TreezOrderID = nil`, `Order.TreezOrderNumber = nil`.
4. **Second DynamoDB upsert** — persists cleared Treez references.

**Part 3: Rejection email**
- Event type: `LisTrackEventTypeOrderRejectedDelivery`.
- Non-fatal: failure logged + Sentry, execution continues.

**Error propagation:**
- If Part 1 upsert fails: error returned, Parts 2 and 3 don't execute.
- If Part 2 ticket removal fails: error returned, Part 3 doesn't execute.
- If Part 2 second upsert fails: error returned.
- If Part 3 email fails: logged only, handler still returns success.

### 4.6 TaskDelayed

**Endpoint:** `GET/POST /onfleet/task-delayed`
**Service method:** `ProcessTaskDelayed`

**Status transition:** `Order.DeliveryDetails.OnFleet.Status` → `DeliveryStatusDelayed`

Note: Updates the **nested delivery status**, not the main `Order.Status`.

**DynamoDB update:**
- `Order.DeliveryDetails.OnFleet.Status = DeliveryStatusDelayed`

**LisTrack email:** None
**Treez ticket sync:** None
**Activity log:** None

### 4.7 TaskAssigned

**Endpoint:** `GET/POST /onfleet/task-assigned`
**Service method:** `ProcessTaskAssigned`

**Order status transition:** `Order.Status` → `OrderStatusTypeAssigned`

**DynamoDB update:**
- `Order.Status = OrderStatusTypeAssigned`
- `Order.ActivityLogs` += `{Action: "UpdateOrderStatus", User: "API", Status: OrderStatusTypeAssigned, Timestamp: now}`
- `Order.UpdatedAt = now`

**LisTrack email:** None
**Treez ticket sync:** None

### 4.8 OnFleet Summary Table

| Handler | `Order.Status` Change | Nested Delivery Status Change | Activity Log | LisTrack Email | Treez Ticket Sync |
|---|---|---|---|---|---|
| TaskStarted | → `DeliveryStarted` | — | Yes | `OrderOnRoute` | Update: `OutOfDelivery` |
| TaskArriving | — | → `Arriving` | No | — | — |
| TaskCompleted | → `DeliveryFinished` | — | Yes | `OrderCompletedDelivery` | Update: `Completed` + payment details |
| TaskFailed | → `Declined` | OnFleet ref cleared | Yes | `OrderRejectedDelivery` | **RemoveTicket** (full removal) |
| TaskDelayed | — | → `Delayed` | No | — | — |
| TaskAssigned | → `Assigned` | — | Yes | — | — |

### 4.9 DynamoDB Field Changes per Handler

| Handler | Fields Modified |
|---|---|
| TaskStarted | `Status`, `ActivityLogs`, `UpdatedAt` |
| TaskArriving | `DeliveryDetails.OnFleet.Status` |
| TaskCompleted | `Status`, `ActivityLogs`, `UpdatedAt` |
| TaskFailed | `Status`, `DeliveryDetails.OnFleet` (nil), `ActivityLogs`, `UpdatedAt`, `TreezOrderID` (nil), `TreezOrderNumber` (nil) |
| TaskDelayed | `DeliveryDetails.OnFleet.Status` |
| TaskAssigned | `Status`, `ActivityLogs`, `UpdatedAt` |

---

## 5. Request Builders

Each provider has a dedicated `*RequestBuilder` that validates auth, extracts context, and enriches the request before the handler invokes the service.

### 5.1 Treez — `TreezOrderRequestBuilder`

`internal/model/treez.go`

1. Validates `account_id` and `store_id` from query params (via `BaseRequestBuilder`).
2. JSON-decodes body into `TreezOrderRequestData` using custom `UnmarshalJSON` (handles polymorphic `event_type`).
3. Returns `TreezOrderRequest` with `BaseRequest` + `TreezOrderRequestData`.

### 5.2 Jane — `JaneRequestBuilder`

`internal/model/jane.go`

1. Splits `Authorization` header on `"Bearer "`.
2. Validates exactly 2 parts. Returns `"incorrect jane header"` error if not.
3. Compares extracted token to `config.JaneVerificationToken`. Returns `inerror.Unauthorized` on mismatch.
4. Validates `account_id` and `store_id` from query params.
5. JSON-decodes body into `JaneRequestData` (cart ID, status, mode, customer name/email).
6. Returns `JaneRequest` with `BaseRequest` + `JaneRequestData`.

### 5.3 Blaze — `BlazeRequestBuilder[T]`

`internal/model/blaze.go`

1. Validates `account_id` and `store_id` from query params.
2. JSON-decodes body as type `T` (generic — `blaze.Order` for order webhook, `blaze.SearchProduct` for product webhook).
3. Returns `BlazeRequest[T]` with `BaseRequest` + body.

### 5.4 OnFleet — `OnFleetRequestBuilder`

`internal/model/onfleet.go`

The most complex builder — performs data enrichment during `Build()`:

1. JSON-decodes body into `OnFleetTask` and extracts the nested `onfleet.Task`.
2. Iterates `task.Metadata` array to extract `account_id`, `store_id`, `entity_id`.
3. **Fetches Store** from S3 using `{accountID}/{storeID}/store.json`.
4. **Fetches Order** from DynamoDB using `entity_id` and `store.DynamoOrderTableName`.
5. Validates `order.DeliveryDetails.OnFleet.ID == task.ID` (ensures the webhook matches the right order).
6. Returns `OnFleetRequest` containing `BaseRequest`, full `Order`, full `Store`, and `Task`.

---

## 6. Error Handling — Partial Failure Scenarios

### 6.1 Treez

| Step | Failure | Outcome |
|---|---|---|
| Store fetch | Error | 500 to caller |
| Order fetch | Error / nil | 500 or EntityNotFound |
| DynamoDB upsert | Error | 500 to caller |
| LisTrack email | Error | **Warning logged, continues** — Alpine IQ + SNS still fire |
| Alpine IQ config | Missing / nil fields | **Silent skip** — SMS/discount not attempted |
| Alpine IQ discount reversion | Error | **Logged + Sentry, continues** — SMS still fires |
| Alpine IQ SMS | Error | **Logged + Sentry, continues** — SNS still fires |
| SNS publish | Error | Returned (but follows all other side effects) |

### 6.2 Jane

| Step | Failure | Outcome |
|---|---|---|
| Store fetch | Error | 500 to caller |
| Jane order lookup | Error | 500 to caller |
| Order fetch | Error / nil | 500 or 404 |
| Jane API (timestamps) | Error | **Logged + Sentry, continues** — order still updated |
| DynamoDB upsert | Error | 500 to caller |
| LisTrack email | Error | **500 to caller** (fatal — unlike Treez) |

### 6.3 Blaze

| Step | Failure | Outcome |
|---|---|---|
| Store fetch | Error | 500 to caller |
| Blaze order lookup | Error | 500 to caller |
| Order fetch | Error | 500 to caller |
| DynamoDB upsert | Error | 500 to caller |

All Blaze errors are fatal. No side effects to partially fail.

### 6.4 OnFleet

**All OnFleet handlers return 200 to the webhook source regardless of errors.** Errors are logged and sent to Sentry only.

| Step | Failure | Outcome |
|---|---|---|
| DynamoDB upsert | Error | Logged + Sentry, 200 returned |
| Treez app not configured | — | **Silent skip** (returns nil) |
| Treez ticket update/removal | Error | Logged + Sentry, blocks subsequent steps |
| LisTrack email | Error | **Logged + Sentry, continues** |

For TaskFailed specifically: if the first upsert (decline) succeeds but Treez ticket removal fails, the order is left in Declined status with the Treez reference still present — an inconsistent state.

---

## 7. Alpine IQ Discount Reversion

**When:** A Treez order webhook sets the status to `CANCELED` AND the order has an `AlpineiqRedemptionID`.

**Flow:**
1. Load Alpine IQ app config (UserID, APIKey) from `apps.json` in S3.
2. Initialise Alpine IQ service client.
3. Fetch the customer account by email from DynamoDB to get the `ContactID`.
4. Call `aiqService.RevertDiscountRedemption()` with:
   - `UserID` (Alpine IQ user)
   - `ContactID` (customer's Alpine IQ contact ID)
   - `TemplateID` (from Alpine IQ app config)
   - `RedemptionID` (from `order.AlpineiqRedemptionID`)
5. On failure: log error, capture to Sentry, **continue execution** (non-fatal).

This ensures that when a Treez order is cancelled, any loyalty discount that was redeemed for that order is rolled back in Alpine IQ's system.

---

*Generated: 2026-04-13 — Source: github.com/gap-commerce/api-srv @ 649bee6*
