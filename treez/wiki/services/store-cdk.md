# Store CDK

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

AWS CDK stack provisioning infrastructure for a single retail Store within a Group.

**Repo:** [gap-commerce/ops-infra/gap-store-manager/cdk/store](https://github.com/gap-commerce/ops-infra/tree/master/gap-store-manager/cdk/store) | **Language:** TypeScript | **IaC:** AWS CDK | **Part of:** [[ops-infra]]

## Resources created per Store

- **DynamoDB table** — `entity_id` (PK) + `key` (SK). PAY_PER_REQUEST. Streams enabled (NEW_IMAGE). 3 GSIs: status-index, email-index, key-created_at-index. Point-in-time recovery.
- **Cognito User Pool** — per-store for customers. Custom attributes (ADDRESS, CITY, BERBIX fields). Lambda triggers for pre-signup and post-confirmation.
- **2 Lambda functions** — Cognito trigger (from [[e-com-cognito-worker]]) and DynamoDB stream processor (writes to OpenSearch/Algolia)
- **SES email identity** (optional) with DKIM Route 53 records
- **Initial invoice number** seeded via AwsCustomResource at deploy time

## Key patterns

- **DynamoDB stream → search index** — items where `key` starts with `d#` are streamed to OpenSearch for near-real-time indexing.
- **Store depends on Group** — references parent group's SNS topic ARN and S3 config bucket.
- **DESTROY retention on Cognito** — appropriate for dev; risky for prod (user accounts lost on stack delete).

## Cross-repo connections

- Lambda source code from: [[e-com-cognito-worker]] (Cognito trigger), stream processor (DynamoDB → OpenSearch)
- Parent: [[Group CDK]] (provides SNS topic, S3 config bucket)
- Deployed by: [[Store Manager Backend]] via CodeBuild
- DynamoDB table read by: [[e-com-srv]] (via pattern-matched IAM from Group stack)

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Safety | Cognito DESTROY removal policy in prod | High |
| Tech debt | Lambda named "algolia" but uses OpenSearch | Med |
| Tech debt | Algolia code present but commented out | Med |
| Testing | No CDK stack tests | Med |

*Source: [[raw/ops-infra-store-cdk.md]]*