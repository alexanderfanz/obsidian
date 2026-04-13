# selltreez-lib

> Shared Go library providing domain models, business logic, and infrastructure integrations for the SellTreez dispensary e-commerce platform. It is consumed by backend services and Lambda functions and is never deployed independently.

---

## Purpose

Centralizes the domain types and logic that multiple SellTreez services need to agree on: store configuration, product catalogues, discount/promotion rules, and OpenSearch indexing. By living in a single library, it ensures that data shapes and rule evaluations stay consistent across the system.

**Explicitly out of scope**: HTTP handlers, authentication, database access (it models DynamoDB shapes but does not execute queries), and anything service-specific.

---

## Key Entities / Domain Models

- **Config / TreezConfig** — Raw DynamoDB config entity vs. the parsed, API-ready version. Holds store settings: fulfillment types, business hours, pickup addresses, age restrictions, geolocation. `NewTreezConfig()` handles JSON-in-string unmarshaling and default injection.
- **Discount / TreezDiscount** — Discount entity with rules, metadata, and applicability flags. `TreezDiscount` adds boolean defaults for frontend consumption.
- **DiscountRule / DiscountRuleParsed** — Individual rule (category: SCHEDULE, GROUP, BOGO; name: specific pattern) and the structured output of parsing all rules on a given discount.
- **DiscountRuleSchedule / RecursData** — Schedule definition extracted from a discount rule. `RecursData` drives the recurrence evaluation engine.
- **BaseProduct / ProductData / TreezProduct / OpenSearchProduct** — Layered product model. `BaseProduct` holds core attributes (ID, brand, category, THC/CBD, status, pricing). `ProductData` extends with inventory detail, lab results, images, and nested discounts. `TreezProduct` composes both for API responses; `OpenSearchProduct` adds `CustomProduct` for search indexing.
- **CustomProduct** — Per-location/per-customer rule overrides: price ranges, THC/CBD ranges, inventory IDs, discount groups.
- **Group** — Minimal grouping abstraction (Key + SortKey) for product and customer groups, following the DynamoDB partition/sort key pattern.
- **ProductEvent / ProductBatch** — Event wrapper and batch type for streaming product changes.

---

## API Surface

This is a library; it has no HTTP routes or RPC methods. The public Go API is:

| Package | Symbol | Description |
|---------|--------|-------------|
| `config` | `NewTreezConfig(Config) *TreezConfig` | Parse raw DynamoDB config into API-ready struct |
| `config` | `FormatTime(string) (string, error)` | Format 3–4 digit time strings ("900" → "09:00") |
| `discount` | `Discount.ParseRules(timezone) DiscountRuleParsed` | Extract schedule, group, BOGO rules from a discount |
| `discount` | `DiscountRuleSchedule.Evaluate(now, timezone) bool` | Determine if a discount is active at a given moment |
| `discount` | `RecursData.Evaluate*()` | Per-pattern recurrence evaluators (Day, Week, WeekDay, Month, MonthDay) |
| `product` | `BaseProduct`, `TreezProduct`, `OpenSearchProduct`, `ProductBatch` | Product domain types |
| `opensearch` | `NewClientAPI(ctx, config)` | Build an OpenSearch client (basic auth or AWS SigV4) |
| `opensearch` | `NewOpenSearchService(ctx, client, index)` | Service wrapper for bulk writes and search queries |
| `opensearch` | `GapOpenSearch` interface | Abstraction over `WriteBulk` and `Search` |
| `opensearch` | `GetIndexName(ctx, orgId, entityId) string` | Returns index name: `product_{entityId}` |
| `utils` | `ToVal[T]`, `ToPtr[T]`, `CopyPtr[T]` | Generic pointer helpers |
| `utils` | `GetSecret`, `GetOPensearchUserPassFromSecret` | AWS Secrets Manager accessors |
| `utils` | `StringToSlug(string) string` | URL-safe slug generator |

---

## Data Layer

- **DynamoDB** — All entity types carry `dynamodbav` struct tags. The library models the shapes and marshaling behavior but does not execute DynamoDB calls. Config and Discount are the primary DynamoDB-backed entities. Both use a Key + SortKey pattern.
- **OpenSearch** — Products are indexed via the `opensearch` package. One index per location, named `product_{entityId}`. Supports both self-managed clusters (basic auth) and AWS OpenSearch Serverless (SigV4).
- **AWS Secrets Manager** — OpenSearch credentials are fetched at runtime via `utils.GetOPensearchUserPassFromSecret`.
- No caching layer is implemented in this library.

---

## Core Business Logic

### Discount Schedule Evaluation

The most complex part of the library. `DiscountRuleSchedule.Evaluate(now, timezone)` determines whether a discount is active at a given instant, supporting:

