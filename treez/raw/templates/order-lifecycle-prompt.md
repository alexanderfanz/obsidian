# Order Lifecycle Deep-Dive — Repo Prompts

Targeted prompts to extract full order lifecycle details from each repo. Run each prompt inside the corresponding repo with Claude Code.

All outputs go to `raw/order-lifecycle/` in this vault.

---

## 1. e-com-lib

**Run in:** `/path/to/e-com-lib`

**Prompt:**
> Read this codebase and produce a detailed order lifecycle document focused exclusively on how this library defines and supports the order entity. Cover:
>
> 1. **Order model** — every field on the Order struct, what each field means, which are required vs optional, and which are populated at which stage of the lifecycle
> 2. **OrderStatusType enum** — every possible status value, what each means, and any constraints on transitions
> 3. **Order service interface** — every method on the Order service interface, what it does, its signature, and when it's called
> 4. **LineItem model** — all fields, how line items relate to products/variants, how prices are represented (cents? floats?)
> 5. **Promotion evaluation** — how the govaluate-based promotion engine works: what expressions are supported, what context variables are available, how automatic vs coupon promotions differ
> 6. **Cart vs Order distinction** — how drafts (key="d#") differ from submitted orders in the data model
> 7. **Payment-related fields** — all payment fields on Order, how different payment providers are represented
> 8. **Activity log** — how order activity/history is tracked on the model
> 9. **SQS event schemas** — what order-related event types exist in the worker framework, their payload shapes
> 10. **Any helper functions** that manipulate order state (price calculation, tax, totals, discount application)
>
> Be exhaustive — include field names, types, and enum values. This will be used to document the complete order lifecycle across the platform.
>
> Save to `/Users/alex/obsidian/treez/raw/order-lifecycle/e-com-lib-order.md`

---

## 2. e-com-srv

**Run in:** `/path/to/e-com-srv`

This is the most important one — it owns the cart/checkout flow.

**Prompt:**
> Read this codebase and produce a detailed order lifecycle document. This service is the central orchestrator for the order — I need the complete picture of every code path that creates, modifies, or finalizes an order. Cover:
>
> 1. **Cart creation** — how a cart is created, what the initial state looks like, how `key="d#"` works
> 2. **UpdateCart mutation** — the full sequence of what happens on every call: line item management (add/remove/update quantity), price recalculation logic, tax calculation, promotion evaluation (automatic + coupon), how totals are derived. Include the exact order of operations.
> 3. **SubmitCart mutation** — the full sequence: validation checks before submission, payment initiation per provider (AeroPay, Stronghold, Swifter, TreezPay — detail each), POS submission per provider (Treez, Dutchie, Jane, Blaze — detail each handler file and what it does), SNS event publishing, what happens on success vs failure
> 4. **UpdateOrderStatus mutation** — who calls this, what transitions are allowed, what side effects fire
> 5. **UpdateOrderDetails mutation** — what fields can be updated post-submission
> 6. **Multi-POS fan-out** — how a single cart syncs to multiple POS systems, what happens when one POS call fails but another succeeds
> 7. **Payment flows per provider** — AeroPay (token attachment), Stronghold (tipping), Swifter (OAuth), TreezPay (ticket creation, the duplicate ticket bug). What are the exact steps for each?
> 8. **Promotion engine in context** — how ApplyPromotionCode works end-to-end, how promotions interact with POS pricing
> 9. **Error handling** — what happens when payment fails, POS rejects, timeout occurs. What state does the order end up in?
> 10. **CRM subscription flow** — SubscribeToCrm: what triggers it, how fan-out works, error handling
> 11. **ID verification flow** — Berbix and IDScan: when in the checkout flow they're called, what happens on pass/fail
> 12. **GraphQL resolvers** — list every order-related query and mutation with their resolver file locations
> 13. **The TreezPay duplicate ticket bug** — read `docs/TREEZ_DUPLICATE_TICKET_ENTITY_ID_BUG.md` and include the full details: root cause, reproduction steps, impact, any workarounds
>
> For each handler file (handle_order_treez.go, handle_dutchie_order.go, etc.), describe what it does step by step. Don't summarize — be specific about the sequence of operations, error conditions, and edge cases.
>
> Save to `/Users/alex/obsidian/treez/raw/order-lifecycle/e-com-srv-order.md`

