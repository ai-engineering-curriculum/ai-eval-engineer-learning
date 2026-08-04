# Pre-Registered Thresholds, First-Class Regression Blockers, and the Runbook

## Motivation

Chapter 02 gave you a replay set whose scores you can trust across CI runs. Chapter 03 turns those scores into a **decision**: is this change allowed to proceed, or is it blocked?

That decision is not an aesthetic call. If the threshold is decided *after* the eval score is known, it is not a threshold — it is a rationalisation. If the failing gate can be overridden with a click and no follow-up, the gate is decorative. If a regression fires and there is no owner and no runbook, the incident either paged the wrong person or paged nobody and rotted for a week. Product-shaped eval work runs aground on all three of these anti-patterns.

This chapter fixes the mechanics for the second of the five properties from chapter 01: **pre-registered**. You will write down thresholds in the repo, define what regressions block on and what they warn on, name the owner for each threshold, and author the runbook that a human on-call executes when the gate fires. Chapters 04 and 05 wire the mechanics into a runner; the reasoning about *what to block on* and *what to do about it* belongs here.

## Core concepts

### Threshold shapes: absolute, delta, composite, gate

A threshold is a rule that converts one or more continuous eval numbers into a pass/fail decision. Four shapes cover almost every case.

- **Absolute.** The score must sit above (or below) a fixed number. Example: `faithfulness_mean >= 0.85`. Simple; the risk is drift — the score can walk down by 0.001 per PR for months and never trip.
- **Delta vs baseline.** The score must not regress beyond a tolerance from a pinned baseline. Example: `faithfulness_mean_delta >= -0.02` vs `main`. Catches slow drift; requires a stable baseline. Chapter 05 shows how the CI job computes the baseline.
- **Composite.** A boolean combination of clauses across metrics. Example: `(faithfulness_mean >= 0.85) AND (cost_per_query_delta <= 0.05) AND (p95_latency_delta_ms <= 200)`. Necessary for surfaces where quality is not the only SLO. Every clause needs a stated rationale and a stated owner.
- **Gate on a distribution, not a mean.** Sometimes the risk is on the tail. Example: `p05_faithfulness >= 0.60` or `count(faithfulness < 0.5) <= 3`. Especially relevant for safety and refusal criteria.

Rule of thumb: **prefer the delta threshold for quality metrics on established surfaces; prefer the absolute threshold for cost/latency SLOs; use composites when you have both; use distribution thresholds when a small number of very bad outputs is worse than a small average drop**.

### Pre-registration: written down before the run

"Pre-registered" borrows from clinical-trial protocol registration. The claim is that the acceptance criterion is fixed *before* you observe the result. The mechanism is trivial and load-bearing:

- Thresholds live in a versioned file in the repo — `eval/thresholds.yaml` or the equivalent in your framework's config. Chapter 04 walks the Promptfoo and DeepEval shapes.
- The file is code-reviewed like any other. A change to a threshold is a code review, not a chat message.
- The **CODEOWNERS** file names the reviewers for the threshold file (e.g., an `eval-team` GitHub team). A prompt-change PR cannot silently loosen a threshold in the same commit.
- The CI job **fails the gate on any unregistered metric** — if the eval config emits `faithfulness_v4` but `thresholds.yaml` does not list it, the gate exits red. Chapter 04 shows the implementation; the point is that adding a metric without registering a threshold is a bug, not a feature.

The pre-registered file is the *contract*. A negotiation about whether a change ships lives in the PR review conversation on that file, not in a Slack DM after the fact.

### The threshold file shape

A minimal, portable threshold file for a RAG-triad surface:

```yaml
version: 3
last_reviewed: 2026-06-01
owners:
  faithfulness_mean: eval-team
  faithfulness_p05: eval-team
  context_precision_mean: retrieval-team
  cost_per_query_delta: platform-cost
  p95_latency_delta_ms: platform-cost
  refusal_rate: safety-team

metrics:
  faithfulness_mean:
    rubric: faithfulness_v3
    aggregation: mean
    thresholds:
      absolute: {min: 0.85, severity: block}
      delta_vs_main: {min: -0.02, severity: block}
  faithfulness_p05:
    rubric: faithfulness_v3
    aggregation: quantile
    quantile: 0.05
    thresholds:
      absolute: {min: 0.60, severity: block}
  context_precision_mean:
    rubric: context_precision_v2
    aggregation: mean
    thresholds:
      delta_vs_main: {min: -0.03, severity: warn}
  cost_per_query_delta:
    aggregation: mean_delta_ratio
    thresholds:
      absolute: {max: 0.05, severity: block, rationale: "5% cost SLO"}
  refusal_rate:
    aggregation: fraction
    thresholds:
      delta_vs_main: {max: 0.02, severity: block, rationale: "over-refusal SLA"}
```

