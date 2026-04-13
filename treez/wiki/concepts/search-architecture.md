# Search Architecture

Product search is served by the **SellTreez** pipeline — a dedicated system separate from the e-commerce core.

## Architecture

```
Treez POS
  └─▶ EventBridge (cross-account) → SQS (3s buffer)
        └─▶ selltreez-injection-srv (writer Lambda)
              ├─▶ OpenSearch (product documents, bulk NDJSON)
              └─▶ DynamoDB (discounts, configs)

Customer / Storefront
  └─▶ selltreez-api-srv (reader Lambda, API Gateway + API key)
        ├─▶ OpenSearch (query passthrough)
        └─▶ Firehose → TrackingTreez (search analytics)
```

## Components

| Component | Role |
|-----------|------|
| [[selltreez-lib]] | Shared types: product models, discount evaluation, OpenSearch utilities |
| [[selltreez-injection-srv]] | Writes products to OpenSearch, discounts/configs to DynamoDB |
| [[selltreez-api-srv]] | Serves search queries, discount lookups, config; emits analytics |
| [[SellTreez CDK]] | Provisions OpenSearch domain, Lambdas, EventBridge, SQS, API Gateway |
| [[TrackingTreez CDK]] | Analytics data lake: Firehose → Parquet → S3 → Athena |

## Product indexing pipeline

1. Treez POS publishes product change events to EventBridge (cross-account)
2. EventBridge rule routes to SQS with 3-second delivery delay (batching)
3. [[selltreez-injection-srv]] writer Lambda processes the batch:
   - Maps raw Treez products to `OpenSearchProduct` shape (~867 lines of transformation)
   - Tier pricing cleanup (hardcoded entity ID allowlist for "1G" filter)
   - Lab result normalization (HTML stripping, mg fallback values)
   - On-sale bitmask encoding (BOGO, bundle, scheduled discount flags)
   - Bulk writes to OpenSearch via single NDJSON request
4. Products appear in search within seconds

## Search query flow

1. [[treez-tlp]] storefront constructs OpenSearch DSL queries client-side
2. Sends raw query JSON to [[selltreez-api-srv]] via `POST /product/search`
3. Service signs the request and proxies it to OpenSearch (no server-side query building)
4. Results returned to client
5. Search event asynchronously forwarded to Kinesis Firehose (non-blocking, dropped if buffer full)

## Multi-tenant index isolation

Each org+entity combination gets its own OpenSearch index: `{prefix}_{orgId}_{entityId}`. No cross-tenant data leakage. Per-tenant index lifecycle managed independently.

## Analytics

Search events flow to [[TrackingTreez CDK]]:
- Firehose converts JSON → Parquet with dynamic date partitioning
- S3 data lake queryable via Athena
- Partition projection eliminates `MSCK REPAIR TABLE`
- Cross-account access for BI tools and Snowflake

## Algolia vs. OpenSearch

Both Algolia and OpenSearch appear across the platform:
- [[Store CDK]] Lambda is named "algolia" but uses OpenSearch (legacy naming)
- [[gap-dashboard]] has both Algolia and OpenSearch search paths
- [[treez-tlp]] references both backends
- [[ops-infra]] has Algolia in secrets but appears partially commented out

The platform appears to be **mid-migration from Algolia to OpenSearch**. OpenSearch is the primary backend for new work (SellTreez pipeline). Algolia code is present but increasingly dormant. Consolidating would reduce maintenance surface.
