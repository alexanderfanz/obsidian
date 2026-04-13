# e-com-lib

> A Go library that provides the complete shared domain layer for GapCommerce's cannabis e-commerce platform. It owns the GraphQL schema definitions, generated models, service interfaces, AWS infrastructure wrappers, and integration clients for every external system the platform touches. Consuming services (primarily e-com-srv) import this as a versioned dependency rather than duplicating domain logic.

---

## Purpose

`e-com-lib` is the single source of truth for data models and service contracts across the GapCommerce platform. It:
- Defines the GraphQL schema (modularized per domain) and generates Go types, DynamoDB struct tags, and resolver stubs via `gqlgen`
- Exposes interface-typed service contracts (`Product`, `Order`, `Store`, `Account`, etc.) backed by DynamoDB implementations
- Ships ready-made clients for every third-party integration (POS, payments, CRM, delivery, ID verification)
- Provides the SQS-based event worker framework consumed by background processing services

**Out of scope**: HTTP routing, authentication enforcement, rate limiting, and business orchestration — those live in consuming services.

---

## Key Entities / Domain Models

- **Account** — Customer profile. Holds contact info, loyalty points, activity log, medical ID photo, and cross-system links (`treez_id`, `alpine_id`). Used for both retail and medical customers.
- **Product** — Cannabis product with variants, cannabinoid content (THC/CBD/THCA percentages), pricing tiers, images, category/brand associations, and POS system mappings. Supports types: `REGULAR`, `DERIVED`, `BUNDLE`.
- **Variant** — Sellable SKU within a product (weight bucket, price, sale price, discount fields, inventory count).
- **Order** — Order lifecycle record: line items, applied promotions, payment details, delivery address, tracking, POS confirmation references, and `OrderStatusType` state.
- **Store** — Retailer configuration: payment methods, menu type, tax settings, geo config, operating hours, feature flags (e.g., `1g_bulk_flower`, `treez_medical_id_resend`).
- **App** — Polymorphic integration record (discussed further in Core Business Logic). A single struct that can represent any of ~30 third-party integration configurations based on a `Handler` discriminator field.
- **Promotion** — Discount rules with expression-based evaluation via `govaluate`. References products, categories, brands.
- **Category / Brand / Tag** — Product taxonomy entities.
- **BusinessAccount** — B2B customer account distinct from retail `Account`.
- **Notification Setting** — Per-store notification preferences (SMS, email, push).

---

## API Surface

This is a **library**, not a service — there are no HTTP routes. The API surface is:

### Service Interfaces (pkg/services/*)

Each service package exports a named interface. All are DynamoDB-backed.

| Interface | Package | Key Methods |
|-----------|---------|-------------|
| `Product` | `pkg/services/product` | Create, Update, Delete, Get, GetAll, Search |
| `Order` | `pkg/services/order` | Create, Update, Get, GetAll, Filter |
| `Store` | `pkg/services/store` | Create, Update, Get, GetAll |
| `Account` | `pkg/services/account` | Create, Update, Get, GetAll, Search |
| `App` | `pkg/services/app` | Create, Update, Get, GetAll, GetByType |
| `Promotion` | `pkg/services/promotion` | Create, Update, Get, GetAll, Evaluate |
| `Invoice` | `pkg/services/invoice` | Create, Get, GetAll |
| `Category` | `pkg/services/category` | Create, Update, Delete, Get, GetAll |
| `Brand` | `pkg/services/brand` | Create, Update, Delete, Get, GetAll |
| `Page` | `pkg/services/page` | CMS page CRUD |
| `Navigation` | `pkg/services/navigation` | Menu CRUD |
| `Tag` | `pkg/services/tag` | Tagging CRUD |
| `Option` | `pkg/services/option` | Product option/variant CRUD |
| `Tracking` | `pkg/services/tracking` | Order tracking CRUD |
| `Contact` | `pkg/services/contact` | Contact form submissions |
| `BusinessAccount` | `pkg/services/businessaccount` | B2B account CRUD |
| `LandingPage` | `pkg/services/landing_page` | Landing page builder CRUD |
| `InventorySync` | `pkg/services/inventory_sync` | Sync state records |
| `BestSelling` | `pkg/services/best_selling` | Analytics aggregates |
| `Iterable` | `pkg/services/iterable` | Iterable CRM settings |
| `Listrack` | `pkg/services/listrack` | ListTrack CRM settings |

