# Cost detail

Per-service cost drivers and a worked example for [Automatic event hold release for Amazon S3 Object Lock](README.md). See [Cost](README.md#cost) in `README.md` for the summary.

You are responsible for the cost of the AWS services this stack creates. Because the pipeline runs periodically (daily or weekly, per `DetectionSchedule`) rather than continuously, cost scales with dataset size and eligible-version count per run, not with live request volume on the Target_Bucket.

The largest single cost driver at scale is the per-release cost — the S3 PUT request for each `PutObjectRetention` call plus the S3 Batch Operations per-object-processed charge. Both scale with the number of eligible versions released, not the bucket's total object count. At low release volumes the fixed $0.25 per-job Batch Operations charge dominates instead; the two examples below bracket these shapes.

## Cost drivers by service

| AWS service | Cost driver |
|---|---|
| [Amazon S3 Inventory](https://aws.amazon.com/s3/pricing/) | Per-object charge for each version-level inventory report, billed on the Target_Bucket's object/version count, at the configured `DetectionSchedule` cadence. |
| [Amazon Athena](https://aws.amazon.com/athena/pricing/) | Data scanned per `COUNT(*)`/`UNLOAD` query pair (one per `ObjectLockMode` per run) over the Parquet inventory, plus — for `delete`/`overwrite` modes only — one withheld-candidates diagnostic query per run covering both lock modes (see [Classifying versions](CLASSIFYING-VERSIONS.md)); `either` mode never withholds anything and skips the diagnostic entirely. Partition projection avoids crawler cost; Parquet keeps scan volume down relative to CSV. The `previous_manifest` anti-join scans a comparatively tiny table (at most one small manifest per `ObjectLockMode`) and adds negligible cost. |
| [AWS Lambda](https://aws.amazon.com/lambda/pricing/) | Compute time and invocation count for PrereqCheck (once, at deploy), StartQuery, ManifestMaker, CreateJob (each once per run, plus per-mode), and JobCompletion (once per job status-change event for an active-mode run's job — see [Release mechanism](ARCHITECTURE.md#release-mechanism)). |
| [S3 Batch Operations](https://aws.amazon.com/s3/pricing/) | Per-job and per-object-processed charge for each `S3PutObjectRetention` job CreateJob creates (one per approved `ObjectLockMode` per run). |
| [S3 PUT requests](https://aws.amazon.com/s3/pricing/) | One `PutObjectRetention` PUT request per event-hold release, charged even when the S3 Batch Operations job issues it, priced as S3 Standard regardless of the object's storage class, scaled by the number of eligible versions. Never triggers a retrieval charge from an archive storage class, since it is a metadata-only operation. |
| [Amazon EventBridge](https://aws.amazon.com/eventbridge/pricing/) | Per-event charge for matched `JobStatusChanged` events driving the deferred completion email. These events are delivered via the account's existing CloudTrail management-event stream at no extra CloudTrail cost — no trail is created by this stack. Negligible at typical run frequency: one event per job state transition. |
| [Amazon SNS](https://aws.amazon.com/sns/pricing/) | Per-notification charge for run summaries, SafetyThreshold confirmation prompts, and failure notifications. Negligible at typical run frequency. |
| [S3 storage](https://aws.amazon.com/s3/pricing/) | The SolutionBucket's own working artifacts, governed by the lifecycle rules in [SolutionBucket data retention](OPERATIONS.md#solutionbucket-data-retention) so they don't accumulate indefinitely. The retained inventory deliveries dominate this, and they scale with the Target_Bucket's total object version count rather than with release volume. The [audit trail](OPERATIONS.md#audit-trail) prefixes are kept for 90 days rather than 14, but they hold per-run manifests and reports sized by eligible versions, not by bucket size. |
| [AWS KMS](https://aws.amazon.com/kms/pricing/) (optional) | Only if `KMSKeyArn` is set: monthly key charge plus per-request charges for `GenerateDataKey`/`Decrypt` calls against the SolutionBucket, SNS, SQS dead-letter queue, every Lambda log group, and (if applicable) the server access logs log group. |
| [Amazon CloudWatch Logs](https://aws.amazon.com/cloudwatch/pricing/) | Two components, and only the second is optional. **Always:** ingestion and storage for the nine Lambda functions' execution logs, which every deployment writes to the log groups the stack creates with 90-day retention. Tiny in absolute terms — the pipeline logs a handful of structured events per run, not per object — but not zero and not conditional. **Only if `EnableServerAccessLogs=true`, which is not the default:** ingestion and storage for the SolutionBucket's server access logs, which is the larger of the two when enabled. No additional charge for mirroring to S3 Tables. |
| [Amazon CloudWatch metrics and alarms](https://aws.amazon.com/cloudwatch/pricing/) | Two alarms and one metric filter per stack. The first alarm watches for a job completion that arrives with no `_active_jobs.json` record (see [Alarm: job completion with no active-jobs record](OPERATIONS.md#alarm-job-completion-with-no-active-jobs-record)); the second watches the pipeline dead-letter queue for a run that failed outright (see [Alarm: a pipeline stage failed and was dead-lettered](OPERATIONS.md#alarm-a-pipeline-stage-failed-and-was-dead-lettered)). The second needs no metric filter and no custom metric, because SQS publishes queue depth itself. The metric filter itself is free. Each alarm is charged per alarm metric per month whether or not it ever fires. The extracted metric is charged as a custom metric, but only in a month where it actually publishes a data point — the filter deliberately sets no `DefaultValue`, so a month in which the condition never occurs publishes nothing and incurs nothing. CloudWatch's always-free tier covers 10 alarm metrics and 10 custom metrics per account per month, so a single stack in an account not already at those limits pays nothing here. **This is a per-stack charge, so it scales with stack count, not with release volume** — see Example 2. |

## Example 1: one large bucket, high release volume

For a Target_Bucket with 100 million object versions, `DetectionSchedule=WEEKLY`, and roughly 1 million eligible noncurrent versions released per month under a single Object Lock retention mode (governance or compliance):

| AWS service | Rate | Per run | Monthly (×4.3 runs) |
|---|---|---|---|
| S3 Inventory | ~$0.0025 per million objects listed | $0.25 | $1.08 |
| Athena (count, `UNLOAD`, and diagnostic queries) | $5.00 per TB scanned (Parquet inventory scan is small relative to raw object count); includes the withheld-candidates diagnostic query in `delete`/`overwrite` modes | ~$0.05–$0.35 | ~$0.22–$1.50 |
| S3 Batch Operations (job + object processed) | $0.25 per job, $1.00 per million objects processed, one `S3PutObjectRetention` job per run | ~$0.48 | ~$2.06 |
| S3 PUT requests (`PutObjectRetention`) | $0.005 per 1,000 requests, one per release, issued by the S3 Batch Operations job and priced as S3 Standard regardless of the object's storage class | $1.16 | $5.00 |
| Lambda | Compute time + invocations for the per-run orchestration functions; no per-object invocation | <$0.02 | <$0.09 |
| SNS | Per-notification; typically one run-summary notification per run | <$0.01 | <$0.05 |
| CloudWatch Logs | Lambda execution logs always (standard class, $0.50/GB ingested — kilobytes per run, so effectively rounding error), plus the SolutionBucket's server access logs if `EnableServerAccessLogs=true` (vended, Infrequent Access class, $0.25/GB ingested + $0.03/GB/month stored; low hundreds of MB/month at this bucket size). The range below spans access logging off at the bottom and on at the top. | <$0.05 | ~$0.05–$0.22 |
| CloudWatch alarms | $0.10 per standard-resolution alarm metric per month for each of the two alarms the stack creates; $0 if the account is within the 10-alarm free tier. The metric filter is free, and its custom metric publishes — and costs — only in a month where that alarm condition actually occurs. | n/a (per month, not per run) | $0.00–$0.20 |
| S3 storage | Single-digit to low tens of GB retained under default lifecycle rules (Standard, $0.021–23/GB/month), almost entirely the two inventory deliveries the 14-day rule keeps. A version-level Parquet inventory runs roughly 40–120 bytes per object version depending on how well your key names dictionary-compress, so 100 million versions is about 4–12 GB per delivery. The 90-day audit prefixes (combined manifests and decision records, completion reports, withheld-candidates diagnostic) add well under a GB at this release volume. **This line scales with the bucket's total version count, not with how many versions you release** | n/a (driven by retention) | ~$0.45–$0.92 |
| **Total** | | **~$2.02–$2.32** | **~$9.00–$11.10** |

This is illustrative only. Athena cost depends on how much of the Parquet inventory the eligibility query has to scan. Buckets with a higher eligible-version count per run, or `DetectionSchedule=DAILY`, cost more than this example.

## Example 2: twenty small buckets, one batch each per month

`TargetBucket` is fixed at deploy time, so covering 20 buckets means 20 independent stacks, each with its own SolutionBucket, Glue database, Athena workgroup, Lambda functions, and log groups. This example assumes 1 million object versions per bucket, `DetectionSchedule=WEEKLY`, a single Object Lock retention mode, and 1,000 event-hold releases per bucket per month — few enough that eligible versions turn up in only one run a month, producing one S3 Batch Operations job per bucket per month.

| AWS service | Rate | Per bucket, monthly | 20 buckets, monthly |
|---|---|---|---|
| S3 Batch Operations (job + object processed) | $0.25 per job, $1.00 per million objects processed; one job per bucket per month | $0.25 | $5.00 |
| S3 Inventory | ~$0.0025 per million objects listed, on every run whether or not anything is eligible | $0.01 | $0.22 |
| S3 PUT requests (`PutObjectRetention`) | $0.005 per 1,000 requests, one per release | $0.005 | $0.10 |
| Athena (count, `UNLOAD`, and diagnostic queries) | $5.00 per TB scanned, with a per-query minimum; scans are small at this bucket size | <$0.01 | ~$0.06–$0.20 |
| Lambda, SNS, and EventBridge | Per-run orchestration only; no per-object invocation | <$0.02 | <$0.40 |
| CloudWatch Logs and S3 storage | Lambda execution logs plus the SolutionBucket's own working artifacts under default retention; server access logs only if `EnableServerAccessLogs=true`, which is not the default | ~$0.02 | ~$0.40 |
| CloudWatch alarms | $0.10 per alarm metric per month, two alarms per stack, so 20 stacks means 40 alarms. The first 10 fall in the account's free tier, which at two alarms per stack covers only the first five stacks; the range below spans 10 free at the bottom and none free at the top | $0.00–$0.20 | ~$3.00–$4.00 |
| **Total** | | **~$0.32–$0.52** | **~$9.20–$10.30** |

**The fixed per-job charge is the largest line at low release volume.** S3 Batch Operations' $0.25 per job is roughly half the total here, and it is the same $0.25 whether the job releases 10 versions or 10,000. Cost in this shape therefore tracks the number of buckets and jobs, not the number of versions released: 20 buckets releasing 1,000 versions each costs less in absolute terms than the single bucket in Example 1, but roughly 30 times more per release (~$0.0004 versus ~$0.00001).

**Alarms are the one line that scales purely with stack count.** At one stack they are free or $0.20; at 20 they are $3.00–$4.00 per month, roughly a third of this example's total and second only to the per-job charge. The account's 10-alarm free tier covers five stacks rather than ten, since each stack creates two. They buy the same thing in every stack: notification when a run releases holds without telling anyone, and when a run fails outright.

If you are running many stacks and would rather consolidate, both alarms can be replaced with one account-wide equivalent. Delete `JobCompletionMissingRecordAlarm` and build a single alarm across the stacks' log groups — the metric filters are free, so the per-stack metric can stay. Delete `PipelineDLQAlarm` and build one alarm on the `AWS/SQS` `ApproximateNumberOfMessagesVisible` metric across the stacks' dead-letter queues. Neither consolidation loses coverage; both lose the ability to tell you which stack is affected without looking.

Two sensitivities are worth checking against your own buckets:

- **Bucket size drives the inventory line, independent of release volume.** S3 Inventory bills per object listed on every run, so a bucket 100× larger than assumed here pays 100× this example's inventory cost even if still only 1,000 versions become eligible each month. Past roughly 23 million versions per bucket, inventory overtakes the per-job charge as the largest line.
- **`DetectionSchedule=DAILY` multiplies the per-run lines by about 7** (inventory, Athena, Lambda, SNS) without changing the Batch Operations or PUT charges, since the number of releases is unchanged.
