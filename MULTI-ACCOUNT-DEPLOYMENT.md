# Deploying across multiple buckets, accounts, and Regions

How to roll out [Automatic event hold release for Amazon S3 Object Lock](README.md) beyond a single stack: what the deployment unit actually is, the naming limits that bite once you have more than one stack, what is shared across stacks, and how to drive the rollout.

For a single-stack deployment, see [Deployment](README.md#deployment) in `README.md`. Nothing on this page changes how an individual stack works.

## Table of contents

- [The deployment unit is one bucket, not one account or Region](#the-deployment-unit-is-one-bucket-not-one-account-or-region)
- [Why each stack has to sit with its bucket](#why-each-stack-has-to-sit-with-its-bucket)
- [Stack naming rules](#stack-naming-rules)
- [What is shared and what repeats](#what-is-shared-and-what-repeats)
- [KMS keys across many stacks](#kms-keys-across-many-stacks)
- [Per-account prerequisites](#per-account-prerequisites)
- [Notifications across many stacks](#notifications-across-many-stacks)
- [Rolling out with StackSets](#rolling-out-with-stacksets)
- [Rolling out with a script](#rolling-out-with-a-script)
- [Cost](#cost)

## The deployment unit is one bucket, not one account or Region

**One stack covers exactly one bucket.** `TargetBucket` is a single bucket name fixed at deploy time — it cannot be changed on a stack update, and the prerequisite check rejects the attempt (see [Disallowed: changing TargetBucket](OPERATIONS.md#disallowed-changing-targetbucket)). The stack must be deployed in the same account and Region as that bucket.

So the number of stacks you need is driven by the number of buckets, not the number of account/Region pairs:

| Scenario | Stacks |
|---|---|
| 1 bucket in 1 account, 1 Region | 1 |
| 3 buckets in 1 account, 1 Region | 3 |
| 1 bucket in each of 2 Regions, same account | 2 |
| 1 bucket in each of 4 accounts, 1 Region each | 4 |
| 2 buckets in each of 4 accounts, 2 Regions each (2 buckets per pair) | 16 |
| 1 very large bucket sharded across 3 non-overlapping prefixes | 3 |

An account/Region pair is therefore the *minimum* granularity, not the unit: a pair with three Object Lock buckets needs three stacks, and a single bucket can need more than one stack if you shard it by prefix to stay under the Athena DML timeout (see [Athena DML timeout](OPERATIONS.md#athena-dml-timeout)).

Put the other way round: **at least one stack per account/Region pair that contains an Object Lock bucket you want covered, and one per bucket within each pair.**

## Why each stack has to sit with its bucket

The Region constraint is imposed by S3; the account constraint is a design decision in this template.

**Same Region — enforced by S3 Inventory.** The stack creates the SolutionBucket as the inventory destination for `TargetBucket`, and an S3 Inventory destination bucket [must be in the same AWS Region as the source bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html). A stack in a different Region could not receive inventory for the bucket at all. Everything downstream is regional too: the Glue database and table, the Athena workgroup, the S3 Batch Operations jobs, the Lambda functions, and any KMS key you supply.

**Same account — a property of this template.** S3 Inventory itself supports a destination bucket owned by a different account, so this is not a service limit. This template does not use that capability:

- The inventory configuration sets the destination's expected owner to the deploying account.
- `SolutionBucketPolicy` admits inventory delivery only when `aws:SourceAccount` equals the deploying account and `aws:SourceArn` is `TargetBucket`.
- The inventory-config, prerequisite-check, and Batch Operations roles reference `TargetBucket` by name and are assumed within the deploying account, with no cross-account trust or bucket-policy grant on the target side.

A centralised, cross-account variant is possible but would need changes across all four of those points plus a bucket policy on every target bucket. It is not supported as shipped. **Deploy into the account that owns the bucket.**

## Stack naming rules

Multiple stacks in one account and Region are supported by design — every named resource is derived from the stack name so deployments do not collide. That derivation imposes four naming rules. The first two apply to any deployment, including a single stack; the last two only surface once you have more than one.

| Rule | Scope | Why |
|---|---|---|
| Stack name ≤ **43 characters** | Every stack | The stack creates named IAM roles as `<stack-name>-<suffix>-role`. The longest unconditional suffix is `-inventoryconfig-role` (21 characters) and an [IAM role name is capped at 64 characters](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-iam-role.html#cfn-iam-role-rolename). |
| Stack name ≤ **40 characters** | Only when `EnableServerAccessLogs=true` **and** `EnableS3TablesIntegration=true` | Those settings add `-s3tableintegration-role` (24 characters), the longest role suffix in the template. |
| Stack names unique **per account, across all Regions** | Same account, different Regions | An [IAM role name must be unique within the account and is not distinguished by case](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-iam-role.html#cfn-iam-role-rolename) — role names are account-global, not regional. Deploying the same stack name (or one differing only in case) into two Regions of one account fails the second stack on a duplicate role name. Give each Region's stack a distinct name, for example by including the Region in it. |
| Stack names distinct in the **first 32 characters, case-insensitively** | Same account and Region | The SolutionBucket name is the lowercased stack name truncated to 32 characters plus `-<AccountId>-<Region>-an`. Two stacks whose lowercased names match over the first 32 characters resolve to the same bucket name and the second one fails. The Glue database, `<lowercase-stack-name>-db`, is case-insensitive over the whole name. |

The SolutionBucket is created with `DeletionPolicy: Retain`, so a collision on that name leaves a retained bucket behind that you then have to clean up manually before retrying — see [Stack deletion](OPERATIONS.md#stack-deletion).

A naming convention that satisfies all four rules is a short fixed prefix, the Region, and a bucket identifier, kept within 40 characters so it works whether or not you later enable access logging:

```
eh-<region-short>-<bucket-id>     # e.g. eh-use1-records-archive (23 characters)
```

Everything else the stack names — the Lambda functions, the SQS dead-letter queue, the Athena workgroup, the EventBridge rule, the CloudWatch alarm, and the log groups — has enough headroom that the four rules above are the binding ones.

## What is shared and what repeats

Most of the stack is per-stack, which is what makes independent deployments safe. A few things are not, and they are the ones to watch as stack count grows.

| Resource | Scope | Implication for many stacks |
|---|---|---|
| SolutionBucket, Glue database and table, Athena workgroup, Lambda functions, IAM roles, SNS topic, dead-letter queue, CloudWatch alarm | Per stack | Independent. No coordination needed beyond the naming rules above. |
| S3 Inventory configuration on the target bucket | Per stack, identified by stack name | S3 allows [up to 1,000 inventory configurations per bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html), so prefix sharding across several stacks on one bucket is well within limits. Stack deletion removes only that stack's own configuration. |
| S3 Tables integration for access logs | **One per account and Region**, shared by every stack | Only the first stack to need it creates it, and no stack deletes it. Its encryption is decided once, by whichever stack created it. See [Account-and-Region-wide resource, shared across stacks](OPERATIONS.md#solutionbucket-access-logging) and [PERMISSIONS.md](PERMISSIONS.md#s3-tables-integration-for-server-access-logs). |
| Athena Active DML queries quota | **Per account and Region** | The default is [20 in most Regions](https://docs.aws.amazon.com/general/latest/gr/athena.html) (higher in a few, such as 200 in `us-east-1`), and the quota counts queued as well as running queries. Each run issues a `COUNT(*)` and an `UNLOAD` per Object Lock mode, plus a diagnostic query in `delete`/`overwrite` modes. Many stacks in one Region whose inventory lands at the same time can contend; exceeded requests fail with `TooManyRequestsException`, and a failed StartQuery invocation lands in that stack's dead-letter queue rather than being lost. Staggering `DetectionSchedule` across stacks, or raising the quota, avoids it. |
| CloudWatch alarms free tier | Per account | The always-free tier covers 10 alarm metrics per account per month and each stack creates two alarms, so the free tier covers five stacks. Beyond that the alarms become the line that scales with stack count — see [Example 2](COST.md#example-2-twenty-small-buckets-one-batch-each-per-month) in `COST.md`, which also covers consolidating them into account-wide equivalents. |
| Deploying principal permissions | Per account | See [Per-account prerequisites](#per-account-prerequisites). |

## KMS keys across many stacks

`KMSKeyArn` is optional; leaving it blank (the default) removes this section's concerns entirely, because the SolutionBucket falls back to SSE-S3 and SNS to its default AWS-owned key.

If you do use a customer managed key, two things follow at scale. A KMS key is regional, so a multi-Region rollout needs at least one key per Region. And the key policy has to name the stack's roles explicitly — key policies are evaluated independently of the IAM policies the stack attaches, so the [key policy statement](PERMISSIONS.md#key-policy-statement) lists five role ARNs built from the stack name.

Sharing one key across several stacks in an account and Region therefore means either:

- **Listing every stack's roles** in the `AllowEventHoldReleaseRoles` statement. Explicit and easy to audit, but the policy grows by five ARNs per stack, against a [32 KB (32,768 byte) key policy limit](https://docs.aws.amazon.com/kms/latest/developerguide/key-policy-overview.html), and it has to be updated before each new stack deploys.
- **Matching the roles by name pattern**, using `Principal: {"AWS": "ACCOUNT_ID"}` with an `ArnLike` condition on `aws:PrincipalArn` such as `arn:aws:iam::ACCOUNT_ID:role/eh-use1-*-role`. Fixed size regardless of stack count, but it grants key use to *any* role matching the pattern, so the prefix has to be one you reserve for these stacks and control who can create roles under.
- **One key per stack.** No policy coupling at all, at the cost of a monthly key charge per stack.

Whichever you choose, the key policy must be in place **before** the stack deploys — a missing grant fails resource creation. The one exception is `AllowCloudWatchAlarmSNSPublish`, which fails silently much later, at the moment the alarm first tries to publish. See [Timing](PERMISSIONS.md#timing).

## Per-account prerequisites

These are per-account, so they repeat for every account in the rollout and are the usual cause of a batch of failed deployments:

- **Object Lock and Versioning enabled on every target bucket.** The prerequisite check fails the stack at create time otherwise, naming which of the two is missing. Object Lock cannot be enabled retroactively on a bucket without Versioning already on.
- **Deploying principal permissions** in each target account, per [Deploying IAM principal](PERMISSIONS.md#deploying-iam-principal). The list is longer when `EnableServerAccessLogs=true`, longer again with `EnableS3TablesIntegration=true`.
- **`CAPABILITY_NAMED_IAM`**, because the template creates named IAM roles.
- **`iam:PassRole` audit on `<stack-name>-batchops-role`** in each account. This is the main access control between an account principal and an unscheduled hold release, it is not something the template can enforce, and it is per stack — so a rollout multiplies the number of role ARNs you need to keep that grant scoped on. Read [Restricting `iam:PassRole` on the Batch Operations role](PERMISSIONS.md#restricting-iampassrole-on-the-batch-operations-role) before deploying widely.
- **A `NoncurrentVersionExpiration` lifecycle rule on each target bucket.** The stack does not create or modify one, and without it releasing a hold never leads to a version being deleted. See [Lifecycle integration](OPERATIONS.md#lifecycle-integration).

## Notifications across many stacks

Each stack creates its own SNS topic and publishes that stack's run summaries, withheld-mode notices, failures, and alarm notifications to it. There is no built-in aggregation across stacks.

For a handful of stacks, subscribing the same endpoint to every topic is enough; each message identifies its own stack. For a larger estate, subscribe a single Lambda or an SQS queue to all of the topics and aggregate there — SNS supports cross-account and cross-Region subscriptions, so the aggregator can live in one place. Keep in mind that report-only runs and withheld modes are the messages that actually need a human, so whatever you build should not bury them.

## Rolling out with StackSets

StackSets can drive this template across accounts and Regions, with one constraint that shapes the whole design.

**A stack set holds at most one stack in a given account and Region.** A [stack instance is a reference to a stack in a specific account and Region](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_StackInstance.html), and [parameter overrides are keyed by account and Region](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stackinstances-override.html). Since `TargetBucket` is a parameter, **one stack set can cover only one bucket per account/Region pair.** This is the sense in which "one per pair" is true: it is a property of stack sets, not of the solution.

That gives two workable shapes:

- **Uniform bucket names across accounts.** Common in landing zones where each account has a bucket following a fixed naming convention. One stack set, one Region list, `TargetBucket` set once at the stack set level, no overrides. Simple, and it updates every account in one operation.
- **Different bucket names per account, or several buckets per pair.** One stack set per "bucket slot", with `TargetBucket` overridden per account and Region. An account/Region pair with three buckets needs three stack sets.

Points to get right before the first rollout:

- **Stack names are generated, not chosen.** With StackSets you name the stack set; CloudFormation names each stack. Because this template's named IAM roles cap the stack name at 43 characters (40 with the S3 Tables integration role), confirm the generated name fits before rolling out widely. Deploy one stack instance first and check the stack name length in the target account. *This specific interaction has not been verified against a live StackSets deployment of this template — treat the single-instance test as required, not optional.* If the generated name is too long, deploy with a script instead, where you control the stack name.
- **`CAPABILITY_NAMED_IAM`** must be declared on the stack set.
- **Permission model.** Service-managed permissions with AWS Organizations avoid creating the cross-account roles yourself and support automatic deployment to new accounts. Self-managed permissions work for accounts outside an organization. See [Permission models for StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html).
- **A stack set is a regional resource.** It is visible only in the Region you created it in, even though it deploys to many Regions.
- **Region concurrency.** Deploying Regions sequentially with a low failure tolerance surfaces a bad parameter on the first Region instead of across all of them.
- **Prerequisites are not checked by the stack set.** A target account whose bucket lacks Object Lock fails its stack instance, not the operation. Verify prerequisites across accounts before deploying, and read the per-instance status reasons afterwards rather than only the operation status.

## Rolling out with a script

Because each stack is fully independent, a loop over a list of targets is a legitimate alternative to StackSets, and it gives you control over the stack name. It suits an estate with irregular bucket names, and it keeps report-only rollout under your control.

Keep the target list in a file rather than in the script:

```csv
profile,region,bucket,stack
prod-archive,us-east-1,amzn-s3-demo-object-lock-bucket,eh-use1-archive
prod-records,eu-west-1,amzn-s3-demo-records-bucket,eh-euw1-records
```

Then deploy one stack per row, leaving `ReportOnly` at its default so nothing is released until you have reviewed each stack's first run:

```bash
tail -n +2 targets.csv | while IFS=, read -r profile region bucket stack; do
  aws cloudformation deploy \
    --stack-name "$stack" \
    --template-file template.yaml \
    --capabilities CAPABILITY_NAMED_IAM \
    --profile "$profile" \
    --region "$region" \
    --parameter-overrides \
      TargetBucket="$bucket" \
      ReleaseMode=either \
      ReportOnly=true \
      SafetyThreshold=500
done  # Deploy one report-only stack per target bucket
```

Deploy report-only first across the whole estate, review each stack's suspended jobs and eligibility manifests, then switch stacks to `ReportOnly=false` individually as you gain confidence — re-running the same command with the changed value is a stack update and needs no teardown. See [Reviewing the first run and switching to active mode](README.md#reviewing-the-first-run-and-switching-to-active-mode).

Two things worth adding for a real rollout, both left out above to keep the shape clear: check the exit status of each deploy and collect the failures rather than letting the loop run on, and confirm Object Lock and Versioning on each bucket up front so you find out before CloudFormation does.

## Cost

Per-stack fixed costs dominate a wide, shallow rollout. The S3 Batch Operations per-job charge is the same $0.25 whether a job releases 10 versions or 10,000, and the CloudWatch alarm is the one line that scales purely with stack count. [Example 2](COST.md#example-2-twenty-small-buckets-one-batch-each-per-month) in `COST.md` models exactly this shape — twenty small buckets as twenty independent stacks — and is the right starting point for estimating an estate rather than a single bucket.
