# ops-infra

> Infrastructure-as-code monorepo for the Gap Commerce e-commerce platform. Manages all AWS infrastructure across environments — from the Store Manager control plane (frontend, API, entity CDK stacks) to cross-cutting concerns like monitoring, alerting, SSO access control, and CI/CD authentication. The repo is the single source of truth for how the platform's multi-tenant store and group infrastructure is provisioned and operated.

---

## Purpose

This repo solves the problem of managing a multi-tenant e-commerce platform where each merchant "group" and "store" needs its own isolated AWS infrastructure (Cognito pools, Lambda functions, API Gateway, SQS queues, etc.). It provides:

- A **control plane** (Store Manager) where operators create/configure groups and stores, triggering automated AWS CDK deployments per tenant.
- **Monitoring infrastructure** that observes all platform Lambda/API/SQS/DynamoDB resources and routes alarms to Discord and PagerDuty.
- **Access control infrastructure** for both humans (SSO via IAM Identity Center) and machines (GitHub Actions OIDC).
- **Analytics infrastructure** (TrackingTreez) for search event tracking via Kinesis Firehose → S3 → Athena/Glue.

Out of scope: the actual e-commerce application code (order processing, product catalog, payments) — those live in separate repos (api-srv, e-com-srv, etc.).

---

## Sub-Components

This repo contains several independently deployable stacks. Each has its own detailed doc:

| Component | Path | Doc |
|-----------|------|-----|
| Store Manager Backend (control plane API) | `gap-store-manager/backend/` | [[ops-infra-store-manager-backend]] |
| Store Manager Frontend (React SPA) | `gap-store-manager/frontend/` | [[ops-infra-store-manager-frontend]] |
| Group CDK Stack (per-group infra) | `gap-store-manager/cdk/group/` | [[ops-infra-group-cdk]] |
| Store CDK Stack (per-store infra) | `gap-store-manager/cdk/store/` | [[ops-infra-store-cdk]] |
| SellTreez CDK Stack (OpenSearch search) | `gap-store-manager/cdk/selltreez/` | [[ops-infra-selltreez-cdk]] |
| TrackingTreez CDK Stack (analytics lake) | `gap-store-manager/cdk/trackingtreez/` | [[ops-infra-trackingtreez-cdk]] |
| Monitoring & Alerting | `aws-metrics-alarms/`, `monitoring-notification/` | (below) |
| Access Control | `aws-github-openid/`, `aws-iam-indentity-center/` | (below) |
| PagerDuty | `pagerduty/` | (below) |
| New Relic | `newrelic/` | (below) |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Store Manager Control Plane               │
│                                                              │
│  ┌─────────────┐   ┌──────────────────────────────────────┐ │
│  │  Frontend   │   │  Backend (API Gateway + 4 Lambdas)   │ │
│  │  React SPA  │──▶│  api-handler  inventory-manager       │ │
│  │  (Vite)     │   │  actions-manager  build-status-mgr   │ │
│  └─────────────┘   └──────────────┬───────────────────────┘ │
│                                   │ EventBridge              │
│                                   ▼                          │
│                          ┌────────────────┐                  │
│                          │  CodeBuild     │                  │
│                          │  (CDK deploy)  │                  │
│                          └────────┬───────┘                  │
└───────────────────────────────────┼─────────────────────────┘
                                    │ deploys
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Group Stack     Store Stack    SellTreez Stack
              (per tenant)    (per store)   (per search cluster)
                    │
                    └──────────────▶ TrackingTreez Stack
                                     (analytics lake)

