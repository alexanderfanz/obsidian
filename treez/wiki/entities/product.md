# Product

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

Cannabis product with variants, compliance fields, and multi-system representation.

## Where it lives

Products exist in multiple representations across the platform:

| Store | Repo | Shape |
|-------|------|-------|
| S3 (JSON blobs) | [[e-com-srv]] | `products.json` per store — full product catalog. Read on every GraphQL request. |
| OpenSearch | [[selltreez-injection-srv]], [[selltreez-api-srv]] | `OpenSearchProduct` — flattened, enriched for search. Per-tenant index. |
| Treez POS | [[api-srv]] | Source of truth for inventory. Synced via webhooks. |
| Blaze/Jane POS | [[api-srv]], [[e-com-srv]] | Alternative POS sources. Mapped via [[e-com-lib]] mappers. |
| S3 (provider products) | [[api-srv]] | `providers_products/provider-{storeID}.json` — Treez product sync. |

## Key fields

`Name`, `SKU`, `Type` (REGULAR, DERIVED, BUNDLE / flower, edible, etc.), `Brand`, `Category`, `Strain`, pricing tiers, `Variants` (weight bucket, price, sale price, inventory count), images, `Status`.

### Cannabis-specific fields

- **Cannabinoid content** — `thc`, `cbd`, `thca` percentages (first-class fields, not metadata)
- **Cannabis type** — `INDICA`, `SATIVA`, `HYBRID`, `CBD`
- **Lab results** — THC/CBD/CBDA/THCA as percentages or amounts. Normalized during ingestion (HTML stripped, mg fallback values handled).
- **Weight-based variants** — half gram through ounce, with unit-of-measure handling

## Variant

A sellable SKU within a product. Carries weight bucket, price, sale price, discount fields, and inventory count. Defined in [[e-com-lib]].

## Product mapping pipeline

```
Treez POS → api-srv (webhook sync) → S3 (products.json)
                                                ↓
Treez POS → EventBridge → selltreez-injection-srv → OpenSearch
                            (tier pricing cleanup, lab normalization,
                             on-sale bitmask, menu title enrichment)
```

The [[selltreez-injection-srv]] product mapping is the most complex transformation (~867 lines):
- **Tier pricing cleanup** — removes rogue "1G" entries (entity ID allowlist)
- **On-sale bitmask** — encodes BOGO, bundle, scheduled discounts as integer flags
- **Lab result normalization** — strips HTML entities, handles `"10 mg"` fallbacks
- **CustomProduct extraction** — per-location pricing, inventory, discount groups

## Search

Products are searched via [[selltreez-api-srv]] which proxies raw OpenSearch queries. The [[treez-tlp]] storefront constructs OpenSearch DSL queries client-side with filters for type, brand, category, effects, flavors, THC/CBD %, price range, store, and availability.

Algolia is also present as an alternative/legacy search backend in multiple repos. The relationship between Algolia and OpenSearch is unclear — appears to be mid-migration.