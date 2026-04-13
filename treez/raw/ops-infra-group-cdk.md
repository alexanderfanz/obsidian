# ops-infra: Group CDK Stack

> AWS CDK stack that provisions all infrastructure for a single merchant Group (a tenant organization). Deployed dynamically at runtime by CodeBuild whenever a Group entity is created or updated via the Store Manager. Creates Cognito authentication, API Gateways (GraphQL + Webhook), five Lambda workers (order/notification/app/graphql/webhook), SNS pub/sub, SQS queues, and S3 buckets — all namespaced to the group.

---

## Purpose

Every merchant Group in the Gap Commerce platform gets its own isolated AWS infrastructure. This CDK stack is the blueprint for that infrastructure. It is not deployed once — it is deployed N times (once per group) by CodeBuild at runtime, parameterized by the group's configuration JSON.

The group stack is the top of the tenant hierarchy. Store stacks (child stacks) reference resources created here (e.g. the SNS topic ARN, the S3 config bucket name).

Out of scope: per-store resources (Cognito, DynamoDB) are in the [[ops-infra-store-cdk]].

---

## Key Entities / Domain Models

**GroupStack** takes the following configuration at deploy time (passed via CodeBuild env var `ENTITY_CONFIGURATION`):
- `id` — Unique group identifier (used as stack/resource name prefix)
- `domain` — Custom domain for API endpoints
- `stage` — Environment (prod/dev)
- `cognitoConfig` — Custom Cognito attributes and role definitions
- `pubsubs` — Three PubSub worker configs (notification, order, app), each with `name`, `lambdaSrcKey`, `filterPolicyJson`
- `monitor` — CloudWatch alarm config (Lambda thresholds, API thresholds, SQS thresholds)
- `bucketArtifactsName` — Source S3 bucket for Lambda code
- `lambdaGraphqlSrcKey`, `lambdaWebHookGroupSrcKey`, `lambdaSearchSrcKey` — S3 keys for Lambda bundles

---

## AWS Resources Created (per Group)

### Lambda Functions (5)
| Name | Source | Memory | Timeout | Trigger |
|------|--------|--------|---------|---------|
| `{prefix}-graphql` | `lambdaGraphqlSrcKey` | 1024 MB | 300s | API Gateway |
| `{prefix}-webhook` | `lambdaWebHookGroupSrcKey` | 1024 MB | 300s | API Gateway |
| `{prefix}-order-worker` | `pubsubs[order].lambdaSrcKey` | 1024 MB | 300s | SQS |
| `{prefix}-app-worker` | `pubsubs[app].lambdaSrcKey` | 1024 MB | 300s | SQS |
| `{prefix}-notification-worker` | `pubsubs[notification].lambdaSrcKey` | 1024 MB | 300s | SQS |

All Lambdas are Node.js 20.x, loaded from S3 source bucket.

### API Gateways (2 REST APIs)
- **GraphQL API** — `{id}-ag.{domain}` — Routes all requests to `lambdaGraphql`
- **WebHook API** — `{id}-webhook-ag.{domain}` — Routes all requests to `lambdaWebHookGroup`

Both use custom domains with ACM certificates.

### SNS + SQS Pub/Sub (per worker)
For each of the 3 workers (order, notification, app):
- **SNS Subscription** — Subscribes the SQS queue to the group's SNS topic with a filter policy (from `filterPolicyJson`)
- **SQS Queue** — `{prefix}-{worker}-queue` — 5-day retention, 300s visibility timeout
- **SQS DLQ** — `{prefix}-{worker}-dlq` — 5-day retention
- **Lambda Event Source** — SQS queue triggers the worker Lambda (batch size: 1, max retry: 5)

### Cognito User Pool
- Name: `{prefix}-group-stack-cup`
- Custom attributes: defined by `cognitoConfig.customAttributes` (e.g. ACCOUNT_ID, STORE_ID)
- Cognito Groups: created per `cognitoConfig.roles` (e.g. ADMIN, SALLER, SUPER_ADMIN)
- Email-based sign-in, password policy enforcement
- No third-party IdP

### S3 Buckets (3)
- **Config Stores Bucket** — `{prefix}-group-stack-bucket-config-stores` — Group-level store configuration files
- **Account Photos Bucket** — `{prefix}-group-stack-bucket-config-account-photos` — Merchant account photos; CORS enabled for browser uploads
- **CDN Bucket** — Private, served via CloudFront distribution

