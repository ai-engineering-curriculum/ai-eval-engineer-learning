# mod-107 — Online Evaluation and Regression Detection at the App Layer

**Estimated effort:** 16 hours

This is the seventh module of the AI Evaluation Engineer track. It builds the *online half* of the eval program — the always-on loop that scores a sample of live production traces, detects drift at input / output / judge levels, gates canary and shadow launches, monitors continuously with anytime-valid statistics, interfaces with the A/B platform, and presents dashboards a product manager, an on-call engineer, and a safety reviewer can each read at a glance.

Mod-106 closed with the change fully deployed and an online loop watching; that final sentence is the entire premise of this module. The offline gate catches change-induced regressions before merge; the online loop catches drift-induced regressions and any regression that only surfaces at scale. Both matter; this track defends both.

The module deliberately does not re-derive experimental methodology (owned by the peer `model-evaluation-engineer` at the same level) or the safety rubrics themselves (owned by mod-108). It walks the mechanism — sampler, drift monitor, cohort gate, confidence sequence, pre-registration contract, dashboards — and names the interface where the peer takes over.

## Learning objectives

- Design a sampled-trace online-eval loop that scores live traffic on quality / safety / cost / latency without breaking the budget.
- Detect drift at input (query distribution shift), output (response distribution shift), and judge level (model-judge upgrade drift, judge-prompt drift).
- Wire canary and shadow launches with per-cohort quality and safety gates.
- Apply sequential testing / confidence sequences for continuous monitoring so the alert fires when it should and not before.
- Interface with the A/B platform: know the CUPED and pre-registration contract, and know where the deep experimental methodology (owned by `model-evaluation-engineer` / `senior-ml-engineer`) begins.
- Wire dashboards in Arize Phoenix, Langfuse, W&B Weave, or Braintrust that a product manager, an on-call engineer, and a safety reviewer can each read.

## Lecture chapters

1. [Why the online-eval loop](01-why-online-eval-loop.md) — the offline gate / online loop distinction; the loop as a four-component pipeline; six terms the module reuses (**sampled-trace scoring**, **drift**, **cohort**, **sequential test / confidence sequence**, **pre-registration contract**, **CUPED**); three time scales (canary, drift monitor, A/B experiment).
2. [The sampled-trace online-eval loop](02-sampled-trace-online-eval-loop.md) — sampler (uniform / stratified / hash / bias-toward-interesting); tier router with runtime budget guard; judge runner and cache; aggregator with per-cohort minimum-sample floor; budget arithmetic worked end-to-end; backpressure and failure modes; the load-bearing scored-row schema.
3. [Drift detection: input, output, and judge](03-drift-input-output-judge.md) — fixed vs rolling reference; three drift sources with three runbooks; test-choice matrix (PSI, KS, MMD, chi-squared, CUSUM); judge-model-upgrade / judge-prompt / judge-tier-composition drift; multiple-testing correction; pre-declared vs segment-discovered slicing.
4. [Canary and shadow launches with per-cohort gates](04-canary-shadow-cohort-gates.md) — shadow-then-live-slice sequence; the mod-106-compatible gate script and JSON schema; cohort-key discipline; per-cohort delta vs absolute floor; safety carve-out (no soft-fail); progressive-rollout stage gates; rollback shapes including feature-flag flip.
5. [Sequential testing and confidence sequences](05-sequential-testing-confidence-sequences.md) — why peeking on fixed-`n` tests fails; alpha-spending vs anytime-valid inference; mSPRT / GLR / confidence sequences (Hoeffding, Empirical-Bernstein, betting-style); the `eval_delta` interface both drift monitor and canary gate consume; multiple-testing correction across metrics × cohorts.
6. [The A/B platform interface and CUPED contract](06-ab-platform-interface-cuped.md) — the eval program owns metric definitions and variance-reduction covariates, not experimental design; pre-registration contract; the CUPED covariate as a rubric-on-same-unit-in-pre-window; SRM and cohort-sanity checks; the delegation boundary to `model-evaluation-engineer` / `senior-ml-engineer`; sequential-mode experiments.
7. [Cross-persona dashboards](07-cross-persona-dashboards.md) — PM view (aggregate health, experiment ledger); on-call view (active alerts + rollback bank + live strip); safety-reviewer view (cohort × metric matrix, suppression register, injection-rate trend); vendor mapping (Phoenix / Langfuse / Weave / Braintrust) and general-purpose Grafana alternative; alert channels matched to persona; the eval team's own loop-health dashboard.

