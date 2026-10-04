# exercise-02: Model Routing Eval With Cohort Preservation

**Estimated effort:** 3 hours

## Objective

Design and evaluate a **two- or three-tier routing policy** against the chapter 03 discipline: the cohort-preservation contract, the router-error separation, and the always-frontier-when-unsure fallback. Publish the routing report in the chapter 02 shape — same four columns, same cohort matrix, same pre-registered rule — plus the two routing-specific additions (traffic distribution per layer; router-vs-model attribution).

By the end of the exercise you will have a repo-committed routing policy, a pre-registered cohort-preservation contract with per-cohort `δ_c` tolerances signed by product, a scoring harness that runs the eval set through *every* tier (not just the chosen one) so the oracle-tier attribution is computable, and a routing report that lands a ship / hold / reject decision against the pre-registered rule. The report feeds exercise-01's multi-objective report shape as a specialised row.

## Prerequisites

- Chapters 01 – 03 of this module.
- **Exercise 01** of this module. The multi-objective report shape, the `pre_registration/` discipline, and the `candidates/` lineage-key format are reused unchanged.
- A rubric from mod-104 with a stable hash and documented inter-rater agreement — the agreement number is what calibrates sensible `δ_c` values.
- An eval set from mod-106 (replay bundle) or a hand-authored fixture of ≥ 200 cases, with pre-declared cohort tags populated at case-authoring time. Routing eval is cohort-concentration-sensitive; more cases per cohort → tighter deltas.
- At least **three tiers** you can route to: a frontier model, a mid-tier or OSS model, and (recommended) a long-context specialist or an in-house distilled model. If only two tiers are available, the exercise still runs; the oracle-tier arithmetic degenerates to "better of the two per case."
- Trace instrumentation from mod-102 that writes a `routing_decision` attribute on every request span, naming the layer that chose the tier (`cache_hit`, `tier_rule[regulated]`, `classifier[mid]`, `classifier[frontier]`, `fallback_frontier`).
- Python 3.11+; `scikit-learn` or any classifier library if you train the classifier yourself; a plotting library for the per-cohort delta plot.

## Set-up

1. Create `eval/tradeoff/routing/` in your repo:

   ```
   eval/tradeoff/routing/
   ├── config.yaml
   ├── policy/
   │   ├── router.yaml                  # the routing policy under test
   │   ├── classifier/                  # optional — a trained classifier
   │   │   ├── model.pkl
   │   │   ├── train.py
   │   │   └── metrics.json             # classifier's own held-out accuracy / confusion
   │   └── rules.yaml                   # tier-based rules (regulated → frontier, etc.)
   ├── cohorts.yaml                     # reused from exercise-01, extended if needed
   ├── pre_registration/
   │   └── 2026-XX-XX-<surface>-routing.md
   ├── score/
   │   ├── run_all_tiers.py             # score every case on EVERY tier (incl. frontier)
   │   ├── apply_policy.py              # replay cases through the router; record decisions
   │   ├── oracle.py                    # post-hoc oracle tier per case (measurement only)
   │   └── router_error.py              # attribution — router-gap vs. model-gap
   ├── report/
   │   ├── build_routing_report.py      # chapter 03 shape — chapter 02 report + 2 additions
   │   └── cohort_delta_plot.py         # per-cohort Δ bar + CI
   ├── output/
   │   └── <decision_id>/
   │       ├── routing_report.md
   │       ├── traffic_distribution.csv
   │       ├── router_attribution.csv
   │       ├── cohort_delta.svg
   │       └── report.json
   ├── tests/
   │   ├── test_oracle.py
   │   ├── test_router_error.py
   │   ├── test_fallback_rate.py
   │   ├── test_cohort_contract.py
   │   └── test_report_render.py
   └── README.md
   ```

2. Define the policy in `policy/router.yaml`. A realistic shape is a chain — cache → tier-based → classifier → fallback. The policy carries its own stable `routing_policy_id` (hash of the YAML, as per chapter 03):

   ```yaml
   routing_policy_id: auto      # computed = sha256 of this file after normalisation
   name:               support-router-v3
   layers:
     - name: cache_hit
       kind: semantic_cache
       threshold: 0.92
       tier: cache
     - name: tier_rule[regulated]
       kind: rule
       predicate: tenant.compliance_tier == "regulated"
       tier: frontier
     - name: classifier
       kind: classifier
       artifact: ./classifier/model.pkl
       low_tier: mid
       high_tier: frontier
       fallback_tier: frontier
       threshold: 0.70
       low_confidence_threshold: 0.55   # below this, fall through to fallback
   tiers:
     cache:    { cost_per_req: 0.0002, model: n/a }
     mid:      { cost_per_req: 0.0059, model: claude-haiku-4-5 }
     frontier: { cost_per_req: 0.0184, model: claude-opus-4-7 }
   ```

