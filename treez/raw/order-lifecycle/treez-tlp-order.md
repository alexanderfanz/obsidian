# treez-tlp — Customer-Facing Order & Checkout Flow

> Detailed documentation of how the treez-tlp Next.js storefront manages the cart, checkout, payment, and order lifecycle from the customer's perspective.

---

## 1. Cart Management

### How the Cart Is Created

The cart has no explicit creation step. It is **initialized client-side** when a user first adds an item. The backend assigns an `entity_id` on the first `updateTreezCart` mutation — until then, `entity_id` is an empty string.

Initial state is defined in `data/state/variables.ts` with `entity_id: ''`.

### Storage Strategy: Zustand + localStorage + Apollo Cache

Three storage layers work together:

**Zustand with persist middleware** (`data/state/createStore.ts`):
- `ecomStore` — persisted to localStorage under key `@gapcommerce/{STORE_ID}`
  - Uses `persist` middleware with version from `NEXT_PUBLIC_STORAGE_VERSION`
  - Custom merge function handles state migrations between versions
- `wishListStore` — persisted under `@gapcommerce/wl/{STORE_ID}` (favorite product IDs only)
- `uiStore` — non-persisted, in-memory only (tracks `is_cart_open`, `is_search_open`, etc.)

**Apollo Client cache** — Cart totals are cached after `updateTreezCart` mutations. The mutation returns an updated `order` object stored in both Zustand and Apollo cache.

**Cookies** — Age gate token, cart token, selected store, fulfillment type, customer type.

### Cart State Structure (EcomState)

Defined in `types/state.ts`:

```
entity_id: string                    — Cart/order ID assigned by backend
key: string                          — Additional cart key
line_items: CartLineItem[]           — Products in cart
delivery_details: OrderDelivery      — Pickup/delivery mode and time window
delivery_address: OrderAddress       — Delivery or pickup location
payment_details: PaymentDetail       — Payment type (Cash, Card, etc.)
discount_items: OrderDiscountItem[]  — Applied promo codes/discounts
email: string                        — Customer email
date_of_birth: string                — ISO string
customer_type: CustomerType          — MEDICAL or ADULT
medical_id: string                   — MMID
medical_id_expiration_date: string   — ISO format
state_of_residence: string
notes: string
terms_accepted: boolean

— Totals (populated by backend after updateTreezCart):
subtotal_price: number
total_price: number
total_tax: number
total_delivery: number
total_discounts: number
total_service_fee: number
total_line_items_price: number

— Post-submission IDs:
treez_order_id: string
treez_order_number: string
treez_pay_ticket_ids_by_store: TreezPayTicket[]

— Loyalty:
alpineiq_template_id: string
alpineiq_redemption_url: string
sticky_template_id: string

— Pending navigation state:
next_shop_mode: string
next_store_id: string
next_delivery_address: OrderAddress
next_customer_type: CustomerType
allow_custom_delivery_fee: boolean
```

### CartLineItem Structure

Extends the product model (`types/state.ts`):

```
_id: string                  — Unique local ID (product.id + random)
itemLocId: string            — Store location ID
weightVariant: Variant       — Selected weight/size variant
quantity: number             — Quantity ordered
bundleSpecial: JaneSpecial   — BOGO/bundle info if applicable
bundleProducts: GProduct[]   — Related bundle products
bundle_id?: string           — Bundle discount ID
bundle_slug?: string
bundle_title?: string
dependent?: boolean          — Part of GET items in BOGO
independent?: boolean        — Part of BUY items in BOGO
unitPricingTier?: UnitPricingTier
```

### Adding / Removing / Updating Items

**Adding** — `components/Counter/Counter.tsx` → `addItemToTheCart()`:
1. User clicks "Add to Cart" button
2. Creates a `CartLineItem` with unique `_id`, selected `weightVariant`, `quantity = 1` (or tier qty)
3. Calls `add(item)` from `useProducts()` hook
4. Hook does: `setState({ line_items: [...products, product] })`

**Removing** — `useProducts().remove(index)` splices by index, or `removeItem(item)` filters by ID.

**Quantity change** — User clicks +/- in `CounterActions`, calls `update()` with modified `quantity` on the full `line_items` array.

