# Operations guide

Day-to-day operation of [Automatic event hold release for Amazon S3 Object Lock](README.md): safety controls, the lifecycle rule you need to pair it with, scaling limits, the audit trail each run leaves, retention of the solution's own artifacts, and cleanup.

## Table of contents

- [Report-only and active modes](#report-only-and-active-modes)
- [Notification emails](#notification-emails)
- [Safety threshold](#safety-threshold)
- [Withheld: runtime SDK does not support EventHold](#withheld-runtime-sdk-does-not-support-eventhold)
- [Inventory freshness and the release decision](#inventory-freshness-and-the-release-decision)
- [Lifecycle integration](#lifecycle-integration)
- [Operating at scale](#operating-at-scale)
- [Audit trail](#audit-trail)
- [SolutionBucket data retention](#solutionbucket-data-retention)
- [SolutionBucket access logging](#solutionbucket-access-logging)
- [Alarm: job completion with no active-jobs record](#alarm-job-completion-with-no-active-jobs-record)
- [Alarm: a pipeline stage failed and was dead-lettered](#alarm-a-pipeline-stage-failed-and-was-dead-lettered)
- [Stack update and cleanup](#stack-update-and-cleanup)

## Report-only and active modes

**Report-only (`ReportOnly=true`, the default).** The pipeline runs end-to-end: inventory is evaluated, eligible versions are identified, and manifests are written to the SolutionBucket. Every approved mode's S3 Batch Operations job is created **suspended** (`ConfirmationRequired=true`). No `PutObjectRetention` call is made — the job never executes, so there is no completion report to read. Review what the run *would* do from the suspended job in the [S3 Batch Operations console](https://console.aws.amazon.com/s3/batch-jobs) plus its eligibility manifest in the SolutionBucket.

Use report-only mode to verify the solution identifies exactly the versions you expect before enabling active release.

**Active (`ReportOnly=false`).** Each approved mode's job is created non-suspended, so `PutObjectRetention` is called for every manifest row. Eligible holds are released automatically each run, subject to `SafetyThreshold`. The run's summary email arrives once the job(s) finish, not at dispatch time — see [Release mechanism](ARCHITECTURE.md#release-mechanism).

**Switching modes.** Change `ReportOnly` via a stack update (see [Deployment](README.md#deployment)). The change takes effect on the next pipeline run — no teardown or redeploy is required.

## Notification emails

Every run publishes exactly one message to the stack's SNS topic. The stack creates the topic and exports its ARN as the `SNSTopicArn` output, but creates no subscription — subscribe an address or endpoint to that topic yourself after deploying.

### Subject format

```
Auto Remove Event Hold: <outcome> - <Target_Bucket>
Auto Remove Event Hold: <outcome> [Report-Only] - <Target_Bucket>
```

The `[Report-Only]` tag appears only while `ReportOnly=true`. The outcome leads so a run is triageable from an inbox list without opening the mail, and the Target_Bucket name closes the line so runs from different stacks are distinguishable. The run ID is not in the subject — it is the first thing in the message body.

Runs that release nothing (report-only, withheld, or nothing eligible) are announced at dispatch time by the CreateJob Lambda:

| Subject | Meaning |
|---|---|
| `Auto Remove Event Hold: 2 jobs pending, 1,241 versions [Report-Only] - my-bucket` | Report-only run. Two suspended jobs are waiting in the Batch Operations console covering 1,241 eligible versions. Nothing was released. |
| `Auto Remove Event Hold: 1 job withheld, 1,234 versions - my-bucket` | One mode's release was declined and nothing was touched. Usually the count exceeded `SafetyThreshold` — see [Safety threshold](#safety-threshold). The other causes are unverifiable count evidence and a failed `create_job` call, per [Failure handling](ARCHITECTURE.md#failure-handling). The body names the specific reason. |
| `Auto Remove Event Hold: 1 job pending, 1 withheld, 10 versions - my-bucket` | Mixed: one mode suspended for review, another withheld. The version count covers both. |
| `Auto Remove Event Hold: nothing eligible - my-bucket` | The pipeline ran and found no version that qualified. A healthy result, not an error. |

Runs that release something are announced by the JobCompletion Lambda once every Batch Operations job the run created has reached a terminal state, so the counts are actual per-object outcomes rather than dispatch intent:

| Subject | Meaning |
|---|---|
| `Auto Remove Event Hold: 1,234 versions released - my-bucket` | Every job completed and every row succeeded. |
| `Auto Remove Event Hold: 1,230 of 1,234 versions released, 4 failed - my-bucket` | Jobs completed with per-row failures. The failing rows are in the completion report under `reports/<run-id>/<mode>/`. |
| `Auto Remove Event Hold: 1 version released, 1 job did not complete - my-bucket` | A job ended `Failed` or `Cancelled`. Check that job in the Batch Operations console. |

Counts are versions except where the subject says "job". A job covers one `ObjectLockMode`, so a run against a bucket holding both Compliance and Governance versions can report two jobs — see [Buckets with a mix of Compliance and Governance mode objects](ARCHITECTURE.md#buckets-with-a-mix-of-compliance-and-governance-mode-objects). The version count is omitted where the run never established one, which happens when a mode is withheld before its count was verified: `1 job withheld` with no number is that case.

Two further outcomes are rare enough that seeing either is worth investigating. `no modes evaluated` means CreateJob reached its decision with no mode in any category, and `finished, nothing released` means every job completed reporting zero objects processed. Both point at a pipeline problem rather than an empty bucket; read the body and the CreateJob or JobCompletion log group.

### Message body

The body is the run summary as JSON. Its first field, `summary`, is a plain-English recap that opens with the run ID, the Target_Bucket, the inventory delivery timestamp, and the mode, followed by a sentence per `ObjectLockMode`. The structured fields below it carry the same detail machine-readably, including per-mode eligible counts, job IDs, and withholding reasons. The identical object is written to `manifests/combined/<run-id>/_job_summary.json` in the SolutionBucket.

SNS caps a Subject at 100 ASCII characters. Long bucket names are elided in the middle to fit (`acme-prod-obje...est-1-primary`), and in the rare case where the outcome text alone fills the line, its trailing clauses are dropped with a `...`. Nothing the subject elides is missing from the body.

## Safety threshold

`SafetyThreshold` is the maximum number of event-hold releases a single active run may create a job for. StartQuery first executes a dedicated Athena `COUNT(*)` query built from the same candidate CTE as the manifest `UNLOAD`. CreateJob independently verifies that query and its result before deciding whether to create a non-suspended job.

When an active-mode count exceeds the threshold, **no S3 Batch Operations job is created for that mode**. The manifest remains available for review, while SNS and `manifests/combined/<run-id>/_job_summary.json` record the latest decision. Every decision is also preserved at `manifests/combined/<run-id>/_job_summaries/<decision-id>.json`, so later approval does not erase the original withheld record. Withholding also avoids the Batch Operations per-job and per-object-processed charge for a manifest that cannot yet be approved.

If the count is unavailable, stale, failed, or disagrees with the propagated value, that mode is also withheld without creating a job.

**Recommended starting value.** Start with `1000` (the default). After several report-only runs, tune it using observed eligible counts.

**Approving a withheld active-mode run.** A withheld run has not released anything — the versions it identified are still sitting there with an active event hold. Raise `SafetyThreshold` to an approved value via a normal stack update:

```bash
aws cloudformation deploy \
  --stack-name remove-event-hold-my-bucket \
  --template-file template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    TargetBucket=amzn-s3-demo-object-lock-bucket \
    ReleaseMode=either \
    ReportOnly=false \
    SafetyThreshold=5000
```

The next scheduled `DetectionSchedule` run re-evaluates those same versions from a fresh inventory delivery and, if the count now falls within the raised threshold, creates the Batch Operations job automatically — no manual Lambda invocation or synthetic event payload required. This is safe because of the [previous-manifest dedup](ARCHITECTURE.md#idempotency): the run that gets approved anti-joins against the most recently *released* manifest per mode, so it cannot double-submit versions another run already released in the meantime.

**Resolution latency.** Because approval happens through the normal schedule rather than an immediate manual trigger, resolution is bounded by `DetectionSchedule` plus inventory delivery lag (up to ~48 hours) — for `WEEKLY`, up to about 9 days worst case. If that doesn't work for your situation, a stack update can temporarily set `DetectionSchedule=DAILY` to shorten the wait; switch it back afterward if you don't want the ongoing `DAILY` cost and query volume.

## Withheld: runtime SDK does not support EventHold

Occasionally a run reports a mode as withheld with the reason `event_hold_unsupported_by_runtime`, and the summary names an SDK version:

```
GOVERNANCE: WITHHELD -- the AWS SDK bundled in this Lambda runtime
(botocore 1.42.97) does not accept the EventHold parameter, so no Batch
Operations job was created.
```

**This resolves itself and needs no action.** Lambda's managed runtimes supply their own copy of the AWS SDK for Python, and AWS updates it on its own schedule. Until the copy your function loaded includes the `EventHold` parameter, the release call cannot be built, so CreateJob withholds the mode rather than attempting it.

Expect the answer to differ between deployments, and to change over time. Lambda [publishes new runtime versions gradually across Regions and in two phases within a Region](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-update.html): a function picks up the newest version when it is created or updated, and functions left alone are swept up later. Two stacks in the same Region deployed a fortnight apart can therefore load different SDK versions, which is why this is reported per run rather than stated once as a version requirement.

Nothing was released and nothing was lost. The versions the run identified keep their event hold, stay eligible, and are released by the first run that lands after the runtime's SDK updates. Eligible counts are still verified and reported, so a withheld run reads like a [report-only](#report-only-and-active-modes) one: you learn exactly what is queued up, without a suspended job to inspect.

**Confirming which SDK a run used.** Every run records it, successful ones included, in SNS and `manifests/combined/<run-id>/_job_summary.json`:

| Field | Meaning |
|---|---|
| `event_hold_model: native` | The parameter is available. Releases proceed normally. |
| `event_hold_model: missing` | Not yet available to this function. Modes are withheld. |
| `event_hold_model: unknown` | Could not be determined; the run attempts the release anyway. |
| `botocore_version` | The SDK version the run actually loaded. |

The same detail is in the CreateJob log group as a `createjob_event_hold_capability` entry, one per run. This is the authoritative reading for your stack: it comes from the function that made the decision, so it is more reliable than probing a different function or Region.

**Refreshing the SDK sooner.** Because a function picks up the newest available runtime version whenever it is updated, you do not have to wait for Lambda to sweep your function up. Any stack update that changes CreateJob's configuration pulls it forward, and changing `SafetyThreshold` is the simplest one that already has a supported path — see [Approving a withheld active-mode run](#safety-threshold) for the command. The next run then reports the SDK version it loaded, so one run tells you whether it worked.

This gets you the newest version available in your Region at that moment, which may still predate `EventHold`. It costs nothing to try.

**Retrying the same run.** Waiting for the next `DetectionSchedule` run is the normal path and needs nothing from you. To re-run the versions a withheld run already identified, delete `manifests/combined/<run-id>/_createjob_lock.json` from the SolutionBucket, then re-upload that run's `_manifests_ready.json` object to re-trigger CreateJob. The lock is what makes repeated S3 event deliveries safe, so removing it is the one step that cannot be skipped.

## Inventory freshness and the release decision

Eligibility is decided entirely from one S3 Inventory snapshot, and **nothing re-verifies a version's state at release time**. Batch Operations executes the manifest exactly as CreateJob submitted it: no per-object check confirms the hold is still present, that the version is still noncurrent, or that its immediate superseder is still the one the query classified. The evidence behind any given release is therefore as old as `DetectionSchedule` plus inventory delivery lag — up to about 48 hours.

**What this means in practice.** Re-applying an event hold on a version does not take effect immediately against a run already working from an earlier snapshot. That run can still release the hold you just re-applied; the next run sees the new state and leaves the version alone. To stop a run that is already in flight, cancel its job from the [S3 Batch Operations console](https://console.aws.amazon.com/s3/batch-jobs) — the job is a standalone resource, so cancelling it stops the tasks it has not reached yet. Versions it already processed are not reverted.

**What a stale release cannot do.** It still computes `retain-until-date = MAX(existing, release time + EventHoldDuration)`, so it cannot shorten an existing retention and cannot delete a version or a delete marker. The failure mode is "released earlier than you now intend", not data loss: every affected version still carries at least `EventHoldDuration` of protection measured from the moment of release.

This is the single most load-bearing assumption in the design, which is why the reviewable checkpoints sit where they do — [report-only mode](#report-only-and-active-modes) and [`SafetyThreshold`](#safety-threshold) both interpose between the snapshot and the release, and neither depends on the snapshot being fresh to work.

## Lifecycle integration

Releasing an event hold does not delete versions — it sets a fixed retain-until-date. You need a `NoncurrentVersionExpiration` lifecycle rule on the Target_Bucket to delete versions after their retention elapses. **This solution does not modify the Target_Bucket's lifecycle configuration.**

How the two pieces compose:

1. This solution releases the event hold on noncurrent versions, setting `retain-until-date = MAX(existing, release time + EventHoldDuration)`.
2. Your `NoncurrentVersionExpiration` rule expires (permanently deletes) a noncurrent version once it has been noncurrent longer than `NoncurrentDays` **and** its retain-until-date has passed. Both conditions must be true — the rule does not delete a version on `NoncurrentDays` alone if the retain-until-date is still in the future. S3 enforces Object Lock retention regardless of lifecycle rules.

**Example: add a NoncurrentVersionExpiration rule via the AWS CLI.**

```bash
cat > tmp/lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "delete-noncurrent-after-retention",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      }
    }
  ]
}
EOF
```

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket amzn-s3-demo-object-lock-bucket \
  --lifecycle-configuration file://tmp/lifecycle.json
```

**Choosing `NoncurrentDays`.** Set it to at least your `EventHoldDuration`, and comfortably longer than your `DetectionSchedule` cadence plus inventory delivery lag. Two reasons:

1. A larger value costs nothing in practice for held versions — lifecycle cannot permanently delete a version before its retain-until-date regardless of `NoncurrentDays`.
2. A short `NoncurrentDays` can work against `delete` mode. **Delete markers never carry an event hold and have no retain-until-date, so the rule removes noncurrent delete markers on the `NoncurrentDays` schedule alone** — independent of any hold defaults or anything this solution does. If a delete marker is cleaned up before the pipeline has released the data version it superseded, that version's immediate superseder changes, which can reclassify a `delete`-mode candidate as overwrite-superseded and leave it un-released. (The same property is what makes a withheld tie involving a delete marker [self-healing](CLASSIFYING-VERSIONS.md#whether-a-withheld-candidate-stays-stuck).)

## Operating at scale

### Athena DML timeout

Athena's default DML query timeout is 30 minutes. For very large buckets — roughly more than **2 billion object versions** — the eligibility query may exceed this limit.

Mitigation options:

- **Raise the Athena DML timeout service quota** via the AWS Service Quotas console.
- **Shard by prefix** by deploying multiple independent stacks, each scoped to a non-overlapping `Prefix`. Each stack runs its own inventory, query, and Batch Operations job independently, with its own SolutionBucket, Glue table, and IAM roles. Multiple stacks in one account and Region are supported, subject to the stack-naming rules in [MULTI-ACCOUNT-DEPLOYMENT.md](MULTI-ACCOUNT-DEPLOYMENT.md#stack-naming-rules).
- **Open an issue** to discuss your requirements with the authors.

### Many stacks in one account and Region

Most of the stack is per-stack, but a few resources and quotas are shared across every stack in an account and Region — the S3 Tables integration for access logs, the Athena Active DML query quota, and the CloudWatch alarms free tier. See [What is shared and what repeats](MULTI-ACCOUNT-DEPLOYMENT.md#what-is-shared-and-what-repeats).

## Audit trail

Every run leaves a record of what it decided, what it submitted, and what actually happened. All of it lands in the SolutionBucket, never on the Target_Bucket, and none of it is written unless the stage that produces it succeeded.

| Artifact | Location | What it proves |
|---|---|---|
| Eligibility manifest | `manifests/combined/<run-id>/<mode>/manifest.csv` | Exactly which object versions were submitted for release, as `Bucket,Key,VersionId`. The job cannot touch a version this file does not name. |
| Decision record, latest | `manifests/combined/<run-id>/_job_summary.json` | The run's final decision per `ObjectLockMode`: eligible counts, job IDs, withholding reasons. Identical to the body of the SNS email. |
| Decision record, every decision | `manifests/combined/<run-id>/_job_summaries/<decision-id>.json` | Every decision the run reached, not just the last one — including a `SafetyThreshold` withholding followed by a later approval, which `_job_summary.json` alone would overwrite. |
| Completion report | `reports/<run-id>/<mode>/` | Per-version outcome, including any `PermanentFailure`. Written by S3 Batch Operations itself, independently of this pipeline, so it survives a pipeline fault that loses the summary. |
| Withheld-candidates diagnostic | `diagnostics/withheld-candidates/<dt>/manifest.csv` | Which candidates were deliberately *not* released, with a per-row reason. Queryable as the `withheld_candidates_history` Glue table. See [Classifying versions](CLASSIFYING-VERSIONS.md). |
| Inventory snapshot | `inventory/` | The evidence the decision was made from: each version's hold status, lock mode, and currency at snapshot time. |

Pipeline execution logs sit in the nine `<stack-name>-*` CloudWatch Logs groups with 90-day retention, carrying one structured event per stage per run rather than per object.

The manifest, the decision records, the completion report, and the withheld-candidates diagnostic are all retained for 90 days, so a run's inputs, decisions, and outcome expire together rather than at different times. The inventory snapshot behind them expires at 14 days, which is the one part of the chain that goes first. See [SolutionBucket data retention](#solutionbucket-data-retention).

**The durable per-version record is CloudTrail, and it is not on by default.** Everything above lives in the SolutionBucket under those lifecycle rules, so it expires. CloudTrail **data events** for `PutObjectRetention` on the Target_Bucket are the only per-version record of a release that outlives them and sits outside a bucket this stack can write to. This stack does not create a trail and does not enable data events; both are yours to configure, and S3 data events are billed per event, so it is a decision with a cost attached rather than a free default. Without them, a release older than the SolutionBucket's retention window is evidenced only by the object version's own `retain-until-date`.

**Two limits worth stating plainly.**

- **The audit trail records what the pipeline decided, not the state of the bucket at release time.** Nothing re-verifies a version between the inventory snapshot and the `PutObjectRetention` call, so every artifact above describes evidence that is up to about 48 hours old. See [Inventory freshness and the release decision](#inventory-freshness-and-the-release-decision).
- **A run that releases holds without telling you is possible, and alarmed.** If `_active_jobs.json` is missing when a job finishes, no summary is written and no email is sent, though the completion report is still produced. That path is covered by [the missing-record alarm](#alarm-job-completion-with-no-active-jobs-record). A run that fails before creating any job leaves no artifacts at all, and is covered by [the dead-letter queue alarm](#alarm-a-pipeline-stage-failed-and-was-dead-lettered).

## SolutionBucket data retention

The SolutionBucket applies lifecycle expiry rules automatically so working artifacts do not accumulate indefinitely.

| Prefix | Content | Expires after |
|---|---|---|
| `inventory/` | S3 Inventory reports | 14 days |
| `manifests/parts/` | Athena UNLOAD part files | 14 days |
| `manifests/combined/` | Combined eligibility manifests, plus each run's decision records (`_job_summary.json` and the `_job_summaries/` history) | 90 days |
| `athena-results/` | Athena query output | 14 days |
| `reports/` | Batch Operations completion reports | 90 days |
| `diagnostics/withheld-candidates/` | Per-run listings of candidates withheld due to ambiguous superseder classification (see [Classifying versions](CLASSIFYING-VERSIONS.md)) | 90 days |
| `manifests/previous/` | The most recently *released* manifest per `ObjectLockMode` (see [Idempotency](ARCHITECTURE.md#idempotency)) | No expiry — overwritten each time a non-suspended job is created |

Incomplete multipart uploads are aborted after 3 days.

The 14-day window covers one full weekly inventory cycle with buffer, and applies to the raw inventory snapshot and the intermediate query output that the combined manifest supersedes.

The three prefixes that make up the [audit trail](#audit-trail) are retained for 90 days instead: the combined manifests and decision records, the completion reports, and the withheld-candidates diagnostic. They are kept on the same window deliberately, so that a run's inputs, its decisions, and its outcome all remain available for the same period. Retaining the outcome longer than the decision that produced it would leave a window where a release could be confirmed but not explained.

If your compliance policy requires longer than 90 days, copy those three prefixes to a separate archive, and consider CloudTrail data events for a per-version record that does not depend on this bucket at all. `manifests/previous/` holds at most one small file per `ObjectLockMode`, always overwritten in place — it is a live dedup pointer, not an accumulating artifact, so no expiry rule applies to it.

## SolutionBucket access logging

S3 server access logs for the SolutionBucket are delivered to **CloudWatch Logs**, not to a destination S3 bucket, using [CloudWatch Logs vended log delivery](https://docs.aws.amazon.com/AmazonS3/latest/userguide/sal-cw-enabling.html). This requires no destination-bucket setup or bucket policy on the customer's side, supports native KMS encryption on the log group (the S3-bucket-destination path only supports SSE-S3), and is queryable with CloudWatch Logs Insights.

**Off by default — opt in.** `EnableServerAccessLogs` defaults to `false`. Set it to `true` to deliver the SolutionBucket's access logs to CloudWatch Logs.

The default is off because the SolutionBucket holds only this solution's own working artifacts — inventory snapshots, manifests, completion reports — and never customer object data. Access logging on it is an operational aid for debugging the pipeline, not a protection for protected content, so it is not worth the CloudWatch Logs ingestion and storage charges by default. Turn it on when you are troubleshooting, or when your own logging policy requires it. Logging on the **TargetBucket**, where the protected objects actually live, is outside this stack's scope and unaffected by this parameter.

**Destination.** By default (`ServerAccessLogsDestinationLogGroupArn` left blank), the stack creates a dedicated log group with `ServerAccessLogsRetentionDays` retention (default `30` days) and, if `KMSKeyArn` is set, encrypts it with that key. To send logs to a log group you already manage instead, supply its ARN via `ServerAccessLogsDestinationLogGroupArn`; in that case `ServerAccessLogsRetentionDays` is ignored and the existing log group's own retention and encryption settings are used.

**S3 Tables integration (on, but gated).** `EnableS3TablesIntegration` defaults to `true`, but it does nothing on its own: because `EnableServerAccessLogs` defaults to `false`, a default deployment creates no S3 Tables resources at all. It takes effect only once access logging is turned on — at which point you get the mirroring without a second parameter to set. When both are `true`, access logs are additionally mirrored to the account-and-Region-wide `aws-cloudwatch` S3 Tables managed table bucket in Apache Iceberg format, so they can be queried with standard SQL via Athena, Redshift, or any other Iceberg-compatible engine — see [Query Amazon S3 access logs instantly with CloudWatch and S3 Tables](https://aws.amazon.com/blogs/storage/query-amazon-s3-access-logs-instantly-with-cloudwatch-and-s3-tables/). There is no additional charge for storage or table maintenance in the S3 Tables integration; you pay only for CloudWatch Logs ingestion and S3 Tables query pricing.

**Account-and-Region-wide resource, shared across stacks.** The S3 Tables integration and the `amazon_s3`/`server_access` data source association it enables are both single, account-and-Region-wide settings — not scoped to this stack. The stack creates the integration only if none already exists in the account/Region, and reuses it otherwise. Deleting this stack does **not** disassociate the data source or delete the integration, since other buckets or stacks in the account may depend on the same setting. See [PERMISSIONS.md](PERMISSIONS.md#s3-tables-integration-for-server-access-logs) for manual cleanup instructions and the additional IAM permissions this feature requires.

## Alarm: job completion with no active-jobs record

The stack creates two CloudWatch alarms. This one, `<stack-name>-jobcompletion-missing-record`, covers the single path where a run can release event holds and tell nobody. The other, [`<stack-name>-pipeline-dlq-not-empty`](#alarm-a-pipeline-stage-failed-and-was-dead-lettered), covers a run that failed outright. Between them: one for a run that finished and lost its notification, one for a run that never finished.

**What it means.** JobCompletion reads `manifests/combined/<run-id>/_active_jobs.json` to learn which jobs a run created, so it knows when the last of them has finished and the summary email is due. If that object is missing when a job status-change event arrives, the function logs `jobcompletion_no_active_jobs_record` and stops. By that point the S3 Batch Operations job has already run and already released holds — but no SNS notification is sent and no `_job_summary.json` is written. Everything else in this solution announces itself; this is the one branch that releases holds and does not, which is why it is alarmed.

The alarm fires on the first occurrence (any data point above zero in a five-minute period) and notifies the same SNS topic as run summaries, so it reaches whoever is already subscribed. It does not notify on recovery: the underlying metric only ever publishes a breach, so an OK transition would carry no information.

**Two ways to reach it.** A genuine bug that leaves a run's record unwritten, or someone deleting the record between CreateJob writing it and JobCompletion reading it. The SolutionBucket policy denies unauthorized `PutObject` on the pipeline prefixes but deliberately does not deny `DeleteObject`, because a static policy cannot distinguish a malicious delete from the manual cleanup documented under [Stack deletion](#stack-deletion). Restricting `s3:DeleteObject` on the SolutionBucket in your own account is the control that closes that door; this alarm is what tells you if it was not closed.

**What to do when it fires.**

1. Find the `jobcompletion_no_active_jobs_record` entries in the `<stack-name>-jobcompletion` log group. Each carries the affected `run_id` and `job_id`.
2. Reconstruct what the job actually did from its completion report under the SolutionBucket's `reports/` prefix, which is written by Batch Operations independently of this pipeline and is unaffected by the missing record.
3. Confirm the releases against CloudTrail `PutObjectRetention` data events on the TargetBucket, if you have them enabled — the durable per-version record, and the reason to consider enabling them. See [Audit trail](#audit-trail).
4. Check whether `_active_jobs.json` was deleted rather than never written. CloudTrail management events cover the `DeleteObject` call on the SolutionBucket if data events are enabled for it.

**Nothing was released that the manifest did not already name.** The missing record breaks notification and summary writing, not release targeting: the job ran against the manifest CreateJob approved, under the same `SafetyThreshold` and count verification as any other run. The gap is that you were not told, not that the wrong objects were touched.

**Cost.** $0.10 per month, or free within the account's 10-alarm free tier, and the metric behind it costs nothing in a month where the condition never occurs. See [Cost detail](COST.md) — alarms are the one line that scales with stack count rather than release volume, which matters if you run many stacks.

## Alarm: a pipeline stage failed and was dead-lettered

The second alarm, `<stack-name>-pipeline-dlq-not-empty`, tells you a run stopped rather than finishing, so you find out the same week instead of on the next audit. It watches `ApproximateNumberOfMessagesVisible` on the stack's pipeline dead-letter queue and notifies the same SNS topic when the depth goes above zero.

**What it means.** StartQuery, ManifestMaker, CreateJob and JobCompletion are invoked asynchronously, and each deliberately raises on the failures that must not be mistaken for a run with nothing to do: an inventory delivery whose schema no longer carries a column the eligibility query reads, any Athena failure, cancellation or timeout, an unexpected S3 error. Raising is the correct behaviour — the run stops before writing results, so a partial or failed query set can never surface as a successful zero-eligible run. Lambda retries, and if every attempt fails the invocation record lands in the queue.

**Nothing was released.** These failures happen before any Batch Operations job is created, so the run produced no eligibility results and no summary. Without the alarm a stack whose pipeline had stopped working would look identical to one with nothing eligible, and on the `WEEKLY` default that could hold for weeks.

**What to do.**

1. Read the message in the queue (`<stack-name>-pipeline-dlq`) for the failing function name and its request ID. Receiving a message does not delete it, so you can inspect it and leave it in place.
2. Read that function's log group for the exception. The two most likely causes are an S3 Inventory delivery whose schema no longer matches the Glue table, and an Athena failure or timeout.
3. Fix the cause, then wait for the next `DetectionSchedule` run. Nothing needs replaying by hand: the failed run wrote no state, so the next inventory delivery re-evaluates the same versions from scratch.
4. Purge the queue once you are done. The alarm stays in `ALARM` while messages remain, so a dead-letter message keeps announcing itself for its full 14-day life. You get an OK notification when the queue empties.

**Cost.** $0.10 per month, or free within the account's 10-alarm free tier. Unlike the alarm above, this one watches a metric SQS publishes itself, so there is no metric filter and no custom metric charge either way.

## Stack update and cleanup

### Allowed parameter changes on update

These parameters can be changed via a stack update and take effect on the next pipeline run: `Prefix`, `ReleaseMode`, `DetectionSchedule`, `ReportOnly`, `SafetyThreshold`.

**Migration note (removed parameter):** `SettlingWindowDays` no longer exists. An existing stack updating to this template must drop it from `--parameter-overrides`; passing `SettlingWindowDays=...` (or `UsePreviousValue=true` for it) fails with an unknown-parameter error.

### Disallowed: changing TargetBucket

Changing `TargetBucket` on a stack update is rejected by the stack's prerequisite check. The inventory configuration, Glue table, IAM scoping, and Batch Operations role are all bound to the original bucket at deploy time. To target a different bucket, deploy a new stack — see [MULTI-ACCOUNT-DEPLOYMENT.md](MULTI-ACCOUNT-DEPLOYMENT.md).

### Stack deletion

The stack explicitly retains the SolutionBucket and its contents through `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain`. Deleting the stack removes the inventory configuration, Glue and Athena resources, Lambda functions, IAM roles, SNS topic, and other CloudFormation-managed resources, but leaves the SolutionBucket available for audit review and manual cleanup.

**Check for suspended or in-progress S3 Batch Operations jobs before deleting the stack.** A BOPS job is a standalone resource with its own lifecycle, independent of the CloudFormation stack. Deleting the stack deletes `BatchOperationsRole`, which a still-running or suspended job depends on to write its completion report — a job left running or suspended past stack deletion can fail partway through, or (for a suspended job) simply never be confirmable again. Check the [S3 Batch Operations console](https://console.aws.amazon.com/s3/batch-jobs) for any `Suspended`, `Ready`, or `Active` jobs tagged `job-created-by: Auto Remove Event Hold Solution:<stack-name>` and confirm or cancel them first.

Stack deletion does not change the Object Lock state or retain-until-date of any version already processed. Versions that had their event hold released retain their computed `retain-until-date`.

**Manual SolutionBucket cleanup.** The retained bucket is unversioned, so after preserving required reports, `aws s3 rm --recursive` (or the S3 console's **Empty** action) is sufficient to empty it before deletion — there's no version history or delete markers to account for.

After the bucket is empty:

```bash
aws s3api delete-bucket --bucket <SolutionBucketName> --no-cli-pager
```

Replace `<SolutionBucketName>` with the `SolutionBucketName` stack output or the bucket name shown in the S3 console. The bucket name follows the account-regional namespace pattern:

```
<lowercase-stack-name>-<AccountId>-<Region>-an
```

If the stack name exceeds 32 characters, the prefix is truncated at 32 characters. For example, a stack named `remove-event-hold-my-bucket` in account `123456789012` in `us-east-1` would have the bucket name `remove-event-hold-my-bucket-123456789012-us-east-1-an`.
