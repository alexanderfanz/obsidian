# Platform Overview

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

The Treez Ecom platform (branded as **GapCommerce** / **Gap Commerce**) is a multi-tenant cannabis e-commerce system. It provides dispensary operators with online storefronts, order management, inventory sync with POS systems, payment processing, CRM integrations, and delivery logistics — all running on AWS serverless infrastructure.

## Architecture Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  CUSTOMER-FACING                                                │
│  treez-tlp (Next.js storefront)     gap-dashboard (React admin) │
│       ↓ GraphQL                          ↓ GraphQL              │
├─────────────────────────────────────────────────────────────────┤
│  API LAYER                                                      │
│  e-com-srv (GraphQL API)            api-srv (webhook receiver)  │
│       ↓ SQS/SNS                         ↓ SNS                  │
├─────────────────────────────────────────────────────────────────┤
│  ASYNC WORKERS                                                  │
│  e-com-order-worker     e-com-app-worker                        │
│  e-com-notification-worker     e-com-cognito-worker             │
├─────────────────────────────────────────────────────────────────┤
│  SEARCH PIPELINE (SellTreez)                                    │
│  selltreez-injection-srv → OpenSearch ← selltreez-api-srv       │
│       ↑ EventBridge                    ↓ Firehose               │
│                                   trackingtreez (analytics)     │
├─────────────────────────────────────────────────────────────────┤
│  SHARED LIBRARIES                                               │
│  e-com-lib (domain models, services, integrations)              │
│  selltreez-lib (search domain models, discount logic)           │
├─────────────────────────────────────────────────────────────────┤
│  INFRASTRUCTURE (ops-infra)                                     │
│  Store Manager (control plane) → CodeBuild → CDK stacks         │
│  Group CDK │ Store CDK │ SellTreez CDK │ TrackingTreez CDK      │
└─────────────────────────────────────────────────────────────────┘
```

## System Groups

### E-Commerce Core
The main transaction path. [[e-com-lib]] defines all domain models and service interfaces. [[e-com-srv]] is the GraphQL API that orchestrates cart/checkout/order flows across multiple POS systems. [[api-srv]] receives inbound webhooks from POS/delivery providers and updates order state.

Four Lambda workers handle async processing:
- [[e-com-order-worker]] — post-purchase pipeline (invoice, account upsert, integration fan-out)
- [[e-com-app-worker]] — order submission to POS systems, CRM subscriptions, loyalty discounts
- [[e-com-notification-worker]] — templated HTML email delivery via SES
- [[e-com-cognito-worker]] — user signup/confirmation side effects (account creation, CRM, Jane sync)

### SellTreez Search
A separate search pipeline with its own domain library. [[selltreez-injection-srv]] ingests product/discount/config events from EventBridge and writes to OpenSearch + DynamoDB. [[selltreez-api-srv]] serves search queries and emits analytics events to Firehose. [[selltreez-lib]] provides shared types and discount schedule evaluation logic.

### Frontends
- [[treez-tlp]] — Next.js 15 customer storefront. One codebase serving 50+ dispensary chains via per-client env vars. Prismic CMS for content. Age gate, kiosk mode, med/rec switching.
- [[gap-dashboard]] — React admin dashboard for store operators. Manages orders, products, customers, promotions, deliveries, and third-party integrations. Dual auth (Treez SSO + Cognito).

### Infrastructure
[[ops-infra]] is the infrastructure monorepo containing the Store Manager control plane and all CDK stacks. The [[Store Manager Backend]] and [[Store Manager Frontend]] let operators create and manage tenant entities (Groups, Stores, SellTreez clusters). Each entity write triggers a CodeBuild job that runs CDK to provision isolated AWS infrastructure:
- [[Group CDK]] — per-merchant: Cognito, API Gateways, Lambdas, SNS/SQS, S3
- [[Store CDK]] — per-store: Cognito, DynamoDB, stream processor, search indexing
- [[SellTreez CDK]] — per-search-cluster: OpenSearch, EventBridge, SQS, API Gateway
- [[TrackingTreez CDK]] — shared analytics data lake: Firehose, S3, Glue, Athena

## Key Platform Patterns

- **Multi-tenancy** — Every request is scoped to `(accountID, storeID)`. DynamoDB tables, S3 keys, and Cognito pools are namespaced per tenant. See [[Multi-Tenancy]].
- **Shared libraries** — [[e-com-lib]] is the single source of truth for domain models. All backend services import it. [[selltreez-lib]] serves the same role for the search pipeline.
- **POS abstraction** — Multiple cannabis POS systems (Treez, Dutchie, Jane, Blaze) are abstracted behind handler-based polymorphism. See [[POS Integration]].
- **Event-driven workers** — SNS/SQS fan-out with the [[Worker Pipeline]] pattern from e-com-lib. Predicate-based routing, pipeline composition, lazy initialization.
- **Dynamic tenant infrastructure** — Tenant infra is provisioned at runtime via CDK. See [[Tenant Provisioning]].
- **Cannabis-specific domain** — Products carry first-class THC/CBD/THCA fields, weight-based variants, medical vs. recreational classification, age gate enforcement, and compliance-driven ID verification.

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend language | Go |
| Frontend (storefront) | Next.js 15, React, TypeScript |
| Frontend (admin) | React (CRA/Craco), TypeScript |
| Frontend (ops) | React (Vite), TypeScript |
| API | GraphQL (gqlgen) |
| Database | AWS DynamoDB |
| Search | AWS OpenSearch, Algolia (legacy) |
| Queue | AWS SQS, AWS SNS |
| Events | AWS EventBridge |
| Auth | AWS Cognito, Treez SSO |
| Compute | AWS Lambda (ARM64/Graviton2) |
| Storage | AWS S3 |
| CDN | CloudFront, Cloudflare, Vercel |
| CMS | Prismic |
| IaC | AWS CDK (TypeScript) |
| Analytics | Kinesis Firehose, Athena, Glue, Snowflake |
| Monitoring | New Relic, Sentry, CloudWatch, PagerDuty |

## Repos

| Repo | Language | Role |
|------|----------|------|
| [e-com-lib](https://github.com/gap-commerce/e-com-lib) | Go | Shared domain library |
| [e-com-srv](https://github.com/gap-commerce/e-com-srv) | Go | GraphQL API |
| [api-srv](https://github.com/gap-commerce/api-srv) | Go | Webhook receiver |
| [e-com-order-worker](https://github.com/gap-commerce/e-com-order-worker) | Go | Post-purchase worker |
| [e-com-app-worker](https://github.com/gap-commerce/e-com-app-worker) | Go | POS/CRM worker |
| [e-com-notification-worker](https://github.com/gap-commerce/e-com-notification-worker) | Go | Email worker |
| [e-com-cognito-worker](https://github.com/gap-commerce/e-com-cognito-worker) | Go | Auth trigger worker |
| [selltreez-lib](https://github.com/gap-commerce/selltreez-lib) | Go | Search domain library |
| [selltreez-injection-srv](https://github.com/gap-commerce/selltreez-injection-srv) | Go | Search indexer |
| [selltreez-api-srv](https://github.com/gap-commerce/selltreez-api-srv) | Go | Search API |
| [ops-infra](https://github.com/gap-commerce/ops-infra) | TypeScript | Infrastructure monorepo |
| [gap-dashboard](https://github.com/gap-commerce/gap-dashboard) | TypeScript/React | Admin dashboard |
| [treez-tlp](https://github.com/gap-commerce/treez-tlp) | TypeScript/Next.js | Customer storefront |