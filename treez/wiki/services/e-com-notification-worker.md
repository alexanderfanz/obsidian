# e-com-notification-worker

Processes SQS notification events and sends templated HTML emails via AWS SES.

**Language:** Go | **Runtime:** AWS Lambda (ARM64) | **Trigger:** SQS

## Event types handled

| Event | Action |
|-------|--------|
| `order/confirm_notify` | Order confirmation email to customer |
| `order/delivered_notify` | Delivery notification email |
| `order/out_of_stock_notify` | Out-of-stock alert |
| `order/status_changed` | Status transition email (template selected by new status) |
| `contact/business_owner` | Forwards customer contact form to merchant |

## Key behaviors

- **Predicate-based routing** via [[Worker Pipeline]] from [[e-com-lib]]. Each processor registers a predicate on `MessageType`.
- **Per-store notification gating** — checks `store.EmailNotificationActive` before sending. If false, event is silently dropped.
- **External templates** — HTML email bodies stored in S3 per store (not embedded in binary). Template changes deploy without a Lambda release.
- **Multi-tenant key isolation** — S3 lookups use `{accountID}-{storeID}-{keyName}`.

## Dependencies

- [[e-com-lib]] — worker framework, domain models, service interfaces
- AWS SES, S3 (templates + store config), DynamoDB (order data), SQS (event source)

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Reliability | No partial batch failure reporting — unclear if `ReportBatchItemFailures` is enabled | High |
| Config | `.env` bundled in artifact — per-environment builds, not promotable | Med |
| Templates | S3 templates not validated at deploy time | Med |

*Source: [[raw/e-com-notification-worker.md]]*
