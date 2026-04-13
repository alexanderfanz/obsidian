# e-com-srv — Order Lifecycle Deep-Dive

> The central orchestrator for the cart/checkout flow. This service owns the canonical order lifecycle from cart creation through checkout, POS submission, payment processing, approval, fulfillment, and completion. It coordinates across multiple POS systems (Treez, Dutchie, Jane, Blaze), payment providers (AeroPay, Stronghold, Swifter, TreezPay), CRM platforms, and delivery services.

---

## 1. Cart Creation

### How a Cart is Created

Carts are created implicitly on the first `UpdateCart` (or variant) call. There is no dedicated "create cart" mutation. When `UpdateCart` receives an order with no `EntityID`, it generates one:

```go
// handle_order.go:54-59
if order.EntityID == nil || *order.EntityID == "" {
    id := uuid.New().String()
    order.EntityID = &id
    ky := "d#"
    order.Key = &ky
}
```

### Initial State

Fields populated at creation time:

| Field | Value | Notes |
|-------|-------|-------|
| `EntityID` | UUID v4 string | Generated if not provided |
| `Key` | `"d#"` | DynamoDB sort key — "d" for draft |
| `Status` | `OrderStatusTypeAbandonedCart` | Every cart starts as abandoned |
| `CreatedAt` | Current UTC unix timestamp | |
| `UpdatedAt` | Current UTC unix timestamp | |
| `ClientDetail` | From request context | UserAgent, XForwardedFor, etc. |

### How `key="d#"` Works

The DynamoDB table uses composite key `(entity_id, key)`. The `"d#"` value identifies the record as a draft/cart. This allows:
- Querying all draft carts for an email via the email-index with filter `key = "d#"`
- Distinguishing carts from submitted orders in the same table
- The `Order.GetOrderByID()` method queries with `key="d#"` explicitly

### Persistence

Cart is persisted via `Services.Order.Upsert()` which calls DynamoDB `PutItem` — a full-document upsert that overwrites the entire record on every call.

### Cart Creation Entry Points

There are four separate cart update mutations, each for a different POS integration:

| Mutation | Function | File | POS |
|----------|----------|------|-----|
| `UpdateCart` | `App.UpdateCart()` | handle_order.go:39 | Generic / Blaze |
| `UpdateJaneCart` | `App.UpdateJaneCart()` | handle_order.go:91 | Jane |
| `UpdateTreezCart` | `App.UpdateTreezCart()` | handle_order.go:2211 | Treez |
| `UpdateDutchieCart` | `App.UpdateDutchieCart()` | handle_dutchie_order.go:19 | Dutchie |

All four create a cart on first call if no `EntityID` is present.

---

## 2. UpdateCart Mutation — Full Sequence

### Resolver Wiring

```
resolver/order.go:32 → mutationResolver.UpdateCart()
  → extracts ClientDetail from context
  → calls MapOrderInputToOrder() to convert GraphQL input to model.Order
  → delegates to app.UpdateCart()
```

### Exact Order of Operations (`App.UpdateCart`, handle_order.go:39-90)

**Step 1: Fetch Store Configuration (line 47)**
- `Services.Store.Get(ctx, storeKey)` — retrieves store config (tax settings, timezone, DynamoDB table name)
- Error halts execution immediately

**Step 2: Initialize Cart Entity (lines 54-59)**
- If no `EntityID`: generate UUID, set `Key = "d#"`
- If EntityID exists: preserve it (update path)

**Step 3: Set Timestamps and Status (lines 61-66)**
- `CreatedAt` and `UpdatedAt` = current UTC unix timestamp
- `Status` = `OrderStatusTypeAbandonedCart` (always, on every update)
- `ClientDetail` = from request context

**Step 4: Fetch Active Promotions (lines 68-75)**
- `Services.Promotion.GetActives(ctx, promotionKey)` — loads all active promotions for the store from S3

**Step 5: Calculate Order Prices (line 77)**
- `calculateOrderPrice(ctx, order, store, promotions, accountID, storeID)` — see detailed breakdown below

**Step 6: Persist to DynamoDB (lines 82-89)**
- `Services.Order.Upsert(ctx, tableName, order)` — PutItem, full document replacement
- Returns the persisted order

### Line Item Management

Line items are completely replaced on each `UpdateCart` call. The GraphQL input contains the full list of line items; there is no add/remove delta. To remove an item, the frontend omits it from the next `UpdateCart` call.

`MapOrderInputToOrder()` (map_input_to_model.go:449-511) converts each `LineItemInput` to `LineItem`:
- Maps: `ItemID`, `ID`, `Name`, `Quantity`, `Price`, `Weight`, dimensions
- Maps: `Thumbnail`, `SalePrice`, `Brand`, `Category`, `Barcodes`, `PosID`
- Maps: `PriceType`, `TierMethod`, `BaseWeight`
- Creates variant slice from variants list

### Price Recalculation — `calculateOrderPrice()` (handle_order.go:406-526)

**Phase 1: Subtotal Calculation (lines 442-453)**
```
For each line item:
  variant_sum = sum of all variant prices
  line_total = variant_sum * quantity
subtotalPrice = sum of all line_totals
order.SubtotalPrice = subtotalPrice
order.TotalLineItemsPrice = subtotalPrice
```
All prices are in integer cents.

**Phase 2: Tax Calculation (lines 455-465)**
```
if store.TaxActive:
  if store.TaxType == Percentage:
    totalTax = subtotalPrice * store.TaxAmmont / 100
  if store.TaxType == FlatRate:
    totalTax = store.TaxAmmont
else:
  totalTax = 0
```

**Phase 3: Clear Automatic Promotions (lines 470-477)**
```
if discountItems exist AND first item is DiscountApplyTypeAutomatically:
  clear all discountItems  // allows re-evaluation
```
Manual/coupon-code discounts are preserved through this step.

**Phase 4: Calculate Manual Discount (lines 479-484)**
```
if discountItems exist (coupon applied):
  totalDiscount = calculatePromotionRate(promotion, order)
```

**Phase 5: Evaluate Automatic First-Time Patient Promotion (lines 486-514)**
```
if no discountItems AND Blaze service configured:
  for each active promotion where FirstTimePatient == true:
    skip if email == "enter_your@address.com"
    lookup = blazeService.FindUser(email)
    if user NOT found (first-time customer):
      apply promotion → create OrderDiscountItem
      break  // only one automatic promo per order
```
This is the only automatic promotion type. It queries the Blaze POS to check if the customer is new.

