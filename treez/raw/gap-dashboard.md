# gap-dashboard (gc-admin)

> React + TypeScript admin dashboard for Treez e-commerce. A multi-tenant B2B interface used by cannabis retailers to manage stores, orders, products, customers, promotions, deliveries, and third-party integrations. It is the primary operational control plane for the GAP (Gap Commerce) platform.

---

## Purpose

Provide store operators and admins a single UI to configure and operate a cannabis e-commerce storefront. Covers the full stack of store operations: account and store setup, product catalog and inventory, order and delivery management, customer records, promotions and loyalty, landing page and navigation editing, payment gateway configuration, and third-party app integrations (Dutchie, AlpineIQ, QuickBooks, Aeropay, etc.).

Out of scope: the customer-facing storefront itself, POS terminal software, and the GraphQL backend API (all separate repos).

---

## Key Entities / Domain Models

- **Store** — the central entity; a single retailer location with its own configuration, hours, delivery zones, integrations, and branding. Most pages operate in the context of a selected store.
- **Account / Business Account** — the merchant organization that owns one or more stores. Users authenticate at the account level.
- **Product / Variant** — cannabis and non-cannabis SKUs with categories, pricing, inventory, and compliance fields. Variants represent size/weight options.
- **Order** — a customer purchase with line items, discount applications, delivery/pickup scheduling, and status transitions (placed → fulfilled → completed).
- **Customer** — a registered end-user with purchase history, loyalty points, and optional identity verification (Berbix).
- **Promotion** — discount rules (percentage, flat, BOGO) with targeting conditions; tied to loyalty integrations like AlpineIQ.
- **Delivery Window / Time Slot** — scheduled pickup or delivery slots with capacity constraints; managed through the logistics module.
- **Landing Page / Navigation** — CMS-managed homepage blocks and site navigation entries; edited via Prismic and GrapesJS.
- **App Integration** — a third-party service enabled for a store (e.g., Dutchie POS, QuickBooks, Aeropay). Each has its own credential form and sync settings.

---

## API Surface

This is a frontend-only repo. It does not expose its own API. It consumes:

| Method | Path / Name | Description |
|--------|-------------|-------------|
| POST | `REACT_APP_GRAPHQL_ENDPOINT_URL` | All store, order, product, customer, and config mutations/queries via Apollo Client |
| POST | `REACT_APP_OPEN_SEARCH_ENDPOINT` | OpenSearch queries for order and product search (account-id + store-id headers) |
| GET/POST | Algolia REST API | Order and product search (alternative search path) |
| GET/POST | Prismic API | CMS content for homepages, promo carousels, and navigation |
| GET | Treez SSO endpoints | Token exchange and logout (`REACT_APP_TREEZ_LOGIN_URL`, `REACT_APP_TREEZ_LOGOUT_URL`) |
| GET | AWS Cognito | Authentication (fallback mode) |
| GET | Google Maps API | Address autocomplete and delivery zone mapping |

---

## Data Layer

No database in this repo — all persistence is through the GraphQL backend. Client-side state is split across:

- **Apollo InMemoryCache** — GraphQL query results cached by type + ID. Several types (`LineItem`, `Variant`, `OrderDiscountItem`, `Integrations`) use random IDs to avoid cache collisions.
- **TanStack React Query** — used for REST-style calls (OpenSearch, Algolia). Handles caching, loading, and error states for non-GraphQL endpoints.
- **localStorage** — auth tokens (Treez SSO mode), user preferences, and some form state via `useLocalStorage` hook.
- **Apollo Local State** — a small subset of UI state (initial values defined in `config/apollo/data.ts`) stored in the Apollo cache for cross-component access without a separate state manager.

No caching infrastructure beyond the above; no Redis, CDN caching, or service worker.

---

## Core Business Logic

**Dual authentication:** The app supports two auth modes toggled by `isTreezSSOEnabled` (in `src/auth/data.ts`). In SSO mode, an OAuth token exchange happens on load, tokens are stored in localStorage, and refreshed on expiry; the Apollo client injects them as `Treez <token>`. In Cognito mode, AWS Amplify handles the session and JWT injection. Both modes share the same Apollo client — mode detection is at runtime.

**Store context:** Every page operates within a "current store" selected at login. `StoreProvider` resolves the store's domain, Prismic CMS URLs (homepage, promo carousel, navigation editor), and exposes them to all child pages via context. Switching stores effectively re-scopes all data access.

**Delivery window scheduling:** `src/utils/timeSlot.js` implements slot generation — given a store's configured open hours, buffer times, and per-slot capacity, it generates the available pickup/delivery windows shown to customers and enforced at checkout. Timezone handling runs through `src/utils/timezone.ts` (moment.js based) to display times in the store's local timezone regardless of user locale.

**OpenSearch query building:** `src/opensearch/hooks/useSearch.ts` wraps TanStack Query around a POST to the OpenSearch endpoint. It constructs Elasticsearch DSL (filters, aggregations, pagination) and manages hits/facets as query state, enabling faceted search across orders and products.

**App integrations:** Each third-party integration (28 domain modules in `apollo/module-operations/`) follows the same pattern: GraphQL fragments + typed queries/mutations per operation. Store settings are split across 71 separate form components in `pages/store/`, each owning its own mutation, so individual integration configs can be saved independently without touching unrelated fields.

