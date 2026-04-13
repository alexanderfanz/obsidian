# ops-infra: Store Manager Frontend

> React 18 SPA for internal operators to manage the Gap Commerce multi-tenant platform. Provides CRUD interfaces for Groups, Stores, and SellTreez (OpenSearch) clusters, with visibility into deployment workflows, infrastructure outputs, change history, and OpenSearch index mappings. Deployed via Vercel; authenticates against a dedicated AWS Cognito user pool.

---

## Purpose

The Store Manager frontend is an internal tool for the Gap Commerce operations team. It wraps the [[ops-infra-store-manager-backend]] REST API with a polished UI, letting operators:
- Create and configure merchant Groups and Stores (which trigger AWS CDK deployments)
- Manage SellTreez (OpenSearch) search clusters and their index mappings
- Browse the full audit history of any entity
- Inspect infrastructure outputs (API URLs, Cognito IDs, Lambda names) without touching the AWS console
- Track deployment workflow status (REQUESTED → IN_PROGRESS → SUCCEEDED/FAILED)

Out of scope: this is an internal ops tool only. Merchants never interact with this app.

---

## Key Entities / Domain Models

See [[ops-infra-store-manager-backend]] for entity definitions — the frontend models (`src/models/`) mirror those exactly. Key model files:
- `group.ts` — Group + Account types
- `store.ts` — Store type
- `sellTreez.ts` — SellTreez + Index types
- `record.ts` — History/audit record type
- `account.ts` — Account type
- `artifacts.ts` — S3 artifact folder listing type

---

## API Surface

The frontend consumes the Store Manager Backend API exclusively. All calls go through `src/services/dataSources/axiosConfig.ts` which injects the Cognito Bearer token. Base URL is set via `VITE_CORE_URL_BASE`.

Key service modules:
- `src/services/group/` — `GroupService` (getAllGroups, createGroup, updateGroup, getGroupHistory, getGroupAccounts, updateGroupBulk, getLastGroups)
- `src/services/store/` — `StoreService` (getAllStores, createStore, updateStore, getStoreHistory, getLastStores, updateStoreBulk)
- `src/services/sell-treez/` — `SellTreezService` (getAllSellTreez, createSellTreez, updateSellTreez, getSellTreezIndexes, getSellTreezIndexMappings, createSellTreezIndex, updateSellTreezIndexMapping, getSellTreezWorkflows)
- `src/services/record/` — `RecordService` (getLastChangesByLimit)
- `src/services/artifacts/` — `ArtifactService` (getArtifacts)

---

## Data Layer

No local persistence. All data fetched from the backend API via:
- **TanStack React Query v5** — query cache, automatic background refetching, mutation invalidation
- 5-second polling interval on the Groups list (`useGetAllGroups`)
- Query keys: `['all-groups']`, `['sellTreez', id]`, `['group', id]`, etc.
- No Redux, no Context API for server state

Local UI state managed with `useState` and `useReducer` (the Account page uses a reducer for its multi-modal flow).

---

## Core Business Logic

The frontend has minimal business logic — it is primarily a CRUD UI and infrastructure viewer. Non-obvious interactions:

### Deployment Lifecycle Display
Every entity card/page shows a `workflowStatus` badge (REQUESTED / IN_PROGRESS / SUCCEEDED / FAILED / STOPPED). The frontend doesn't poll the workflow endpoint; the Group/Store/SellTreez detail endpoints return `workflowStatus` inline. Operators see deployment state without a separate endpoint call.

### Infrastructure Output Parsing
The "Infra" tab on Group and Store detail pages parses the `outputs` field (a serialized JSON string from CloudFormation stack outputs). It extracts:
- API Gateway URLs (GraphQL, Webhook)
- Cognito User Pool ARN, Pool ID, Client ID
- Lambda function names
- SNS topic names / SQS queue URLs
- S3 bucket names

Each value has a clipboard button for quick copying. This is the primary way operators get endpoint URLs without opening the AWS console.

### Bulk Operations
The Groups and Stores list pages have a "Bulk Edit" modal that allows updating Lambda S3 source keys across all entities simultaneously. Used when a new Lambda version is released — operators paste the new S3 key and apply it to all groups or stores at once.

### OpenSearch Index Management (SellTreez)
The SellTreez detail page's Indexes tab allows:
- Creating indexes with field mappings via a JSON editor
- Viewing current mappings (full OpenSearch mapping JSON)
- Cloning indexes (copy one index definition to another name)
- Workflow comparison — diff two deployment events side by side

---

## Pages & Navigation

