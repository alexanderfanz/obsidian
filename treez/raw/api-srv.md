# api-srv

> Webhook orchestration service that acts as the integration backbone between external cannabis POS / ERP systems (Blaze, Jane, Treez, OnFleet) and the internal GapCommerce e-commerce platform. It receives inbound webhooks, normalises the data, updates order and inventory state in DynamoDB/S3, and fans out notifications (SMS via Alpine IQ, email via LisTrack, SNS events for downstream workers).

---

## Purpose

**Business problem**: Multiple point-of-sale and delivery systems need to push status changes back into the GapCommerce order lifecycle in real time. Each system has its own payload shape, auth scheme, and status vocabulary.

**Responsible for**:
- Receiving and authenticating webhooks from Blaze, Jane, Treez, and OnFleet
- Mapping provider-specific order/delivery statuses to internal `model.OrderStatusType`
- Updating orders in DynamoDB and product catalogs in S3
- Triggering downstream effects: Alpine IQ SMS, LisTrack transactional email, SNS order-changed events, Cloudflare cache purge, Vercel rebuild

**Out of scope**:
- Customer-facing APIs or storefronts (handled by other repos)
- Payment processing
- The internal order management system itself (reads/writes via the shared `e-com-lib` interfaces)

---

## Key Entities / Domain Models

- **`BlazeOrder`** — DynamoDB record mapping a Blaze cart ID to an internal order ID. Key prefix `blaze#`. Fields: `EntityID`, `Key`, `OrderID`, `CreatedAt`.

- **`JaneOrder`** — Same pattern for Jane POS. Key prefix `j#`. Includes `CartID`.

- **`BaseRequest`** — Common base extracted from every inbound webhook: `AccountID`, `StoreID` (from query params). All provider requests embed this.

- **`TreezData`** — Order payload from Treez: `OrderNumber`, `ExternalOrderNumber`, `OrderStatus`, `TicketNote`, `RevenueSource`.

- **`TreezOrderRequestData`** — Wraps `TreezData` with a custom `UnmarshalJSON` that normalises `event_type` from either a string or an array (test payloads send an array). Exposes `EventTypes []TreezEventType`.

- **`OnFleetTask`** — Delivery webhook payload with `taskId` and nested task metadata. The request builder enriches this with the full `Order` and `Store` fetched from DynamoDB during request construction.

- **`TreezOrderStatus`** — Enum: `VERIFICATION_PENDING`, `AWAITING_PROCESSING`, `IN_PROCESS`, `PACKED_READY`, `OUT_FOR_DELIVERY`, `COMPLETED`, `CANCELED`.

- **`JaneOrderStatus`** — Enum with 11 values including `dismissed`, `cancelled`, `finished`, `ready_for_pickup`, `delivery_started`, `delivery_arrived`, etc.

---

## API Surface

All routes require `account_id` and `store_id` as query parameters to identify the merchant tenant and store.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/healthcheck` | Returns service status, version, build date |
| POST | `/blaze/product-updated` | Sync Blaze inventory (products, categories, brands, vendors) to S3 |
| POST | `/blaze/order-updated` | Process Blaze order status change |
| POST | `/jane/order-updated/` | Process Jane POS order status change (Bearer token auth) |
| POST | `/listrack/jane-products/` | Sync products from Jane to LisTrack (x-api-key header auth) |
| POST | `/treez/order-updated` | Process Treez cannabis order status change (event type + revenue source filtering) |
| POST | `/treez/product-updated` | Sync a single Treez product to the store catalog in S3 |
| GET/POST | `/onfleet/task-started` | Delivery driver started task |
| GET/POST | `/onfleet/task-arriving` | Delivery driver arriving |
| GET/POST | `/onfleet/task-completed` | Delivery task completed |
| GET/POST | `/onfleet/task-failed` | Delivery task failed or cancelled |
| GET/POST | `/onfleet/task-delayed` | Delivery task delayed |
| GET/POST | `/onfleet/task-assigned` | Delivery task assigned to driver |

Swagger docs served at `/api-docs/` in non-prod environments.

---

## Data Layer

**AWS DynamoDB**
- Stores `BlazeOrder` and `JaneOrder` records (cart-ID → internal-order-ID mapping)
- Table name is stored per-store in the store metadata (`Store.DynamoOrderTableName`) — resolved at runtime
- Composite key: `(EntityID, Key)` where `Key` is a provider prefix (`blaze#`, `j#`)
- Access via a generic `dynamo.Repository[T]` with a single `Get()` method

