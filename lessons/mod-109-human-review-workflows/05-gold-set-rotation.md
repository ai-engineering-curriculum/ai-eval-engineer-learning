# Gold-Set Rotation with Product-Surface Change

## Motivation

The mod-106 release-gate report is a number against a reference set — the gold set. A model regression is a number that moved the wrong way against that reference. The reference is only load-bearing if the rows in it still resemble the traffic the surface sees. When the product ships a new tool, promotes a new policy category, upgrades a model, or revises a rubric, the gold set that was representative yesterday is partially obsolete today. A gold that is 40 % obsolete is reporting a number on a surface that no longer exists.

Three grounded reasons this chapter exists as its own axis.

- **Drift against the surface is not the same as drift in the metric.** The mod-107 drift monitor catches shifts in the *scored* numbers. If the gold set itself drifted away from the product (new user cohorts arrived that the gold does not cover, a new tool is now the primary path but the gold has zero coverage), the scored number goes up and the drift monitor is silent. The surface regressed; the eval did not notice.
- **The old number has to stay reproducible.** A release-gate from six months ago that reported 0.84 faithfulness is cited in change logs, compliance audits, and vendor communications. Six months later someone asks "does the current model pass the same bar?" The current model is fine; the gold has rotated. Without a sealed snapshot of the old gold, the historical number is unreproducible — the audit question has no clean answer.
- **The rotation is a human-review step, not an automated regeneration.** A gold that regenerates itself from production traces drifts with the traces; the whole point of a gold is that it is *anchored*. The anchor is a human reviewer (or a panel) who approved the rotation against the rubric. This module is where that approval lives.

This chapter walks why gold sets drift (four surface-change triggers), the frozen-vs-rolling split, the rotation cadence tied to the mod-106 release calendar and the mod-107 drift alerts, the sealed-snapshot discipline that keeps old numbers reproducible, the human-review approval step, and the coordination with mod-104's calibration set and mod-108's safety set.

## Core concepts

### Why gold sets drift — the four triggers

Four kinds of change in the product surface invalidate part of a gold set. Watch for all four; each triggers a different rotation.

- **New product features or surfaces.** A support-bot launches a "cancel my subscription" flow. The gold set has zero rows on cancel flows; the mod-106 report has nothing to say about whether the model handles cancellations. Rotation: add representative cancel-flow rows; label them against the rubric.
- **New policy categories or taxonomy shifts.** The mod-108 refusal reconciliation adds a `financial_advice` category. Old gold rows had no `policy_verdict` for financial-advice prompts; the reconciliation report cannot score them. Rotation: back-label the relevant subset of the gold against the new policy, add net-new rows that exercise the category.
- **Model upgrades.** The upstream provider ships `gpt-4o-2025-11-01` (or the in-house model promotes a new checkpoint). The old gold was labelled with a judge calibrated on the old model's outputs; the labels may need to be re-validated. Rotation: re-run the mod-104 quick-calibration on the gold; if κ drops, re-label the affected rows.
- **Policy or rubric updates.** The rubric adds a label, splits a category, or changes a definition. Every row's label against the old rubric no longer tells the full story under the new. Rotation: re-label a sample against the new rubric; adjudicate the delta; rewrite the affected rows.

Three of the four are external events the eval team reacts to (a feature launch, a provider snapshot, a policy revision). The fourth — rubric updates — is an internal event chapter 04 governs. The rotation discipline reads all four from a single change log.

### Frozen gold vs rolling gold — the distinction that keeps old numbers alive

Two gold-set shapes serve two different jobs; a mature program keeps both.

- **Frozen gold.** A sealed snapshot at a point in time, with the rubric version, policy version, and model version of that time attached. Frozen golds do not change. The mod-106 report for release `v2025.08.14` ran against the frozen gold `support-bot/gold_v11`; six months later the same report, re-run against `support-bot/gold_v11`, must produce the same number bit-identically.
- **Rolling gold.** The current working gold the eval team maintains. Rolls forward as surface changes land. Represents *today's surface*. Mod-106's current release-gate reads this; mod-104's judge calibrates against this; mod-107's drift baseline references this.

The relationship: each rotation produces a new rolling gold and seals the prior rolling gold as a frozen version. The working artefact rolls; the sealed artefact never changes. Downstream consumers reference either by name (`support-bot/gold_current` vs `support-bot/gold_v11`).

Two anti-patterns:

- **Rolling only, no freeze.** The gold rotates monthly; the old numbers are unreproducible by week two of the next month. An audit asking "what was faithfulness at v2025.08.14?" has no clean answer.
- **Frozen only, no roll.** The gold never changes; by month six the surface is 40 % new and the gold covers none of it. The release-gate report reports a number on yesterday's surface.

