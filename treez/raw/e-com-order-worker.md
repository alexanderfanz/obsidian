# e-com-order-worker

> AWS Lambda worker that processes order completion events from the Gap Commerce e-commerce platform. It enriches order data, upserts customer accounts, and fans out to downstream integrations (Treez, Blaze, Jane, Klaviyo, notification service) via SNS.

---

## Purpose

Handles the post-purchase fulfillment pipeline for e-commerce orders. When an order is marked complete, this worker picks up the SQS event, fetches the full order and store context, assigns an invoice number, updates the order status, and conditionally triggers each downstream integration based on per-store configuration. Notification-only (`order-kiosk-completed`) and full (`order-completed`) pipelines are handled as separate routes.

**Out of scope**: Order creation, payment processing, inventory management, and returns are all handled elsewhere.

---

## Key Entities / Domain Models

- **OrderCompleteEvent** — lightweight SQS payload carrying `account_id`, `store_id`, and `entity_id` (order ID). Resolved to a full order via DynamoDB.
- **Order** — the full order record; key fields include line items, customer info, medical ID, DOB, payment details, and status. Sourced from DynamoDB and enriched in-flight by the pipeline.
- **Account** — the customer record. Upserted on every order; aggregates lifetime spend and carries medical ID + DOB from the order.
- **Store Config** — per-store object loaded from DynamoDB/S3 (`store.json`). Controls which integrations are active, queue URLs, table names, and approval flags.
- **App Config** — per-store integration flags loaded from S3 (`apps.json`). Drives conditional routing — e.g., Klaviyo enabled, Blaze enabled, Jane enabled.
- **Invoice** — assigned during processing; counter incremented atomically in DynamoDB per store.

---

## API Surface

This service has no inbound HTTP API. Its interface is entirely event-driven.

**Consumed (SQS)**:

| Event Type | Description |
|------------|-------------|
| `order/completed` | Full pipeline: enrich, notify, fan-out to all configured integrations |
| `order/kiosk-completed` | Partial pipeline: enrich only — no notification or integration fan-out |

**Published (SNS)**:

| Event Name | Description |
|------------|-------------|
| `place_order_klaviyo` | Triggers Klaviyo email marketing flow |
| `process_blaze` | Triggers Blaze fulfillment processing |
| `process_jane` | Triggers Jane POS/inventory processing |
| `process_treez` | Triggers Treez compliance/POS processing |
| `order_status_changed_notify` | Triggers user-facing order status notification |

---

## Data Layer

- **DynamoDB** — primary store for orders, accounts, stores, and invoice counters. Multi-tenant: table names are resolved per-store from config. Keys use prefixed composite patterns (`blaze#`, `j#`, `c#`).
- **S3** — stores per-store app config (`apps.json`) and store metadata (`store.json`). Loaded at startup and cached in the Lambda execution context.
- **SSM Parameter Store** — Lambda environment variables are resolved from SSM at cold start (via `srv-utils`).
- No relational DB; no in-process cache beyond the Lambda lifecycle.

---

## Core Business Logic

**Invoice assignment**: Each store has an atomic counter in DynamoDB. On every `order/completed` event, the counter is incremented and the resulting number becomes the invoice number. This is the only strict ordering guarantee in the pipeline.

**Conditional integration routing**: Each downstream integration (Blaze, Jane, Treez, Klaviyo) is only triggered if it's enabled in the store's `apps.json`. The pipeline checks these flags at runtime — no static routing. Treez additionally checks an `automatic_approval` flag: if enabled, the `process_treez` event is published; otherwise, manual approval is assumed.

**Account upsert**: On every order, the customer account is created or updated. Lifetime spend is aggregated. Medical ID and DOB are sourced from the order and written onto the account record — these are cannabis compliance fields.

**Kiosk vs. standard order**: The `order/kiosk-completed` event type runs a shorter processor chain — it does not publish notification or integration events. This allows kiosk-originated orders to go through fulfillment enrichment without triggering duplicate downstream flows.

**Status progression**: Order status is set to `PENDING` at the start of the pipeline. Activity is logged as `UpdateOrderStatus by API`.

