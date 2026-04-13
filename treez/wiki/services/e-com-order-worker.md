# e-com-order-worker

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Processes order completion events — enriches orders, upserts customer accounts, and fans out to downstream integrations.

**Repo:** [gap-commerce/e-com-order-worker](https://github.com/gap-commerce/e-com-order-worker) | **Language:** Go | **Runtime:** AWS Lambda (ARM64) | **Trigger:** SQS

## Event types handled

| Event | Pipeline |
|-------|----------|
| `order/completed` | Full: enrich → notify → fan-out to all integrations |
| `order/kiosk-completed` | Partial: enrich only — no notifications or integration fan-out |

## What it publishes (SNS)

| Event | Downstream |
|-------|-----------|
| `place_order_klaviyo` | [[e-com-app-worker]] |
| `process_blaze` | [[e-com-app-worker]] |
| `process_jane` | [[e-com-app-worker]] |
| `process_treez` | [[e-com-app-worker]] |
| `order_status_changed_notify` | [[e-com-notification-worker]] |

## Key behaviors

- **Invoice assignment** — atomic DynamoDB counter per store. Only strict ordering guarantee in the pipeline.
- **Conditional integration routing** — each downstream (Blaze, Jane, Treez, Klaviyo) triggered only if enabled in `apps.json`. Treez also checks `automatic_approval` flag.
- **Account upsert** — lifetime spend aggregated; medical ID and DOB sourced from order (cannabis compliance).
- **Kiosk vs. standard** — kiosk orders skip notification and integration events.

## Dependencies

- [[e-com-lib]] — worker framework, domain models
- Publishes to SNS; consumed by [[e-com-app-worker]] and [[e-com-notification-worker]]

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Testing | Most tests skipped (`t.Skip()`); pipeline effectively untested in CI | High |
| Error handling | Some processors silently swallow errors | Med |
| Observability | New Relic events don't capture integration outcomes | Med |

*Source: [[raw/e-com-order-worker.md]]*