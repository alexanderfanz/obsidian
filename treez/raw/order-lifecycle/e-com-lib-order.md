# e-com-lib — Order Lifecycle Deep-Dive

> Exhaustive documentation of how e-com-lib defines and supports the order entity: models, enums, service interfaces, payment types, event schemas, and helper functions.

---

## 1. Order Model

**Source:** `pkg/model/model_gen.go` (generated from `pkg/graphql/modules/order/schema.graphqls`)

The `Order` struct has **54 fields**. All fields are pointer types (nullable). Prices are `*int` in **cents**.

### Identifiers & Keys

| Field | Go Type | Description |
|-------|---------|-------------|
| `EntityID` | `*string` | Unique order UUID |
| `Key` | `*string` | Sort key — `"d#"` for all orders in the primary table (see section 6) |
| `OrderNumber` | `*int` | Sequential display number, assigned post-submission by order-worker's invoice counter |

### Status & Control

| Field | Go Type | Description |
|-------|---------|-------------|
| `Status` | `*OrderStatusType` | Current lifecycle state (see section 2) |
| `Retry` | `*int` | Number of processing retry attempts |
| `ErrorReason` | `*string` | Human-readable error if processing failed |
| `DeclinedFromAdmin` | `*bool` | True if an admin (not system) declined the order |
| `Notes` | `*string` | Customer-provided order notes or curbside instructions |

### Customer Information

| Field | Go Type | Description |
|-------|---------|-------------|
| `Email` | `*string` | Customer email — used as GSI partition key (`email-index`) |
| `DateOfBirth` | `*scalar.Timestamp` | Customer DOB (Unix seconds) for age verification |
| `BuyerAcceptsMarketing` | `*bool` | Marketing opt-in consent |
| `OrderType` | `*AccountType` | `ADULT` or `MEDICAL` — determines compliance path |
| `OrderMedicalID` | `*string` | Medical license number (medical orders only) |
| `OrderMedicalIDExpirationDate` | `*scalar.Timestamp` | Medical ID expiration (medical orders only) |
| `StateOfResidence` | `*string` | Customer's state — passed to Treez for compliance |

### Line Items & Discounts

| Field | Go Type | Description |
|-------|---------|-------------|
| `LineItems` | `[]*LineItem` | Array of cart/order items (see section 4) |
| `DiscountItems` | `[]*OrderDiscountItem` | Array of applied promotions/rewards (see section 5) |

### Pricing Totals (all `*int`, in cents)

| Field | Description |
|-------|-------------|
| `SubtotalPrice` | Subtotal before tax and discounts |
| `TotalLineItemsPrice` | Sum of all line item prices (before discounts) |
| `TotalDiscounts` | Total discount amount applied |
| `TotalTax` | Total tax amount |
| `TotalDelivery` | Delivery/shipping fee |
| `TotalServiceFee` | Platform service fee |
| `TotalPrice` | Final total (subtotal + tax + delivery + fees - discounts) |

### Delivery

| Field | Go Type | Description |
|-------|---------|-------------|
| `DeliveryAddress` | `*OrderAddress` | Shipping/pickup address (14 subfields — see below) |
| `DeliveryDetails` | `*OrderDelivery` | Delivery method, timing, and tracking (see below) |
| `AllowCustomDeliveryFee` | `*bool` | Whether a custom delivery fee is sent to Treez |

### Payment

| Field | Go Type | Description |
|-------|---------|-------------|
| `PaymentDetails` | `*PaymentDetail` | Payment method and provider-specific details (see section 7) |
| `TreezPayTicketIdsByStore` | `[]*TreezPayTicket` | TreezPay ticket IDs per store location |

### POS Integration IDs

| Field | Go Type | Description |
|-------|---------|-------------|
| `TreezOrderID` | `*string` | Treez ERP ticket ID |
| `TreezOrderNumber` | `*string` | Treez ticket number |
| `JaneCartID` | `*string` | Jane POS cart ID |
| `JaneOrderID` | `*string` | Jane POS order ID |
| `BlazeCartID` | `*string` | Blaze POS cart ID |
| `DutchieOrderID` | `*string` | Dutchie POS order ID |

### Provider Store Fields (generic replacements for deprecated Jane-specific fields)

| Field | Go Type | Description |
|-------|---------|-------------|
| `ProviderStoreID` | `*string` | Store ID from any POS provider |
| `ProviderStoreName` | `*string` | Store name from provider |
| `ProviderStoreFullAddress` | `*string` | Store address from provider |
| `ProviderStorePhoneNumber` | `*string` | Store phone from provider |

**Deprecated fields** (still in schema with `@deprecated` directive):
- `JaneStoreID`, `JaneStoreName`, `JaneStoreFullAddress`, `JaneStorePhoneNumber` — use `Provider*` equivalents

### CRM Integration Fields

