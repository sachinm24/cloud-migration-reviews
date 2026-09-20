## License

Unless otherwise noted, the original reports, methodology, checklists,
and documentation in this repository are licensed under the
[Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).

© 2026 Sachin Manpathak.

Third-party code, excerpts, diagrams, and materials remain subject to
their respective upstream licenses and attribution requirements.

The project name, logos, and other brand features are not licensed for use
without prior written permission.

# AWS → AzureMigration-Risk Review

## Subject

[aws-samples/amazon-sqs-best-practices-cdk](https://github.com/aws-samples/amazon-sqs-best-practices-cdk)

## Methodology

This review examines the application's use of AWS services and identifies
behavioral assumptions that may require validation during an AWS → Azure
migration.

Findings are based on the publicly available source code at the pinned
revision identified above.

A finding does not mean that the application cannot be migrated to Azure.
It means that the AWS behavior observed in the application should be
explicitly evaluated when selecting or implementing the corresponding
Azure service.

The review focuses on application-level behavior rather than simply
mapping AWS services to Azure service names.

## Review metadata

| Field | Value |
|---|---|
| Upstream repository | `aws-samples/amazon-sqs-best-practices-cdk` |
| Repository type | Public AWS sample |
| License | MIT-0 |
| Migration path | AWS → Azure |
| Assessment type | Educational public-code review |
| Scope | S3, Lambda, SQS, DynamoDB, and CDK configuration |
| Review date | 2026-09-20 |
| Revision reviewed | Replace with pinned commit SHA before publishing |

## Architecture summary

The sample describes an inventory-management workflow:

```text
CSV file upload
      |
      v
Amazon S3
      |
      v
Lambda: parse records
      |
      v
Amazon SQS
      |
      v
Lambda: process inventory update
      |
      v
Amazon DynamoDB
```

One possible Azure service mapping is:

```text
CSV file upload
      |
      v
Azure Blob Storage
      |
      v
Azure Functions: parse records
      |
      v
Azure Service Bus
      |
      v
Azure Functions: process inventory update
      |
      v
Azure Cosmos DB
```

This report does not prescribe that target design. It identifies behavioral assumptions that should be validated if this kind of workflow moves from AWS to Azure.

## Key findings

| ID | Severity | Confidence | Finding |
|---|---|---|---|
| [AWS-SQS-001](./findings/AWS-SQS-001.md) | High | High | Retry and duplicate-delivery behavior requires an explicit idempotency strategy |
| [AWS-SQS-002](./findings/AWS-SQS-002.md) | High | Medium | Message visibility / lock duration must cover worst-case processing and retries |
| [AWS-DDB-001](./findings/AWS-DDB-001.md) | High | High | Inventory-update correctness requires an explicit concurrency and conflict model |

## What this review is not

This is not:

- a complete migration assessment
- an architectural recommendation
- a performance benchmark
- a security assessment
- a statement that the application will fail on Azure

The findings identify areas where AWS-specific behavior may be
embedded in the application and therefore deserve investigation
during migration.

## Report

Read the complete [assessment report](./report.md).
