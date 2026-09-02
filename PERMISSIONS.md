# KMS Key Policy Prerequisites

When you supply a customer managed KMS key via the `KMSKeyArn` parameter, the key is used for the SolutionBucket's default encryption configuration (SSE-KMS, so objects written without their own encryption header are encrypted with this key), the SNS topic, the pipeline dead-letter SQS queue, **every Lambda function's CloudWatch Logs log group** (all nine — PrereqCheck, InventoryConfig, LowerCase, SetupNotifications, StartQuery, ManifestMaker, CreateJob, JobCompletion, and S3TableIntegrationSetup, the last of which exists only when `EnableServerAccessLogs=true`), and — when `EnableServerAccessLogs=true` and a new log group is created by the stack (i.e. `ServerAccessLogsDestinationLogGroupArn` is blank) — the server access logs CloudWatch Logs log group too. The key policy must grant the following permissions **before** deploying the stack.

Enable [automatic key rotation](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html) (`EnableKeyRotation`) on any customer managed key used here — AWS managed keys rotate automatically, but customer managed keys require this to be turned on explicitly.

## Table of contents

- [Required actions](#required-actions)
- [Principals that need access](#principals-that-need-access)
- [Restricting `iam:PassRole` on the Batch Operations role](#restricting-iampassrole-on-the-batch-operations-role)
- [Restricting `lambda:InvokeFunction` on the pipeline functions](#restricting-lambdainvokefunction-on-the-pipeline-functions)
- [Key policy statement](#key-policy-statement)
- [Timing](#timing)
- [Deploying IAM principal](#deploying-iam-principal)
- [Governance bypass — not required](#governance-bypass--not-required)
- [S3 Tables integration for server access logs](#s3-tables-integration-for-server-access-logs)

## Required actions

| Action | Why |
|---|---|
| `kms:GenerateDataKey*` | Encrypt objects written to the SolutionBucket (inventory, manifests, completion reports) and encrypt SNS/SQS messages. |
| `kms:Decrypt` | Decrypt objects read from the SolutionBucket and decrypt SQS messages. |
| `kms:Encrypt`, `kms:ReEncrypt*`, `kms:DescribeKey` | Required by the **CloudWatch Logs service principal specifically** — associating a KMS key with a log group (every Lambda's log group, and the server access logs log group) needs this fuller set, not just `GenerateDataKey*`/`Decrypt`. Confirmed directly: a deployment with only `GenerateDataKey*`/`Decrypt` granted to `logs.amazonaws.com` fails every Lambda log group's creation with `AccessDenied` before any Lambda function itself is even created. See [AWS's CloudWatch Logs KMS encryption guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/encrypt-log-data-kms.html) for the full permission set this service needs. |

## Principals that need access

| Principal | Role name pattern | Needs |
|---|---|---|
| StartQuery Lambda | `<stack-name>-startquery-role` | GenerateDataKey*, Decrypt |
| ManifestMaker Lambda | `<stack-name>-manifestmaker-role` | GenerateDataKey*, Decrypt |
| CreateJob Lambda | `<stack-name>-createjob-role` | GenerateDataKey*, Decrypt |
| S3 Batch Operations service role | `<stack-name>-batchops-role` | GenerateDataKey*, Decrypt. Assumed by `batchoperations.s3.amazonaws.com`, not a Lambda — reads the manifest, issues the `PutObjectRetention` calls, and writes the completion report. |
| JobCompletion Lambda | `<stack-name>-jobcompletion-role` | GenerateDataKey*, Decrypt. Triggered by EventBridge on S3 Batch Operations job status-change events (delivered via CloudTrail's always-on management-event stream — no CloudTrail trail is created by this stack); sends the deferred completion email for an active-mode run once every job it created reaches a terminal state — see ["Release mechanism"](ARCHITECTURE.md#release-mechanism) in `ARCHITECTURE.md`. |
| S3 Inventory service | `s3.amazonaws.com` | GenerateDataKey* (writes encrypted inventory to SolutionBucket) |
| CloudWatch Logs service | `logs.amazonaws.com` | Encrypt, Decrypt, ReEncrypt*, GenerateDataKey*, DescribeKey (associates the key with — and reads/writes encrypted data in — every Lambda's log group, always when a KMS key is supplied, plus the server access logs log group when `EnableServerAccessLogs=true`). |
| CloudWatch Alarms service | `cloudwatch.amazonaws.com` | GenerateDataKey*, Decrypt. Needed only because the SNS topic is encrypted with this key: both of the stack's alarms (see [Alarm: job completion with no active-jobs record](OPERATIONS.md#alarm-job-completion-with-no-active-jobs-record) and [Alarm: a pipeline stage failed and was dead-lettered](OPERATIONS.md#alarm-a-pipeline-stage-failed-and-was-dead-lettered)) publish to that topic, and without key access the publish fails with `CloudWatch Alarms does not have authorization to access the SNS topic encryption key`. **The alarm is the only notification for a run that released holds without telling anyone, so losing it is losing exactly the signal it exists to carry** — and it fails at publish time, not at deploy time, so a missing grant here will not show up until the alarm first needs to fire. |

The stack creates the five roles above automatically (four Lambda execution roles plus the S3 Batch Operations service role) and attaches inline policies granting `kms:GenerateDataKey*` and `kms:Decrypt` on the key ARN. However, KMS key policies are evaluated independently of IAM policies — the key policy must explicitly allow these principals to use the key.

## Restricting `iam:PassRole` on the Batch Operations role

**This one is on you, and it is one of two access controls standing between an account principal and an unscheduled hold release.** The other is `lambda:InvokeFunction` on the pipeline functions — see [Restricting `lambda:InvokeFunction` on the pipeline functions](#restricting-lambdainvokefunction-on-the-pipeline-functions) below. Both need restricting; neither is sufficient alone.

`<stack-name>-batchops-role` holds the only `s3:PutObjectRetention` grant in the stack. S3 Batch Operations assumes it to release event holds, so its trust policy must allow `batchoperations.s3.amazonaws.com` to do so — and Batch Operations offers no mechanism to restrict that trust to jobs this solution created. Job IDs do not exist until `CreateJob` is called, so no specific job ARN can be pinned in advance, and `aws:SourceAccount`/`aws:SourceArn` do not help: a same-account principal satisfies both.

The consequence is that **any principal who can both create a Batch Operations job and pass this role can release event holds on any object version under `TargetBucket`/`Prefix`**, with their own manifest, bypassing the eligibility query, `SafetyThreshold`, count verification, and the pipeline's run locks.

The stack scopes its own `iam:PassRole` correctly — `<stack-name>-createjob-role` may pass this role and nothing else. But no template can prevent a *different* identity policy in your account from granting `iam:PassRole` on it. Treat that grant as privileged:

- Do not attach `iam:PassRole` with `Resource: "*"` to any role or user in this account. That is equivalent to granting the ability to release holds.
- Audit who holds `iam:PassRole` covering `arn:aws:iam::ACCOUNT_ID:role/STACK_NAME-batchops-role`, and reduce it to the stack's own `createjob-role`.
- Consider a Service Control Policy or permissions boundary denying `iam:PassRole` on this role ARN for every principal except the stack's `createjob-role`.
- `s3:CreateJob` is the other half of the pair. Restricting it narrows the same path.

**What this is not.** The exposure is an event hold released earlier than intended. The role carries no `s3:BypassGovernanceRetention` and no delete permission, so it cannot shorten an existing `retain-until-date`, delete a version, or remove a delete marker — releasing a hold computes `retain-until-date = MAX(existing, release time + EventHoldDuration)`. Object Lock remains the backstop. See ["Governance bypass — not required"](#governance-bypass--not-required).

## Restricting `lambda:InvokeFunction` on the pipeline functions

**Also on you, and it reaches the same outcome by a different route.** A principal who can invoke the stack's own Lambda functions needs neither `iam:PassRole` nor `s3:CreateJob`, because the functions already hold them.

The stack attaches `AWS::Lambda::Permission` resources so S3, EventBridge and CloudFormation can invoke the functions they need. Those policies only ever grant; they cannot exclude. So an identity policy in your account granting `lambda:InvokeFunction` on these functions lets that principal call them directly with a hand-built payload, and no handler checks who invoked it.

What each function reaches:

- **`<stack-name>-startquery`** — an unscheduled run on a caller-chosen inventory delivery. Deliveries are kept 14 days, so a run can be driven on evidence two weeks old. Eligibility SQL, the schema guard, count verification, `SafetyThreshold` and `ReportOnly` all still apply, and the Glue table location is pinned, so the caller controls timing and snapshot rather than what qualifies.
- **`<stack-name>-createjob`** — claims a run's idempotency lock, suppressing the legitimate run for that delivery.
- **`<stack-name>-jobcompletion`** — records a run as complete and consumes its one completion notification.
- **The five CloudFormation custom-resource functions** — a `RequestType: Delete` payload removes what they set up, including the SolutionBucket's event notifications and the TargetBucket's S3 Inventory configuration.

Treat invoke rights on all nine functions as privileged, as with `iam:PassRole` above:

- Do not grant `lambda:InvokeFunction` with `Resource: "*"` in this account.
- Audit who holds it covering `arn:aws:lambda:REGION:ACCOUNT_ID:function:STACK_NAME-*`.
- Consider a Service Control Policy or permissions boundary denying `lambda:InvokeFunction` on those ARNs for every principal except the roles the stack creates.

**What this is not.** The same bound as above applies: releases still run through `BatchOperationsRole`, which cannot shorten a `retain-until-date`, delete a version, or bypass governance. The worst case is an event hold released earlier than intended, or a run suppressed and its audit record lost.

## Key policy statement

Add explicit statements granting only this solution's roles and the specific service principals it needs — the solution's own roles and S3 need only `kms:GenerateDataKey*`/`kms:Decrypt`; the CloudWatch Logs service principal needs the fuller set explained above and in its own statement below:

```json
{
  "Sid": "AllowEventHoldReleaseRoles",
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "arn:aws:iam::ACCOUNT_ID:role/STACK_NAME-startquery-role",
      "arn:aws:iam::ACCOUNT_ID:role/STACK_NAME-manifestmaker-role",
      "arn:aws:iam::ACCOUNT_ID:role/STACK_NAME-createjob-role",
      "arn:aws:iam::ACCOUNT_ID:role/STACK_NAME-batchops-role",
      "arn:aws:iam::ACCOUNT_ID:role/STACK_NAME-jobcompletion-role"
    ]
  },
  "Action": [
    "kms:GenerateDataKey*",
    "kms:Decrypt"
  ],
  "Resource": "*"
},
{
  "Sid": "AllowS3InventoryDelivery",
  "Effect": "Allow",
  "Principal": {
    "Service": "s3.amazonaws.com"
  },
  "Action": [
    "kms:GenerateDataKey*",
    "kms:Decrypt"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:SourceAccount": "ACCOUNT_ID"
    }
  }
},
{
  "Sid": "AllowCloudWatchLogsEncryption",
  "Effect": "Allow",
  "Principal": {
    "Service": "logs.REGION.amazonaws.com"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt*",
    "kms:GenerateDataKey*",
    "kms:DescribeKey"
  ],
  "Resource": "*",
  "Condition": {
    "ArnLike": {
      "kms:EncryptionContext:aws:logs:arn": "arn:aws:logs:REGION:ACCOUNT_ID:log-group:*"
    }
  }
},
{
  "Sid": "AllowCloudWatchAlarmSNSPublish",
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudwatch.amazonaws.com"
  },
  "Action": [
    "kms:GenerateDataKey*",
    "kms:Decrypt"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:SourceAccount": "ACCOUNT_ID"
    }
  }
}
```

Replace `ACCOUNT_ID` with your 12-digit AWS account ID, `STACK_NAME` with the CloudFormation stack name used at deployment, and `REGION` with the Region you're deploying to (e.g. `us-east-1`) — the CloudWatch Logs service principal is Region-scoped (`logs.<region>.amazonaws.com`), unlike `s3.amazonaws.com`. Keep IAM role principals in the first statement without `aws:SourceAccount`; that context key is available for applicable service-principal requests, not direct calls authorized as those roles. Apply confused-deputy conditions only to the service-principal statements, and note the CloudWatch Logs statement uses an encryption-context condition rather than `aws:SourceAccount`, matching [AWS's own documented pattern](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/encrypt-log-data-kms.html) for scoping a key to specific log groups.

**Confirmed against a real deployment.** A key policy granting `logs.amazonaws.com` only `kms:GenerateDataKey*`/`kms:Decrypt` (an earlier, insufficient version of this statement) fails every Lambda log group's `AWS::Logs::LogGroup` creation with `AccessDenied: The specified KMS key does not exist or is not allowed to be used`, before any Lambda function is created. The five actions above are the confirmed-working set.

**`AllowCloudWatchAlarmSNSPublish` is the exception: documented, not deployment-confirmed.** The two actions come from the AWS guidance for [alarms that notify a KMS-encrypted SNS topic](https://repost.aws/knowledge-center/cloudwatch-metric-filter-alarms) rather than from an observed failure and fix in this stack, because neither alarm has had cause to fire in testing. Unlike every other statement here, a missing or wrong version of this one **will not fail deployment** — it fails silently at the moment an alarm tries to publish. If you supply `KMSKeyArn`, the way to verify it is to set an alarm to `ALARM` by hand and confirm the notification arrives. One statement covers both alarms, so checking either one checks the statement:

```bash
aws cloudwatch set-alarm-state --alarm-name <stack-name>-jobcompletion-missing-record --state-value ALARM --state-reason "key policy check" --no-cli-pager
```

The dead-letter queue alarm is the easier of the two to check against reality rather than by hand, because it watches a metric SQS publishes on its own: send a message to `<stack-name>-pipeline-dlq` and it moves to `ALARM` within a period or two on its own, then back to `OK` once you purge the queue.

**The SolutionBucket's own bucket policy enforces this key.** When `KMSKeyArn` is supplied, `SolutionBucketPolicy` denies any `PutObject` that explicitly specifies a different encryption algorithm or a different KMS key. This does not require any write in the pipeline to set an encryption header itself — the bucket's default encryption (set from `KMSKeyArn` via `BucketEncryption`) already applies transparently to every write that omits the header, which is how every Lambda in this pipeline writes today. The bucket policy only closes the gap default encryption leaves open: a caller that actively names the wrong algorithm or key.

## Timing

The key policy must be in place **before** you run `aws cloudformation deploy`. If the key policy is missing, the stack will fail during resource creation when a Lambda or the S3 Inventory service attempts to use the key.

`AllowCloudWatchAlarmSNSPublish` is the one statement not on that critical path — it is only exercised when the alarm publishes, so its absence is invisible at deploy time and surfaces as a lost notification much later. Add it with the rest rather than deferring it.

## Deploying IAM principal

The deploying IAM principal needs permissions to create CloudFormation stacks, Lambda functions, IAM roles, S3 buckets, Glue databases, Athena workgroups, SNS topics, CloudWatch Logs metric filters, and CloudWatch alarms.

When `EnableServerAccessLogs=true` — which is **not** the default, so a default deployment needs none of the following — the deploying principal also needs:

| Action | Why |
|---|---|
| `logs:PutDeliverySource` | Register the SolutionBucket as a CloudWatch Logs vended log delivery source. |
| `logs:PutDeliveryDestination` | Register the CloudWatch Logs log group as the delivery destination. |
| `logs:CreateDelivery` | Link the delivery source to the destination and start log delivery. |
| `s3:AllowVendedLogDeliveryForResource` | Required on the SolutionBucket for CloudWatch Logs vended delivery to be permitted. |

When `EnableS3TablesIntegration=true` (its own default, but inert unless `EnableServerAccessLogs=true`) is also set, the deploying principal additionally needs `observabilityadmin:CreateS3TableIntegration`, `observabilityadmin:ListS3TableIntegrations`, and `observabilityadmin:GetS3TableIntegration`, plus `iam:PassRole` for the role the stack creates for the integration (`<stack-name>-s3tableintegration-role`).

## Governance bypass — not required

This solution releases event holds using `PutObjectRetention` with `EventHold=OFF`. Both the Compliance-mode and Governance-mode release paths set `BypassGovernanceRetention=false`.

Releasing an event hold is an **allowed** S3 Object Lock operation that does not require a governance bypass, because it never shortens protection. `s3:BypassGovernanceRetention` is required only to override governance-mode protection — for example, to permanently delete a version, change the retention mode, or shorten the retain-until-date. The `retain-until-date = MAX(existing, release time + EventHoldDuration)` formula ensures protection is never shortened, so no override is needed and the S3 Batch Operations role (which issues the `PutObjectRetention` calls, see ["Release mechanism"](ARCHITECTURE.md#release-mechanism) in `ARCHITECTURE.md`) carries no `s3:BypassGovernanceRetention` permission.

## S3 Tables integration for server access logs

The `AWS::ObservabilityAdmin::S3TableIntegration` resource is scoped to the whole AWS account and Region — only one exists per account per Region, shared by every CloudWatch Logs source that opts into S3 Tables mirroring. Because of this, the stack does not create it unconditionally: a custom resource checks whether an integration already exists in the account/Region and reuses it if so, creating one only when none exists.

**On stack deletion**, this custom resource intentionally does **not** disassociate the `amazon_s3`/`server_access` data source or delete the integration. Both are account-and-Region-wide settings that other buckets, stacks, or AWS services may depend on; removing them would silently stop S3 Tables mirroring for those other resources. If you need to fully remove the integration (for example, decommissioning the last stack using it), do so manually via the CloudWatch Logs console or `aws logs delete-delivery-source` / `aws observabilityadmin delete-s3-table-integration`, after confirming no other source depends on it.

**The IAM role the integration uses is retained for the same reason, and is the third thing to clean up by hand.** The integration is created holding the ARN of `<stack-name>-s3tableintegration-role`, and nothing ever updates that reference. So the role carries `DeletionPolicy: Retain` — if the stack deleted it, the surviving integration would point at a role that no longer exists, CloudWatch Logs could not assume it, and mirroring would stop for every bucket and stack sharing the integration in that account and Region. That is exactly the outcome the no-op delete above exists to prevent.

The consequence is that deleting a stack leaves the role behind. That is correct while any other stack still shares the integration, and it is litter once none does. When you remove the integration, remove the role too:

```bash
aws iam delete-role-policy --role-name <stack-name>-s3tableintegration-role --policy-name IntegrateWithS3Table --no-cli-pager
aws iam delete-role --role-name <stack-name>-s3tableintegration-role --no-cli-pager
```

Check which role the integration references before deleting anything. With several stacks deployed over time, the surviving integration may point at a role from a stack deleted long ago, and that role is the one that has to outlive the rest.

If a customer managed KMS key is used for the log group and the S3 Tables integration, additional KMS key policy grants are required for the `systemtables.cloudwatch.amazonaws.com` and `maintenance.s3tables.amazonaws.com` service principals — see the [Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/sal-cw-enabling.html#sal-cw-enabling-encryption) section of the AWS documentation for the exact statements required.

### `KMSKeyArn` propagates to the integration, and the integration is shared

**Supplying `KMSKeyArn` encrypts the S3 Tables integration with that key, and this has account-wide reach.** When the stack creates the integration, it passes `SseAlgorithm: aws:kms` with your key; when no `KMSKeyArn` is given it uses SSE-S3 (`AES256`), which is the behaviour every deployment had previously. Two consequences to weigh before supplying a key:

- **Only the creating stack sets this.** The integration is one account-and-Region-wide resource, and the stack reuses an existing one untouched. A stack deployed into an account that already has an integration will not change its encryption, whatever `KMSKeyArn` is set to — so encryption of the integration is decided once, by whichever stack created it.
- **It applies to every source sharing the integration.** All S3 server access log mirroring in that account and Region flows through the one integration, so mirrored log data for buckets this stack knows nothing about is encrypted under your key. Any principal reading those tables needs `kms:Decrypt` on it, and the two service principals above need their key policy grants. In a shared account, coordinate before supplying a key, or leave `KMSKeyArn` empty so the integration stays on SSE-S3.

This affects mirrored server access log metadata only. It has no bearing on TargetBucket object content or on the release path.

**Not verified against a live deployment with a customer managed key.** Whether the `-s3tableintegsetup-role` additionally needs `kms:DescribeKey` on the key to call `CreateS3TableIntegration` has not been exercised, so no such grant is in the template. If stack creation fails at the `S3TableIntegrationSetup` custom resource with a KMS authorization error, that grant is the first thing to add.
