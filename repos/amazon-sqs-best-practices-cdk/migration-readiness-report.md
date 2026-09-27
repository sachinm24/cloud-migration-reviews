# AWS-to-Azure Migration Semantic-Risk Review

## Executive summary

Source repository: [aws-samples/amazon-sqs-best-practices-cdk](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/tree/54143c7efd610b2345964951403bbde628f54ba0)  
Reviewed commit: `54143c7efd610b2345964951403bbde628f54ba0`

Scan date: 2026-09-27  
Offline mode: enabled  
Files scanned: 5  
Files skipped: 1  
Findings generated: 3

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
| AWS-SQS-001 | Medium | Medium | Python AST | Review SQS consumers for duplicate-delivery safety |
| AWS-SQS-002 | Medium | Medium | Python AST | Explicit SQS visibility timeout requires target lock-duration mapping |

## Assessment boundaries

The scanner analyzes eligible local text files only. It excludes common secret files, credential directories, generated artifacts, VCS metadata, and files containing likely credential material.

Static analysis cannot determine production traffic, deployment topology, runtime configuration, external-system behavior, business invariants, or operational procedures unless represented in scanned files.

SQS detection uses Python syntax trees (ASTs) to recognize boto3 and AWS CDK calls with locally resolved imports and bindings. It does not resolve cross-file aliases, runtime factories, control flow, instance attributes, or configuration passed through variables or **kwargs. Other languages and unparseable Python are outside these rules' coverage.

Queue/Lambda timeout correlation supports Python CDK Queue and Function constructs and explicit event-source links in the same file. Numeric comparisons require literal Duration.seconds, minutes, or hours values. Missing defaults, dynamic expressions, conditional resource definitions, and deployed settings are not inferred. Unbound configurations are review candidates, not evidence that the resources communicate.

DynamoDB detection uses text-pattern heuristics, without resolving call receivers or proving that matched controls apply to a write. A missing finding does not establish workload safety.

Processing paths require an explicit Python Lambda runtime and resolve literal CDK Code.from_asset and handler settings against eligible files inside the scan root. Only direct boto3 DynamoDB PutItem calls in the configured handler are linked; helper calls, dynamic assets, dependency injection, and deployed behavior remain unresolved. Omitted queue FIFO settings and Lambda timeouts are not inferred.

## Resolved processing path

These are source relationships, not observed executions. AWS-SQS-001 and AWS-DDB-001 contribute to the same replay review; they are not counted as independent correctness failures.

`InventoryUpdatesQueue (sqs_blog/sqs_blog_stack.py:50)` → `SQSToDynamoDBFunction (sqs_blog/sqs_blog_stack.py:145)` → `sqs_blog/lambda/SQSToDynamoDBFunction.py:lambda_handler` → **DynamoDB PutItem**

Evidence confidence: **Medium**. Contributing rules: AWS-SQS-001, AWS-DDB-001.

Queue visibility: 300 seconds; Lambda timeout: absent. Unknown values are not replaced by service defaults.

Validate whether replay creates another item, or replaces intended state when keys are reused. The source path alone does not establish either effect.