| Field | Go Type | Description |
|-------|---------|-------------|
| `AlpineiqTemplateID` | `*string` | AlpineIQ loyalty template used |
| `AlpineiqRedemptionID` | `*string` | AlpineIQ redemption transaction ID |
| `AlpineiqRedemptionURL` | `*string` | AlpineIQ redemption URL |
| `StickyTemplateID` | `*string` | Sticky CRM template used |

### Timestamps & Audit

| Field | Go Type | Description |
|-------|---------|-------------|
| `CreatedAt` | `*scalar.Timestamp` | Order creation time (Unix seconds) |
| `UpdatedAt` | `*scalar.Timestamp` | Last update time (Unix seconds) |
| `InvoiceDate` | `*scalar.Timestamp` | Invoice generation time |
| `ActivityLogs` | `[]*ActivityLog` | Append-only history (see section 8) |
| `ClientDetail` | `*ClientDetail` | Device, User-Agent, and IP of the request |

---

## 2. OrderStatusType Enum

**Source:** `pkg/graphql/modules/order/schema.graphqls` lines 1-14

12 possible values:

| Status | Description |
|--------|-------------|
| `ABANDONED_CART` | Cart was created but never submitted — timed out or customer left |
| `PENDING` | Order submitted and awaiting processing. Set by order-worker after invoice number assignment |
| `PROCESSING` | Order is being prepared/fulfilled by the dispensary |
| `APPROVED` | Order approved for processing (used by Treez auto-approval flow) |
| `ASSIGNED` | Order assigned to a delivery driver or fulfillment staff |
| `READY` | Order is ready for customer pickup or driver dispatch |
| `DELIVERY_STARTED` | Delivery driver en route to customer |
| `DELIVERY_FINISHED` | Delivery route completed — awaiting final handoff |
| `DELIVERED` | Order successfully delivered or picked up by customer |
| `DECLINED` | Order rejected by POS, payment failure, or admin action |
| `VOID` | Order voided/cancelled (by customer via `customerCancelOrder` or admin) |
| `REFUNDED` | Order has been fully refunded |

**Transition constraints:** The library defines no formal state machine — transitions are enforced by consuming services (e-com-srv resolvers, api-srv webhooks, worker processors). The `UpdateOrderStatusInput` accepts any valid status value.

---

## 3. Order Service Interface

**Source:** `pkg/services/order/types.go`

```go
type Order interface {
    Upsert(ctx context.Context, tableName string, o model.Order) (*model.Order, error)
    GetOrderByID(ctx context.Context, tableName, id string) (*model.Order, error)
    All(ctx context.Context, tableName string, filter *Filter) ([]model.Order, *string, error)
    GetOrdersByEmail(ctx context.Context, tableName, email string, filter *Filter) ([]model.Order, *string, error)
    Search(ctx context.Context, tableName string, query string, pagination pagination.Pagination) (*model.OrderConnection, error)
}
```

### Filter struct

```go
type Filter struct {
    TotalPrice *int
    Status     []model.OrderStatusType
    Cursor     *string
    Customer   *model.CustomerOrderFilter  // { FirstName, LastName }
}
```

### Method Details

#### `Upsert(ctx, tableName, order) → (*Order, error)`
- DynamoDB PutItem — creates or fully replaces the order document
- No partial update — the entire order model is written
- Used by both cart mutations (update) and status transitions (replace full doc)

#### `GetOrderByID(ctx, tableName, id) → (*Order, error)`
- DynamoDB GetItem with composite key: `entity_id = id`, `key = "d#"`
- Returns `ErrNotFound` if no item matches
- The `"d#"` key is hardcoded — all orders use this sort key

#### `All(ctx, tableName, filter) → ([]Order, *cursor, error)`
- DynamoDB Scan with filter expression `key = "d#"`
- Additional optional filters:
  - `Status` — multi-value IN filter (e.g., `status IN (:0, :1, :2)`)
  - `Customer.FirstName` / `Customer.LastName` — case-sensitive OR lowercase OR uppercase match against `delivery_address.first_name` / `delivery_address.last_name`
- Cursor-paginated via DynamoDB `ExclusiveStartKey`

#### `GetOrdersByEmail(ctx, tableName, email, filter) → ([]Order, *cursor, error)`
- DynamoDB Query on `email-index` GSI
- Key condition: `email = :email`
- Always filters for `key = "d#"`
- Optional `TotalPrice` filter with `>` operator
- Cursor pagination (key includes: `email`, `entity_id`, `created_at`, `key`)

#### `Search(ctx, tableName, query, pagination) → (*OrderConnection, error)`
- Uses `govaluate` expression engine to parse the `query` string
- Requires either `status` or `email` condition (enforced; error if neither)
- Queryable fields and their evaluation:
  - `id` → `EntityID`
  - `email` → `Email`
  - `status` → `Status` (string)
  - `order_number` → `OrderNumber` (int, 0 if nil)
  - `total_price` → `TotalPrice` (int, 0 if nil)
  - `customer_first_name` → `DeliveryAddress.FirstName`
  - `customer_last_name` → `DeliveryAddress.LastName`
  - `created_at` → `CreatedAt` (int64 Unix timestamp)
