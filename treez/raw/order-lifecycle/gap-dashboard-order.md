# gap-dashboard — Admin Order Management

> How the admin dashboard exposes order management, status transitions, delivery scheduling, search, and fulfillment capabilities to store operators.

---

## 1. Order List View

### Views

The order list page (`/orders`) supports two views, toggled via a switch persisted in localStorage:

- **Kanban board** (`src/pages/order/kanban/Kanban.tsx`) — orders grouped by status lane
- **Table view** (`src/pages/order/OrderTableList.tsx`) — paginated table with sortable columns

A separate legacy table view exists for abandoned carts at `/orders_abandoned` (`src/pages/order/OrdersList.jsx`).

### Kanban Lanes

Defined in `src/config/constant.js`:

| Lane | Color | Status(es) |
|------|-------|------------|
| Processing | metalic | `PENDING` |
| Approved | info | `APPROVED`, `READY` |
| Assigned | info | `ASSIGNED` |
| On Route | info | `DELIVERY_STARTED` |
| Completed | success | `DELIVERED`, `DELIVERY_FINISHED` |
| Cancelled | error | `DECLINED`, `REFUNDED`, `VOID` |

### Data Source

Orders are queried via **GraphQL with cursor-based pagination**:

```graphql
QUERY_LIST_STORE_ORDER(
  account_id, store_id, cursor, query, last, before
) → { totalCount, edges { cursor, node { ...OrderFragment } }, pageInfo }
```

The `query` argument is an OpenSearch-style query string built from the filter state. Example:

```
(order_number == '1234' || email == 'x@y.com') && created_at >= 1681344000 && (status == 'APPROVED' || status == 'READY')
```

Query construction happens in `src/pages/order/context/OrdersDataContext.tsx`, which holds centralized filter state and builds the query string from filter inputs.

### Table Columns

The table view (`src/pages/order/OrderTable.jsx`) displays:

- Order number
- Customer name / email
- Status (badge)
- Order type (pickup / delivery)
- Delivery time window
- Total price
- City
- Created at

---

## 2. Order Detail View

**Route:** `/order/detail/:id`
**Component:** `src/pages/order/OrderForm.tsx` — works in both full-page and modal mode (controlled by `isModal` prop)

**Data query:**

```graphql
QUERY_GET_ORDER(entity_id, account_id, store_id) → { ...OrderFragment }
```

The detail view has four tabs:

### Tab 1: Order Data (`OrderDataPane.tsx`)

**Displayed fields:**
- Order number, status, delivery type (pickup/delivery)
- Customer email
- Subtotal, tax, delivery fee, service fee, discounts, total price
- Line items with: item_id, name, quantity, price, variants, barcodes
- Discount items with: discountType, discount_rate_type, discount_apply_type, rate
- Delivery address: first_name, last_name, address1, address2, city, province, zip, country, phone
- Payment details: payment_method, card_type, last_four, refund_id
- Delivery time window: deliver_after, time_window
- Notes
- Promo code

**Editable fields (only when order status is `PENDING`):**
- Delivery address (all fields; address1, city, zip, phone required)
- Payment method: `CASH`, `AEROPAY`, `DEBIT_CARD`, `CREDIT_CARD`, `CHECK`
- Line items: add, remove, edit quantities
- Delivery time window: deliver_after + time_window (preset hourly windows 8am–11pm)
- Notes
- Promo code

**Edit mutation:**

```graphql
MUTATION_UPDATE_ORDER_DETAIL(account_id, store_id, input: {
  id, address: OrderAddressInput, line_items: [LineItemInput],
  notes, payment_method: PaymentType, promo_code, time_window
}) → { ...OrderFragment }
```

### Tab 2: Additional Data (`AdditionalDataPane.tsx`)

Read-only metadata:

| Section | Fields |
|---------|--------|
| Client Details | device, user_agent, IP (x_forwarded_for) |
| Jane Store Info | shop, order ID, store ID, address, phone |
| Treez Info | treez_order_id, treez_order_number |
| OnFleet Driver | worker_id, worker_name, worker_image, status, tracking_url, complete_after, complete_before |
| Order Type | Medical vs Adult (derived from order_medical_id) |
| Customer Verification | Driver's license verification status |

