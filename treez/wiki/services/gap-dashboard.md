# gap-dashboard

React admin dashboard for cannabis retailers to manage stores, orders, products, customers, promotions, deliveries, and integrations.

**Repo:** [gap-commerce/gap-dashboard](https://github.com/gap-commerce/gap-dashboard) | **Language:** TypeScript/React | **Build:** CRA + Craco | **Hosting:** Vercel

## What operators do here

- Configure stores (hours, delivery zones, payment methods, integrations)
- Manage orders (status transitions, fulfillment)
- Manage product catalog and inventory
- Manage customers and loyalty
- Configure promotions (percentage, flat, BOGO with targeting rules)
- Edit landing pages (GrapesJS) and navigation (Prismic)
- Configure 28+ third-party integrations (Dutchie, AlpineIQ, QuickBooks, Aeropay, etc.)
- Schedule delivery windows with capacity constraints

## Key behaviors

- **Dual authentication** — Treez SSO (primary, OAuth) and Cognito (fallback). Mode detected at runtime. Both share the same Apollo Client.
- **Store context** — every page operates within a selected store. `StoreProvider` resolves domain, Prismic URLs, and scopes all data access.
- **Mixed data-fetching** — Apollo Client for GraphQL (most data), TanStack React Query for REST (search).
- **28 integration modules** — each has its own GraphQL fragments + typed queries/mutations. 71 form components in `pages/store/` for individual integration configs.
- **Feature flags** — inline checks in components, no central registry.

## Tech stack

React (CRA/Craco), Apollo Client 2.6.8, TanStack React Query, React Router 5.1.2, Pendo analytics

## Dependencies

- [[e-com-srv]] — GraphQL API (all domain data)
- OpenSearch / Algolia — search
- Prismic — CMS content
- Treez SSO / Cognito — authentication

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Tech debt | `config/constant.js` is 75 KB with no ownership | High |
| Tech debt | CRA is unmaintained; needs Vite/Next.js migration | High |
| Testing | Near-zero test coverage | High |
| Tech debt | Apollo Client 2.6.8 — multiple major versions behind | Med |
| Tech debt | React Router 5.1.2 — two major versions behind | Med |
| Performance | No code splitting beyond CRA defaults (300+ routes) | Med |

*Source: [[raw/gap-dashboard.md]]*