- **All-day events** and **time-windowed events** (start/end time within a day)
- **Multi-day spans** — an event starting on day N and ending on day N+2 is checked against both the current occurrence start and the previous occurrence (to cover spans crossing midnight)
- **Recurrence patterns**:
  - `Day` — every N days with optional occurrence limits
  - `Week` — specific named weekdays (MON–SUN) or continuous spans, every N weeks
  - `WeekDay` — Monday–Friday only
  - `Month` — on a specific day of the month, every N months
  - `MonthDay` — on a specific weekday of the month

**End rules** (parsed by `parseEndRule`):
- `Never` — recurs indefinitely
- `On <date>` — stops on a specific date (DateOnly or full timestamp, timezone-aware)
- `After <N>` — stops after N occurrences

Rule parsing (`ParseRules`) extracts SCHEDULE rules into `DiscountRuleSchedule` and GROUP rules into boolean toggles (customer group, fulfillment type, purchase limits, BOGO flags, etc.).

### Config Transformation

`NewTreezConfig` does non-trivial work: it unmarshals several JSON-in-string fields from DynamoDB, normalizes boolean representations (`"true"` / `"1"` → bool), injects age-restriction defaults (Medical: 18, Adult: 21), and generates a URL slug from the store name.

---

## Cross-Repo Connections

- **Called by**: SellTreez backend services and Lambda functions that need product, discount, or config types. Exact consumers are in the `gap-commerce` GitHub org.
- **Calls**: None — this is a pure library. At runtime, consuming services use the `opensearch` and `utils` packages to reach AWS.
- **Shared types / contracts**: The `OpenSearchProduct` struct defines the search document schema shared between indexing workers and query services.
- **Shared dependency**: `github.com/gap-commerce/e-com-lib` (private) is a peer library fetched via `make lib`.
- **Events**: `ProductEvent` and `ProductBatch` define the event payload schema for product event streams, but publishing/consuming is handled by the caller.

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS OpenSearch Serverless | Infrastructure / Search | Product search index storage and querying |
| AWS Secrets Manager | Infrastructure | Storing OpenSearch credentials |
| AWS STS / SSOOIDCy / EC2 IMDS | Auth / Infrastructure | AWS credential resolution via SDK v2 |

---

## Notable Patterns & Decisions

- **Two-tier models (raw vs. parsed)** — `Config`/`TreezConfig` and `Discount`/`TreezDiscount` keep DynamoDB storage shapes separate from API shapes. The `NewX` constructors handle all transformation, keeping consumers clean.
- **JSON-in-string DynamoDB fields** — Several Config fields are stored as serialized JSON strings rather than DynamoDB maps, then unmarshaled in `NewTreezConfig`. This is a historical data shape choice, not a design preference.
- **Timezone-aware schedule evaluation** — Every public schedule function accepts a `timezone` string and converts all times before comparison. The library does not assume UTC.
- **Multi-day span safety** — The week/month/year recurrence evaluators check both the current and the previous occurrence when a multi-day event could span across the evaluation point. This is a subtle correctness requirement documented only in tests.
- **Generic pointer helpers** — `ToVal`, `ToPtr`, `CopyPtr` use Go 1.18 generics for type-safe pointer operations, avoiding repeated nil-guard boilerplate across the codebase.
- **OpenSearch auth abstraction** — `NewClientAPI` transparently supports basic auth (self-hosted) and AWS SigV4 (managed/serverless) based on whether credentials are provided, making environment migration straightforward.
- **ARM64 Lambda target** — `make package` cross-compiles for `linux/arm64` (AWS Graviton), which is intentional for cost and performance in Lambda.
- **No main package** — This repo is purely a library. Any `make deploy` target is for Lambda consumers that vendor this library, not for deploying the library itself.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Tech debt | `GetOPensearchUserPassFromSecret` has a typo in the function name (`OPensearch`) that propagates to all call sites | Low |
| Testing | `opensearch_test.go` appears to require a live OpenSearch endpoint; no mock or contract test exists for offline CI | Med |
| Architecture | `parseEndRule` in `schedule.go` handles both DateOnly and full-timestamp end dates with branching string logic; a dedicated date-parsing type would reduce fragility | Med |
| DX / tooling | `make lib` fetches `e-com-lib` manually; this could be replaced by a proper `go.mod` replace directive or GONOSUMCHECK config for a cleaner workflow | Low |
| Observability | No structured logging or tracing instrumentation in any package; consumers get no visibility into schedule evaluation decisions (useful for debugging discount issues) | Med |
| Completeness | `group/types.go` is nearly empty (just Key + SortKey); if group membership logic grows, it should move here rather than being scattered in consumers | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/selltreez-lib @ bdeae51*