**Weight variant change** — `updateWeightVariant()` in Counter updates the `weightVariant` property and calls `useProducts().update()`.

All of these write to Zustand which auto-syncs to localStorage.

### Backend Sync

`components/CartNavigation/CartNavigation.tsx` orchestrates sync:

1. When the cart drawer opens (`is_cart_open` becomes true) **and** items exist, `useUpdateTreezCart` fires.
2. The mutation sends the full cart state to the backend:
   ```graphql
   mutation updateTreezCart($account_id, $store_id, $input: updateCartInput!) {
     updateTreezCart(account_id, store_id, input) {
       status
       specials { title, amount }
       order { entity_id, key, status, line_items, subtotal_price, total_price, total_tax, ... }
       error { code, message, data }
     }
   }
   ```
3. On completion, `setStoredState()` syncs `entity_id` and all totals from the backend response back into Zustand.
4. Cart auto-re-triggers when `stringifiedItems` changes (item add/remove/quantity change).

**Frontend → Backend line item mapping** (`utils/utils.ts` → `mapLineItems()`):
```
CartLineItem → LineItemInput:
  id, item_id, pos_id ← lineItem.id (as string)
  name ← lineItem.name
  price ← lineItem.weightVariant.price
  quantity ← lineItem.quantity
  sale_price ← lineItem.weightVariant.salePrice
  variants ← [{ id, name, price (salePrice || price), type }]
```

---

## 2. Checkout Flow

### Route Structure

```
app/(Storefront)/(Checkout)/
├── layout.tsx              — Wraps with SimpleHeader, Notification, SimpleFooter; sets robots: no-index
├── loading.tsx             — Loading skeleton
├── checkout/
│   ├── page.tsx           — Server component (fetches Prismic CMS data)
│   ├── page.component.tsx — Client component (main checkout logic, ~500 lines)
│   ├── types.ts
│   ├── constant.ts
│   └── checkout.module.scss
└── thank-you/
    ├── page.tsx           — Server component
    ├── page.component.tsx — Client component (order confirmation)
    └── type.ts
```

### Step-by-Step Customer Journey

**Step 1 — Sign-In Check** (`page.component.tsx` lines 290-317):
- If guest checkout disabled (`NEXT_PUBLIC_ALLOW_GUEST_CHECKOUT === 'false'`) and user not authenticated → show sign-in prompt instead of checkout.

**Step 2 — ID Verification (conditional)**:
- **Berbix/DIVE** — If Berbix app enabled and user authenticated for delivery: displays `DIVEComponent` modal (`components/DIVE/DIVE.component.tsx`). Max 5 attempts before lockout. Updates Cognito attribute `custom:BERBIX_VERIFY`.
- **Driver License Upload** — Alternative: `VerifyUploadingDLPhoto` component (`components/VerifyUploadingDLPhoto/VerifyUploadingDLPhoto.tsx`). Used when `allowIdVerificationDelivery` or `allowIdVerificationPickup` flags enabled.

**Step 3 — Checkout Form (managed by `@gap-commerce/checkout`)**:
The `<GCCheckout />` component renders and manages all internal checkout steps:
- Cart/line items review
- Delivery address or pickup location selection
- Delivery time window selection (unless `disbleDateAndTimeStep` flag)
- Customer information (name, email, phone, DOB)
- Payment method selection and processing
- Order confirmation/review

**Step 4 — Order Submission**:
Handled entirely inside `GCCheckout`. On success:
1. `onOrderUpdated(order)` callback fires → syncs `entity_id`, totals, Treez order ID, loyalty URLs back to Zustand
2. `onOrderSubmitted()` callback fires → `router.replace('/thank-you?entity_id={entity_id}')`

**Step 5 — Thank You Page** (`thank-you/page.component.tsx`):
1. Validates `entity_id` query param matches stored state
2. **Clears the ecom state** — resets cart, line items, discounts to initial values
3. Records purchase analytics:
   - `recordPurchase(email, entity_id, line_items)` — Amplify
   - `measurePurchaseGA4()` — Google Analytics 4
   - `measurePurchase()` — Blaze tracking
   - `measureSurfsideTransaction()` — Surfside analytics
