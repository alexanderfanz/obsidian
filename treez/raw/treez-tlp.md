# treez-tlp

> Next.js 15 storefront template powering 50+ cannabis dispensary e-commerce sites on the GapCommerce platform. Serves as the customer-facing UI layer, integrating Treez POS inventory, Prismic CMS content, and GapCommerce checkout/auth into a single multi-tenant web application.

---

## Purpose

Provides the complete web storefront for cannabis retail clients (dispensaries) built on GapCommerce. Responsible for: product discovery & browsing, age gate enforcement, CMS-driven page content, cart/checkout flow, user authentication, and kiosk self-service UI. Not responsible for order fulfillment, inventory management, or POS operations — those live in GapCommerce backend services and Treez.

---

## Key Entities / Domain Models

- **Product** — Cannabis product with fields: name, SKU, type (flower/edible/etc.), brand, category, strain, THC/CBD %, price tiers, lab results, images, and compliance flags. Sourced from Treez POS via OpenSearch.
- **Cart** — Managed by the `@gap-commerce/checkout` package; holds line items, quantities, applied discounts, and fulfillment type (pickup/delivery).
- **Order** — Submitted cart; includes payment method, customer info, fulfillment window, and Treez POS order reference.
- **Store** — Dispensary location with address, hours, delivery radius, available payment methods, and associated Treez store name.
- **Customer** — Authenticated user with profile, order history, wishlist, and medical/recreational status. Backed by AWS Cognito.
- **Promotion / Special** — Time-bounded discounts: bundle specials, BOGO, category discounts. Evaluated at cart time via `useCurrentPromotion` and `useBundleSpecial` hooks.
- **Page (CMS)** — Prismic document composed of up to 62 slice types (hero, product carousel, blog, etc.). Types include: homepage, category, brand, strain, blog post, location, and more (45+ custom types total).
- **AgeGate** — Cookie-based session record confirming the visitor has passed age verification. Has a configurable expiry.

---

## API Surface

This is a frontend app — it does not expose a traditional REST or RPC API. Internal API routes and public-facing pages:

| Method | Path / Name | Description |
|--------|-------------|-------------|
| GET/POST | `/api/revalidate` | ISR cache invalidation triggered by Prismic webhooks |
| GET | `/api/image` | Server-side WebP image conversion proxy |
| GET/POST | `/api/*` | CORS-enabled routes for GapCommerce Admin Dashboard |
| GET | `/slice-simulator` | Prismic Slice Machine preview environment |

Public-facing routes (Next.js App Router pages):

| Route | Description |
|-------|-------------|
| `/` | Homepage |
| `/shop`, `/search` | Product listing / search |
| `/products/[slug]` | Product detail page |
| `/categories/[slug]`, `/brands/[slug]`, `/strains/[slug]` | Taxonomy pages |
| `/checkout/*` | Checkout flow (via `@gap-commerce/checkout`) |
| `/account/*` | Auth and customer profile |
| `/locations/*` | Store locator |
| `/blog/*` | Blog posts |
| `/kiosk/*` | Self-service kiosk UI |
| `/(age)` | Age gate verification |
| `/(CMS)/*` | Prismic-driven dynamic content pages |

---

## Data Layer

- **No owned database.** All persistent data lives in GapCommerce backend services.
- **OpenSearch** — Product catalog and search index synced from Treez POS. Custom wrapper in `/opensearch/` handles query building, filter mapping, and result normalization. Supports filtering by: type, brand, category, effects, flavors, THC/CBD %, price range, store, and availability.
- **Apollo Client (GraphQL)** — Primary data access pattern for GapCommerce API. Cache managed client-side; server-side fetching via `/data/` functions for SSR/ISR routes.
- **Prismic CMS** — Headless CMS for all editable page content. Fetched server-side with ISR (`force-cache` in prod, 5s revalidation in dev). Cache invalidated via webhook → `/api/revalidate`.
- **Cookies** — Session state: age gate token, cart token, user preferences, fulfillment type, selected store.
- **Zustand store** — Client-side ephemeral state (cart UI, modal state, selected store, kiosk session).
- **AWS Cognito** — Customer identity and auth tokens via AWS Amplify.

---

## Core Business Logic

- **Age Gate** — Next.js middleware enforces age verification on every storefront route before the request reaches the page. Bots (detected via user-agent) bypass the gate for SEO. Cookie expiry is configurable. Medical vs. recreational flows may differ by client config.
- **Medical / Recreational Switching** — Feature-flagged (`FEATURE_FLAG_MED_REC_SELECTION_ACTIVE`). Users toggle their status, which filters available products and can affect pricing tiers.
- **Gram Limit Enforcement** — `NEXT_PUBLIC_GR_LIMIT` caps total cannabis weight per order per customer; enforced at cart level.
- **Promotions Engine** — Bundle specials, BOGO, and category discounts evaluated in real time. Logic lives in `@gap-commerce/checkout` but surfaced via `useCurrentPromotion` and `useBundleSpecial` hooks.
- **Multi-Store / Fulfillment** — Customers select pickup location or delivery; product availability and pricing can vary per store. Store selection is persisted in a cookie and drives the OpenSearch filter context throughout the session.
- **ISR Cache Strategy** — CMS pages are statically generated and revalidated on Prismic content publish via webhook. Prevents stale content without sacrificing SSG performance.
- **Kiosk Mode** — Isolated UI with idle timeout (`KIOSK_IDLE_TIMEOUT`), separate layout and providers, and a stripped-down checkout flow for in-store self-service terminals.

