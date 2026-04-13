# ops-infra: Store Manager Backend

> The control plane API for the Gap Commerce multi-tenant platform. A fully serverless AWS application (API Gateway + 4 Lambda functions + EventBridge) that manages the lifecycle of tenant entities — Groups, Stores, and SellTreez clusters — by storing configuration in DynamoDB and triggering AWS CDK deployments via CodeBuild whenever an entity is created or updated.

---

## Purpose

The backend is the brain of the Store Manager system. It exposes a REST API consumed by the [[ops-infra-store-manager-frontend]] for CRUD operations on Groups, Stores, and SellTreez entities. Beyond simple CRUD, every write triggers an asynchronous infrastructure deployment workflow: CodeBuild checks out this repo and runs `cdk deploy` to provision or update the entity's AWS resources (Cognito, Lambda, API Gateway, SQS, etc.).

Out of scope: the actual deployed tenant infrastructure (that lives in the Group/Store/SellTreez CDK stacks). The backend only manages entity metadata and orchestrates deployments.

---

## Key Entities / Domain Models

### Group
Represents a merchant organization. Key fields:
- `id`, `name`, `description`, `domain`, `stage`
- `bucketArtifactsName` — S3 bucket where compiled Lambda source code lives (written by e-com worker repos)
- `lambdaGraphqlSrcKey`, `lambdaWebHookGroupSrcKey`, `lambdaSearchSrcKey` — S3 keys pointing to Lambda bundles
- `cognitoConfig` — Custom Cognito attributes + roles (`ADMIN`, `SALLER`, `SUPER_ADMIN`)
- `pubsubs` — 3 workers (notification/order/app), each with a Lambda S3 key and SNS filter policy JSON
- `outputs` — Serialized CloudFormation stack outputs (API URLs, Cognito IDs, etc.) stored as JSON string
- `monitor` — Lambda/API/SQS/SNS CloudWatch alarm config
- `workflowStatus`, `workflowId` — Current deployment state
- `legacy` — Backward compat fields for old Cognito pools

### Store
Child of a Group; represents an individual retail location. Key fields:
- `id`, `name`, `domain`, `groupId`, `accountId`
- `cognitoConfig` — Custom attributes + SES email config
- `bucketConfigsStoresGroupName` — Store config S3 bucket name (from parent Group)
- `lambdaAlgoliaSrcKey`, `cognitoLambdaTriggerSrcKey` — Lambda S3 keys
- `topicPubSubArn` — Parent Group's SNS topic ARN (stores subscribe to group pub/sub)
- `outputs` — CloudFormation outputs JSON

### SellTreez
Represents an OpenSearch search cluster for product inventory. Key fields:
- `id`, `name`, `description`, `stage`
- `lambdaReader`, `lambdaWriter` — Lambda config (srcKey, memorySize, timeoutSeconds)
- `events` — EventBridge config (source, detailType, externalAccountId for cross-account publishing)
- `opensearch` — Cluster config (instanceType, dataNodes, volumeSize)
- `domain` — Custom domain config (name, enableCustomDomain)
- `tracking` — TrackingTreez integration enabled flag

### Workflow
Tracks a deployment state machine. Key fields:
- `entityType`, `entityId` — What's being deployed
- `workflowStatus` — REQUESTED / IN_PROGRESS / SUCCEEDED / STOPPED / FAILED
- `startedAt`, `completedAt`, `stoppedAt`
- `buildId`, `buildLogs` — CodeBuild job reference
- `event` — Original EventBridge event payload

### History
Immutable audit log. Fields:
- `entityType`, `entityId` — What changed
- `currentValue`, `newValue` — JSON strings (before/after snapshots)

### AuthConfig
Maps an e-commerce account ID to a group's GraphQL URL. Fields:
- `accountId`, `groupId`, `graphqlUrl`, `graphqlUrlRaw`
- Used by the e-commerce platform to discover the correct GraphQL endpoint per merchant

---

## API Surface

