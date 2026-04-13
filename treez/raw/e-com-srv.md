# e-com-srv

> GraphQL eCommerce backend that acts as the central orchestration layer for cannabis retail operations. It manages orders, accounts, products, and promotions while coordinating with multiple POS systems, payment processors, CRM platforms, and logistics services. Runs as an AWS Lambda (serverless) function and serves as the primary API consumed by Treez/GAP Commerce storefronts.

---

## Purpose

Provide a unified GraphQL API that abstracts over multiple cannabis POS/ERP systems (Treez, Dutchie, Jane, Blaze) and payment/CRM integrations. The service owns the canonical cart/order lifecycle from product browsing through checkout and status tracking. It does **not** own inventory as source of truth — that lives in the connected POS systems. It does **not** serve the frontend assets; those are deployed on Vercel.

---

## Key Entities / Domain Models

- **Order** — The central entity. Doubles as a shopping cart (`key: "d#"` = draft) and a submitted order. Key fields: `EntityID` (UUID), `Status` (AbandonedCart → Pending → Approved → Rejected → Completed), `LineItems`, `SubtotalPrice`/`TotalPrice` (cents), `ProviderStoreID`, and provider-specific IDs (`DutchieOrderID`, `TreezPayTicketIdsByStore`, `JaneOrderID`). Stored in DynamoDB.

- **Account** (Customer) — Stored in DynamoDB alongside orders. Holds PII (email, phone, addresses), age/payment verification flags, and photo references (S3). Managed via AWS Cognito for auth.

- **Store** — Multi-tenant configuration unit. Each `(accountID, storeID)` pair has its own DynamoDB table, S3 resource paths, connected provider stores, app integrations, and tier plan. Drives all feature gating.

- **App** (Integration) — Configuration record for a third-party integration attached to a store. Fields: `handler` (dutchie, treez, aeropay, etc.), `category` (Payment, CRM, Inventory, etc.), `status` (enabled/disabled), and handler-specific credentials.

- **Product** — Loaded from S3 metadata files synced from POS systems. Includes pricing, variants, categories, and availability. Read-heavy, rarely mutated directly.

- **Promotion** — Discount rules stored in S3. Types: `AUTOMATICALLY` (applied always) and `COUPON_CODE` (applied by code). Contain scheduling, eligibility rules, and discount amounts.

- **LineItem** — A product within an order. Carries price, quantity, barcode (for POS submission), and variant details.

---

## API Surface

Single GraphQL endpoint. All operations are POST to `/query` (prod) or `/dev/query` (dev). GraphiQL playground at `/graphiql` in non-prod only.

**Key Queries:**

| Method | Path / Name | Description |
|--------|-------------|-------------|
| Query | `GetOrder` | Fetch a single order/cart by entityID |
| Query | `ListStoreOrders` | Paginated orders for a store (cursor-based) |
| Query | `CustomerOrderHistory` | Order history for a customer email |
| Query | `ListAccounts` | Paginated customer accounts with filter |
| Query | `GetAccount` | Fetch single account |
| Query | `ListStoreApps` | All integration apps for a store |
| Query | `GetStoreApp` | Single integration app config |
| Query | Products, Promotions, Categories, Brands, Vendors, Pages, Navigation | S3-backed metadata reads |

**Key Mutations:**

| Method | Path / Name | Description |
|--------|-------------|-------------|
| Mutation | `UpdateCart` | Add/remove items, apply discounts, recalculate totals |
| Mutation | `UpdateJaneCart` | Jane-specific cart update flow |
| Mutation | `SubmitCart` | Convert cart to order, push to POS, charge payment |
| Mutation | `UpdateOrderStatus` | Change order status (admin/POS webhook) |
| Mutation | `CreateAccount` / `UpdateAccount` | Customer account management |
| Mutation | `MigrateUser` | Cognito user migration helper |
| Mutation | `UploadAccountPhoto` | Upload ID/profile photo to S3 |
| Mutation | `AddAeropayPaymentDetail` | Attach Aeropay payment to order |
| Mutation | `ApplyPromotionCode` | Validate and apply coupon code |
| Mutation | `SubscribeToCrm` | Enroll customer in CRM (async via SNS) |
| Mutation | `CreateBerbixToken` / `IDScanVerification` | ID verification flows |
| Mutation | `UpdateOrderDetails` | Update order metadata (notes, scheduled time, etc.) |

