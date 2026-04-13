# ops-infra: TrackingTreez CDK Stack

> AWS CDK stack that builds a serverless analytics data lake for search event tracking. Receives search events from SellTreez clusters via Kinesis Firehose, converts them to Parquet format, partitions them by date in S3, and makes them queryable via Athena with a Glue Data Catalog. Supports cross-account read access for BI tools and optional Snowflake integration.

---

## Purpose

TrackingTreez answers the question: "What are users searching for, and when?" Every search query handled by a [[ops-infra-selltreez-cdk]] reader Lambda can optionally be forwarded to this stack's Kinesis Firehose. Over time, this builds a queryable history of search patterns, enabling product analytics, search tuning, and business intelligence.

The stack is deployed independently (not per-entity) and is shared across all SellTreez clusters that opt into tracking. It's the analytics tier of the Gap Commerce platform.

---

## Key Entities / Domain Models

**TrackingTreezStack** is parameterized by environment config:
- `stage` — prod/dev (controls KMS encryption, versioning, S3 backup)
- `firehoseStreamName` — Name for the Kinesis Firehose stream (max 64 chars, forced)
- `crossAccountAccess` — Array of external account configs for BI/Snowflake access
- `glueSchema` — Table schema definition (columns + types) for search events
- `athena` — Query config (bytes scanned cutoff per query, optional)
- `lifecycle` — S3 lifecycle rules (IA transition, Glacier transition, expiration days)
- `intelligentTiering` — Optional S3 Intelligent Tiering config

**Default search event schema** (Glue table columns):
- Standard fields defined in `glueSchema.columns` (search terms, results, timestamps, etc.)
- Partition keys: `year`, `month`, `day`

---

## AWS Resources Created

### S3 Buckets (3)
| Bucket | Name | Purpose |
|--------|------|---------|
| Data Lake | `{prefix}-datalake` | Parquet search event files, partitioned by date |
| Athena Results | `{prefix}-athena-results` | Temporary query result storage (30-day cleanup) |
| Access Logs | `{prefix}-access-logs` | S3 server access logs (90-day expiration) |

**Data Lake bucket (prod):**
- KMS encryption with customer-managed key
- Versioning enabled
- Intelligent Tiering (optional)
- Lifecycle rules: configurable IA transition, Glacier transition, expiration
- CORS: GET + HEAD (for direct browser access by BI tools)
- Server access logs → access logs bucket

**Data Lake bucket (non-prod):**
- SSE-S3 encryption (no KMS)
- No versioning

### Kinesis Firehose
- Mode: Direct Put (no Kinesis stream — SellTreez Lambdas call PutRecord directly)
- Destination: Extended S3 with:
  - **Dynamic partitioning**: `year=.../month=.../day=.../` via JQ metadata extraction
  - **Format conversion**: JSON → Parquet (SNAPPY compression)
  - Glue catalog integration for schema awareness
  - S3 prefix: `data/` with dynamic partition suffix
  - CloudWatch error logging enabled
  - **S3 backup mode**: enabled in prod (raw JSON backup before conversion)
  - Buffer: event-size based (not interval-based) for efficient batching

### Glue Data Catalog
- **Database**: `{prefix}-search-events`
- **Table**: `{prefix}-search-events-table`
  - Table type: EXTERNAL_TABLE
  - Format: Parquet
  - Location: `s3://{data-lake-bucket}/data/`
  - Partition projection enabled:
    - `year`: 2024–2030
    - `month`: 1–12
    - `day`: 1–31
  - Projection eliminates the need for `MSCK REPAIR TABLE` — Athena auto-discovers partitions

### Athena Workgroup
- Engine: version 3 (latest)
- Query result location: `{athena-results-bucket}/`
- Optional bytes scanned cutoff (prevents runaway query costs)
- CloudWatch metrics enabled

### CloudWatch Alarms (2)
- **Firehose delivery freshness**: fires if data age > 15 minutes (delivery is stale)
- **Firehose throttling**: fires if > 100 throttled records (ingestion backpressure)

### IAM Roles (cross-account access, optional)
For each configured cross-account consumer:
- **Read role** — allows:
  - `s3:GetObject`, `s3:ListBucket` on data lake bucket and Athena results bucket
  - `glue:GetTable`, `glue:GetDatabase`, `glue:GetPartitions` on search events catalog
  - `athena:StartQueryExecution`, `athena:GetQueryResults`, etc. on the workgroup
- Optional external ID support for strict trust policy
- Supports multiple cross-account consumers (e.g. a data science AWS account, a BI AWS account)

### Snowflake Storage Integration (optional)
- **IAM Role**: Trusted by Snowflake's AWS account + external ID
- Grants Snowflake `s3:GetObject`, `s3:ListBucket` on the data lake
- Enables Snowflake External Tables pointing at the Parquet files

