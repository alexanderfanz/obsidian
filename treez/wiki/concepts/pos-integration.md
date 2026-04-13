# POS Integration

The platform abstracts over multiple cannabis POS/ERP systems. Each store can connect to one or more POS providers.

## Supported POS systems

| Provider | Type | Capabilities |
|----------|------|-------------|
| **Treez** | ERP / POS | Order submission, ticket management, product sync, verification photo upload, TreezPay payments, status webhooks |
| **Dutchie** | POS / E-commerce | Cart sync, order creation, customer data sync |
| **Jane** | POS / E-commerce | Order history sync, special pricing, customer data, user migration |
| **Blaze** | ERP | Inventory sync (products, categories, brands, vendors), order status |

## How POS abstraction works

### Outbound (order submission)

**[[e-com-srv]]** uses handler-based polymorphism — no interface hierarchy, just `switch/if` blocks with per-handler files:
- `handle_order_treez.go`
- `handle_dutchie_order.go`
- etc.

A single `UpdateCart` can sync to multiple POS systems simultaneously. `SubmitCart` pushes to the connected POS.

**[[e-com-app-worker]]** handles async order placement:
- `place-order-treez` — most complex: order submission + verification photo upload + AlpineIQ/Sticky discount chaining
- `place-order-blaze`, `place-order-jane` — simpler direct submission

### Inbound (webhooks)

**[[api-srv]]** receives status webhooks and normalizes them:

| Provider | Statuses | Special handling |
|----------|----------|-----------------|
| Treez | 7 statuses (VERIFICATION_PENDING through COMPLETED) | Filters by revenue source (`e-commerce` only) and event type (`TICKET_STATUS`). Handles polymorphic `event_type` (string or array). |
| Jane | 11 statuses including delivery-specific | Distinguishes delivery vs. pickup paths for LisTrack emails. Fetches delivery timestamps from Jane API. |
| Blaze | Order status + inventory sync | Full inventory types (products, categories, brands, vendors). Triggers Cloudflare cache purge + Vercel rebuild. |
| OnFleet | 6 task lifecycle events | Maps to status transitions. TaskFailed reverts order + removes Treez ticket. TaskCompleted syncs payment details to Treez. |

### Data mapping

**[[e-com-lib]]** contains POS data mappers (`pkg/mappers`) that transform external API responses into internal models. The Jane mapper handles variant weight bucketing, price tier normalization, and image URL resolution.

**[[selltreez-injection-srv]]** handles Treez-specific product transformation: tier pricing cleanup, lab result normalization, on-sale bitmask encoding.

## Integration selection

Which POS is active for a store is determined by the [[App Integration]] records in `apps.json` (S3). The `handler` field on each App record identifies the provider. Workers check these flags at runtime — no static routing.

## Known issues

- **Handler dispatch** is `switch/if` in [[e-com-srv]] — adding a new POS requires touching multiple handler files. An interface hierarchy (`OrderHandler`, `PaymentHandler`) would be cleaner.
- **TreezPay race condition** — rapid duplicate checkout creates multiple tickets. See [[e-com-srv#Known issues]].
- **Treez `event_type`** — production sends string, test sends array. Custom `UnmarshalJSON` normalizes both.
