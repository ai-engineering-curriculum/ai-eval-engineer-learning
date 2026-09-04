# Distillation and Replacement Regression: Prove Where Quality Moves

## Motivation

A cheaper model is available. Maybe the vendor released a mid-tier snapshot that benchmarks say is 90 % of the frontier at 30 % of the price. Maybe the model team distilled the frontier's outputs into an in-house 8B model that runs on the org's own GPUs at 5 % of the vendor cost. Maybe the community released an open-source model that a small fine-tune brings up to production quality. Someone in the room says "let's swap it in."

The swap-in is not a launch. It is a **regression event**: the same eval set that scored the incumbent is scored on the candidate, and every case where the candidate scores differently is a per-case delta the report has to explain. Skip the per-case work, average across the suite, and the shipping decision is "quality looks fine" — until a specific product surface (the enterprise support tab, the long-context research feature, the `de` locale) silently degrades and the swap-in cannot be attributed.

This chapter builds the replacement-regression discipline the trade-off report (chapter 02) needs when the whole model changes, not just the routing. Distillation is one case of replacement — the candidate is a student model of the incumbent teacher — and gets a specific attribution treatment.

The chapter's central pattern is a **paired-comparison eval**: the same inputs run through both incumbent and candidate, the per-case deltas surfaced, the per-cohort quality movement attributed, the ship / hold / reject decision landed against a pre-registered rule. Everything the report needs to defend a swap-in.

## Core concepts

### The swap-in as a regression event, not a launch

Every replacement is a regression risk. The eval discipline treats it that way:

- The candidate goes through the **same eval set** the incumbent last passed. New tests for the candidate do not count — a candidate that passes tests the incumbent never took is not comparable.
- The eval is **paired** — same input, both models — so per-case deltas are meaningful. Un-paired A/B (different inputs to each model) confounds candidate quality with input distribution.
- The report shows **per-case deltas** first, per-cohort second, per-suite third. The whole point is to surface *where* quality moved, not *whether* it moved on average.
- The decision has a **pre-registered rule** (chapter 02) and a **rollback trigger** (mod-107). A swap-in without a rollback trigger is a permanent decision made against a temporary set of numbers.

Two consequences worth naming:

- **The candidate's improvements do not compensate for the candidate's regressions in the decision.** If the candidate gains 0.3 rubric points on cohort A but loses 0.5 on cohort B, the answer is not "net +0.1, ship." The answer is "cohort B lost 0.5; is that acceptable per the pre-registered rule?" Chapter 02's cohort-preservation contract carries into replacement eval unchanged.
- **The paired eval's per-case deltas expose the cohorts the pre-declared roster missed.** If the pre-declared cohorts all pass, but the per-case delta plot shows a bulge of regressions concentrated in cases about a specific product feature, that is a cohort the report *should* have carried. Chapter 03 of mod-107 covered slice discovery; the same tools apply.

### Distillation vs. general replacement

Both are replacements; the attribution differs.