- Supports comparison operators: `>`, `>=`, `<`, `<=`, `==`
- Returns `OrderConnection { TotalCount, Edges []*OrderEdge, PageInfo }`
- Supports forward (`First` + `After`) and backward (`Last` + `Before`) pagination
- Cursor format: base64-encoded `entity_id|status|key|created_at` (status queries) or `entity_id|email|key|created_at` (email queries)

### Cursor Encoding (pkg/services/order/cursor.go)

```go
func EncodeCursor(values []string) (string, error)  // joins with "|", base64 encodes
func DecodeCursor(cursor string) (string, error)     // base64 decodes
```

### Error

```go
var ErrNotFound = errors.New("order not found")
```

---

## 4. LineItem Model

**Source:** `pkg/model/model_gen.go` lines 1595-1622, `pkg/graphql/modules/order/schema.graphqls` lines 127-154

All prices are `*int` in **cents**. All fields are pointers (nullable).

| Field | Go Type | Description |
|-------|---------|-------------|
| `ItemID` | `*string` | Product/inventory item ID |
| `ID` | `*string` | Line item unique ID |
| `Name` | `*string` | Product display name |
| `Quantity` | `*int` | Number of units ordered |
| `Price` | `*int` | Base unit price (cents) |
| `PriceType` | `*PriceType` | `FLAT` (fixed price) or `TIER` (weight/quantity-based) |
| `SalePrice` | `*int` | Discounted unit price (cents) — the "effective" price |
| `Weight` | `*float64` | Item weight |
| `Length` | `*float64` | Item length |
| `Height` | `*float64` | Item height |
| `Width` | `*float64` | Item width |
| `ProductWidth` | `*float64` | Product packaging width |
| `ProductHeight` | `*float64` | Product packaging height |
| `WeightUnit` | `*string` | Weight UOM (e.g., "g", "oz") |
| `LengthUnit` | `*DimensionUnitType` | `FT`, `IN`, `CM`, `MM`, `M` |
| `WidthUnit` | `*DimensionUnitType` | Same as above |
| `HeightUnit` | `*DimensionUnitType` | Same as above |
| `Thumbnail` | `*string` | Product image URL |
| `Variants` | `[]*Variant` | Selected product variant(s) — embedded, not referenced |
| `InStock` | `*bool` | Stock availability at time of cart creation |
| `Barcodes` | `[]*string` | Physical product barcodes (assigned post-submission for barcode scanning) |
| `Brand` | `*string` | Product brand name |
| `Category` | `*string` | Product category name |
| `PosID` | `*string` | POS system product identifier |
| `TierMethod` | `*TierMethod` | `WEIGHT_GRAMS`, `BULK_WEIGHT`, or `UNIT` |
| `BaseWeight` | `*float64` | Base weight for tier price calculations |

### Product/Variant Relationship

- `LineItem.ItemID` links to the product catalog
- `LineItem.Variants` embeds the selected variant(s) directly (denormalized, not a reference)
- Each `Variant` has independent `Price`, `SalePrice`, `DiscountPercent`, `DiscountAmount` fields (all `*int` in cents)
- `Variant.Type` is a string like `"gram"`, `"eighth_ounce"`, `"ounce"` — used for weight-to-quantity mapping
- `Variant.PosID` maps to the POS system's product variant ID

### Price Representation

- **All monetary amounts are `*int` in cents** (e.g., $12.50 = 1250)
- `Price` is the base/list price
- `SalePrice` is the effective/discounted price — this is what's used in total calculations
- The Treez mapping function divides by 100 to convert cents → dollars for API calls (`float64(*lineItem.SalePrice) / 100`)
- `ErrorResponseData` uses `*float64` (dollars, not cents) — this is an inconsistency

### Tier Pricing Logic

When `PriceType == TIER`:
- `TierMethod` determines how quantity is calculated:
  - `WEIGHT_GRAMS`: quantity = `(resolveQuantity(variants) / baseWeight) * quantity`
  - `BULK_WEIGHT` / `UNIT`: quantity = `resolveQuantity(variants) * quantity`
- `resolveQuantity()` sums mapped weights across all variants using `MapQuantity()`

---

## 5. Promotion Model & Evaluation

**Source:** `pkg/model/model_gen.go`, `pkg/graphql/modules/promotion/schema.graphqls`, `pkg/services/promotion/promotion.go`

### Promotion Struct (33 fields)