### SSM Parameter Store Output
- `{appName}/{stage}/tracking-firehose-stream-name` — Published globally so SellTreez stacks can discover the stream name at deploy time

---

## Data Layer

### S3 (Data Lake)
Primary storage. Parquet files, partitioned as:
```
s3://{datalake-bucket}/data/
  year=2024/month=11/day=15/
    *.parquet
  year=2024/month=11/day=16/
    *.parquet
```
Partition projection in the Glue table means Athena never needs to scan the full bucket — queries with date filters only read relevant partitions.

### Kinesis Firehose (Ingestion)
Buffer: Firehose batches records before writing to S3. The buffer is optimized for throughput, not latency — there can be a delay of seconds to minutes between a search event and it appearing in S3.

### Athena (Query Engine)
SQL queries against the Parquet data. Athena is serverless — no cluster to manage. Cost is per-byte-scanned. The optional bytes cutoff prevents accidental full-table scans.

### Glue Data Catalog
Schema registry and partition management. The Glue table is EXTERNAL — dropping the table doesn't delete data. Partition projection avoids maintaining partition metadata manually.

---

## Core Business Logic

### JSON → Parquet Conversion
Firehose uses the Glue schema definition to convert raw JSON search events into Parquet columnar format before writing to S3. This is handled by Firehose's built-in data format conversion — no Lambda in the ingestion path.

### Dynamic Partitioning
Firehose's JQ metadata extraction pulls `year`, `month`, `day` from each event's timestamp and uses them to construct the S3 prefix. This means:
- No Lambda needed for partitioning
- Athena queries with date filters skip non-matching partitions automatically
- No daily partition maintenance job required

### Partition Projection
The Glue table uses Athena partition projection:
```
year: [2024, 2030] integer
month: [1, 12] integer
day: [1, 31] integer
```
Athena generates partition paths mathematically rather than listing S3 or querying the Glue catalog. This makes queries start faster and eliminates stale partition issues.

---

## Cross-Repo Connections

**Called by:**
- [[ops-infra-selltreez-cdk]] — Reader Lambda calls `firehose:PutRecord` with each search event; reads the stream name from SSM Parameter Store

**Provides to:**
- Cross-account BI tools — read S3 data via IAM role assumption
- Snowflake — reads S3 data via storage integration
- Athena-compatible tools — run SQL queries against the workgroup

**Shares:**
- SSM Parameter Store: publishes `tracking-firehose-stream-name` for SellTreez stacks to consume

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| Snowflake | Analytics | External Table reading Parquet files from S3 (optional) |
| Amazon Athena | Analytics | Ad-hoc SQL queries on search event data |
| Amazon Glue | Infrastructure | Data catalog and schema registry |
| Kinesis Firehose | Infrastructure | Buffered event delivery with format conversion |

---

## Notable Patterns & Decisions

1. **No Lambda in the ingestion path** — Kinesis Firehose handles format conversion (JSON → Parquet) and dynamic partitioning natively. This reduces operational complexity and cost — no Lambda to maintain, scale, or debug for routine ingestion.

2. **Partition projection over MSCK REPAIR TABLE** — Traditional Athena + Glue setups require `MSCK REPAIR TABLE` to discover new partitions (or a daily Glue crawler). Partition projection eliminates this: Athena computes possible partitions mathematically, so new date partitions are instantly queryable.

3. **S3 backup before format conversion (prod)** — Firehose backs up raw JSON to a separate S3 prefix before converting to Parquet. If schema changes break the conversion, the raw data is preserved for re-processing.

4. **Environment-tiered encryption** — Prod uses KMS (customer-managed key, auditable). Non-prod uses SSE-S3 (simpler, no extra cost). Appropriate risk-based differentiation.

5. **Cross-account via IAM role assumption** — External consumers (BI team, Snowflake) assume a role provisioned by this stack. No credentials shared, no VPC required, clean boundary.

6. **Bytes scanned cutoff** — Optional but important for cost control. An accidental `SELECT *` on years of search data could cost significant money. The cutoff cancels queries that exceed the threshold.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Observability | No alarm if Firehose fails to convert records (conversion failure just drops records by default) | High |
| Architecture | Partition projection assumes years 2024-2030 — hardcoded range needs updating every 5 years | Low |
| Cost | No compaction strategy — many small Parquet files accumulate over time, degrading Athena query performance | Med |
| DX | No sample Athena queries provided in docs or CDK — operators must write queries from scratch | Low |
| Data quality | No schema evolution strategy — adding new fields to the Glue table requires a redeploy and may break queries against old partitions | Med |
| Security | KMS key policy not shown in stack — ensure the cross-account roles can use the key for decryption | Med |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/ops-infra/gap-store-manager/cdk/trackingtreez (commit 9ed4512)*
