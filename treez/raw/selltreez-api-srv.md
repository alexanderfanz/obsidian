# selltreez-api-srv

> A multi-tenant REST API service for the Treez cannabis retail e-commerce platform. It handles product search (via OpenSearch), discount management, store configuration, and customer group lookups. Deployable as an AWS Lambda function or a standalone HTTP server. Acts as the primary backend for the SellTreez storefront experience.

---

## Purpose

Expose a thin, tenant-aware API over two data stores — OpenSearch (product search) and DynamoDB (config, discounts, groups) — and emit structured analytics events to AWS Firehose on every product search. The service is stateless and horizontally scalable by design.

**Out of scope:** inventory management, order processing, payment handling, and any write operations to product data (products are indexed by a separate process).

---

## Key Entities / Domain Models

All domain types come from the shared `selltreez-lib` library (imported as `github.com/gap-commerce/selltreez-lib`).

- **`config.TreezConfig`** — Store-level configuration keyed by store name. Retrieved from DynamoDB with key `config#<store_name>`.
- **`discount.TreezDiscount`** — A promotional discount rule. Includes schedule rules (day-of-week, time ranges, date ranges), soft-delete flag (`CurrentFlag`), cart-applicability flag, and discount type/value. Stored in DynamoDB with key `discount#<entity_id>`.
- **`group.Group`** — A customer group (e.g. medical, recreational). Stored in DynamoDB with key `group#<entity_id>`.
- **`product.OpenSearchProduct`** — Product document stored and searched in OpenSearch. Index is named `<prefix>_<org_id>_<entity_id>` for per-tenant isolation.
- **Analytics Events** — `SearchEvent` and `SearchErrorEvent` structs built internally, serialized to JSON, and streamed to Firehose. Not persisted locally.

---

## API Surface