| Field | Go Type | Description |
|-------|---------|-------------|
| `ID` | `*string` | Unique promotion ID |
| `Name` | `*string` | Internal name |
| `Label` | `*string` | Display label |
| `Status` | `*bool` | Active/inactive toggle |
| `NeverExpire` | `*bool` | If true, ignores date range |
| `StartDate` | `*scalar.Timestamp` | Start of validity window (Unix seconds) |
| `ExpirationDate` | `*scalar.Timestamp` | End of validity window |
| `Priority` | `*int` | Evaluation priority (higher = applied first) |
| `DiscountType` | `*DiscountType` | What the discount applies to |
| `DiscountRateType` | `*DiscountRateType` | How the rate is interpreted |
| `Rate` | `*int` | Discount value — cents if AMOUNT, whole number if PERCENTAGE |
| `DiscountApplyType` | `*DiscountApplyType` | How it's triggered |
| `PromoCode` | `*string` | Code string (if `PROMOTION_CODE` apply type) |
| `PromoCodeMaxNumberOfUses` | `*int` | Max total uses of this promo code |
| `EnabledMonday`..`EnabledSunday` | `*bool` | Day-of-week availability (7 fields) |
| `MinimumOrderRequired` | `*bool` | Enforce minimum order amount |
| `MinimumOrderAmount` | `*int` | Minimum order total (cents) |
| `MaximumOrderRequired` | `*bool` | Enforce maximum order amount |
| `MaximumOrderAmount` | `*int` | Maximum order total (cents) |
| `ProductsID` | `[]*string` | Product IDs this promotion targets (product discount only) |
| `FirstTimePatient` | `*bool` | Only applies to first-time customers |
| `BlazeID` | `*string` | External Blaze promotion ID |
| `Broadcast` | `*bool` | Show on storefront |
| `BroadcastType` | `*BroadcastType` | `HEADER_BAR_HOME_PAGE`, `HEADER_BAR_ALL_PAGE`, `EMAIL_POPUP` |
| `BroadcastColor` | `*string` | Display color |
| `Photo` | `*Asset` | Promotional image |

### DiscountType Enum

| Value | Meaning |
|-------|---------|
| `AMOUNT_OFF_ORDER` | Fixed dollar amount off entire order |
| `PRODUCT_DISCOUNT` | Discount applied to specific products |
| `FREE_DELIVERY_DISCOUNT` | Waived delivery fee |
| `REWARDS` | Loyalty/rewards points redemption |

### DiscountRateType Enum

| Value | Meaning |
|-------|---------|
| `AMOUNT` | Fixed dollar amount (in cents) |
| `PERCENTAGE` | Percentage (e.g., 15 = 15%) |

### DiscountApplyType Enum

| Value | Meaning |
|-------|---------|
| `AUTOMATICALLY` | Applied to qualifying orders without user input |
| `PROMOTION_CODE` | Customer must enter `PromoCode` to activate |

### Promotion Storage

Promotions are stored in **S3 as JSON files** (not DynamoDB). The service reads/writes the entire promotion list as a JSON array per store.

### Promotion Service Interface (`pkg/services/promotion/promotion.go`)

```go
Create(ctx, promotion, key) → (*Promotion, error)
CreateMultiple(ctx, []Promotion, key) → error
Update(ctx, promotion, key) → (*Promotion, error)
GetAll(ctx, key) → ([]Promotion, error)
GetActives(ctx, key) → ([]Promotion, error)
Get(ctx, id, key) → (*Promotion, error)
GetActive(ctx, id, key) → (*Promotion, error)
Delete(ctx, id, key) → error
```

### GetActives Filtering Logic

Active promotions must satisfy:
1. `Status == true`
2. Either `NeverExpire == true` OR (`StartDate <= now` AND `ExpirationDate >= now`)

Day-of-week filtering and order amount validation happen in the consuming service (e-com-srv), not in this library.

### Automatic vs. Coupon Promotions

| Aspect | Automatic | Coupon Code |
|--------|-----------|-------------|
| `DiscountApplyType` | `AUTOMATICALLY` | `PROMOTION_CODE` |
| Activation | Applied during cart evaluation if conditions match | Customer enters code; `applyPromoCode` mutation validates |
| `PromoCode` field | null | Required string |
| Use limit | None (unlimited) | `PromoCodeMaxNumberOfUses` caps total uses |
| Condition evaluation | Same: date range, day-of-week, order amount, product targeting | Same |

### How Promotions Are Applied to Orders

When applied, a promotion creates an `OrderDiscountItem` entry on the order:

```go
type OrderDiscountItem struct {
    ID                *string
    DiscountType      *DiscountType
    DiscountRateType  *DiscountRateType
    DiscountApplyType *DiscountApplyType
    BlazeID           *string
    PromoCode         *string
    Rate              *int
}
```

The actual discount calculation (subtotal modification, line-item allocation) happens in **e-com-srv**, not in this library. The `applyPromoCode` GraphQL mutation is defined here but implemented there.

### govaluate Usage

**Clarification:** `govaluate` is used in the **Order Search** service method for parsing query expressions — **not** for promotion condition evaluation. Promotion conditions are declarative (static field checks).

### Jane Specials

A separate `JaneSpecial` type exists for Jane POS-native promotions with their own richer structure: bundle products, brand/kind targeting, dependent/independent bundle settings, max applications per cart.

---

## 6. Cart vs Order Distinction (the "d#" Key)