### Integration Clients (pkg/ecommerce, pkg/payment, pkg/crm, pkg/delivery, pkg/idcheck, pkg/erp)

| Package | Integrations |
|---------|-------------|
| `pkg/ecommerce` | Jane POS, Dutchie POS |
| `pkg/payment` | AeroPay, Swifter, Stronghold, TreezPay |
| `pkg/crm` | Omnisend, Iterable, Klaviyo, AlpineIQ, Sticky, Feefo, ListTrack |
| `pkg/delivery` | OnFleet |
| `pkg/idcheck` | Berbix, IDScan |
| `pkg/erp` | LeafLogix (Treez) |

### GraphQL Schema Modules (pkg/graphql/modules/*)

30 `.graphqls` schema files covering every domain above. These are the canonical contract — consuming services mount these schemas via `gqlgen`.

---

## Data Layer

- **Primary store**: AWS DynamoDB. All models carry `dynamodbav` (AWS SDK v2) and `dynamo` struct tags, generated automatically by the custom `gqlgen` mutation hook in `pkg/graphql/main.go`.
- **Search**: Algolia (`algoliasearch-client-go`) and OpenSearch (`opensearch-go`) are both available as search backends (products, accounts).
- **File storage**: AWS S3 via `pkg/aws/s3`.
- **Caching**: No dedicated cache layer in the library itself; consuming services manage caching if needed.
- **Pagination**: Cursor-based pagination implemented in `pkg/pagination/`. All `GetAll`-style methods return a `*XxxConnection` with `PageInfo` and `edges`.
- **Event queue**: AWS SQS via `pkg/aws/sqs` — used by the worker framework for async event processing.

---

## Core Business Logic

### Polymorphic App Integration System

The `App` entity uses a discriminated union pattern with a `Handler` string field. `UnmarshalJSON` on `App` reads `Handler` and populates the correct embedded integration struct (one of ~30 types: Jane, Dutchie, Treez, AeroPay, Klaviyo, OnFleet, Berbix, etc.). This allows a single DynamoDB table to store every store integration config without schema migrations when new integrations are added.

### SQS Worker Framework (pkg/worker)

A pipeline-based event processor with a fluent builder:
```
SQS Message → Parser → Mapper → Processor → Handler → Deleter
```
The `Manager` polls SQS, deserializes messages, and passes them through the chain. Each stage is a pluggable interface, allowing consuming services to wire up domain-specific logic. This is the foundation for all async workflows (inventory sync, order status updates, etc.).

### Promotion Evaluation

Promotions use `govaluate` for expression-based discount rules. Conditions can reference product attributes, account properties, and cart state. This allows non-engineer-defined discount logic without code changes.

### Cannabis-Specific Domain

Product models carry first-class fields for cannabinoid content (`thc`, `cbd`, `thca` percentages), cannabis type enum (`INDICA`, `SATIVA`, `HYBRID`, `CBD`), and unit-of-measure handling for cannabis weights. The platform accounts for these at the schema level rather than treating them as generic metadata.

### POS Data Mapping (pkg/mappers)

Mappers transform external POS API responses (Jane, Dutchie) into internal product/order models. The Jane mapper handles variant weight bucketing, price tier normalization, and image URL resolution. This is where POS-specific quirks are isolated.

### Medical ID Lifecycle

`Account` tracks `medical_id_photo_uri` at the root level and a `treez_medical_id_resend` feature flag on `Store`. The recent commit history shows active work on Treez medical ID flows (resend flag, photo URI placement).

---

## Cross-Repo Connections