4. Displays: order number (`#treez_order_number`), line items, totals, delivery/billing address, store map, contact info
5. Action buttons: "Continue shopping" (→ `/`), "Order detail" (→ account page)
6. Cancel order option if `allowCancelOrder` flag enabled and user authenticated

### Order Construction

The order object passed to `GCCheckout` is built from multiple sources (`page.component.tsx` lines 135-203):

- **Cart state** — `line_items`, `delivery_details`, `discount_items` from Zustand
- **Authenticated user** — email, DOB, name, phone from Cognito user attributes
- **Store address** — for pickup orders, uses store's address as delivery_address
- **Medical info** — `medical_id`, `medical_id_expiration_date` if customer_type is MEDICAL
- **Order type** — `AccountType.Medical` if MEDICAL, otherwise `AccountType.Adult`

### Address Validation and Store Switching

`resolveAddressForStore()` function (`page.component.tsx` lines 364-423):

When the customer changes their delivery address during checkout:
1. Validates the address is in a serviceable delivery zone
2. Checks if the address maps to a different Treez store
3. If store needs to change and customer has cart items:
   - **Express delivery** (provider_inventory_location_id exists): clears cart, switches store immediately
   - **Normal delivery**: preserves cart, sets `next_store_id` / `next_delivery_address`, redirects to home
4. Returns status: `SUCCESS`, `INVALID_ADDRESS`, or `REDIRECTING_DUE_TO_STORE_CHANGE`

### Store & Schedule Data

`useCheckoutStore()` hook (`hooks/useCheckoutStore.ts`):
- Maps Treez store data to the `Store` format expected by `GCCheckout`
- Filters store content based on delivery vs. pickup and geolocation
- Provides `refetchStoreSchedules()` to revalidate time slots via `/api/ssr-cache`

### Account Data Integration

`useGetAccount()` fetches (on mount, if authenticated):
- **Loyalty points** — Treez provider loyalty points → passed as `rewardPoints`
- **AlpineIQ verification** — `alpineiq_verified` → `alpineVerified` prop
- **Sticky customer ID** — `sticky_customer_id` → `hasStickyCardCustomerId` prop

### Feature Flags Controlling Checkout Behavior

| Flag | Purpose |
|------|---------|
| `isTaxAppliedMessage` | Display "Included" vs calculated tax |
| `requireStateOfResidence` | Require state selection |
| `allowCurbsidePickup` | Enable curbside option |
| `curbsidePickupPlaceholder` | Curbside note placeholder text |
| `checkoutCountDownTime` | Cart timeout in minutes |
| `allowCouponLoyaltyStacking` | Allow coupon + loyalty stacking |
| `allowIdVerificationDelivery` | Require ID for delivery |
| `allowIdVerificationPickup` | Require ID for pickup |
| `showStriketroughPrice` | Strikethrough original prices |
| `allowCancelOrder` | Allow cancel on thank-you page |
| `showRegulationModal` | Show regulation compliance modal |

---

## 3. Payment UI

### Available Payment Types

**Store-level payment configuration** (per reservation mode — DELIVERY or PICKUP):
- `ACH` — Bank transfers
- `CASH` — Cash at pickup
- `CREDIT` — Credit card
- `DEBIT` — Debit card

**Order-level payment types** (recorded on the order):
- `CASH`, `CHECK`, `CREDIT_CARD`, `DEBIT_CARD`
- `TREEZPAY` — TreezPay processor with ACH support
- `STRONGHOLD` — Payment gateway with ACH / bank link / tipping
- `AEROPAY` — Alternative payment processor
- `SWIFTER` — OAuth-based payment processor

### Payment Method Selection

Payment methods available to the customer are determined by the store's `payment_settings`:

```typescript
// utils/utils.ts → getPaymentMethods()
const paymentTypes = store?.payment_settings
  ?.map(payment => payment?.payment_type)
  ?.filter(Boolean) as StorePaymentType[];
return [...new Set(paymentTypes)];
```

Each `StorePaymentSettings` entry includes a `reservation_mode` (DELIVERY or PICKUP), so different methods can be offered for each fulfillment type.

### Payment Processing (Delegated to @gap-commerce/checkout)

