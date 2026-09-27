# AWS-to-Azure Migration Semantic-Risk Review

## Executive summary

Source repository: [aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/tree/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d)  
Reviewed commit: `7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d`

Scan date: 2026-09-27  
Offline mode: enabled  
Files scanned: 17  
Files skipped: 6  
Findings generated: 4

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
| AWS-SQS-002 | Medium | Medium | Python AST | Explicit SQS visibility timeout requires target lock-duration mapping |
| AWS-SQS-005 | Informational | Low | Python AST; binding unresolved | Queue and consumer timeout relationship requires operational validation |

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

A DynamoDB-style PutItem call was detected. If events can be retried, duplicated, or arrive out of order, validate replay behavior and whether a write using an existing primary key may replace the stored item. Calls that generate new keys instead require review for unintended additional records. This is a migration review question; intended ordering and idempotency design are not established.

### Observed behavior

The scanner matched DynamoDB-style write calls in source text. Conditional-write or version-related text was also matched.

### Evidence

| Location | Type | Matched text |
|---|---|---|
| [`error_handling/main.py:32`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/error_handling/main.py#L32) | ddb_write_put | `dynamodb.put_item(` |
| [`event_processing/main.py:72`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/event_processing/main.py#L72) | ddb_write_put | `dynamodb.put_item(` |

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

Matched writes do not establish the full data model, conflict rules, or transaction boundaries. A text-pattern match in `.projen/deps.json` was excluded because dependency-version metadata is not application evidence.

## AWS-SQS-001 — Review SQS consumers for duplicate-delivery safety

| Field | Value |
|---|---|
| Category | Messaging |
| Severity | Medium |
| Confidence | Low |
| Detection | Python AST |
| Rule definition | `invariant.rules.sqs.AWS_SQS_001` |

### Summary

SQS consumption patterns were detected. Because SQS can redeliver messages, review consumers for idempotency when processing can produce persistent state changes or external side effects.

### Observed behavior

The scanner detected SQS receive calls or SQS event-source configuration. Idempotency mechanisms and business effects are not analyzed by this rule.

### Evidence

| Location | Type | Matched text |
|---|---|---|
| [`event_processing/main.py:93`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/event_processing/main.py#L93) | sqs_receive | `response = sqs_client.receive_message(` |
| [`infrastructure/event_processing/main.py:312`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L312) | sqs_event_source | `SqsEventSource(self.__failed_jobs_dead_letter_queue))` |
| [`event_processing/main.py:104`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/event_processing/main.py#L104) | sqs_delete | `sqs_client.delete_message(` |

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
| [`infrastructure/event_processing/main.py:144`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L144) | visibility_timeout | `visibility_timeout=Duration.seconds(error_handling_timeout)` |
| [`infrastructure/event_processing/main.py:249`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L249) | visibility_timeout | `visibility_timeout=Duration.seconds(event_processing_timeout)` |

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

## AWS-SQS-005 — Queue and consumer timeout relationship requires operational validation

| Field | Value |
|---|---|
| Category | Messaging |
| Severity | Informational |
| Confidence | Low |
| Detection | Python AST; binding unresolved |
| Rule definition | `invariant.rules.sqs_lambda.AWS_SQS_005` |

### Summary

Queue visibility and Lambda timeout settings were found, but a direct queue-to-consumer binding could not be resolved for these resources. No timeout comparison or mismatch is asserted.

### Observed behavior

Found 2 queue(s) and 1 Lambda function(s) with explicit timeout settings and no resolved event-source link. These resources may be unrelated; retained evidence lists candidates, not pairs.

### Evidence

| Location | Type | Matched text |
|---|---|---|
| [`infrastructure/event_processing/main.py:139`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L139) | queue_definition | `Queue( self, "FailedJobsDeadLetterQueue", encryption=QueueEncryption.KMS, encryption_master_key=self.__jobs_queue_key, visibility_timeout=Duration.seconds(error_handling_timeout), )` |
| [`infrastructure/event_processing/main.py:144`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L144) | visibility_timeout | `visibility_timeout=Duration.seconds(error_handling_timeout)` |
| [`infrastructure/event_processing/main.py:238`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L238) | queue_definition | `Queue( self, "JobsQueue", encryption=QueueEncryption.KMS, encryption_master_key=self.__jobs_queue_key, queue_name="jobs_queue", dead_letter_queue=DeadLetterQueue( max_receive_count=1, queue=self.__failed_jobs_dead_letter_queue, ), retention_period=Duration.seconds(max_event_age), visibility_timeout=Duration.seconds(event_processing_timeout), )` |
| [`infrastructure/event_processing/main.py:249`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L249) | visibility_timeout | `visibility_timeout=Duration.seconds(event_processing_timeout)` |
| [`infrastructure/event_processing/main.py:198`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L198) | lambda_definition | `Function( self, "ErrorHandlingFunction", code=Code.from_asset( str( Path(__file__). parent. parent. parent. joinpath("error_handling"). resolve() ), bundling=BundlingOptions( command=[ "bash", "-c", ("cp /asset-input/main.py " "--target /asset-output " "--update"), ], image=Runtime.PYTHON_3_9.bundling_image, ), ), environment={ "TABLE_NAME": self.jobs_table.table_name, "QUEUE_NAME": self. __failed` |
| [`infrastructure/event_processing/main.py:236`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L236) | lambda_timeout | `timeout=Duration.seconds(error_handling_timeout)` |
| [`infrastructure/event_processing/main.py:227`](https://github.com/aws-samples/asynchronous-event-processing-api-gateway-sqs-cdk/blob/7c25cbd370a4b945d4479555fd83d2a3c1b9cb6d/infrastructure/event_processing/main.py#L227) | lambda_handler | `handler="main.handler"` |

### Source behavior

The queue's visibility period and its consumer's execution limit are separate settings.

### Potential invariant to validate

If these resources are connected, their processing and retry budgets should preserve business correctness.

### Azure target question

Which queue feeds which consumer, and what processing and retry budget must the Azure design preserve?

### Why it matters

Comparing unrelated resource timeouts can produce a false finding; a confirmed binding is needed first.

### Recommendation

1. Resolve event-source mappings from synthesized infrastructure or deployed configuration.
2. For each confirmed pair, compare queue visibility with Lambda timeout and the batching window.

### Validation scenario

1. Confirm whether the listed queue and Lambda resources are connected.
2. For confirmed pairs, measure processing and retry behavior under slow dependencies.
3. Verify duplicate processing does not create an unintended duplicate business effect.

### Limitations

The presence of both settings does not establish a connection or a timeout mismatch.


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
