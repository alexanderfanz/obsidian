# ops-infra: SellTreez CDK Stack

> AWS CDK stack that provisions a dedicated OpenSearch-powered product search cluster for a SellTreez entity. Deployed dynamically by CodeBuild. Creates an OpenSearch domain, reader + writer Lambda functions, an EventBridge event bus (supporting cross-account event publishing), an SQS queue for buffered ingestion, a REST API with API key auth, and optionally a TrackingTreez Firehose integration for search event analytics.

---

## Purpose

SellTreez is the search layer for product inventory in Gap Commerce. Each SellTreez entity is an independent search cluster — operators can create multiple clusters with different configurations (e.g. different instance types, different data schemas). The CDK stack provisions all the AWS infrastructure for one cluster.

The core pattern is event-driven ingestion: external accounts (e-com services) publish product events to an EventBridge bus → SQS buffers them → writer Lambda indexes them into OpenSearch. The reader Lambda serves search queries via REST API.

Distinct from Groups and Stores (which are tied to tenant hierarchy), SellTreez clusters can be shared across groups or operated independently.

---

## Key Entities / Domain Models

**SellTreezStack** is parameterized at deploy time:
- `id` — Cluster identifier (resource name prefix)
- `name`, `description`, `stage`
- `lambdaReader` — Reader Lambda config: `srcKey` (S3), `memorySize`, `timeoutSeconds`
- `lambdaWriter` — Writer Lambda config: same fields
- `events` — EventBridge config: `source`, `detailType`, `externalAccountId` (for cross-account publishing)
- `opensearch` — Cluster config: `instanceType`, `dataNodes`, `volumeSize`
- `domain` — Custom domain: `name`, `enableCustomDomain`
- `tracking` — `enabled` flag for TrackingTreez Firehose integration

---

## AWS Resources Created (per SellTreez)

### OpenSearch Domain
- Engine: OpenSearch 2.19
- Cluster config: from `opensearch` entity field (instanceType, dataNodes, volumeSize)
- Fine-grained access control enabled (master credentials stored in Secrets Manager)
- TLS enforced, HTTP enabled
- Node-to-node encryption enabled
- Encryption at rest enabled
- Slow log enabled (index + search)
- No dedicated master node (data nodes handle coordination)

### Lambda Functions (2)
| Name | Source | Memory | Timeout | Trigger |
|------|--------|--------|---------|---------|
| `{prefix}-reader` | `lambdaReader.srcKey` | configurable | configurable | API Gateway |
| `{prefix}-writer` | `lambdaWriter.srcKey` | configurable | configurable | SQS |

**Reader:** Handles search queries (proxied from API Gateway), optionally emits search events to Kinesis Firehose if tracking is enabled.

**Writer:** Processes messages from SQS, indexes documents into OpenSearch. Batch processing with buffering.

### SQS Queues (2)
- **Search Queue** — `{prefix}-search-queue`
  - 3 MB buffering, 3-second delivery delay (batching optimization)
  - Receives events from EventBridge rule
- **Search DLQ** — `{prefix}-search-dlq`
  - Receives failed processing attempts

### EventBridge
- **Event Bus** — `{prefix}-event-bus`
- **Event Bus DLQ** — for failed event deliveries
- **Rule** — Matches on `source` + `detailType` from entity config → routes to SQS queue
- **Cross-account publishing policy** (optional) — If `externalAccountId` is set, grants that AWS account permission to publish events to this bus. Enables e-com services in a different account to push product updates.

### DynamoDB Table
- Name: `{prefix}-table`
- PK: `key`, SK: `sort_key`
- PAY_PER_REQUEST
- Used for metadata (index definitions, ingestion state, etc.)

### REST API Gateway
- Proxy integration to reader Lambda
- API key required (`X-Api-Key` header)
- Usage plan attached to API key
- CORS headers: `Authorization`, `X-Api-Key`, `entity-id`, `org-id`
- Optional custom domain: `search-{id}.{domain}` with ACM certificate

### Kinesis Firehose Integration (optional)
If `tracking.enabled = true`:
- Reader Lambda gets IAM permissions: `firehose:PutRecord`, `firehose:PutRecordBatch`
- Environment variable `TRACKING_FIREHOSE_STREAM_NAME` set to the TrackingTreez stack's Firehose stream
- Every search query the reader handles gets forwarded to the Firehose for analytics

---

## Data Layer

### OpenSearch
- Primary data store for product inventory search
- Index definitions and mappings are managed by operators via the Store Manager UI (SellTreez Indexes tab)
- Credentials stored in Secrets Manager (master user/pass)
- Direct access is fine-grained: CodeBuild and the AWS account have collection + index + data access via OpenSearch resource policies

