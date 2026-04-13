# selltreez-api-srv

Multi-tenant REST API for product search, discount management, store config, and customer groups. Primary backend for the SellTreez storefront experience.

**Repo:** [gap-commerce/selltreez-api-srv](https://github.com/gap-commerce/selltreez-api-srv) | **Language:** Go | **Runtime:** AWS Lambda or standalone HTTP | **API:** REST

## Routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/config/{store_name}` | Store config |
| GET | `/discount` | All discounts (supports `excludeInactive`, `timezone`) |
| GET | `/discount/{key}` | Single discount |
| POST | `/product/search` | Proxy OpenSearch query; emits analytics events |
| GET | `/group` | Customer groups |
| GET | `/group/{id}` | Single group |
| POST | `/opensearch/clear-index` | Delete products by injection date |

## Key behaviors

- **OpenSearch query passthrough** — client sends raw query JSON; service signs and proxies it. No server-side query construction.
- **Discount schedule evaluation** — filters by soft-delete, inactive, cart-applicability, and time-based schedule rules using [[selltreez-lib]] logic.
- **Non-blocking analytics** — Firehose publisher uses a buffered channel (256 items) with a background worker. If full, events are dropped (never blocks request path).
- **Multi-tenant index isolation** — each org+entity gets its own OpenSearch index.
- **Generic DynamoDB repository** — Go generics (`repository[T]`) shared across Config, Discount, and Group.
- **Dual deployment** — Lambda or plain HTTP on port 8080, detected at startup.

## Dependencies

- [[selltreez-lib]] — domain models
- Firehose → [[TrackingTreez CDK]] (search analytics)
- Infrastructure provisioned by [[SellTreez CDK]]

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Observability | Analytics event parsing is 1,300 lines with no visible tests | High |
| Architecture | Raw OpenSearch queries passed through with no validation | Med |
| Architecture | Discount schedule evaluation lives in handler layer, not domain layer | Med |

*Source: [[raw/selltreez-api-srv.md]]*
