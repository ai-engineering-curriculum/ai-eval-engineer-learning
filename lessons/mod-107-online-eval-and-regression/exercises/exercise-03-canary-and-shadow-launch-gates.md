# exercise-03: Canary and Shadow Launch Gates

**Estimated effort:** 3 hours

## Objective

Build the **shadow → canary → progressive rollout** gate on top of the mod-106 deploy-gate contract. You will ship three scripts — `shadow.sh`, `canary.sh`, and `rollout-stage.sh` — each satisfying the chapter 04 JSON schema, each consuming the scored-row store from exercise-01, each applying the per-cohort thresholds from chapter 04, and each rehearsed against a real (or simulated) rollback flow.

By the end you will have the interface the platform team's rollout controller (Argo Rollouts `AnalysisTemplate`, Flagger `MetricTemplate`, Spinnaker canary analysis, LaunchDarkly experiment, or an in-house equivalent) reads to decide whether to progress or halt.

## Prerequisites

- Chapter 04 of this module.
- The exercise-01 scored-row store, populated by the loop against traces from at least two flag-variant cohorts (e.g., `stable` and one experimental variant). If the flag-variant plumbing is not in place, simulate — the exercise's shape does not require a real deploy.
- The mod-106 chapter 06 deploy-gate directory (`eval/deploy-gates/offline.sh` etc.). If mod-106 exercise-05 is not done, stub `offline.sh` with a passing exit code.
- A feature-flag system, real or stubbed (`launchdarkly`, `openfeature`, `statsig`, or a JSON file the app reads on each request).
- Optionally: a Kubernetes cluster with Argo Rollouts, a Flagger install, or Spinnaker; the interface is portable and can be demonstrated with `bash` alone if the cluster is not available.

## Set-up

1. Extend `eval/deploy-gates/` with the three new scripts:

   ```
   eval/deploy-gates/
   ├── offline.sh          # from mod-106 exercise-05
   ├── shadow.sh           # new
   ├── canary.sh           # new
   ├── rollout-stage.sh    # new
   ├── gate_lib.py         # shared logic (score fetch, threshold, JSON output)
   ├── thresholds.yaml     # extends mod-106 thresholds with per-cohort clauses
   └── rehearsal/
       ├── shadow_ok.sh
       ├── canary_regression.sh
       └── rollback_run.sh
   ```

2. Extend `thresholds.yaml` with per-cohort deltas per chapter 04:

   ```yaml
   metrics:
     faithfulness:
       absolute_floor: 0.80
       delta_vs_reference: -0.02
       per_cohort_delta_vs_reference: -0.05
       severity: block-and-page
       owner: eval-team
       runbook: runbooks/faithfulness.md
     safety_refusal_correctness:
       absolute_floor: 0.98
       delta_vs_reference: -0.01
       per_cohort_delta_vs_reference: -0.02
       severity: block-and-auto-rollback
       owner: safety-team
       runbook: runbooks/safety_refusal.md
       cool_down_seconds: 0
     latency_e2e_p95_ms:
       absolute_ceiling: 4000
       delta_vs_reference: +200
       per_cohort_delta_vs_reference: +400
       severity: block-and-page
       owner: eval-team
       runbook: runbooks/latency_e2e.md
     cost_per_query_usd:
       absolute_ceiling: 0.05
       delta_vs_reference: +0.005
       per_cohort_delta_vs_reference: +0.010
       severity: warn
       owner: eval-team
       runbook: runbooks/cost_per_query.md
   ```

3. In `gate_lib.py`, implement the shared functions: `fetch_scores(cohort_keys, window)`, `apply_thresholds(candidate, reference, thresholds)`, `emit_json(result, out_path)`. The output JSON must match the chapter 04 schema (verdict, per-metric, cohort attribution, runbook links).

## Requirements

Produce a PR against your working branch that adds:

1. **`shadow.sh`** — takes `--candidate-cohort`, `--reference-cohort`, `--window-seconds`, `--threshold-file`, `--out`. Reads scored rows for the shadow-served candidate cohort (rows whose `cohort_keys.flag_variant == "shadow"` or equivalent). Applies thresholds. Emits JSON per chapter 04's schema. Exits `0` on `PASS`, `1` on `FAIL`, `2` on `INSUFFICIENT_DATA`.
2. **`canary.sh`** — same interface as `shadow.sh`; reads live-slice-served scored rows (rows whose `cohort_keys.flag_variant == "canary"` or equivalent) and applies the same thresholds; emits the same JSON. Exit codes as above.
3. **`rollout-stage.sh`** — takes `--stage-cohort` (the newly-included cohort for this rollout stage), `--reference-cohort`, `--window-seconds`, `--threshold-file`, `--out`. Same behaviour. Reads only rows from the newly-included cohort (not the cumulative served cohort — chapter 04's per-stage-cohort rule).
4. **Per-cohort attribution.** All three scripts run the threshold check per pre-declared cohort (locale × tenant tier × product surface at minimum). Any one cohort's `FAIL` fails the aggregate; the JSON's `failures[]` names every failing cohort × metric pair.
5. **Effective-sample-size handling.** Weighted sample counts (from chapter 02's `sample_weight`) drive the CI width. Below the per-metric minimum-effective-`n`, the metric returns `INSUFFICIENT_DATA`; aggregate returns `INSUFFICIENT_DATA` if any ship-critical metric is under-covered.
6. **Rehearsal scripts** — three end-to-end demonstrations:
   - `rehearsal/shadow_ok.sh` — synthesises a shadow window where the candidate scores are within tolerance; confirms `shadow.sh` exits `0` and the JSON `verdict` is `PASS`.
   - `rehearsal/canary_regression.sh` — synthesises a canary window with a −0.05 faithfulness delta on a single locale cohort; confirms `canary.sh` exits `1`, the JSON names the failing cohort × metric, and (if configured with `block-and-auto-rollback`) the rehearsal script fires a flag-flip stub.
   - `rehearsal/rollback_run.sh` — end-to-end: gate fires → runbook link opened → rollback command executed → flag flipped off → next `canary.sh` window returns to `PASS`. Time from gate fire to flag flip is recorded; the whole flow completes in under 5 minutes.
7. **Runbook per metric** — extends mod-106's runbook shape with the deploy-pipeline specifics: which pipeline stage fired, how to trigger a rollback manually, how to re-promote after fix, which rollback shape (code-only / model-version / retrieval-index / **feature-flag flip** — chapter 04's fourth shape).
8. **README addition** — a short "deploy gate" section in `eval/deploy-gates/README.md` explaining the three stages, the JSON schema, the rehearsal scripts, and the interface to the platform team's rollout controller.

## Starter guidance

- **Do not aggregate away a locale regression.** Per-cohort gates are the whole point of chapter 04. A candidate whose aggregate faithfulness is unchanged but whose Japanese-locale faithfulness dropped 15 points is not a passing gate.
- **Cohort comparability.** The candidate and reference cohorts must differ only on the axis the deploy changes. When they do not (e.g., the canary picked up a different locale distribution because of a routing quirk), the correct output is `INSUFFICIENT_DATA` with a `reason`, not a false `PASS`.
- **Do not skip stages.** Chapter 04 is explicit: a candidate that passed the 5 % stage is not automatically going to pass the 25 % stage. The rollout-stage gate re-runs against the newly-included cohort; the platform team's controller is what advances.
- **Feature-flag rollback is often the right shape.** If the change is behind a flag, the runbook's step 1 is "flip the flag off" — one command, sub-second effect. Do not default to redeploy when a flag exists.
- **Rehearse the rollback in the exercise.** A gate that has never been used to roll back is a gate that will fail in production. The `rollback_run.sh` script is not a formality; it is proof the whole loop works.
- **Confidence intervals as first-class output.** Chapter 04 requires the JSON to report the delta *and* the CI. A gate that returns only a point estimate is fine on paper and unusable in incidents.
- **Safety metrics: no soft-fail.** The `safety_refusal_correctness` clause is `block-and-auto-rollback`; a candidate that fails it does not fall through to a `warn`.

## Acceptance criteria

You are done when:

- The three scripts satisfy the chapter 04 JSON schema (`verdict`, per-metric with CI, cohort attribution, `failures[]`, `runbook_urls[]`, `n_effective_weighted`, `window`).
- Each script exits `0 / 1 / 2` for `PASS / FAIL / INSUFFICIENT_DATA` consistently and reproducibly.
- Per-cohort attribution is enforced: a rehearsal with a single-cohort regression on faithfulness fails the aggregate and names the exact cohort.
- `rehearsal/canary_regression.sh` fires the gate; the runbook link opens; `rehearsal/rollback_run.sh` executes end-to-end in under 5 minutes; the subsequent `canary.sh` window returns `PASS`.
- The runbook for at least the demonstrated regression names the rollback shape (feature-flag flip) and the one-line rollback command.
- The gate is idempotent on the same inputs (same JSON out on a re-run).
- Configuring `severity: block-and-auto-rollback` on a metric wires the rollback trigger (real or stubbed) automatically; `severity: block-and-page` requires an explicit human decision to progress.

## Stretch goals

- **Wire against Argo Rollouts.** Ship a `Rollout` + `AnalysisTemplate` YAML that consumes `canary.sh`'s JSON via a Prometheus adapter or a webhook provider; run a full canary → progressive rollout against a demo service in a cluster; confirm the controller halts on a `FAIL`.
- **Wire against Flagger.** Alternative to Argo: a `Canary` custom resource with a `MetricTemplate` that reads `canary.sh`'s exit code. Confirm the same rollout / halt behaviour.
- **Wire against a LaunchDarkly experiment.** The flag-driven variant: no traffic-splitting at the mesh level; the flag is the canary knob and the rollback knob. Confirm the "flip flag on gate fire" path fires within one polling interval.
- **Sequential-mode canary.** Replace the per-metric fixed-`n` test with the exercise-04 confidence-sequence library; confirm the canary reaches a decision faster on a strong-signal regression and equally quickly on a null.
- **Cohort-comparability check.** Add a pre-flight sanity check that fails-fast when candidate and reference cohort distributions differ on axes not intended to differ; produce a "cohort-mismatch" reason in the JSON.
- **Rollback rehearsal as scheduled job.** Once a month, a scheduled job runs `rollback_run.sh` on a synthesised regression against a non-production cohort. Confirms the rollback command still works even when nothing has actually gone wrong recently — the same rehearsal cadence mod-106 chapter 06 named.

## What this exercise does *not* cover

You are not implementing the confidence sequence (exercise-04), the A/B interface (exercise-05), or the dashboards (exercise-06). You are shipping the gate that reads exercise-01's store and follows the chapter 04 rules; the sequential statistics live in the next exercise; the A/B experiment interface lives after that; the dashboards close the module.