### How It Works

All orders in the DynamoDB table use a **composite primary key**:
- **Partition key:** `entity_id` (UUID)
- **Sort key:** `key`

The value `"d#"` is used as the sort key for **all orders** accessed through the Order service in this library. Every service method hardcodes this:

- `GetOrderByID` — queries `entity_id = id, key = "d#"`
- `GetOrdersByEmail` — filters `key = "d#"`
- `All` — filters `key = "d#"`
- `Search` — filters `key = "d#"` (as `:keyDoc`)

### Cart vs. Submitted Order

The distinction between a cart (draft) and a submitted order is made by the **Status field**, not the key:

| State | `Key` | `Status` | `OrderNumber` |
|-------|-------|----------|----------------|
| Empty cart | `"d#"` | null or no status set | null |
| Cart with items | `"d#"` | null | null |
| Submitted | `"d#"` | `PENDING` (set by order-worker) | Assigned by invoice counter |
| Processing | `"d#"` | `PROCESSING` | Present |
| Completed | `"d#"` | `DELIVERED` | Present |

### updateCartInput

The cart mutation wraps `OrderInput` — it writes the full order model each time:

```graphql
input updateCartInput {
    order: OrderInput!
}
```

### POS-Specific Cart Mutations

Different POS integrations have their own cart mutation paths:
- `updateCart` — generic (non-POS) cart update → returns `Order`
- `updateJaneCart` — Jane POS cart → returns `Order`
- `updateTreezCart` — Treez POS cart → returns `UpdateTreezCartResponse` (order + specials + error)
- `updateDutchieCart` — Dutchie POS cart → returns `UpdateDutchieCartResponse` (status + order + error)

The Treez and Dutchie variants return richer responses with error details and (for Treez) applicable specials.

---

## 7. Payment-Related Fields

### PaymentDetail Struct

```go
type PaymentDetail struct {
    Type             *PaymentType
    CardType         *string                  // "VISA", "MASTERCARD", etc.
    LastFour         *string                  // Last 4 digits
    Refunded         *bool
    RefundID         *string
    AeropayDetail    *AeroPayPaymentDetail
    SwifterDetail    *SwifterPaymentDetail
    StrongholdDetail *StrongholdPaymentDetail
    TreezpayDetail   *TreezPayPaymentDetail
}
```

### PaymentType Enum (8 values)

| Value | Description |
|-------|-------------|
| `CASH` | Cash payment |
| `DEBIT_CARD` | Debit card |
| `CREDIT_CARD` | Credit card |
| `CHECK` | Check payment |
| `AEROPAY` | AeroPay ACH-based payment |
| `SWIFTER` | Swifter payment processor |
| `STRONGHOLD` | Stronghold payment processor |
| `TREEZPAY` | TreezPay (Treez-native payments) |

### Provider-Specific Payment Details

#### AeroPayPaymentDetail
| Field | Type | Description |
|-------|------|-------------|
| `TransactionID` | `string` | AeroPay transaction ID |
| `Amount` | `int` | Transaction amount (cents) |
| `MerchantName` | `string` | Merchant name |
| `CustomerName` | `string` | Customer name |
| `UUID` | `*string` | Unique reference |

#### SwifterPaymentDetail
| Field | Type | Description |
|-------|------|-------------|
| `SessionID` | `string` | Swifter session ID |
| `OrderID` | `*string` | Associated order |
| `TrackNumber` | `*string` | Tracking number |
| `Status` | `*SwifterStatus` | `STARTED` or `COMPLETED` |

#### StrongholdPaymentDetail
| Field | Type | Description |
|-------|------|-------------|
| `PaylinkID` | `string` | Payment link ID |
| `OrderID` | `*string` | Associated order |
| `ChargeID` | `*string` | Charge transaction ID |
| `Status` | `*StrongholdPayLinkStatus` | `CREATED`, `USED`, or `CANCELED` |

#### TreezPayPaymentDetail
| Field | Type | Description |
|-------|------|-------------|
| `ProcessorName` | `string` | Processor name |
| `PaymentMethod` | `*string` | Method used |
| `PaymentID` | `*string` | Payment transaction ID |
| `InvoiceID` | `*string` | Invoice reference |

### TreezPayTicket (per-store tickets)

```go
type TreezPayTicket struct {
    TicketID        string
    ProviderStoreID string
    CreatedAt       *scalar.Timestamp
}
```

Order field: `TreezPayTicketIdsByStore []*TreezPayTicket` — maps ticket IDs to store locations for multi-store TreezPay scenarios.

### Payment-Related Mutations

| Mutation | Description |
|----------|-------------|
| `addAeropayPaymentDetailToOrder` | Attaches AeroPay transaction to an existing order |
| `createSwifterSession` | Initiates a Swifter payment session → returns `PaymentDetail` |
| `submitSwifterPayment` | Finalizes Swifter payment → returns `PaymentDetail` |
| `createStrongholdPayLink` | Creates Stronghold payment link → returns URL |
| `updateStrongholdChargeId` | Records Stronghold charge completion |
| `getTreezPayToken` | Retrieves TreezPay auth token for a specific order + store |

