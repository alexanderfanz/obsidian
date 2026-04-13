# ops-infra

Infrastructure-as-code monorepo — the single source of truth for how the GapCommerce multi-tenant platform is provisioned and operated.

**Repo:** [gap-commerce/ops-infra](https://github.com/gap-commerce/ops-infra) | **Language:** TypeScript | **IaC:** AWS CDK | **Type:** Monorepo

## Sub-components

| Component | Description | Wiki page |
|-----------|-------------|-----------|
| Store Manager Backend | Control plane API (4 Lambdas + API Gateway) | [[Store Manager Backend]] |
| Store Manager Frontend | React SPA for operators | [[Store Manager Frontend]] |
| Group CDK | Per-group infrastructure | [[Group CDK]] |
| Store CDK | Per-store infrastructure | [[Store CDK]] |
| SellTreez CDK | Per-search-cluster infrastructure | [[SellTreez CDK]] |
| TrackingTreez CDK | Shared analytics data lake | [[TrackingTreez CDK]] |
| Monitoring & Alerting | CloudWatch → Discord/PagerDuty | (inline) |
| Access Control | GitHub Actions OIDC + IAM Identity Center | (inline) |

## Architecture

```
Store Manager UI → Backend API → EventBridge → CodeBuild → cdk deploy
                                                    ↓
                                    Group / Store / SellTreez stacks
```

All tenant infrastructure is **deployed dynamically at runtime**. Operator creates entity in Store Manager → API stores in DynamoDB + emits EventBridge event → `actions-manager` Lambda starts CodeBuild → CDK deploys → result flows back via EventBridge → `build-status-manager` updates status.

## Key patterns

- **CodeBuild as CDK runtime** — same stack code deployed N times per entity, parameterized by config JSON.
- **Single DynamoDB table** — all entity types (Group, Store, SellTreez, Workflow, History, AuthConfig) in one table with PK/SK pattern.
- **Branch-based environment isolation** — every resource namespaced by `GITHUB_BRANCH_NAME`.
- **OIDC over static credentials** — all GitHub Actions use short-lived OIDC tokens.
- **Event-driven workflow** — fully async deployment pipeline, no synchronous waiting.

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Testing | No test coverage in any CDK stack or Lambda handler | High |
| Security | CodeBuild has AdministratorAccess — should be scoped down | Med |
| Tech debt | `aws-iam-indentity-center` dir has a typo | Low |
| Tech debt | Algolia integration partially commented out | Med |

*Source: [[raw/ops-infra.md]]*