The actual payment UI (form fields, card inputs, gateway integration) lives inside the `<GCCheckout>` component. treez-tlp passes configuration and the component handles:
- Stripe Elements integration (card entry)
- ACH bank link flows (Stronghold)
- Debit card with fee display (`debitCardFee` prop)
- ACH information banner (`bannerACHPaymentUrl` prop)
- Stronghold charge completion (`strongholdChargeId` from URL query params)

### Payment Hooks Available (defined but called by GCCheckout internally)

`data/api/payment/hooks.js` exports:
- `useCreateStripePayment(options)` — Stripe payment creation
- `useCreateAuthorizePayment(options)` — Authorize.net integration
- `useCreateWebpayTransaction(options)` — Webpay integration
- `useCreateWebpayMeasureTransaction(options)` — Webpay measurement

GraphQL mutations:
- `CREATE_STRIPE_PAYMENT_MUTATION` → returns `StripePaymentResponse` (charge_id, balance_transaction, card details)
- `CREATE_AUTHORIZE_PAYMENT_MUTATION`
- `CREATE_WEBPAY_TRANSACTION_MUTATION` → returns `{ token, url }`
- `CREATE_WEBPAY_MEASURE_TRANSACTION_MUTATION` → returns `{ token, url }`

### Payment Detail on Order

```typescript
type PaymentDetail = {
  type?: PaymentType;           // CASH, CREDIT_CARD, DEBIT_CARD, TREEZPAY, STRONGHOLD, AEROPAY, SWIFTER
  card_type?: string;           // Visa, Mastercard, etc.
  last_four?: string;           // Last 4 digits
  refund_id?: string;
  refunded?: boolean;
  aeropay_detail?: AeroPayPaymentDetail;
  stronghold_detail?: StrongholdPaymentDetail;
  swifter_detail?: SwifterPaymentDetail;
  treezpay_detail?: TreezPayPaymentDetail;  // { invoice_id, payment_id, payment_method, processor_name }
};
```

### TreezPay Integration

- Creates tickets per store: `treez_pay_ticket_ids_by_store: TreezPayTicket[]`
- Each ticket: `{ ticket_id, provider_store_id, created_at }`
- Payment method can include ACH (`treezpay_detail.payment_method === 'ACH'`)
- Thank-you page renders differently for TreezPay orders: `payment_details?.type === 'TREEZPAY'`

### Stronghold Tipping

```typescript
type StrongholdCredentials = {
  allow_tiping?: boolean;           // Note: typo in field name
  api_host?: string;
  sh_integration_key?: string;
  sh_secret_key?: string;
  store_provider_id?: string;
};

enum StrongholdPayType {
  BankLink = 'BANK_LINK',
  Checkout = 'CHECKOUT',
  Tipping = 'TIPPING'
}
```

Tipping is controlled by the `allow_tiping` flag on Stronghold credentials. Stronghold supports bank link, standard checkout, and tipping payment flows.

### ACH Configuration

Environment/config-driven:
- `ACH_ENABLED: boolean` — Global toggle
- `CURRENT_ACH_PROVIDER: string` — e.g., "Stronghold"
- `ACH_PROVIDER_ENVIRONMENT: string` — Sandbox vs production
- `ACH_PROVIDER_JS_LIBRARY_URL: string` — JavaScript SDK URL
- `ACH_PROVIDER_PUBLISHABLE_KEY: string` — Public API key
- `STRONGHOLD_ECOMMERCE_ACH_PAYMENT_ENABLED: string` — Stronghold-specific ACH flag

---

## 4. Promotion / Discount Application

### Promo Code Entry

**Kiosk checkout** — `app/(Kiosk)/kiosk/[store]/checkout/_components/sections/CheckoutOrderSummary/CheckoutOrderSummary.tsx`:
- Input field with "Coupon code" placeholder inside `OrderSummaryPromotionCode` component
- Applied via `applyPromoCode(promoCode)` from `CheckoutProvider`
- Once applied, input is disabled (`disabled={!!promo}`)
- Promo code stored in `discount_items` array as `promo_code` field
- Helper `isPromoItem()` identifies promo discount items

**Storefront checkout** — Promo code entry is handled inside the `<GCCheckout>` component. The `allowCouponLoyaltyStacking` feature flag controls whether coupons can stack with loyalty redemptions.

