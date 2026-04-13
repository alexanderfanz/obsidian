# e-com-app-worker

> AWS Lambda worker that processes asynchronous SQS events for the Gap Commerce e-commerce platform. Handles order placement across multiple cannabis POS/ERP providers (Treez, Blaze, Jane), CRM subscriptions, loyalty discount redemptions, and new store initialization.

---

## Purpose

Decouples long-running, side-effect-heavy operations from the synchronous request path. When a customer places an order or subscribes to a CRM, the API enqueues an event; this Lambda picks it up and performs the actual work: submitting the order to the downstream POS/ERP, uploading compliance documents (verification photos), redeeming loyalty discounts, and publishing completion events.

**In scope**: Order submission to Treez / Blaze / Jane, CRM subscription (Klaviyo, Omnisend, AlpineIQ, Listrack), loyalty discount redemption (AlpineIQ, Sticky), store scaffold creation.

**Out of scope**: Payment processing, storefront rendering, cart/checkout logic, customer authentication.

---

## Key Entities / Domain Models

- **OrderEvent** (`internal/model/models.go:8`) — Minimal event payload routing an order to the right processor. Fields: `account_id`, `store_id`, `entity_id` (DynamoDB key of the full order record).

- **SubscribeCRM** (`internal/model/models.go:28`) — Wraps `e-com-lib`'s `SubscribeCRMInput` (customer PII, list preferences) with `account_id` and `store_id` for multi-tenant routing.

- **CreateStoreEvent** (`internal/model/models.go:34`) — Payload for provisioning a new store: Cognito user pool IDs, tier plan, Vercel project ID, Prismic API endpoint, deployment URL.

- **BlazeOrder / JaneOrder** (`internal/model/models.go:14,21`) — Thin DynamoDB records that map an internal `entity_id` to the provider-assigned `order_id`, plus a composite `key` field.

- **Order** (from `e-com-lib`) — Full order record stored in DynamoDB. Retrieved by processors using `entity_id`. Contains line items, customer info, payment method, and status.

- **AppConfig / StoreConfig** (from `e-com-lib`, stored in S3) — Per-store JSON blobs holding provider credentials (Treez API keys, Klaviyo API keys, etc.), feature flags, and table names.

---

## API Surface

This is a consumer, not a provider. It exposes no HTTP routes. The contract is the SQS message schema: each message must have an `event_type` message attribute and a JSON body matching the expected model.

| Event Type | Message Body Model | Processors |
|---|---|---|
| `subscribe-to-klaviyo` | `SubscribeCRM` | SubscribeKlaviyo |
| `subscribe-to-omnisend` | `SubscribeCRM` | SubscribeOmnisend |
| `subscribe-to-alpine-iq` | `SubscribeCRM` | SubscribeAlpineIQ |
| `subscribe-to-listrack` | `SubscribeCRM` | SubscribeListrack |
| `place-order-klaviyo` | `OrderEvent` | PlaceOrderKlaviyo |
| `place-order-blaze` | `OrderEvent` | PlaceOrderBlaze |
| `place-order-jane` | `OrderEvent` | PlaceOrderJane |
| `place-order-treez` | `OrderEvent` | PlaceOrderTreez → AlpineIQDiscount → StickyDiscount |
| `create-store` | `CreateStoreEvent` | CreateStore |

---

## Data Layer

- **DynamoDB** — Source of truth for orders and accounts. Processors read the full order record by `entity_id` and write back provider order IDs (e.g. Blaze/Jane order ID mappings). Table names are not hardcoded; they are resolved at runtime from the per-store S3 config.

- **S3** — Stores two categories of data:
  - *Config blobs*: `apps.json` and `store.json` per account/store, keyed via `util.GetKey(accountID, storeID, keyName)`. All credentials and feature flags live here.
  - *Account photos*: Driver's license and medical ID images uploaded by customers, fetched during Treez order submission for age-verification compliance.

- **SQS** — Inbound event queue (`aws_sqs_app_url`). Messages are deleted after successful processing via `ReceiptHandle`. Failure leaves the message on the queue for retry/DLQ handling.

- **SNS** — Outbound completion events published after order placement (e.g. to trigger downstream notifications). Topic ARN configured via `aws_sns_topic`.

- **No dedicated caching layer.** S3 config reads are per-invocation; Lambda container reuse provides implicit short-lived caching of processor instances (lazy-initialized singletons on `App`).

---

## Core Business Logic