| Location | Type | Matched text |
|---|---|---|
| [`sqs_blog/lambda/SQSToDynamoDBFunction.py:6`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/lambda/SQSToDynamoDBFunction.py#L6) | lambda_handler_definition | `def lambda_handler(event, context):` |
| [`sqs_blog/lambda/SQSToDynamoDBFunction.py:34`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/lambda/SQSToDynamoDBFunction.py#L34) | ddb_put_item | `dynamodb_client.put_item(TableName=table_name, Item=item)` |
| [`sqs_blog/sqs_blog_stack.py:50`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L50) | queue_definition | `sqs.Queue( self, "InventoryUpdatesQueue", visibility_timeout=Duration.seconds(300), #encryption=sqs.QueueEncryption.KMS_MANAGED, dead_letter_queue=sqs.DeadLetterQueue( max_receive_count=5, # Number of retries before sending the message to the DLQ queue=dlq ) )` |
| [`sqs_blog/sqs_blog_stack.py:52`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L52) | visibility_timeout | `visibility_timeout=Duration.seconds(300)` |
| [`sqs_blog/sqs_blog_stack.py:145`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L145) | lambda_definition | `_lambda.Function( self, "SQSToDynamoDBFunction", runtime=_lambda.Runtime.PYTHON_3_8, code=_lambda.Code.from_asset('sqs_blog/lambda'), handler='SQSToDynamoDBFunction.lambda_handler', role=role, tracing=Tracing.ACTIVE # Enable active tracing with X-Ray )` |
| [`sqs_blog/sqs_blog_stack.py:148`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L148) | lambda_code_asset | `code=_lambda.Code.from_asset('sqs_blog/lambda')` |
| [`sqs_blog/sqs_blog_stack.py:149`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L149) | lambda_handler | `handler='SQSToDynamoDBFunction.lambda_handler'` |
| [`sqs_blog/sqs_blog_stack.py:162`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L162) | sqs_lambda_binding | `sqs_to_dynamodb_function .add_event_source_mapping( "MyQueueTrigger", event_source_arn=queue.queue_arn, batch_size=10 )` |

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
| [`sqs_blog/lambda/SQSToDynamoDBFunction.py:34`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/lambda/SQSToDynamoDBFunction.py#L34) | ddb_write_put | `dynamodb_client.put_item(TableName=table_name, Item=item)` |

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
| Confidence | Medium |
| Detection | Python AST |
| Rule definition | `invariant.rules.sqs.AWS_SQS_001` |

### Summary

A resolved SQS-triggered Lambda handler contains a direct DynamoDB PutItem call. Validate that duplicate, retried, or reordered records do not create unintended items or replace intended state when keys are reused. The source path does not establish actual execution or an unsafe business effect.

### Observed behavior

Resolved source paths: InventoryUpdatesQueue (sqs_blog/sqs_blog_stack.py:50) → SQSToDynamoDBFunction (sqs_blog/sqs_blog_stack.py:145) → sqs_blog/lambda/SQSToDynamoDBFunction.py:lambda_handler → DynamoDB PutItem. Item identity, external controls, and business semantics remain unverified.

### Evidence

| Location | Type | Matched text |
|---|---|---|
| [`sqs_blog/lambda/SQSToDynamoDBFunction.py:6`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/lambda/SQSToDynamoDBFunction.py#L6) | lambda_handler_definition | `def lambda_handler(event, context):` |
| [`sqs_blog/lambda/SQSToDynamoDBFunction.py:34`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/lambda/SQSToDynamoDBFunction.py#L34) | ddb_put_item | `dynamodb_client.put_item(TableName=table_name, Item=item)` |
| [`sqs_blog/sqs_blog_stack.py:50`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L50) | queue_definition | `sqs.Queue( self, "InventoryUpdatesQueue", visibility_timeout=Duration.seconds(300), #encryption=sqs.QueueEncryption.KMS_MANAGED, dead_letter_queue=sqs.DeadLetterQueue( max_receive_count=5, # Number of retries before sending the message to the DLQ queue=dlq ) )` |
| [`sqs_blog/sqs_blog_stack.py:52`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L52) | visibility_timeout | `visibility_timeout=Duration.seconds(300)` |
| [`sqs_blog/sqs_blog_stack.py:145`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L145) | lambda_definition | `_lambda.Function( self, "SQSToDynamoDBFunction", runtime=_lambda.Runtime.PYTHON_3_8, code=_lambda.Code.from_asset('sqs_blog/lambda'), handler='SQSToDynamoDBFunction.lambda_handler', role=role, tracing=Tracing.ACTIVE # Enable active tracing with X-Ray )` |
| [`sqs_blog/sqs_blog_stack.py:148`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L148) | lambda_code_asset | `code=_lambda.Code.from_asset('sqs_blog/lambda')` |
| [`sqs_blog/sqs_blog_stack.py:149`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L149) | lambda_handler | `handler='SQSToDynamoDBFunction.lambda_handler'` |
| [`sqs_blog/sqs_blog_stack.py:162`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L162) | sqs_lambda_binding | `sqs_to_dynamodb_function .add_event_source_mapping( "MyQueueTrigger", event_source_arn=queue.queue_arn, batch_size=10 )` |

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

## AWS-SQS-002 — Explicit SQS visibility timeout requires target lock-duration mapping

| Field | Value |
|---|---|
| Category | Messaging |
| Severity | Medium |
| Confidence | Medium |
| Detection | Python AST |
| Rule definition | `invariant.rules.sqs.AWS_SQS_002` |

### Summary

Explicit SQS visibility-timeout configuration was detected. If Azure Service Bus is selected, map this timing behavior to lock duration, lock renewal, retry policy, and the target consumer's processing budget. The setting alone does not establish sufficient or insufficient time.

### Observed behavior

The scanner found explicit visibility-timeout arguments or literal attributes in recognized SQS calls. The evidence below preserves the configured expressions; runtime values are not evaluated.

### Evidence

| Location | Type | Matched text |
|---|---|---|
| [`sqs_blog/sqs_blog_stack.py:44`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L44) | visibility_timeout | `visibility_timeout=Duration.seconds(300)` |
| [`sqs_blog/sqs_blog_stack.py:52`](https://github.com/aws-samples/amazon-sqs-best-practices-cdk/blob/54143c7efd610b2345964951403bbde628f54ba0/sqs_blog/sqs_blog_stack.py#L52) | visibility_timeout | `visibility_timeout=Duration.seconds(300)` |

### Source behavior

After receipt, SQS hides a message for the visibility period. If it is not deleted before that period expires, it can be received again. Visibility does not guarantee that duplicates cannot occur during the period. A timeout of zero makes the message immediately visible.

### Potential invariant to validate

If the workload requires exclusive processing, a message should not be processed concurrently while its original consumer is still legitimately processing it. Any overlap should preserve business correctness.

### Azure target question

Will Azure Service Bus lock duration and renewal cover normal and degraded processing?

### Why it matters

Lock expiry can make a message available again and cause concurrent or duplicate processing.

### Recommendation

1. Measure normal and degraded processing duration.
2. Include retries and dependency latency in timing budgets.
3. Configure and test lock duration and renewal.

### Validation scenario

1. Delay a handler past the target lock duration.
2. Observe whether the message becomes available again.
3. Verify concurrent handling preserves business invariants.

### Limitations

An explicit timeout does not establish that its duration is sufficient or that exclusive processing is required.


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
