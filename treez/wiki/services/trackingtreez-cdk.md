# TrackingTreez CDK

Serverless analytics data lake for search event tracking. Shared across all SellTreez clusters.

**Language:** TypeScript | **IaC:** AWS CDK | **Part of:** [[ops-infra]]

## Resources created

- **S3 data lake** — Parquet files partitioned by `year/month/day`. KMS encryption in prod, lifecycle rules (IA → Glacier → expiration).
- **Kinesis Firehose** — Direct Put mode. JSON → Parquet conversion via Glue schema. Dynamic partitioning. S3 backup of raw JSON in prod.
- **Glue Data Catalog** — database + table with partition projection (2024-2030). No `MSCK REPAIR TABLE` needed.
- **Athena workgroup** — serverless SQL queries with optional bytes-scanned cutoff.
- **CloudWatch alarms** — delivery freshness (>15 min) and throttling (>100 records).
- **Cross-account IAM roles** — for BI tools and Snowflake integration.
- **SSM parameter** — publishes Firehose stream name for SellTreez stacks to discover.

## Key patterns

- **No Lambda in ingestion path** — Firehose handles format conversion and partitioning natively.
- **Partition projection** — Athena auto-discovers partitions mathematically, no maintenance.
- **Environment-tiered encryption** — KMS in prod, SSE-S3 in non-prod.
- **Cross-account via IAM role assumption** — BI team and Snowflake assume provisioned roles.

## Data flow

```
SellTreez reader Lambda → PutRecord → Firehose → JSON→Parquet → S3 (partitioned)
                                                                      ↓
                                                          Athena / Snowflake queries
```

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Observability | No alarm for Firehose conversion failures (dropped records) | High |
| Cost | No Parquet compaction strategy — many small files degrade Athena | Med |
| Data quality | No schema evolution strategy for Glue table | Med |
| Architecture | Partition projection years hardcoded 2024-2030 | Low |

*Source: [[raw/ops-infra-trackingtreez-cdk.md]]*