### DynamoDB
- Secondary store for operational metadata
- Not the source of truth for search data (that's OpenSearch)

---

## Core Business Logic

### Event-Driven Ingestion Pipeline
```
External Account (e-com service, different AWS account)
  └─▶ PutEvents → EventBridge Bus ({prefix}-event-bus)
        └─▶ Rule matches source + detailType
              └─▶ SQS Queue (3s buffering delay)
                    └─▶ Writer Lambda
                          └─▶ OpenSearch (bulk index)
```

The 3-second delivery delay on SQS is intentional — it batches small bursts of events so the writer Lambda indexes them in bulk rather than one-at-a-time, reducing OpenSearch write overhead.

### Cross-Account Event Publishing
The EventBridge bus policy includes a statement allowing a specific external AWS account ID to publish events. This is how the injection service (`selltreez-injection-srv`) in the e-com account pushes product data to the search cluster in the ops-infra account — no credentials shared, just IAM resource policies.

### API Key Auth for Search
Search queries come through API Gateway with `X-Api-Key`. This provides a lightweight auth mechanism for the search API without requiring Cognito (appropriate for server-to-server calls where the client is an e-com service).

### Search Event Tracking (optional)
When enabled, the reader Lambda forwards each search request to Kinesis Firehose → [[ops-infra-trackingtreez-cdk]]. This builds an analytics trail of what users searched for, powering the TrackingTreez data lake.

---

## Cross-Repo Connections

**Calls:**
- `selltreez-api-srv` — Reader Lambda source code (search query handling)
- `selltreez-injection-srv` — Publishes events to this stack's EventBridge bus; writer Lambda source may come from here
- [[ops-infra-trackingtreez-cdk]] — Reads the Firehose stream name from SSM Parameter Store; reader Lambda sends search events to it

**Called by:**
- [[ops-infra-store-manager-backend]] — `actions-manager` triggers CodeBuild to deploy this stack
- E-com services (external account) — publish product events to EventBridge bus
- E-com frontend — calls search REST API (via API key)

**Shares:**
- Secrets Manager: `/gap-store-manager/allSecrets`
- SSM Parameter Store: reads `tracking-firehose-stream-name` from TrackingTreez stack output

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| OpenSearch (AWS) | Search | Full-text product inventory search |
| Kinesis Firehose (AWS) | Analytics | Forwarding search events to TrackingTreez data lake |
| CloudWatch | Monitoring | Lambda invocation and SQS depth metrics |

---

## Notable Patterns & Decisions

1. **SQS 3-second delivery delay** — Deliberate batching to reduce OpenSearch write pressure. Product events are rarely so time-sensitive that 3-second indexing lag is a problem, but the reduction in write amplification is meaningful at scale.

2. **Cross-account EventBridge via resource policy** — Avoids the need for shared credentials or VPC peering. The injection service in another AWS account can publish events directly to this bus without any IAM role assumption — a clean service boundary.

3. **Memory/timeout is operator-configurable** — Unlike Group/Store stacks where Lambda config is hardcoded, SellTreez Lambda memory and timeout are stored in the entity configuration and applied at CDK deploy time. Operators can right-size each cluster independently.

4. **OpenSearch fine-grained access control** — Master credentials are generated and stored in Secrets Manager. Both CodeBuild (for index management) and the Lambda functions (for document writes) get explicit access via OpenSearch resource-based policies, not just VPC-level access.

5. **API key auth instead of Cognito** — Search is a server-to-server API call in this architecture. API keys are simpler to manage for this use case and avoid requiring the calling service to implement Cognito auth.

6. **Optional custom domain** — The API can run on the default API Gateway URL or on a custom domain (`search-{id}.{domain}`). Custom domain is controlled per-entity.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Observability | No CloudWatch alarms for OpenSearch cluster health (JVM pressure, indexing errors, shard failures) | High |
| Cost | OpenSearch domain is provisioned even for unused/test SellTreez entities — could be dormant cost | Med |
| Security | API key auth doesn't support rotation — a compromised key requires manual intervention | Med |
| Architecture | Cross-account EventBridge policy is set to a single `externalAccountId` — doesn't support multiple source accounts per cluster | Low |
| Testing | No tests for CDK stack synthesis or Lambda handler logic | Med |
| DX | OpenSearch master password rotation is manual — no automated rotation via Secrets Manager | Med |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/ops-infra/gap-store-manager/cdk/selltreez (commit 9ed4512)*
