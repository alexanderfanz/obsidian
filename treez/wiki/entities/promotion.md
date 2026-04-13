# Promotion

Discount rules applied to orders. Two parallel systems exist: the e-com promotion engine and the SellTreez discount system.

## E-com Promotions (e-com-lib / e-com-srv)

Stored as JSON in S3 (`promotions.json` per store). Two types:
- `AUTOMATICALLY` — applied on every cart update if eligible
- `COUPON_CODE` — applied explicitly by customer

Uses `govaluate` for expression-based discount rules. Conditions can reference product attributes, account properties, and cart state — allows non-engineer-defined discount logic without code changes.

**Evaluated on every `UpdateCart` call** by [[e-com-srv]]. Automatic promotions always applied; coupon-code promotions applied via `ApplyPromotionCode` mutation.

## SellTreez Discounts (selltreez-lib)

Stored in DynamoDB. Managed by [[selltreez-api-srv]], indexed by [[selltreez-injection-srv]].

Much more complex schedule evaluation via [[selltreez-lib]]:
- All-day and time-windowed events
- Multi-day spans
- Recurrence: Day, Week, WeekDay, Month, MonthDay patterns
- End rules: Never, On date, After N occurrences
- Timezone-aware evaluation
- Rule types: SCHEDULE, GROUP, BOGO

Discounts have `CurrentFlag` (soft-delete), cart-applicability flag, and are filtered by the API based on schedule, timezone, and active status.

## Loyalty integration

- **AlpineIQ** — loyalty discount redemption at order time (via [[e-com-app-worker]]). Discount reverted on order decline (via [[api-srv]]).
- **Sticky** — loyalty reward redemption chained after Treez order placement (via [[e-com-app-worker]]).

## Where promotions/discounts appear

| Service | Role |
|---------|------|
| [[e-com-srv]] | Evaluates e-com promotions on every cart update |
| [[e-com-app-worker]] | Redeems AlpineIQ/Sticky loyalty discounts post-order |
| [[api-srv]] | Reverts AlpineIQ discounts on declined orders |
| [[selltreez-api-srv]] | Serves discount data to storefront with schedule filtering |
| [[selltreez-injection-srv]] | Indexes discounts to DynamoDB; computes on-sale bitmask for products |
| [[gap-dashboard]] | Operators create/manage promotions |
| [[treez-tlp]] | Displays promotions, applies at checkout |