### Tab 3: Activity Logs (`ActivityLogPane.tsx`)

Audit trail — each entry shows:
- `action` — what happened
- `user` — who did it
- `status` — resulting status
- `timestamp` — when

### Tab 4: Raw

Full JSON view of the order object.

### Additional Actions

- **Print invoice** — PDF generation via pdfmake
- **Print packing slip**
- **Update barcodes** — `MUTATION_UPDATE_ORDER_LINE_ITEMS_BARCODES`
- **Check barcodes** — `MUTATION_CHECK_ORDER_LINE_ITEMS_BARCODES`
- **Add AeroPay payment** — `MUTATION_ADD_AEROPAY_PAYMENT_DETAIL`

---

## 3. Status Transitions

**Component:** `src/pages/order/OrderActionButtons.tsx`

**Mutation called:**

```graphql
MUTATION_UPDATE_ORDER_STATUS(account_id, store_id, input: {
  id, status: OrderStatusType
}) → { ...OrderFragment }
```

### Allowed transitions by current status

| Current Status | Can transition to |
|----------------|-------------------|
| `PENDING` | `APPROVED` |
| `APPROVED` | `PENDING`, `DELIVERED`, `DECLINED` |
| `READY` | `PENDING`, `DELIVERED`, `DECLINED` |
| `ASSIGNED` | `PENDING`, `DECLINED` |
| `DECLINED` | `PENDING` |
| `ABANDONED_CART` | _(no actions — read-only)_ |
| `DELIVERED` | _(terminal — no actions)_ |
| `DELIVERY_STARTED` | _(managed by OnFleet — no manual actions)_ |
| `DELIVERY_FINISHED` | _(terminal — no actions)_ |
| `VOID` | _(terminal — no actions)_ |
| `REFUNDED` | _(terminal — no actions)_ |

### All OrderStatusType values

From `src/types/backend.generated.ts`:

```
PENDING, APPROVED, READY, ASSIGNED, DELIVERED, VOID,
DECLINED, REFUNDED, DELIVERY_STARTED, DELIVERY_FINISHED,
ABANDONED_CART, PROCESSING
```

Note: `DELIVERY_STARTED`, `DELIVERY_FINISHED`, `ASSIGNED` are set by external systems (OnFleet webhooks via api-srv), not by admin actions. `VOID` and `REFUNDED` also have no direct admin trigger in the UI.

---

## 4. Delivery Management

### Delivery Schedule Configuration

**Components:**
- `src/pages/store/DeliverySchedules.jsx` — delivery zone CRUD
- `src/pages/store/PickupSchedules.jsx` — pickup schedule CRUD
- `src/pages/store/ScheduleForm.jsx` — shared form (used for both)

**Context:** `src/context/schedule/ScheduleProvider.tsx` manages form state

Admins configure **delivery zones** with:

| Field | Description |
|-------|-------------|
| `name` | Zone identifier |
| `stores_provider_id` | Maps to a store provider |
| `delivery_fee` | Standard delivery charge |
| `free_after` | Order amount for free delivery |
| `min` | Minimum order value for delivery |
| `geo_type` | `ZIP`, `RADIUS`, or `GEOFENCE` |
| `zips` | Comma-separated ZIP codes (when geo_type = ZIP) |
| `radius` | Delivery radius in miles (converted to meters: miles × 1609.34) |
| `polygon_geo_zone` | GeoJSON polygon (when geo_type = GEOFENCE; drawn on Google Map) |

**Express delivery** can be enabled per zone:
- `active` / `active_by_default` toggles
- `start` / `end` times
- `expressName` / `expressNote`
- `inventory_provider_ids` — inventory locations for express menu

**Working hours & time slots** per day of week:
- Start time, end time per day
- Time slots within each day: start, end, cutoff (default 30min before end), disable flag, disable-for-today flag, is-express flag
- Supports overnight shifts (e.g., 22:00–02:00)
- "Apply to every day" toggle for bulk configuration
- Auto-generation via `generateTimeSlots()` in `src/utils/timeSlot.js`

**Persistence:** All schedule config is saved via `MUTATION_UPDATE_STORE` as `delivery_schedules[]` and `pickup_schedules[]` arrays on the Store object.