### StrongholdPayType Enum

| Value | Description |
|-------|-------------|
| `CHECKOUT` | Standard checkout payment |
| `BANK_LINK` | Bank account linking flow |
| `TIPPING` | Post-checkout tip payment |

---

## 8. Activity Log

**Source:** `pkg/graphql/modules/order/schema.graphqls` line 435-440

```go
type ActivityLog struct {
    Action    *string           // Description (e.g., "Order submitted", "Status changed to PROCESSING")
    User      *string           // Actor — admin email, system identifier, or customer ID
    Status    OrderStatusType   // Resulting status after this action (required — non-nullable in GraphQL)
    Timestamp *scalar.Timestamp // When the action occurred (Unix seconds)
}
```

### Characteristics

- **Append-only** — the `ActivityLogs []*ActivityLog` array on `Order` is only ever appended to, never modified
- **No dedicated service method** — activity logs are embedded in the Order model and written with `Upsert`
- **Status is required** — every log entry records what status the order transitioned to
- **Actor tracking** — `User` field identifies who caused the transition (admin, webhook system, customer)
- Written by consuming services (e-com-srv resolvers, api-srv webhooks, worker processors), not by this library directly

---

## 9. SQS Event Schemas

**Source:** `pkg/worker/event.go`

### Event Type Constants

#### Order Events

| Constant | Value | Published When |
|----------|-------|----------------|
| `OrderCompletedEventType` | `"order/completed"` | Cart submitted successfully (standard checkout) |
| `OrderKioskCompletedEventType` | `"order/kiosk_completed"` | Cart submitted via kiosk flow |
| `OrderConfirmNotifyEventType` | `"order/confirm_notify"` | Order needs confirmation email sent |
| `OrderOutOfStockNotifyEventType` | `"order/out_of_stock_notify"` | Out-of-stock notification needed |
| `OrderDeliveryNotifyEventType` | `"order/delivered_notify"` | Delivery complete notification needed |
| `OrderStatusChangedNotifyEventType` | `"order/status_changed_notify"` | Status changed, notification needed |

#### App Integration Events

| Constant | Value | Published When |
|----------|-------|----------------|
| `SubscribeKlaviyoEventType` | `"app/klaviyo"` | Customer subscribes to Klaviyo |
| `SubscribeOmnisendEventType` | `"app/omnisend"` | Customer subscribes to Omnisend |
| `SubscribeAlpineIQEventType` | `"app/alpineiq"` | Customer subscribes to AlpineIQ |
| `SubscribeLisTrackEventType` | `"app/listrack"` | Customer subscribes to ListTrack |
| `PlaceOrderKlaviyoEventType` | `"app/klaviyo_place_order"` | Order placed, send to Klaviyo |
| `ProcessBlazeEventType` | `"app/blaze"` | Place order in Blaze POS |
| `ProcessJaneEventType` | `"app/jane"` | Place order in Jane POS |
| `ProcessTreezEventType` | `"app/treez"` | Place order in Treez ERP |
| `CreateStoreEventType` | `"app/create_store"` | New store created |

#### Other Events

| Constant | Value | Published When |
|----------|-------|----------------|
| `ContactBusinessEventType` | `"contact/business_owner"` | Contact form submission to business |

### Message Types

```go
type Message string       // Event type identifier
type QueueEvent struct {
    Event       string    // Raw SNS message body (JSON string containing the order/payload)
    MessageType Message   // Parsed event type constant
}
```

### SQS Message Structure

Messages arrive as SNS-over-SQS (SNS publishes → SQS subscribes):

```go
type SQSMessage struct {
    Type              string                         // "Notification"
    MessageID         uuid.UUID                      // SNS message ID
    TopicARN          string                         // Source SNS topic
    Message           string                         // JSON payload (the actual event data)
    Timestamp         string                         // ISO 8601 timestamp
    SignatureVersion  string
    Signature         string
    SigningCertURL    *url.URL
    UnsubscribeURL    *url.URL
    MessageAttributes map[string]MessageAttribute     // { Type, Value } pairs
}
```

The `Message` field contains the JSON-serialized event payload. The `MessageAttribute` with key `type` (or similar) carries the event type constant for routing.

### Worker Processing Pipeline

```
SQS Lambda Event
  → Parser.Parse(SQSMessage) → *QueueEvent
    → HandlerManager dispatches based on QueueEvent.MessageType
      → WhenBuilder predicate matches → Mapper.Map(event.Event) → domain object
        → Processor[0].Process(ctx, domainObj)
        → Processor[1].Process(ctx, domainObj)
        → ...
      → Deleter.Delete(ctx, receiptHandle)
```

**Error behavior:** If any processor returns an error, the message ID is added to `batchItemFailures` and the SQS message is NOT deleted (will be retried or moved to DLQ). If a processor succeeds but the deleter fails, same behavior.

