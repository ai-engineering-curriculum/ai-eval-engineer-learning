# Practical Inter-Annotator Agreement at Product Scale

## Motivation

Chapters 02 and 03 produce two reviewers' labels on the same rows. The question this chapter answers is: *should the eval team trust those labels?* Concretely, when two trained reviewers read the same prompt / response pair against the same rubric and write different labels, is the rubric unstable (and needs a rewrite), are the reviewers under-trained (and need re-calibration), or is the row genuinely borderline (and the disagreement is signal, not noise)?

Inter-annotator agreement (IAA) is the statistic the eval team runs to make that call. It is the single most important numerical artefact in human review — without it, a labelled dataset is a stack of opinions; with it, the dataset becomes a measurable ground truth with a known confidence interval.

Three grounded reasons the product-altitude version of IAA is enough for this module (and where the peer track takes over).

- **The decision at product cadence is three-way, not continuous.** Ship the rubric (IAA is high enough to anchor the mod-104 calibration), revise the rubric (IAA is low enough that the mod-104 judge cannot be trusted against it), or escalate to the `model-evaluation-engineer` peer (IAA is in a grey zone, or the rubric is ordinal with unequal intervals, or the sample size is too small for the bootstrap). Chapter 04's kappa and alpha compute enough to reach the three-way call.
- **The peer track owns the full methodology.** Variance decomposition, prevalence and bias correction, power analysis, unequal-interval ordinal treatment, hierarchical IAA across reviewer cohorts — all are peer artefacts. If this module tried to reproduce them, the peer would reject the numbers and the program would carry two incompatible IAA reports.
- **Running IAA on product cadence requires a cheap, defensible number.** The eval team computes IAA every sprint; the peer runs the full methodology once a quarter. The sprint version is what this chapter walks.