### Promotional Banners (CMS-Driven)

`useCurrentPromotion` hook (`hooks/useCurrentPromotion.ts`):
- Fetches promotional messages from Prismic CMS
- Validates by location slug **and** day of week
- Falls back to default promotion messages if no location-specific ones are active
- `Promotion` component (`components/Promotion/Promotion.tsx`) displays messages in a Swiper carousel with auto-rotate (6-second default)
- Promotions are dismissible if `dismissible: true` flag set in CMS
- Can link internally or externally

### Bundle Specials & BOGO

`useBundleSpecial` hook (`hooks/useBundleSpecial/useBundleSpecial.ts`):
- Manages BOGO (Buy X Get Y) and bundle specials from Treez
- Tracks **dependent products** (GET items — discounted/free) and **independent products** (BUY items — required purchase)
- Discount method types include: `BOGO`, `BUNDLE`, `TARGET_PRICE`, and others from Treez specials

UI components:
- `SpecialCounter` — Visualizes BOGO progress with step indicators and a progress bar
- `SpecialBanner` (`components/SpecialBanner/SpecialBanner.tsx`) — Displays special title and description
- Bundle marked complete when required product counts are reached

### Discount Display in Cart

`components/CartNavigation/Discount.tsx`:
- Collapsible section in the cart footer
- Lists individual special discounts with formatted titles and amounts
- Displayed as `-$X.XX` format
- Accessible with aria-labels and expand/collapse toggle

### Backend Evaluation

Discounts are **evaluated server-side** during the `updateTreezCart` mutation. The response includes:
- `specials: [{ title, amount }]` — Applied specials
- `order.total_discounts` — Total discount amount
- Updated `order.total_price` — Reflects discounts

---

## 5. Gram Limit Enforcement

### Configuration

- Environment variable: `NEXT_PUBLIC_GR_LIMIT` (default: `"35"` grams)
- Feature flag: `NEXT_PUBLIC_FEATURE_FLAG_GRLIMIT_ACTIVE` (default: `"false"`)

### Calculation

`hooks/useSummary.js`:
```javascript
const gramsLimit = parseInt(process.env.NEXT_PUBLIC_GR_LIMIT);
// totalGrams = sum of (formatWeightNumber(weightVariant) * quantity) for all line items
```

### Display

`components/GrLimit/GrLimit.tsx`:
- Shows: `{localGrams} gr / {gramsLimit} gr limit`
- Clicking opens the cart drawer

### Enforcement Status: Currently Disabled

The enforcement logic is **commented out** in the codebase:

- `components/Counter/Counter.tsx` (lines 293-296) — `// Disable it for now`
- `components/Counter/CounterActions.tsx` (lines 62-66) — `// Disable it for now`

When enabled, it would:
- Block adding items in `addItemToTheCart()` if limit exceeded
- Block quantity increases in `updateQuantity()`
- Block weight variant changes if the new weight would exceed the limit
- Show error notification via `useNotify()`: `GRAM_LIMIT_EXCEEDED_ERROR`

Currently, the `GrLimit` component **displays** the usage but does **not prevent** exceeding the limit.

---

## 6. Medical vs Recreational

### Feature Flag

`NEXT_PUBLIC_FEATURE_FLAG_MED_REC_SELECTION_ACTIVE`:
- When enabled: default `customer_type` = `ADULT` (recreational)
- When disabled: default `customer_type` = `NOT_SET`

### UI Components

**MedRecSwitch** (`components/MedRecSwitch/MedRecSwitch.tsx`) — Header toggle:
- Two buttons: "REC" and "MED" with visual selection indicator
- Only renders when `isMedAndRecSelectionActive` is true
- Stores selection in cookie: `CookiesNames._CUSTOMER_TYPE`

**MedRecValidator** (`components/MedRecValidator/MedRecValidator.tsx`) — Modal on mode switch:
- Triggered when user switches customer types **and has items in cart**
- Warning: "Be careful! You are about to change shopping mode. Your items will be removed from your shopping cart."
- For MEDICAL mode switch, shows:
  - Medical ID input (min 5 chars; label varies if `enableMMIDTooltip` flag)
  - Medical ID expiration date picker (must be a future date)
