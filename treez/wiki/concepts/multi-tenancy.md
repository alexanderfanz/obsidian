# Multi-Tenancy

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Every piece of the GapCommerce platform is multi-tenant. A single deployment serves all merchants. Tenant isolation is achieved at the data level, not the code level.

## Tenant hierarchy

```
Group (merchant organization) — owns compute infrastructure
  └── Account (e-commerce account ID) — bridge entity
        └── Store (individual dispensary location) — owns data infrastructure
```

## Isolation mechanisms

### Data isolation

| Layer | Isolation strategy |
|-------|-------------------|
| DynamoDB | Per-store table. Table name resolved at runtime from store config (`DynamoOrderTableName`). |
| S3 | Composite key: `{accountID}-{storeID}-{keyName}` or `{accountID}/{storeID}/{resource}`. |
| OpenSearch | Per-tenant index: `{prefix}_{orgId}_{entityId}`. |
| Cognito | Per-store user pool (customers). Per-group user pool (admins). |

### Infrastructure isolation

| Layer | Isolation strategy |
|-------|-------------------|
| CloudFormation | One stack per Group ([[Group CDK]]) and one per Store ([[Store CDK]]). |
| Lambda | Shared across stores within a group, but scoped via event payload `(accountID, storeID)`. |
| API Gateway | Per-group with custom domain. |
| SNS/SQS | Per-group topic; workers subscribe with filter policies. |
| S3 buckets | Per-group config bucket; stores share it with scoped keys. |

### Request scoping

Every request carries `accountID` and `storeID`:
- **GraphQL** ([[e-com-srv]]) — extracted from auth middleware (Cognito JWT claims)
- **Webhooks** ([[api-srv]]) — passed as query parameters
- **SQS events** (all workers) — embedded in event payload
- **REST** ([[selltreez-api-srv]]) — `Entity-Id` and `org-id` request headers

## Pattern: single code, N deployments

The CDK stacks are parameterized via environment variables (`ENTITY_CONFIGURATION`). CodeBuild deploys the same CDK code for every group/store with different configuration. This means:
- Tenant isolation via CloudFormation stacks, not separate code
- Adding a new merchant = one Store Manager API call → automatic infrastructure provisioning
- No pre-allocation of AWS resources
- Stack outputs (API URLs, Cognito IDs) serialized back to Store Manager DynamoDB

## Pattern: shared workers, scoped data

Worker Lambdas ([[e-com-order-worker]], [[e-com-app-worker]], [[e-com-notification-worker]]) are deployed once per group but handle events for all stores in that group. They:
1. Extract `(accountID, storeID)` from the SQS event
2. Load store config from S3
3. Resolve the correct DynamoDB table name
4. Execute business logic within that scope

This avoids per-store Lambda deployments while maintaining data isolation.

## Pattern: DynamoDB IAM wildcards

The [[Group CDK]] grants DynamoDB access via `{stage}-*-store-stack-table*` pattern. New stores automatically fall within the pattern — no group stack update needed when stores are added. Trade-off: broader permissions than strictly necessary.