This chapter walks *when* to compute IAA, the three statistics the module computes (Cohen's kappa, Fleiss' kappa, Krippendorff's alpha), the bootstrap confidence interval, the interpretation thresholds, the three-way rubric-stability decision, and the escalation contract with the peer track.

## Core concepts

### When to compute IAA — four triggers and one anti-trigger

IAA is not a once-a-quarter report. Four triggers run it on cadence; one anti-trigger explains when *not* to run it.

- **Rubric change.** Any revision to the rubric vocabulary (adding a category, changing a label's definition, promoting a free-text `reason_code` to an enum) invalidates the prior IAA. Re-run on the first 50 – 100 rows after the change. If kappa drops, the rubric change was not an improvement — roll back or re-train the reviewers.
- **Judge re-calibration.** Before the mod-104 judge recalibrates against a labelled set, run IAA on the set. A judge calibrated against a low-IAA set calibrates against noise. Mod-104 chapter 04 names this precondition.
- **New reviewer onboarding.** A new reviewer's first 50 labelled rows are co-labelled by a senior reviewer. Compute the pairwise Cohen's kappa on the overlap. A new reviewer at κ < 0.6 against the senior panel is not yet trained; another round of rubric walkthrough and pilot labelling, then re-measure.
- **Gold-set rotation.** Chapter 05's rotation adds new rows to the rolling gold. Run IAA on the new rows against the panel before the rotated gold ships. A gold set that two reviewers disagree on cannot be the reference for the automated suite.

The one anti-trigger: **do not compute IAA on a set the panel has already adjudicated.** Once the senior adjudicator resolved the two-reviewer disagreements (chapter 02's workflow), the labels are no longer independent. IAA on adjudicated labels is always artificially high. Chapter 04's IAA reads the *pre-adjudication* labels; the post-adjudication label is the shipped ground truth.

### Cohen's kappa — two reviewers, nominal rubric

Cohen's kappa is the two-reviewer, two-or-more-category nominal statistic. The definition:

```
κ = (p_o − p_e) / (1 − p_e)
```

where `p_o` is the observed agreement (fraction of rows both reviewers labelled the same) and `p_e` is the expected agreement under chance (computed from the marginals). The subtraction corrects for the agreement two reviewers would reach by chance given the label distribution.

Three product-altitude properties to know:

- **Nominal rubric means label order does not matter.** If the rubric is `{aligned_refuse, over_refuse, aligned_complete, missing_warning}`, treat it as four unordered labels; use Cohen's κ. If the rubric is ordinal (e.g. `{very_bad, bad, ok, good, very_good}`), use Krippendorff's α instead — κ penalises an "off by one" disagreement the same as a "flipped at opposite ends" disagreement.
- **Prevalence and bias can distort κ.** An extremely skewed label distribution (99 % of rows are `aligned_complete`) produces a low κ even when the reviewers agree on almost every row. The product-altitude mitigation is to report κ alongside raw agreement and the per-label marginal; the full-methodology mitigation (PABAK, κ-max correction) is peer-track.
- **The implementation is one function call.** `sklearn.metrics.cohen_kappa_score(y1, y2)` in Python computes it. Both Argilla and Label Studio export the pre-adjudication labels in a shape that feeds directly in.

### Fleiss' kappa — three or more reviewers

When three or more reviewers label the same row, Cohen's κ no longer applies (it is pairwise). Fleiss' κ generalises to any number of reviewers.

- Each row has `n` reviewer verdicts; standard Fleiss' κ, and the `statsmodels.stats.inter_rater.fleiss_kappa` implementation specifically, requires the same `n` on every row. If the senior reviewer samples a subset of the primary reviewers' rows, the resulting table is ragged — the senior-reviewed subset has `n = 3`, the rest has `n = 2` — and Fleiss on the mixed table either asserts out (statsmodels) or returns a number that is not comparable across rows.
- The statistic reads `p_o` as the mean per-row agreement across all reviewer pairs and `p_e` as the expected chance agreement given the overall label distribution.
- Interpretation is the same scale as Cohen's κ; do not re-scale when switching from two to three reviewers.

Fleiss' κ is the default when three or more reviewers each label the same rows. In the rota (chapter 02) where two primary reviewers cover the full set and a senior reviewer reads a sample, compute Fleiss' κ on the fully-overlapped senior-reviewed subset only; for the ragged full table, switch to Krippendorff's α (next section), which handles the missing-label case cleanly. The three-reviewer κ on the sample is a more conservative stability signal than the two-reviewer κ on the full set.

The implementation lives in `statsmodels.stats.inter_rater.fleiss_kappa` <!-- needs-research: re-verify the current statsmodels module path at https://www.statsmodels.org/stable/ --> or in `krippendorff` (which also computes Fleiss as a degenerate case).

### Krippendorff's alpha — ordinal rubrics and missing labels

Two situations where kappa breaks and Krippendorff's α is the right statistic.

- **Ordinal rubrics.** The rubric is `{very_bad, bad, ok, good, very_good}` and an "off by one" disagreement is less severe than a "flipped at opposite ends" disagreement. Krippendorff's α parameterises the distance function; the ordinal-distance variant weights disagreements by their rank difference.
- **Missing labels.** Reviewer A labelled rows 1 – 100; reviewer B labelled rows 50 – 150. The 50 – 100 overlap is where IAA lives. κ does not handle the missing-label case cleanly; Krippendorff's α does.

The ordinal case is the common one for product-side rubrics with graded-quality dimensions (helpfulness, specificity, friendliness). Use α with the ordinal distance; report alongside the unweighted κ for cross-comparability.

Implementation: the `krippendorff` Python package <!-- needs-research: re-verify the current `krippendorff` package maintainer and API surface at https://pypi.org/project/krippendorff/ --> ; one call takes a reliability-data matrix (rows = reviewers, columns = items, cells = labels with `np.nan` for missing) and a distance metric (`nominal`, `ordinal`, `interval`, `ratio`).

### The bootstrap confidence interval

A κ of 0.74 on 50 rows is not the same as a κ of 0.74 on 500 rows. The bootstrap CI quantifies the difference.

The product-altitude recipe:

- Resample the reviewed rows with replacement `B = 1000` times.
- Compute κ (or α) on each resample.
- Report the 2.5th and 97.5th percentiles as the 95 % CI.

A CI of `[0.68, 0.79]` is a different artefact from a point estimate of `0.74`. Two decisions the CI drives:

- **Comparing two κ values across rubrics or across time.** If `[0.68, 0.79]` on rubric v11 and `[0.72, 0.81]` on rubric v12 overlap substantially, the rubric change did not move the needle; if they do not overlap, the change is real.
- **The ship / revise / escalate decision.** The three-way call is against the CI, not the point estimate. A point κ of 0.62 with CI `[0.51, 0.72]` is in the grey zone even though the point says "revise".

The `scipy.stats.bootstrap` function (or a hand-rolled resampler, since IAA is cheap to compute) implements the recipe.

### Interpretation thresholds — product-altitude reading

The κ / α scales are the same (−1 to 1; 0 is chance; 1 is perfect agreement). Several interpretation charts exist in the literature; the module uses a conservative three-bucket read tuned for product use.

| Range | Interpretation at product altitude | Decision |
|---|---|---|
| κ / α ≥ 0.8 | High agreement; rubric is stable; labels are trustworthy | **Ship the rubric.** Use the labels for calibration, release-gate, drift reference. |
| 0.6 ≤ κ / α < 0.8 | Moderate agreement; rubric has some ambiguous cases; labels are usable with caveats | **Revise the rubric.** Identify the disagreement clusters, rewrite the ambiguous categories, re-measure within two weeks. Labels may still be used for calibration if the mod-104 chapter 04 procedure accepts the uncertainty. |
| κ / α < 0.6 | Low agreement; the rubric is unstable or the reviewers are under-trained | **Escalate.** Either re-train reviewers or escalate to the peer for a full methodology pass. Do not calibrate the mod-104 judge against this set. |

<!-- needs-research: the Landis & Koch (1977) chart is still the most-cited reference for kappa interpretation ranges; re-verify no newer consensus has displaced it in the IAA methodology literature. -->

Three important caveats to the chart:

- **The thresholds are rubric-shape-dependent.** A binary safety rubric (safe / unsafe) at κ 0.65 is less reassuring than a 10-way nominal rubric at κ 0.65. The chart is a rough guide; the per-rubric sanity check (does the disagreement cluster match a known ambiguous category?) is the actual signal.
- **The CI is a soft-fail trigger.** A point κ of 0.82 with CI `[0.71, 0.89]` straddles the high / moderate threshold. Treat straddling as moderate — the conservative call.
- **The chart is a decision input, not a certification.** A regulator asking "what's your IAA?" and accepting a one-number answer is not running IAA properly either. The eval team publishes the number, the CI, the sample size, the rubric version, and the disagreement-cluster audit.

### The disagreement-cluster audit

The IAA number alone does not tell the rubric author what to fix. The disagreement-cluster audit does.

- Export every row where the two reviewers disagreed.
- Group by the pair of labels (e.g. `aligned_complete` vs `over_refuse`).
- For each group, read a sample of the rows (not all — a 10-row sample is often enough to see the pattern) and write a one-line description of the ambiguity.

Typical findings:

- **Label-definition ambiguity.** The rubric says `over_refuse` is "refusing a request that the policy permits"; one reviewer read the request as policy-permitted, the other read the policy as silent. Fix: tighten the policy wording (the policy is chapter 06 of mod-108).
- **Rubric-category overlap.** `incorrect_answer` and `not_grounded_in_retrieval` both apply when the model hallucinated a citation from the context. Fix: pick one as primary; demote the other to a secondary tag or merge.
- **Reviewer drift.** The two reviewers used different implicit thresholds for "good enough citation". Fix: add a worked example to the rubric; re-train both reviewers.

The audit output is attached to the rubric revision proposal; the next IAA run measures whether the revision moved the needle.

### Escalation to the `model-evaluation-engineer` peer

The peer track takes over when the product-altitude pass cannot answer the question. Three common escalation shapes.

- **Unequal-interval ordinal rubrics.** A rubric scored on `{very_bad, bad, ok, good, very_good}` where "bad" and "ok" are much closer together than "very_bad" and "bad". Krippendorff's α with ordinal distance treats the intervals as equal; the correct analysis uses interval or ratio distance with an estimated metric. Peer work.
- **Variance decomposition.** Is the disagreement driven by the row (some rows are genuinely harder), by the reviewer (one reviewer runs tight, another loose), or by the interaction? The product-altitude IAA cannot say; the peer runs a mixed-effects model and returns the variance components.
- **Power analysis.** How many reviewed rows does the module need to measure a κ difference of 0.05 with 80 % power? The peer runs the sample-size calculation; the module adjusts the rota.

The escalation contract is lightweight: the module sends the pre-adjudication reviewed rows (CSV with reviewer_id, row_id, label, rubric_version), the rubric document, and the question. The peer returns a methodology report; the module reads it against the three-way decision.

### The IAA job on the pipeline

The IAA computation runs as a job in the mod-110 eval data platform, not as a one-off notebook. The job reads the pre-adjudication labels from `reviewer_actions`, filters to the rubric and the time window, computes κ or α, computes the bootstrap CI, writes the result to `iaa_reports` with the rubric version and the sample size, and surfaces a tile on the mod-107 safety-reviewer dashboard.

Two properties of a good job:

- **Nightly, not quarterly.** A nightly run on the trailing-14-day window catches rubric instability early. A quarterly run catches it when the quarter is done.
- **Fails loud on sample-size drops.** If the trailing window has fewer than the minimum-sample-size cutoff (the module uses 30 as a floor), the job writes `insufficient_sample` to the report rather than computing a misleading number.

## Summary

- IAA is the statistic that decides whether a rubric is stable enough to anchor the eval program. Without it, labels are opinions; with it, labels are a measurable ground truth.
- **Four triggers to run IAA**: rubric change, judge re-calibration, new reviewer onboarding, gold-set rotation. **One anti-trigger**: never on adjudicated labels.
- **Three statistics**: Cohen's κ (two reviewers, nominal), Fleiss' κ (three or more reviewers, nominal), Krippendorff's α (ordinal rubrics or missing labels). Always alongside a **bootstrap CI** (B = 1000, 95 %).
- **Three-bucket interpretation**: κ / α ≥ 0.8 ship the rubric; 0.6 – 0.8 revise; < 0.6 escalate. Treat CIs that straddle the threshold as the lower bucket.
- The **disagreement-cluster audit** turns the IAA number into a rubric-rewrite plan. The audit output feeds the next rubric revision; the next IAA run measures the revision.
- **Escalate to the peer** when the rubric is ordinal with unequal intervals, when variance decomposition is needed, or when a power analysis is needed. The product-altitude pass does enough to decide; the peer does enough to publish.
- The **IAA job** runs nightly on the trailing-14-day window; results land in `iaa_reports`; the mod-107 dashboard surfaces the tile. Fail loud on insufficient sample.

Chapter 05 walks the gold-set rotation discipline — the artefact the stability signal from this chapter protects.
