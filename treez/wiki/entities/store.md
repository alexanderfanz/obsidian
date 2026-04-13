# Store

A retail dispensary location — the primary tenant boundary for customer-facing operations.

## Where it lives

| System | Purpose |
|--------|---------|
| DynamoDB (Store Manager) | Entity metadata, deployment config. Managed via [[Store Manager Backend]]. |
| S3 (`store.json`) | Per-store configuration loaded by all backend services. |
| [[Store CDK]] | Isolated AWS infrastructure: Cognito pool, DynamoDB table, stream processor. |
| [[e-com-lib]] | Domain model definition with service interface. |

## Key fields

Payment methods, menu type, tax settings, geo config, operating hours, feature flags (e.g. `1g_bulk_flower`, `treez_medical_id_resend`), `DynamoOrderTableName`, `FromTxEmail`, branding fields, `EmailNotificationActive`, `AutoConfirmUser`, `AutoVerifyEmail`, `AutoVerifyPhone`, delivery zones.

## Relationship hierarchy

```
Group (merchant organization)
  └── Account (e-commerce account ID)
        └── Store (individual location)
```

A [[Group]] can have many Accounts; each Account has many Stores. The Group provides the API layer (GraphQL, webhook Lambdas, SNS/SQS). The Store provides the data layer (DynamoDB table, Cognito pool).

## Multi-tenancy scoping

All business logic operates within a `(accountID, storeID)` scope:
- S3 keys: `{accountID}-{storeID}-{keyName}` or `{accountID}/{storeID}/{resource}`
- DynamoDB tables: per-store, name resolved from config
- Cognito: per-store user pool
- OpenSearch: per-tenant index `{prefix}_{orgId}_{entityId}`

See [[Multi-Tenancy]] for the full pattern.

## Infrastructure per store

The [[Store CDK]] creates for each store:
- DynamoDB table with streams
- Cognito User Pool (customer auth)
- Stream processor Lambda (DynamoDB → OpenSearch)
- Cognito trigger Lambda ([[e-com-cognito-worker]])
- SES email identity (optional)

## Store configuration

The `store.json` file in S3 drives behavior across the platform:
- Feature flags control which integrations are active
- `apps.json` holds per-integration credentials
- Notification templates in `notification_settings.json`
- Products, promotions, categories, brands, navigation in separate S3 objects
