# Store Manager Frontend

React SPA for internal operators to manage the GapCommerce multi-tenant platform.

**Language:** TypeScript/React 18 | **Build:** Vite | **Hosting:** Vercel | **Part of:** [[ops-infra]]

## What operators do here

- Create and configure [[Group]]s and [[Store]]s (triggering CDK deployments)
- Manage SellTreez (OpenSearch) clusters and index mappings
- Browse audit history of any entity
- Inspect infrastructure outputs (API URLs, Cognito IDs, Lambda names) without the AWS console
- Track deployment workflow status (REQUESTED → IN_PROGRESS → SUCCEEDED/FAILED)
- Bulk update Lambda S3 source keys across all entities

## Tech stack

React 18, Vite, TailwindCSS, TanStack React Query v5, React Router v6, React Hook Form + Yup, ag-Grid Community, AWS Amplify Auth, Axios

## Key patterns

- **TanStack Query for server state** — 5-second polling on Groups list. No Redux.
- **ag-Grid** for tabular data (stores, records, accounts, workflows, indexes).
- **Amplify Authenticator** wraps entire app. Sign-up hidden in UI but not disabled in Cognito.
- **Axios interceptor** injects fresh Cognito Bearer token on every request.

## Dependencies

- [[Store Manager Backend]] — all data via REST API
- AWS Cognito — operator authentication

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Performance | 5-second polling — should use websockets/SSE | Med |
| DX | No Storybook or component tests | Med |
| Auth | Sign-up hidden in UI but not disabled at pool level | Med |

*Source: [[raw/ops-infra-store-manager-frontend.md]]*
