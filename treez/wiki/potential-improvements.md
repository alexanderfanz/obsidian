# Potential Improvements

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Consolidated view of issues and improvement opportunities spotted across all repos. Organized by priority, then by theme.

## High Priority

| Repo | Area | Issue |
|------|------|-------|
| [[e-com-srv]] | Architecture | **TreezPay duplicate ticket race condition** — rapid duplicate checkout creates multiple Treez tickets. Needs idempotency lock or conditional write. Documented in `docs/TREEZ_DUPLICATE_TICKET_ENTITY_ID_BUG.md`. |
| [[e-com-order-worker]] | Testing | Most test files skipped (`t.Skip()`). Pipeline effectively untested in CI. |
| [[e-com-cognito-worker]] | Testing | No test files exist despite injection-friendly architecture. |
| [[e-com-cognito-worker]] | Error handling | CRM subscription failures silently absorbed — no retry or DLQ for missed subscriptions. |
| [[e-com-cognito-worker]] | DX | `.env.*` files with non-prod credentials committed to repo. |
| [[ops-infra]] | Testing | No test coverage in any CDK stack or Lambda handler. |
| [[Store Manager Backend]] | Testing | No unit or integration tests for Lambda handlers. |
| [[gap-dashboard]] | Tech debt | `config/constant.js` is 75 KB with no ownership — needs domain splitting. |
| [[gap-dashboard]] | Tech debt | CRA is unmaintained — needs Vite/Next.js migration. |
| [[gap-dashboard]] | Testing | Near-zero test coverage despite complex business logic. |
| [[treez-tlp]] | Security | Sensitive keys prefixed `NEXT_PUBLIC_` — exposed to browser bundle. |
| [[treez-tlp]] | Testing | 48 hooks + 161 components but light test coverage. |
| [[e-com-lib]] | Event contracts | SQS event schemas are implicit — no formal registry or versioning. |
| [[selltreez-injection-srv]] | Missing feature | `ProcessGroups()` is a stub — GROUP_DATA_CHANGE events silently dropped. |
| [[selltreez-api-srv]] | Observability | Analytics event parsing (1,300 lines, `events.go`) has no visible tests. |
| [[Store CDK]] | Safety | Cognito pool has DESTROY removal policy — should be RETAIN in prod. |
| [[SellTreez CDK]] | Observability | No CloudWatch alarms for OpenSearch cluster health. |
| [[TrackingTreez CDK]] | Observability | No alarm for Firehose conversion failures (records silently dropped). |

## Medium Priority

| Repo | Area | Issue |
|------|------|-------|
| [[e-com-srv]] | Architecture | S3 read-modify-write for metadata is not atomic. |
| [[e-com-srv]] | Architecture | Handler dispatch is `switch/if` — needs interface hierarchy. |
| [[e-com-srv]] | Tech debt | `handle_order.go` is 3000+ LOC. |
| [[e-com-srv]] | Performance | No cache for S3 metadata — every request hits S3. |
| [[e-com-lib]] | Architecture | App union struct (~30 types) growing unwieldy. |
| [[e-com-lib]] | Architecture | Both Algolia and OpenSearch present — unclear migration status. |
| [[e-com-lib]] | Event contracts | Worker event versioning — no `version` field for SQS messages. |
| [[e-com-notification-worker]] | Config | `.env` bundled in artifact — per-environment builds, not promotable. |
| [[e-com-notification-worker]] | Templates | S3 email templates not validated at deploy time. |
| [[e-com-order-worker]] | Error handling | Processors silently swallow errors — can't distinguish "disabled" from "failed". |
| [[e-com-order-worker]] | Observability | New Relic events don't capture integration outcomes. |
| [[e-com-app-worker]] | Architecture | Discount chain is Treez-only; other providers need duplication. |
| [[e-com-app-worker]] | DX | Integration tests unconditionally skipped. |
| [[e-com-cognito-worker]] | Config | S3 config loaded once at cold start — changes require redeploy. |
| [[api-srv]] | Performance | `apps.json` fetched from S3 on every request — no caching. |
| [[api-srv]] | Error handling | OnFleet continues processing after Treez ticket update failure. |
| [[selltreez-injection-srv]] | Tech debt | `products.go` is 867 lines mixing mapping, I/O, and parsing. |
| [[selltreez-injection-srv]] | Architecture | 1G tier pricing allowlist is hardcoded. |
| [[selltreez-api-srv]] | Architecture | Raw OpenSearch queries passed through with no validation. |
| [[selltreez-api-srv]] | Architecture | Discount schedule evaluation in handler layer, not domain layer. |
| [[selltreez-lib]] | Testing | OpenSearch tests require live endpoint; no mock. |
| [[selltreez-lib]] | Observability | No structured logging in any package. |
| [[ops-infra]] | Security | CodeBuild has AdministratorAccess — should be scoped down. |
| [[ops-infra]] | Tech debt | Algolia integration partially commented out. |
| [[Store Manager Backend]] | Architecture | `actions-manager` 10-min timeout blocks on Vercel/Prismic. |
| [[Store Manager Backend]] | Tech debt | Algolia in secrets + code but dormant. |
| [[Store Manager Frontend]] | Performance | 5-second polling — should use websockets/SSE. |
| [[Store Manager Frontend]] | Auth | Sign-up hidden in UI but not disabled at pool level. |
| [[Store CDK]] | Tech debt | Lambda named "algolia" but uses OpenSearch. |
| [[Store CDK]] | Tech debt | Algolia code present but commented out. |
| [[SellTreez CDK]] | Security | API key doesn't support rotation. |
| [[SellTreez CDK]] | DX | OpenSearch password rotation is manual. |
| [[TrackingTreez CDK]] | Cost | No Parquet compaction strategy. |
| [[TrackingTreez CDK]] | Data quality | No schema evolution strategy for Glue table. |
| [[gap-dashboard]] | Tech debt | Apollo Client 2.6.8 — several major versions behind. |
| [[gap-dashboard]] | Tech debt | React Router 5.1.2 — two major versions behind. |
| [[gap-dashboard]] | Performance | No code splitting beyond CRA defaults (300+ routes). |
| [[treez-tlp]] | Feature flags | Env-var booleans only — no runtime toggle or gradual rollout. |
| [[treez-tlp]] | Search | Algolia and OpenSearch both present — unclear relationship. |

## Cross-cutting themes

### Testing gap
Nearly every repo has significant testing gaps. Workers skip tests, CDK stacks are untested, frontends have near-zero coverage. The platform's most complex business logic (order lifecycle, POS integration, promotion evaluation, discount scheduling) has the least test coverage.

### Algolia → OpenSearch migration
Algolia code is present across 5+ repos but appears increasingly dormant. OpenSearch is the primary backend. The migration should be completed and Algolia code removed to reduce maintenance.

### Implicit event contracts
SQS message schemas are defined by code, not by a shared schema registry. A single breaking change in e-com-lib's model types can silently break all downstream workers.

### Config bundled in artifacts
Multiple workers bundle `.env` files into deployment zips, creating per-environment builds. Moving to SSM Parameter Store or Lambda env vars would enable build-once/deploy-anywhere.

### Hardcoded `us-west-1`
Multiple services hardcode the AWS region. Not a problem today but blocks multi-region deployment.

### Large files
Several repos have files exceeding reasonable sizes: `handle_order.go` (3000+ LOC), `products.go` (867 lines), `events.go` (1300 lines), `config/constant.js` (75 KB). These should be split by domain concern.