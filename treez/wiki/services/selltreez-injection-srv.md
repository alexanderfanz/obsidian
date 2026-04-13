# selltreez-injection-srv

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Ingests product, discount, config, and group events from EventBridge (via SQS) and writes to OpenSearch and DynamoDB.

**Repo:** [gap-commerce/selltreez-injection-srv](https://github.com/gap-commerce/selltreez-injection-srv) | **Language:** Go | **Runtime:** AWS Lambda (ARM64) | **Trigger:** SQS (fed by EventBridge)

## Event types handled

| Detail Type | Destination |
|-------------|------------|
| `PRODUCT_DATA_CHANGE` | OpenSearch (bulk NDJSON) |
| `CONFIG_DATA_CHANGE` | DynamoDB (batch write) |
| `DISCOUNT_DATA_CHANGE` | DynamoDB (batch write) |
| `GROUP_DATA_CHANGE` | **Stub — not implemented** |

## Key behaviors

- **Product mapping** (~867 lines) — the most complex part. Includes:
  - Tier pricing cleanup (rogue "1G" tier filtered by entity ID allowlist)
  - On-sale bitmask encoding (BOGO, bundle, scheduled discount flags)
  - Lab result normalization (strips HTML, handles mg fallback values)
  - Menu title enrichment
- **Parallel batch processing** — all four batch types processed concurrently via goroutines. Failure slices merged after all complete.
- **Single bulk OpenSearch request** — all product events in one NDJSON payload. Efficient but a network failure affects the whole batch.
- **SQS partial failure** — returns failed message IDs so only failures are redelivered.
- **DynamoDB batching** — 25-item chunks with exponential backoff retries for unprocessed items.

## Dependencies

- [[selltreez-lib]] — domain types, OpenSearch utilities (tightly coupled)
- Infrastructure provisioned by [[SellTreez CDK]]

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Missing feature | `ProcessGroups()` is a stub — GROUP events silently dropped | High |
| Tech debt | `products.go` is 867 lines mixing mapping, I/O, and parsing | Med |
| Architecture | 1G tier pricing allowlist is hardcoded | Med |

*Source: [[raw/selltreez-injection-srv.md]]*