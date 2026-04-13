# SellTreez CDK

AWS CDK stack provisioning a dedicated OpenSearch-powered product search cluster.

**Language:** TypeScript | **IaC:** AWS CDK | **Part of:** [[ops-infra]]

## Resources created per SellTreez cluster

- **OpenSearch domain** — v2.19, fine-grained access control, TLS, encryption at rest, credentials in Secrets Manager
- **2 Lambda functions** — reader (search queries, API Gateway trigger) and writer (indexing, SQS trigger). Memory and timeout are operator-configurable.
- **EventBridge bus + rule** — matches on source + detailType, routes to SQS. Supports cross-account publishing.
- **SQS queue + DLQ** — 3-second delivery delay for write batching
- **REST API Gateway** — API key auth (`X-Api-Key`), optional custom domain
- **DynamoDB table** — metadata storage (PK: key, SK: sort_key)
- **Kinesis Firehose integration** (optional) — search events → [[TrackingTreez CDK]]

## Key patterns

- **Event-driven ingestion** — external accounts publish to EventBridge → SQS buffers → writer Lambda indexes to OpenSearch.
- **Cross-account via resource policy** — no shared credentials or VPC peering needed.
- **3-second SQS delay** — batches events for bulk OpenSearch writes.
- **API key auth** — simpler than Cognito for server-to-server search queries.
- **Operator-configurable sizing** — Lambda memory/timeout stored in entity config, applied at CDK deploy time.

## Cross-repo connections

- Lambda source: [[selltreez-api-srv]] (reader), [[selltreez-injection-srv]] (writer)
- Feeds analytics to: [[TrackingTreez CDK]] (via Firehose)
- Deployed by: [[Store Manager Backend]] via CodeBuild

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Observability | No CloudWatch alarms for OpenSearch cluster health | High |
| Security | API key doesn't support rotation | Med |
| Testing | No CDK stack tests | Med |

*Source: [[raw/ops-infra-selltreez-cdk.md]]*
