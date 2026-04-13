# e-com-notification-worker — Order Notification Processing

> Detailed breakdown of every event type this worker handles, including data loading, template rendering, recipient selection, and error behavior.

---

## 1. Event-by-Event Breakdown

### 1.1 `order/confirm_notify` — Order Confirmation

**Processor:** `confirmOrderProcessor` (`internal/processor/confirm_order.go`)

**Trigger:** Published by an upstream service (likely `e-com-order-worker` or `e-com-srv`) when an order is submitted/confirmed. The SQS message arrives with `event_type: "order/confirm_notify"`.

**Worker pipeline predicate:** `WhenConfirmOrder` matches `worker.OrderConfirmNotifyEventType`.

**Event payload model:** `NotificationEvent { account_id, store_id, entity_id }` — `entity_id` is the order ID.

**Data loading sequence:**
1. **Store config from S3** — Key: `{account_id}-{store_id}-{store_key_name}` (e.g. `acct123-store456-store.json`). Retrieves store branding, address, email settings, `DynamoOrderTableName`, `EmailNotificationActive` flag.
2. **EmailNotificationActive gate** — Uses `util.PointerToValue(store.EmailNotificationActive)`. If nil, PointerToValue returns `false` (zero value for bool), so **nil = not active = email skipped**.
3. **Order from DynamoDB** — `GetOrderByID(ctx, store.DynamoOrderTableName, entity_id)`. Retrieves full order with line items, delivery address, pricing.
4. **Notification template from S3** — Key: `{account_id}-{store_id}-{notification_setting_key_name}`. Handler name: `"order/confirm_notify"`. Retrieves the `Content` field (HTML template string) and other settings.

**Template context (Params struct):**

| Field | Source | Notes |
|-------|--------|-------|
| `StoreEmail` | `store.Email` | |
| `StorePhoneNumber` | `store.PhoneNumber` | |
| `CompanyName` | `store.Name` | |
| `StoreLogo` | `store.Logo` | |
| `CompanyEmail` | `store.Email` | Same as StoreEmail |
| `StoreAddress` | Formatted from store fields | `"{Address1}, {City}, {ProvinceCode} {Zip}, {CountryCode}"` |
| `StoreLogoSmall` | `store.LogoSmall` | |
| `StoreDomain` | `store.Domain` | |
| `Instagram` | `store.Instagram` | |
| `Facebook` | `store.Facebook` | |
| `Twitter` | `store.Twitter` | |
| `Yelp` | `store.Yelp` | |
| `OrderNumber` | `order.OrderNumber` | int |
| `DeliveryAddress` | Formatted from order | `"{Address1} {Address2}, {CityCode}, {ProvinceCode} {Zip}, {CountryCode}"` — prefers code fields over full names |
| `DeliveryAddress.FirstName` | `order.DeliveryAddress.FirstName` | |
| `DeliveryAddress.LastName` | `order.DeliveryAddress.LastName` | |
| `LineItems[]` | Built from `order.LineItems` | See line item details below |
| `SubtotalPrice` | `order.SubtotalPrice` | int (cents) |
| `TotalDiscounts` | `order.TotalDiscounts` | int (cents) |
| `TotalTax` | `order.TotalTax` | int (cents) |
| `TotalDelivery` | `order.TotalDelivery` | int (cents) |
| `TotalPrice` | `order.TotalPrice` | int (cents) |
| `TotalServiceFee` | `order.TotalServiceFee` | int (cents), only set if non-nil |
| `Email` | `order.Email` | Customer email |
| `Date` | `time.Now().Format("02-01-2006")` | DD-MM-YYYY format |
| `License` | `store.License` | |