## Exercises

Each exercise builds on the last. Do them in order and keep the outputs — the artefacts (sampled-loop workers, drift-alert configs, canary-gate script, confidence-sequence library, A/B pre-registration bundle, three cross-persona dashboards) are the deliverables mod-108 – mod-112 will assume you have.

1. [Sampled-trace online-eval loop](exercises/exercise-01-sampled-trace-online-eval-loop.md) — stand up the four-component loop against your mod-102 backend: sampler with stratification + hashing + bias, tier router with budget guard, judge runner with caching, aggregator emitting the scored-row schema.
2. [Input, output, and judge drift alerts](exercises/exercise-02-input-output-and-judge-drift-alerts.md) — wire PSI / KS / MMD monitors against a fixed and a rolling reference; run the judge-calibration daily job; apply Benjamini–Hochberg across the metric × cohort × window family.
3. [Canary and shadow launch gates](exercises/exercise-03-canary-and-shadow-launch-gates.md) — write `shadow.sh`, `canary.sh`, and `rollout-stage.sh` consuming the chapter 04 JSON schema; wire per-cohort thresholds; rehearse the flag-flip rollback.
4. [Confidence sequence for continuous monitoring](exercises/exercise-04-confidence-sequence-for-continuous-monitoring.md) — implement `eval_delta` with an Empirical-Bernstein CS on rubric scores and a Hoeffding CS on rate metrics; simulate the peeking-inflation contrast against a naive t-test; ship the library.
5. [A/B platform interface and CUPED contract](exercises/exercise-05-ab-platform-interface-and-cuped-contract.md) — file a full pre-registration bundle (metric definition, CUPED covariate, SRM and cohort-sanity checks) against your chosen A/B platform for a real or simulated experiment.
6. [Cross-persona dashboards](exercises/exercise-06-cross-persona-dashboards.md) — build the three dashboards on top of the chapter 02 scored-row store; wire the digest / PagerDuty / weekly-review channels; do the walk-through with a proxy for each persona.

## Labs and quizzes

- Labs (see [`labs/`](labs)) build an end-to-end online-eval reference on top of the mod-102 instrumented backend and the mod-106 gate pipeline: the four-component loop, the drift-alert suite, the canary + shadow gate, the CS library, the pre-registration bundle, and the three dashboards. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary — the six terms, the three time scales, the four sampling strategies, the three drift levels, the shadow-vs-live-slice distinction, the peeking-vs-anytime-valid distinction, the CUPED variance-reduction factor, and the three-persona dashboard split. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- **Safety and guardrail eval** reuses the loop, the drift monitors, and the canary gate with stricter thresholds and zero-tolerance safety floors → [`mod-108-app-safety-and-guardrails-eval`](../mod-108-app-safety-and-guardrails-eval).
- **Human review workflows** consume alerts that the on-call runbook cannot resolve as review-queue items; the safety-reviewer dashboard cross-references this queue → [`mod-109-human-review-workflows`](../mod-109-human-review-workflows).
- **Eval-data platform** stores the chapter 02 scored-row schema, the reference-distribution snapshots, the pre-registration bundle, and the alert-history as first-class tables → [`mod-110-eval-data-platform-slice`](../mod-110-eval-data-platform-slice).
- **Cost / latency / quality trade-off** reads the same scored-row store for per-request cost and latency numbers and shares the aggregator with the chapter 07 PM dashboard → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- **Program-owner posture** treats this module's sampler, thresholds, runbooks, dashboards, and A/B interface as the load-bearing artefacts of the release-gate-plus-online-monitor architecture, and defends them to the Governance-family peer → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
