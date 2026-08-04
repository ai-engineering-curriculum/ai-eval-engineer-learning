# exercise-04: Noise Sensitivity and Context Utilisation

**Estimated effort:** 2 hours

## Objective

Wire the **four diagnostic metrics** from chapter 04 next to the triad from exercise-01 — noise sensitivity, context utilisation, answer completeness, and refusal quality — build a per-slice × per-metric dashboard from the exercise-03 gold set, and prove the diagnostics catch a **silent regression** the triad aggregates miss. This is the final piece that makes the RAG eval suite paging-worthy for mod-107's online-eval loop.

By the end you will have the four diagnostics running against your gold set, a per-slice × per-metric report that includes both the triad and the diagnostics, and a seeded silent regression that fires only on a diagnostic (not on the triad) — the demonstration a program owner (mod-112) can point to when explaining why the diagnostics exist.

## Prerequisites

- Chapter 04 of this module.
- Exercises 01, 02, and 03 completed — RAG-triad in RAGAS/TruLens, attribution CLI, product-shaped gold set with slice tags.
- The RAGAS install from exercise-01 (the four diagnostics slot cleanly into RAGAS; if you kept TruLens as the primary framework, you still run RAGAS for noise sensitivity and answer completeness which TruLens does not ship natively).
- A live retriever + generator loop you can shim to inject distractors on demand (for the silent-regression seeding).

## Requirements

Produce a single directory `mod-105/exercise-04/` in your working repo containing:

1. **`diagnostics.py`** — implements the four diagnostics against the exercise-03 gold set:
   - `NoiseSensitivity` from RAGAS (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/noise_sensitivity/>) on gold-set items. Runs on `distractor_heavy` and on the overall gold set separately.
   - `ContextUtilization` from RAGAS (<https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/#context-utilization>) on all items with retrieval (no reference answer required — this is the one diagnostic viable online).
   - `ResponseCompleteness` (or the equivalent claim-recall metric per current RAGAS docs) on multi-hop / multi-chunk items.
   - `refusal_quality` — a mod-104-style absolute rubric returning `{correct_refusal, false_answer, false_refusal, correct_answer}` on out-of-scope items only. Author the rubric per mod-104/chapter-01 and score with your exercise-01 framework choice.
   Emit per-item scores to `results-diagnostics.csv`.
2. **`dashboard.md`** — the per-slice × per-metric dashboard, in the shape mod-107 will alert against:
   - Rows are the eight metrics (four triad + four diagnostics).
   - Columns are the slices from the exercise-03 gold set (`in_scope_single_chunk`, `in_scope_multi_chunk`, `distractor_heavy`, `out_of_scope`, `hallucination_tempting`, plus any surface-specific slices you added).
   - Each cell: `<mean> (n=<denominator>)` and a per-slice threshold you would page on.
   - Below the table, a legend column showing which metric applies to which slice and where `not_applicable` is expected.
3. **`silent-regression/`** — the seeded silent-regression demo:
   - `seed.md` — a description of the regression you seeded (e.g., a retriever change that adds one extra plausible-looking distractor to the top-k for `distractor_heavy` items — an over-fetching change). Include the diff and the mechanism.
   - `before.csv` and `after.csv` — per-item scores for all eight metrics before and after the seeded change.
   - `regression-report.md` — 300–500 words. Which metrics moved and how much. Show the triad aggregates are within noise-floor. Show the noise-sensitivity or context-utilisation metric on the affected slice moved by a paging-worthy amount. Argue that a triad-only dashboard would have missed this regression until it grew.
4. **`alerting-policy.md`** — 200–300 words. The alerting policy for the diagnostics:
   - Per-metric, per-slice threshold with holdover (e.g., "noise sensitivity on `distractor_heavy` > 0.15 for 3 consecutive daily runs").
   - Which team gets paged (this module vs `rag-engineer` peer) per chapter 05's contract, per metric.
   - The paging severity for a diagnostic-only regression vs a triad-and-diagnostic regression.

## Starter guidance

- **Noise sensitivity requires a reference answer.** It is offline-only. Do not try to add it to the online-eval loop — chapter 04 covers this. Run it against the gold set nightly (mod-107 nightly job shape).
- **Context utilisation is your one online-viable diagnostic.** Wire it against sampled production traffic (if available) with the same threshold discipline as faithfulness. If you don't have live traffic, at least document the plan.
- **Refusal quality's rubric is where the module's mod-104 discipline pays back most.** Anchor the four labels with observable properties; hand-test on 10 out-of-scope items before you rely on the rubric. See mod-104/exercise-01.
- **Seed a *silent* regression, not a loud one.** The demonstration is that the triad aggregate is *within noise floor* while a diagnostic moves. If your seeded change breaks the triad aggregate too, either the change is too aggressive or your gold set is too small to distinguish the two — pick a smaller nudge.
- **Include denominators everywhere.** A mean-with-no-denominator dashboard is a dashboard that lies about noise-floor. Every cell in `dashboard.md` shows `n=`.

## Acceptance criteria

You are done when:

- All four diagnostic metrics run against the exercise-03 gold set and produce per-item scores.
- `dashboard.md` shows the 8-row × N-slice table with per-slice thresholds and clear `not_applicable` cells (noise sensitivity on out-of-scope, refusal quality on in-scope, etc.).
- The seeded silent regression demonstrably moves at least one diagnostic on the affected slice by a paging-worthy delta while leaving the triad aggregate within noise floor on the affected slice.
- `regression-report.md` argues the point cleanly with numbers and denominators — a stakeholder unfamiliar with the module could read it and understand why the diagnostics exist.
- `alerting-policy.md` names the per-metric, per-slice thresholds and holdovers and the paging assignment per chapter 05.
- The rubric for refusal quality is version-controlled with its prompt hash and follows mod-104/chapter-01 shape.

## Stretch goals

- **Add a self-consistency check.** Rerun `NoiseSensitivity` (or `ContextUtilization`) twice at temperature 0 and once at temperature 0.3; report the per-item variance as your metric noise floor. Use this to justify the paging thresholds in `alerting-policy.md`.
- **Add a distractor-injection stress test.** For a subset of gold items, deliberately inject a plausible distractor into the retrieved context (bypass the retriever) and re-score. A robust generator's reply barely changes; a fragile generator shifts. Report the fragile fraction as a per-slice diagnostic alongside the RAGAS noise sensitivity.
- **Correlate the diagnostics to the triad.** Compute the per-item correlation between (noise sensitivity, context utilisation, completeness, refusal quality) and (faithfulness, answer relevancy). If a diagnostic is perfectly correlated with a triad metric on your data, it's not adding paging value — drop it or replace it with a diagnostic that isn't. If it is *decorrelated*, that is exactly where the paging value lives.
- **Wire the dashboard to a real observability backend.** If your team runs Grafana / Datadog / your trace-backend's dashboard, publish the per-slice × per-metric report there rather than in a markdown file. Confirm the alert fires on the seeded regression.

## What this exercise does *not* cover

You are not calibrating the diagnostics against a human gold set (that's mod-104/chapter-04 applied to these rubrics — do it separately if you own the diagnostic rubric long-term). You are not wiring the CI gate on the diagnostics (mod-106). You are not tuning the retriever to reduce the distractor rate (`rag-engineer` peer's work; chapter 05). Stay narrow: four diagnostics, one dashboard, one silent regression, one alerting policy.
