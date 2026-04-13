# e-com-cognito-worker

Processes Cognito user lifecycle events — auto-confirms users, bootstraps accounts, syncs to Jane, subscribes to CRMs.

**Repo:** [gap-commerce/e-com-cognito-worker](https://github.com/gap-commerce/e-com-cognito-worker) | **Language:** Go | **Runtime:** AWS Lambda (ARM64) | **Trigger:** Cognito Lambda triggers

## Trigger types handled

| Trigger | Action |
|---------|--------|
| `PreSignUp_SignUp` | Check Jane for existing user; auto-confirm/verify per store config |
| `PreSignUp_AdminCreateUser` | Same auto-confirm/verify path |
| `PostConfirmation_ConfirmSignUp` | **Main flow**: add to CUSTOMER group, write Account to DynamoDB, create Jane user, CRM subscriptions, NewRelic event |
| `PostConfirmation_ConfirmForgotPassword` | Create Account if not exists, add custom attributes, create Jane user |
| `UserMigration_ForgotPassword` | Check Jane for existing user; enable migration if found |

## Key behaviors

- **Account bootstrap** — PostConfirmation reads Cognito custom attributes (ACCOUNT_ID, STORE_ID, COUNTRY_CODE, ACCOUNT_TYPE, etc.) and maps them to an [[Account]] struct in DynamoDB.
- **Concurrent CRM subscriptions** — Klaviyo, ListTrack, AlpineIQ run in parallel via `sync.WaitGroup`. Errors are absorbed (don't fail the Cognito trigger).
- **Medical vs. adult** — `custom:ACCOUNT_TYPE` gates medical ID and expiration date fields.
- **User migration** — Forgotten-password flow checks Jane for existing users, enabling passwordless migration into Cognito.

## Dependencies

- `srv-emberz` — shared service library (Account, Store, App, Jane, ListTrack services)
- Note: uses `srv-emberz` rather than [[e-com-lib]] directly
- Publishes to SNS `gc-topic` for CRM worker downstream processing

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Testing | No test files exist despite injection-friendly architecture | High |
| Error handling | CRM failures silently absorbed — no retry or DLQ for missed subscriptions | High |
| DX | `.env.*` files with non-prod credentials committed to repo | High |
| Config | S3 config loaded once at cold start; changes require redeploy | Med |

*Source: [[raw/e-com-cognito-worker.md]]*