- Buttons: "remove items & change mode" or "Keep items & don't change mode"
- **Cart is cleared** when mode switches with items present

### Effect on Checkout

- `order_type` set based on customer_type:
  - `MEDICAL` → `AccountType.Medical`
  - All others → `AccountType.Adult`
- Medical ID and expiration date passed in order: `order_medical_id`, `order_medical_id_expiration_date`
- Different minimum age enforcement based on medical vs adult
- Products can be filtered by customer type (`PRODUCTS_BY_CUSTOMER_TYPE` in Treez site config) — medical-only, adult-use-only, or both

### State Fields

```
customer_type: CustomerType            — Current: ADULT, MEDICAL, ALL, NOT_SET
next_customer_type: CustomerType       — Pending mode change
medical_id: string                     — MMID
medical_id_expiration_date: string     — ISO expiration date
```

---

## 7. Age Gate

### Middleware Enforcement

`middleware.ts` enforces age verification at the request level:

1. Checks cookie `treez_terms_accepted` (`CookiesNames._TERMS_ACCEPTED`)
2. Cookie value can be:
   - Simple string `'true'`
   - JSON: `{ terms_accepted: true, terms_accepted_exp_date: <ISO date or null> }`
3. If `terms_accepted_exp_date` exists, validates it hasn't expired
4. If missing or expired → redirect to `/age-gate`

### Routes Exempt from Age Gate

The middleware matcher (lines 136-146) excludes:
- `/api` routes
- `_next`, `_vercel` internals
- Static files (favicon, images, icons, locales)
- `/age-gate` itself
- **`/kiosk` routes — kiosk is fully excluded from age gate**
- Static file extensions (.js, .json, .txt, .xml, .webmanifest)

### Interaction with Checkout

The age gate is **not re-checked during checkout**. The cookie set on initial visit persists across all routes. During checkout:
- No additional age verification occurs at the middleware level
- The cookie remains valid for 10 years (set via `age-gate/auto-accept/route.ts` with `maxAge`)
- If the cookie somehow expires mid-session, the **next navigation** would redirect to age gate (not mid-checkout)

### Bot Bypass

The middleware detects bots via user-agent matching and skips the age gate for SEO crawlers.

### Kiosk Exemption

Kiosk routes are explicitly excluded from the middleware matcher. Age verification in kiosk contexts is expected to happen physically (store staff / in-person).

---

## 8. Kiosk Checkout

### Route Structure

```
app/(Kiosk)/kiosk/[store]/checkout/
├── page.tsx                    — Wraps in CheckoutProvider
├── _context/
│   ├── CheckoutContext.ts     — Context type definition
│   ├── CheckoutProvider.tsx   — State, validation, mutations
│   └── useCheckoutContext.ts  — Consumer hook
└── _components/
    ├── header/CheckoutHeader.tsx
    ├── main-content/CheckoutMainContent.tsx
    ├── order-button/PlaceOrderButton.tsx
    └── sections/
        ├── CheckoutOrderSummary/    — Promo codes, products, totals
        ├── CheckoutContactDetails/  — Name, email, phone, DOB, state
        ├── CheckoutPaymentDetails/  — Display accepted payment methods
        └── CheckoutProductsPreview/ — Cart items with quantity controls
```

### How It Differs from Standard Web Checkout

| Aspect | Kiosk | Standard Web |
|--------|-------|--------------|
| Checkout component | Custom-built (CheckoutProvider + sections) | `@gap-commerce/checkout` (`<GCCheckout>`) |
| Layout | Two-column (left: forms, right: products) | Multi-step managed by GCCheckout |
| Payment | **Display only** — "Payment will be made upon pickup" | Online payment processing |
| Fulfillment | Pickup only (implicit) | Delivery + pickup |
| Delivery address | Not collected | Required for delivery |
| Age gate | Excluded from middleware | Required |
| Idle timeout | Yes (configurable) | No |
| Product editing | Can adjust quantity / remove in checkout | Locked at checkout entry |
| Submit mutation | `submitKioskCheckout` | Internal to GCCheckout |
| Post-submit | Auto-redirects after timeout | Stays on thank-you page |

### Kiosk Checkout Provider

