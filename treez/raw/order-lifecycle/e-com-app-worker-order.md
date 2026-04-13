# e-com-app-worker — Order-Related Event Processing

> Detailed analysis of every order-related processor in the e-com-app-worker Lambda. This worker consumes SQS events and handles order submission to POS/ERP systems, discount redemption, and order event tracking.

---

## 1. place-order-treez — Full Processor Chain

The `place-order-treez` event type triggers **three processors in sequence**:

```
PlaceOrderTreez → AlpineIQDiscount → StickyDiscount
```

Defined in `internal/app/manager.go:50-56`. If `PlaceOrderTreez` returns an error, the subsequent discount processors **do not run** — the pipeline aborts and the SQS message stays on the queue for retry.

### 1.1 PlaceOrderTreez — Main Processor

**Source**: `internal/processor/place_order_treez.go`

The `Process()` method is broken into four well-defined phases:

#### Phase 1: `loadProcessInputs` (lines 130-176)

1. Assert event is `OrderEvent` (account_id, store_id, entity_id)
2. Derive S3 key: `util.GetKey(accountID, storeID, storeKey)` → load store config
3. Load full order from DynamoDB via `order.GetOrderByID(tableName, entityID)`
4. Load Treez service:
   - Fetch app config via `app.GetByHandler(appKey, "treez")`
   - Check `treezApp.Status` is enabled
   - **Check `treezApp.AutomaticApproval` is enabled** — if not, processor returns nil (no-op). This is how stores opt in/out of automatic Treez submission.
   - Match Treez credentials by `order.ProviderStoreID` — the Treez app config has multiple credential sets, one per physical store
   - Construct Treez service with `clientID`, `apiKey`, `dispensaryName`, optional headless credentials
   - Sandbox mode supported via `treezApp.Sandbox` flag
5. Set context value: `tracking.OrderIDKey → order.EntityID`
6. If no Treez service could be created (config missing, disabled, no matching credentials), return nil — **message is deleted, no error**
7. Load account by email from DynamoDB: `account.GetByEmail(email, tableName)`

Returns `treezProcessInputs{data, store, order, account, treezSrv}`.

#### Phase 2: `resolveTreezOrderContext` (lines 178-204)

Determines the Treez customer ID and the initial order status.

**Customer handling** (`handleTreezCustomer`, lines 520-622):

Three paths:

**Path A — Known customer (TreezIds already stored on account)**:
1. Iterate `account.TreezIds` to find a match by `order.ProviderStoreID`
2. If found: fetch Treez customer by ID via `service.GetCustomerByID(treezID.ID)`
3. Call `checkOrderType()` — if order is medical but Treez customer is adult-use, update customer to medical with MMID + expiration date
4. If delivery order: update customer address on Treez with the order's delivery address
5. Return `(customerID, isVerified)` — verified means `VerificationStatus == "VERIFIED"`

**Path B — Unknown customer, exists in Treez**:
1. Lookup by email: `treezService.GetCustomerByEmail(account.Email)`
2. If not found, lookup by phone: `treezService.GetCustomerByPhone(phone)`
3. If found in Treez: add the Treez customer ID to `account.TreezIds` and persist
4. If delivery order: update customer address on Treez
5. Call `checkOrderType()` for medical→adult conversion
6. Return `(customerID, isVerified)`

**Path C — Customer does not exist in Treez**:
1. Build `treez.CustomerRequest` from account + order data, including:
   - Name, email, phone, DOB, gender, driver's license, medical ID, addresses
   - Patient type: Adult (default), Medical, or MedicalMMID (if medical ID number exists)
   - DOB fallback: if neither account nor order has DOB, uses `now - 22 years` (constant `minTreezAllowedAge`)
2. Call `treezService.CreateCustomer(ctx, request)`
3. Store new Treez ID on Gap Commerce account
4. Return `(customerID, false)` — new customers are never verified

**Initial status selection**:
- Default: `treez.VerificationPendingOrderStatusType`
- If customer is verified: `treez.AwaitingProccessingOrderStatusType`
- After photo upload (see below): may revert to `VerificationPendingOrderStatusType`

