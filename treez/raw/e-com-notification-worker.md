# e-com-notification-worker

> AWS Lambda service that consumes SQS notification events and sends templated HTML emails via SES for the Gap Commerce / Emberz e-commerce platform. Handles order confirmations, delivery updates, status changes, out-of-stock alerts, and customer contact-form forwarding across multiple tenant accounts and stores.

---

## Purpose

Provides the outbound email notification layer for the e-commerce platform. When order lifecycle events occur (placed, confirmed, out-of-stock, status changed, delivered), other services publish to an SQS queue; this worker processes those events and delivers personalized HTML emails to customers and merchants via AWS SES.

**In scope:** Email notification dispatch for order events and customer contact forms. Per-store template configuration. Multi-tenant account/store isolation.

**Out of scope:** Push notifications, SMS, webhook delivery, order state management, payment processing.

---

## Key Entities / Domain Models

- **NotificationEvent** — Trigger payload for order-related notifications: `account_id`, `store_id`, `entity_id` (order ID). Thin — all order detail is fetched on demand.
- **ContactBusinessEvent** — Customer contact form submission: `account_id`, `store_id`, `name`, `email`, `message`. Forwarded to the business owner's configured email address.
- **Params** — Rich template context assembled per-notification: store branding, order details, delivery address, line items with pricing, totals. Passed to `html/template` for email rendering.
- **EmailAttributes** — Outbound email envelope: `to[]`, `source`, `subject`, `content` (HTML string). Passed directly to SES.
- **Store** (from e-com-lib) — Merchant config retrieved from S3; gates sending via `EmailNotificationActive` flag; provides `DynamoOrderTableName`, `FromTxEmail`, and branding fields.
- **NotificationSetting** (from e-com-lib) — Per-store, per-event email template retrieved from S3; provides subject line and HTML body template string.

---

## API Surface

This service has no HTTP API. Its interface is the SQS message contract.

**Inbound SQS message envelope:**
```json
{
  "Message": "<JSON-encoded event body>",
  "MessageAttributes": {
    "event_type": { "Value": "order/confirm_notify" }
  }
}
```

**Supported event types:**

| event_type | Processor | Action |
|---|---|---|
| `order/confirm_notify` | ConfirmOrder | Order confirmation email to customer |
| `order/delivered_notify` | Delivery | Delivery notification email to customer |
| `order/out_of_stock_notify` | OrderOutOfStock | Out-of-stock alert email |
| `order/status_changed` | StatusChanged | Status transition email (template selected by new status) |
| `contact/business_owner` | ContactBusinessOwner | Forwards customer contact form to merchant |

---

## Data Layer

- **DynamoDB** — Order records. Table name is store-specific, read from the store's S3 config (`store.DynamoOrderTableName`). Queried by order ID to assemble the email template context.
- **S3** — Two config objects per store, keyed as `{account_id}-{store_id}-{key_name}`:
  - `store.json` — Store metadata, branding, flags, email settings.
  - `notification_settings.json` — Per-event HTML email templates and subject lines.
- No relational database. No caching layer (S3/DynamoDB calls happen per Lambda invocation; warm-start reuse is incidental).

---

## Core Business Logic

**Event routing via predicates.** The `worker.Manager` from e-com-lib matches each incoming SQS message to a processor using predicate functions (`WhenConfirmOrder`, `WhenDelivery`, etc.) that inspect `MessageType`. This is the only routing mechanism — there is no central switch statement.

**Per-store notification gating.** Before sending any email, processors check `store.EmailNotificationActive`. If false, the event is silently dropped (no email sent, message still deleted from SQS).

**Dynamic template selection for `status_changed`.** Unlike other processors which use a fixed template key, `StatusChanged` selects the template based on the order's new status value. The mapping between status strings and template keys lives in the processor.

**Multi-tenant key isolation.** All S3 lookups use a composite key `{accountID}-{storeID}-{keyName}`, ensuring stores never see each other's config or templates.

**`FromTxEmail` override.** If a store has a `FromTxEmail` configured, outbound SES emails use that as the sender; otherwise the global `transactional_email_source` env var is used.

**Template rendering.** HTML email bodies are Go `html/template` strings stored in S3. Template context (`Params`) is assembled from S3 store config + DynamoDB order data. Two helper funcs are registered: `inc` (integer increment for loop indices) and `formatPrice` (decimal formatting for currency).