**AWS S3**
- Primary store for product catalogs, store metadata, and third-party app credentials
- Key structure: `{accountID}/{storeID}/{resource}` (e.g. `account123/store456/apps.json`)
- Resources: `store.json`, `products.json`, `apps.json`, `promotions.json`, `categories.json`, `brands.json`, `vendors.json`, `providers_products/provider-{storeID}.json`
- App credentials (Blaze, Jane, Treez, Alpine IQ) are stored in `apps.json` and loaded per request — allows credential rotation without redeployment

**Caching**
- Cloudflare CDN cache is purged asynchronously after a Blaze inventory sync
- No in-process caching; HTTP clients use go-resty connection pooling

---

## Core Business Logic

**Treez order processing** (most complex):
1. Filter by revenue source — only processes orders where `RevenueSource == "e-commerce"` (or treez-specific constant). Others are silently skipped with 200 OK.
2. Filter by event type — `EventTypes` must contain `TICKET_STATUS`. Supports both string and array format for test payloads.
3. Skip abandoned cart state.
4. Handle "DeclinedFromAdmin" flag resets.
5. Map Treez status → internal status + derive notification flags (SMS, email, etc.).
6. Append activity log entry (`user = "API"`, UTC timestamp).
7. Dispatch Alpine IQ SMS for status changes (confirmation, ready, cancelled, completed). For declined orders, also revert any discount redemption.
8. Send LisTrack transactional email for delivery orders.
9. Publish `OrderStatusChangedNotifyEventType` to SNS for downstream notification workers.

**OnFleet delivery flow**:
- Each task event maps to a specific status transition and triggers a combination of: DynamoDB order update → LisTrack email → Treez ticket sync
- `TaskFailed`: Reverts order to Declined, removes OnFleet reference, removes Treez ticket entirely, sends rejection email
- `TaskCompleted`: Sets DeliveryFinished, updates Treez ticket with payment details, sends completion email

**Jane order processing**:
- Maps 11 Jane statuses to internal statuses, distinguishing delivery vs. pickup paths (affects LisTrack event type)
- Fetches delivery timestamps from Jane API for enrichment before emailing

**Blaze inventory sync**:
- Retrieves credentials from S3 (`apps.json`)
- Syncs all inventory types (products, categories, brands, vendors) via the shared `inventory_sync` library
- Fires Cloudflare cache purge + Vercel rebuild in a goroutine (non-blocking, failures are non-fatal)

**Treez product sync**:
- Fetches product from Treez API, maps lab results (CBD, CBDA, THC, THCA as percentages or amounts), maps variants by weight/unit (half_gram → ounce), formats price as cents (`price * 100`)
- Stores result as `providers_products/provider-{storeID}.json` in S3

---

## Cross-Repo Connections

**Calls**:
- `github.com/gap-commerce/e-com-lib` — shared domain models (`model.Order`, `model.Store`, `model.Product`, etc.) and provider service interfaces (Blaze, Jane, Treez, Alpine IQ, OnFleet, LisTrack). Pinned at v1.163.7 with a local `replace` path for dev.
- `github.com/gap-commerce/srv-utils` — AWS/SSM config loader
- `github.com/gap-commerce/glog` — structured JSON logger

**Called by**:
- Blaze, Jane, Treez, OnFleet platforms (inbound webhooks)
- LisTrack cron job (calls `/listrack/jane-products/` with `x-api-key` auth)

**Events published**:
- AWS SNS topic (`aws_sns_topic` env var): `OrderStatusChangedNotifyEventType` with `{AccountID, StoreID, EntityID}` payload. Downstream workers consume this for additional notification logic.

