# selltreez-lib

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Shared Go library for the SellTreez search pipeline — domain models, discount evaluation logic, and OpenSearch utilities.

**Repo:** [gap-commerce/selltreez-lib](https://github.com/gap-commerce/selltreez-lib) | **Language:** Go | **Type:** Library (not deployed independently)

## What it owns

- **Product models** — layered: `BaseProduct` → `ProductData` → `TreezProduct` → `OpenSearchProduct`. The `OpenSearchProduct` adds `CustomProduct` for per-location/per-customer rule overrides.
- **Discount models & evaluation** — `Discount` → `TreezDiscount` with rule parsing. Schedule evaluation supports: all-day, time-windowed, multi-day spans, and recurrence patterns (Day, Week, WeekDay, Month, MonthDay) with end rules (Never, On date, After N occurrences). Timezone-aware.
- **Config transformation** — `NewTreezConfig()` unmarshals JSON-in-string DynamoDB fields, normalizes booleans, injects age-restriction defaults (Medical: 18, Adult: 21).
- **OpenSearch client** — abstraction supporting both basic auth (self-managed) and AWS SigV4 (serverless). Bulk writes and search queries.
- **Generic helpers** — `ToVal[T]`, `ToPtr[T]`, `CopyPtr[T]` pointer utilities.

## Key patterns

- **Two-tier models** — `Config`/`TreezConfig` and `Discount`/`TreezDiscount` keep DynamoDB shapes separate from API shapes. `NewX` constructors handle all transformation.
- **Multi-day span safety** — recurrence evaluators check both current and previous occurrence when a multi-day event could span the evaluation point.
- **Peer dependency** — shares `github.com/gap-commerce/e-com-lib` as a sibling library (fetched via `make lib`).

## Consumed by

[[selltreez-injection-srv]], [[selltreez-api-srv]]

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Testing | `opensearch_test.go` requires live endpoint; no offline mock | Med |
| Tech debt | `GetOPensearchUserPassFromSecret` — typo propagated to all call sites | Low |
| Observability | No structured logging in any package | Med |

*Source: [[raw/selltreez-lib.md]]*