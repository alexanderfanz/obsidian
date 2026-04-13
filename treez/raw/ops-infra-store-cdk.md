# ops-infra: Store CDK Stack

> AWS CDK stack that provisions all infrastructure for a single retail Store within a merchant Group. Deployed dynamically at runtime by CodeBuild. Creates a dedicated Cognito user pool (for store customers), a DynamoDB table (for store entity data), a DynamoDB stream-triggered Lambda (for OpenSearch/Algolia indexing), and a Cognito trigger Lambda (for pre-signup/post-confirmation hooks).

---

## Purpose

Each Store in Gap Commerce gets isolated customer-facing infrastructure. The Store CDK stack is that infrastructure. It runs independently of the Group stack but depends on the Group's SNS topic and S3 config bucket (passed in at deploy time via group config).

The store is the unit where customers authenticate and where store-level data (orders, customers, etc.) is stored. The Group provides the API layer; the Store provides the data layer and customer identity.

---

## Key Entities / Domain Models

**StoreStack** is parameterized at deploy time via `ENTITY_CONFIGURATION`:
- `id` — Unique store identifier (prefix for all resource names)
- `accountId` — E-commerce account ID (links to merchant account)
- `groupId` — Parent group ID
- `stage` — Environment (prod/dev)
- `cognitoConfig` — Custom attributes, roles, SES email config
- `bucketConfigsStoresGroupName` — S3 bucket name from parent Group stack (for config files)
- `topicPubSubArn` — Parent Group's SNS topic ARN (stores subscribe to this)
- `lambdaAlgoliaSrcKey` — S3 key for the Algolia/DynamoDB stream processor Lambda
- `cognitoLambdaTriggerSrcKey` — S3 key for the Cognito trigger Lambda
- `monitor` — CloudWatch alarm config (Lambda/DynamoDB thresholds)

---

## AWS Resources Created (per Store)

### DynamoDB Table
- Name: `{prefix}-store-stack-table`
- Partition key: `entity_id` (STRING)
- Sort key: `key` (STRING)
- Billing: PAY_PER_REQUEST
- **DynamoDB Streams**: enabled (NEW_IMAGE capture)
- Point-in-time recovery: enabled
- GSIs:
  - `status-index` — query by status
  - `email-index` — query by email
  - `key-created_at-index` — query by key + creation time

### Lambda Functions (2)
| Name | Source | Memory | Timeout | Trigger |
|------|--------|--------|---------|---------|
| `{prefix}-store-stack-cognito-trigger` | `cognitoLambdaTriggerSrcKey` | 1024 MB | 300s | Cognito pre-signup + post-confirmation |
| `{prefix}-store-stack-algolia` | `lambdaAlgoliaSrcKey` | 1024 MB | 300s | DynamoDB Stream |

**DynamoDB Stream → Algolia Lambda:**
- Filter: only processes items where `key` starts with `d#`
- Batch size: 1
- Maximum retry attempts: 5
- Integrates with OpenSearch Serverless (AOSS) — and optionally Algolia (currently commented out in code)

**Cognito Trigger Lambda:**
- Runs on Cognito pre-signup (validate/enrich) and post-confirmation (side effects)
- Receives Cognito event context; can write to store's DynamoDB, call external APIs

### Cognito User Pool
- Name: `{prefix}-store-stack-cup`
- Email-based sign-in
- Custom attributes: from `cognitoConfig.customAttributes` (e.g. ADDRESS2, CITY, COUNTRY_CODE, BERBIX_*, etc.)
- Cognito Groups: from `cognitoConfig.roles` (typically CUSTOMER, PUBLIC)
- Password policy: 8+ chars, uppercase, lowercase, digits, symbols
- Lambda triggers: pre-signup, post-confirmation → `cognito-trigger` Lambda
- SES integration (optional): custom email domain with DKIM records in Route 53
- Retention: DESTROY (table cleaned up with stack — careful with prod)

### Initial Invoice Number
- An `AwsCustomResource` is used to write an initial invoice number to the store's DynamoDB table at stack creation time
- This is a bootstrap operation — sets the starting counter for the store's billing sequence

---

## Data Layer

### DynamoDB Schema (per Store)
The store table uses a generic entity schema:
- `entity_id` (PK) — Entity type prefix (e.g. `customer`, `order`)
- `key` (SK) — Unique identifier or composite key
- `key-created_at-index` — Enables paginated listing sorted by creation time
- `email-index` — Customer lookup by email
- `status-index` — Filter by entity status

