# Third-Party Services Map

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Cross-repo view of every external service the platform depends on.

## POS / ERP

| Service | Used by | Purpose |
|---------|---------|---------|
| **Treez** | [[e-com-srv]], [[e-com-app-worker]], [[api-srv]], [[treez-tlp]] | Primary cannabis POS. Order submission, ticket management, product sync, verification photo upload, TreezPay payments, status webhooks. |
| **Dutchie** | [[e-com-srv]], [[e-com-lib]] | Cannabis POS. Cart sync, order creation, customer data sync. |
| **Jane** | [[e-com-srv]], [[e-com-app-worker]], [[e-com-cognito-worker]], [[api-srv]], [[e-com-lib]] | Cannabis POS. Order sync, special pricing, customer data, user migration. |
| **Blaze** | [[e-com-srv]], [[e-com-app-worker]], [[e-com-order-worker]], [[api-srv]] | Cannabis ERP. Inventory sync, order status, fulfillment. |

## Payments

| Service | Used by | Purpose |
|---------|---------|---------|
| **AeroPay** | [[e-com-srv]], [[e-com-lib]] | ACH-based cannabis payments with merchant accounts. |
| **Stronghold** | [[e-com-srv]], [[e-com-lib]] | Payment processing with tipping support. |
| **Swifter** | [[e-com-srv]], [[e-com-lib]] | Payment processing with OAuth flow. |
| **TreezPay** | [[e-com-srv]], [[e-com-lib]] | Treez-native ACH payment ticketing. |
| **Stripe** | [[treez-tlp]] | Online card payment processing. |
| **Webpay** | [[treez-tlp]] | Alternative debit payment method. |

## CRM / Email / SMS

| Service | Used by | Purpose |
|---------|---------|---------|
| **Klaviyo** | [[e-com-app-worker]], [[e-com-cognito-worker]], [[e-com-order-worker]] | Email/SMS marketing, list subscriptions, order event tracking. |
| **AlpineIQ** | [[e-com-app-worker]], [[e-com-cognito-worker]], [[api-srv]], [[gap-dashboard]] | Cannabis-specific loyalty/CRM. Discount redemption/reversion, SMS notifications, favorite store. |
| **Omnisend** | [[e-com-app-worker]], [[e-com-lib]] | Marketing automation, list subscriptions. |
| **ListTrack** | [[e-com-app-worker]], [[e-com-cognito-worker]], [[api-srv]], [[e-com-order-worker]] | Transactional email, signup events, order tracking. |
| **Iterable** | [[e-com-lib]] | Customer engagement platform. |
| **Sticky** | [[e-com-app-worker]], [[e-com-lib]] | CRM/loyalty reward/discount redemption. |
| **Feefo** | [[e-com-lib]] | Reviews/feedback platform. |

## ID Verification

| Service | Used by | Purpose |
|---------|---------|---------|
| **Berbix** | [[e-com-srv]], [[e-com-lib]], [[treez-tlp]], [[gap-dashboard]] | Age/identity verification. |
| **IDScan** | [[e-com-srv]], [[e-com-lib]] | Alternative ID scanning. |

## Delivery

| Service | Used by | Purpose |
|---------|---------|---------|
| **OnFleet** | [[api-srv]], [[e-com-lib]] | Last-mile delivery management. Task lifecycle webhooks. |

## CMS

| Service | Used by | Purpose |
|---------|---------|---------|
| **Prismic** | [[treez-tlp]], [[gap-dashboard]], [[Store Manager Backend]] | Headless CMS. 62 slice types for storefront content. CMS repo cloned for new groups. |
| **GrapesJS** | [[gap-dashboard]] | In-browser drag-and-drop landing page builder. |

## Search

| Service | Used by | Purpose |
|---------|---------|---------|
| **OpenSearch (AWS)** | [[selltreez-injection-srv]], [[selltreez-api-srv]], [[SellTreez CDK]], [[Store CDK]], [[e-com-lib]] | **Primary search backend.** Product catalog, per-tenant indexes. |
| **Algolia** | [[gap-dashboard]], [[treez-tlp]], [[e-com-lib]], [[Store CDK]] | **Legacy/secondary search.** Present in code but appears increasingly dormant. Mid-migration to OpenSearch. |

## Analytics

| Service | Used by | Purpose |
|---------|---------|---------|
| **Snowflake** | [[TrackingTreez CDK]] | External S3 integration for BI queries on search data lake. |
| **Google Analytics** | [[treez-tlp]] | Web traffic and conversion analytics. |
| **FullStory** | [[treez-tlp]] | Session recording and ecommerce event tracking. |
| **Pendo** | [[gap-dashboard]] | User analytics and product tours. |

## Auth

| Service | Used by | Purpose |
|---------|---------|---------|
| **AWS Cognito** | Nearly all services | Customer and admin user pools, JWT auth. |
| **Treez SSO** | [[gap-dashboard]] | Primary OAuth-based login for admin operators. |

## Monitoring

| Service | Used by | Purpose |
|---------|---------|---------|
| **New Relic** | All Go services, [[ops-infra]] | APM, distributed tracing, custom events. |
| **Sentry** | All Go services, [[ops-infra]] | Error tracking and panic capture. |
| **PagerDuty** | [[ops-infra]] | Incident management with on-call rotation. |
| **Discord** | [[ops-infra]] | Team alarm notifications. |
| **CloudWatch** | All CDK stacks | Lambda invocation, SQS depth, DynamoDB alarms. |

## Infrastructure

| Service | Used by | Purpose |
|---------|---------|---------|
| **Vercel** | [[treez-tlp]], [[Store Manager Frontend]], [[Store Manager Backend]] | SPA/SSR hosting, deployment, ISR. |
| **Cloudflare** | [[api-srv]] | CDN cache purging after inventory sync. |
| **Google Maps** | [[treez-tlp]], [[gap-dashboard]] | Address autocomplete, delivery zone mapping. |

## Redundancy observations

- **Search**: Algolia and OpenSearch overlap significantly. Migration to OpenSearch appears in progress.
- **CRM**: 7 CRM/email/loyalty platforms. Some stores may use multiple simultaneously (concurrent fan-out in workers).
- **Monitoring**: New Relic + Sentry + CloudWatch — three observability layers. Not necessarily redundant (APM vs. errors vs. alarms) but overhead for each new service.
- **Payments**: 6 payment providers. Per-store selection via App config — not redundancy, but complexity.