3. Write and sign the pre-registered contract. The chapter 03 shape — per-cohort `δ_c`, baseline, decision rule, rollback trigger — committed before `score/` runs:

   ```markdown
   # Pre-registered routing contract — <surface> <decision_id>
   Written: 2026-XX-XX
   Signed:  PM=<name>, ENG=<name>

   Baseline: frontier-only on every request (candidate_id = frontier-only-<snapshot>)

   Per-cohort δ_c (rubric points, 0-5 scale):
     regulated      = 0.0   # zero tolerance
     enterprise     = 0.3
     en-US          = 0.5
     en-GB          = 0.5
     es             = 0.5
     de             = 0.5
     ja             = 0.5
     short-input    = 0.5
     long-input     = 0.5
     free-tier      = 1.0   # loosest — cost saving justifies more drop

   Ship the routing policy if ALL of:
     1. cohort-preservation contract passes on every cohort (quality(routed, c) ≥ quality(baseline, c) − δ_c)
     2. weighted mean cost / req drops ≥ 40% vs. baseline
     3. streaming p95 does not regress by more than 100 ms vs. baseline
     4. zero OWASP LLM Top-10 category regressions vs. baseline

   Hold if any clause is unmet but the gap is < 25% of the threshold.
   Reject otherwise.

   Rollback trigger:
     If mod-107 online-loop faithfulness rubric on the regulated cohort drops > 0.3 points
     over a rolling 24h window, OR fallback_rate on any critical cohort drops by more than
     20 percentage points (classifier confidence drift), auto-rollback via flag
     <flag_name> to baseline-tier for the affected cohort.
   ```

4. Train / configure the classifier (if your policy includes one). The classifier is itself a candidate — its own hash is part of the `routing_policy_id`. Train on a held-out set that is *disjoint* from the eval set; label with "which tier produced the better quality on this case" from a historical run. The classifier is tuned on the hold-out, never on the eval set — chapter 03's anti-pattern.

5. Verify the trace pipeline writes `routing_decision` on every production span. Without this attribute, the online rollback trigger has no signal; without it on a dev replay, the per-layer traffic-distribution table cannot be built. If the attribute is missing, extend mod-102's instrumentation before this exercise.

## Requirements

Produce a PR against your working branch that adds:

1. **`policy/router.yaml`** — the policy under test. Fully populated layer chain; every tier referenced names its model snapshot and current cost-per-request from the pricing snapshot. `routing_policy_id` is the hash of the normalised file.
2. **`pre_registration/<decision_id>.md`** — signed by product and engineering, committed before `score/` runs. The report generator refuses to build if the file is missing or unsigned.
3. **`score/run_all_tiers.py`** — scores every case on *every* tier (not just the one the policy selects). This is what the oracle-attribution needs. Idempotent against unchanged lineage keys; writes to the mod-110 `eval_results` table (or CSV substitute) with `tier` as an extra column.
4. **`score/apply_policy.py`** — replays every case through the routing policy and records `(case_id, chosen_tier, chosen_layer, classifier_confidence_if_applicable)`. Writes to `eval_results` with a `routing_policy_id` column.
5. **`score/oracle.py`** — for each case, computes the *oracle tier* = argmax over tiers of the rubric score, using the results from `run_all_tiers.py`. Writes `(case_id, oracle_tier, oracle_score)`. **This is a measurement device, not a shipping candidate** — the oracle uses information not available at inference time; the exercise's acceptance criteria enforce that `oracle.py` is never wired into the routing policy itself.
6. **`score/router_error.py`** — the chapter 03 attribution:
   - `route_correct[c]` — did the policy pick the same tier as the oracle for case `c`?
   - `router_attributable_gap[c]` = for router-error cases, `score(oracle_tier) − score(chosen_tier)`.
   - `model_attributable_gap[c]` = for router-correct cases, `score(baseline / frontier) − score(chosen_tier)`.
   - Aggregates per cohort. Emits `router_attribution.csv` with per-cohort router-gap and model-gap, with 95 % confidence intervals (paired-t for continuous, Wilson for pass-rates).