**Shared types / contracts**:
- Order, Store, Account, Product models from `e-com-lib` are the shared schema between this repo and the rest of the platform.

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| Blaze ERP | Other (Cannabis POS) | Inventory sync, order status webhooks |
| Jane POS | Other (Cannabis POS) | Order status webhooks, cart retrieval for delivery timestamps |
| Treez ERP | Other (Cannabis POS) | Order status webhooks, product sync, ticket management |
| OnFleet | Shipping | Delivery task event webhooks |
| Alpine IQ | SMS | Order status SMS notifications, discount redemption/reversion |
| LisTrack | Email | Transactional order email (new, approved, on-route, completed, rejected) |
| AWS DynamoDB | Infrastructure | Order entity mapping storage |
| AWS S3 | Storage | Product catalogs, store metadata, app credentials |
| AWS SNS | Infrastructure | Publishing order status change events to downstream consumers |
| AWS Lambda | Infrastructure | Primary serverless runtime (ARM64, us-west-1) |
| Cloudflare | Infrastructure | CDN cache purging after inventory sync |
| Vercel | Infrastructure | Triggering static site rebuilds after inventory sync |
| Sentry | Monitoring | Error tracking with business context tags (account_id, store_id, order_id) |
| New Relic | Monitoring | APM tracing via middleware |

---

## Notable Patterns & Decisions

**Generic request builders**: Every provider has a `*RequestBuilder` that validates auth, extracts query params, decodes the body, and optionally enriches the request (e.g. `OnFleetRequestBuilder` fetches the full Order and Store from DynamoDB as part of `Build()`). Handlers receive a typed, already-validated request object — no raw `*http.Request` parsing in business logic.

**Multi-tenancy via query params**: Every request carries `account_id` and `store_id`. S3 keys and DynamoDB table names are resolved from the store metadata at runtime, so a single deployment serves all merchants.

**App credentials in S3**: Third-party API keys (Blaze, Treez, Alpine IQ, etc.) live in `apps.json` in S3, resolved per request. This means credentials can be rotated or added for new stores without redeploying the Lambda.

**Lambda + local fallback**: `main.go` detects whether `AWS_REGION` is set. If yes, starts a Lambda handler via `gorillamux` proxy adapter. If no, starts a plain HTTP server on port 8080. Same binary, two runtimes.

**Polymorphic `event_type`**: Treez test payloads send `event_type` as a JSON array; production payloads send a string. A custom `UnmarshalJSON` on `TreezOrderRequestData` tries string first, then array, normalising both into `[]TreezEventType`. This avoids a breaking change in the handler.

**Non-blocking side effects**: Cloudflare cache purge and Vercel build trigger run in goroutines after a successful Blaze sync. Failures are logged but don't affect the HTTP response.

**VCR for external API tests**: Uses `go-vcr` to record and replay HTTP interactions with external APIs, keeping tests deterministic without mocking at the interface level.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Dead code | `TreezOrderRequestData.EventType` (singular) is set in `UnmarshalJSON` and the handler's `Build()` result but never read — only `EventTypes` is used. Can be removed. | Low |
| Swagger drift | The `@Param` annotation on `OrderUpdated` still references `treezRequest[treezDataRequest]` but the generated docs show `treezTestRequest`. Regenerating docs would silently revert the schema. The annotation needs to be updated to match. | Med |
| Credential loading latency | `apps.json` is fetched from S3 on every request for Treez product sync. No caching means one extra S3 round-trip per webhook. A short TTL cache would reduce latency without risk. | Med |
| OnFleet error handling | Several OnFleet service methods continue processing subsequent steps after a failure (e.g. Treez ticket update failure is captured in Sentry but execution continues). The failure semantics aren't always explicit. | Med |
| Hardcoded region | AWS region is hardcoded to `us-west-1` in the wiring layer. If the service ever needs multi-region deployment, this is a refactor point. | Low |
| Test coverage gaps | OnFleet and Listrack handlers have thinner test coverage than Blaze/Jane/Treez. The VCR cassettes exist but test case breadth is narrower. | Low |
| Vercel timeout | Build trigger timeout is hardcoded to 3 seconds in the service. Should be configurable. | Low |

---

*Generated: 2026-04-13 — Source: github.com/gap-commerce/api-srv @ 649bee6*