`CheckoutProvider` (`_context/CheckoutProvider.tsx`) manages:
- Form state: inputs, validators, change handlers
- Cart sync: `useUpdateTreezCart` mutation, fired on mount and item changes
- Promo code: `applyPromoCode(code)` function
- Submit: `useSubmitKioskCheckout` mutation
- Error notifications: `useNotify()` for mutation errors

### Kiosk Payment Display

`CheckoutPaymentDetails` shows accepted methods but does **not** collect payment:
- "Payment will be made upon pickup. We accept the following payment methods:"
- Displays icons for: Cash, Debit Card, Credit Card, ACH
- Methods come from `getPaymentMethods(store)` filtered by reservation mode

### Kiosk Submission

`PlaceOrderButton` (`_components/order-button/PlaceOrderButton.tsx`):
1. Validates form via `useValidator`
2. Calls `useSubmitKioskCheckout` mutation:
   ```graphql
   mutation submitKioskCheckout($account_id, $store_id, $input: updateCartInput!) {
     submitKioskCheckout(account_id, store_id, input) { orderId }
   }
   ```
3. On success → redirect to `/kiosk/[store]/thank-you?orderId={orderId}`
4. On error → toast notification

### Idle Timeout

`IdleTimeModal` (`app/(Kiosk)/_components/IdleTimeModal/IdleTimeModal.tsx`):
- Configurable: `NEXT_PUBLIC_KIOSK_TIMEOUT_SECONDS` (default 15s), `NEXT_PUBLIC_KIOSK_COUNTDOWN_TIME` (default 10s)
- After inactivity timeout, shows "Are you still here?" with countdown
- "Yes" resets timer
- On countdown expiry: clears cart, resets UI, redirects to kiosk start

### Kiosk Thank-You Page

`app/(Kiosk)/kiosk/[store]/thank-you/page.tsx`:
- Displays order ID and total payment
- `ShopAgain` component auto-redirects after `NEXT_PUBLIC_KIOSK_THANK_YOU_TIMEOUT_SECONDS` (default 10s)
- Clears ecom state and UI state on redirect

---

## 9. Error Handling

### Notification System

`useNotify()` hook (`hooks/useNotify.ts`):
- Uses `react-toastify` with position: bottom-center, auto-close: 3000ms, slide transition, colored theme
- Severity levels: `success`, `warn`, `error`, `default`

### Cart Update Errors

When `updateTreezCart` mutation completes:
- Checks for `error` object in response
- If error: `notify('error', error.message)`
- On GraphQL error: catches and notifies

### Checkout Submission Errors

**Kiosk** — `PlaceOrderButton`:
- `onError`: `notify('error', error.message)` — red toast for 3 seconds
- Button disabled + loading spinner during submission; re-enabled on failure so user can retry

**Storefront** — handled internally by `<GCCheckout>`; error handling is within the package.

### Promo Code Errors

`CheckoutProvider.applyPromoCode()`:
- Try/catch with `notify('error', error?.message)` on failure
- Finally block resets `isApplyingPromoCode` loading state

### Validation Errors

- Field-level feedback: red text below invalid inputs
- Form-wide validation prevents submission if any field invalid
- Place Order button disabled while cart update is in progress

### Payment / POS Rejection

- **Kiosk**: Not applicable — payment happens at POS counter after customer places the order. POS rejection is outside this app's scope.
- **Storefront**: Handled internally by `<GCCheckout>`. Error states are rendered by the checkout component.

### Global Error Fallback

`components/ErrorFallback/ErrorFallback.tsx`:
- Catches unhandled component errors via Error Boundary
- Displays: "500 - Oops! Something went wrong"
- Offers: Retry button and "Return to Home" link

---

## 10. `@gap-commerce/checkout` Package

### Overview

- **Version**: 1.9.12
- **Registry**: GitHub Packages (npm.pkg.github.com)
- **Peer deps**: React ^18 || ^19
- **Imported in**: exactly one file — `app/(Storefront)/(Checkout)/checkout/page.component.tsx`

### What It Exports

- `Checkout` component (imported as `GCCheckout`)
- CSS: `@gap-commerce/checkout/dist/style.css`

### What Lives in the Package vs. treez-tlp

