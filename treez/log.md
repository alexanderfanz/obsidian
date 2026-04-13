# Log

Append-only record of ingests, queries, and lint passes.

## [2026-04-13] ingest | Initial ingest — 18 repo documents

**Sources:** e-com-lib, e-com-srv, e-com-notification-worker, e-com-order-worker, e-com-app-worker, e-com-cognito-worker, api-srv, selltreez-lib, selltreez-injection-srv, selltreez-api-srv, ops-infra (overview), ops-infra-store-manager-backend, ops-infra-store-manager-frontend, ops-infra-group-cdk, ops-infra-store-cdk, ops-infra-selltreez-cdk, ops-infra-trackingtreez-cdk, gap-dashboard, treez-tlp

**Pages created:** ~35 wiki pages across services, entities, concepts, and cross-cutting maps.

**Summary:** First full ingest of the Treez Ecom / GapCommerce platform. Covers the complete stack: shared libraries (e-com-lib, selltreez-lib), GraphQL API (e-com-srv), webhook service (api-srv), 4 async workers, SellTreez search pipeline (injection + API), infrastructure-as-code (ops-infra with 6 CDK stacks + Store Manager control plane), admin dashboard (gap-dashboard), and customer storefront (treez-tlp). Cross-repo synthesis produced entity pages, concept pages, a dependency map, a third-party services map, and a consolidated improvements tracker.

## [2026-04-13] ingest | Order lifecycle deep-dive — 8 targeted repo documents

**Sources:** raw/order-lifecycle/e-com-lib-order.md, e-com-srv-order.md, api-srv-order.md, e-com-order-worker-order.md, e-com-app-worker-order.md, e-com-notification-worker-order.md, treez-tlp-order.md, gap-dashboard-order.md

**Pages updated:** wiki/concepts/order-lifecycle.md (rewritten from overview to exhaustive deep-dive)

**Summary:** Second ingest targeting the order lifecycle specifically. Replaced the high-level overview with a comprehensive page covering: all 12 status values and their transitions, 4 POS-specific cart mutations, price calculation per POS, the TreezPay duplicate ticket bug, all 4 payment provider flows, ID verification (Berbix + IDScan), the 6-step post-purchase pipeline, Treez/Blaze/Jane/Klaviyo async processors, Alpine IQ + Sticky discount chaining, all 4 webhook providers with full status mappings, OnFleet 6-event delivery lifecycle, admin status transitions with Treez/OnFleet rollback, notification templates with 6 cross-processor bugs identified, and the complete event flow diagram.
