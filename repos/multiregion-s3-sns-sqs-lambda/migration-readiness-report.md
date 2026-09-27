# AWS-to-Azure Migration Semantic-Risk Review

## Executive summary

Source repository: [aws-samples/multiregion-s3-sns-sqs-lambda](https://github.com/aws-samples/multiregion-s3-sns-sqs-lambda/tree/e596ad0f05ec19891c2bd12a1cf0d44edbc2786f)  
Reviewed commit: `e596ad0f05ec19891c2bd12a1cf0d44edbc2786f`

Scan date: 2026-09-27  
Offline mode: enabled  
Files scanned: 9  
Files skipped: 0  
Findings generated: 2

This report identifies application behaviors and infrastructure configurations
that may encode assumptions about AWS semantics and therefore require validation
when migrating to Azure. It is not a security audit, production-readiness
certification, or guarantee of compatibility.

### Severity and confidence

**Severity** describes the review priority supported by the available evidence:

- **High:** a resolved configuration or code path shows a likely correctness,
  data-integrity, duplicate-effect, loss-of-work, or recovery failure condition.
- **Medium:** a meaningful migration behavior needs explicit mapping and testing,
  but a failure condition has not been shown.
- **Informational:** a relevant platform pattern needs documentation or review
  without a demonstrated correctness concern.

**Confidence** describes how strongly the available static evidence supports
the finding.

A finding with high severity and low confidence is therefore a high-impact
review question, not a claim that the application has a confirmed defect.

Each finding follows: observed behavior → source behavior → potential invariant
→ target-cloud question → validation. Potential invariants are review candidates
for the workload owner to confirm, not inferred business requirements.

## Findings summary

| ID | Severity | Confidence | Detection | Finding |
|---|---|---|---|---|
| AWS-DDB-001 | Medium | Low | Text-pattern heuristic | DynamoDB PutItem replay and overwrite behavior requires validation |
| AWS-SQS-001 | Medium | Low | Python AST | Review SQS consumers for duplicate-delivery safety |

## Assessment boundaries

The scanner analyzes eligible local text files only. It excludes common secret files, credential directories, generated artifacts, VCS metadata, and files containing likely credential material.

Static analysis cannot determine production traffic, deployment topology, runtime configuration, external-system behavior, business invariants, or operational procedures unless represented in scanned files.

SQS detection uses Python syntax trees (ASTs) to recognize boto3 and AWS CDK calls with locally resolved imports and bindings. It does not resolve cross-file aliases, runtime factories, control flow, instance attributes, or configuration passed through variables or **kwargs. Other languages and unparseable Python are outside these rules' coverage.

Queue/Lambda timeout correlation supports Python CDK Queue and Function constructs and explicit event-source links in the same file. Numeric comparisons require literal Duration.seconds, minutes, or hours values. Missing defaults, dynamic expressions, conditional resource definitions, and deployed settings are not inferred. Unbound configurations are review candidates, not evidence that the resources communicate.

DynamoDB detection uses text-pattern heuristics, without resolving call receivers or proving that matched controls apply to a write. A missing finding does not establish workload safety.

Processing paths require an explicit Python Lambda runtime and resolve literal CDK Code.from_asset and handler settings against eligible files inside the scan root. Only direct boto3 DynamoDB PutItem calls in the configured handler are linked; helper calls, dynamic assets, dependency injection, and deployed behavior remain unresolved. Omitted queue FIFO settings and Lambda timeouts are not inferred.

## Candidate risk chain

Evidence confidence: **Low**.

SQS usage and an unlinked DynamoDB write occur in this repository. A shared processing path has not been established for that write; these resources may be unrelated. If the consumer invokes it, validate whether replay creates additional items or replaces state when keys are reused. AWS-SQS-001 and AWS-DDB-001 contribute to this candidate review question, not two independently confirmed failures.

## AWS-DDB-001 — DynamoDB PutItem replay and overwrite behavior requires validation

| Field | Value |
|---|---|
| Category | Data |
| Severity | Medium |
| Confidence | Low |
| Detection | Text-pattern heuristic |
| Rule definition | `invariant.rules.dynamodb.AWS_DDB_001` |

### Summary

A DynamoDB-style PutItem call was detected. If events can be retried, duplicated, or arrive out of order, validate replay behavior and whether a write using an existing primary key may replace the stored item. Calls that generate new keys instead require review for unintended additional records. This is a migration review question; intended ordering and idempotency design are not established. No conditional-write or optimistic-concurrency pattern was detected in scanned text; this is a review question, not proof that controls do not exist elsewhere.

### Observed behavior

The scanner matched DynamoDB-style write calls in source text. No conditional-write or version-related text was matched; controls may exist outside these patterns.

### Evidence

| Location | Type | Matched text |
|---|---|---|
| [`services/UserApplication-ProcessSQSandS3Messages/lambda_function.py:63`](https://github.com/aws-samples/multiregion-s3-sns-sqs-lambda/blob/e596ad0f05ec19891c2bd12a1cf0d44edbc2786f/services/UserApplication-ProcessSQSandS3Messages/lambda_function.py#L63) | ddb_write_put | `data = dynamodb.put_item(` |

### Source behavior

PutItem creates an item or replaces the item with the same primary key; conditional expressions can restrict the write. Different keys can create additional items. A call alone does not establish the business rule.

### Potential invariant to validate

A duplicate or older logical event must not replace newer intended state unless last-writer-wins is the explicit business rule. Reprocessing with a new key should not create an unintended additional record.

### Azure target question

If Azure Cosmos DB is selected as the target datastore, how will its design preserve correctness for concurrent, duplicate, retried, or reordered writes?

### Why it matters

Data persistence does not automatically preserve a workload's source conflict model.

### Recommendation

1. Establish whether item keys represent a logical event, an entity, or a new record per processing pass.
2. Write business invariants explicitly.
3. Classify updates as additive, absolute, or versioned.
4. Choose optimistic concurrency, transaction, or serialization strategy.
5. Align partition design with transaction boundaries.

### Validation scenario

1. Send overlapping updates for the same entity.
2. Inject retries and duplicate delivery.
3. Replay an older update after a newer update.
4. Verify invariants.

### Limitations

Matched writes do not establish the full data model, conflict rules, or transaction boundaries.

## AWS-SQS-001 — Review SQS consumers for duplicate-delivery safety

| Field | Value |
|---|---|
| Category | Messaging |
| Severity | Medium |
| Confidence | Low |
| Detection | Python AST |
| Rule definition | `invariant.rules.sqs.AWS_SQS_001` |

### Summary

SQS usage was detected. Because SQS can redeliver messages, review consumers for idempotency when processing can produce persistent state changes or external side effects.

### Observed behavior

The scanner detected SQS usage or configuration; this does not establish that a consumer or business side effect exists in the scanned code. Idempotency mechanisms and business effects are not analyzed by this rule.

### Evidence

| Location | Type | Matched text |
|---|---|---|
| [`services/UserApplication-ProcessSQSandS3Messages/lambda_function.py:39`](https://github.com/aws-samples/multiregion-s3-sns-sqs-lambda/blob/e596ad0f05ec19891c2bd12a1cf0d44edbc2786f/services/UserApplication-ProcessSQSandS3Messages/lambda_function.py#L39) | sqs_delete | `sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt_handle)` |
| [`services/UserApplication-ProcessSQSandS3Messages/lambda_function.py:70`](https://github.com/aws-samples/multiregion-s3-sns-sqs-lambda/blob/e596ad0f05ec19891c2bd12a1cf0d44edbc2786f/services/UserApplication-ProcessSQSandS3Messages/lambda_function.py#L70) | sqs_delete | `sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt_handle)` |

### Source behavior

SQS standard queues use at-least-once delivery. Messages that are not deleted before visibility expires can become available again.

### Potential invariant to validate

Reprocessing the same logical message should not create an unintended duplicate business effect.

### Azure target question

For Azure Service Bus, how will the consumer identify and safely handle a message redelivered after a partial failure?

### Why it matters

At-least-once delivery can repeat work after transient failure and create duplicate writes or external side effects.

### Recommendation

1. Attach a stable event identifier to each logical work item.
2. Use durable idempotency or deduplication.
3. Document whether operations are additive, absolute-state-based, or versioned.

### Validation scenario

1. Persist a business-state change.
2. Force handler failure before message completion.
3. Allow re-delivery.
4. Verify duplicate processing does not create an unintended duplicate business effect.

### Limitations

This rule does not establish missing idempotency controls, ordering of business writes and message deletion, or runtime retry policy.


## Recommended next steps

1. Review every finding with the workload owner and migration lead.
2. Identify the business invariant associated with each applicable finding.
3. Convert relevant validation scenarios into pre-migration integration tests.
4. Record target-platform configuration decisions and accepted risks.
5. Re-run the scan after material migration-design changes.

## Privacy statement

This scan runs entirely locally and does not require network access.
Invariant does not upload source code,
findings, repository metadata, or scan results.

## License and disclaimer

This generated report is an engineering review aid. It is not legal, security,
compliance, or production-readiness advice.