### Manager Services Available to Workers

The `Manager` struct provides these service dependencies to all processors:

| Service | Type | Purpose |
|---------|------|---------|
| `Order` | `order.Order` | Read/write orders |
| `Store` | `store.Store` | Read store config |
| `Invoice` | `invoice.IInvoice` | Invoice number generation |
| `Account` | `accountsrv.Account` | Customer account management |
| `File` | `fileupload.FileUpload` | S3 file access |
| `App` | `app.IApp` | Integration config |
| `SNS` | `pkgAws.SNS` | Publish events |
| `SQS` | `pkgAws.SQS` | Queue operations |
| `SES` | `pkgAws.SES` | Send emails |
| `DynamoDB` | `pkgAws.DynamoDB` | Direct DB access |
| `Instrumentation` | `instrumentation.Instrumentation` | New Relic events |
| `LisTrack` | `listrack.LisTrack` | ListTrack CRM |
| `Notifiation` | `notification_setting.NotificationSetting` | Notification preferences |
| `Logger` | `glog.Logger` | Structured logging |
| `NewRelicSDK` | `*newrelic.Application` | APM transactions |

---

## 10. Helper Functions That Manipulate Order State

**Source:** `pkg/erp/treez/map.go`

These functions live in the Treez ERP package — they transform internal order models for Treez API submission.

### `MapOrderToTreezTicket(order, customerID, status, timezone, mapItemsDiscounts, mapPromo) → *Ticket`

Primary order-to-Treez transformation. Steps:
1. Set source (`"Treez Ecommerce 2.0"`) and ticket type (PickUp default)
2. Handle curbside pickup notes — override `Notes` with curbside message
3. Set `TicketPatientType` to "medical" if `OrderType == MEDICAL`
4. Extract promo code from `DiscountItems` if `mapPromo == true`
5. Set custom delivery fees if `AllowCustomDeliveryFee == true` (converts cents → dollars)
6. Calculate rewards total from discount items
7. Determine reward type from reward ID prefix (`alpine_`, `sticky_`, `reward_`)
8. Build line items:
   - Without barcodes: calculate quantity using tier method, set price, reward dollars
   - With barcodes: one item per barcode, quantity from tier, allocate reward dollars
9. Apply Jane-specific discounts if `mapItemsDiscounts == true`
10. Set delivery address and schedule dates if delivery order

### `getRewardsTotalDiscount(order) → int`

Sums all REWARDS-type discounts:
- For `AMOUNT` rate type: uses `Rate` directly (cents)
- For `PERCENTAGE` rate type: calculates `TotalPrice * Rate / 100`

### `mapTreezItemRewardsDollars(subtotal, rewardsTotal, lineItemPrice, qty, rewardsID) → *float64`

Allocates Treez-native reward dollars to a line item:
- Only runs if reward ID starts with `"reward_"`
- Formula: `(lineItemPrice * qty / subtotal) * rewardsTotal`, then `math.Round() / 100` → dollars

### `mapAlpineItemRewardsDollars(subtotal, rewardsTotal, lineItemPrice, qty, rewardsID) → []POSDiscount`

Same allocation for AlpineIQ rewards:
- Only runs if reward ID starts with `"alpine_"`
- Returns `POSDiscount` with `DiscountTitle` (extracted discount ID) and `DiscountAmount`

### `mapTreezItemDiscounts(order, items)`

Distributes Jane cart discounts across Treez line items:
- Converts `TotalDiscounts` from cents to dollars
- Iterates items, allocating up to `price - 0.1` per item (prevents penny discounts)
- Labels each as "E-commerce Discount" with `DiscountMethodDollar`
- Stops when total discount is fully allocated

### `resolveQuantity(variants) → float64`

Sums `MapQuantity()` across all variants in a line item.

### `MapQuantity(variantType) → float64`

Converts cannabis weight type strings to numeric gram values:

| Input | Output |
|-------|--------|
| `"half_gram"` | 0.5 |
| `"gram"` | 1.0 |
| `"two_gram"` | 2.0 |
| `"eighth_ounce"` | 3.5 |
| `"quarter_ounce"` | 7.0 |
| `"half_ounce"` | 14.0 |
| `"ounce"` | 28.0 |
| `"N_anything"` | N (parses leading integer) |

### `getPromoCode(order) → string`

Finds the first discount item with `DiscountApplyType == PROMOTION_CODE` and returns its `PromoCode`.

### `getDiscountIDFromRewardsID(rewardID) → string`

Parses reward ID format `{type}_{page_title}_{pos_discount_id}` and returns the third component (the POS discount ID).

### `getStickyDiscountIDFromRewardsID(rewardID) → string`

Same as above but semantically named for Sticky rewards (same format: `sticky_{title}_{id}`).

### `resolveDate(timestamp, timezone) → *string`

Converts Unix timestamp to ISO 8601 string with timezone: `"2006-01-02T15:04:05.000-07:00"`.