**Idempotency / retry behavior.** SQS message deletion happens after successful processing. If the Lambda crashes mid-flight, the message becomes visible again and is retried. No deduplication key is stored; downstream sends (SES) are assumed idempotent enough for the use case.

---

## Cross-Repo Connections

- **Calls:**
  - `github.com/gap-commerce/e-com-lib` — Core shared library: Order, Store, NotificationSetting service interfaces; `worker.Manager` pipeline framework; shared models and mocks.
  - AWS SES — Email delivery.
  - AWS S3 — Store config and notification templates.
  - AWS DynamoDB — Order data.
  - AWS SQS — Event source (consumed, not published to).

- **Called by:**
  - Any service that publishes to the SQS notification queue. Likely: order management service, contact-form handler.

- **Events subscribed to:** `order/confirm_notify`, `order/delivered_notify`, `order/out_of_stock_notify`, `order/status_changed`, `contact/business_owner` — all via SQS.

- **Events published:** None. This is a terminal consumer.

- **Shared types / contracts:** `e-com-lib` defines the SQS message envelope schema, `worker.Processor` interface, and domain model types shared across gap-commerce services.

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS SQS | Infrastructure | Event source — inbound notification messages |
| AWS SES | Email | Transactional email delivery to customers and merchants |
| AWS S3 | Storage | Store config and per-store email template storage |
| AWS DynamoDB | Infrastructure | Order data retrieval for template context |
| Sentry | Monitoring | Runtime error tracking and panic capture |
| New Relic | Monitoring | Lambda performance monitoring and metrics |

---

## Notable Patterns & Decisions

**Worker pipeline from shared lib.** The `worker.Manager` (e-com-lib) structures every Lambda invocation as: Parse → Predicate match → Map JSON → Process → Delete. All five processors and the deleter plug into this framework. Changing the pipeline order or adding a new event type requires no changes to the routing core.

**Predicate-based routing.** Each processor registers a predicate on `MessageType`. This avoids a central dispatcher and makes it trivial to add new event types without touching existing code.

**Generic mapper.** `MapToModel[T]()` is a generic function that unmarshals the SQS message body into any target type. Reduces per-processor boilerplate.

**Lazy processor initialization.** `App` initializes processors on first use and caches them. Reduces cold-start overhead if a given event type hasn't been seen yet in a Lambda instance.

**External templates.** Email HTML is not embedded in the binary — it lives in S3 per store. This means template changes can be deployed without a Lambda code release, but also means a missing/malformed S3 object will cause a runtime failure.

**ARM64 build target.** Binary compiled for `GOARCH=arm64` (AWS Graviton2). Lower Lambda cost and better throughput for Go workloads.

**Bundled `.env` in zip.** Each environment's config is baked into the deployment artifact (`cmd/main.zip` contains `bootstrap` + `.env`). This means environment-specific builds, not a single artifact promoted across environments.

**No message-level error isolation.** If any processor panics, Sentry captures it, but the SQS batch behavior (partial failures vs. full failure) depends on the Lambda event source mapping config, which is managed outside this repo (likely CDK). Worth verifying that partial batch failure reporting is enabled so a single bad message doesn't block the whole batch.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Reliability | No partial batch failure reporting in code — if one SQS message fails, unclear whether other messages in the batch are reprocessed. Confirm Lambda event source mapping has `ReportBatchItemFailures` enabled. | High |
| Config | `.env` files bundled into the artifact means per-environment builds rather than a single promotable artifact. Moving config to Lambda env vars or SSM Parameter Store would enable true build-once / deploy-anywhere. | Med |
| Templates | HTML email templates stored in S3 are not versioned or validated at deploy time. A bad template only fails at runtime per-invocation. A template validation step in CI would catch issues earlier. | Med |
| Observability | SQS message deletion happens even if email sending fails silently (no SES error surface observed in processors). Confirm processors return errors on SES failure so the worker framework can handle retry vs. discard correctly. | Med |
| Testing | Integration tests exist but are disabled by default (file rename required). Making these runnable via a Makefile target against a local/dev environment would improve confidence. | Low |
| DX | `make deploy` uploads directly to a Lambda function, bypassing the S3 artifact pipeline used in CI. This creates a path for manual hotfixes that skip the release process. Consider removing or gating this target. | Low |
| Architecture | `FromTxEmail` fallback to global `transactional_email_source` env var is implicit. A store with no `FromTxEmail` set silently uses the global sender, which may not match the store's brand. Could be surfaced as a warning in logs. | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-notification-worker @ 4322e35*