---

## Cross-Repo Connections

- **Calls**:
  - GapCommerce GraphQL API (`NEXT_PUBLIC_GRAPHQL_ENDPOINT_URL`) — product, order, store, and customer data
  - Treez POS (via GapCommerce backend / OpenSearch) — live inventory and product catalog
  - Prismic CMS — all editable page content
  - AWS Cognito — customer authentication
  - Berbix — ID verification (age and identity)

- **Called by**:
  - Prismic webhooks → `/api/revalidate` to trigger ISR cache invalidation
  - GapCommerce Admin Dashboard → CORS-enabled internal API routes

- **Shared types / contracts**:
  - `@gap-commerce/checkout` (private GitHub npm package) — shared checkout state, cart types, payment logic
  - GraphQL codegen output (`__generated__/`) — typed GapCommerce API contracts auto-generated from schema

- **Events**:
  - Publishes analytics events to AWS Amplify Analytics, Google Analytics, and FullStory
  - Consumes Prismic webhook events for cache invalidation

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| Prismic | Other (CMS) | Headless CMS for all page content and marketing slices |
| Treez POS | Other (POS) | Cannabis inventory, product catalog, order submission |
| AWS Cognito / Amplify | Auth | Customer authentication and user pools |
| OpenSearch / Elasticsearch | Search | Primary product search and filtering |
| Algolia | Search | Optional/backup product search index |
| Stripe | Payments | Online card payment processing |
| Webpay | Payments | Alternative debit payment method |
| ACH | Payments | Bank transfer payments |
| Google Maps | Infrastructure | Store locator and delivery radius display |
| Berbix | Auth | ID and age verification |
| AWS Amplify Analytics | Analytics | Customer event tracking |
| Google Analytics | Analytics | Web traffic and conversion analytics |
| FullStory | Analytics | Session recording and ecommerce event tracking |
| Vercel | Infrastructure | Hosting, CDN, ISR, and deployment |
| Vercel Speed Insights | Monitoring | Core Web Vitals and performance monitoring |
| i18next / next-i18next | Other | Internationalization / multi-language support |

---

## Notable Patterns & Decisions

- **Multi-tenant via env vars** — A single codebase serves 50+ dispensary chains. Client identity (`ACCOUNT_ID`, `STORE_ID`), theming, feature flags, and store configs are all driven by environment variables, allowing Vercel to deploy per-client instances from the same repo.
- **Prismic Slice Machine (62 slices)** — Content editors compose pages from a library of 62 pre-built slices without code changes. This is the primary mechanism for client-specific content customization. Slice Simulator at `/slice-simulator` enables live preview.
- **App Router with route groups** — Next.js 15 App Router with `(Storefront)`, `(Checkout)`, `(Auth)`, `(Age)`, `(CMS)`, and `(Kiosk)` route groups. Each group has its own layout, providers, and middleware behavior — a clean isolation pattern.
- **Atomic component design** — 161 components structured atoms → molecules → organisms. Each component is a self-contained folder with `.tsx`, `.module.scss`, and `index.ts`.
- **Hook-first logic** — Business logic lives in 48 custom hooks rather than in components. Hooks are the primary unit of reuse and the right place to look for domain logic.
- **OpenSearch wrapper layer** — `/opensearch/` acts as an anti-corruption layer between the app and the search index, handling query building, filter mapping (`NEXT_PUBLIC_TREEZ_FILTERS_TITLE_MAPPING`), and result normalization.
- **Feature flags via env vars** — ~12 `NEXT_PUBLIC_FEATURE_FLAG_*` booleans control capability rollout per client. No runtime flag management service; enabling/disabling a flag requires a redeployment.
- **1Password CLI for secrets** — Developers inject env variables via `op` CLI rather than sharing `.env` files. `.env.tlp` is the 1Password template; `.env.example` is the schema reference.
- **Build-time version tagging** — `build-info.js` writes a build manifest (git SHA, timestamp, env) for deployment traceability.

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Security | Multiple sensitive keys (`TREEZ_CLIENT_SECRET`, Algolia write keys, etc.) are prefixed `NEXT_PUBLIC_`, exposing them to the browser bundle. These should be proxied through server-side API routes | High |
| Testing | 48 custom hooks and 161 components but test coverage appears light. Hooks containing business logic (promotions, gram limits, med/rec switching) are high-ROI targets | High |
| Feature flags | Flags are env-var booleans with no runtime toggle, targeting, or gradual rollout. A proper flag service (e.g., GrowthBook) would eliminate redeployments for flag changes | Med |
| Dual search backends | Algolia and OpenSearch are both present. The relationship, fallback behavior, and migration intent are unclear — adds maintenance surface and potential inconsistency | Med |
| Multi-tenant config | Client config is fully env-var driven with no schema validation. A typed, validated config layer would catch misconfiguration at build time rather than runtime | Med |
| Bundle size | `size-limit` and bundle analyzer are configured, suggesting this is a known concern. Per-client analysis would help since env-driven feature inclusion affects bundle differently across deployments | Low |
| Internal API docs | No API contract documentation for internal routes. GraphQL codegen covers the GapCommerce contract but internal `/api/*` routes are undocumented | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/treez-tlp (branch: master, commit: 600eab80)*
