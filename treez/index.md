# Index

Content catalog for the Treez Ecom wiki. Updated on every ingest.

## Overview

- [[overview]] — Platform architecture, system groups, technology stack, repo listing

## Services

### E-Commerce Core
- [[e-com-lib]] — Shared Go library: domain models, service interfaces, integration clients, worker framework
- [[e-com-srv]] — GraphQL API: cart/order lifecycle, multi-POS orchestration, payment, CRM
- [[api-srv]] — Webhook receiver: Treez/Jane/Blaze/OnFleet status updates and inventory sync
- [[e-com-order-worker]] — Post-purchase worker: invoice assignment, account upsert, integration fan-out
- [[e-com-app-worker]] — POS/CRM worker: order submission to POS, CRM subscriptions, loyalty discounts
- [[e-com-notification-worker]] — Email worker: templated SES emails for order lifecycle events
- [[e-com-cognito-worker]] — Auth trigger worker: account bootstrap, Jane sync, CRM subscriptions

### SellTreez Search Pipeline
- [[selltreez-lib]] — Shared Go library: search domain models, discount schedule evaluation, OpenSearch utilities
- [[selltreez-injection-srv]] — Search indexer: EventBridge events → OpenSearch + DynamoDB
- [[selltreez-api-srv]] — Search API: product search, discounts, config, analytics events

### Frontends
- [[treez-tlp]] — Customer storefront: Next.js 15, 50+ dispensary sites, Prismic CMS, age gate, kiosk
- [[gap-dashboard]] — Admin dashboard: React, store/order/product/customer management, 28+ integrations

### Infrastructure (ops-infra)
- [[ops-infra]] — Infrastructure monorepo overview
- [[Store Manager Backend]] — Control plane API: CRUD + async CDK deployment orchestration
- [[Store Manager Frontend]] — Operator SPA: entity management, infra inspection, bulk operations
- [[Group CDK]] — Per-group stack: Cognito, API Gateways, 5 Lambdas, SNS/SQS, S3, CloudFront
- [[Store CDK]] — Per-store stack: DynamoDB, Cognito, stream processor, SES
- [[SellTreez CDK]] — Per-search-cluster stack: OpenSearch, EventBridge, SQS, API Gateway
- [[TrackingTreez CDK]] — Shared analytics: Firehose, S3 data lake, Glue, Athena, Snowflake

## Entities

- [[Order]] — Central transactional entity; doubles as cart and submitted purchase
- [[Account]] — Customer profile with compliance fields (medical ID, DOB)
- [[Product]] — Cannabis product with variants, cannabinoid content, multi-system representation
- [[Store]] — Retail dispensary location; primary tenant boundary
- [[Group]] — Merchant organization; parent of stores; owns compute infrastructure
- [[App Integration]] — Polymorphic third-party integration config (~30 types via Handler discriminator)
- [[Promotion]] — Discount rules: e-com expression engine + SellTreez schedule evaluation

## Concepts

- [[Multi-Tenancy]] — Tenant hierarchy, data isolation, infrastructure isolation, request scoping
- [[Order Lifecycle]] — Full flow: cart → checkout → post-purchase → webhooks → notifications
- [[POS Integration]] — Treez/Dutchie/Jane/Blaze abstraction, inbound/outbound flows
- [[Worker Pipeline]] — SQS pipeline DSL from e-com-lib: predicate routing, processor chaining
- [[Tenant Provisioning]] — Dynamic infrastructure: Store Manager → EventBridge → CodeBuild → CDK
- [[Search Architecture]] — SellTreez pipeline: EventBridge → OpenSearch, analytics via Firehose

## Cross-Cutting

- [[Dependency Map]] — Service dependencies: libraries, event flows, webhooks, infrastructure
- [[Third-Party Services Map]] — Every external service across all repos with usage context
- [[Potential Improvements]] — Consolidated issues by priority with cross-cutting themes
