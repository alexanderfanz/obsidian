# selltreez-injection-srv

> AWS Lambda that ingests product, discount, config, and group events from EventBridge (via SQS) and synchronizes them into OpenSearch and DynamoDB. It sits at the core of the SellTreez product catalog pipeline — translating raw Treez POS data into the search-optimized shape consumed by the storefront.

---

## Purpose

Receives batched change events from EventBridge and fans them out to the appropriate data stores. Products go to OpenSearch (for storefront search and filtering). Discounts and configs go to DynamoDB (for pricing and store configuration lookups). The service is responsible for the transformation logic between the Treez POS domain model and the SellTreez e-commerce representation — including pricing tier cleanup, lab result normalization, on-sale flag encoding, and menu title enrichment.

Out of scope: it does not query or serve data back to any consumer; it is write-only. It does not handle order events or POS transactions.

---

## Key Entities / Domain Models

- **ProductBatch** — the unit of work for product events; contains an array of `messages` each with an `action` (create/update/delete) and a `data` payload typed as `product.ProductBatch` from `selltreez-lib`. Each product carries inventory, pricing tiers, lab results, discount eligibility flags, and taxonomy metadata.
- **OpenSearchProduct** — the transformed shape written to OpenSearch; flattened and enriched compared to the raw Treez product. Includes computed fields like `on_sale` bitmask, `menu_title`, cleaned tier pricing, and normalized lab values.
- **Discount** — a promotion or pricing rule stored in DynamoDB. Keyed by `key#entityId` + timestamp sort key. Actions are upsert or delete.
- **Config** — store-level configuration record stored in DynamoDB alongside discounts. Same key pattern and batch-write mechanics.
- **Group** — entity type received via `GROUP_DATA_CHANGE` events; processing is a stub (logging only, not yet implemented).
- **BatchEvent** — the internal envelope for a classified SQS batch; carries `EntityId`, `OrganizationId`, and an array of messages with action + JSON data.

---

## API Surface

This service has no HTTP API. It is a Lambda event processor. The contract is the SQS/EventBridge event schema.

**Input: EventBridge detail types routed via SQS**

| Detail Type | Handler | Destination |
|---|---|---|
| `PRODUCT_DATA_CHANGE` | `ProcessProducts()` | OpenSearch (bulk NDJSON) |
| `CONFIG_DATA_CHANGE` | `ProcessConfigs()` | DynamoDB (batch write) |
| `DISCOUNT_DATA_CHANGE` | `ProcessDiscounts()` | DynamoDB (batch write) |
| `GROUP_DATA_CHANGE` | `ProcessGroups()` | (no-op — stub) |

**Event envelope schema:**
```json
{
  "dataVersion": "1.0.0",
  "entityId": "store-id",
  "organizationId": "org-id",
  "messages": [
    {
      "id": "item-id",
      "action": "create|update|delete",
      "updated_at": "2024-01-01T00:00:00Z",
      "data": { }
    }
  ]
}
```

**Output:** `events.SQSBatchResponse` — returns the SQS message IDs of any records that failed processing; SQS will redeliver those messages.

---

## Data Layer

- **OpenSearch** — stores product documents for storefront search. Written via bulk NDJSON API (single request per Lambda invocation batch). Partial failures are parsed from the bulk response and surfaced as SQS failures. 404s on delete operations are ignored. Credentials fetched from AWS Secrets Manager at startup.
- **DynamoDB** — stores discounts and configs. Uses a composite key scheme: `key#entityId` as partition key, timestamp as sort key. Batch writes capped at 25 items (AWS SDK limit), with up to 5 exponential-backoff retries for unprocessed items.
- **No caching layer** — each Lambda invocation reads/writes directly to the data stores.

---

## Core Business Logic

**Product mapping (`internal/products.go`, ~867 lines)** is the most complex part of the service:

- **Tier pricing cleanup (`cleanTierPricing`)** — removes rogue "1G" tier entries for products that shouldn't have them (filtered by entity ID allowlist). Also converts `weight_gram`-denominated tiers to a normalized form.
- **On-sale bitmask (`on_sale` field)** — encodes multiple concurrent discount types as a bitmask integer. Flags include: BOGO, bundle, scheduled discount, and others. Storefront reads this field to determine sale badge logic without querying discounts.
- **Lab result normalization** — parses and sanitizes THC/CBD values: strips HTML entities, removes special symbols (%, `<`, `>`), handles spaced milligram fallback values (e.g. `"10 mg"` → normalized form), and promotes fallback fields when primary values are absent.
- **Menu title enrichment** — if a product has no `MenuTitle`, the service derives one from other product fields before writing to OpenSearch.
- **Custom product extraction (`getCustomProduct`)** — pulls together inventory status, pricing, on-sale flags, and lab values into the enriched OpenSearch shape from the raw Treez product.

