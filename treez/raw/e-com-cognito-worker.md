# e-com-cognito-worker

> AWS Lambda microservice that processes Amazon Cognito user lifecycle events for the Gap Commerce (Emberz) e-commerce platform. On each trigger (sign-up, post-confirmation, password recovery, user migration), it auto-verifies users, syncs accounts to DynamoDB, creates users in Jane, subscribes them to CRM platforms, and fires analytics events.

---

## Purpose

Handles the full Cognito trigger surface so that other services don't need Cognito awareness. When a shopper signs up or confirms their account, this worker is the single place responsible for: auto-confirming/verifying based on store config, writing the canonical Account record to DynamoDB, creating the corresponding Jane platform user, subscribing to email/SMS/loyalty CRMs (Klaviyo, ListTrack, AlpineIQ), and emitting a signup event to NewRelic. It also enables legacy user migration by checking Jane for an existing user when a forgotten-password flow begins.

Out of scope: order processing, inventory, product catalog, payment, and any post-login business logic beyond what is needed to bootstrap the account.

---

## Key Entities / Domain Models

- **CognitoEvent** — extends the AWS Lambda Cognito trigger event. Carries `TriggerSource` (e.g. `PostConfirmation_ConfirmSignUp`), typed `Request`/`Response` payloads, and helper methods (`userAttribute()`, `clientMetadata()`, `getPreSignupResponse()`). The same struct covers all trigger types; fields not relevant to a given trigger are zero-valued.

- **Account** (from `srv-emberz`) — the canonical shopper record stored in DynamoDB. Key fields: `EntityID`, `Email`, `FirstName`, `LastName`, `Phone`, `DateOfBirth`, `CountryCode`, address fields, `AcceptMarketing`, `TotalOrderExpense`, `Type` (ADULT or MEDICAL). Medical accounts carry `MedicalID` and `MedExpirationDate`. DynamoDB key prefix: `"c#" + EntityID`.

- **Store config** (S3 `store.json`) — per-store feature flags including `AutoConfirmUser`, `AutoVerifyEmail`, `AutoVerifyPhone`, and CRM configuration (Klaviyo list IDs, ListTrack source, AlpineIQ store slugs, Jane integration toggle).

- **App config** (S3 `apps.json`) — per-application metadata including Jane credentials and SNS topic mappings.

- **SubscribeCRM** — internal struct that maps an Account to the CRM-specific payloads required by each platform subscription call.

---

## API Surface

This service has no inbound HTTP API. It is invoked exclusively by Amazon Cognito as a Lambda trigger.

| Trigger Source | Description |
|---|---|
| `PreSignUp_SignUp` | Check Jane for existing user (migration); auto-confirm/verify if store config permits |
| `PreSignUp_AdminCreateUser` | Same auto-confirm/verify path for admin-created users |
| `PostConfirmation_ConfirmSignUp` | Main flow: add to CUSTOMER group, write Account to DynamoDB, create Jane user, CRM subscriptions, NewRelic event |
| `PostConfirmation_ConfirmForgotPassword` | Create Account if not exists, add custom attributes, create Jane user |
| `PreAuthentication_Authentication` | Pass-through (no-op beyond logging) |
| `PostAuthentication_Authentication` | Pass-through |
| `UserMigration_ForgotPassword` | Check Jane for existing user; if found, set migration response |
| `CustomMessage_*` | Pass-through for all custom message sub-triggers |

**Outbound calls (fire-and-forget or sync):**
- Jane API — `CheckUserExist`, `CreateUser`
- Klaviyo — list subscription
- ListTrack — signup event + email subscription
- AlpineIQ — favorite store assignment
- NewRelic Events API — custom `ecom_signup` event
- AWS SNS — publish to CRM worker topic

---

## Data Layer

- **DynamoDB** — primary account store. Accounts are keyed with a `"c#"` prefix on `EntityID` to support GSI queries. Also reads order data tables. All DynamoDB operations are delegated to the `Account` service from `srv-emberz`.
- **S3** — read-only config source. Two objects per environment: `store.json` (store feature flags) and `apps.json` (app-level credentials and mappings). Loaded at Lambda cold-start and cached in memory for the lifetime of the execution environment.
- No relational database. No caching layer beyond the in-memory S3 config loaded on cold start.

---

## Core Business Logic

**Auto-confirm / auto-verify** — On `PreSignUp`, the worker reads the store's `AutoConfirmUser`, `AutoVerifyEmail`, and `AutoVerifyPhone` flags and sets the corresponding Cognito response fields. This lets stores opt into skipping email/phone verification entirely.

**Account bootstrap on confirmation** — `PostConfirmation_ConfirmSignUp` is the most complex trigger. It:
1. Adds the user to Cognito's `CUSTOMER` group.
2. Reads custom Cognito attributes (`custom:ACCOUNT_ID`, `custom:STORE_ID`, `custom:COUNTRY_CODE`, `custom:ACCEPT_MARKETING`, `custom:ACCOUNT_TYPE`, medical fields, rewards store ID) and maps them onto an `Account` struct.
3. Upserts the Account in DynamoDB (create if not exists, update if already present).
4. Conditionally creates the user in Jane (feature-flagged per store).
5. Fires CRM subscriptions concurrently via goroutines if `AcceptMarketing` is true.
6. Emits a NewRelic signup event unconditionally.

