# api-srv

Webhook orchestration service — the integration backbone between external POS/delivery systems and the GapCommerce platform.

**Language:** Go | **Runtime:** AWS Lambda (ARM64) | **API:** REST

## What it does

Receives inbound webhooks from Blaze, Jane, Treez, and OnFleet. Normalizes data, updates order/inventory state in DynamoDB/S3, and fans out notifications (Alpine IQ SMS, LisTrack email, SNS events).

## Webhook routes

| Route | Provider | Purpose |
|-------|----------|---------|
| `POST /blaze/product-updated` | Blaze | Inventory sync (products, categories, brands, vendors) to S3 |
| `POST /blaze/order-updated` | Blaze | Order status change |
| `POST /jane/order-updated/` | Jane | Order status change (Bearer token auth) |
| `POST /listrack/jane-products/` | LisTrack | Product sync from Jane to LisTrack (API key auth) |
| `POST /treez/order-updated` | Treez | Order status change (revenue source + event type filtering) |
| `POST /treez/product-updated` | Treez | Single product sync to S3 |
| `*/onfleet/task-*` | OnFleet | 6 delivery task lifecycle events |

## Key behaviors

- **Treez order processing** (most complex) — filters by revenue source (`e-commerce` only), filters by event type (`TICKET_STATUS`), maps Treez status → internal status, dispatches Alpine IQ SMS, sends LisTrack email, publishes SNS event.
- **OnFleet delivery** — each task event maps to a status transition. `TaskFailed` reverts to Declined and removes Treez ticket. `TaskCompleted` updates Treez with payment details.
- **Jane orders** — maps 11 Jane statuses to internal, distinguishes delivery vs. pickup for LisTrack.
- **Blaze inventory sync** — syncs all inventory types, then fires Cloudflare cache purge + Vercel rebuild (non-blocking).
- **Generic request builders** — each provider has a `*RequestBuilder` that validates auth, extracts params, decodes body. Handlers receive typed, validated request objects.
- **App credentials in S3** — per-store API keys in `apps.json`, loaded per request. Rotation without redeployment.

## Dependencies

- [[e-com-lib]] — domain models, provider service interfaces
- Publishes to SNS for [[e-com-notification-worker]]
- Called by external systems: Blaze, Jane, Treez, OnFleet

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Credential loading | `apps.json` fetched from S3 on every request — no caching | Med |
| OnFleet errors | Continues processing after Treez ticket update failure | Med |
| Swagger | Annotation drift between actual params and generated docs | Med |

*Source: [[raw/api-srv.md]]*