**Discount/Config batch writes (`internal/discounts.go`, `internal/configs.go`):**
- Items are split into upserts and deletes based on `action` field.
- DynamoDB `BatchWriteItem` with automatic chunking (25 items/batch) and retry loop for `UnprocessedItems`.
- Both modules share the same batching helper pattern.

---

## Cross-Repo Connections

- **Calls:**
  - `github.com/gap-commerce/selltreez-lib` — shared domain types for products, discounts, configs, and OpenSearch utilities. The mapping logic is tightly coupled to the version of this lib.
  - `github.com/gap-commerce/srv-utils` — config loader (reads environment variables into a struct).
  - `github.com/gap-commerce/glog` — structured logging utilities.
  - AWS OpenSearch, DynamoDB, SQS, Secrets Manager via SDK.

- **Called by:** AWS SQS, which is fed by EventBridge rules triggered by the SellTreez POS event bus.

- **Shared types / contracts:** `selltreez-lib` is the shared contract between this service and anything consuming the product/discount/config domain. Version pinned in `go.mod` (`v0.17.4`).

- **Events:** Consumes `PRODUCT_DATA_CHANGE`, `CONFIG_DATA_CHANGE`, `DISCOUNT_DATA_CHANGE`, `GROUP_DATA_CHANGE`. Does not publish events.

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS Lambda | Infrastructure | Runtime host |
| AWS SQS | Infrastructure | Event queue; batch delivery and failure requeue |
| AWS EventBridge | Infrastructure | Event source routing Treez POS events |
| AWS DynamoDB | Infrastructure | Discount and config storage |
| AWS OpenSearch Service | Search / Storage | Product catalog search index |
| AWS Secrets Manager | Infrastructure | OpenSearch credentials at runtime |
| New Relic | Monitoring | APM transactions; batch size and error count metrics |
| Sentry | Monitoring | Error tracking with org/entity/product context tags |

---

## Notable Patterns & Decisions

**Parallel batch processing:** All four batch types (products, configs, discounts, groups) are processed concurrently using goroutines and a `sync.WaitGroup`. Failure slices are merged after all goroutines complete. This matters because a single SQS batch can contain mixed event types and product processing (OpenSearch bulk) has different latency than DynamoDB writes.

**Single bulk request to OpenSearch:** All product events in a Lambda invocation are assembled into one NDJSON payload and sent in a single HTTP request. This is more efficient but means a network failure affects the whole product batch. Partial failures within the bulk response are parsed item-by-item.

**SQS batch item failure semantics:** The service returns failed SQS message IDs rather than erroring the entire Lambda. This allows partial success — successfully processed items are not redelivered, only the failures.

**DynamoDB key design:** Composite keys in the form `key#entityId` with timestamp sort key. This enables efficient range queries by entity and ordering by recency.

**Observability wrapping:** New Relic wraps the entire `Process()` call as a named transaction with custom attributes (`batch_size`, `success_count`, `error_count`). Sentry captures errors with structured tags (`org_id`, `entity_id`, `product_id`) to make failures traceable to specific catalog items.

**`selltreez-lib` coupling:** The product mapping code is deeply coupled to the domain types and OpenSearch utilities exported by `selltreez-lib`. Any breaking change to that lib requires a coordinated update here.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Missing feature | `ProcessGroups()` is a stub — `GROUP_DATA_CHANGE` events are silently dropped | High |
| Tech debt | `products.go` is 867 lines; product mapping, OpenSearch I/O, and bulk response parsing are mixed in one file | Med |
| Architecture | The entity ID allowlist for the 1G tier pricing filter is hardcoded in `cleanTierPricing` — should be config-driven | Med |
| Tech debt | `aws_region` is hardcoded to `us-west-1` in `cmd/main.go` | Low |
| DX / tooling | No integration test against a real OpenSearch or DynamoDB (LocalStack); tests rely on JSON fixtures and unit-level mocks | Low |
| Performance | If a single product event fails OpenSearch auth/connectivity, the entire product batch fails and all messages are requeued — no partial-retry at the product level | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/selltreez-injection-srv @ 8e3c01b*