7. **`report/build_routing_report.py`** — the chapter 03 report shape:
   - Header block (reused shape from exercise-01).
   - **Traffic distribution** table — per layer, share of eval-set traffic, mean cost per request, mean quality score.
   - **Cohort quality Δ vs. baseline** — per cohort, Δ, `δ_c`, pass / fail. ⚠ marker on fails.
   - **Router-error separation** — per cohort, `router_attributable_gap` and `model_attributable_gap` with CIs.
   - **Fallback rate per cohort** — the chapter 03 first-class row.
   - **Decision block** — pre-registered rule quoted verbatim, each clause evaluated, ship / hold / reject per candidate.
   - **Rollback trigger** stanza.
   - Report hash in the footer; `report.json` as the machine-readable sibling.
8. **`report/cohort_delta_plot.py`** — bar plot, one bar per cohort, Δ vs. baseline with 95 % CI whisker and the `δ_c` line drawn. Cohorts failing the contract are highlighted. SVG output; embedded or linked from `routing_report.md`.
9. **A classifier evaluation sub-report** (if the policy includes one) — the classifier's own held-out confusion matrix, its drift metrics, and the per-cohort fallback rate derived from its low-confidence predictions. The chapter 03 anti-pattern "classifier as accuracy score" is countered by always presenting these numbers *alongside* the per-cohort quality deltas, never as a substitute.
10. **`tests/*`** — unit tests for:
    - `test_oracle.py` — a 10-case fixture with hand-computed oracle tiers; `oracle.py` returns the same.
    - `test_router_error.py` — a fixture where the policy mis-routes 3 of 10 cases; the attribution isolates the correct router-gap vs. model-gap numbers.
    - `test_fallback_rate.py` — a classifier that returns `low_confidence` on 30 % of the inputs; the report's fallback-rate row reads 30 %.
    - `test_cohort_contract.py` — fixtures that pass and fail per-cohort contracts; `pass?` cells match the hand-computed expected result.
    - `test_report_render.py` — fixture-driven render; stable Markdown bytes modulo timestamps.
11. **A demonstration run** on ≥ 200 eval cases, 6+ cohorts, 3 tiers. The generated report must show:
    - At least one cohort that passes the contract and one that fails (or that is close) — a routing eval where every cohort is comfortably within δ is a routing eval that is not pushing cost down enough.
    - A traffic distribution where the mid-tier carries ≥ 40 % of traffic and the fallback catches ≥ 15 % on at least one critical cohort.
    - Clear router-gap vs. model-gap attribution on at least two cohorts, with the fix implication (which to tune: the classifier or the model) spelled out in the recommendations.
12. **`README.md`** — how to declare a policy, how to pre-register, how to run `score/*`, how to generate the report, how to read the attribution. A page-long walkthrough of the demonstration run.

## Starter guidance

- **Score every tier on every case, even if it looks wasteful.** The oracle-attribution needs the full matrix. Skipping this is the single most common bug in routing evals — the per-cohort "router-attributable gap" number is uncomputable without it, and the shortcut (eval the chosen tier only) silently conflates router error and model error. If budget is tight, score a stratified sample of the eval set on all tiers plus the full set on the chosen tier; the attribution is less tight but still meaningful.
- **Pin the baseline before you tune the policy.** The chapter 03 rule — baseline is *frontier-only on every case*, not "the previous production routed system." If frontier-only is unaffordable at full eval-set scale, use the stratified-sample shortcut; the report footer names the sample size per cohort.
- **Train the classifier on a hold-out, never on the eval set.** The anti-pattern "routing tuned on the eval set" is how a routing policy wins the report and loses production. If you have an online-loop scored-row store from mod-107, that is your hold-out. If not, split the eval set 70 / 30 and keep the 30 as the hold-out; score the policy on both and report both numbers.
- **Pick `δ_c` from the rubric's inter-rater agreement.** Rubric noise on a cohort of 50 cases might be ±0.2 rubric points at 95 % confidence. Setting `δ_c = 0.1` on that cohort is noise-chasing — the report will flag drift that is not there. Setting `δ_c = 1.5` is permissive — real regressions get waved through. A sensible floor: `δ_c ≥ 2 × (rubric SE)` on the cohort's case count. Chapter 04 of mod-104 defined the SE discipline.
- **The oracle is a measurement, never a candidate.** A reviewer who sees the oracle numbers will ask "can we ship the oracle?" No — the oracle requires the correct score at inference time, which is only knowable after the fact. The report labels the oracle rows "attribution only; not a shipping candidate" to pre-empt the question.
- **Fallback rate is a drift sensor, not a budget line.** A classifier whose fallback rate on the `regulated` cohort drops from 71 % to 42 % over a month is a classifier that is drifting toward over-confidence. The report tracks the fallback rate over time; the rollback trigger uses a drop of ≥ 20 percentage points on a critical cohort as a signal.
- **The traffic-distribution table is where cost actually gets attributed.** Weighted mean cost per request is `Σ (share_layer × cost_layer)`. The report shows the share and the cost separately so readers can see which layer is doing the heavy lifting on cost — if `cache_hit` carries 22 % at near-zero cost, that is where the saving is, not the classifier.
- **Treat classifier re-training like a model swap.** When the classifier is updated, the whole routing report re-runs. The `routing_policy_id` changes; the pre-registration is re-signed; the rollback trigger watches the new fallback rate.