Items where `key` starts with `d#` are the "document" entries that get streamed to OpenSearch/Algolia for search indexing.

No caching layer. DynamoDB is the source of truth.

---

## Core Business Logic

### DynamoDB Stream → Search Index
Every write to the store's DynamoDB table is captured by the stream. Items matching the `d#` key prefix are processed by the Algolia Lambda:
1. Extract the new image from the stream event
2. Determine the index (OpenSearch collection or Algolia index)
3. Upsert the document into the search index

This provides near-real-time search indexing for store entities (customers, products, etc.) without polling.

### Cognito Pre-signup Hook
The Cognito trigger Lambda fires before a user is created. Used for:
- Custom validation (e.g. domain allow-listing)
- Enriching the user record with store-specific attributes
- Blocking sign-ups that don't meet business criteria

### SES Email Integration
If the store config enables SES, the stack:
1. Creates an SES email identity for the store's domain
2. Adds DKIM records to Route 53
3. Configures Cognito to send emails via SES instead of the default Cognito email

This is required for production stores that need branded transactional emails (e.g. "Verify your email — CannaCo").

---

## Cross-Repo Connections

**Calls:**
- `e-com-cognito-worker` — Lambda source uploaded here; deployed as the Cognito trigger Lambda
- `e-com-dynamodbstream-srv` — Lambda source uploaded here; deployed as the Algolia/stream Lambda
- OpenSearch Serverless — Algolia Lambda writes search documents here

**Called by:**
- [[ops-infra-store-manager-backend]] — `actions-manager` triggers CodeBuild to deploy this stack
- [[ops-infra-group-cdk]] — Group GraphQL Lambda reads from this stack's DynamoDB table (via the pattern-matched IAM policy)

**Shares with parent Group:**
- SNS topic ARN — store subscribes to group pub/sub for relevant events
- S3 config bucket — group writes store configuration files here; store Lambda reads them

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS SES | Email | Transactional emails from store domain (optional) |
| Algolia | Search | Search index (code present, appears commented out — OpenSearch used instead) |
| OpenSearch (AWS Serverless) | Search | Primary search backend for store entities |
| CloudWatch (cdk-monitoring-constructs) | Monitoring | Lambda + DynamoDB alarms |

---

## Notable Patterns & Decisions

1. **AwsCustomResource for initial data** — CDK's `AwsCustomResource` construct runs a DynamoDB PutItem at stack deploy time to seed the invoice counter. This is a rare CDK pattern — using a custom resource to write initial data rather than a separate migration step.

2. **DynamoDB stream filter (`d#` prefix)** — Not all DynamoDB writes should trigger search indexing. The `d#` prefix convention separates "document" records (searchable) from metadata records (internal). This is a naming convention, not enforced by a schema.

3. **Algolia vs OpenSearch** — The Lambda is named "algolia" but the actual implementation uses OpenSearch Serverless. Algolia code appears to be present but commented out. The naming is legacy and should be cleaned up.

4. **DESTROY retention on Cognito pool** — The Cognito user pool has `removalPolicy: DESTROY`, meaning it gets deleted when the stack is deleted. This is appropriate for ephemeral dev stacks but risky for production — user accounts would be lost.

5. **Pattern-matched S3 bucket permissions** — Like the Group stack, the store Lambdas get S3 access via wildcards rather than specific bucket ARNs, enabling zero-touch access to new buckets that match the naming pattern.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Safety | Cognito pool has DESTROY removal policy — should be RETAIN in prod to prevent accidental data loss | High |
| Tech debt | Lambda named "algolia" but uses OpenSearch — confusing naming, should be renamed to "search" or "stream-processor" | Med |
| Tech debt | Algolia integration code present but commented out — dead code that should be removed if not intended to be used | Med |
| Architecture | Initial invoice number written via AwsCustomResource — fragile pattern; if the CDK custom resource Lambda fails, the counter isn't initialized | Med |
| Testing | No CDK stack tests — GSI configurations, stream filters, and Cognito triggers are untested | Med |
| Security | Lambda functions at 1024 MB regardless of actual workload — over-provisioned | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/ops-infra/gap-store-manager/cdk/store (commit 9ed4512)*