---

## 3. api-srv

**Run in:** `/path/to/api-srv`

**Prompt:**
> Read this codebase and produce a detailed document about how this service handles order status changes from external systems. Cover:
>
> 1. **Treez order webhook** (`/treez/order-updated`) — the complete processing flow:
>    - Revenue source filtering (which orders are processed vs skipped)
>    - Event type filtering (TICKET_STATUS and others)
>    - The polymorphic event_type handling (string vs array)
>    - Full status mapping: every TreezOrderStatus value → internal OrderStatusType
>    - What side effects fire for each status: Alpine IQ SMS, LisTrack email, SNS events
>    - The DeclinedFromAdmin flag reset logic
>    - Abandoned cart handling
>    - Activity log entries
> 2. **Jane order webhook** (`/jane/order-updated/`) — complete flow:
>    - Auth (Bearer token)
>    - Full status mapping: every JaneOrderStatus (all 11) → internal status
>    - Delivery vs pickup path differences
>    - Delivery timestamp enrichment from Jane API
>    - LisTrack email triggers per status
>    - SNS notification publishing
> 3. **Blaze order webhook** (`/blaze/order-updated`) — complete flow:
>    - Status mapping
>    - Side effects
> 4. **OnFleet delivery webhooks** (all 6 task events) — for each:
>    - What order status transition occurs
>    - What combination of DynamoDB update, LisTrack email, and Treez ticket sync happens
>    - TaskFailed: the full revert flow (Declined status, OnFleet reference removal, Treez ticket removal, rejection email)
>    - TaskCompleted: payment detail sync to Treez, completion email
> 5. **Request builders** — how each provider's RequestBuilder validates auth, extracts context, and enriches the request
> 6. **Error handling** — what happens when a status update partially fails (e.g., DynamoDB succeeds but Treez ticket sync fails)
> 7. **Alpine IQ discount reversion** — when and how loyalty discounts are reverted on order decline
>
> Include the exact status string mappings (e.g., `"COMPLETED" → model.OrderStatusCompleted`). Be exhaustive on every status value each provider can send.
>
> Save to `/Users/alex/obsidian/treez/raw/order-lifecycle/api-srv-order.md`

---

## 4. e-com-order-worker

**Run in:** `/path/to/e-com-order-worker`

**Prompt:**
> Read this codebase and produce a detailed document about the post-purchase order processing pipeline. Cover:
>
> 1. **order/completed event** — the full processor chain in exact order:
>    - How the event payload is parsed and the full order is loaded
>    - Invoice number assignment: the atomic counter mechanism, what happens on counter failure
>    - Order status update to PENDING: what fields are set, activity log entry
>    - Account upsert: what fields are created/updated, lifetime spend aggregation, medical ID/DOB copy logic
>    - Conditional routing: how each integration check works (apps.json flags)
>    - Each SNS event published: exact event name, payload shape, conditions under which it fires
>    - Treez automatic_approval flag: what it controls
> 2. **order/kiosk-completed event** — what's different from the full pipeline, which processors are skipped and why
> 3. **Processor chain details** — for each processor in the chain:
>    - What it does
>    - What it reads (DynamoDB, S3, etc.)
>    - What it writes/publishes
>    - Error behavior (does it fail the pipeline or continue?)
> 4. **Non-critical failure handling** — which processors return nil on failure and why
> 5. **New Relic events** — what custom analytics events are emitted, with what attributes
>
> Save to `/Users/alex/obsidian/treez/raw/order-lifecycle/e-com-order-worker-order.md`

---

## 5. e-com-app-worker

**Run in:** `/path/to/e-com-app-worker`