**Phase 6: Compute Final Totals (lines 516-525)**
```
order.TotalDiscounts = totalDiscount
order.TotalDelivery = 0  // always zero in UpdateCart
order.TotalPrice = SubtotalPrice + TotalDelivery + TotalTax - TotalDiscounts
```

### `calculatePromotionRate()` (handle_order.go:528-534)

```go
if DiscountRateType == Amount:
  return Rate  // fixed dollar amount in cents
else:  // Percentage
  return SubtotalPrice * Rate / 100
```

### External Calls During UpdateCart

| Call | Target | Required? | Purpose |
|------|--------|-----------|---------|
| `Services.Store.Get()` | S3 | Yes | Store config |
| `Services.Promotion.GetActives()` | S3 | Yes | Active promotions |
| `blazeService.FindUser()` | Blaze API | No (optional) | First-time customer check |
| `Services.Order.Upsert()` | DynamoDB | Yes | Persistence |

Blaze errors are silently ignored (`_`) — they don't block the cart update.

---

## 3. SubmitCart Mutation — Full Sequence

### Resolver: `resolver/order.go:50-62`

### Implementation: `App.SubmitCart()` (handle_order.go:136-144)

SubmitCart is surprisingly thin. It does **not** change order status, process payment, or submit to a POS. It only publishes an SNS event:

```
1. publishSNSEvent(ctx, accountID, storeID, entityID, OrderCompletedEventType)
2. Return "success" or error
```

### SNS Event Published

**Event type:** `worker.OrderCompletedEventType` (value: `"OrderCompleted"`)

**Payload:**
```json
{
  "account_id": "<accountID>",
  "store_id": "<storeID>",
  "entity_id": "<entityID>"
}
```

**Message attributes:**
- `event_type`: String = `"OrderCompleted"`
- `publisher`: String = `"gapcommerce"`
- `Subject`: `"order complete"`
- `TopicArn`: From app config (`a.Config.TopicArn`)

### What Happens Downstream

The SNS event triggers the **e-com-order-worker** (separate service) which:
- Assigns an invoice number
- Transitions status from AbandonedCart to Pending
- Creates/updates the customer account
- Publishes downstream events to trigger POS submission, CRM sync, etc.

### No Payment Processing

Payment is NOT initiated at SubmitCart. Payment must be set up beforehand via the per-provider mutations (AeroPay, Swifter, Stronghold, TreezPay) or handled by the downstream worker.

### On Success vs Failure

| Outcome | State | Recovery |
|---------|-------|----------|
| SNS publish succeeds | Cart remains `AbandonedCart` in DynamoDB; worker will transition it | N/A |
| SNS publish fails | Cart remains `AbandonedCart`; no state change | Client retries SubmitCart |

### SubmitKioskCheckout (handle_order.go:2442-2618)

Alternative submission path for in-store kiosks. Unlike SubmitCart, this **does** create the Treez order synchronously:

1. Retrieve store and order
2. Get Treez service
3. Customer resolution: lookup by email/phone, create if needed (minimal data: first name, last name, DOB, patient type = Adult)
4. Map order to Treez ticket, call `UpsertTicket()`
5. Map Treez totals back to order
6. Store `TreezOrderID`
7. Publish `OrderKioskCompletedEventType` SNS event
8. Return order

---

## 4. UpdateOrderStatus Mutation — Full Sequence

### Resolver: `resolver/order.go:72-77`

### Implementation: `App.UpdateOrderStatus()` (handle_order.go:536-620)

### Who Calls This

- **Admin UI** — for manual approval/rejection/completion
- **External webhook handlers** (in api-srv) — for delivery status updates from OnFleet, Treez, etc.

### Validation & Guards

1. Fetch store by storeKey
2. Fetch order by `input.ID` from DynamoDB
3. Order must exist (nil check)
4. **Same-status rejection:** `"order new status cannot be equal than previous status"`
5. **Declined guard:** If current status is `Declined`, only transition to `Pending` is allowed. Otherwise: `"order is already declined"`

### Exact Execution Order

```
1. Fetch store
2. Fetch order by ID
3. Validate order exists
4. Validate status != current status
5. Validate declined → only pending allowed
6. updateOrderActivityLog() — append ActivityLog entry
7. Set order.Status = input.Status
8. Execute status-specific handler (switch):
   - APPROVED → handleApproveOrCancelOrder(create=true)
   - PENDING or DECLINED → set DeclinedFromAdmin=true, handleApproveOrCancelOrder(create=false)
   - DELIVERY_FINISHED or DELIVERED → handleCompleteOrder()
9. Upsert order to DynamoDB
10. handleListrack() — send transactional email
11. publishSNSEvent(OrderStatusChangedNotifyEventType) — errors logged, don't fail
12. Return updated order
```

### APPROVED Transition — `handleApproveOrCancelOrder(create=true)` (handle_order.go:622-671)

**Barcode Validation (unless `BypassBarcodeValidation` is true):**
- Delivery time check: if `now > DeliverBefore` → error: `"This delivery order cannot be approved as the delivery time has already passed."`
- For each line item:
  - `Barcodes.length > 0 AND Barcodes.length == Quantity` required → error: `"line item {name} barcodes required"`
  - `PosID` must be non-nil and non-empty → error: `"line item {name} pos id is empty"`

**Operations in sequence:**

1. **`handleApproveOrCancelOrderTreezOrder(create=true)`** (lines 726-795)
   - Fetch Treez app config
   - Get Treez service using `order.GetProviderStoreID()`
   - Fetch account by email; handle driver license expiration (set to 3 years if missing)
   - `handleTreezCustomer()` — get or create Treez customer (lookup by email, then phone; create if not found)
   - `treez.MapOrderToTreezTicket()` — map order to Treez request with `PackedOrderStatusType`
   - `treezService.UpsertTicket()` — create ticket in Treez
   - On success: store `TreezOrderID` and `TreezOrderNumber` on order
   - On error: return error, order stays unapproved

2. **`handleApproveOrCancelOrderOnFleetOrder(create=true)`** (lines 797+) — only if delivery enabled
   - Create OnFleet task with delivery details
   - Store OnFleet task info in `order.DeliveryDetails.OnFleet`
   - **On error: ROLLBACK** — calls `handleApproveOrCancelOrderTreezOrder(create=false)` to remove the Treez ticket, then returns error