Four properties worth highlighting:

- **Every metric has an owner.** No orphan thresholds; the owner is the on-call for the gate firing.
- **Every threshold has a severity.** `block` fails the gate; `warn` posts to the PR report but does not fail. Chapter 05's report shape treats them differently. `warn` is the escape hatch for a threshold you are still calibrating — but you time-box its life (see below).
- **Every non-obvious threshold has a rationale.** `0.85` is not obvious; `5% cost SLO` is. Rationales are what let a future engineer decide whether to preserve or revise the threshold under new evidence.
- **The file is versioned.** A threshold change ticks `version`. Chapter 05's PR gate compares the file version against `main`; a change flagged for owner review.

### `warn` vs `block`: don't let `warn` become forever

Every gate has a temptation to leave everything at `warn` when a threshold is new or noisy. That is fine for a week; it is not fine forever. Two rules that keep the discipline:

- **A new threshold ships at `warn` with a mandatory `block_by: <date>` field.** The CI job reads `block_by`; on the date, if the severity is still `warn`, the gate itself fails, forcing a promote-or-revise decision. This is uncomfortable and correct — the alternative is that everything sits at `warn` and blocks nothing.
- **The percent of metrics at `warn` is a monitored SLI on the eval program itself.** More than 25% at `warn` is a smell — chapter 03 of mod-112 (program-owner posture) treats this as a program health metric.

### First-class regression blocker semantics

When the gate fires on the PR, the failure has to behave the same way as any other CI failure the team already respects. Concretely:

- The GitHub check is `required` (branch protection rule). Merge is blocked without a maintainer override.
- The failure reason appears on the PR — chapter 05's report format walks this. The reviewer does not have to click through to a vendor UI to see which metric on which fixture regressed by how much.
- The override path exists, but it costs something: a documented rationale, an auto-filed follow-up issue owned by a named person, and a note that shows up on the eval-program dashboard. Chapter 06 covers the same pattern at the deploy gate.

The single most common mistake is shipping the gate as advisory-only. It never blocks; nobody reads the report; the eval work quietly ossifies. **Flip to blocking in the same PR that adds the gate — or, if that is politically impossible, ship the flip as a scheduled follow-up with a named date and named owner in the initial PR description.** Advisory-only gates are dead on arrival.

### Regressions need owners, not authors

When a regression fires, the person who has to act is not necessarily the person who wrote the PR. Two failure modes:

- The PR author sees the failure, does not know what the threshold means, and pings a channel. Nobody in the channel owns the metric; the message rots.
- The PR author suppresses the failure locally and re-runs. This is the failure mode the audit trail prevents.

The runbook pattern below is the mechanism that turns a fired threshold into an assigned task with a stated first step.

### The runbook: what to write, where it lives

Every metric in the threshold file has (or points to) a runbook. A minimum runbook has five parts:

1. **What the metric measures**, in one sentence a new engineer understands. Pointers to the mod-101 criteria row and the mod-104 rubric.
2. **Why the threshold is what it is.** Pin the reasoning. If it is `p05 >= 0.60`, why 0.60 and not 0.50? Cite the evidence — a mod-104 calibration table, a mod-107 online regression, a stakeholder ask.
3. **First step when it fires.** The concrete command or query the on-call runs to see the failing fixtures — a link to the CI artefact, a `promptfoo view` command, a Braintrust URL. Not "investigate"; "run `X`, look at `Y`".
4. **Common causes.** The three or four things that historically cause this metric to move (prompt change, retrieval index rebuild, model snapshot, judge model change). One-line diagnostics per cause.
5. **Escalation.** Who to page if the on-call cannot resolve within N hours. This is where the owner alias in the threshold file appears again as a real human on-call rotation.

Runbooks live in the repo (`eval/runbooks/faithfulness_mean.md`, one per metric). The PR-gate report links to them by URL. If a runbook is missing when the gate fires, the CI check itself flags that as a gate config bug — no metric ships without a runbook.

### A worked runbook