**Validation:** `useScheduleValidation` hook enforces:
- Start/end required on each slot
- End after start (with overnight exception)
- Cutoff within slot bounds
- At least one working day per week

### OnFleet Integration

**Configuration UI:** `src/pages/store/OnFleetForm.jsx`

| Field | Description |
|-------|-------------|
| `app_id` | OnFleet API ID (required) |
| `user` | OnFleet user account (required) |
| `test_user` | Sandbox account (required) |
| `sandbox` | Test mode toggle |
| `auto_assign` | Enable automatic driver assignment |
| `auto_assign_mode` | `LOAD` (balance workload) or `DISTANCE` (nearest driver) |
| `merchants` | Multiple merchant location configs |

**Saved via:** `MUTATION_UPDATE_ONFLEET_APP`

### Driver Management

**Page:** `/app/logistics/onfleet-drivers` (`src/pages/logistics/OnfleetDrivers.jsx`)

**Query:**
```graphql
QUERY_LIST_DELIVERY_WORKERS_IN_LOCATION(
  account_id, store_id, location, radius
) → [{ id, name, on_duty, phone, location, is_responding, image }]
```

Displays: driver name, ID, phone, on-duty status. Filter by location proximity.

### Order-Level Delivery Info

**In the order detail view:**
- `delivery_details.delivery` (boolean) — is this a delivery order
- `delivery_details.pickup` (boolean) — is this a pickup order
- `delivery_details.deliver_after` / `deliver_before` — delivery window timestamps
- `delivery_details.time_window` — human-readable window (e.g., "8:00 AM - 9:00 AM")
- `delivery_details.started_at` / `finished_at` — actual delivery timestamps
- `delivery_details.curbside_pickup` + `curbside_pickup_note`
- `on_fleet.worker_id`, `worker_name`, `worker_image`, `status`, `tracking_url`

**ETA query:**
```graphql
QUERY_ORDER_DELIVERY_ETA(accountId, storeId, entityId)
```

Note: Driver assignment is display-only in the dashboard — assignment is handled by OnFleet's auto-assign or OnFleet's own UI.

---

## 5. Refund / Cancellation

### Cancellation (Decline)

Admins decline orders by transitioning status to `DECLINED` via `MUTATION_UPDATE_ORDER_STATUS`. Available from: `PENDING`, `APPROVED`, `READY`, `ASSIGNED`.

Declined orders can be re-opened by transitioning back to `PENDING`.

### Refund

**Mutation exists:**
```graphql
MUTATION_REFUND_PAYMENT(input: RefundPaymentInput) → { status, error, refund_id }
```

**Note:** The refund mutation is defined in `src/apollo/module-operations/order.js` but no UI button or form was found wired to it in the order detail components. The `refund_id` field is displayed in payment details (read-only), suggesting refunds are processed through another channel (backend, POS, or payment provider portal) and the result syncs back.

### Void

`VOID` status exists but has no direct admin trigger in the UI. Likely set by external systems.

---

## 6. Order Search

### Primary Search: OpenSearch via GraphQL

The main order query (`QUERY_LIST_STORE_ORDER`) accepts a `query` parameter — an OpenSearch query string built client-side from filters.

**Searchable fields** (from `OrdersDataContext.tsx`):
- `order_number` — exact match
- `email` — exact match
- `created_at` — range (unix timestamps)
- `status` — enum match (supports multiple via OR)
- `type` — pickup / delivery (filter infrastructure exists but is currently commented out)
- `total_price` — range (filter infrastructure exists but is currently commented out)

**Search UI** (`Kanban.tsx`):
- Text input with 300ms debounce
- Dropdown to select search field (currently: Email, Order Number)

### Filters

**Component:** `src/pages/order/components/OrdersFilters.tsx`

**Date filter** (`OrdersDateFilter.tsx`):
- All / Today / 24 Hours / Custom date range
- Stored as unix timestamps (`created_at_start`, `created_at_end`)

**Status filter** (`OrdersStatusFilter.tsx`):
- Button group, one status at a time (toggleable)
- Values: `PENDING`, `APPROVED`, `READY`, `ASSIGNED`, `DELIVERED`, `VOID`, `DECLINED`, `REFUNDED`, `DELIVERY_STARTED`, `DELIVERY_FINISHED`
- Default: `APPROVED` for today