**Concurrent CRM subscriptions** — Klaviyo, ListTrack, and AlpineIQ subscriptions run in parallel via `sync.WaitGroup`. Errors are logged but do not halt processing or fail the Cognito trigger.

**User migration for forgotten passwords** — `UserMigration_ForgotPassword` allows shoppers with a Jane account (but not yet in Cognito) to reset their password. The worker checks Jane; if the user exists it returns a migration response, letting Cognito create the user record on the fly.

**Medical vs. adult account type** — `custom:ACCOUNT_TYPE` gates medical ID and expiration date fields. Medical users carry compliance-relevant data surfaced in CRM subscriptions.

**Rewards store handling** — `custom:REWARDS_STORE_ID` is treated separately from the primary store ID in CRM payloads; AlpineIQ receives the rewards store slug, not the transacting store slug.

---

## Cross-Repo Connections

- **Calls:**
  - `github.com/gap-commerce/srv-emberz` — shared service library providing Account, Store, App, Jane, and ListTrack service implementations. All AWS and third-party operations are wrapped here.
  - Jane platform API (via `srv-emberz` Jane service)
  - NewRelic Events API (direct HTTP)
  - Klaviyo, ListTrack, AlpineIQ (via `srv-emberz` CRM services or SNS)

- **Called by:**
  - Amazon Cognito User Pools (Lambda trigger invocation) — no other internal service calls this directly.

- **Events published:**
  - AWS SNS topic (`gc-topic`) — CRM worker messages for downstream processing.

- **Shared types / contracts:**
  - `Account`, `Store`, `App` models from `srv-emberz`. Any schema change to these in `srv-emberz` requires a dependency update here.

---

## Third-Party Services

| Service | Category | What it's used for |
|---|---|---|
| AWS Cognito | Auth | User pool trigger source; user group assignment; custom attribute storage |
| AWS DynamoDB | Infrastructure | Account persistence and order data reads |
| AWS S3 | Infrastructure | Store and app config loading |
| AWS SNS | Infrastructure | Publishing events to CRM worker |
| AWS Lambda | Infrastructure | Runtime environment |
| Jane | Other | E-commerce platform user existence check and user creation |
| Klaviyo | Email | Marketing list subscription on signup |
| ListTrack | Email | Transactional signup event + email subscription |
| AlpineIQ | Other | Loyalty/rewards favorite store assignment |
| New Relic | Monitoring | Custom `ecom_signup` event instrumentation |
| Sentry | Monitoring | Panic capture and error tracking |

---

## Notable Patterns & Decisions

**No HTTP server** — Pure Lambda; no Gin/Echo/net-http. The entire surface is Cognito triggers. This keeps cold-start overhead minimal and removes the need for API Gateway.

**Service struct injection** — All AWS clients (Cognito, DynamoDB, S3, SNS) and business service implementations are initialized in `cmd/main.go` and injected into `App` via the `Service` struct. This makes dependencies explicit and theoretically testable, though no tests currently use this.

**Goroutine parallelism for CRM** — Concurrent subscriptions reduce tail latency on confirmation flows where multiple CRM platforms are configured. Errors from any single platform are absorbed (logged, not returned), so a Klaviyo timeout doesn't fail the Cognito trigger and block user sign-up.

**S3 as config store** — Rather than environment variables for all store/app configuration, feature flags and credentials are stored in JSON objects in S3, loaded at cold start. This lets per-store configuration change without a Lambda redeployment.

**Dual observability** — NewRelic for business-event instrumentation (custom events, timing), Sentry for exception tracking. Both are flushed with a deferred timeout at handler exit to ensure events aren't lost on Lambda container reuse.

**Conventional commits + semantic-release** — Version numbers are derived from commit messages automatically on merge to master. The CI pipeline tags the release, builds a Linux ARM64 binary, and uploads it to S3.

**ARM64 Lambda target** — Compiled for `GOARCH=arm64` (Graviton2), which gives better price/performance on Lambda than x86.

---

## Potential Improvements

| Area | Observation | Priority |
|---|---|---|
| Testing | No test files exist. The `Service` injection pattern is already in place to support mocking, but nothing is tested. A regression in trigger routing or attribute mapping would be invisible until prod. | High |
| Error handling | CRM subscription failures are silently absorbed. There is no dead-letter path or retry mechanism — a ListTrack outage during signup means that user is never subscribed and there's no way to recover without replaying the event. | High |
| S3 config reload | Store/app config is loaded once at cold start. Config changes require a Lambda cold start (redeploy or function update) to take effect. A short TTL cache or config refresh on each invocation would reduce deployment toil. | Med |
| Hardcoded region | `AWS_REGION` is hardcoded to `us-west-1` in the initialization code. Multi-region deployment or moving to a different region requires a code change. | Med |
| Cognito group assignment error handling | If adding a user to the `CUSTOMER` group fails, the error is logged but the handler continues. The user ends up in Cognito but without proper group membership, potentially breaking downstream authorization checks. | Med |
| DX / tooling | The `configs/` directory contains committed `.env.*` files including non-prod credentials and NewRelic API keys. These should be in a secrets manager, not in version control. | High |
| Architecture | The `PostConfirmation_ConfirmSignUp` handler does too many things (group assignment, DDB write, Jane sync, CRM, analytics). Splitting these into composable handlers or using an event-driven fanout pattern would improve maintainability. | Low |

---

*Generated: 2026-04-13 — Source: /Users/alex/repos/treez/gap/e-com-cognito-worker @ 8406d29*