- **Distillation** — the candidate is a *student* trained on the *teacher*'s outputs (the incumbent, or another frontier model). The student is expected to approximate the teacher; the eval question is *how much of the teacher's quality survived the compression*. The student-vs-teacher gap on each cohort is the primary metric; a per-cohort **coverage rate** ("student matches teacher within ε on X % of cases") supplements the mean.
- **General replacement** — the candidate is an independently-trained model (a different vendor's snapshot, an open-source model, a fine-tune of a different base). No teacher-student relationship; the eval is a straight paired comparison against the incumbent.

Distillation has a specific failure mode general replacement does not: the student can pass the eval suite by matching the teacher's *style* while failing the underlying task on the tail. The teacher's "sounds right" quality transfers before the teacher's "is right" quality. The distillation report always includes a **factuality / task-completion sub-column** that measures the *underlying* task quality, not just the style match to the teacher.

Two additional distillation-specific report elements:

- **Teacher-agreement rate.** For factual / verifiable tasks, the fraction of cases where the student's answer semantically matches the teacher's. High agreement with high task-quality → distillation worked. High agreement with low task-quality → the teacher was also wrong and the student inherited the wrong answers. Low agreement with high task-quality → the student is generalising past the teacher (usually surprising and worth investigating).
- **Compression-attributable regression.** For cases where both the teacher and the student are correct on the incumbent scale, the student's shorter / cheaper answer is a *feature*, not a regression. The distillation report separates "student worse than teacher" from "student cheaper than teacher, same quality" — the second is the reason to distill.

### The paired-comparison methodology

The core methodology, in six steps.

1. **Freeze the eval set.** Use the mod-106 replay-bundle hash. The set is unchanged for the duration of the eval.
2. **Run the incumbent on every case.** Record scores per rubric, per cohort, per case. If the incumbent's scores are already stored (from the last eval run), re-use them — but check the incumbent's `model_snapshot` is unchanged; a vendor snapshot bump on the incumbent silently changes the baseline.
3. **Run the candidate on every case.** Record scores per rubric, per cohort, per case.
4. **Compute per-case deltas.** For each case, `Δ = score(candidate, case) − score(incumbent, case)`. The paired-Δ distribution is the primary artefact.
5. **Aggregate deltas per cohort.** Mean and worst-cohort delta per cohort × rubric. Chapter 02's cohort matrix shape.
6. **Test each cohort's delta against the pre-registered contract.** For each cohort `c`, `Δ̄_c ≥ −δ_c`. Pass / fail per cohort.

Per-case deltas are the artefact the discipline hinges on. Two derived plots the report always ships:

- **Δ histogram** — one bar per cohort, mean Δ centred, the whole distribution shown. A cohort with a mean of −0.05 and a two-modal distribution (half the cases +0.5, half −0.6) is very different from a cohort with a mean of −0.05 and a tight cluster around 0. The mean hides the mode.
- **Regression bucket count** — the count of cases with `Δ ≤ −threshold` (default: 1 rubric point on a 0 – 5 scale). A pre-registered maximum on this count is a legitimate rule: "reject the candidate if more than 3 % of cases regress by ≥ 1 point."

For binary / pass-rate rubrics, the paired analogue is McNemar's test (see the resources for the reference); the test's `b` (was passing, now failing) and `c` (was failing, now passing) counts are the paired-Δ analogue.

### The delta report shape

The trade-off report (chapter 02) extended with paired-comparison specifics.

```
# Replacement regression report — support-bot: frontier-v3 → mid-v1

decision_id:          RRR-2026-08-27-support-bot
eval_set:             support-eval-v14 (hash sha256:f2c...c13)
incumbent:            support-bot-frontier-v3 (snapshot 2026-06-15)
candidate:            support-bot-mid-v1 (snapshot 2026-08-04)
methodology:          paired comparison, same inputs, same rubric hash
pre_registered:       rules/support-replacement-2026-08-22.md (sig: PM=alex, ENG=jamie)

## Per-cohort quality Δ (candidate − incumbent, rubric points, ±SE)

|              | mean Δ | worst-case Δ | 95th-pct Δ | 5th-pct Δ | regressions ≥ 1.0 | δ_c    | pass? |
|--------------|--------|--------------|------------|-----------|-------------------|--------|-------|
| regulated    | -0.56  | -2.31        | +0.10      | -1.80     | 22 of 84 (26%)    | 0.0    | FAIL  |
| enterprise   | -0.51  | -2.10        | +0.14      | -1.61     | 18 of 96 (19%)    | 0.3    | FAIL  |
| en-US        | -0.14  | -1.83        | +0.31      | -0.98     | 5 of 421 (1.2%)   | 0.5    | PASS  |
| en-GB        | -0.19  | -1.61        | +0.28      | -1.10     | 6 of 342 (1.8%)   | 0.5    | PASS  |
| es           | -0.31  | -1.94        | +0.19      | -1.20     | 8 of 288 (2.8%)   | 0.5    | PASS  |
| de           | -0.28  | -1.83        | +0.22      | -1.11     | 7 of 271 (2.6%)   | 0.5    | PASS  |
| ja           | -0.42  | -2.11        | +0.10      | -1.51     | 15 of 199 (7.5%)  | 0.5    | FAIL  |
| short-input  | -0.11  | -1.44        | +0.29      | -0.81     | 4 of 512 (0.8%)   | 0.5    | PASS  |
| long-input   | -0.67  | -2.51        | +0.02      | -2.00     | 41 of 218 (18.8%) | 0.5    | FAIL  |

## Regression concentration

Of 121 cases with Δ ≤ -1.0:
  - 63% concentrated in cohort `long-input` (candidate weaker on 8k+ inputs)
  - 18% concentrated in cohort `regulated` (candidate weaker on policy-sensitive Qs)
  - 14% concentrated in cohort `ja` (candidate weaker on Japanese)

## Cost & latency delta

cost per request: $0.0184 → $0.0059  (-68%)
streaming p95:     3.9 s → 2.1 s     (-1.8 s)
ttft p95:          620 ms → 380 ms   (-240 ms)
safety:            no OWASP category regression

## Decision matrix (pre-registered)

|            | Ship | Ship w/ router | Hold | Reject |
|------------|------|----------------|------|--------|
| All δ_c pass?           | ✗       |               |      |         |
| Any regulated Δ ≤ -δ_c? | ✗       | ✗ if router keeps regulated on incumbent | ✓ | if unresolvable |
| Cost win ≥ 30% & latency neutral? | ✓ | ✓          | ✓    |         |

Result: HOLD, with routing recommendation.
  The candidate fails the pre-registered contract on regulated, enterprise, ja, and long-input.
  However, the cost / latency win is substantial. Escalate to routing eval (chapter 03):
  route regulated + enterprise + long-input traffic to the incumbent (frontier),
  everything else to the candidate. Then re-run the trade-off report as a routing report.
```

The report's decision matrix is the key structural difference from chapter 02's shape. Replacement eval frequently produces a candidate that fails a straight swap but succeeds as a routing tier — the matrix names that path explicitly rather than forcing binary ship / reject.

### The three decisions replacement eval produces

Every replacement eval lands on one of four outcomes. The decision matrix in the pre-registered rule enumerates them.

- **Ship.** All cohorts pass the pre-registered contract. Cost / latency win at or above threshold. Safety unchanged. The candidate becomes the new incumbent; the incumbent moves to the history table.
- **Ship with routing.** Some cohorts fail the contract, but the cohorts are cleanly separable (a set of `regulated + enterprise + long-input` is a routing cohort). The recommendation escalates to chapter 03: build a routing policy that keeps the failing cohorts on the incumbent and moves the rest to the candidate. Re-run as a routing report.
- **Hold.** The candidate fails the contract in a way that a router could not cleanly separate. Recommendation: send the failure back to the model owner. If the candidate is a distilled student, more distillation data on the failing cohorts; if it is a vendor snapshot, wait for the next snapshot; if it is an open-source fine-tune, more fine-tune data.
- **Reject.** The candidate fails the contract and the cost / latency win does not justify the operational cost of holding indefinitely. Close the eval; the candidate is not shipping.

Every outcome is a valid replacement-eval result. The pre-registered rule makes clear which one this run produced.

### Statistical rigour on paired comparisons

The paired-Δ mean has a standard error. The report shows it. Two rules the exercise enforces:

- **Every reported cohort delta has a confidence interval.** For continuous rubric scores, a paired-t 95 % CI. For binary pass rates, McNemar's exact CI. If the CI on the delta straddles the contract's δ, the report notes "insufficient evidence" — the run does not conclude pass or fail; more cases in that cohort are needed.
- **Multiple-testing correction on the cohort-level pass tests.** If the report evaluates 10 cohorts against contracts, a naive per-cohort α = 0.05 test has an expected 0.5 false failures per run. Benjamini-Hochberg FDR control (or Holm-Bonferroni for a small cohort count) keeps the report's false-positive rate at a defended level. Mod-107 chapter 03 covered the same discipline for online drift monitors; it applies here.

The report's `pass?` cells are the result of the corrected test, not the raw comparison. The report footer names the correction method used.

### Rubric stability across the swap

A subtle failure mode: the rubric itself has drifted between the incumbent's last eval run and the candidate's. If the judge model was updated, or the rubric YAML was tweaked, the incumbent's stored scores and the candidate's fresh scores are not comparable.

The mod-110 chapter 02 lineage key set is the defense. Every stored score row carries the `rubric_hash` and `judge_model_snapshot`. The replacement-eval methodology:

- **Re-score the incumbent on the current rubric + judge snapshot before scoring the candidate.** If the incumbent's stored score is under a stale `rubric_hash` or `judge_model_snapshot`, discard and re-run. The extra inference cost is not optional.
- **Report the rubric and judge lineage keys on the delta report.** Both incumbent and candidate scores share the keys. A reader can verify at a glance that the comparison is apples-to-apples.

Un-versioned rubrics are how paired comparisons silently become unpaired. The re-score is the discipline that keeps them paired.

### Distillation attribution: the coverage-and-quality plot

A distillation-specific artefact.

For each case in the eval set, plot:

- x-axis: the *teacher's* quality score (the incumbent, or the model whose outputs the student was trained on)
- y-axis: the *student's* quality score
- one point per case

Interpretation:

- **Points on the diagonal** — the student matches the teacher. Distillation worked on this case.
- **Points below the diagonal** — the student is worse than the teacher. Distillation has a quality gap on this case.
- **Points above the diagonal** — the student is better than the teacher. Unusual but not impossible (regularisation, better generalisation on some cases).
- **Points at the top-right** — both models correct. The case is easy.
- **Points at the bottom-left** — both models wrong. The case is hard; distillation did not create or fix the failure.
- **Points at the top-left** — teacher wrong, student correct. Surprising; investigate.
- **Points at the bottom-right** — teacher correct, student wrong. The distillation regression cases; where the student lost the teacher's quality.

The plot is small-multiple per cohort. A distillation report that ships without the coverage-and-quality plot is missing its central attribution artefact.

### Anti-patterns to avoid

**"Unpaired A/B."** Route half the traffic to the candidate, half to the incumbent, and compare average quality. Confounds candidate quality with input distribution because each side sees different inputs. Use paired eval on a frozen set.

**"Suite-average delta."** Report a single Δ for the whole eval set. Hides cohort concentration; hides mode-splits. Report per-cohort, with the histogram.

**"Rescore optional."** Trust the incumbent's stored scores even though the rubric has been updated since. The comparison is no longer paired. Re-score is mandatory when the rubric or judge snapshot differs.

**"Style-match distillation."** Distillation report tracks only teacher-agreement rate. High agreement with a wrong teacher = the student learned the wrong answers. Task-quality is the primary metric; agreement is the secondary.

**"No rollback trigger."** Ship and hope. Mod-107's rollback trigger is not optional for replacements — the paired eval is on a frozen set; production traffic is not that set; drift is guaranteed.

**"Regression bucket ignored."** Cohort mean Δ is small; regression bucket count is large. A candidate with mean Δ = −0.05 and 5 % of cases regressing by ≥ 1 point is a candidate that shipped a bimodal quality outcome. Track the bucket count.

**"Novel eval set on the candidate."** The candidate passes a new eval suite the incumbent never took. Not comparable. Freeze the set; both models take it.

### Where this chapter plugs into the trade-off report

- The `quality` column on the multi-objective report gains per-cohort Δ, worst-case Δ, and regression-bucket-count sub-columns.
- The decision matrix is the extension the multi-objective report needs when a candidate fails a straight swap but succeeds as a routing tier — chapter 02's pre-registered rule format accommodates it as a decision-tree extension.
- The rollback trigger is a mod-107 online-loop condition on the per-cohort quality of the newly-shipped candidate — the same shape as chapter 02, populated with the specific cohorts that were tightest in the pre-registration.
- Distillation reports contribute the coverage-and-quality plot as an appendix to the multi-objective report.

## Summary

- Every replacement is a **regression event**, not a launch. Paired-comparison eval on a frozen set is the method; per-case Δ is the primary artefact; per-cohort aggregation is the reporting shape.
- **Distillation** is a specific replacement pattern (student trained on teacher outputs) with a specific failure mode — the student learns the teacher's style before the teacher's task quality. The **coverage-and-quality plot** and the **teacher-agreement rate** are distillation-specific report elements.
- The **six-step paired methodology**: freeze set → score incumbent → score candidate (with re-score on lineage-key drift) → per-case Δ → per-cohort aggregation → pre-registered contract test with FDR correction.
- The **delta report** has cohort mean Δ, worst-case Δ, 5th / 95th percentile Δ, regression-bucket count (cases with Δ ≤ −1.0), δ_c, and pass / fail — with confidence intervals and multiple-testing correction.
- The **decision matrix** enumerates the four possible outcomes: **ship / ship with routing / hold / reject**. Every outcome is a valid replacement-eval result; the pre-registered rule names which one.
- **Rubric stability across the swap** is enforced by the mod-110 lineage keys — the incumbent is re-scored under the current `rubric_hash` and `judge_model_snapshot` before the paired comparison is meaningful.
- **Statistical rigour**: paired-t (continuous) or McNemar (binary) confidence intervals per cohort; Benjamini-Hochberg FDR correction across cohorts; "insufficient evidence" is a valid per-cohort result.
- **Anti-patterns**: unpaired A/B, suite-average delta, optional re-score, style-match distillation, no rollback trigger, ignored regression bucket, novel candidate eval set.

Chapter 05 opens the latency column of the trade-off report — TTFT, TPOT, streaming p95, the reconciliation with the MLPerf-shaped benchmark, and the app-altitude discipline the report needs so the latency number is a user experience and not a vendor claim.