**Commented-out filters:**
- Type filter (delivery vs pickup)
- Total price range filter
- Env var override: `REACT_APP_ALGOLIA_ORDERS_FILTER_SORT`

### Algolia (Legacy/Alternative)

Algolia types exist in `src/types/algolia.ts` (AlgoliaOrder type) and Algolia config env vars are present, suggesting Algolia was the original search provider. OpenSearch appears to have replaced it for orders, but the type definitions and some infrastructure remain.

---

## 7. All Order-Related Routes

From `src/config/routes/routes.js`:

| Route | Page | Description |
|-------|------|-------------|
| `/orders` | `Orders.tsx` | Main order list (Kanban/Table) |
| `/orders_abandoned` | `OrdersList.jsx` | Abandoned carts table |
| `/order/create` | — | Manual order creation |
| `/order/detail/:id` | `OrderForm.tsx` | Order detail/edit |
| `/order/fulfillment/:id` | — | Fulfillment workflow |
| `/app/logistics/onfleet-drivers` | `OnfleetDrivers.jsx` | Driver listing |

---

## 8. All Order-Related GraphQL Operations

From `src/apollo/module-operations/order.js`:

### Queries

| Name | Arguments | Returns |
|------|-----------|---------|
| `QUERY_GET_ORDER` | entity_id, account_id, store_id | Full Order fragment |
| `QUERY_LIST_STORE_ORDER` | account_id, store_id, cursor, query, last, before | Paginated edges with Order nodes |
| `QUERY_CUSTOMER_ORDER_HISTORY` | account_id, store_id, cursor, email | OrderReducedFragment (limited fields) |
| `QUERY_ORDER_DELIVERY_ETA` | accountId, storeId, entityId | ETA data |

### Mutations

| Name | Input | Returns |
|------|-------|---------|
| `MUTATION_UPDATE_ORDER` | OrderInput | Full Order fragment |
| `MUTATION_UPDATE_ORDER_STATUS` | { id, status: OrderStatusType } | Full Order fragment |
| `MUTATION_UPDATE_ORDER_DETAIL` | { id, address, line_items, notes, payment_method, promo_code, time_window } | Full Order fragment |
| `MUTATION_REFUND_PAYMENT` | RefundPaymentInput | { status, error, refund_id } |
| `MUTATION_UPDATE_ORDER_LINE_ITEMS_BARCODES` | { id, line_items: [{ id, bar_codes, weight }] } | Full Order fragment |
| `MUTATION_CHECK_ORDER_LINE_ITEMS_BARCODES` | UpdateOrderLineItemsBarCodesInput | Boolean |
| `MUTATION_ADD_AEROPAY_PAYMENT_DETAIL` | AddAeroPayPaymentDetailInput | Full Order fragment |

### From delivery.js

| Name | Arguments | Returns |
|------|-----------|---------|
| `QUERY_LIST_DELIVERY_HUBS` | account_id, store_id | [{ id, name, location, address }] |
| `QUERY_LIST_DELIVERY_WORKERS_IN_LOCATION` | account_id, store_id, location, radius | [{ id, name, on_duty, phone, ... }] |

---

## 9. Order Utility Functions

From `src/utils/order.ts`:

| Function | Signature | Purpose |
|----------|-----------|---------|
| `mapProductToLineItem` | `(product: Product) → LineItem` | Converts a product to a line item with variant pricing |
| `mapProductsToLineItems` | `(products: Product[]) → LineItem[]` | Batch conversion |
| `getVariantPrice` | `(salePrice, price) → number` | Returns the lower of sale or regular price |

---

## 10. Notable Limitations

1. **Refund UI missing** — mutation exists but no form/button in the order detail view. Refunds appear to be processed externally.
2. **Driver assignment is display-only** — admins cannot assign drivers from the dashboard; OnFleet handles this.
3. **No manual status transition to VOID or REFUNDED** — these terminal states are set by external systems.
4. **Type and price filters commented out** — infrastructure exists but is disabled.
5. **Search limited to email and order number** — no full-text search across customer name, address, etc.
6. **Editing restricted to PENDING status** — once approved, order details cannot be modified from the dashboard.

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/dashboard (commit 09f1c76)*
