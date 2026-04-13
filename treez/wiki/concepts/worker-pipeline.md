# Worker Pipeline

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

All async workers in the platform use a pipeline DSL from [[e-com-lib]] (`pkg/worker`).

## Pipeline structure

```
SQS Message → Parser → Predicate Match → Mapper → Processor(s) → Deleter
```

The `worker.Manager` polls SQS, deserializes messages, matches them to a handler via predicate functions, maps the payload to a typed struct, runs one or more processors in sequence, and deletes the message on success.

## How it's used

| Worker | Event types | Notes |
|--------|-------------|-------|
| [[e-com-order-worker]] | `order/completed`, `order/kiosk-completed` | Dual-route: separate processor chains per event type |
| [[e-com-app-worker]] | 9 event types (place-order-*, subscribe-to-*, create-store) | Multi-processor chain for Treez (PlaceOrder → AlpineIQ → Sticky) |
| [[e-com-notification-worker]] | 5 event types (order/confirm, delivered, out_of_stock, status_changed, contact) | Predicate-based routing per event |

## Key design decisions

**Predicate-based routing** — each processor registers a predicate on `MessageType`. No central switch statement. Adding a new event type is additive: one `When(...)` block with a predicate, mapper, and processor.

**Fluent builder** — entire pipeline declared in ~60 lines of builder code. The DSL lives in `e-com-lib`, so debugging requires reading that repo.

**Lazy-initialized processor singletons** — each processor constructed once per Lambda container lifetime via nil-guard factory methods. Avoids repeated allocation across SQS batch messages.

**Error-tolerant chaining** — discount processors (AlpineIQ, Sticky) in [[e-com-app-worker]] are designed not to fail the overall message if they error. The order has already been placed; discount failure is logged but the SQS message is still deleted.

**Generic mapper** — `MapToModel[T]()` unmarshals SQS message body into any target type. Reduces per-processor boilerplate.

## Event contracts

SQS event schemas are **implicit** — defined only by parser/mapper interfaces, not by a formal schema registry. This is flagged as a high-priority concern in [[e-com-lib]]. Adding a `version` field and version-aware parsers would make schema evolution safer.

## SQS behavior

- Messages are deleted after successful processing
- Failed processing leaves the message visible for retry (or DLQ)
- No deduplication key stored; downstream sends are assumed idempotent
- Partial batch failure reporting (`ReportBatchItemFailures`) status is unclear in some workers