**Multi-processor pipeline for Treez orders**: Unlike other providers, `place-order-treez` chains three processors sequentially: `PlaceOrderTreez` → `AlpineIQDiscount` → `StickyDiscount`. Each subsequent processor only runs if the prior one succeeded, and discount processors are designed to be error-tolerant (non-critical failures are logged but don't fail the order).

**Verification photo upload (Treez)**: Before finalizing a Treez ticket, the processor builds an upload plan based on whether the customer is new or returning and which document types are present in S3. It fetches raw image bytes from S3 and uploads them directly to Treez's document API. Logic in `internal/processor/place_order_treez.go` with thorough table-driven tests covering the plan-building step.

**Payment method detection (Treez)**: Treez requires distinguishing credit vs. debit cards. The processor inspects the order's payment data to select the correct Treez payment type — tested explicitly in `TestIsDebitCard` variants.

**Store scaffold creation**: `create-store` writes multiple default S3 config keys (products, pages, navigation, categories, promotions, tags, brands, vendors, landing pages, taxes, best-selling products) seeding a brand-new store with empty-but-valid configuration. Uses a large set of S3 key configs passed into the processor at construction.

**CRM parallel list subscription (Klaviyo)**: A single `SubscribeCRM` event may target multiple Klaviyo lists. The processor fans out with `errgroup.WithContext()` to subscribe concurrently and collects all errors.

**Multi-tenant routing**: Every event carries `account_id` + `store_id`. Processors derive S3 keys using `util.GetKey(accountID, storeID, keyName)` and load per-store credentials at runtime. There is no shared credential; each store is fully isolated.

---

## Cross-Repo Connections

- **Calls**:
  - `github.com/gap-commerce/e-com-lib` — Shared service interfaces (Order, Store, Account, App, AccountFile), domain models, and the `worker` package that provides the pipeline DSL. This is the heaviest dependency; most business logic lives here or is invoked through its interfaces.
  - `github.com/gap-commerce/srv-utils` — Config loading from SSM Parameter Store at Lambda cold-start.
  - Treez, Blaze, Jane, Klaviyo, Omnisend, AlpineIQ, Listrack, Sticky HTTP APIs (via `e-com-lib` service wrappers or direct resty calls).

- **Called by**: The API layer (unknown repo) enqueues SQS messages that this worker consumes. The `create-store` event is likely triggered by an admin/onboarding flow.

- **Shared types / contracts**: `e-com-lib/pkg/model` defines `SubscribeCRMInput`, order models, `TierPlanType`, and the `worker.Processor` interface. Changes to those types require coordinated updates here.

- **Events published**: SNS topic (`aws_sns_topic`) receives order completion events after successful placement. Downstream subscribers (notification service, analytics, etc.) are in other repos.

---

## Third-Party Services

| Service | Category | What it's used for |
|---|---|---|
| Treez | Other (Cannabis ERP) | Submit and manage cannabis retail orders; upload age-verification docs |
| Blaze | Other (Cannabis POS) | Submit orders to Blaze POS system |
| Jane | Other (Cannabis POS) | Submit orders to Jane POS system |
| Klaviyo | Email | CRM list subscription; order event tracking |
| Omnisend | Email | CRM list subscription |
| AlpineIQ | Other (Loyalty/CRM) | Customer subscription + loyalty discount redemption |
| Listrack | Other (Loyalty) | Customer loyalty list subscription |
| Sticky | Other (Loyalty) | Loyalty reward/discount redemption at order time |
| AWS SQS | Infrastructure | Inbound event queue |
| AWS DynamoDB | Infrastructure | Order and account storage |
| AWS S3 | Storage | App/store config blobs; customer verification photos |
| AWS SNS | Infrastructure | Outbound order completion event bus |
| Sentry | Monitoring | Exception capture and panic recovery |
| NewRelic | Monitoring | APM, distributed tracing, Lambda performance metrics |

---

## Notable Patterns & Decisions

**Chain-of-responsibility pipeline DSL** (`e-com-lib/pkg/worker`): The entire event routing and processing pipeline is declared in ~60 lines of fluent builder code in `manager.go`. Parser → Predicate → Mapper → Processor(s) → Deleter. Adding a new event type is additive: one `When(...)` block with a predicate, mapper, and processor. This is clean but the DSL itself lives in `e-com-lib`, so debugging pipeline behavior requires reading that repo.

**Lazy-initialized processor singletons**: Each processor is constructed once per Lambda container lifetime via nil-guard factory methods on `App`. This avoids repeated allocation across SQS batch messages within the same invocation while keeping construction explicit.

**Credentials live in S3, not SSM**: Per-store API keys (Klaviyo, Treez, etc.) are stored in S3 JSON blobs, not in SSM. Lambda environment variables only hold AWS infrastructure config. This means rotating a third-party credential requires updating S3 objects, not Lambda config.

**ARM64 Lambda target**: The Makefile hard-codes `GOARCH=arm64 GOOS=linux` for Graviton2 cost savings. Developers on x86 Macs must cross-compile; this is handled automatically by `make build`.

**Error-tolerant discount processors**: `AlpineIQDiscount` and `StickyDiscount` are chained after `PlaceOrderTreez` but are designed not to fail the overall message processing if they error. The order has already been placed; discount failure is logged but the SQS message is still deleted.

**No local config file at runtime**: The `.env.*` files are bundled into the deployment zip but only used for env var seeding. Runtime secrets come from SSM at cold-start via `srv-utils`.

---

## Potential Improvements

| Area | Observation | Priority |
|---|---|---|
| Tech debt | `place-order-treez` processor is large (~600+ lines in test alone); verification photo logic could be extracted to its own file/package | Med |
| Architecture | Discount processing is chained inline on the Treez event type only; if other providers need loyalty integration the current structure would require duplicating the chain | Med |
| Observability | NewRelic and Sentry are both initialized but it's unclear if they are fully integrated (traces correlated). Redundant monitoring adds cold-start overhead | Low |
| DX / tooling | Many integration tests are unconditionally skipped (`t.Skip()`) — these should be gated on a build tag or env var rather than a hardcoded skip to avoid silent gaps in CI coverage | Med |
| Architecture | No dead-letter queue (DLQ) handling is visible in this repo; repeated processing failures silently retry. DLQ monitoring and alerting belong in infrastructure config but should be documented | Low |
| Config | The `CreateStoreProcessor` takes 14 S3 key arguments; this constructor is fragile and hard to extend. A config struct would be safer | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-app-worker @ bfbb85c*
