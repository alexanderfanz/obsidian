# e-com-srv

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Central GraphQL API for the GapCommerce e-commerce platform. Orchestrates the full cart/order lifecycle across multiple POS systems, payment processors, and CRM platforms.

**Repo:** [gap-commerce/e-com-srv](https://github.com/gap-commerce/e-com-srv) | **Language:** Go | **Runtime:** AWS Lambda (serverless) | **API:** GraphQL (`/query`)

## What it does

- Unified GraphQL API consumed by [[treez-tlp]] storefronts and [[gap-dashboard]]
- Owns the cart → order lifecycle: `UpdateCart` recalculates everything on each call, `SubmitCart` pushes to POS and initiates payment
- Abstracts over multiple POS systems (Treez, Dutchie, Jane, Blaze) with per-handler files
- Manages customer accounts, promotions, ID verification, CRM subscriptions

## Key behaviors

- **Multi-POS fan-out** — A single `UpdateCart` call may sync to multiple POS systems simultaneously. Each POS has a dedicated handler (`handle_order_treez.go`, `handle_dutchie_order.go`, etc.).
- **Promotion engine** — Automatic and coupon-code promotions evaluated on every cart update.
- **Payment abstraction** — AeroPay, Stronghold, Swifter, TreezPay selected per store based on enabled [[App Integration]] records.
- **CRM fan-out** — `SubscribeToCrm` publishes SNS events; downstream consumers fan out to AlpineIQ, Sticky, Klaviyo concurrently.
- **S3 as document store** — Products, promotions, pages stored as JSON in S3. Cheap reads, but writes require read-modify-write (not atomic).
- **Tenant isolation** — All logic scoped to `(accountID, storeID)`. S3 keys and DynamoDB tables namespaced by this pair.

## Data stores

- **DynamoDB** — Orders and accounts. Per-store tables. Orders and accounts share a table, distinguished by `key` attribute (`d#` for drafts, `ACCOUNT` for accounts).
- **S3** — Semi-static metadata: products, promotions, categories, brands, pages, navigation, apps, stores, taxes, tags.
- **Cognito** — Customer auth (JWT).

## Dependencies

- [[e-com-lib]] — all domain models, service interfaces, storage implementations
- Publishes SNS events to `gc-topic` (consumed by [[e-com-order-worker]], [[e-com-notification-worker]], [[e-com-app-worker]])

## Known issues

| Area | Issue | Priority |
|------|-------|----------|
| **TreezPay race condition** | Rapid duplicate checkout creates multiple Treez tickets. Documented in `docs/TREEZ_DUPLICATE_TICKET_ENTITY_ID_BUG.md`. Needs idempotency lock. | High |
| S3 writes | Read-modify-write for metadata is not atomic | Med |
| Handler dispatch | `switch/if` blocks across large files; no interface hierarchy | Med |
| File size | `handle_order.go` is 3000+ LOC | Med |
| Performance | No cache for S3 metadata — every request hits S3 | Med |

*Source: [[raw/e-com-srv.md]]*