# Dependency Map

How services depend on each other across the GapCommerce platform.

## Library dependencies

```
e-com-lib ◄── e-com-srv
           ◄── e-com-order-worker
           ◄── e-com-app-worker
           ◄── e-com-notification-worker
           ◄── api-srv
           ◄── (e-com-cognito-worker via srv-emberz)

selltreez-lib ◄── selltreez-injection-srv
               ◄── selltreez-api-srv

srv-utils ◄── (all Go services for config loading)
glog      ◄── (all Go services for structured logging)
```

## Service-to-service calls

```
treez-tlp ──GraphQL──▶ e-com-srv
gap-dashboard ──GraphQL──▶ e-com-srv
treez-tlp ──REST──▶ selltreez-api-srv
Store Manager Frontend ──REST──▶ Store Manager Backend
```

## Event flow (SNS/SQS)

```
e-com-srv
  │ SNS (gc-topic)
  ├──▶ e-com-order-worker (order/completed, order/kiosk-completed)
  └──▶ e-com-notification-worker (various)

e-com-order-worker
  │ SNS (gc-topic)
  ├──▶ e-com-app-worker (place-order-treez/blaze/jane/klaviyo)
  └──▶ e-com-notification-worker (order_status_changed_notify)

e-com-app-worker
  │ SNS (gc-topic)
  └──▶ e-com-notification-worker (completion events)

api-srv
  │ SNS
  └──▶ e-com-notification-worker (OrderStatusChangedNotify)

e-com-cognito-worker
  │ SNS (gc-topic)
  └──▶ CRM worker (downstream)
```

## Event flow (EventBridge)

```
Store Manager Backend
  │ EventBridge
  ├──▶ inventory-manager → actions-manager → CodeBuild
  └──▶ build-status-manager (on CodeBuild completion)

External POS (Treez)
  │ EventBridge (cross-account)
  └──▶ selltreez-injection-srv (product/discount/config changes)
```

## Webhook flow (inbound from external)

```
Treez POS ──webhook──▶ api-srv
Jane POS ──webhook──▶ api-srv
Blaze ERP ──webhook──▶ api-srv
OnFleet ──webhook──▶ api-srv
Prismic CMS ──webhook──▶ treez-tlp (/api/revalidate)
```

## Infrastructure dependencies

```
Store Manager Backend ──CodeBuild──▶ Group CDK
                       ──CodeBuild──▶ Store CDK
                       ──CodeBuild──▶ SellTreez CDK

Group CDK ──provides SNS/S3──▶ Store CDK (parent→child)
Group CDK ──deploys──▶ e-com-srv (graphql Lambda)
           ──deploys──▶ e-com-srv (webhook Lambda)
           ──deploys──▶ e-com-order-worker
           ──deploys──▶ e-com-app-worker
           ──deploys──▶ e-com-notification-worker

Store CDK ──deploys──▶ e-com-cognito-worker (Cognito trigger)
           ──deploys──▶ DynamoDB stream processor (search indexing)

SellTreez CDK ──deploys──▶ selltreez-api-srv (reader)
               ──deploys──▶ selltreez-injection-srv (writer)
               ──reads SSM──▶ TrackingTreez CDK (Firehose stream name)
```

## Shared infrastructure

| Resource | Used by |
|----------|---------|
| SNS topic `gc-topic` | e-com-srv, e-com-order-worker, e-com-app-worker, e-com-notification-worker, e-com-cognito-worker, api-srv |
| S3 config bucket (per group) | All backend services (store.json, apps.json, products, etc.) |
| Cognito (per store) | treez-tlp, gap-dashboard, e-com-cognito-worker, e-com-srv |
| Secrets Manager `/gap-store-manager/allSecrets` | All CDK stacks |
| SSM Parameter Store | All Lambda services (config at cold start) |