Monitoring: CloudWatch Alarms → SNS → Lambda → Discord / PagerDuty
Access:     OIDC (machines) + IAM Identity Center (humans)
```

---

## Key Entities / Domain Models

- **Group** — Top-level multi-tenant entity representing a merchant organization. Has its own AWS infrastructure: Cognito user pool, API Gateway (GraphQL + webhook), Lambda workers (order/notification/app), SNS topic, SQS queues, S3 buckets for config.
- **Store** — Child of a Group, representing an individual dispensary/retail location. Has its own Cognito user pool, DynamoDB table for store entities, Lambda triggers.
- **Account** — Bridges a Group to an e-commerce account ID (foreign key to the e-com platform). A Group can have many Accounts; each Account has many Stores.
- **SellTreez** — Represents an OpenSearch-backed search cluster for product inventory. Contains reader + writer Lambdas, an EventBridge bus, SQS queue, and optional DLQ. Independent of groups/stores.
- **TrackingTreez** — Analytics data lake stack. Receives search events from SellTreez via Kinesis Firehose, stores Parquet in S3, queryable via Athena.
- **Workflow** — Tracks the deployment state machine for a Group/Store/SellTreez. States: REQUESTED → IN_PROGRESS → SUCCEEDED/FAILED/STOPPED.
- **History** — Immutable audit log entry for every entity change. Stores old + new value as JSON snapshots.

---

## Deployment Model

All tenant infrastructure is **deployed dynamically at runtime**:
1. Operator creates/edits a Group or Store via the Store Manager UI.
2. Backend API stores the entity in DynamoDB and emits an EventBridge event.
3. `actions-manager` Lambda starts a CodeBuild job.
4. CodeBuild checks out this `ops-infra` repo, `cd`s into the appropriate CDK stack directory, and runs `cdk deploy`.
5. CloudFormation creates/updates the tenant's AWS resources.
6. Build result flows back via EventBridge → `build-status-manager` Lambda.

This means **tenant infrastructure = data in DynamoDB + live CDK deployments**. The Store Manager is a control plane that drives infrastructure.

---

## Cross-Repo Connections

**Calls:**
- `api-srv` — The Gap Commerce GraphQL API. Store Manager provisions Cognito pools that `api-srv` uses for auth.
- `e-com-app-worker`, `e-com-order-worker`, `e-com-notification-worker` — Workers deployed into Group stacks. Source code is stored in S3 by these repos; Store Manager references S3 keys.
- `selltreez-api-srv`, `selltreez-injection-srv` — Lambda source code deployed into SellTreez stacks.

**Called by:**
- All e-com repos use the GitHub OIDC role provisioned here for CI/CD deployments.

**Events:**
- Publishes EventBridge events on entity lifecycle: `Entity.Added`, `Entity.Edited`, `Entity.Deploy.Requested`, `Entity.Deploy.Succeeded`, `Entity.Deploy.Failed`
- Subscribes to CodeBuild state change events to update Workflow status

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| GitHub Actions OIDC | Auth / CI | Keyless AWS access for all Gap Commerce repos |
| Vercel | Infrastructure | Frontend SPA deployments for store/group web apps |
| Prismic | CMS | Headless CMS repo cloning for new groups |
| Algolia | Search | Alternative search index (referenced in secrets, partially implemented) |
| OpenSearch (AWS) | Search | Primary search backend for SellTreez |
| Snowflake | Analytics | External S3 integration for reading TrackingTreez data lake |
| PagerDuty | Monitoring | Incident management with on-call rotation |
| Discord | Monitoring | Team alarm notifications |
| New Relic | Monitoring | Lambda function observability |
| Sentry | Monitoring | Application error tracking (DSN in secrets) |

---

## Notable Patterns & Decisions

1. **CodeBuild as a CDK runtime** — Rather than pre-deploying tenant infrastructure into fixed templates, CodeBuild dynamically runs `cdk deploy` with per-entity config. This lets the same CDK stack code serve all tenants without a separate deployment pipeline per tenant.

2. **Single DynamoDB table (single-table design)** — All backend entities (Group, Store, SellTreez, Workflow, History, AuthConfig) live in one table with PK/SK pattern. GSI1 enables entity-type queries; GSI2 enables recency sorting.

3. **Branch-based environment isolation** — Every resource is namespaced by `GITHUB_BRANCH_NAME`. A `develop` branch gets its own table, event bus, API Gateway, etc. This makes ephemeral environments trivial.

4. **Decorator-based routing** — The API Lambda uses reflect-metadata decorators (`@Get`, `@Post`, etc.) for controller routing — a mini framework without importing Express or similar.

5. **OIDC over static credentials** — All GitHub Actions use short-lived OIDC tokens exchanged for IAM role credentials. No AWS access keys stored anywhere.

6. **Event-driven workflow** — The deployment pipeline is fully asynchronous: API → EventBridge → Lambda → CodeBuild → EventBridge → Lambda. No synchronous waiting.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Tech debt | `aws-iam-indentity-center` has a typo in the dir name (indentity vs identity) | Low |
| Architecture | CodeBuild jobs have AdministratorAccess — could be scoped down per entity type | Med |
| DX | No local development environment documented; CodeBuild deployments can't be tested locally | Med |
| Observability | New Relic integration exists but no doc on what dashboards/alerts are configured there | Low |
| Testing | No test coverage visible in any CDK stack or Lambda handler | High |
| Tech debt | Algolia integration is present in secrets and store stack but appears partially commented out | Med |
| Architecture | Frontend polling interval (5s for groups) could cause unnecessary API load at scale | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/ops-infra (commit 9ed4512)*