### Rotation cadence — tied to the release calendar and the drift monitor

The rotation does not run on its own clock; it runs on two upstream signals.

- **The mod-106 release calendar.** Each release is a candidate for a rotation. If the release introduces a new surface, a new policy category, or a model upgrade, the rotation is required before the release ships. The gate is two-sided: the release does not ship without a rotated gold; the gold does not rotate without a signed-off reviewer panel.
- **The mod-107 drift monitor.** A persistent drift signal that resolves to "the gold no longer covers this cohort" — detected during on-call triage — triggers a mid-cycle rotation. This is less common but necessary when a cohort shift arrives between releases.

The typical cadence for a monthly-release product is one rotation per month, aligned with the release. A quarterly product aligns with the quarterly release. A continuously-releasing product with a slow-changing surface may rotate quarterly; a continuously-releasing product with a fast-changing surface may rotate every two weeks. The cadence is a trade-off between operational cost (each rotation is reviewer labour, chapter 04 IAA re-runs, sealed-snapshot bookkeeping) and representational freshness.

### The rotation step — what actually happens

A rotation is a six-step procedure, run per surface, per rubric.

- **Step 1 — Diff the surface.** List the net-new behaviours the release introduced: new tools, new policy categories, new supported user intents. Pull a sample of production traces for each from the mod-102 trace store.
- **Step 2 — Propose candidate rows.** From the sampled traces, propose a set of candidate rows for the new rolling gold. Target coverage is proportional to the surface's share of production traffic; a new feature at 5 % of traffic warrants ~5 % of the rolling gold by row count.
- **Step 3 — Label candidate rows.** Pass the candidates through the chapter 02 workbench. Two reviewers label; chapter 04's IAA is computed before any row ships into the gold. A cluster of rows with κ < 0.6 triggers a rubric review before the rotation closes.
- **Step 4 — Approve the rotation.** A senior reviewer (the rubric owner for product-quality rubrics, the policy owner for safety rubrics) reads the delta — what rows are new, what rows are retired, what labels changed — and signs off. The sign-off is a row in `rotation_approvals` in mod-110 with the reviewer, the rubric version, the policy version, and the date.
- **Step 5 — Seal the prior rolling gold.** Snapshot `support-bot/gold_current` as `support-bot/gold_v11` (or whatever the next frozen tag is). Freeze the rubric version, the policy version, the model version at seal time, and attach them to the snapshot.
- **Step 6 — Publish the new rolling gold.** Replace `support-bot/gold_current` with the rotated set. Notify the mod-106 release-gate job, the mod-104 calibration job, and the mod-107 drift-reference job. Each re-runs on the new gold; mod-106 compares numbers across rotations and reports the delta in the next release note.

Two disciplines make the rotation defensible:

- **The new gold passes the old numbers with an explainable delta.** Bootstrap the mod-106 release-gate score itself on the new gold — resample rows with replacement, recompute the score per resample, take the 2.5 / 97.5 percentiles — to get a CI on the release-score. Chapter 04's κ / α CI quantifies reviewer-agreement sampling variance and is not a valid threshold for a release-score shift; the two bootstraps measure different things. If the release-gate score delta between the old and new gold exceeds the release-score CI, the rotation materially moved the top-line number — re-examine the new rows. A gold rotation should not silently move the top-line number.
- **The retired rows do not disappear.** Rows in the old gold that no longer apply (e.g. a feature that was removed) are not deleted; they are sealed in the prior frozen snapshot and archived. The sealed snapshot is complete even for retired rows.

### Sealed snapshots — the lineage columns and the re-run

A sealed snapshot is a tar / parquet / dataset-registry artefact with these fields per row:

- All content and lineage columns from chapter 02's schema.
- The sealed `rubric_version` and `policy_version` at seal time.
- The sealed `model_version` (the model the release-gate will run against for historical re-runs is pinned here).
- A `snapshot_id` with the frozen tag and an `is_frozen=true` flag.

Re-running the historical number requires:

- Pulling the frozen snapshot by `snapshot_id`.
- Running the mod-106 release-gate against the snapshot with the sealed `model_version` pinned (either against the sealed model directly if the provider still supports the snapshot, or against the replay cache if the model has been retired — chapter 06 of mod-106 formalises the replay cache).
- Comparing the current result to the sealed result; the two should agree bit-identically modulo known provider nondeterminism.

<!-- needs-research: on the next research cycle, re-verify the current retention policies and snapshot-availability guarantees of the major providers (OpenAI, Anthropic, Google) for named model snapshots — the retirement cadence changes. -->

Mod-110 chapter on dataset storage walks the mechanics; this module only guarantees the lineage columns are present and the approval row exists.