3. **`handleJaneRedeemOrderPoints()`** (line 674) — non-blocking, errors logged but don't fail the transition

### PENDING / DECLINED Transition — `handleApproveOrCancelOrder(create=false)`

1. Set `order.DeclinedFromAdmin = true`
2. `handleApproveOrCancelOrderTreezOrder(create=false)` → calls `treezService.RemoveTicket()`, clears `TreezOrderID` and `TreezOrderNumber`
3. If delivery enabled: `handleApproveOrCancelOrderOnFleetOrder(create=false)` → calls `onFleetService.RemoveTask()`, clears `OnFleet` reference

### DELIVERY_FINISHED / DELIVERED Transition — `handleCompleteOrder()` (handle_order.go:1662-1745)

1. Check `TreezOrderID` exists (if nil, return silently — some orders don't have Treez tickets)
2. `treezService.GetTicket()` — fetch current ticket
3. If already `CompletedOrderStatusType`: return (idempotent)
4. Determine payment method:
   - Non-cash payment type → `PaymentMethodDebit`
   - Cash → `PaymentMethodCash`
5. Build `UpdateTicket` request:
   ```
   Payments: [{
     PaymentSource: EccommerceOrderSourceType,
     PaymentMethod: (Debit or Cash),
     AmountPaid: ticket.Total
   }],
   Status: CompletedOrderStatusType
   ```
6. `treezService.UpdateTicket()` — mark completed in Treez
7. Send LisTrack transactional email:
   - Pickup → `LisTrackEventTypeOrderCompletedPickup`
   - Delivery → `LisTrackEventTypeOrderCompletedDelivery`

### Activity Log

Each status change appends to `order.ActivityLogs` via `updateOrderActivityLog()` (handle_order.go:1609-1624):
```go
ActivityLog{
  Action:    "UpdateOrderStatus",
  Timestamp: Unix(now),
  Status:    order.Status,
  User:      current_user.Email,
}
```

### Side Effects Summary

| Side Effect | APPROVED | DECLINED | DELIVERED |
|-------------|----------|----------|-----------|
| Treez ticket created | Yes | No (removed) | Completed |
| OnFleet task created | Yes (if delivery) | No (removed) | N/A |
| Jane points redeemed | Yes (non-blocking) | No | No |
| LisTrack email | Approved email | Rejected email | Completed email |
| SNS event published | Yes | Yes | Yes |
| DynamoDB upserted | Yes | Yes | Yes |

---

## 5. UpdateOrderDetails Mutation

### Resolver: `resolver/order.go:93-98`

### Implementation: `App.UpdateOrderDetails()` (handle_order.go:1284-1451)

### Allowed Update Window

Order must be in `Pending` status (post-submission, pre-approval).

### Fields That Can Be Updated

| Field | Input Key | Storage Location | Notes |
|-------|-----------|------------------|-------|
| Delivery time window | `TimeWindow` | `order.DeliveryDetails.DeliverAfter/Before` | Parsed from `"HH:MM AM/PM - HH:MM AM/PM"` format |
| Delivery address | `Address` | `order.DeliveryAddress` | OrderAddress type |
| Payment method | `PaymentMethod` | `order.PaymentDetails.Type` | |
| Promo code | `PromoCode` | `order.DiscountItems[].PromoCode` | Triggers LisTrack email |
| Notes | `Notes` | `order.Notes` | |
| Line items | `LineItems` | `order.LineItems` | Full replacement, triggers recalculation |

### Exact Operation Order

```
1. Fetch store
2. Fetch order by ID
3. Validate order exists
4. Initialize DeliveryDetails and PaymentDetails if nil
5. Parse and validate TimeWindow (split on "-", parse with store timezone)
6. Update delivery address if provided
7. Update payment method if provided
8. Update promo code if provided → triggers LisTrack email (LisTrackEventTypeCuoponAdded)
9. Update notes if provided
10. Map input line items to order line items
11. Recalculate: subtotal from line items, totalPrice = subtotal + delivery + tax - discounts
12. checkOrderOnTreez() → call treezService.PreviewTicket() for validation
13. If Treez ticket returned: overwrite all prices with Treez-calculated values
    - TotalPrice = Treez.Total * 100
    - TotalTax = Treez.TaxTotal * 100
    - Handle PostTaxPricing vs PreTaxPricing for SubtotalPrice and Discounts
    - Extract delivery and service fees from Treez fees array
14. updateOrderActivityLog() with action="UpdateOrderDetails"
15. Upsert order to DynamoDB
16. Return updated order
```

### Error Handling

All validation and calculation happens before any DB write. No partial updates are possible — if any step fails, no changes are persisted.

---

## 6. Multi-POS Fan-Out

### How It Works

Each order is associated with ONE provider store via `order.ProviderStoreID`. The cart update mutation chosen by the frontend determines which POS handler runs:

- `UpdateTreezCart` → Treez API calls
- `UpdateDutchieCart` → Dutchie GraphQL calls
- `UpdateJaneCart` → Jane passthrough (no external calls)
- `UpdateCart` → Generic / Blaze only (optional FindUser)

**There is no automatic fan-out to multiple POS systems from a single mutation.** Each mutation targets one POS.

### TreezPay Multi-Store Ticket Tracking

For TreezPay, an order can have tickets across multiple provider stores, tracked in `order.TreezPayTicketIdsByStore`:

```go
type TreezPayTicket struct {
    TicketID        string
    ProviderStoreID string
    CreatedAt       *Timestamp
}
```

Helper functions:
- `updateTreezPayTicketIdsByStore(order, ticket)` — append or update entry by provider store
- `getTreezPayTicketByProviderID(order, providerStoreID)` — lookup ticket for specific store
- `removeTreezPayTicketIdByTicketId(tickets, ticketID)` — remove entry by ticket ID

### Failure Handling

**During approval (`UpdateOrderStatus → APPROVED`):**
- If Treez ticket creation fails → entire approval fails, order stays unapproved
- If OnFleet task creation fails AFTER Treez ticket created → **explicit rollback**: Treez ticket is removed via `RemoveTicket()`, then error returned
- If DynamoDB upsert fails after both Treez and OnFleet succeed → **inconsistent state**: external resources created but order not updated in DB (no automatic rollback)

---

## 7. Payment Flows Per Provider

### AeroPay — Token Attachment

**Function:** `AddAeropayPaymentDetailToOrder()` (handle_order.go:1186-1282)

**Sequence:**
1. Validate order exists and status is `AbandonedCart` or `Pending`
2. Ensure no payment details already exist
3. Retrieve AeroPay app config (handler: `"aeropay"`)
4. Initialize AeroPay service with credentials (HTTP host, API key, secret, merchant ID)
5. `aeroPayService.GetTransaction(ctx, input.TransactionID)` — fetch transaction details
6. Create `PaymentDetail` with type `PaymentTypeCreditCard`:
   - Store: TransactionID, MerchantName, CustomerName
   - Amount: `int(amount * 100)` converted to cents
7. Upsert order

**Error handling:** Transaction lookup failure → error returned. Amount not numeric → `"aeropay transaction amount is not numeric"`. No retry logic.

### Stronghold — PayLink with Tipping

**Function:** `CreateStrongholdPayLink()` (handle_order.go:2768-2928)

**Sequence:**
1. Retrieve Stronghold app config (handler: `"stronghold"`)
2. Initialize client — returns service instance + `allowTiping` boolean flag from credentials
3. Customer resolution (3-step cascade):
   - `GetCustomer(ctx, gapAccountID, true)` — by GAP Commerce ID
   - `ListCustomers(ctx, email, "")` — by email
   - `ListCustomers(ctx, "", phone)` — by phone
   - If not found: `CreateCustomer()` with Individual info (name, email, phone, country="US")
4. Build PayLink:
   - Type: from input
   - CustomerID: resolved
   - AuthorizeOnly: `true`
   - Order amount: `order.TotalPrice`
   - Callbacks: SuccessUrl, ExitUrl from input
   - If `allowTiping`: include Tip object with `BeneficiaryName: "unknown"`
5. `srv.CreatePayLink(ctx, paylink)` — returns PayLink with ID and URL
6. Store payment detail: Type=`PaymentTypeStronghold`, PaylinkID, Status=`StrongholdPayLinkStatusCreated`
7. Upsert order
8. Return PayLink URL

**Charge completion:** Separate `UpdateStrongholdChargeID()` mutation stores the `ChargeID` from webhook callback.

### Swifter — OAuth Two-Step Flow

**Step 1: `CreateSwifterSession()`** (handle_order.go:1747-1872)

1. Validate order ID (UUID format), email (mail.ParseAddress)
2. Retrieve order, verify email ownership, verify status = `AbandonedCart`
3. **Idempotency:** If `SwifterDetail.SessionID` already set, return existing PaymentDetail
4. Resolve customer info from account (firstName, lastName, phone, driverLicenseID)
5. Initialize Swifter client with OAuth credentials (authHost, apiHost, clientID, clientSecret)
6. `srv.CreateSession(ctx, request)` with: Amount, Type="ConsumerCheckout", ConsumerInfo
7. Store: `SwifterDetail.SessionID = session.ID`, `Status = SwifterStatusStarted`
8. Upsert order

**Step 2: `SubmitSwifterPayment()`** (handle_order.go:1902-1973)

1. Retrieve order, verify status = `AbandonedCart`
2. Verify SessionID exists and OrderID not yet set (prevent double-payment)
3. `srv.CreateOrder(ctx, request)` with: SessionID, ExternalID=order.EntityID
4. Store: `OrderID = swifterOrder.ID`, `TrackNumber`, `Status = SwifterStatusCompleted`
5. Upsert order

**State machine:**
```
No payment → CreateSwifterSession → Status=Started (SessionID set)
                                          → SubmitSwifterPayment → Status=Completed (OrderID set)
```

**Error handling:**
- Invalid order ID → `"order id format is not valid"`
- Email mismatch → `"order does not belongs to requested email"`
- Wrong status → `"payment cannot be processed order is not on correct status"`
- Already processed → `"swifter payment has already been processed"`
- Session already created → returns existing (idempotent)

### TreezPay — Draft Ticket Integration

TreezPay is not a separate payment mutation. It's integrated into `UpdateTreezCart()` via the `createDraftTicket` flag. See section 2 (Treez handler) for the full flow.

**Token retrieval:** `GetTreezPayToken()` (handle_order.go:2620-2656)
1. Retrieve TreezPay app config
2. Initialize TreezPay service via `getTreezPayClient()`
3. `srv.Authenticate(ctx)` — OAuth token exchange
4. Return `auth.Tokens.AccessToken`
5. Frontend uses this token to initialize the TreezPay payment iframe

**Ticket lifecycle:**
- Draft tickets are created via `UpsertTicket()` with `DraftOrderStatusType`
- Tickets expire after 20 minutes of inactivity (`isTreezPayTicketExpired()`)
- Expired tickets are abandoned via `abandonTreezPayTicket()` — async `RemoveTicket()` call
- Paid tickets cannot be modified (CANNOT_MODIFY_TICKET error)
- Ticket IDs tracked per provider store in `TreezPayTicketIdsByStore`

---

## 8. Treez POS Handler — `handle_order_treez.go` (embedded in handle_order.go)

### UpdateTreezCart — Full Step-by-Step (handle_order.go:2211-2440)

**Input validation:**
- Email: default to `"enter_your@address.com"` if empty (guest checkout support), validate format
- `ProviderStoreID`: required, error if empty

**Initialization:**
- Store config, UUID generation, timestamps, status = AbandonedCart, key = "d#"
- Delivery time conversion to UTC

**Treez service setup:**
- Retrieve Treez app config via handler `"treez"`
- Validate app is enabled
- `getTreezService()` with credentials (sandbox flag, username, password)

**Customer ID resolution — `resolveUpdateTreezCartCustomerID()` (lines 2658-2696):**
1. If email is empty placeholder → return nil (guest)
2. Fetch account by email from DynamoDB
3. If no account → return nil (guest proceeds without customer)
4. Check for cached Treez ID in `account.TreezIds` by `providerStoreID`
5. If cached → return immediately (avoids API call)
6. If not cached → `treezCustomerLookup()`:
   - `GetCustomerByEmail(ctx, email)` first
   - `GetCustomerByPhone(ctx, phone)` fallback
7. If found → async `updateAccountTreezID()` to cache, return ID
8. If not found → `createTreezCustomer()`:
   - Build request via `mapTreezCustomerRequest()` (name, address, DOB, driver license, patient type, phone)
   - `treezService.CreateCustomer(ctx, request)`
   - Async cache new ID in account

**Ticket request mapping:**
- `treez.MapOrderToTreezTicket()` — converts order to Treez-format request
- `ExternalOrderNumber` = `order.EntityID` (this is the root cause of the duplicate ticket bug)

**Two execution paths:**

**Path A — Preview Mode (TreezPay disabled or `createDraftTicket=false`):**
1. `treezService.PreviewTicket(ctx, request)` — no ticket created, validation only
2. If existing TreezPay ticket: `abandonTreezPayTicket()` to cleanup

**Path B — Draft Ticket Mode (TreezPay enabled and `createDraftTicket=true`):**
1. Check for existing draft ticket via `getTreezPayTicketByProviderID()`
2. If existing ticket expired (>20 min) and not paid: `abandonTreezPayTicket()`
3. If no existing ticket (or was abandoned): `UpsertTicket(nil, request)` → creates new
4. If existing ticket AND ACH payment: `GetTicket()` — fetch only, don't update
5. If existing ticket AND non-ACH: `UpsertTicket(existingID, request)` → update
6. **CANNOT_MODIFY_TICKET fallback:** Remove ticket ID, retry `UpsertTicket(nil, ...)` → creates NEW ticket (bug path)

**Error handling with retry loop (max 10 retries):**
- `CUSTOMER_NOT_FOUND`: Remove cached Treez ID from account, retry
- `INACTIVE_CUSTOMER`: Check for `MergedIntoCustomerId`, update account with merged ID if found, retry
- Other errors: Map to internal error type via `resolveUpdateTreezCartErrorResponse()`, return

**Treez error code mapping:**

| Treez Error | Internal Code |
|-------------|---------------|
| VALIDATION_ERROR | ValidationError |
| PARTIAL_PAYMENT_NOT_SUPPORTED | PartialPaymentNotSupported |
| CAN NOT UPDATE 'PAID' TICKET | CanNotUpdatePaidTicket |
| LOCATION_NAME DOES NOT MATCH | LocationNameDoesNotMatch |
| INSUFFICIENT_SELLABLE_QUANTITY | InsufficientSellableQuantity |
| RESPONSE_LIMIT_EXCEEDED | ResponseLimitExceeded |
| INVALID_COUPON | InvalidCoupon |
| INVALID_TOKEN | InvalidToken |
| INVALID_REQUEST | InvalidRequest |
| DELIVERY_MINIMUM_ERROR | DeliveryMinimumError |

**Discount reconciliation (lines 2386-2415):**
- If Alpine discount IDs were sent in request, verify they appear in Treez response's `POSDiscounts`
- If POS didn't apply the discount: remove Alpine discount items from order, clear `AlpineiqTemplateID`, `AlpineiqRedemptionID`, `AlpineiqRedemptionURL`

**Price mapping — `mapTreezTotalsToOrderTotal()` (lines 3003-3041):**
- `TotalPrice` = `ticket.Total * 100`
- `TotalTax` = `ticket.TaxTotal * 100`
- If `PostTaxPricing`:
  - `SubtotalPrice` = `ticket.PostTaxSubTotal * 100`
  - `TotalDiscounts` = `ticket.PostTaxDiscountTotal * 100 * -1`
- Else (pre-tax):
  - Discount = `ticket.DiscountTotal * 100 * -1`
  - `SubtotalPrice` = `ticket.SubTotal * 100 + discount`
- Fees iterated: `FeeDeliveryType` → `TotalDelivery`, `FeeServiceType` → `TotalServiceFee`

**TreezPay ticket storage (if draft creation active):**
- `order.TreezOrderID = ticket.ID`
- `order.TreezOrderNumber = ticket.OrderNumber`
- `updateTreezPayTicketIdsByStore(order, ticket)` — append/update by provider store

**Persistence:** Upsert order to DynamoDB.

**Return:** `UpdateTreezCartResponse{Status: true, Order: order, Specials: mapTreezItemsDiscounts(ticket)}`

---

## 9. Dutchie POS Handler — `handle_dutchie_order.go`

### UpdateDutchieCart — Full Step-by-Step (lines 19-147)

**Input validation:**
- Email required and valid (mail.ParseAddress)
- `ProviderStoreID` required

**Early exit:** If no `DutchieOrderID` → just save the cart and return (no Dutchie API call needed yet).

**When DutchieOrderID is present:**
1. Retrieve Dutchie app config (handler: `"dutchie"`)
2. Validate app is enabled
3. Create Dutchie GraphQL service via `getDutchieService()` — matches credentials by `providerID` (retailer ID)
4. `dutchieService.GetOrderByOrderNumber(ctx, *order.DutchieOrderID)` — fetch from Dutchie

**Data mapping (all from Dutchie → order):**

| Dutchie Field | Order Field | Conversion |
|---------------|-------------|------------|
| `PaymentMethod` enum | `order.PaymentDetails.Type` | CASH→Cash, DEBIT/DEBIT_ONLY/DEBIT_CARD→DebitCard, CREDIT/CREDIT_CARD→CreditCard, CHECK→Check, AEROPAY→Aeropay |
| `ReservationDate.StartTime` | `order.DeliveryDetails.DeliverAfter` | Mapped to UTC |
| `ReservationDate.EndTime` | `order.DeliveryDetails.DeliverBefore` | Mapped to UTC |
| `Pickup` / `Delivery` booleans | `order.DeliveryDetails.Pickup/Delivery` | Direct |
| `Customer.Name` | `order.FirstName / LastName` | Split on space |
| `Customer.Phone` | `order.Phone` | Direct |
| `Customer.Birthdate` | `order.DateOfBirth` | Parsed from "01/02/2006" |
| `Total` | `order.TotalPrice` | `int(Total * 100)` |
| `Subtotal` | `order.SubtotalPrice` | `int(Subtotal * 100)` |
| `Tax` | `order.TotalTax` | `int(Tax * 100)` |
| `Discounts.Total` | `order.TotalDiscounts` | `int(Discounts.Total * 100)` |
| `Fees.Delivery` | `order.TotalDelivery` | `int(Fees.Delivery * 100)` |
| `Fees.Total` | `order.TotalServiceFee` | `int(Fees.Total * 100)` |

Key difference from UpdateCart: **Dutchie owns all pricing.** No calculation happens locally — all totals are imported from Dutchie.

5. Upsert order to DynamoDB
6. Return `UpdateDutchieCartResponse{Status: true, Order: order}`

---

## 10. Jane POS Handler (embedded in handle_order.go)

### UpdateJaneCart (handle_order.go:91-135)

**Key differences from UpdateCart:**
- **No price calculation** — no `calculateOrderPrice()` call
- **No promotion evaluation** — no Blaze lookup, no automatic discounts
- **Delivery time conversion** — converts `DeliverAfter`/`DeliverBefore` to UTC via `mapUpdateTimeStampToUTC()`
- **No external service calls** — only Store service for config + DynamoDB for persistence

Jane handles all pricing server-side. The e-com service is a passthrough.

### Jane Order History Import

`CustomerOrderHistory()` (handle_order.go:145-192) can import historical orders from Jane:
- `getOrdersHistoryFromJane()` — queries Jane e-commerce API
- `saveAccountNewOrders()` — bulk saves imported orders to DynamoDB
- `updateAccountImportStatus()` — marks orders as imported

### Jane Order Cancellation

`cancelJaneOrder()` (handle_order.go:2052-2103):
- Retrieves Jane app config
- Calls Jane API to cancel the order
- Maps Jane cart reservation back to order via `mapJaneCartReservationToOrder()`

---

## 11. Promotion Engine in Context

### ApplyPromotionCode — End-to-End (promotion.go:21-56)

1. Fetch all active promotions for the store from S3
2. Iterate looking for: `DiscountApplyType == PromotionCode` AND `PromoCode == input`
3. If found: return `OrderDiscountItem` with ID, DiscountType, DiscountRateType, Rate, BlazeID
4. If not found: return `nil, nil` (graceful — not an error)

The returned `OrderDiscountItem` is then included by the frontend in subsequent `UpdateCart` calls, where `calculateOrderPrice()` picks it up in Phase 4 and calculates the discount.

### How Promotions Interact with POS Pricing

**For Treez orders:** When `UpdateTreezCart` runs, discount IDs are sent to Treez in the ticket request. Treez applies its own discount logic. The service then validates that Treez actually applied the discounts by checking `ticket.Items[].POSDiscounts[]`. If a discount was requested but not applied by Treez (e.g., Alpine IQ discount), it's removed from the order.

**For Dutchie orders:** Dutchie owns discounts. `TotalDiscounts` is imported directly from Dutchie.

**For generic/Blaze orders:** Discounts are calculated locally in `calculatePromotionRate()`.

### Jane Specials (promotion.go:58-206)

`ListJaneSpecials()` — fetches specials from Jane API with caching:
- Cache stored in S3 at `jane_promotions.json`
- `flushCache` flag forces re-fetch
- Day-of-week filtering (EnabledSunday through EnabledSaturday)
- Date range filtering (StartDate/EndDate)
- Returns filtered specials based on store IDs and current day

---

## 12. Error Handling — Comprehensive Matrix

### Payment Failure

| Provider | Failure Point | Order State | External State | Recovery |
|----------|---------------|-------------|----------------|----------|
| AeroPay | GetTransaction fails | No change | N/A | Client retries |
| AeroPay | Amount not numeric | No change | N/A | Fix transaction data |
| Stronghold | Customer creation fails | No change | N/A | Client retries |
| Stronghold | PayLink creation fails | No change | N/A | Client retries |
| Swifter | CreateSession fails | No change | N/A | Client retries |
| Swifter | CreateOrder fails | Session created | Session exists in Swifter | Re-submit with same session |
| TreezPay | Ticket creation fails | AbandonedCart | No ticket | Client retries UpdateTreezCart |

### POS Rejection (during approval)

| Failure | Order State | Treez State | OnFleet State | Recovery |
|---------|-------------|-------------|---------------|----------|
| Treez UpsertTicket fails | Unapproved | No ticket | Not attempted | Admin retries |
| OnFleet CreateTask fails | Unapproved | **Ticket exists → ROLLED BACK** | No task | Treez ticket removed by rollback |
| DynamoDB upsert fails | In-memory updated, DB unchanged | Ticket created (NOT rolled back) | Task created (NOT rolled back) | **Inconsistent state** — manual cleanup needed |

### Timeout

All operations wrap in a 5-minute timeout (`app.CtxTimeout`). When timeout occurs:
- Current operation is cancelled via `context.WithTimeout`
- Partial side effects may have completed (Treez ticket created, OnFleet task created)
- No automatic rollback on timeout
- Order state in DynamoDB reflects last successful upsert

---

## 13. CRM Subscription Flow

### SubscribeToCrm — End-to-End (crm.go:16-83)

**Trigger:** `SubscribeToCrm` GraphQL mutation from frontend (typically during checkout or account creation).

**Sequence:**
1. `Services.App.GetAllByCategory(ctx, AppCategoryTypeCrm, key)` — discover all CRM apps for store
2. Filter to enabled apps (`GetStatus() == true`) that are used for subscriptions (`GetUsedForSubscription() == true`)
3. Validate input: at least one of `email` or `phone` required
4. Build payload: `SubscribeCRM{input, accountID, storeID}` → JSON marshal
5. Fan-out with goroutines:
   ```
   for each subscription-enabled CRM app:
     go sendSubscription(ctx, app, data)
   WaitGroup.Wait()
   ```
6. `sendSubscription()` publishes SNS event:
   - Event type: `"app/{handler}"` (e.g., `"app/klaviyo"`, `"app/omnisend"`)
   - Message attributes: `event_type`, `publisher: "gapcommerce"`
   - Payload: JSON with subscriber info + accountID + storeID

**Error handling:** Individual SNS publish errors are logged but NOT propagated. The mutation returns nil (success) even if some providers fail. This is fire-and-forget.

**Typical CRM providers:** Klaviyo, Omnisend, Iterable (identified by handler string).

---

## 14. ID Verification Flow

### Berbix (idcheck.go:20-132)

**Step 1: CreateBerbixClientToken (lines 20-48)**
- Called early in checkout when age verification is required
- Retrieves Berbix app config (handler: `"berbix"`)
- Creates Berbix service with `SecretKey` and `Template`
- `berbix.CreateTransaction(ctx, input.UserID)` → generates session
- Returns: `BerbixClientToken{AccessToken, ClientToken, RefreshToken, ExpiresAt}`
- Frontend uses `ClientToken` to initialize Berbix SDK modal

**Step 2: GetBerbixTransactionResult (lines 50-132)**
- Called after user completes verification in Berbix modal
- `berbix.GetTransactionResult(ctx, accessToken, refreshToken)` → fetch results
- Extract ID data from response:
  - `PhotoID.IDNumber` → `Account.DriverLicenseID`
  - `PhotoID.DateOfBirth` → `Account.DateOfBirth`
  - `PhotoID.ExpiryDate` → `Account.DriverLicenseIDExpirationDate` (defaults to 3 years if not provided)
- Persist to DynamoDB: `Services.Account.Upsert()`
- Update Cognito custom attributes:
  - `BerbixID`, `BerbixVerified`, `BerbixRefreshToken`, `BerbixError`
- Return: `BerbixTransactionResult{ID, ValidatedUser, ReasonNotValidated}`

**Pass/fail:** `resp.Action == ActionAccept` → verified. Any other action → not verified, reason included in response.

### IDScan (idcheck.go:149-259)

**Single mutation: IDScanVerification**
- Retrieves IDScan app config (handler: `"idscan"`)
- Creates service with `SecretKey`
- `service.Validate(ctx, request)` → sends document photo + selfie

**Cascade of confidence checks (ALL must pass):**

| Check | Threshold | Failure Code |
|-------|-----------|--------------|
| Document confidence | >= 70 | Generic failure |
| Face match confidence | >= 70 | Generic failure |
| Anti-spoofing confidence | >= 70 | Generic failure |
| Age verification | >= 21 years | `IDScanValidationErrorCodeAgeBelowLimit` |

**Age calculation (lines 267-282):**
```
duration = time.Since(parsedDOB)
ageLimit = 21 * 365 * 24 * time.Hour
underage = duration < ageLimit
```

**On any failure:**
- Update Cognito with failure status
- Return error response (not exception — response with error details)

**On success:**
- Extract: Document.ID → DriverLicenseID, Document.DOB → DateOfBirth, Document.Expires → ExpirationDate
- Persist to DynamoDB account
- Update Cognito: `BerbixVerified = true`, `BerbixID = response.RequestID` (note: uses Berbix naming for both providers — legacy)

---

## 15. GraphQL Resolvers — Complete Order-Related Listing

### Queries (resolver/order.go)

| Query | Line | App Method | Description |
|-------|------|------------|-------------|
| `GetOrder` | 11 | `GetOrder()` | Single order by ID with timezone conversion |
| `ListStoreOrders` | 18 | `ListStoreOrders()` | Paginated order search with query string |
| `CustomerOrderHistory` | 25 | `CustomerOrderHistory()` | Customer orders by email with cursor pagination |
| `GetOrderDeliveryEta` | 79 | `GetOrderDeliveryEta()` | OnFleet delivery ETA for order |
| `GetTreezPayToken` | 175 | `GetTreezPayToken()` | TreezPay OAuth access token |

### Mutations (resolver/order.go)

| Mutation | Line | App Method | Description |
|----------|------|------------|-------------|
| `UpdateCart` | 32 | `UpdateCart()` | Generic cart update with price calculation |
| `UpdateJaneCart` | 41 | `UpdateJaneCart()` | Jane cart passthrough |
| `SubmitCart` | 50 | `SubmitCart()` | Publish OrderCompleted SNS event |
| `CreateOrder` | 64 | `panic("")` | **STUB — not implemented** |
| `UpdateOrder` | 68 | `panic("")` | **STUB — not implemented** |
| `UpdateOrderStatus` | 72 | `UpdateOrderStatus()` | Status transition with POS sync |
| `AddAeropayPaymentDetailToOrder` | 86 | `AddAeropayPaymentDetailToOrder()` | Attach AeroPay transaction |
| `UpdateOrderDetails` | 93 | `UpdateOrderDetails()` | Update delivery/payment/promo/items |
| `UpdateOrderLineItemsBarCodes` | 100 | `UpdateOrderLineItemsBarCodes()` | Scan barcodes for fulfillment |
| `CheckOrderLineItemsBarCodes` | 107 | `CheckOrderLineItemsBarCodes()` | Validate barcode completeness |
| `CreateSwifterSession` | 114 | `CreateSwifterSession()` | Swifter OAuth session |
| `SubmitSwifterPayment` | 122 | `SubmitSwifterPayment()` | Swifter payment completion |
| `CustomerCancelOrder` | 129 | `CustomerCancelOrder()` | Customer-initiated cancel |
| `UpdateTreezCart` | 136 | `UpdateTreezCart()` | Treez cart with optional draft ticket |
| `UpdateDutchieCart` | 145 | `UpdateDutchieCart()` | Dutchie cart sync |
| `CreateStrongholdPayLink` | 154 | `CreateStrongholdPayLink()` | Stronghold payment link |
| `UpdateStrongholdChargeID` | 161 | `UpdateStrongholdChargeID()` | Stronghold charge callback |
| `SubmitKioskCheckout` | 168 | `SubmitKioskCheckout()` | Kiosk checkout (synchronous Treez) |

### Promotion Resolvers (resolver/promotion.go)

| Operation | Type | Line | Description |
|-----------|------|------|-------------|
| `ListPromotions` | Query | 11 | All promotions (active + inactive) |
| `ListActivePromotions` | Query | 33 | Active promotions only |
| `GetActivePromotion` | Query | 55 | Single active promotion by ID |
| `GetPromotion` | Query | 73 | Single promotion (any status) |
| `ApplyPromoCode` | Mutation | 148 | Apply coupon code to order |
| `ListJaneSpecials` | Query | 163 | Jane specials with caching |

### CRM Resolver (resolver/crm.go)

| Operation | Type | Line | Description |
|-----------|------|------|-------------|
| `SubscribeToCrm` | Mutation | 11 | Fan-out CRM subscription via SNS |

### ID Verification Resolvers (resolver/idcheck.go)

| Operation | Type | Line | Description |
|-----------|------|------|-------------|
| `CreateBerbixClientToken` | Mutation | 12 | Initialize Berbix session |
| `GetBerbixTransactionResult` | Query | 19 | Fetch Berbix verification result |
| `IDScanVerification` | Mutation | 28 | Run IDScan multi-check verification |

### Delivery Resolver (resolver/delivery.go)

| Operation | Type | Line | Description |
|-----------|------|------|-------------|
| `GetDeliveryEta` | Query | 26 | Calculate delivery ETA for location |

---

## 16. The TreezPay Duplicate Ticket Bug

*Source: `docs/TREEZ_DUPLICATE_TICKET_ENTITY_ID_BUG.md`*

### Root Cause — Primary Race Condition

The bug occurs in `UpdateTreezCart` when two concurrent requests arrive for the same order:

1. Both requests carry the **request payload** (not fresh DB state) containing stale order data
2. Both see "no existing ticket ID" in `order.TreezPayTicketIdsByStore`
3. Both call `UpsertTicket(nil, request)` → each creates a new ticket
4. Both tickets share the same `ExternalOrderNumber` (= `order.EntityID`)
5. Request A hasn't persisted its updated order (with new ticket ID) before Request B reads

**Critical code path:**
- Line 2309: `getTreezPayTicketByProviderID(order, ...)` reads from REQUEST payload, not DB
- Line 2340: `UpsertTicket(ctx, treezOrderID, ...)` → `treezOrderID` is nil from stale data
- Line 2426: `Services.Order.Upsert(...)` → persistence happens AFTER ticket creation

### Root Cause — Secondary Path (CANNOT_MODIFY_TICKET)

When `UpsertTicket(existingID, ...)` returns `CANNOT_MODIFY_TICKET`:
1. Line 2341-2347: Remove old ticket ID from order
2. Retry with `UpsertTicket(nil, ...)` → creates a new ticket
3. Ticket removal is async (`go a.removeTreezTicket(...)` at line 3175)
4. Creates duplicate if removal doesn't complete before other operations check

### Reproduction Steps

**Scenario 1 — Rapid Refresh:**
1. User enters checkout, no Treez ticket yet
2. Frontend sends `UpdateTreezCart` with `createDraftTicket=true`
3. Request A starts processing, creates ticket (slow network to Treez)
4. User rapidly refreshes or connection stalls Request A
5. Frontend resends with same order state (still no ticket ID in payload)
6. Request B processes concurrently, also sees no ticket
7. Both create tickets with same `ExternalOrderNumber`

**Scenario 2 — CANNOT_MODIFY_TICKET:**
1. Order has existing ticket in paid/non-modifiable state
2. `UpdateTreezCart` called with changes
3. `UpsertTicket` returns `CANNOT_MODIFY_TICKET`
4. Code removes ticket ID and retries with nil → creates new ticket
5. Old ticket removal is async, may not complete before next check

### Conditions Required

All must be true:
1. TreezPay enabled for store
2. `createDraftTicket = true`
3. Two concurrent `UpdateTreezCart` calls for same order
4. First call hasn't persisted before second starts
5. Both read stale request payload

### Impact

- Multiple Treez tickets with identical `ExternalOrderNumber`
- POS/backoffice confusion (multiple tickets for single order)
- Downstream systems assuming 1:1 order-to-ticket mapping break
- Intermittent — depends on timing, Treez API latency, and frontend retry behavior

### Proposed Fixes

**Option 1 — Idempotency Lock (Recommended):**
Guard ticket creation with `(order.EntityID, providerStoreID)` lock. Re-read latest order from DB while holding lock. Only one request can create per key.

**Option 2 — Re-read Latest Order:**
Before creating ticket, fetch latest order from DB instead of using request payload. Reduces stale data but still vulnerable to true parallel race without atomic write.

**Option 3 — Disable Fallback Create:**
On `CANNOT_MODIFY_TICKET`, return error to caller instead of creating new ticket. Prevents secondary duplicate path.

**Option 4 — Treez API Dedup Check:**
Before `UpsertTicket(nil, ...)`, query Treez for existing ticket with same `ExternalOrderNumber`. Reuse if found.

**Option 5 — Synchronous Ticket Removal:**
Make `RemoveTicket()` call synchronous (not async) on `CANNOT_MODIFY_TICKET`, reducing overlap window.

### Status

Not yet fixed as of commit `3099116`.

---

## 17. Customer Order Cancellation

### `CustomerCancelOrder()` (handle_order.go:1975-2051)

**Sequence:**
1. Fetch store and order
2. Validate order exists
3. Cancel Jane order: `cancelJaneOrder()` — calls Jane API if Jane app configured
4. Cancel Treez ticket: `cancelTreezTicket()` — calls `treezService.RemoveTicket()` if TreezOrderID exists
5. Cancel OnFleet task: `cancelOnFleetTask()` — calls `onFleetService.RemoveTask()` if OnFleet reference exists
6. Set `order.Status = OrderStatusTypeDeclined`
7. Upsert order

Each cancellation step is independent — failure in one doesn't prevent the others.

---

## 18. Complete Order Status State Machine

```
                    +──────────────────+
                    |  AbandonedCart    |  ← Initial state (every cart)
                    +──────────────────+
                            |
                     SubmitCart (SNS)
                     + worker processing
                            |
                            v
                    +──────────────────+
               ┌──> |     Pending      |  ← Worker assigns invoice, transitions
               |    +──────────────────+
               |            |         \
               |    (Admin approves)   (Admin/customer declines)
               |            |           \
               |            v            v
               |    +──────────────+    +──────────────────+
               |    |   Approved   |    |    Declined      |
               |    +──────────────+    +──────────────────+
               |            |                   |
               |    (Delivery completes)  (Can re-submit)
               |            |                   |
               |            v                   |
               |    +──────────────────+        |
               |    | DeliveryFinished |        |
               |    +──────────────────+        |
               |            |                   |
               |            v                   |
               |    +──────────────────+        |
               |    |    Delivered     |        |
               |    +──────────────────+        |
               |                                |
               +────────────────────────────────+
                    (Declined → Pending only)
```

**Constraints:**
- Same-status transitions are rejected
- From Declined: only Pending is allowed
- DeliveryFinished and Delivered trigger `handleCompleteOrder()` (Treez ticket completion)

---

## 19. SNS Events Published by This Service

| Event Type | Constant | When | Payload |
|------------|----------|------|---------|
| `OrderCompleted` | `worker.OrderCompletedEventType` | SubmitCart | `{account_id, store_id, entity_id}` |
| `OrderKioskCompleted` | `worker.OrderKioskCompletedEventType` | SubmitKioskCheckout | `{account_id, store_id, entity_id}` |
| `OrderStatusChangedNotify` | `OrderStatusChangedNotifyEventType` | UpdateOrderStatus | `{account_id, store_id, entity_id}` |
| `app/{handler}` | Dynamic | SubscribeToCrm | `{SubscribeCRMInput, account_id, store_id}` |

All published to the SNS topic configured in `a.Config.TopicArn` with message attributes `event_type` and `publisher: "gapcommerce"`.

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-srv (commit 3099116)*