- **Consumed by**: `e-com-srv` (primary consumer — the GapCommerce GraphQL API service). Likely also background worker services.
- **Calls**: No direct service-to-service HTTP calls within this library. Integration clients call external APIs (see Third-Party Services).
- **Shared types**: This library *is* the shared type layer. All domain models originate here via `pkg/model/model_gen.go`.
- **Internal dependencies**: `github.com/gap-commerce/glog` (internal structured logging library).
- **Events**: Produces/consumes SQS events via the worker framework. Event schemas are implicitly defined by the mapper/parser interfaces — not formally versioned.

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS DynamoDB | Infrastructure | Primary data store for all entities |
| AWS S3 | Storage | File/image storage |
| AWS Cognito | Auth | User identity, JWT issuance |
| AWS SQS | Infrastructure | Async event queue for worker framework |
| AWS SES | Email | Transactional email |
| AWS SNS | SMS/Push | Notifications |
| AWS CloudWatch | Monitoring | Metrics and alarms |
| AWS Pinpoint | Analytics | Customer engagement analytics |
| New Relic | Monitoring | APM, distributed tracing (via middleware) |
| Sentry | Monitoring | Error tracking (`pkg/util`) |
| Algolia | Search | Product/account search |
| OpenSearch | Search | Alternative search backend |
| Jane POS | POS | Dispensary POS integration |
| Dutchie | POS | Dispensary POS integration |
| LeafLogix (Treez) | ERP | Inventory/ERP integration |
| AeroPay | Payments | ACH-based cannabis payments |
| Swifter | Payments | Payment processor |
| Stronghold | Payments | Payment processor |
| TreezPay | Payments | Treez-native payment processor |
| Omnisend | Email/CRM | Marketing automation |
| Iterable | CRM | Customer engagement platform |
| Klaviyo | CRM | Email/SMS marketing |
| AlpineIQ | CRM | Cannabis-specific loyalty/CRM |
| Sticky | CRM | CRM integration |
| Feefo | CRM | Reviews/feedback platform |
| ListTrack | CRM | Email marketing |
| OnFleet | Delivery | Last-mile delivery management |
| Berbix | Auth | ID verification (age/identity) |
| IDScan | Auth | ID scanning verification |

---

## Notable Patterns & Decisions

**Code generation as the source of truth.** GraphQL schemas (`*.graphqls`) drive everything: models, DynamoDB struct tags, and resolver stubs are all generated. The custom `gqlgen` mutation hook (`pkg/graphql/main.go`) injects AWS-specific struct tags during generation, eliminating manual annotation. Adding a field means editing the schema and running `make generate`.

**Interface-first service design.** Every service is defined as a Go interface before any implementation. This is what makes the `pkg/mocks/` package viable and keeps consuming services testable. The pattern is uniform across all 20+ services.

**Library versioning via conventional commits.** The repo uses conventional commits (`feat:`, `fix:`) to trigger automated version bumps. Consuming services pin to a specific version and bump explicitly after testing. The `replace` directive in `go.mod` is the local dev escape hatch.

**Polymorphic App via JSON discriminator.** Rather than a type hierarchy or separate tables per integration, all store integrations live in one `App` shape with a `Handler` field. This keeps DynamoDB access patterns simple (one table, one key schema) at the cost of a large union struct with many optional fields.

**No business orchestration in the library.** The library provides building blocks (service interfaces, integration clients, worker framework) but contains no workflow logic. Consuming services compose these. This is a deliberate boundary — the library should not know about multi-step business processes.

**Cursor pagination everywhere.** All list operations use cursor-based pagination returning `*XxxConnection` types — consistent with GraphQL relay conventions and appropriate for DynamoDB's continuation token model.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Event contracts | SQS event schemas are implicit (defined only by parser/mapper interfaces). A formal event schema registry or versioned struct types would reduce cross-service breakage risk. | High |
| App union size | The `App` struct embeds ~30 integration types. As integrations grow this becomes unwieldy. A registry pattern or interface-based dispatch might scale better. | Med |
| Search backend duplication | Both Algolia and OpenSearch are present. It's unclear if both are actively used or if one is being migrated to the other. Consolidating would reduce maintenance surface. | Med |
| Generated file size | `generated.go` and `pkg/model/model_gen.go` are very large (139K+ lines). This can slow builds and diffs. Splitting generated output by domain could help. | Low |
| Worker event versioning | The SQS worker framework has no built-in versioning for message formats. Adding a `version` field and version-aware parsers would make schema evolution safer. | Med |
| Test coverage gaps | VCR cassettes exist but coverage of integration edge cases (POS API failures, payment timeouts) appears limited from the structure. | Low |
| DX: local development | Docker Compose is referenced in the Makefile but the compose file wasn't examined. If it requires live AWS credentials, a localstack setup would improve the onboarding experience. | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-lib (commit b4cbde5)*
