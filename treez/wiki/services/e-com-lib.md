# e-com-lib

Shared Go library — the single source of truth for domain models and service contracts across the GapCommerce platform.

**Repo:** [gap-commerce/e-com-lib](https://github.com/gap-commerce/e-com-lib) | **Language:** Go | **Type:** Library (not deployed independently)

## What it owns

- **GraphQL schema** — 30 modularized `.graphqls` files covering every domain. `gqlgen` generates Go types with DynamoDB struct tags via a custom mutation hook.
- **Service interfaces** — 20+ named Go interfaces (Product, Order, Store, Account, App, Promotion, etc.) with DynamoDB-backed implementations.
- **Integration clients** — Ready-made clients for every third-party: POS (Jane, Dutchie), payments (AeroPay, Swifter, Stronghold, TreezPay), CRM (Omnisend, Iterable, Klaviyo, AlpineIQ, Sticky, Feefo, ListTrack), delivery (OnFleet), ID verification (Berbix, IDScan), ERP (LeafLogix/Treez).
- **Worker framework** — SQS-based pipeline: `SQS Message → Parser → Mapper → Processor → Handler → Deleter`. Used by all async workers.
- **POS data mappers** — Transform Jane/Dutchie API responses into internal models.

## Key patterns

- **Code generation as source of truth** — Schema changes drive everything. `make generate` produces models, DynamoDB tags, and resolver stubs.
- **Polymorphic App** — The [[App Integration]] entity uses a `Handler` discriminator field. `UnmarshalJSON` populates the correct embedded struct (~30 types). One DynamoDB table, one key schema, no migrations.
- **Cursor pagination everywhere** — All list operations return `*XxxConnection` types (GraphQL relay convention, fits DynamoDB continuation tokens).
- **No orchestration** — The library provides building blocks; consuming services compose them into workflows.

## Key entities defined here

[[Order]], [[Account]], [[Product]], [[Store]], [[App Integration]], [[Promotion]], Variant, Category, Brand, Tag, BusinessAccount, NotificationSetting

## Consumed by

[[e-com-srv]], [[e-com-order-worker]], [[e-com-app-worker]], [[e-com-notification-worker]], [[e-com-cognito-worker]] (via srv-emberz), [[api-srv]]

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Event contracts | SQS event schemas are implicit — no formal registry | High |
| App union size | ~30 integration types in one struct; growing unwieldy | Med |
| Search backends | Both Algolia and OpenSearch present; unclear if migrating | Med |
| Generated files | `model_gen.go` is 139K+ lines; slows builds | Low |

*Source: [[raw/e-com-lib.md]]*