**Promotions and loyalty:** Promo rules are created in the dashboard and optionally synced with AlpineIQ for loyalty redemption. The promotion module handles condition targeting (product category, customer tier, order minimum) and discount type (flat, percentage, BOGO).

**Feature flags:** Some features are guarded by feature flags exposed through the store or account config (e.g., the hide-1g-bulk-flower flag visible in recent commits). These are checked inline in components — no dedicated feature flag service.

---

## Cross-Repo Connections

- **Calls**: GAP GraphQL API (primary backend — all domain data), OpenSearch cluster, Algolia, Prismic CMS, Google Maps API, Berbix (ID verification), Treez SSO service
- **Called by**: Nothing calls this dashboard; it is a terminal UI
- **Shared types / contracts**: `src/types/backend.generated.ts` is auto-generated from the GraphQL schema via `codegen.yaml` — any schema changes in the backend must be followed by `npm run generate:gc` here
- **Events**: None published or subscribed to (no event bus or WebSocket)

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS Cognito | Auth | Fallback authentication (username/password) |
| Treez SSO | Auth | Primary OAuth-based login for operators |
| Algolia | Search | Order and product search index |
| AWS OpenSearch | Search | Alternative/primary search for orders and products |
| Prismic | Storage / CMS | Homepage, promo carousel, and navigation content editing |
| GrapesJS | Other | In-browser drag-and-drop landing page builder |
| Google Maps | Other | Address autocomplete and delivery zone mapping |
| Berbix | Other | Customer identity verification |
| Aeropay | Payments | ACH payment gateway integration |
| AlpineIQ | Other | Loyalty program integration |
| QuickBooks | Other | Accounting sync integration |
| Dutchie | Other | POS integration (product/order sync) |
| Pendo | Analytics | User analytics and product tours |
| AWS Parameter Store (SSM) | Infrastructure | Env var management via `ssm.js` + Makefile |
| AWS Amplify | Auth / Infrastructure | Cognito auth SDK, used in fallback mode |

---

## Notable Patterns & Decisions

**Path aliases via Craco:** All imports use `@/`-prefixed aliases (`@/components`, `@/hooks`, `@/types`, `@/data`, etc.) configured in `craco.config.js` and `tsconfig.paths.json`. This makes imports location-agnostic but requires both Craco and TS configs to stay in sync.

**Apollo fragment registry:** Each domain module exports a `fragments` object alongside its query/mutation constants. Fragments are pre-built and registered via `build-fragment` on startup, avoiding runtime introspection. The `fragmentTypes.json` result of introspection is committed and must be regenerated when the schema adds union/interface types.

**Provider tower:** Eight nested providers wrap the app root (`QueryClientProvider → BrowserRouter → AuthProvider → ApolloProviderWrapper → StoreProvider → ScheduleProvider → ToastContainer`). Provider order is load-bearing — auth must resolve before Apollo is initialized, and store context depends on auth.

**Configuration-driven layout:** Sidebar navigation, header actions, and footer are driven by config objects in `src/config/layout/`, not hardcoded in components. Adding a new top-level nav section requires editing `aside.js` and `routes.js`, not layout components.

**`config/constant.js` (75 KB):** A single large file centralizing app-wide constants — status codes, label maps, dropdown options, regex patterns, integration credentials placeholders, etc. It is imported widely and has grown without clear ownership. Anything that doesn't fit elsewhere ends up here.

**Mixed data-fetching layers:** Apollo Client handles GraphQL (most data), TanStack React Query handles REST (search). This is intentional — each library is used for what it does best — but engineers need to know which layer a given page uses to debug effectively.

**Minimal test coverage:** Only one test file exists (`src/utils/utils.test.js`). AWS Amplify is mocked in `jest.setup.js`. The gap between test infrastructure and actual coverage is significant.

**CRA via Craco:** The project uses Create React App wrapped by Craco to add webpack aliases. CRA is no longer actively maintained upstream; this is a latent migration risk.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Tech debt | `config/constant.js` is 75 KB with no clear ownership; should be split by domain | High |
| Tech debt | Create React App (CRA) is unmaintained; Vite or Next.js migration would improve build times and DX | High |
| Testing | Near-zero test coverage; no component or integration tests despite complex business logic | High |
| Architecture | 71 form components in `pages/store/` with no higher-level grouping; difficult to navigate and maintain | Med |
| Architecture | Provider tower of 8 nested wrappers — some could be colocated closer to their consumers | Med |
| Tech debt | Apollo Client 2.6.8 is several major versions behind (current is v3); upgrade path has breaking changes | Med |
| Tech debt | React Router 5.1.2 is two major versions behind (v6 has different API) | Med |
| DX / tooling | `fragmentTypes.json` is committed and must be manually regenerated; automate in CI | Med |
| Performance | No code splitting beyond CRA defaults despite 300+ routes; initial bundle likely large | Med |
| Architecture | Feature flags checked inline in components with no central registry; hard to audit what's gated | Low |
| DX / tooling | No documented local setup guide; env var generation requires AWS credentials and tribal knowledge | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/dashboard (commit 09f1c76)*
