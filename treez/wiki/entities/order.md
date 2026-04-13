# Order

The central transactional entity in the platform. An Order doubles as a shopping cart (draft state) and a submitted purchase.

## Lifecycle

```
Cart (draft, key="d#") → Pending → Approved → Completed
                                  → Rejected
                       → AbandonedCart
```

Status transitions are triggered by:
- **Customer actions** — `UpdateCart`, `SubmitCart` via [[e-com-srv]]
- **POS webhooks** — Treez/Jane/Blaze status changes via [[api-srv]]
- **Delivery events** — OnFleet task lifecycle via [[api-srv]]
- **Admin actions** — `UpdateOrderStatus` mutation

## Where it lives

| Store | Repo | Purpose |
|-------|------|---------|
| DynamoDB (per-store table) | [[e-com-srv]], [[e-com-order-worker]], [[e-com-app-worker]] | Source of truth. Table per `(accountID, storeID)`. PK: `entity_id`, SK: `key`. Draft orders keyed with `d#`. |
| DynamoDB (mapping tables) | [[api-srv]] | `BlazeOrder` (`blaze#` prefix) and `JaneOrder` (`j#` prefix) map provider IDs to internal order IDs. |

## Key fields

`EntityID` (UUID), `Status` (OrderStatusType), `LineItems`, `SubtotalPrice`/`TotalPrice` (cents), `ProviderStoreID`, provider-specific IDs (`DutchieOrderID`, `TreezPayTicketIdsByStore`, `JaneOrderID`), delivery address, tracking, payment details, applied promotions, activity log.

## How each service touches it

| Service | Operations |
|---------|-----------|
| [[e-com-srv]] | Creates carts, recalculates on every `UpdateCart`, submits to POS on `SubmitCart`, manages status |
| [[e-com-order-worker]] | Post-purchase: assigns invoice number, sets status to PENDING, fans out to integrations |
| [[e-com-app-worker]] | Submits order to Treez/Blaze/Jane POS, uploads verification photos, redeems loyalty discounts |
| [[e-com-notification-worker]] | Reads order data to assemble email template context |
| [[api-srv]] | Updates order status from inbound POS/delivery webhooks |
| [[gap-dashboard]] | Operators view and manage orders |
| [[treez-tlp]] | Customers browse, build carts, checkout |

## Cannabis-specific fields

Orders carry medical ID references and DOB for compliance. The [[e-com-order-worker]] copies these onto the [[Account]] record. Treez order submission includes verification photo upload for age verification.

## Known issues

- **TreezPay duplicate ticket race condition** — rapid duplicate checkout can create multiple Treez tickets for the same order (documented in e-com-srv). See [[e-com-srv#Known issues]].
- **Invoice counter** — atomic DynamoDB counter per store, assigned by [[e-com-order-worker]]. Only strict ordering guarantee.