---

## Cross-Repo Connections

- **Calls**: `github.com/gap-commerce/e-com-lib` — all shared domain models, DynamoDB service interfaces, S3/SNS/SQS wrappers, and the `WorkerManager` pipeline framework. This is the heaviest dependency.
- **Called by**: The upstream order service publishes to the SQS queue this worker consumes. Exact repo unknown from this codebase.
- **Events published to**: `e-com-notification-worker` (via `order_status_changed_notify`), Klaviyo worker (via `place_order_klaviyo`), Blaze worker (via `process_blaze`), Jane worker (via `process_jane`), Treez worker (via `process_treez`) — all via shared SNS topic.
- **Shared types / contracts**: Order, Account, Store, and App config types are defined in `e-com-lib` and shared across all workers.

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS SQS | Infrastructure | Inbound order completion event queue |
| AWS SNS | Infrastructure | Fan-out to downstream worker queues |
| AWS DynamoDB | Infrastructure | Order, account, store, invoice data store |
| AWS S3 | Storage | Store and app config files |
| AWS SSM | Infrastructure | Runtime config/secrets resolution |
| New Relic | Monitoring | Custom analytics events per order (amount, store) |
| Sentry | Monitoring | Exception capture and panic recovery |
| Klaviyo | Email | Order confirmation / marketing email trigger |
| Blaze | Other | Cannabis fulfillment system integration |
| Jane | Other | Cannabis POS / inventory system integration |
| Treez | Other | Cannabis compliance / POS system integration |
| LisTrack | Email | Transactional email (referenced in pipeline) |

---

## Notable Patterns & Decisions

**Pipeline pattern via `WorkerManager`**: Processing steps are composed as an ordered list of `Processor` implementations. Each processor is a discrete unit of work; failure in one can short-circuit or be ignored depending on criticality. This makes it easy to add/remove integration steps without touching control flow.

**Lazy processor initialization**: Processor instances are created only when the pipeline runs (`manager.go`), not at Lambda cold start. This avoids unnecessary initialization for integration paths that won't be triggered.

**Dual-route handler**: Rather than branching inside a single processor, the two order types (`order-completed`, `order-kiosk-completed`) are registered as separate handler routes in the `WorkerManager`, each with their own processor slice. Clean separation without conditionals in the pipeline itself.

**Multi-tenant by design**: There is no single-tenant assumption anywhere. Every DynamoDB table name, S3 key, and integration flag is resolved per `account_id` + `store_id` at runtime. The Lambda handles orders for any tenant.

**Non-critical failure handling**: Some processors return `nil` on integration-specific failures (e.g., if Klaviyo is not configured). This means the pipeline can succeed even if an optional integration step is skipped — by design, not by accident.

**Panic recovery with Sentry**: The Lambda handler wraps execution in a `defer/recover` block that reports panics to Sentry before re-panicking. This ensures no silent failures in production.

**ARM64 Lambda target**: The build explicitly targets `GOARCH=arm64 GOOS=linux` (Graviton). Cost and performance optimization over x86.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Testing | Most test files are skipped (`t.Skip()`). The upsert_account test is the only one with real assertions. Pipeline is effectively untested in CI. | High |
| Error handling | Some processors silently swallow errors by returning `nil` — makes it hard to distinguish "integration disabled" from "integration failed". | Med |
| Config | Environment config is baked into the deployment zip as a flat `.env` file. No runtime secret rotation without a redeploy. | Med |
| Architecture | The `manager.go` file directly instantiates all processors and knows all integration types — adding a new integration requires editing this file. A registry pattern would be cleaner. | Low |
| Observability | New Relic events capture order amount and store but not integration outcomes. Unclear if Blaze/Jane/Treez publish failures are visible anywhere. | Med |
| Region | `us-west-1` is hardcoded in deploy scripts. China region has a separate config but the deploy tooling doesn't reflect this cleanly. | Low |
| DX / tooling | `make deploy` requires manual ENV and AWS_PROFILE env vars. No guardrail against deploying dev config to prod. | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-order-worker (master @ c55ec05)*