**Prompt:**
> Read this codebase and produce a detailed document about order-related event processing. Cover:
>
> 1. **place-order-treez** — the full processor chain (PlaceOrderTreez → AlpineIQDiscount → StickyDiscount):
>    - Order loading and store/app config resolution
>    - Treez order submission: API calls made, payload construction, how internal order maps to Treez ticket
>    - Verification photo upload: the upload plan logic (new vs returning customer, which doc types), S3 fetch, Treez document API upload, table-driven test coverage details
>    - Payment type detection: credit vs debit card logic, how it maps to Treez payment types
>    - AlpineIQ discount redemption: when it runs, what API calls, error tolerance
>    - Sticky discount redemption: when it runs, what API calls, error tolerance
>    - What happens when PlaceOrderTreez fails — do discounts still run?
>    - SNS completion event published
> 2. **place-order-blaze** — full flow: how internal order maps to Blaze API, what's stored in DynamoDB (BlazeOrder record)
> 3. **place-order-jane** — full flow: how internal order maps to Jane API, what's stored in DynamoDB (JaneOrder record)
> 4. **place-order-klaviyo** — what order data is sent to Klaviyo, event tracking details
> 5. **Error handling per processor** — which errors are fatal (message stays on queue) vs non-fatal (logged, message deleted)
>
> Save to `/Users/alex/obsidian/treez/raw/order-lifecycle/e-com-app-worker-order.md`

---

## 6. e-com-notification-worker

**Run in:** `/path/to/e-com-notification-worker`

**Prompt:**
> Read this codebase and produce a detailed document about order notification processing. Cover:
>
> 1. **For each event type** (order/confirm_notify, order/delivered_notify, order/out_of_stock_notify, order/status_changed, contact/business_owner):
>    - What triggers this event (which upstream service publishes it)
>    - What data is loaded to build the email (DynamoDB order, S3 store config, S3 notification templates)
>    - The template context (Params struct): every field available to the email template
>    - Subject line selection logic
>    - Recipient selection (customer email, merchant email, both?)
>    - The `FromTxEmail` override behavior
> 2. **status_changed dynamic template selection** — the full mapping between order status values and template keys
> 3. **EmailNotificationActive gating** — what happens when disabled, is the message still deleted from SQS?
> 4. **Template rendering** — the helper functions registered (inc, formatPrice), any formatting quirks
> 5. **Error scenarios** — what happens if template is missing in S3, if SES send fails, if order can't be loaded
>
> Save to `/Users/alex/obsidian/treez/raw/order-lifecycle/e-com-notification-worker-order.md`

---

## 7. treez-tlp (optional but valuable)

**Run in:** `/path/to/treez-tlp`

**Prompt:**
> Read this codebase and produce a detailed document about the customer-facing order/checkout flow. Cover:
>
> 1. **Cart management** — how the cart is created, stored (cookies? Zustand? Apollo?), and synced with the backend
> 2. **Checkout flow** — the step-by-step pages/components the customer goes through: cart review → delivery/pickup selection → customer info → payment → confirmation. What happens at each step?
> 3. **Payment UI** — which payment methods are shown, how payment method selection works, Stripe integration, ACH/debit flows
> 4. **Promotion/discount application** — how coupons are applied in the UI, bundle specials, BOGO display
> 5. **Gram limit enforcement** — where and how the limit is checked, what the user sees when exceeded
> 6. **Medical vs recreational** — how this affects the checkout flow, product availability, pricing
> 7. **Age gate** — how it interacts with checkout (is it re-checked?)
> 8. **Kiosk checkout** — how the kiosk flow differs from the standard web checkout
> 9. **Error handling** — what the customer sees on payment failure, POS rejection, timeout
> 10. **`@gap-commerce/checkout` package** — what logic lives in this shared package vs in treez-tlp itself
>
> Save to `/Users/alex/obsidian/treez/raw/order-lifecycle/treez-tlp-order.md`

---

## 8. gap-dashboard (optional)

**Run in:** `/path/to/gap-dashboard`

**Prompt:**
> Read this codebase and produce a detailed document about admin order management. Cover:
>
> 1. **Order list view** — how orders are queried (GraphQL? OpenSearch?), filtering options, status display
> 2. **Order detail view** — what information is shown, which fields are editable
> 3. **Status transitions** — what status changes can admins make, which mutations are called
> 4. **Delivery management** — scheduling, window assignment, driver assignment (if applicable)
> 5. **Refund/cancellation** — admin flows for rejecting or cancelling orders
> 6. **Order search** — how search works (Algolia? OpenSearch?), what fields are searchable
>
> Save to `/Users/alex/obsidian/treez/raw/order-lifecycle/gap-dashboard-order.md`

---

## After all prompts are run

Come back to this vault and say:

> **"ingest order lifecycle"**

Claude will read all `raw/order-lifecycle/*.md` files and produce a comprehensive, cross-repo order lifecycle wiki page that covers every status transition, every code path, every edge case, and every notification — with links to the relevant service pages.
