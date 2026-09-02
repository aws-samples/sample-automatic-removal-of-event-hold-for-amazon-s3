# Classifying versions

How [Automatic event hold release for Amazon S3 Object Lock](README.md) decides whether a noncurrent version was *deleted* or *overwritten*, what happens when that can't be determined, and which [release mode](README.md#release-modes) to choose for write-once data.

## `either` mode needs no classification

`is_latest='false'` alone establishes that a version is noncurrent — S3 ensures at most one `is_latest='true'` entry (version or delete marker) per key — so `either` mode selects every noncurrent version with an active event hold directly, without looking at what superseded it. **Nothing is ever withheld in `either` mode.** Everything below applies only to `delete` and `overwrite`.

## How classification works for `delete`/`overwrite`

`delete`/`overwrite` mode inspects the **immediate superseding version** — the next version created on the same key, ordered by `LastModifiedDate` — to determine whether a noncurrent version was deleted or overwritten.

If a key was deleted and then re-uploaded (delete marker → new data version), the original version is classified as delete-superseded because its immediate superseder is the delete marker, regardless of the later put. `delete` mode therefore correctly captures ransomware-recovery scenarios even when objects are subsequently restored.

## Timestamp ties fail closed

S3 Inventory provides no sub-second chronological tiebreaker, but `is_latest` still carries ordering information: the current version is always the last version in a key's chain. Rather than guessing an order among timestamp-tied versions, the pipeline classifies each candidate by the **set of versions its immediate superseder could possibly be**: any other version sharing the candidate's timestamp, plus the earliest-possible version at the key's next timestamp.

If every possible superseder agrees on being a data version (or all are delete markers), the classification is determined no matter which one actually came first — the candidate is classified and released normally. Only when the possibilities genuinely disagree — a data version *and* a delete marker are both possible superseders — is that candidate **withheld** from `delete`/`overwrite` eligibility for the run. Two consequences worth noting:

- A tie involving only the candidate and the current version is always resolved (`is_latest` pins the current version last) — the common rapid-overwrite or delete case where the superseding write lands in the same inventory-reported second.
- Withholding is per candidate, not per key: other versions of the same key with determinable superseders remain eligible.

`either` mode doesn't identify an immediate superseder at all, so it's unaffected — even a genuinely-tied version is still eligible under `either`.

## Reviewing withheld candidates

Withheld candidates aren't lost. Every `delete`/`overwrite` run that withholds at least one candidate writes a diagnostic listing to `diagnostics/withheld-candidates/<dt>/manifest.csv`, backed by a `withheld_candidates_history` Glue table (query it via Athena in the same database as the main `inventory` table).

It reports **exactly** the candidates the eligibility query withheld — both are derived from the same predicate — one row per withheld candidate: `key`, `version_id`, `object_lock_mode`, `last_modified_date`, and `reason`:

| `reason` | Meaning |
|---|---|
| `tied_noncurrent_peer` | The candidate shares its timestamp with another noncurrent version. |
| `ambiguous_successor_classification` | The candidate isn't itself tied, but the versions at its next timestamp mix a data version and a delete marker. |

Both lock modes are covered in the one listing. This is purely informational — it never gates a job decision — but it's how you find out a version is stuck this way. Resolved ties are not reported, since they aren't withheld. The count for the most recent run is also carried in `_job_summary.json` and the SNS notification as `withheld_candidate_count`.

## Whether a withheld candidate stays stuck

`LastModifiedDate` on an existing version never changes, so a tie only clears when one of the tied versions is removed. Two cases:

- **Tie involving a delete marker — self-heals.** A delete marker can never carry an event hold, so a `NoncurrentVersionExpiration` lifecycle rule removes a noncurrent delete marker on its `NoncurrentDays` schedule regardless of this solution. Once the delete marker is gone, the surviving data version's superseder set becomes unanimous on a later inventory run and it is classified and released automatically. No manual action needed — just time.
- **Tie between two held data versions — genuinely stuck.** Both versions keep their holds (neither can be classified), nothing ages out, and the pair keeps appearing in the diagnostic every run. Clearing it is a manual step: release the holds yourself and let the lifecycle rule delete the versions after their retention elapses. There's no automated remediation for this case.

## Classification is per-hop, not per-history

In a chain v1 → v2 → delete marker, v1 (superseded by v2) is `Overwrite_Superseded` and stays that way permanently — a later delete of v2 does not reclassify v1. v2 (superseded by the delete marker) is `Delete_Superseded` on its own, independent of how v2 itself came to exist.

This matters when a version's own supersession reason and its provenance (how *it* was created) diverge. For example, if v2 was created by an unexpected overwrite of a key that a downstream system expected to be write-once, releasing v2 under `delete`/`either` mode isn't a correctness problem for the current data (v1 is already protected), but it does shorten the retention window on the anomalous version itself before it can be reviewed. If preserving that evidence matters, review `Overwrite_Superseded` versions before enabling active release, regardless of `ReleaseMode`.

## Recommendation for write-once table formats (e.g. Apache Iceberg)

If the Target_Bucket holds table data from a format whose write path is write-once per key (data files, metadata files, and manifests are never overwritten in place — Apache Iceberg is the primary example), **use `ReleaseMode=delete`**, not `overwrite` or `either`.

Such formats never issue a second PUT to an existing key as part of normal operation. If a key under the table's management has more than one version where the current version is a *newer data version at the same key* (an `Overwrite_Superseded` noncurrent version), that version did not come from the table format's own write path — it came from something outside normal operation. `overwrite` and `either` modes would release the event hold on that version, which may be exactly the version the table's own metadata still expects to find.

`delete`-superseded noncurrent versions are, by contrast, a normal and expected outcome of the format's own maintenance operations (for Iceberg: `expireSnapshots`, `removeOrphanFiles`), which only delete a file after confirming it is no longer referenced. `ReleaseMode=delete` is therefore the safer setting for this class of Target_Bucket, while `overwrite`/`either` are not — see the [per-hop classification note](#classification-is-per-hop-not-per-history) for a caveat that still applies even under `delete` mode.

**Note.** Even under `ReleaseMode=delete`, a delete marker's origin cannot be distinguished from S3 Inventory data alone — a version-level inventory row looks the same whether the delete came from a trusted, reference-checked maintenance job or from something outside the table format's control (a stray manual delete, a misfired lifecycle rule, ransomware). `ReleaseMode=delete` removes the write-once anomaly risk above but does not by itself re-verify that a specific noncurrent version is unreferenced by the table's current metadata.
