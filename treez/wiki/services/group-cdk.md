# Group CDK

AWS CDK stack provisioning all infrastructure for a single merchant Group. Deployed N times (once per group) by CodeBuild.

**Language:** TypeScript | **IaC:** AWS CDK | **Part of:** [[ops-infra]]

## Resources created per Group

- **5 Lambda functions** — graphql, webhook, order-worker, app-worker, notification-worker (all 1024 MB / 300s)
- **2 REST API Gateways** — GraphQL API + Webhook API with custom domains
- **SNS topic** + 3 SQS queues with DLQs (one per worker) + filter policies
- **Cognito User Pool** — per-group with custom attributes and roles (ADMIN, SALLER, SUPER_ADMIN)
- **3 S3 buckets** — config stores, account photos (CORS), CDN (CloudFront)
- **CloudFront distribution** for static assets
- **OpenSearch Serverless** (optional) for group-level search
- **SES email identity** (optional) with DKIM Route 53 records

## Key patterns

- **Pattern-matched DynamoDB IAM** — policy allows access to `{stage}-*-store-stack-table*`. New stores automatically fall within the pattern.
- **SNS fan-out** — single topic per group; workers subscribe with filter policies.
- **Stack deployed N times** — CDK parameterized via CodeBuild env vars. Tenant isolation via CloudFormation stacks.
- **CloudFormation outputs as entity data** — serialized to JSON, stored back in Store Manager DynamoDB.

## Cross-repo connections

- Lambda source code from: [[e-com-srv]] (graphql), [[e-com-srv]] (webhook), [[e-com-order-worker]], [[e-com-app-worker]], [[e-com-notification-worker]]
- Parent of: [[Store CDK]] (stores reference this group's SNS topic and S3 bucket)
- Deployed by: [[Store Manager Backend]] via CodeBuild

## Notable concerns

| Area | Issue | Priority |
|------|-------|----------|
| Security | DynamoDB IAM wildcard grants access to all stage-matching tables | Med |
| Testing | No CDK stack tests | Med |
| Cost | All Lambdas at 1024 MB even for simple workers | Low |

*Source: [[raw/ops-infra-group-cdk.md]]*