All routes require Cognito JWT authentication except the `/authconfig` routes (which are public).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/group` | List all groups |
| POST | `/group` | Create group (triggers deployment) |
| GET | `/group/last/{n}` | Get last N groups by creation time |
| GET | `/group/{id}` | Get group detail |
| PUT | `/group/{id}` | Update group (triggers deployment) |
| GET | `/group/{id}/history` | Group change history |
| GET | `/group/{id}/workflow` | Current deployment workflow |
| GET | `/group/{id}/accounts` | List accounts in group |
| POST | `/group/{id}/accounts` | Create account |
| GET | `/group/{id}/accounts/{accountId}` | Get account detail |
| PUT | `/group/{id}/accounts/{accountId}` | Update account |
| DELETE | `/group/{id}/accounts/{accountId}` | Delete account |
| POST | `/group/bulk/groups` | Bulk update groups (Lambda keys, Cognito configs) |
| GET | `/store` | List all stores |
| POST | `/store` | Create store (triggers deployment) |
| GET | `/store/last/{n}` | Get last N stores |
| GET | `/store/{id}` | Get store detail |
| PUT | `/store/{id}` | Update store |
| GET | `/store/{id}/history` | Store change history |
| GET | `/store/{id}/workflow` | Current deployment workflow |
| GET | `/store/{id}/stores` | Get stores by group (cross-entity) |
| POST | `/store/bulk/stores` | Bulk update stores |
| GET | `/selltreez` | List all SellTreez clusters |
| POST | `/selltreez` | Create SellTreez cluster (triggers deployment) |
| GET | `/selltreez/{id}` | Get SellTreez detail |
| PUT | `/selltreez/{id}` | Update SellTreez |
| GET | `/selltreez/{id}/history` | SellTreez change history |
| GET | `/selltreez/{id}/workflow` | Deployment workflow |
| GET | `/selltreez/{id}/indexes` | List OpenSearch indexes |
| POST | `/selltreez/{id}/indexes` | Create index |
| GET | `/selltreez/{id}/indexes/{name}` | Get index |
| DELETE | `/selltreez/{id}/indexes/{name}` | Delete index |
| GET | `/selltreez/{id}/indexes/{name}/mapping` | Get index mapping |
| PUT | `/selltreez/{id}/indexes` | Update index mapping |
| POST | `/selltreez/{id}/indexes/{name}/clone/{target}` | Clone index |
| PUT | `/selltreez/{id}/scripts` | Upsert OpenSearch script |
| GET | `/selltreez/{id}/scripts/{scriptId}` | Get script |
| DELETE | `/selltreez/{id}/scripts/{scriptId}` | Delete script |
| GET | `/history` | List all history records |
| GET | `/history/{id}` | Get history for entity |
| GET | `/artifacts` | List Lambda artifact folders in S3 |
| GET | `/authconfig/{ecommAccountId}` | Get GraphQL URL for account (public) |
| POST | `/authconfig/{ecommAccountId}` | Upsert auth config (public) |

---

## Data Layer

**DynamoDB — single-table design**
- Table name: `{branchName}-table`
- Billing: PAY_PER_REQUEST
- PK: entity type string (e.g. `"GapGroup"`, `"GapStore"`)
- SK: entity type + UUID (e.g. `"GapGroup#550e8400-..."`)
- GSI1: PK=`GSI1_PK` / SK=`GSI1_SK` — enables entity-type list queries
- GSI2: PK=`GSI1_PK` / SK=`_ct` (creation timestamp) — enables `GET /last/{n}` queries
- All entity types coexist: Group, Store, SellTreez, Workflow, History, AuthConfig
- Abstraction layer: `dynamodb-toolbox@1.0.2` Entity classes map PK/SK to domain models

**S3 — two buckets**
- `{branchName}-frontend-bucket` — CloudFront-served SPA static assets
- `{branchName}-source-bucket` — Lambda source code artifacts uploaded by e-com repos

**No relational DB, no caching layer.** DynamoDB is used for all reads. The 5-second polling from the frontend is mitigated by the fact that DynamoDB reads are cheap at this scale.

---

## Core Business Logic

### Entity Deployment Workflow
The core complexity isn't CRUD — it's the async infrastructure deployment triggered by every write:

```
POST /group (or /store, /selltreez)
  └─▶ api-handler stores entity in DynamoDB
  └─▶ emits EventBridge: Entity.Added / Entity.Edited
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

CodeBuild runs: `cd gap-store-manager/cdk/{entity-type} && cdk deploy`
  └─▶ CodeBuild state change → EventBridge
        └─▶ build-status-manager Lambda
              ├─▶ fetches Workflow by buildId
              ├─▶ updates Workflow status (SUCCEEDED/FAILED)
              ├─▶ updates entity workflowStatus (rollback if failed)
              └─▶ emits Entity.Deploy.Succeeded / Failed