### Coordination with mod-104's calibration set and mod-108's safety set

The gold set is not the only labelled artefact the eval program maintains. Three artefacts share rows but have distinct roles.

| Artefact | Owner | Role | Rotation |
|---|---|---|---|
| **Gold set** (this chapter) | eval engineer + rubric owner | Reference for mod-106 release-gate and mod-104 judge calibration | Tied to release calendar / drift |
| **Calibration set** (mod-104) | eval engineer | Working set the judge calibrates against; may rotate faster than the gold | Nightly refresh from reviewed rows |
| **Safety set** (mod-108) | policy owner | Policy-reconciliation set for the refusal / over-refusal eval | Tied to policy revision |

Three disciplines hold the coordination:

- **Row provenance is tracked, not re-labelled.** A row that originated in the gold and then entered the calibration set is still labelled against the same rubric version; the two sets reference the same labelled artefact in mod-110, not two copies.
- **Rotation of one triggers a check on the others.** A rubric revision rotates the gold and invalidates the calibration set (chapter 04 IAA runs); a policy revision rotates the safety set and may invalidate the gold's safety-category rows (back-label).
- **The frozen snapshot is per-artefact.** Gold snapshots live in `gold_snapshots`; safety snapshots in `safety_snapshots`; the calibration set is working-only and is not frozen (it is derived from reviewed rows; the reviewed rows are the sealed artefact).

### The human-review step — who signs off

The rotation step 4 approval is the primary human-review checkpoint for the gold. Two role sign-offs cover the common cases:

- **Rubric owner for product-quality rubrics.** Faithfulness, helpfulness, completeness, groundedness — rubrics about the quality of the response. The rubric owner is typically the eval engineer who wrote the rubric; a senior reviewer may stand in.
- **Policy owner for safety rubrics.** Refusal, over-refusal, harmful-content detection — rubrics about what the model should decline or warn on. The policy owner is a trust-and-safety lead; the eval team does not substitute.

Two anti-patterns to avoid:

- **Eval engineer approving a safety rotation alone.** The eval team runs the queue but does not own the policy. A safety-set rotation signed only by the eval engineer is an unauthorised policy change.
- **PM approving a rubric change alone.** The rubric is an engineering artefact with downstream eval-math consequences; a PM approving it without the eval engineer's sign-off creates a rubric the automated pipeline cannot calibrate against.

### Three failure modes that recur

- **Rotation runs but no one re-runs the historical numbers.** The sealed snapshot exists; nobody checked that `mod-106 release-gate(gold_v11)` reproduces the v11 number. By the time an auditor asks, the replay cache has decayed. Mitigation: a nightly replay job that re-runs the three most recent sealed snapshots against their sealed model versions and fails loud on divergence.
- **The rolling gold grows unbounded.** Every rotation adds rows; no rotation retires rows. By year two the rolling gold has 8 000 rows, the release-gate takes six hours, and nobody can find the covered-cohort gaps. Mitigation: each rotation has a target size (per surface, per rubric); retiring rows is as much of the discipline as adding them.
- **Rows mutate in place.** Someone edits a label on a gold row without a rotation and without a snapshot. The historical number is no longer reproducible. Mitigation: gold rows are write-once; corrections are a rotation, with the old row sealed in the prior snapshot and the new row in the new rolling gold.

## Summary

- Gold sets drift because the surface changes — **four triggers**: new features / surfaces, new policy categories, model upgrades, rubric updates. Three are external events the eval team reacts to; the fourth is internal.
- **Frozen vs rolling**: the frozen gold is a sealed snapshot at a point in time, with rubric / policy / model versions attached; the rolling gold is the current working artefact. A mature program keeps both.
- **Rotation cadence** is tied to the mod-106 release calendar and the mod-107 drift monitor. Typical cadence is one rotation per release, aligned with the release.
- The **six-step rotation**: diff the surface → propose candidate rows → label with two-reviewer IAA → senior approval → seal prior rolling → publish new rolling. Downstream jobs (mod-106, mod-104, mod-107) re-run on the new gold.
- **Sealed snapshots** carry the rubric version, policy version, and model version. Re-running a historical number requires pulling the snapshot and running against the pinned model version (or the replay cache).
- **Coordination** with the mod-104 calibration set and mod-108 safety set: shared rows, distinct rotation cadences, per-artefact snapshots, write-once row discipline.
- **Approval** is by rubric owner for product-quality rubrics and by policy owner for safety rubrics. The eval engineer does not substitute for the policy owner.
- **Three common failure modes**: unreplayed snapshots, unbounded growth, in-place row mutation. Each has a standard mitigation.

Chapter 06 walks the vendor-vs-in-house decision — the one axis of this module's economics that scales the labour behind the rotation and the review workflow.
