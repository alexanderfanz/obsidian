# App Integration

A polymorphic configuration record representing a third-party integration attached to a store.

## The polymorphic pattern

The `App` entity in [[e-com-lib]] uses a **discriminated union** with a `Handler` string field. `UnmarshalJSON` reads `Handler` and populates the correct embedded integration struct — one of ~30 types. This allows a single DynamoDB table to store every store integration config without schema migrations.

Handler values include: `dutchie`, `treez`, `jane`, `blaze`, `aeropay`, `stronghold`, `swifter`, `treezpay`, `klaviyo`, `omnisend`, `iterable`, `alpineiq`, `sticky`, `feefo`, `listrack`, `onfleet`, `berbix`, `idscan`, `quickbooks`, and more.

## Key fields

`handler` (discriminator), `category` (Payment, CRM, Inventory, etc.), `status` (enabled/disabled), and handler-specific credentials/configuration.

## How integrations are selected

- [[e-com-srv]] reads enabled `App` records for a store to determine which POS, payment, CRM, and delivery integrations are active.
- [[e-com-order-worker]] checks `apps.json` in S3 to decide which downstream events to publish.
- [[e-com-app-worker]] loads per-store credentials from `apps.json` to call third-party APIs.
- [[api-srv]] loads credentials from `apps.json` per webhook request.

## Integration categories

| Category | Integrations |
|----------|-------------|
| POS / ERP | Treez, Dutchie, Jane, Blaze |
| Payments | AeroPay, Swifter, Stronghold, TreezPay, Stripe, Webpay |
| CRM / Email | Klaviyo, Omnisend, Iterable, AlpineIQ, Sticky, Feefo, ListTrack |
| Delivery | OnFleet |
| ID Verification | Berbix, IDScan |
| Accounting | QuickBooks |

See [[Third-Party Services Map]] for the full cross-repo view.

## Storage

- **DynamoDB** — App records in the store table (via [[e-com-lib]] App service)
- **S3** — `apps.json` per store, containing all integration configs and credentials. Loaded at runtime by workers and webhook service.
- **[[gap-dashboard]]** — 71 form components for configuring integrations (28 domain modules with GraphQL fragments)

## Concerns

The App struct embedding ~30 types is growing unwieldy. A registry pattern or interface-based dispatch might scale better. See [[e-com-lib#Notable concerns]].