**Line item construction:**
- `Name` = `lineItem.Name`
- `Quantity` = `lineItem.Quantity`
- `Price` = `lineItem.Variants[0].Price` (first variant's price, in cents)
- `Total` = `Price * Quantity`
- `Gram` = `lineItem.Variants[0].Type` (raw string like `"eighth_ounce"`, NOT converted to grams — differs from StatusChanged processor)
- `InStock` = `true` by default, `false` only if `lineItem.InStock` is explicitly `false`
- `Thumbnail` = `lineItem.Thumbnail`

**Template helper functions registered:** `inc` (index + 1), `formatPrice` (cents → `"%.2f"` dollars string).

**Subject line:** `"{StoreName} Order #{OrderNumber}: Confirmed"` — hardcoded format, not configurable.

**Recipient:** `order.Email` — the customer's email address.

**Source/From email:** `"{StoreName} - Order" <{source}>` where `source` is `store.FromTxEmail` if set, otherwise the global `transactional_email_source` env var.

---

### 1.2 `order/delivered_notify` — Delivery Notification

**Processor:** `deliveryProcessor` (`internal/processor/delivery.go`)

**Trigger:** Published when an order is delivered (likely from `api-srv` via OnFleet TaskCompleted or a POS status webhook).

**Worker pipeline predicate:** `WhenDelivery` matches `worker.OrderDeliveryNotifyEventType`.

**Event payload model:** `NotificationEvent { account_id, store_id, entity_id }`

**Data loading sequence:**
1. **Store config from S3** — Key: `{account_id}-{store_id}-{store_key_name}`. Same as confirm.
2. **EmailNotificationActive gate** — Uses `store.EmailNotificationActive != nil && !*store.EmailNotificationActive`. Different from confirm: **nil = active = email proceeds**. Only explicitly `false` skips.
3. **Order from DynamoDB** — Same as confirm.
4. **Notification template from S3** — **BUG:** Key is `{store_id}-{store_key_name}-{notification_key_name}` (uses `util.GetKey(data.StoreID, p.storeKey, p.notificationKey)` — note `accountID` is missing, replaced by `storeID` and `storeKey`). All other processors use `{account_id}-{store_id}-{notification_key_name}`. Handler name: `"order/delivered_notify"`.

**Template context:** Nearly identical to confirm, with differences:
- `Date` = `time.Now().Format("02-01-2006")` (same DD-MM-YYYY)
- Delivery address format: `"{Address1}, {City}, {Province} {Zip}, {CountryCode}"` — uses full `Province` name and only `CountryCode`, NOT the code-preferred logic of confirm/status_changed
- Store address format: uses `Province` and `CountryCode` (not ProvinceCode/CountryCode)
- `Gram` field: raw variant type string (same as confirm, NOT converted via `MapQuantity`)
- `TotalServiceFee`: NOT populated (missing from this processor)

**Template helper functions registered:** NONE — `template.Must(template.New("letter").Parse(...))` without `Funcs`. Templates using `formatPrice` or `inc` will fail at runtime.

**Subject line:** `"{StoreName} Order #{OrderNumber}: Delivered"` — hardcoded.

**Recipient:** `order.Email` — customer.

**Source/From email:** `"Orders - {StoreName}" <{source}>` — note different format from confirm (`"Orders - {StoreName}"` vs `"{StoreName} - Order"`).

---

### 1.3 `order/out_of_stock_notify` — Out of Stock Notification

**Processor:** `orderOutOfStockProcessor` (`internal/processor/order_out_of_stock.go`)

**Trigger:** Published when items in an order become unavailable (likely from POS sync after order submission).

**Worker pipeline predicate:** `WhenOrderOutOfStock` matches `worker.OrderOutOfStockNotifyEventType`.

**Event payload model:** `NotificationEvent { account_id, store_id, entity_id }`

**Data loading sequence:**
1. **Store config from S3** — Key: `{account_id}-{store_id}-{store_key_name}`.
2. **EmailNotificationActive gate** — Same pattern as delivery: `nil && !*` — **nil = active = email proceeds**.
3. **Order from DynamoDB** — Same as others.
4. **Notification template from S3** — Key: `{account_id}-{store_id}-{notification_key_name}`. Handler name: `"order/out_of_stock_notify"`.

**Template context differences:**
- `Date` = `time.Now().Format("2006")` — **YEAR ONLY** (e.g. `"2026"`), not a full date. Likely a bug or intentional for copyright-style footer use.
- Store address: uses `ProvinceCode` and `CountryCode` (consistent with confirm)
- Delivery address: uses full `Province` and `CountryCode` (same as delivery, not code-preferred)
- `Gram` field: raw variant type string (not converted)
- `TotalServiceFee`: NOT populated

**Template helper functions registered:** NONE.

**Subject line:** `"{StoreName} Order #{OrderNumber}: Declined"` — hardcoded. Note: says "Declined" even though the event is "out_of_stock". This is the customer-facing label for an out-of-stock order.

**Recipient:** `*order.Email` — customer (uses raw pointer dereference, not `PointerToValue` — will panic on nil).

**Source/From email:** `"{StoreName} - Order" <{source}>` — same format as confirm.

---

### 1.4 `order/status_changed` — Order Status Change

**Processor:** `statusChangedProcessor` (`internal/processor/status_changed.go`)

**Trigger:** Published when an order's status changes. This is the most general-purpose notification event — fired by `api-srv` webhook handlers (Treez, Jane, Blaze, OnFleet) and potentially by `e-com-srv` UpdateOrderStatus.

**Worker pipeline predicate:** `WhenStatusChanged` matches `worker.OrderStatusChangedNotifyEventType`.

**Event payload model:** `NotificationEvent { account_id, store_id, entity_id }`

**Data loading sequence:**
1. **Store config from S3** — Key: `{account_id}-{store_id}-{store_key_name}`.
2. **EmailNotificationActive gate** — Uses `util.PointerToValue(store.EmailNotificationActive)` — **nil = false = email skipped** (same as confirm).
3. **Order from DynamoDB** — Same.
4. **Order status check** — If `order.Status` is nil, returns error `"order status not set"`.
5. **Dynamic handler/action selection** — Based on `order.Status` (see mapping below).
6. **Notification template from S3** — Key: `{account_id}-{store_id}-{notification_key_name}`. Handler name varies by status.
7. **NotificationActive per-template gate** — Checks `tlp.NotificationActive`. If false, returns nil (email skipped). This is a **per-notification-type** toggle independent of the store-level `EmailNotificationActive`.

### 1.4.1 Status → Template/Handler Mapping

| `order.Status` value | Handler (S3 template key) | Email subject action | Notes |
|---|---|---|---|
| `model.OrderStatusTypePending` | `"order/confirm_notify"` | `"Confirmed"` | Same template as the dedicated confirm processor |
| `model.OrderStatusTypeDeclined` | `"order/declined_notify"` | `"Cancelled"` | Customer-facing label is "Cancelled", not "Declined" |
| `model.OrderStatusTypeDelivered` | `"order/completed_notify"` | `"Completed"` | Maps to "completed", not "delivered" — different template key |
| `model.OrderStatusTypeApproved` | `"order/ready_notify"` | `"Ready"` | When POS approves the order |
| `model.OrderStatusTypeDeliveryStarted` | `"order/on_route_notify"` | `"On Route"` | Driver dispatched |
| Any other status | — | — | Prints `"order status doesn't match any notification"` to stdout and returns nil (no email) |

**Template context (Params struct):** The `GetEmailParams` method builds a richer context than the other processors:

| Field | Source | Unique to this processor |
|-------|--------|--------------------------|
| `OrderType` | `"DELIVERY"` or `"PICKUP"` based on `order.DeliveryDetails.Pickup` | Yes |
| `OrderDetailsURL` | `"{StoreDomain}/account?section=orders&orderId={EntityID}"` | Yes |
| `StoreDealsUrl` | `"{StoreDomain}/deals"` | Yes |
| `TreezTicketNumber` | `order.TreezOrderNumber` | Yes |
| `TotalServiceFee` | `order.TotalServiceFee` | Also in confirm |
| `Date` | Delivery time window (see below) | Different format |
| `Gram` | Converted via `libTreez.MapQuantity()` → formatted as `"{N}G"` or `"each"` (0g) | Different from other processors |
| `StoreAddress` | From store config, OR overridden by `order.ProviderStoreFullAddress` if set | Unique override logic |

**Date/time window logic:**
- If `order.DeliveryDetails` is nil or `DeliverAfter`/`DeliverBefore` are nil → empty string
- Otherwise: `"MM/DD/YYYY HH:MM AM - HH:MM AM"` formatted in the store's timezone
- Uses `getDateOffset` to apply timezone offset to the date portion
- Example: `"12/31/2022 04:00 PM - 05:00 PM"`

**Gram mapping via `libTreez.MapQuantity`:**
- Converts variant type strings to gram weights: `"gram"` → 1, `"two_gram"` → 2, `"eighth_ounce"` → 3.5, `"half_gram"` → 0.5, `"each"` → 0, etc.
- Formatted as `"{N}G"` (e.g. `"3.5G"`, `"1G"`, `"2G"`) or `"each"` when gram == 0
- Other processors pass the raw type string instead (e.g. `"eighth_ounce"`)

**Subject line logic (multi-step):**
1. Default: `"{StoreName} Order #{OrderNumber}: {Action}"` where `OrderNumber` is the internal integer order number
2. **TreezOrderNumber override:** If `order.TreezOrderNumber` is set AND the template content contains the string `"TreezTicketNumber"`, the subject uses the Treez ticket number instead of the internal order number
3. **Custom subject template:** If `tlp.EmailSubject` is set and non-empty, the subject is rendered as a Go template using the same `Params` context. On template execution error, falls back to the default/Treez subject

**Template helper functions registered:** `inc` and `formatPrice` (same as confirm).

**Recipient:** `order.Email` — customer.

**Source/From email:** `"{StoreName} - Order" <{source}>` with `FromTxEmail` override.

---

### 1.5 `contact/business_owner` — Customer Contact Form

**Processor:** `contactBusinessOwnerProcessor` (`internal/processor/contact_business_owner.go`)

**Trigger:** Published when a customer submits a contact form on the storefront.

**Worker pipeline predicate:** `WhenContactBusinessOwner` matches `worker.ContactBusinessEventType`.

**Event payload model:** `ContactBusinessEvent { account_id, store_id, name, email, message }` — note: different model from order events.

**Data loading sequence:**
1. **Store config from S3** — Key: `{account_id}-{store_id}-{store_key_name}`.
2. **NO EmailNotificationActive check** — This processor does not gate on the email notification flag. Contact form emails always send.
3. **NO order loaded** — Not an order event, no DynamoDB call.
4. **Notification template from S3** — Key: `{account_id}-{store_id}-{notification_key_name}`. Handler name: `"contact/business_owner"`.

**Template context (ContactBusinessOwner struct — different from Params):**

| Field | Source |
|-------|--------|
| `StoreLogo` | `store.Logo` |
| `StoreLogoSmall` | `store.LogoSmall` |
| `StoreDomain` | `store.Domain` |
| `Instagram` | `store.Instagram` (pointer, not dereferenced) |
| `Facebook` | `store.Facebook` (pointer) |
| `Twitter` | `store.Twitter` (pointer) |
| `Yelp` | `store.Yelp` (pointer) |
| `Email` | `data.Email` (submitter's email) |
| `Name` | `data.Name` (submitter's name) |
| `Message` | `data.Message` (contact form message) |
| `CompanyName` | `store.Name` |
| `StoreAddress` | `store.Address1` (just address line 1, not formatted) |
| `StorePhoneNumber` | `store.PhoneNumber` |
| `StoreEmail` | `store.Email` |

**Template helper functions registered:** NONE.

**Subject line:** `"{Name} want to contact the business"` — note grammatical issue ("want" instead of "wants").

**Recipient:** `store.Email` — the **merchant/business owner**, NOT the customer. This is the only processor that sends to the store email.

**Source/From email:** `"{source}"` — plain source address with no display name formatting. Uses `FromTxEmail` override.

---

## 2. Status Changed Dynamic Template Selection

Full mapping in `statusChangedProcessor.Process()` (`status_changed.go:74-95`):

```
order.Status                         → Handler Key (S3)              → Subject Action  → Template in assets/
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
model.OrderStatusTypePending         → "order/confirm_notify"        → "Confirmed"     → received.html
model.OrderStatusTypeDeclined        → "order/declined_notify"       → "Cancelled"     → declined.html
model.OrderStatusTypeDelivered       → "order/completed_notify"      → "Completed"     → complete.html
model.OrderStatusTypeApproved        → "order/ready_notify"          → "Ready"         → package_and_ready.html
model.OrderStatusTypeDeliveryStarted → "order/on_route_notify"       → "On Route"      → driver_on_route.html
<any other status>                   → (no email sent)               → —               → —
```

**Statuses that do NOT trigger a notification:** Any status not in the list above is silently skipped. This includes at minimum:
- `OrderStatusTypeNew` / `OrderStatusTypeDraft`
- `OrderStatusTypeSubmitted`
- `OrderStatusTypeProcessing`
- Any custom/future status values

**Dual gating:** Both `store.EmailNotificationActive` (store-level) AND `tlp.NotificationActive` (per-notification-type) must be true/set. This is the only processor with the per-notification gate.

---

## 3. EmailNotificationActive Gating Behavior

There is an **inconsistency** across processors in how nil `EmailNotificationActive` is handled:

| Processor | Check pattern | nil behavior |
|-----------|--------------|--------------|
| ConfirmOrder | `!util.PointerToValue(store.EmailNotificationActive)` | nil → `false` → **email skipped** |
| StatusChanged | `!util.PointerToValue(store.EmailNotificationActive)` | nil → `false` → **email skipped** |
| Delivery | `store.EmailNotificationActive != nil && !*store.EmailNotificationActive` | nil → check skipped → **email proceeds** |
| OrderOutOfStock | `store.EmailNotificationActive != nil && !*store.EmailNotificationActive` | nil → check skipped → **email proceeds** |
| ContactBusinessOwner | (no check) | **email always proceeds** |

**Impact:** A store that has never explicitly set `EmailNotificationActive` (field is nil/missing from S3 config) will receive delivery and out-of-stock emails but NOT confirmation or status-changed emails. This is almost certainly a bug — the behavior should be consistent.

**In all cases:** When the check triggers (email skipped), the processor returns `nil` (not an error). The SQS message is still deleted. The event is silently dropped with no log entry.

---

## 4. Template Rendering

### 4.1 Template Source

Templates are stored in S3 as part of the notification settings JSON object. The `notification.Get(ctx, handlerName, key)` call retrieves a `NotificationSetting` struct for the given handler name. The `Content` field is the raw HTML template string.

The `assets/` directory in the repo contains **reference/default** HTML templates, but these are NOT embedded in the binary or used at runtime. The runtime templates always come from S3.

### 4.2 Helper Functions

| Function | Registered by | Signature | Behavior |
|----------|---------------|-----------|----------|
| `inc` | ConfirmOrder, StatusChanged | `func(int) int` | Returns `index + 1`. Used for 1-based loop counters in templates. |
| `formatPrice` | ConfirmOrder, StatusChanged | `func(int) string` | Converts cents to dollars: `float64(p) / 100` → `fmt.Sprintf("%.2f", v)`. E.g. `3500` → `"35.00"`. |

**NOT registered by:** Delivery, OrderOutOfStock, ContactBusinessOwner. If S3 templates for these event types use `{{formatPrice .TotalPrice}}` or `{{inc $index}}`, the template will **panic at execution time** (inside `template.Must`), crashing the Lambda invocation.

### 4.3 Price Representation

All price fields (`SubtotalPrice`, `TotalPrice`, `TotalTax`, `TotalDiscounts`, `TotalDelivery`, `TotalServiceFee`, line item `Price` and `Total`) are **integers in cents**. Templates must use `formatPrice` to display them as dollar amounts, or the raw cent value will appear in the email.

### 4.4 Template Execution

Templates are compiled and executed per-invocation (no caching). If the template string is malformed, `template.Must` panics. If template execution fails (e.g. missing field referenced in template), the error is returned, causing the SQS message to not be deleted (retry).

---

## 5. Error Scenarios

### 5.1 Store config not found in S3

`p.store.Get(ctx, sKey)` returns an error → processor returns error → SQS message NOT deleted → message will be retried after visibility timeout.

### 5.2 Order not found in DynamoDB

`p.order.GetOrderByID(ctx, tableName, entityID)` returns an error → processor returns error → SQS message retried.

### 5.3 Notification template not found in S3

`p.notification.Get(ctx, handler, key)` returns an error → processor returns error → SQS message retried.

### 5.4 Template rendering fails

- **Malformed template string:** `template.Must` panics → caught by Lambda's `recovered()` → Sentry gets the error → Lambda invocation fails → entire SQS batch retried.
- **Template execution error** (missing field, type mismatch): `te.Execute(&tpl, r)` returns error → processor returns error → SQS message retried.

### 5.5 SES send fails

`sendEmail` returns the SES error → processor returns error → SQS message retried. The `sendEmail` function (`common.go`) is a thin wrapper around `ses.SendEmail` — no retry logic, no error classification.

### 5.6 Order status nil (StatusChanged only)

Returns `errors.New("order status not set")` → SQS message retried.

### 5.7 Unmatched status (StatusChanged only)

Prints to stdout, returns nil → SQS message deleted. No email sent. No error.

### 5.8 Nil pointer panics

- **Delivery processor:** Dereferences `*order.DeliveryAddress` directly (line 96) — will panic if nil.
- **OutOfStock processor:** Same direct dereference of `*order.DeliveryAddress` (line 74).
- **OutOfStock processor:** Uses `*order.Email` directly for recipient (line 173) — will panic if nil.
- **All processors:** Access `order.LineItems[i].Variants[0]` without bounds checking — will panic if a line item has no variants.
- **All processors:** Dereference `*store.DynamoOrderTableName` without nil check — will panic if store config has no table name.

Panics are caught by the Lambda-level `recovered()` function, sent to Sentry, and cause the entire SQS batch to fail (retry all messages).

### 5.9 Delivery processor notification key bug

The delivery processor builds the notification settings key as `util.GetKey(data.StoreID, p.storeKey, p.notificationKey)` instead of `util.GetKey(data.AccountID, data.StoreID, p.notificationKey)`. This means:
- Key becomes `{storeID}-{storeKeyName}-{notificationKeyName}` (e.g. `store456-store.json-notification_settings.json`)
- All other processors use `{accountID}-{storeID}-{notificationKeyName}` (e.g. `acct123-store456-notification_settings.json`)
- This will fail to find the template in S3 unless there happens to be an object at the wrong key path. In practice, delivery notification templates likely never load correctly.

---

## 6. Cross-Processor Inconsistencies Summary

| Behavior | ConfirmOrder | Delivery | OutOfStock | StatusChanged | ContactBusiness |
|----------|-------------|----------|------------|---------------|-----------------|
| EmailNotificationActive nil behavior | skip | proceed | proceed | skip | no check |
| Notification key construction | correct | **BUG** | correct | correct | correct |
| Template funcs (inc, formatPrice) | yes | **no** | **no** | yes | no |
| Gram conversion (MapQuantity) | no (raw) | no (raw) | no (raw) | **yes** | N/A |
| TotalServiceFee populated | yes | **no** | **no** | yes | N/A |
| Date format | DD-MM-YYYY | DD-MM-YYYY | **YYYY only** | time window | N/A |
| Subject line format | `"StoreName - Order"` | `"Orders - StoreName"` | `"StoreName - Order"` | `"StoreName - Order"` | plain address |
| Delivery address format | code-preferred | full names | full names | code-preferred | N/A |
| Per-notification active check | no | no | no | **yes** | no |
| Nil-safe email recipient | yes | yes | **no** (`*order.Email`) | yes | N/A (store email) |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-notification-worker @ 4322e35*