```

### AuthConfig Lookup
When the e-commerce platform boots for a merchant, it calls `GET /authconfig/{ecommAccountId}` (public, no auth) to discover its GraphQL endpoint. The backend does a reverse lookup: account → group → GraphQL URL from group outputs.

### Bulk Operations
`POST /group/bulk/groups` and `POST /store/bulk/stores` allow updating Lambda S3 source keys and Cognito configs across all entities in one call — used when a new Lambda version is deployed and needs to be rolled out across all tenants.

---

## Cross-Repo Connections

**Calls:**
- AWS CodeBuild — starts build jobs, reads build state
- AWS EventBridge — publishes/subscribes to entity lifecycle events
- OpenSearch Serverless — direct API calls for index and script management (SellTreez controller)
- Vercel API — creates projects and deployments for new groups
- Prismic CMS API — clones content repos for new groups (assets, migration, auth endpoints)
- S3 — reads/writes Lambda artifacts and config files

**Called by:**
- [[ops-infra-store-manager-frontend]] — primary consumer of all API routes
- e-com platform — calls `/authconfig/{ecommAccountId}` at boot time

**Events published (EventBridge):**
- `Entity.Added`, `Entity.Edited` — on successful write
- `Entity.Deploy.Requested` — triggers CodeBuild
- `Entity.Deploy.Succeeded`, `Entity.Deploy.Failed` — after build completes

**Events consumed:**
- `aws.codebuild` / Build State Change — CodeBuild → build-status-manager

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| Vercel | Infrastructure | Deploying group SPAs; creating projects via Vercel API |
| Prismic | CMS | Cloning CMS content repos for new groups (headless CMS) |
| OpenSearch (AWS Serverless) | Search | Index and script management for SellTreez |
| Sentry | Monitoring | Error tracking (DSN stored in Secrets Manager) |
| Algolia | Search | Referenced in secrets and defaults — partially implemented, appears dormant |

---

## Notable Patterns & Decisions

1. **Decorator-based routing without a framework** — `@Get`, `@Post`, `@Put`, `@Delete` decorators use `reflect-metadata` to register routes on controller classes. The main Lambda handler iterates controllers and matches the incoming HTTP method + path. No Express, Fastify, or similar — just TypeScript decorators and a manual dispatcher.

2. **CodeBuild as CDK runtime** — Tenant infrastructure is not pre-provisioned. Instead, CodeBuild clones this repo at runtime and runs `cdk deploy` per entity. This means tenant isolation costs one CloudFormation stack per entity, but zero pre-allocation of AWS resources.

3. **Single-table DynamoDB** — All five entity types share one table. This simplifies IAM (one table ARN), reduces costs, and enables cross-entity queries. The trade-off is that querying e.g. "all stores by group" requires a GSI scan rather than a simple table query.

4. **Branch-namespaced resources** — All resource names include `{branchName}`. `develop` and `master` branches each have isolated DynamoDB tables, event buses, S3 buckets, etc. This allows staging to be a true replica of prod infra.

5. **`ENABLE_CODEBUILD` feature flag** — The `actions-manager` Lambda checks an env var before starting CodeBuild. In some environments this is set to `'false'` to allow testing the workflow without actually deploying infrastructure.

6. **Zod for runtime schema validation** — All entity shapes are defined as Zod schemas, providing both TypeScript types and runtime validation. This catches malformed API payloads before they reach DynamoDB.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Testing | No unit or integration tests for Lambda handlers — complex orchestration logic is untested | High |
| Security | CodeBuild role has AdministratorAccess — should be scoped to only CDK/CloudFormation actions | Med |
| Architecture | `actions-manager` has 10-minute timeout; if Vercel/Prismic calls are slow, the Lambda is blocking for the full duration | Med |
| Tech debt | Algolia integration is present in code and secrets but appears commented out in store stack — unclear if it should be removed or completed | Med |
| Observability | No structured logging format — log statements are ad-hoc; harder to query in CloudWatch Logs Insights | Med |
| DX | No OpenAPI spec — API routes are only discoverable by reading controller source code or the Postman collection | Low |
| Architecture | `authconfig` routes are public (no Cognito auth) — relies on the obscurity of the ecommAccountId; consider rate limiting | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/ops-infra/gap-store-manager/backend (commit 9ed4512)*