All routes are registered in `internal/app/wiring.go:103-143`. Multi-tenancy is established via `Entity-Id` and `org-id` request headers, extracted by middleware into the request context.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/healthcheck` | Health check |
| GET | `/config/{store_name}` | Get store config by name |
| GET | `/discount` | Get all discounts for an entity (supports `excludeInactive`, `timezone` query params) |
| GET | `/discount/{key}` | Get a single discount by key |
| POST | `/product/search` | Proxy an OpenSearch query for products; emits analytics events |
| GET | `/group` | Get all customer groups for an entity |
| GET | `/group/{id}` | Get a single customer group by ID |
| POST | `/opensearch/clear-index` | Delete products from an index by injection date (used by indexing pipeline) |
| GET | `/api-docs/` | Swagger UI (non-production only) |

---

## Data Layer

**DynamoDB** — Single-table design. Partition key: `key`, sort key: `sort_key`. All three entity types (config, discount, group) share one table, differentiated by key prefix. Accessed via a generic Go repository (`internal/storage/dynamo/db.go`) parameterized with the entity type.

**OpenSearch** — AWS-managed OpenSearch cluster. One index per tenant, named `<prefix>_<org_id>_<entity_id>`. The service proxies raw OpenSearch query JSON from the client directly to the cluster — it does not construct queries itself. Credentials are retrieved from AWS Secrets Manager at startup (not via env vars).

**No caching layer.** Each request hits DynamoDB or OpenSearch directly.

---

## Core Business Logic

**Discount filtering and schedule evaluation** (`internal/handler/discount.go:111-157`):
- Soft-deleted discounts (`CurrentFlag == false`) are always excluded.
- Inactive discounts can be optionally excluded via `excludeInactive=true`.
- Discounts without cart applicability are filtered out for cart contexts.
- Schedule rules are evaluated against the request's timezone and current wall-clock time — a discount is only returned if it is currently active per its schedule (days-of-week, time windows, date ranges).

**Product search with analytics** (`internal/handler/product.go`):
- The client sends a raw OpenSearch query body; the service signs and forwards it.
- On success, a `SEARCH` event is extracted and emitted asynchronously (non-blocking, buffered channel, drops if full).
- On 404 (index not found), a `SEARCH_ERROR` event is emitted; Sentry is skipped since this is an expected operational state.
- On other errors, a `SEARCH_ERROR` is emitted and the error is captured to Sentry.

**Analytics event parsing** (`internal/services/analytics/events.go`, ~1,300 lines):
- Deeply parses the OpenSearch query JSON to extract: search term, all applied filters (range, terms, match), pagination, aggregations.
- Performs deterministic JSON serialization (sorted map keys) for the `filters_applied` field to enable downstream deduplication.
- Parses `User-Agent` for device type, browser, and OS.
- Truncates payloads to 4,096 bytes max.

**OpenSearch index cleanup** (`internal/storage/opensearch/index.go`):
- `DeleteByQuery` removes products by `customInjectionDate` field — used to evict stale index data after a re-index cycle.
- `ClearIndex` deletes all documents from an index.

---

## Cross-Repo Connections

- **Calls:**
  - `github.com/gap-commerce/selltreez-lib` — shared domain models (Config, Discount, Group, Product). All DynamoDB and OpenSearch models are defined here.
  - `github.com/gap-commerce/glog` — structured logging wrapper.
  - AWS DynamoDB, OpenSearch (AWS-managed), Firehose, Secrets Manager.

- **Called by:** SellTreez storefront (the customer-facing e-commerce UI). The `POST /opensearch/clear-index` endpoint suggests it is also called by the product indexing pipeline.

- **Shared types / contracts:** Domain models from `selltreez-lib` are the contract between this service and its consumers. Swagger docs at `/docs/swagger.yaml` define the HTTP contract.

- **Events published:** `SEARCH` and `SEARCH_ERROR` events to an AWS Firehose stream (`TRACKING_FIREHOSE_STREAM_NAME`). These likely flow to a data warehouse for analytics.

- **Events subscribed to:** None.

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS DynamoDB | Infrastructure | Stores config, discounts, and customer groups |
| AWS OpenSearch (Managed) | Search | Product full-text and faceted search |
| AWS Firehose | Infrastructure | Analytics event streaming to data warehouse |
| AWS Secrets Manager | Infrastructure | OpenSearch credentials at startup |
| AWS Lambda | Infrastructure | Primary deployment runtime |
| Sentry | Monitoring | Runtime error tracking and alerting |

---

## Notable Patterns & Decisions

**Generic DynamoDB repository** — `internal/storage/dynamo/db.go` uses Go generics (`repository[T]`) to provide a single `Get`/`GetAll` implementation shared by Config, Discount, and Group services. This avoids boilerplate while keeping type safety.

**Lazy-initialized services** — `internal/app/services.go` initializes each service on first use (e.g. `app.ConfigService()`), not at startup. This keeps boot time low and avoids ordering issues.

**Dual deployment model** — `cmd/main.go` checks `AWS_LAMBDA_FUNCTION_NAME` at startup. If set, it wraps the Gorilla Mux router in `aws-lambda-go-api-proxy` and starts the Lambda handler. Otherwise it starts a plain HTTP server on port 8080. No code differences between the two paths.

**Non-blocking async analytics** — The Firehose publisher (`internal/services/analytics/firehose.go`) uses a buffered channel (`AnalyticsBufferSize`, default 256) and a single background worker goroutine. If the buffer is full, the event is dropped with a warning log — the request path is never blocked.

**OpenSearch query passthrough** — The service does not build or validate OpenSearch queries. The client sends raw query JSON and the service signs and proxies it. This makes the service a dumb proxy for search, which is intentional — query construction lives in the frontend/SDK.

**Multi-tenant index isolation** — Each org+entity combination gets its own OpenSearch index (`<prefix>_<org_id>_<entity_id>`). This avoids cross-tenant data leakage and enables per-tenant index lifecycle management without coordination.

**Context-injected request state** — Middleware injects `org-id`, `entity-id`, a request-scoped logger, and Sentry DSN into every request context. Handlers retrieve these via typed context keys (`internal/context/`). This enables log correlation and multi-tenancy without passing values through every function signature.

**Secrets Manager for credentials** — OpenSearch credentials are fetched from AWS Secrets Manager at startup rather than passed as environment variables. This is a security best practice that keeps credentials out of Lambda environment configs and CloudFormation templates.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Architecture | OpenSearch query is passed through as raw `map[string]any` with no validation — malformed or adversarial queries go straight to OpenSearch. Consider basic structural validation. | Med |
| Observability | Analytics event parsing is 1,300 lines in a single file (`events.go`) with no tests visible. Bugs here silently produce bad analytics data. | High |
| DX / tooling | No integration or end-to-end tests. The `make test` target exists but coverage is unclear without running it. | Med |
| Performance | Secrets Manager is called at startup synchronously; no retry or caching if it fails transiently during a cold start. | Low |
| Tech debt | `internal/handler/opensearch.go` mixes index management (clear, reindex) with product-domain concerns — this would fit better in an ops/admin route group. | Low |
| Architecture | Discount schedule evaluation happens in the handler layer (`handler/discount.go`), not the service/domain layer. Business logic in handlers makes it harder to test and reuse. | Med |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/selltreez-api-srv @ 24102a9*