## Acceptance criteria

You are done when:

- The pre-registered contract is signed and committed before `score/` runs, with per-cohort `δ_c` justified against rubric inter-rater agreement in the `pre_registration/` note.
- `run_all_tiers.py` has populated `eval_results` rows for every (case × tier) pair; `apply_policy.py` has recorded the chosen tier and layer per case.
- `oracle.py` returns the expected oracle tier on the unit-test fixtures.
- `router_error.py` separates the router-attributable gap from the model-attributable gap with 95 % confidence intervals per cohort; the sum of the two gaps per case equals the total gap vs. baseline.
- The routing report shows the chapter 03 shape: traffic distribution, cohort Δ matrix with `δ_c` and pass / fail cells, router-error attribution, fallback rate per cohort, pre-registered rule evaluated verbatim, rollback trigger stanza.
- The decision verdict is reproducible — running the generator twice on the same inputs produces identical `report.json` bytes.
- The demonstration run surfaces at least one cohort near the contract boundary and shows the recommendation path (classifier re-tune vs. model-tier adjustment) based on the attribution.
- Tests pass in CI; `test_oracle.py`, `test_router_error.py`, and `test_fallback_rate.py` each include a fixture that would catch a regression in the arithmetic.
- The `README.md` walkthrough is followable end-to-end by a reader who has not read chapter 03.

## Stretch goals

- **Online-loop drift watcher.** A scheduled job reads the mod-107 online-loop scored rows for the past 7 days, computes the per-cohort quality Δ against the pre-ship baseline, and auto-fires the rollback trigger if the signal breaches. Validates the chapter 03 rollback shape in production.
- **Classifier threshold sweep.** Script a sweep over the classifier's confidence threshold (e.g., 0.50 → 0.90 in 0.05 steps); for each threshold, compute the resulting cohort-preservation outcome and the cost / latency delta. The sweep's output is a 2D plot — threshold on x, each cohort's Δ as a line — that makes the threshold's sensitivity visible.
- **Cost-aware oracle.** A secondary oracle that minimises cost subject to meeting the cohort-preservation contract per case. The gap between the cost-aware oracle and the actual policy is the *cost-efficiency gap* — a number worth tracking because it caps the realistic savings from any router iteration.
- **Multi-rubric routing.** Extend the attribution to multiple rubrics simultaneously (faithfulness, helpfulness, safety-refusal). A per-rubric cohort Δ matrix; the pre-registered rule can require pass on every rubric.
- **Routing-report diff.** Given two routing reports (previous policy vs. new policy), produce a diff that highlights which cohorts moved, which layers shifted traffic share, and what the decision would have been.
- **Explain the mis-routes.** For every router-error case, surface the input features the classifier used (SHAP values or a simpler feature-attribution). Readers see *why* the router sent a `regulated` case to mid-tier; the retraining signal is targeted.
- **Semantic-cache correctness audit.** For cache-hit cases, re-score a 10 % sample against the frontier tier and verify the cache answer's quality has not degraded (upstream data drift). Flag any cache entry whose quality has moved by more than `δ_cache`.

## What this exercise does *not* cover

You are not building the multi-objective report shape (that is exercise 01, and this exercise reuses it); the replacement-regression paired-comparison harness (exercise 03); the app-altitude latency measurement (exercise 04); or the per-feature cost accounting (exercise 05). You are shipping the *routing report with cohort preservation* — the specialised row the chapter 02 shape carries when the shipping decision is "route" rather than "swap."
