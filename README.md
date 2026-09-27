# Cloud Migration Reviews

Independent, source-based reviews of public AWS projects through the lens of
AWS-to-Azure migration. The reviews focus on application behavior and the
invariants a migration should preserve, rather than one-to-one service mapping.

These are static-analysis reviews, not production audits. Findings are review
questions; they do not establish a defect or prove that a project is unsafe.
The reports distinguish direct source evidence from inferred relationships and
state the analyzer's coverage limits.

## Reviews

| Project | Source revision | Scope and result | Report |
|---|---|---|---|
| [amazon-sqs-best-practices-cdk](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/tree/54143c7efd610b2345964951403bbde628f54ba0) | `54143c7` | AWS sample; resolved SQS → Lambda → DynamoDB path; 3 review findings | [Report](repos/amazon-sqs-best-practices-cdk/migration-readiness-report.md) · [JSON](repos/amazon-sqs-best-practices-cdk/migration-findings.json) |
| [asynchronous-event-processing-api-gateway-sqs-cdk](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/tree/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d) | `7c25cbd` | AWS sample; SQS/Fargate workflow; includes unresolved resource candidates and heuristic matches | [Report](repos/asynchronous-event-processing-api-gateway-sqs-cdk/migration-readiness-report.md) · [JSON](repos/asynchronous-event-processing-api-gateway-sqs-cdk/migration-findings.json) |
| [multiregion-s3-sns-sqs-lambda](https://github.com/aws-samples/multiregion-s3-sns-sqs-lambda/tree/e596ad0f05ec19891c2bd12a1cf0d44edbc2786f) | `e596ad0` | AWS sample; S3/SNS/SQS/Lambda workflow; candidate relationships remain unconfirmed | [Report](repos/multiregion-s3-sns-sqs-lambda/migration-readiness-report.md) · [JSON](repos/multiregion-s3-sns-sqs-lambda/migration-findings.json) |

The repositories above are public AWS samples, not a representative sample of
production customer applications. They demonstrate the review method and its
current coverage; they do not establish general precision or recall.

## Method

Each report records the upstream commit, scan date, offline status, observed
source evidence, potential invariant, target-cloud question, validation
scenario, and limitations. Source code is scanned locally and is not uploaded.
Manual review is identified separately from automated findings.

## License

Unless otherwise noted, original reports, methodology, checklists, and
documentation in this repository are licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). © 2026 Sachin
Manpathak. Third-party code and materials remain subject to their upstream
licenses and attribution requirements. Project names, logos, and other brand
features are not licensed for use without prior written permission.