**Verification photo upload** (`uploadVerificationPhotosToTreez`, lines 299-418):

This is the most complex sub-step. It decides which photos to upload based on the customer's history.

**Upload plan construction** (`buildVerificationPhotoUploadPlan`, lines 314-355):

Uses `account.TreezCustomerInfo` — an array of per-Treez-customer metadata:

1. Search for a matching `TreezCustomerInfo` entry by Treez customer ID
2. **Existing customer**: read `TreezDlUpdated` and `TreezMedicalIDUpdated` flags (these indicate "a new photo was uploaded to Gap Commerce and needs to be pushed to Treez")
   - Reset both flags to `false` (they'll be re-set to `true` if upload fails)
3. **New customer (no entry found)**: always upload DL; upload medical ID if `account.MedicalDocumentPhotoURI` is non-empty. Append new `TreezCustomerInfo` entry.
4. `shouldAttemptDLUpload` is gated by **store feature flags**:
   - Delivery order: requires `AllowIDVerificationForDelivery == true`
   - Pickup order: requires `AllowIDVerificationForPickup == true`
   - If feature flags are nil: no DL upload attempted

**Upload execution** (`applyVerificationPhotoUploadPlan`, lines 357-418):

For each photo type (DL, Medical ID):
1. Get S3 presigned URL via `file.GetAssetFile(ctx, photoPath)`
2. HTTP GET the image bytes from the presigned URL
3. Base64-encode the image
4. Call `treezService.UploadCustomerPhoto()` with `FileTypeDriversLicence` or `FileTypeDoctorPermit`
5. On success: set status to `VerificationPendingOrderStatusType`, persist account to DynamoDB
6. On failure: **error is logged + Sentry-captured, but does NOT fail the processor**. Status remains unchanged.

**Test coverage** (lines 23-658 of `place_order_treez_test.go`):
- `TestBuildVerificationPhotoUploadPlan_NewCustomerPickupWithMedicalPhoto` — new customer, both DL + medical
- `TestBuildVerificationPhotoUploadPlan_ExistingCustomerUsesPendingFlags` — uses TreezDlUpdated/TreezMedicalIDUpdated
- `TestBuildVerificationPhotoUploadPlan_DeliveryWithoutFeatureFlagSkipsDLUpload` — delivery without AllowIDVerificationForDelivery
- `TestBuildVerificationPhotoUploadPlan_NoPendingFlagsProducesNoUploadActions` — existing customer with nothing pending
- `TestUploadVerificationPhotosToTreez_UploadsPendingPhotos` — full upload with mock HTTP server
- `TestUploadVerificationPhotosToTreez_KeepsStatusWhenDLUploadFails` — error tolerance

#### Phase 3: `prepareTreezTicketRequest` (lines 206-237)

1. Map order to Treez ticket via `treez.MapOrderToTreezTicket(order, customerID, status, timezone, false, true)` (from `e-com-lib`)
2. **Payment type handling** — per-type resolution:

   **Stronghold** (`resolveStrongholdChargeID`, lines 955-979):
   - If `ChargeID` already exists on order: no-op
   - Otherwise: look up Stronghold credentials from app config by `storeProviderID`
   - Call `stronghold.GetPayLink(ctx, paylinkID)` to retrieve the charge
   - Set `detail.ChargeID` from the PayLink's charge

   **AeroPay** (`resolveAeroPayTransactionID`, lines 1026-1060):
   - If `TransactionID` already exists: no-op
   - If UUID is nil: no-op (instrumentation gap noted in code)
   - Lookup AeroPay credentials from app config by `storeProviderID`
   - Call `aeropay.SearchTransaction(SearchTypeUUID, uuid)`
   - Use first transaction from results, warn if multiple returned

   **TreezPay**:
   - Check `PaymentMethod` field: if `"Credit"`, set `isTreezPayCredit = true`
   - This flags the ticket to **skip UpsertTicket entirely** (credit card orders are handled differently — the ticket was already created by TreezPay)

3. Set HCH (hosted checkout) payment info on ticket:
   - Maps payment type → Treez payment method: AeroPay → `PaymentMethodAeropay`, Swifter → `PaymentMethodSwifter`, Stronghold → `PaymentMethodStronghold`
   - Sets `PaymentAuthorization`, `PaymentChargeId`, `PaymentStatus` = `Authorized`
   - Payment source is always `EccommerceOrderSourceType`

4. **Error handling for payment resolution**: Stronghold/AeroPay errors are logged+Sentry-captured via `captureProcessOrderError` but **do not fail the processor**

#### Phase 4: `submitTreezTicket` + `finalizeApprovedOrder` (lines 239-288)

**Ticket submission** (lines 239-258):
1. If `isTreezPayCredit`: skip submission entirely (ticket already exists from TreezPay flow)
2. Otherwise: call `treezSrv.UpsertTicket(ctx, order.TreezOrderID, ticket)`
   - If `order.TreezOrderID` is nil: creates a new ticket
   - If `order.TreezOrderID` is set: updates the existing ticket
3. On Treez API error: return error (message stays on queue for retry)

**Order finalization** (lines 260-288):
1. **Clean up unused draft orders** (`clearUnusedTreezDraftOrders`, lines 1107-1135):
   - Iterate `order.TreezPayTicketIdsByStore`
   - For each ticket where `ProviderStoreID != order.ProviderStoreID`: load that store's Treez service and call `treezSrv.RemoveTicket(ticketID)`
   - These are draft tickets created by TreezPay for other stores during multi-store shopping. Error is logged but **does not fail**.
2. If ticket was not skipped: set `order.TreezOrderID` and `order.TreezOrderNumber` from the Treez response
3. **Set order status to `Approved`**: `order.Status = model.OrderStatusTypeApproved`
4. Persist order to DynamoDB
5. If upsert fails: return error (fatal)

**Important**: Unlike `place-order-blaze`, this processor does **not** publish an SNS completion event. The status change to `Approved` is the output; downstream processing is triggered by other mechanisms.

### 1.2 AlpineIQ Discount Redemption

**Source**: `internal/processor/process_alpineiq_discount.go`

Runs as the second processor in the Treez chain.

**Flow**:
1. Get AlpineIQ service from app config. If app is missing, disabled, or misconfigured → **return nil (no error, processor skipped)**
2. Load store config, load order from DynamoDB
3. Check `order.AlpineiqRedemptionURL` — if empty → return nil (no loyalty discount to redeem)
4. Call `client.ApplyDiscountRedemption(ctx, redemptionURL)` — this finalizes the discount that was previewed during checkout
5. Store returned `redemptionID` on `order.AlpineiqRedemptionID`
6. Persist order to DynamoDB

**Error handling**:
- AlpineIQ service creation failure: **returns nil** (message deleted)
- Store/order load failure: **returns error + Sentry capture** (message stays on queue)
- `ApplyDiscountRedemption` failure: **returns error + Sentry capture** (message stays on queue)
- Order upsert failure: **returns error + Sentry capture** (message stays on queue)

**Note**: Because this is chained in the pipeline, if this processor returns an error, the SQS message is **not deleted** and the entire chain will be retried (including re-running PlaceOrderTreez, which will upsert the same ticket via `UpsertTicket`).

### 1.3 Sticky Discount Redemption

**Source**: `internal/processor/process_sticky_discount.go`

Runs as the third processor in the Treez chain.

**Flow**:
1. Load store config, load order from DynamoDB
2. Get Sticky service from app config:
   - If app handler returns `ErrStickyAppDisabled` → return nil (skipped, logged as info)
   - If service is nil for other reasons → **return nil** (Sentry-captured but skipped)
3. Load account by email from DynamoDB
4. Check `account.StickyCustomerID` — if nil → return nil (Sentry-captured)
5. Check `order.DiscountItems` has at least one item with a non-nil ID
6. Parse the discount ID: `getDiscountIDFromRewardsID` splits on `_` and takes the third segment
   - Format: `{rewardType}_{pageTitle}_{posDiscountId}` (matches UI construction)
7. Call `client.RedeemReward(ctx, RedeemRequest{orderID, customerID, discountID})`

**Error handling**:
- Missing StickyCustomerID, empty DiscountItems, nil DiscountItem.ID: **return nil** (Sentry-captured, message deleted)
- Sticky API RedeemReward failure: **returns error + Sentry capture** (message stays on queue)

**Bug**: `isValidStickyApp` always returns `false` (line 179 returns `false` unconditionally after the nil checks). This means **the Sticky discount processor is effectively disabled for all stores** unless there's a different code path constructing the service. The final `return false` on line 179 appears to be a bug — it should be `return true`.

---

## 2. place-order-blaze — Full Flow

**Source**: `internal/processor/place_order_blaze.go`

### Flow

1. **Load Blaze service** (`getBlazeService`, lines 321-355):
   - Get app config by handler `"blaze"`, assert `Status` is enabled
   - Sandbox mode: swap `PartnerKey`/`AuthKey` for `DevPartnerKey`/`DevAuthKey`
   - Construct `blaze.Service` with partner key, auth key, sandbox flag

2. **Load order + account**: Store config from S3, order from DynamoDB by entity_id, account by email

3. **Get driver license photo**: If account has `DriverLicensePhoto`, get presigned S3 URL via `file.GetAssetFile`. Error is logged but **non-fatal**.

4. **Stock verification** (lines 99-128):
   - For each line item: call `blazeSrv.GetProduct(productID)`
   - If product is nil, inactive, or out of stock: mark `ln.InStock = false` and reduce `SubtotalPrice`, `TotalLineItemsPrice`, `TotalPrice` by `price * quantity`
   - Floor `TotalPrice` at 0 (discounts could make it negative)
   - If **all items** are out of stock (totalInStock == 0):
     - Upsert order to DynamoDB (saves the updated line item statuses)
     - **Publish `OrderOutOfStockNotifyEventType` SNS event** — this is the only SNS event published by any processor in this file
     - Return nil (message deleted, no order placed)

5. **Customer management** (lines 145-207):
   - Find user by email via `blazeSrv.FindUser`
   - If not found: register new user with `blazePassword = "5Tv6b4ZfFT"` (hardcoded constant), marketing source "Emberz Delivery Website"
   - If DOB on order: update user DOB if mismatched
   - Login user to get session (required for cart operations)
   - If login fails with `ErrUserNotExist`: update user password and retry login

6. **Cart construction** (lines 209-299):
   - Get active cart via `blazeSrv.GetActiveCart(userID, sessionID)`
   - Set delivery address from order
   - Map in-stock line items to Blaze cart items (productID + quantity)
   - Set `ExternalID` to order's `EntityID`
   - Set `ConsumerType` to `AdultUseConsume`
   - **Payment mapping**: Cash (default), Check, or CreditCard based on `order.PaymentDetails.Type`
   - Delivery type: `"Delivery"` or `"PickUp"` based on `order.DeliveryDetails.Pickup`
   - If `DeliverAfter` is set: convert to millisecond timestamp for `CompleteAfter`
   - Memo field: formatted string with delivery type, driver license photo URL, payment type
   - Source: `"Gap Commerce"`
   - **Promotions**: Map order's `DiscountItems` to Blaze `PromotionReqLog` entries (only items with a `BlazeID`). Rate is converted from cents to dollars with rounding.

7. **Submit cart**: `blazeSrv.SubmitCart(userID, cartID, cart)` — this places the order in Blaze

8. **Post-submission**:
   - Store `cart.ID` as `order.BlazeCartID`
   - Upsert order to DynamoDB
   - Upsert account (`upsertAccount`): if existing account, add order total to `TotalOrderExpense`; if no account, create a new one with key `"c#"` and `HasAccount = false`
   - **Save Blaze order record** (`saveBlazeOrder`, lines 395-421): Write a `BlazeOrder` to DynamoDB with `EntityID = blazeCartID`, `Key = "blaze#"`, `OrderID = entityID`. This enables reverse-lookup from Blaze cart ID to internal order.

### Error Handling

All errors are **fatal** (returned, message stays on queue) except:
- Driver license photo fetch failure (logged only)
- All-items-out-of-stock (publishes notification, deletes message)

---

## 3. place-order-jane — Full Flow

**Source**: `internal/processor/place_order_jane.go`

### Flow

1. **Load store + order**: Standard S3/DynamoDB load pattern

2. **Early exit**: If `order.JaneCartID` is nil → return nil. Jane orders must have a cart ID already set by the checkout flow.

3. **Save Jane order record** (`saveJaneOrder`, lines 129-154):
   - Write `JaneOrder` to DynamoDB: `EntityID = janeCartID`, `Key = "j#"`, `OrderID = entityID`
   - This enables reverse-lookup from Jane cart ID to internal order
   - If save fails: return error "couldn't save order to jane"

4. **If `order.JaneOrderID` is nil**: Set `DeliveryDetails.TimeWindow = "NOT SET"`, return nil. The order hasn't been assigned a Jane order ID yet — this may mean it's still queued on Jane's side.

5. **Load Jane service** (`getJaneService`, lines 156-191):
   - Get app config by handler `"jane"`
   - Construct Jane service with extensive credentials: AppID, APIKey, ProductIndex, ReviewsIndex, StoresIndex, StoreID, Host, ClientID, ClientSecret, OperationClientID, OperationClientSecret
   - Uses CloudWatch tracker for observability

6. **Retrieve cart from Jane**: `janeSrv.RetrieveCart(ctx, janeOrderID)`
   - On error: **log and return nil** (non-fatal, message deleted)

7. **Enrich order with Jane data**:
   - **Discount mapping** (`mapDiscounts`, lines 193-272):
     - **Promo code discounts**: Compare `CheckoutPrice` vs `DiscountedCheckoutPrice` for each product. Calculate total discount in cents across all matching line items. Creates one `OrderDiscountItem` with `DiscountType = ProductDiscount`, `RateType = Amount`, `ApplyType = PromotionCode`.
     - **CRM redemptions**: Sum all `CRMRedemption.Reward.Amount` values (converted to cents). Creates one `OrderDiscountItem` with `DiscountType = AmountOffOrder`, `ApplyType = Automatically`.
   - Set `order.JaneStoreID` (string conversion of int64)
   - Set `order.JaneStoreName`
   - Set `order.Notes` from cart's checkout message
   - **AeroPay enrichment**: If `cart.AeropayPreauthorizationID` exists:
     - Set/create `order.PaymentDetails` with `Type = PaymentTypeAeropay`
     - Set `AeropayDetail.TransactionID` and `Amount`

8. **Persist order**: Upsert to DynamoDB with all enrichments

### Error Handling

- Jane cart retrieval failure: **returns nil** (message deleted, loss of enrichment data)
- DynamoDB save failures: **fatal** (message stays on queue)
- No SNS events published

---

## 4. place-order-klaviyo — Order Event Tracking

**Source**: `internal/processor/place_order_klaviyo.go`

### Flow

1. **Load store + order**: Standard pattern
2. **Load Klaviyo app config**: Get by handler `"klaviyo"`. If error, **returns nil** (silently skipped — note: this swallows the error). If app is nil or disabled, return nil.
3. **Map order to Klaviyo "Placed Order" event** (`mapKlaviyoEventFromOrder`, lines 78-124):
   - **Event type**: `"Placed Order"` (Klaviyo metric name)
   - **Order ID**: `OrderNumber` formatted as string, or `"gen_{uuid}"` if no order number
   - **Profile**: email, name, phone, address from delivery address. Country hardcoded to `"US"`.
   - **Value**: `TotalPrice / 100` (cents → dollars)
   - **Discount**: Takes the **first** discount item only — code and rate (rate / 100)
   - **Categories**: Deduplicated set of all line item categories
   - **Brands**: Deduplicated set of all line item brands
   - **Item names**: All line item names
   - **Line items**: Each with productID (from `ItemID`, not `ID`), name, quantity, imageURL, brand, categories, `ItemPrice = Price / 100`, `RowTotal = SalePrice / 100`
   - **Time**: Current time in RFC3339 format
   - **UniqueID**: `order.EntityID`
4. **Send to Klaviyo**: `srv.PlaceOrder(ctx, event)`

### Error Handling

- Klaviyo app lookup error: **returns nil** (message deleted, error lost)
- Klaviyo API call failure: **returns error** (message stays on queue)
- All price fields assume cents (integer) → dollars (float) conversion via `/100`

---

## 5. Error Handling Summary Per Processor

| Processor | Config/app missing | External API failure | DynamoDB failure | Overall |
|---|---|---|---|---|
| **PlaceOrderTreez** | nil return (message deleted) | Ticket upsert: fatal. Payment resolution: logged, non-fatal. Photo upload: logged, non-fatal. | Fatal | Mixed — core path is fatal, peripherals are tolerant |
| **AlpineIQDiscount** | nil return (message deleted) | Fatal (stays on queue) | Fatal | Fatal on API/DB errors |
| **StickyDiscount** | nil return (message deleted) | Fatal (stays on queue) | Fatal | Fatal on API/DB errors, but **effectively disabled due to bug** |
| **PlaceOrderBlaze** | nil return if service nil | Fatal for all Blaze API calls | Fatal | Mostly fatal, except photo and out-of-stock paths |
| **PlaceOrderJane** | nil return | Jane cart retrieve: non-fatal (nil return) | Fatal | Lenient on Jane, strict on DynamoDB |
| **PlaceOrderKlaviyo** | nil return (**swallows errors**) | Fatal | Fatal | Fatal except config load |

### What "fatal" means in context

When a processor returns an error, the SQS message is **not deleted**. It will be retried based on SQS visibility timeout and eventually land in a dead-letter queue (DLQ) after max receives. The processors are idempotent-ish via upsert semantics (`UpsertTicket`, `Upsert` on order), but side effects like Treez customer creation, Blaze user registration, and photo uploads are **not idempotent** — they may create duplicates on retry.

### Chained processor retry behavior (Treez)

If `AlpineIQDiscount` or `StickyDiscount` fails:
- The **entire chain retries** from `PlaceOrderTreez`
- `PlaceOrderTreez` will upsert the same ticket (safe) but may re-upload photos and re-create customer records
- This is a known tradeoff: discount redemption is important enough to retry the full chain

---

## Notable Details

### Hardcoded values
- `blazePassword = "5Tv6b4ZfFT"` — static password for all Blaze customer accounts (line 33 of place_order_blaze.go)
- `minTreezAllowedAge = 22` — DOB fallback age for Treez customers without a birthday
- `treezSource = "Treez Ecommerce 2.0"` — source identifier
- Marketing source for Blaze: `"Emberz Delivery Website"` (hardcoded, not configurable per store)

### Price representation
- Internal model: prices in **cents** (integer)
- Treez: mapped via `treez.MapOrderToTreezTicket` (in e-com-lib)
- Blaze: promotions are `rate / 100` (cents → dollars, rounded)
- Klaviyo: all prices divided by 100 (cents → dollars)
- Jane: discounts computed from string price fields (`ParseFloat`)

### `isValidStickyApp` bug
`internal/processor/process_sticky_discount.go:179` — the function returns `false` unconditionally after validating Status and APIKey. The final `return false` should be `return true`. As written, **no Sticky discount is ever redeemed** because `getStickyDiscountService` always gets an "invalid/misconfigured" error, which is then caught and returns nil (processor skipped silently).

### SNS event publishing
Only `PlaceOrderBlaze` publishes an SNS event (`OrderOutOfStockNotifyEventType`), and only for the all-items-out-of-stock case. The `publishEvent` function (`internal/processor/common.go`) serializes the event as JSON and publishes to the configured SNS topic with `event_type` and `publisher=gapcommerce` message attributes.

`PlaceOrderTreez` does **not** publish a completion event from this worker — the order status change to `Approved` is the signal for downstream processing.

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-app-worker @ bfbb85c*