**Inside @gap-commerce/checkout:**
- Full checkout flow UI (multi-step form)
- Payment processing logic (Stripe Elements, ACH, etc.)
- Order submission (GraphQL createOrder mutation)
- Address validation against delivery schedules
- Time slot selection
- Loyalty points redemption UI
- Age verification checks
- Medical ID validation
- Discount/coupon application UI
- Tax display and calculation
- Cart countdown timer

**Inside treez-tlp:**
- Global state management (Zustand ecomStore)
- Store data enrichment (`useCheckoutStore()` maps Treez store → GCCheckout's expected `Store` shape)
- Authentication (Cognito via `useAuth()`)
- Account data fetch (loyalty points, AlpineIQ status, Sticky customer ID)
- Delivery address validation with store switching (`resolveAddressForStore()`)
- Order state sync (`updateEcomStateFromOrder()` maps GCCheckout's Order back to EcomState)
- ID verification pre-checkout (Berbix DIVE, DL upload)
- Age gate enforcement (middleware)
- Product management (cart add/remove via `useProducts()`)
- Analytics recording on thank-you page
- Theming and feature flag assembly

### Props Interface

```typescript
<GCCheckout
  // Config
  config={{ accountId, serverUrl, storeId }}
  authToken={token}
  isAuth={boolean}
  googleMapsApiKey={string}

  // Order & Store
  order={OrderInput}
  store={Store}
  msoOrgId={string}
  msoStoreEntityId={string}

  // Callbacks
  onOrderSubmitted={() => void}
  onOrderUpdated={(order: Order) => void}
  onCartItemAdded={(product) => void}
  onCartItemRemoved={(product) => void}
  onCartItemChangeCount={() => void}
  resolveAddressForStore={(address) => Promise<status>}
  refetchStoreSchedules={() => Promise}
  navigateTo={(route) => void}

  // Feature Flags
  featuredFlag={{
    showStriketroughPrice: boolean,
    disableStoreRewards: boolean,
    disbleDateAndTimeStep: boolean,
    requireStateAtCheckout: boolean,
    enableCurbsidePickup: boolean,
    curbsidePickupPlaceholder: string,
    allowCouponLoyaltyStacking: boolean,
  }}

  // Pricing & Validation
  minAgeAllowed={number}
  isBelowDeliveryMinimum={boolean}
  displayPostTax={boolean}
  debitCardFee={number}
  cartTimeoutMinutes={number}

  // Loyalty & Rewards
  rewardPoints={number}
  rewardType="DOLLAR"
  hasStickyCardCustomerId={boolean}
  alpineVerified={boolean}

  // Payment
  bannerACHPaymentUrl={string}
  strongholdChargeId={string}

  // Theme
  theme={{
    backgroundColor, primaryButtonBackgroundColor, primaryButtonTextColor,
    secondaryButtonBackgroundColor, secondaryButtonTextColor,
    buttonBorderRadius, modalBorderRadius,
  }}
/>
```

### State Flow

1. **Mount** — EcomState (Zustand) contains cart, delivery, customer info
2. **Enrichment** — `useCheckoutStore()` maps store data with geo info and schedules
3. **User Interaction** — GCCheckout manages its own internal form state
4. **Sync Back** — `onOrderUpdated` writes changes back to EcomState
5. **Submit** — GCCheckout submits internally, calls `onOrderSubmitted`
6. **Post-Checkout** — EcomState persists for the thank-you page, then is cleared

---

## GraphQL Operations Summary

| Operation | Mutation/Query | Used By | Purpose |
|-----------|---------------|---------|---------|
| `updateTreezCart` | Mutation | CartNavigation, CheckoutProvider | Sync cart, recalculate totals |
| `submitKioskCheckout` | Mutation | PlaceOrderButton (kiosk) | Submit kiosk order |
| `submitCart` | Mutation | Legacy (Jane-based) | Submit cart (older flow) |
| `customerCancelOrder` | Mutation | Thank-you page | Cancel confirmed order |
| `getOrder` | Query | Thank-you page, order detail | Fetch order by entity_id |
| `createStripePayment` | Mutation | GCCheckout (internal) | Create Stripe charge |
| `createWebPayTransaction` | Mutation | GCCheckout (internal) | Create Webpay transaction |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/treez-tlp (branch: master, commit: 600eab80)*