---

## Data Layer

- **DynamoDB** — Transactional store for Orders and Accounts. Each `(accountID, storeID)` pair gets its own table (`DynamoOrderTableName`). Orders and accounts live in the same table distinguished by the `key` attribute (`"d#"` for cart drafts, `"ACCOUNT"` for accounts).

- **S3** — Semi-static metadata store for Products, Promotions, Categories, Brands, Vendors, Pages, Navigation, Notifications, Apps, Stores, Taxes, and Tags. Resources are keyed as `{accountId}#{storeId}#{resourceKey}`. A separate assets bucket hosts public CDN content. Account photos go to a dedicated bucket.

- **AWS SSM Parameter Store** — Service configuration and secrets loaded at startup via `srv-utils.NewInOsConfig`.

- **Cognito** — User pool for customer auth (JWT issuance and verification). One user pool per environment.

- **No explicit caching layer** — S3 metadata is loaded into memory per request. Jane specials are cached back to S3 (`saveSpecialsCache`). No Redis or ElastiCache.

---

## Core Business Logic

**Order/Cart Lifecycle:**
`UpdateCart` recalculates the entire order on each call — line item prices, promotion discounts, taxes, and totals — before persisting to DynamoDB. On `SubmitCart`, the service pushes the order to the connected POS system and, if payment is configured, initiates the payment flow. Order status transitions are: AbandonedCart → Pending (on submit) → Approved/Rejected (from POS or admin) → Completed.

**Multi-POS Fan-Out:**
A single `UpdateCart` call may synchronize state to multiple POS systems simultaneously (Treez + Jane, for example). Each POS has a dedicated handler file (`handle_order_treez.go`, `handle_dutchie_order.go`, etc.). The handlers translate canonical order models into POS-specific API calls.

**Promotion Engine:**
Promotions are evaluated on every cart update. Automatic promotions are always applied if eligible; coupon-code promotions are applied explicitly. Logic lives in `promotion.go` and is exercised by `promotion_test.go`.