```markdown
# Runbook: faithfulness_mean

Metric owner: eval-team (@eval-oncall)
Rubric: faithfulness_v3 (mod-104 chapter 01)
Threshold: absolute >= 0.85, delta_vs_main >= -0.02 (both blocking)

## What it measures
Mean judge-scored faithfulness of the reply to the retrieved context on the RAG replay set (150 fixtures). Rubric detail in `eval/rubrics/faithfulness_v3.md`; calibration in `eval/calibration/faithfulness_v3-2026-03-14.md`.

## Why 0.85 / -0.02
0.85 is the P25 of the top-quartile release from the mod-104 calibration set. -0.02 is one standard error of the mean on the current 150-item replay set (bootstrap CI in `eval/calibration/faithfulness_v3-2026-03-14.md`).

## First step
1. Open the CI artefact `pr-eval-report.md` posted on the PR.
2. Sort fixtures by delta vs main; the top five regressions are the bisection targets.
3. For each: open the fixture, replay locally against the PR branch with `promptfoo eval -c eval/promptfoo.yaml --filter-fixtures <fixture-id>`.

## Common causes
- **Prompt change.** The reply prompt now under-emphasises "use only the retrieved documents". Diagnostic: diff `prompts/*.md` and re-run against the pinned model.
- **Retrieval index rebuild.** The index hash in the fixture no longer matches. Diagnostic: `python eval/verify_index_hash.py`; if hash mismatch, either refresh fixture (with review) or restore the pinned index snapshot.
- **Model snapshot rolled.** `system_fingerprint` differs from baseline. Diagnostic: check the CI header; note the pin against `mod-102`'s `gen_ai.request.model` on the trace. Escalation on this cause: file with the platform team to pin the previous snapshot until re-calibration lands.
- **Judge model version bump.** Rare, but happens when the frontier tier is upgraded. Diagnostic: compare `judge_model` field on `pr-eval-report.md` against `main`. If changed, refer to mod-104 chapter 05's frontier-upgrade playbook.

## Escalation
If unresolved in 24h, page @eval-oncall via PagerDuty rotation `EVAL-1`. If the root cause is a retrieval bug, hand off to @retrieval-team per their runbook `RET-3`.
```

The document reads like an on-call runbook because it *is* an on-call runbook. It does not require the reader to have written the code; it requires them to be able to run the commands and follow the escalation.

### CODEOWNERS and the review contract

The threshold file, the runbook files, the rubric files, and the fixtures are all governed by CODEOWNERS. A minimal `.github/CODEOWNERS`:

```
/eval/thresholds.yaml    @your-org/eval-team
/eval/rubrics/           @your-org/eval-team
/eval/runbooks/          @your-org/eval-team @your-org/on-call
/eval/fixtures/          @your-org/eval-team
/eval/calibration/       @your-org/eval-team @your-org/model-eval
```

Two consequences:

- A PR that loosens a threshold requires an eval-team review — the review shows up on the PR check UI.
- A PR that adds a fixture requires an eval-team review — the reviewer confirms that the fixture is derived from a real trace with a valid `source_trace_id`, has proper `reference` material, and did not silently promote `recorded_output` to `reference`.

CODEOWNERS is the mechanism that makes the pre-registration contract real. Without it, the file is a suggestion.

### The four things that must be true for the gate to fail loudly

Rolled up: when a threshold fires, the gate has to behave the same way a failing test does. The four things:

1. The **PR check is red** (branch-protected, override requires a documented reason on the PR).
2. The **report on the PR** names the failing metric, the fixture(s), the delta, and links to the runbook.
3. The **runbook** is up to date and executable by an engineer who did not write the code.
4. The **owner** in the threshold file resolves to a real on-call rotation.

Miss any one and the gate is decorative.

## Summary

- Thresholds come in four shapes: absolute, delta vs baseline, composite, distribution/tail. Pick the shape by risk profile — delta for quality, absolute for cost/latency, distribution for safety.
- **Pre-registered** means the threshold file is in the repo, code-reviewed via CODEOWNERS, versioned, and the CI job fails on an unregistered metric.
- Every metric has a severity (`block` or `warn`), an owner, and — where not obvious — a stated rationale. `warn` is time-boxed with a mandatory promote-or-revise date.
- Regressions block merge. Overrides exist but cost — a documented rationale, an auto-filed issue, a program-dashboard entry. Do not ship a gate in advisory-only mode without a scheduled flip-to-blocking.
- Every metric has a runbook: what it measures, why the threshold, first step, common causes, escalation. Missing runbook = gate config bug.
- CODEOWNERS makes the pre-registration contract enforceable: threshold, rubric, runbook, and fixture changes require eval-team review.

Chapter 04 walks the declarative eval-config layer — Promptfoo YAML, DeepEval pytest, and the Braintrust / Weave / Langfuse CI runners — that reads this threshold file and produces the scores it gates on.
