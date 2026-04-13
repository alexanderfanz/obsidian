# Tenant Provisioning

New merchants and stores are provisioned dynamically — no manual AWS console work. The [[Store Manager Backend]] drives the entire lifecycle.

## Provisioning flow

```
Operator creates entity in Store Manager UI
  └─▶ api-handler Lambda stores entity in DynamoDB
  └─▶ emits EventBridge: Entity.Added
        └─▶ inventory-manager Lambda
              ├─▶ creates Workflow record (status: REQUESTED)
              ├─▶ updates entity workflowStatus
              └─▶ emits Entity.Deploy.Requested
                    └─▶ actions-manager Lambda
                          ├─▶ starts CodeBuild project
                          ├─▶ (for new Groups) clones Prismic CMS repo
                          ├─▶ (for new Groups) creates Vercel project
                          ├─▶ (for SellTreez) creates OpenSearch indexes/scripts
                          └─▶ copies asset JSON files to S3

CodeBuild runs: cd gap-store-manager/cdk/{entity-type} && cdk deploy
  └─▶ CloudFormation creates/updates AWS resources
  └─▶ CodeBuild state change → EventBridge
        └─▶ build-status-manager Lambda
              ├─▶ updates Workflow status (SUCCEEDED/FAILED)
              ├─▶ updates entity workflowStatus
              └─▶ emits Entity.Deploy.Succeeded/Failed
```

## Entity types provisioned

| Entity | CDK Stack | What gets created |
|--------|-----------|-------------------|
| [[Group]] | [[Group CDK]] | Cognito, 2 API Gateways, 5 Lambdas, SNS/SQS, 3 S3 buckets, CloudFront |
| [[Store]] | [[Store CDK]] | DynamoDB table, Cognito user pool, 2 Lambdas, SES (optional) |
| SellTreez | [[SellTreez CDK]] | OpenSearch domain, 2 Lambdas, EventBridge bus, SQS, API Gateway, DynamoDB |

## Key patterns

- **CodeBuild as CDK runtime** — CDK is not pre-deployed. CodeBuild checks out the `ops-infra` repo and runs `cdk deploy` with per-entity config passed via `ENTITY_CONFIGURATION` env var.
- **Event-driven workflow** — fully async. No synchronous waiting for CloudFormation to complete.
- **Branch-based environments** — all resources namespaced by `GITHUB_BRANCH_NAME`. `develop` and `master` each get isolated infrastructure.
- **CloudFormation outputs as data** — stack outputs (API URLs, Cognito IDs, SNS ARNs) serialized to JSON, stored in Store Manager DynamoDB, and displayed in the UI's "Infra" tab.
- **Bulk updates** — when a new Lambda version is released, operators use bulk update endpoints to roll out the new S3 key across all entities at once.

## Workflow states

`REQUESTED → IN_PROGRESS → SUCCEEDED / FAILED / STOPPED`

Each entity carries a `workflowStatus` field. The [[Store Manager Frontend]] shows this as a badge on every entity card. Operators can see deployment progress without touching AWS.

## Store scaffolding

When a new store is created via `create-store` event, [[e-com-app-worker]] writes 14 default S3 config keys (products, pages, navigation, categories, promotions, tags, brands, vendors, landing pages, taxes, best-selling products) to seed the store with empty-but-valid configuration.
