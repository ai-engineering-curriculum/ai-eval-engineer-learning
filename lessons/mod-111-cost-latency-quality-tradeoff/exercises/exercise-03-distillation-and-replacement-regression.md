# exercise-03: Distillation and Replacement Regression

**Estimated effort:** 3 hours

## Objective

Run a **paired replacement regression** on a candidate model — a vendor-snapshot bump, a mid-tier substitute, a distilled student, or an open-source replacement — using the chapter 04 methodology. Freeze the eval set; re-score the incumbent under the current rubric and judge snapshot; score the candidate on the same inputs; compute per-case deltas with confidence intervals; aggregate per cohort; evaluate against a pre-registered contract; land one of **ship / ship with routing / hold / reject** against the chapter 04 decision matrix.

By the end of the exercise you will have a delta report in the chapter 02 shape (header, main table, cohort matrix, decision block, rollback trigger) extended with chapter 04 specifics — paired-Δ histograms, regression-bucket counts with FDR-corrected pass tests, and (if the candidate is a distilled student) the coverage-and-quality plot plus the teacher-agreement rate. The report lands a reproducible shipping decision tied to a mod-107 online-loop rollback trigger.

## Prerequisites

- Chapters 01, 02, and 04 of this module.
- **Exercise 01** of this module. The multi-objective report shape, `pre_registration/` discipline, and `candidates/` lineage-key format are reused unchanged.
- A frozen eval set with a stable hash (mod-106 replay bundle, or exercise-01's fixture set). ≥ 300 cases recommended so the per-cohort power is enough for a paired-t to reject at `α = 0.05` after FDR correction across 6 – 10 cohorts.
- A rubric from mod-104 with a stable hash. The incumbent's stored scores (if any) must be checked for `rubric_hash` and `judge_model_snapshot` match — a mismatch forces a re-score.
- Two models with API access: the **incumbent** (frontier, currently serving) and the **candidate** (the cheaper / newer / distilled model). If the candidate is a distilled student, you also need the teacher outputs on the eval set — either stored from the training run or re-generated.
- Python 3.11+; `scipy.stats` (paired-t, McNemar), `statsmodels.stats.multitest` (Benjamini-Hochberg), a plotting library.

## Set-up

1. Create `eval/tradeoff/replacement/` in your repo:

   ```
   eval/tradeoff/replacement/
   ├── config.yaml
   ├── candidates/
   │   ├── incumbent.yaml               # full lineage keys for the current production system
   │   └── candidate.yaml               # full lineage keys for the proposed replacement
   ├── distillation/                    # populated only when the candidate is a student
   │   ├── teacher.yaml
   │   ├── training_summary.md          # training set hash, loss curves, early stopping
   │   └── teacher_scores.jsonl         # teacher's scores on the eval set
   ├── cohorts.yaml                     # reused from exercise-01
   ├── pre_registration/
   │   └── 2026-XX-XX-<surface>-replacement.md
   ├── score/
   │   ├── rescore_incumbent.py         # re-score incumbent if rubric_hash or judge drifted
   │   ├── score_candidate.py
   │   └── pair.py                      # per-case paired-Δ table
   ├── stats/
   │   ├── intervals.py                 # paired-t / McNemar CIs per cohort
   │   ├── fdr.py                       # Benjamini-Hochberg correction
   │   └── bucket.py                    # regression-bucket count: cases with Δ ≤ -threshold
   ├── report/
   │   ├── build_replacement_report.py
   │   ├── delta_histogram.py           # per-cohort Δ histogram with mean + CI
   │   └── coverage_quality_plot.py     # distillation-only; teacher × student scatter
   ├── decision/
   │   └── matrix.py                    # chapter 04 ship/ship-with-routing/hold/reject
   ├── output/
   │   └── <decision_id>/
   │       ├── replacement_report.md
   │       ├── delta_histogram.svg
   │       ├── coverage_quality.svg     # distillation only
   │       ├── per_case_deltas.csv
   │       └── report.json
   ├── tests/
   │   ├── test_pair.py
   │   ├── test_intervals.py
   │   ├── test_fdr.py
   │   ├── test_bucket.py
   │   ├── test_decision_matrix.py
   │   └── test_report_render.py
   └── README.md
   ```

2. Populate `candidates/*.yaml`. Both files have a full lineage-key set — model snapshot, prompt hash, chain hash, retriever index hash, decoding config, cache policy. The `candidate_id` is `sha256(yaml)`. If a lineage key other than `model_snapshot` differs between incumbent and candidate, name it in the config — a swap-in with a changed prompt is not a pure model-replacement eval; split the change or acknowledge the confound in the report footer.

3. Write and sign the pre-registered contract. Chapter 04's decision matrix needs its thresholds pre-set. The anti-motivated-reasoning discipline from chapter 02 applies — committed and signed before `score/` runs:

   ```markdown
   # Pre-registered replacement rule — <surface> <decision_id>
   Written: 2026-XX-XX
   Signed:  PM=<name>, ENG=<name>

   Baseline / incumbent: <candidate_id of incumbent>  (rubric_hash=<x>, judge=<y>)
   Candidate:            <candidate_id of candidate>

   Per-cohort δ_c (rubric points):
     regulated   = 0.0
     enterprise  = 0.3
     en-US       = 0.5
     ja          = 0.5
     long-input  = 0.5
     ... (one row per declared cohort)

   Regression-bucket contract (per cohort):
     max % of cases with Δ ≤ -1.0 rubric points = 5%

   Multiple-testing:
     Benjamini-Hochberg FDR at q = 0.05 across cohort-level pass tests.

   Decision matrix (chapter 04 shape):
     SHIP               — all cohorts pass; cost drops ≥ 40%; latency does not regress by > 100 ms.
     SHIP WITH ROUTING  — failing cohorts are cleanly separable (named below); recommendation
                          is to escalate to exercise-02 and route failing cohorts to incumbent.
     HOLD               — failures are not cleanly separable; cost / latency win is substantial
                          enough that the model team should iterate. Timeline for re-eval: <date>.
     REJECT             — failures not separable and cost / latency win is below the threshold
                          that justifies holding.

   Separable-cohort list (for SHIP WITH ROUTING path):
     regulated, enterprise, long-input    # the three a chapter-03 router can cleanly isolate

   Rollback trigger:
     If mod-107 online-loop faithfulness rubric on any critical cohort drops > 0.5 points
     over rolling 24 h window vs. pre-ship baseline, auto-rollback via flag <name>.
   ```

4. If the candidate is a distilled student, populate `distillation/`:
   - `teacher.yaml` — teacher model's lineage keys. May be the same as `incumbent.yaml` (common case: student distilled from the current production frontier).
   - `training_summary.md` — training-set hash, training loss curves (links to upstream artefacts), early-stopping epoch, any known teacher-coverage gaps on the training set. This is read-only context for the report, not re-derived here.
   - `teacher_scores.jsonl` — the teacher's rubric scores on every eval-set case. Either stored from the training run or re-scored here.

5. **Re-score the incumbent if needed.** `rescore_incumbent.py` reads the current rubric hash and judge snapshot; compares against the stored incumbent scores; if either drifts, re-scores from scratch. The report refuses to proceed with stale incumbent scores — chapter 04's "rubric stability across the swap" rule.

## Requirements

Produce a PR against your working branch that adds:

1. **`candidates/incumbent.yaml`, `candidates/candidate.yaml`** — full lineage-key sets; `candidate_id`s stable; any non-model lineage change explicitly flagged.
2. **`pre_registration/<decision_id>.md`** — signed by product and engineering, committed before `score/*` runs. Includes per-cohort `δ_c`, regression-bucket contract, FDR method, decision matrix thresholds, separable-cohort list, rollback trigger.
3. **`score/rescore_incumbent.py`** — reads `(rubric_hash, judge_model_snapshot)` from the current rubric; checks stored incumbent scores; re-scores any case whose stored run has stale lineage keys. Idempotent against unchanged keys.
4. **`score/score_candidate.py`** — scores the candidate on every eval-set case under the current rubric + judge. Writes rows with `candidate_id`, `rubric_hash`, `judge_model_snapshot`, pricing snapshot.
5. **`score/pair.py`** — produces the paired-Δ table: one row per `(case_id, cohort, rubric, Δ = score_candidate − score_incumbent)`. The paired-Δ table is the primary artefact of the exercise; every subsequent calculation reads from it. Writes `per_case_deltas.csv`.
6. **`stats/intervals.py`** — per-cohort paired-t 95 % CI for continuous rubrics; McNemar exact CI for binary pass/fail. The report cell shows `mean Δ ± SE`; cells whose CI straddles `δ_c` are flagged "insufficient evidence."
7. **`stats/fdr.py`** — Benjamini-Hochberg across cohort-level pass tests at the pre-registered `q`. The report's `pass?` cells reflect the corrected result; the footer names the raw p-values and the correction method.
8. **`stats/bucket.py`** — count and percentage of cases with `Δ ≤ -1.0` per cohort (threshold configurable). The report flags cohorts where the regression-bucket percentage exceeds the contract's cap.
9. **`report/build_replacement_report.py`** — the chapter 02 shape extended with chapter 04 specifics:
   - Header block.
   - Per-cohort Δ table: `mean Δ`, `worst-case Δ`, `5th-pct Δ`, `95th-pct Δ`, `regression-bucket count`, `δ_c`, `pass?`.
   - Regression concentration summary (which cohorts hold most of the regressions).
   - Cost & latency delta row.
   - Decision matrix with the per-clause pass / fail, the verdict, and (if `SHIP WITH ROUTING`) the escalation to exercise-02.
   - Rollback trigger stanza.
   - Report hash in the footer.
10. **`report/delta_histogram.py`** — per-cohort Δ histogram with the mean centred and the whole distribution visible. Bimodal cohorts are the whole point — a cohort with `mean Δ = -0.05` and a two-peak distribution tells a very different story from one with `mean Δ = -0.05` and a tight cluster.
11. **(Distillation only) `report/coverage_quality_plot.py`** — scatter plot: teacher score on x, student score on y, one point per case, small-multiples per cohort. The chapter 04 interpretation (diagonal / below / above, top-left / top-right / bottom-right) is spelled out in the report's commentary section. **Teacher-agreement rate** is a report metric: fraction of cases where the student's answer semantically matches the teacher's (equality for binary, within-ε for continuous).
12. **`decision/matrix.py`** — parses the pre-registered rule into executable form; evaluates the four-way decision; returns the verdict with the per-clause breakdown. Reproducible — running twice on the same inputs returns identical bytes.
13. **`tests/*`** — unit tests for:
    - `test_pair.py` — paired-Δ computation; a known fixture produces the expected per-case Δ.
    - `test_intervals.py` — paired-t CI matches a `scipy.stats.ttest_rel`-computed CI on a fixture; McNemar CI matches a known table.
    - `test_fdr.py` — a BH fixture (known p-values; known corrected q-values) produces the correct accept / reject set.
    - `test_bucket.py` — a fixture with hand-computed regression-bucket counts.
    - `test_decision_matrix.py` — fixtures representing each of the four outcomes; the matrix returns the correct verdict.
    - `test_report_render.py` — stable Markdown render modulo timestamps.
14. **A demonstration run** on ≥ 300 eval cases, 6+ cohorts, incumbent + candidate (optionally + teacher). The report must show:
    - At least one cohort where the candidate's `Δ` is on the edge of the contract.
    - The regression concentration summary identifying which cohort or input-length-band holds most of the regressions.
    - A decision landing on one of the four verdicts — ideally **SHIP WITH ROUTING** so the escalation to exercise-02 is exercised.
15. **`README.md`** — how to configure the candidate, how to pre-register, how to re-score, how to generate the report, how to read the decision matrix; one page walking through the demonstration run.

## Starter guidance

- **Freeze the eval set before you start.** Even a 10-case addition between the incumbent score and the candidate score silently breaks pairing. The `eval_set_hash` is the discipline — if it changes mid-run, abort and restart.
- **Re-score the incumbent even if it feels wasteful.** The judge model has probably bumped a snapshot since the incumbent's stored scores were written. The rubric YAML has probably seen a small clarification. Either breaks pairing. Chapter 04's rule: when `rubric_hash` or `judge_model_snapshot` differs, re-score. The cost is finite; the alternative is a non-paired comparison masquerading as paired.
- **Report per-case deltas as the primary artefact; aggregates are summaries.** The `per_case_deltas.csv` is the file an auditor queries. Every number on the report is derivable from it; nothing in the report is derived from only the aggregates.
- **Multiple-testing correction is not optional.** With 10 cohort tests at `α = 0.05`, the family-wise false-positive rate is ~40 %. Benjamini-Hochberg (or Holm-Bonferroni for small cohort counts) is the discipline. The report footer names the method.
- **Insufficient evidence is a valid cohort result.** If the paired-t CI straddles `δ_c`, the correct `pass?` cell is "⟳ insufficient evidence — N cases, SE too wide." The exercise acceptance criteria explicitly list this as expected on small cohorts; the fix is more cases, not a hand-wave at the threshold.
- **SHIP WITH ROUTING is a routinely-correct outcome.** A candidate that fails on 2 – 3 named cohorts but wins big on cost / latency is a canonical case for routing. The decision matrix says so; the escalation to exercise-02 is a chaining pattern the module expects.
- **For distillation, the coverage-and-quality plot is the central attribution.** Mean Δ hides mode splits; the scatter shows them. Points in the bottom-right (teacher correct, student wrong) are the distillation-regression cases; they are the fix target. Points in the top-left (teacher wrong, student correct) are worth investigating too — the student may have generalised past a teacher error.
- **Teacher-agreement rate is a decoy if used alone.** Chapter 04's anti-pattern "style-match distillation" — high agreement with low task quality means the student learned the teacher's wrong answers. The report always shows task-quality next to agreement; the exercise acceptance criteria require this co-presentation.
- **Rollback trigger is written for the cohort that was tightest in the pre-registration.** If the `regulated` cohort was `δ_c = 0.0`, the online trigger watches `regulated`. Don't watch a random cohort; watch the one closest to failing the contract.

## Acceptance criteria

You are done when:

- Pre-registration is signed and committed before any `score/` script runs; `candidate_id`s are stable hashes of their YAML.
- Incumbent re-score ran if `rubric_hash` or `judge_model_snapshot` differed from the stored scores; the report footer confirms both the incumbent and candidate scored under identical lineage keys.
- `per_case_deltas.csv` has one row per `(case_id, cohort, rubric)`; aggregates in the report are all derivable from this file.
- Per-cohort CIs are present on every `mean Δ` and visible on the report; cells with insufficient evidence are marked, not forced into pass or fail.
- Benjamini-Hochberg (or declared alternative) correction is applied across cohorts; the report footer names the method and lists the raw p-values.
- Regression-bucket counts (cases with `Δ ≤ -1.0`) are per cohort and tested against the pre-registered cap.
- The decision matrix produces one of ship / ship with routing / hold / reject, reproducibly, with the per-clause breakdown and the rollback-trigger stanza.
- (Distillation only) coverage-and-quality scatter is rendered per cohort; teacher-agreement rate and task-quality are co-presented; a bottom-right cluster, if present, is called out in the report commentary.
- Tests pass in CI; each `tests/*.py` includes a fixture that would catch a regression in the arithmetic it is testing.
- The demonstration run lands on a verdict the pre-registered rule actually dictates — not a verdict forced by threshold-fiddling after the numbers were seen.
- `README.md` walkthrough reads cleanly for someone who has not read chapter 04.

## Stretch goals

- **Permutation-based CIs.** For small cohorts where paired-t assumptions are shaky, replace the paired-t CI with a permutation-test CI; show both numbers and let the report reader pick.
- **Bimodality detection.** Flag cohorts whose Δ distribution has a bimodality coefficient above a threshold — the report's commentary section calls them out automatically.
- **Rubric-stability audit.** Scheduled job that re-scores a 50-case canary subset of the incumbent on a weekly cadence and alerts if the rubric or judge snapshot has drifted enough to shift the stored scores by more than `0.1`. Catches silent judge drift between replacement evals.
- **Distillation gap sources.** For each bottom-right scatter point, surface the input features the teacher used (via a reasoning-chain extraction or a shorter probe) and compare to the student's. The output is a "failure cluster" taxonomy — "all these are reasoning-chain length > N" or "all these are tool-call-heavy." The next distillation run has a targeted training-data expansion.
- **Teacher ensemble.** If the student was distilled from an ensemble of teachers, score each teacher separately and compute the student-vs-each-teacher coverage. Surfaces whether the student collapsed onto one teacher or genuinely interpolated.
- **Routing-plan auto-generator.** When the verdict is `SHIP WITH ROUTING`, auto-generate a candidate `policy/router.yaml` for exercise-02 that routes the failing cohorts to the incumbent and everything else to the candidate. Hand-review the generated file; use it as the starting point for the routing-eval.
- **Historical comparison.** Query mod-110's `eval_results` for prior replacement runs on the same surface; a side panel shows the trajectory of the per-cohort Δ across replacements, so a slow drift of the mid-tier vs. frontier gap is visible.

## What this exercise does *not* cover

You are not training the distilled student (upstream — model team), tuning the fine-tune, or choosing the teacher. You are not building the routing policy that `SHIP WITH ROUTING` recommends (that is exercise 02); the app-altitude latency measurement (exercise 04); or the per-feature cost accounting (exercise 05). You are shipping the *paired-comparison replacement harness and the delta report with the four-way decision matrix* — the chapter 04 artefact.
