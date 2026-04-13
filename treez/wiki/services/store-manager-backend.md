# Store Manager Backend

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Control plane API for the GapCommerce multi-tenant platform. Manages the lifecycle of Groups, Stores, and SellTreez clusters.

**Repo:** [gap-commerce/ops-infra/gap-store-manager/backend](https://github.com/gap-commerce/ops-infra/tree/master/gap-store-manager/backend) | **Language:** TypeScript | **Runtime:** 4 AWS Lambdas + API Gateway | **Part of:** [[ops-infra]]

## Lambda functions

| Lambda | Role |
|--------|------|
| `api-handler` | REST API for CRUD on Groups, Stores, SellTreez, AuthConfig |
| `inventory-manager` | Reacts to entity events; creates Workflow records, triggers deploy |
| `actions-manager` | Starts CodeBuild, clones Prismic CMS, creates Vercel projects, copies S3 assets |
| `build-status-manager` | Reacts to CodeBuild completion; updates Workflow and entity status |

## Key APIs

Groups, Stores, and SellTreez each have full CRUD + history + workflow endpoints. Bulk update endpoints enable rolling out Lambda S3 keys across all entities at once. `GET /authconfig/{ecommAccountId}` (public, no auth) enables the e-commerce platform to discover its GraphQL endpoint.

## Key behaviors

- **Entity deployment workflow** — every write triggers async infrastructure deployment via EventBridge → CodeBuild → CDK. See [[Tenant Provisioning]].
- **Single-table DynamoDB** — all entity types share one table with PK/SK pattern + 2 GSIs.
- **Branch-namespaced resources** — isolated per git branch.
- **Zod schema validation** — all entity shapes validated at runtime.
- **Decorator-based routing** — `@Get`, `@Post` decorators via reflect-metadata. No Express or Fastify.

## Dependencies

- Triggers [[Group CDK]], [[Store CDK]], [[SellTreez CDK]] via CodeBuild
- Called by [[Store Manager Frontend]]
- Vercel API, Prismic CMS API, OpenSearch Serverless API

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Testing | No unit or integration tests | High |
| Security | CodeBuild AdministratorAccess | Med |
| Architecture | `actions-manager` 10-min timeout blocks on Vercel/Prismic | Med |
| Auth | `authconfig` routes are public — no rate limiting | Low |

*Source: [[raw/ops-infra-store-manager-backend.md]]*