**Payment Abstraction:**
Payment providers (Aeropay, Stronghold, Swifter, TreezPay) are selected per store based on enabled `App` records. Each has its own handler file. Payment operations are embedded in `SubmitCart` rather than being separate mutations (except Aeropay's token attachment).

**CRM Subscriptions:**
`SubscribeToCrm` publishes an SNS event rather than calling CRM APIs directly. Downstream consumers fan out to Alpine IQ, Sticky, Klaviyo, etc. Multiple CRM apps per store are processed concurrently via `sync.WaitGroup`.

**ID Verification:**
Two providers (Berbix, IDScan) are supported. Verification state is stored on the Account record.

**Tenant Isolation:**
All business logic operates within a `(accountID, storeID)` scope. S3 keys and DynamoDB table names are namespaced by this pair. GraphQL auth middleware enforces that callers can only access their own account/store.

---

## Cross-Repo Connections

- **Calls**:
  - `github.com/gap-commerce/e-com-lib` — shared domain models, service interfaces (Order, Account, Store, Product, etc.), and service implementations for S3/DynamoDB access. This is the primary internal dependency.
  - `github.com/gap-commerce/glog` — structured logging.
  - `github.com/gap-commerce/srv-utils` — config loading, AWS client setup.

- **Called by**: Storefront frontends (Nextjs apps deployed on Vercel) and internal admin tooling consume the GraphQL API. No known service-to-service callers at the HTTP level.

- **Shared types / contracts**: All canonical domain models (`Order`, `Account`, `Store`, `Product`, `Promotion`, `App`) are defined in `e-com-lib`, not this repo. Schema changes require coordinated updates to `e-com-lib`.

- **Events (publishes)**:
  - SNS topic `gc-topic` — event types include `app/{handler}` (CRM subscription events), `order/completed`, `order/status_changed`. Message attribute `publisher = "gapcommerce"`.

- **Events (subscribes)**: None directly. Status updates come in through `UpdateOrderStatus` mutations (likely called by webhook forwarders or admin tools).

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| Treez | POS / ERP | Cannabis POS; order submission, ticket management, product sync, TreezPay payments |
| Dutchie | POS / E-commerce | Cannabis POS; cart sync, order creation, customer data sync |
| Jane | POS / E-commerce | Cannabis POS; order history sync, special pricing, customer data |
| Blaze | ERP | Inventory sync (products, categories, promotions, brands, vendors) |
| Aeropay | Payments | ACH-based payment gateway with merchant accounts |
| Stronghold | Payments | Payment processing with tipping support |
| Swifter | Payments | Payment processing with OAuth flow |
| TreezPay | Payments | Treez-integrated ACH payment ticketing |
| Alpine IQ | CRM | Loyalty platform; contact verification, wallet management, loyalty points |
| Sticky | CRM | CRM platform; subscription management |
| Klaviyo | Email | Email marketing; customer subscriptions |
| Berbix | Auth / ID Verification | Age/identity verification |
| IDScan | Auth / ID Verification | Alternative ID scanning |
| OnFleet | Logistics | Delivery routing, driver management |
| LisTrack | Logistics | Order tracking with event publishing |
| Vercel | Infrastructure | Deployment status checks for frontend apps |
| AWS Lambda | Infrastructure | Serverless compute runtime |
| AWS DynamoDB | Infrastructure | Transactional order/account storage |
| AWS S3 | Storage | Metadata, assets, user photos |
| AWS Cognito | Auth | Customer user pool, JWT auth |
| AWS SNS | Infrastructure | Async event publishing |
| AWS SSM | Infrastructure | Config and secrets management |
| New Relic | Monitoring | APM, transaction tracing, custom metrics |
| Sentry | Monitoring | Error tracking |

---

## Notable Patterns & Decisions

**Serverless-first with local dev escape hatch.** The binary detects `AWS_LAMBDA_FUNCTION_NAME` at startup and switches between Lambda handler and a plain HTTP server. This avoids a SAM/LocalStack dependency for local development.

**Handler-based integration polymorphism.** Third-party integrations are selected at runtime via a `handler` string on the `App` record (`"dutchie"`, `"treez"`, `"aeropay"`, etc.). No interface hierarchy — just `switch/if` blocks in handler files. Simple but means adding a new POS requires touching multiple handler files.

**e-com-lib as the boundary.** All domain models and storage service interfaces live in the shared `e-com-lib` module. This repo contains only business logic orchestration and GraphQL wiring. The tradeoff: tight coupling to the shared library's release cadence (the `make lib` target updates it).

**S3 as a document store.** Semi-static data (products, promotions, pages) is stored as JSON blobs in S3 rather than a relational DB. This keeps read scaling trivially cheap but makes writes/queries awkward — mutations require a full read-modify-write of the JSON file.

**Context timeouts are global.** Every operation wraps in a 5-minute `CtxTimeout`. There's no per-operation tuning.

**Concurrency for CRM fan-out.** `SubscribeToCrm` uses `sync.WaitGroup` to fan out to multiple CRM platforms concurrently. This is fire-and-forget via SNS in the main path; the WaitGroup is used in the synchronous CRM caller path.

**Known race condition in TreezPay.** Rapid duplicate checkout submissions can create multiple Treez tickets for the same external order number. Root cause is stale frontend state causing concurrent `UpdateTreezCart` calls + a `CANNOT_MODIFY_TICKET` fallback that creates a new ticket. Documented in `docs/TREEZ_DUPLICATE_TICKET_ENTITY_ID_BUG.md`. Not yet fixed as of the last commit.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Architecture | TreezPay duplicate ticket race condition — needs idempotency lock or conditional write before creating new ticket | High |
| Architecture | S3 read-modify-write for metadata is not atomic and does not scale well under concurrent writes | Med |
| Architecture | Handler dispatch is `switch/if` across large files; a small interface (`OrderHandler`, `PaymentHandler`) would make integration testing and addition of new providers cleaner | Med |
| Tech debt | `handle_order.go` is 3000+ LOC; splitting by domain sub-concern (cart calculation, POS sync, payment) would improve navigability | Med |
| Performance | No in-process cache for S3 metadata — every GraphQL request that reads products/promotions hits S3. A short TTL in-memory cache (e.g., 30s) would cut latency | Med |
| DX / tooling | No integration test environment; tests mock AWS clients but there is no end-to-end test against real Lambda/DynamoDB | Low |
| Observability | New Relic custom events are sent on every order update; no sampling — could become costly at scale | Low |
| Config | Multiple `.env.*` files committed to the repo; secrets should move fully to SSM/Secrets Manager | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-srv (commit 3099116)*
