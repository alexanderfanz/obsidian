# treez-tlp

Next.js 15 storefront template powering 50+ cannabis dispensary e-commerce sites.

**Repo:** [gap-commerce/treez-tlp](https://github.com/gap-commerce/treez-tlp) | **Language:** TypeScript/Next.js 15 | **Hosting:** Vercel | **CMS:** Prismic

## What it does

Customer-facing web storefront for cannabis retail. Product discovery, age gate, CMS-driven content, cart/checkout, user auth, and kiosk self-service. One codebase serves 50+ dispensary chains via per-client environment variables.

## Key features

- **Age gate** — middleware enforces verification on every route. Bots bypass for SEO. Cookie-based with configurable expiry.
- **Medical/recreational switching** — feature-flagged. Filters products and can affect pricing.
- **Gram limit enforcement** — caps total cannabis weight per order per customer.
- **Promotions** — bundle specials, BOGO, category discounts evaluated in real time via `@gap-commerce/checkout`.
- **Multi-store fulfillment** — customer selects pickup location or delivery. Availability varies per store.
- **Kiosk mode** — isolated UI with idle timeout for in-store terminals.
- **ISR cache strategy** — CMS pages statically generated, revalidated on Prismic publish via webhook.
- **62 Prismic slices** — content editors compose pages from pre-built slices without code changes.
- **App Router with route groups** — `(Storefront)`, `(Checkout)`, `(Auth)`, `(Age)`, `(CMS)`, `(Kiosk)`.
- **161 atomic components** (atoms → molecules → organisms), 48 custom hooks.

## Tech stack

Next.js 15, React, TypeScript, Zustand, Apollo Client, OpenSearch, Prismic Slice Machine, AWS Cognito/Amplify, Stripe, Berbix

## Dependencies

- [[e-com-srv]] — GraphQL API (products, orders, stores, customers)
- OpenSearch — product search (via [[selltreez-api-srv]])
- Prismic — all editable page content
- Cognito — customer authentication

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Security | Sensitive keys prefixed `NEXT_PUBLIC_` — exposed to browser | High |
| Testing | 48 hooks + 161 components but light coverage | High |
| Feature flags | Env-var booleans only — no runtime toggle or gradual rollout | Med |
| Search | Algolia and OpenSearch both present — unclear relationship | Med |
| Config | Env-var-driven multi-tenant config with no schema validation | Med |

*Source: [[raw/treez-tlp.md]]*