---

## Appendix: All Order-Related Enums Summary

| Enum | Values |
|------|--------|
| `OrderStatusType` | ABANDONED_CART, PENDING, PROCESSING, DELIVERY_STARTED, DELIVERY_FINISHED, DELIVERED, DECLINED, VOID, REFUNDED, READY, APPROVED, ASSIGNED |
| `PaymentType` | CASH, DEBIT_CARD, CREDIT_CARD, CHECK, AEROPAY, SWIFTER, STRONGHOLD, TREEZPAY |
| `DiscountType` | AMOUNT_OFF_ORDER, PRODUCT_DISCOUNT, FREE_DELIVERY_DISCOUNT, REWARDS |
| `DiscountRateType` | AMOUNT, PERCENTAGE |
| `DiscountApplyType` | AUTOMATICALLY, PROMOTION_CODE |
| `PriceType` | FLAT, TIER |
| `TierMethod` | WEIGHT_GRAMS, BULK_WEIGHT, UNIT |
| `DeliveryStatus` | COMPLETED, FAILED, STARTED, ARRIVING, DELAYED, SUBMITED |
| `SwifterStatus` | STARTED, COMPLETED |
| `StrongholdPayLinkStatus` | CREATED, USED, CANCELED |
| `StrongholdPayType` | CHECKOUT, BANK_LINK, TIPPING |
| `AccountType` | ADULT, MEDICAL |
| `DimensionUnitType` | FT, IN, CM, MM, M |
| `BroadcastType` | HEADER_BAR_HOME_PAGE, HEADER_BAR_ALL_PAGE, EMAIL_POPUP |
| `ErrorResponseCodeType` | VALIDATION_ERROR, PARTIAL_PAYMENT_NOT_SUPPORTED, CAN_NOT_UPDATE_PAID_TICKET, LOCATION_NAME_DOES_NOT_MATCH, INSUFFICIENT_SELLABLE_QUANTITY, RESPONSE_LIMIT_EXCEEDED, INVALID_COUPON, INSUFFICIENT_REWARDS_POINTS, INVALID_TOKEN, INVALID_REQUEST, CUSTOMER_NOT_FOUND, INACTIVE_CUSTOMER, DELIVERY_MINIMUM_ERROR, UNKNOWN |

## Appendix: All Order-Related GraphQL Operations

### Queries

| Operation | Auth Required | Returns |
|-----------|---------------|---------|
| `listStoreOrders` | SUPER_ADMIN, ADMIN | `OrderConnection` |
| `customerOrderHistory` | SUPER_ADMIN, ADMIN, CUSTOMER | `OrderCursorPagination` |
| `getOrder` | SUPER_ADMIN, ADMIN, CUSTOMER, SALLER | `Order` |
| `getOrderDeliveryETA` | SUPER_ADMIN, ADMIN | `Int` |
| `getTreezPayToken` | (none) | `String` |

### Mutations

| Operation | Auth Required | Returns | Description |
|-----------|---------------|---------|-------------|
| `updateCart` | (none) | `Order` | Generic cart update |
| `updateJaneCart` | (none) | `Order` | Jane POS cart update |
| `updateTreezCart` | (none) | `UpdateTreezCartResponse` | Treez POS cart update (includes specials + error) |
| `updateDutchieCart` | (none) | `UpdateDutchieCartResponse` | Dutchie POS cart update (includes error) |
| `submitCart` | (none) | `String` (order ID) | Submit cart for processing |
| `submitKioskCheckout` | (none) | `String` (order ID) | Kiosk-specific submission |
| `createOrder` | (none) | `Order` | Direct order creation |
| `updateOrder` | SUPER_ADMIN, ADMIN, SALLER | `Order` | Update existing order |
| `updateOrderStatus` | SUPER_ADMIN, ADMIN, SALLER | `Order` | Change order status |
| `updateOrderDetails` | SUPER_ADMIN, ADMIN, SALLER | `Order` | Update address, payment, items, time window, notes |
| `updateOrderLineItemsBarCodes` | SUPER_ADMIN, ADMIN, SALLER | `Order` | Assign barcodes to line items |
| `checkOrderLineItemsBarCodes` | SUPER_ADMIN, ADMIN, SALLER | `Boolean` | Validate barcode assignments |
| `addAeropayPaymentDetailToOrder` | (none) | `Order` | Attach AeroPay transaction |
| `createSwifterSession` | (none) | `PaymentDetail` | Start Swifter payment |
| `submitSwifterPayment` | (none) | `PaymentDetail` | Complete Swifter payment |
| `customerCancelOrder` | CUSTOMER | `Order` | Customer self-service cancel |
| `createStrongholdPayLink` | CUSTOMER | `CreateStrongholdPayLinkResponse` | Create Stronghold payment link |
| `updateStrongholdChargeId` | CUSTOMER | `String` | Record Stronghold charge |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-lib (commit b4cbde5)*