```
/ ─────────────── Dashboard (last N groups, last N stores, latest audit records)
/groups ────────── Groups list (card grid) → Create/Edit modal
/groups/:id ────── Group detail (tabs: Details, Infra, Accounts, History, Raw)
/groups/:id/:accountId ─── Account detail (tabs: Details, Raw) + inline store list
/groups/:id/:accountId/:storeId ── Store detail (tabs: Details, Infra, History, Raw)
/stores ────────── Stores list (ag-Grid table) → Create/Edit modal
/stores/:id ────── Store detail (same tabs as above)
/sell-treez ────── SellTreez list (card grid) → Create/Edit modal
/sell-treez/:id ── SellTreez detail (tabs: Details, Infra, Indexes, Workflows, History, Raw)
```

Navigation uses React Router v6 nested routes. The Group → Account → Store hierarchy is reflected in the URL path.

---

## Cross-Repo Connections

**Calls:**
- [[ops-infra-store-manager-backend]] — all data, via REST API over HTTPS

**Auth:**
- AWS Cognito User Pool provisioned by the backend CDK stack (`GapStoreManager-{branchName}`)
- Pool ID, client ID, identity pool ID set via Vite env vars

**Depends on:**
- Vercel for hosting (vercel.json present)
- AWS Amplify Auth SDK for token management

---

## Third-Party Services

| Service | Category | What it's used for |
|---------|----------|--------------------|
| AWS Cognito | Auth | User authentication (Amplify Authenticator wraps entire app) |
| Vercel | Infrastructure | SPA hosting and deployment |
| ag-Grid Community | UI | Data tables for stores, records, accounts, workflows, indexes |
| ApexCharts | Analytics | Bar chart component (infrastructure present, minimally used today) |

---

## Notable Patterns & Decisions

1. **AWS Amplify Authenticator wraps the entire app** — The `<Authenticator>` component from `@aws-amplify/ui-react` gates all routes. User sign-up is hidden (UI only shows sign-in). Password requirements: 8+ chars, mixed case, digits, symbols. No third-party OAuth.

2. **Axios interceptor for auth** — Every outbound request goes through an interceptor that calls `fetchAuthSession()` to get a fresh Cognito access token and injects `Authorization: Bearer {token}` header. Token refresh is handled transparently by Amplify.

3. **React Query mutation → cache invalidation pattern** — After any create/update, the relevant queries are invalidated, triggering a background refetch. This keeps the list/detail views consistent without manual state updates.

4. **ag-Grid for tabular data** — All list views that need sort/filter use ag-Grid (Community edition). Column definitions are typed, with custom cell renderers for action buttons and status badges. Double-click on a row navigates to the detail page.

5. **JSON editor in modals** — The OpenSearch index mapping editor and the index details viewer display raw JSON. No dedicated JSON editor library — uses `<textarea>` with JSON validation on submit.

6. **Tab-based detail pages** — Every detail page (Group, Store, SellTreez) uses Headless UI `<Tab>` components. The "Raw" tab always shows the full entity JSON for debugging.

7. **Card grid vs table** — Groups and SellTreez use card grids (better for visual metadata-heavy items). Stores use ag-Grid (better for large flat lists with filtering).

---

## Tech Stack

| Layer | Library | Version |
|-------|---------|---------|
| Framework | React | 18.3.1 |
| Language | TypeScript | 5.2.2 |
| Build | Vite | 5.3.4 |
| Styling | TailwindCSS | 3.4.6 |
| Routing | React Router DOM | 6.25.1 |
| Server state | TanStack React Query | 5.59.19 |
| Forms | React Hook Form + Yup | 7.52.1 / 1.4.0 |
| Tables | ag-Grid Community | 32.0.2 |
| Charts | ApexCharts + react-apexcharts | 3.51.0 |
| Auth | AWS Amplify Auth | 6.4.2 |
| HTTP | Axios | 1.7.2 |
| Notifications | React Toastify | 10.0.5 |
| Icons | Heroicons + React Icons | — |
| UI primitives | Headless UI | — |

---

## Potential Improvements

| Area | Observation | Priority |
|------|-------------|----------|
| Performance | Groups list polls every 5s — should use websockets or SSE for workflow status updates instead | Med |
| DX | No Storybook or component tests — components are untestable in isolation | Med |
| UX | The JSON editor for OpenSearch mappings is a raw textarea — a proper JSON editor (Monaco, CodeMirror) would prevent syntax errors | Med |
| Architecture | ApexCharts infrastructure is in place but charts aren't meaningfully used — either implement trend views or remove the dependency | Low |
| Auth | Sign-up is hidden in the UI but isn't actually disabled in Cognito — should be disabled at the pool level | Med |
| DX | No .env.example file — engineers need to hunt down env var names from the CDK stack outputs | Low |
| Tech debt | Some pages use `useReducer` for modal state while others use `useState` — inconsistent pattern | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/ops-infra/gap-store-manager/frontend (commit 9ed4512)*