### CloudFront Distribution
- Origin: CDN bucket
- Used for group-level static asset delivery

### OpenSearch Serverless Collection (optional)
- Created if the group's config enables it
- Provides a search backend for the GraphQL Lambda

---

## Data Layer

No DynamoDB table at the group level. Data lives in:
- S3 buckets — configuration files, account photos, static assets
- The parent Store Manager's DynamoDB table — Group entity metadata
- Cognito — user identities and group membership
- OpenSearch Serverless (optional) — search data

---

## Core Business Logic

### SNS Filter Policies
Each worker Lambda (order/notification/app) subscribes to the group SNS topic with a filter policy defined by `filterPolicyJson`. This means a single SNS topic fans out to three different processing pipelines based on message attributes. The filter policy JSON is authored by operators in the Store Manager UI.

### DynamoDB Access (Legacy Support)
The GraphQL Lambda needs access to DynamoDB tables for multiple stores within the group. The group stack grants DynamoDB access via a pattern-matched IAM policy:
- Policy allows access to tables matching `{stage}-*-store-stack-table*`
- This avoids hard-coding individual table ARNs; new stores automatically fall within the pattern

### SES Email Integration (optional)
If the group config enables SES, the stack creates email identity and DKIM records in Route 53. Used by notification workers to send transactional emails from the group's domain.

---

## Cross-Repo Connections

**Calls:**
- `e-com-app-worker`, `e-com-order-worker`, `e-com-notification-worker` — Lambda source code is uploaded to S3 by these repos; the group stack reads S3 keys from group config
- `api-srv` (graphql) — The GraphQL Lambda deployed here is the `api-srv` bundle
- `e-com-srv` (webhook) — The webhook Lambda is the `e-com-srv` bundle

**Called by:**
- [[ops-infra-store-manager-backend]] — `actions-manager` Lambda triggers CodeBuild to deploy this stack
- [[ops-infra-store-cdk]] — Store stacks reference the SNS topic ARN and S3 config bucket from this stack

**Shares:**
- Secrets Manager path: `/gap-store-manager/allSecrets` — same secret read by all CDK stacks
- S3 artifacts bucket — written by e-com repos, read by this stack to get Lambda bundles

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS SES | Email | Transactional email for notification workers (optional, per-group) |
| CloudWatch (cdk-monitoring-constructs) | Monitoring | Lambda invocation, error, latency alarms; SQS queue depth alarms |

---

## Notable Patterns & Decisions

1. **Pattern-matched DynamoDB IAM policy** — Rather than enumerating store table ARNs (which would require updating the group stack every time a store is added), the IAM policy uses a wildcard pattern `{stage}-*-store-stack-table*`. New stores get access automatically. This is a deliberate trade-off: broader permissions in exchange for zero-touch scaling.

2. **All workers at 1024 MB / 300s** — All Lambda functions are provisioned at high memory and long timeout regardless of typical load. These are shared across all stores in the group, so over-provisioning protects against burst traffic.

3. **SNS fan-out pattern** — A single SNS topic serves all inter-service communication for the group. Workers subscribe with filter policies, so they only receive relevant messages. This allows adding new workers without changing the publisher.

4. **Stack deployed N times** — The CDK stack code is parameterized via environment variables injected by CodeBuild. The same TypeScript file deploys infrastructure for every group — tenant isolation via CloudFormation stacks, not via separate code.

5. **CloudFormation outputs as entity data** — Stack outputs (API URLs, Cognito IDs, SNS ARN, etc.) are serialized to JSON and stored back in the Store Manager's DynamoDB table as the `outputs` field. This is how the frontend's "Infra" tab gets its data.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Security | DynamoDB IAM policy uses `*` wildcard — grants access to all stage-matching tables, not just the group's stores | Med |
| Cost | All Lambdas at 1024 MB even for simple event processing — could right-size based on worker type | Low |
| Testing | No tests for the CDK stack — stack synthesis and resource configuration are untested | Med |
| Architecture | SNS filter policy JSON is stored as a raw string in the DB — no validation that it's valid SNS filter syntax | Low |
| DX | Stack must be deployed via CodeBuild — no way to run `cdk deploy` locally for a specific group without replicating the entire env setup | Med |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/ops-infra/gap-store-manager/cdk/group (commit 9ed4512)*
