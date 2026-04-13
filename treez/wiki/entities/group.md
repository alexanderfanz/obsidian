# Group

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Top-level multi-tenant entity representing a merchant organization. The parent of [[Store]]s.

## Where it lives

| System | Purpose |
|--------|---------|
| DynamoDB (Store Manager) | Entity metadata, deployment config, CloudFormation outputs. |
| [[Group CDK]] | Isolated AWS infrastructure per group. |
| [[Store Manager Backend]] / [[Store Manager Frontend]] | CRUD and deployment management. |

## Key fields

`id`, `name`, `domain`, `stage`, `bucketArtifactsName` (S3 bucket for Lambda code), Lambda S3 keys (graphql, webhook, search), `cognitoConfig` (custom attributes, roles: ADMIN, SALLER, SUPER_ADMIN), `pubsubs` (3 workers with filter policies), `outputs` (CloudFormation stack outputs as JSON), `monitor` (CloudWatch alarm config), `workflowStatus`.

## Infrastructure per group

The [[Group CDK]] creates:
- 5 Lambda functions (graphql, webhook, order-worker, app-worker, notification-worker)
- 2 API Gateways (GraphQL + Webhook) with custom domains
- SNS topic + 3 SQS queues with DLQs and filter policies
- Cognito User Pool (merchant admin auth)
- 3 S3 buckets (config, account photos, CDN)
- CloudFront distribution
- Optional: OpenSearch Serverless, SES email identity

## Relationship to other entities

```
Group
  ├── Cognito (admin users)
  ├── API Gateways (GraphQL, Webhook)
  ├── SNS Topic (shared by all stores in group)
  ├── Worker Lambdas (shared by all stores)
  └── Account(s)
        └── Store(s) → own DynamoDB table, own Cognito pool
```

The Group provides the **compute layer** (Lambdas, APIs). Stores within the group provide the **data layer** (DynamoDB tables, customer Cognito pools). Workers are shared across all stores in the group — they identify the correct store via `(accountID, storeID)` in each event.

## AuthConfig

The `AuthConfig` entity maps an e-commerce account ID to a group's GraphQL URL. When the storefront boots, it calls `GET /authconfig/{ecommAccountId}` (public endpoint) to discover its GraphQL endpoint.