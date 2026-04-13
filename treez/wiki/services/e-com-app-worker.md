# e-com-app-worker

Processes async SQS events for order placement (to POS), CRM subscriptions, loyalty discounts, and new store initialization.

**Repo:** [gap-commerce/e-com-app-worker](https://github.com/gap-commerce/e-com-app-worker) | **Language:** Go | **Runtime:** AWS Lambda (ARM64) | **Trigger:** SQS

## Event types handled

| Event | Processors |
|-------|-----------|
| `place-order-treez` | PlaceOrderTreez → AlpineIQDiscount → StickyDiscount |
| `place-order-blaze` | PlaceOrderBlaze |
| `place-order-jane` | PlaceOrderJane |
| `place-order-klaviyo` | PlaceOrderKlaviyo |
| `subscribe-to-klaviyo` | SubscribeKlaviyo |
| `subscribe-to-omnisend` | SubscribeOmnisend |
| `subscribe-to-alpine-iq` | SubscribeAlpineIQ |
| `subscribe-to-listrack` | SubscribeListrack |
| `create-store` | CreateStore (scaffolds S3 config for new store) |

## Key behaviors

- **Multi-processor pipeline for Treez** — `place-order-treez` chains three processors sequentially. Discount processors are error-tolerant (order already placed; failure logged but message still deleted).
- **Verification photo upload** — Before finalizing a Treez ticket, fetches customer ID/medical photos from S3 and uploads to Treez document API for age-verification compliance.
- **Payment type detection** — Inspects order payment data to distinguish credit vs. debit for Treez.
- **Store scaffold** — `create-store` writes 14 default S3 config keys seeding a brand-new store.
- **Credentials in S3** — Per-store API keys stored in `apps.json`, not SSM. Rotation requires S3 update, not Lambda config change.

## Dependencies

- [[e-com-lib]] — worker framework, domain models, integration clients
- Upstream: [[e-com-order-worker]] (publishes events this worker consumes)
- Downstream: publishes SNS completion events for [[e-com-notification-worker]]

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Tech debt | `place-order-treez` processor is large (~600+ lines in test) | Med |
| Architecture | Discount chain is Treez-only; other providers would need duplication | Med |
| DX | Many integration tests unconditionally skipped (`t.Skip()`) | Med |
| Architecture | No DLQ handling visible in this repo | Low |

*Source: [[raw/e-com-app-worker.md]]*
