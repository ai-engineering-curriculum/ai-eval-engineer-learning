# Quick-Calibrating a Judge Against a Small Human Gold Set

## Motivation

A judge without a calibration number is a judge you cannot defend. The dashboard says "84% pass"; the release gate says "regression at p < 0.05"; the PM asks *"how do we know the judge is right?"* — and the honest answer is *"we do not."* The number that answers the PM's question is **the agreement of the judge with human raters on a small representative gold set.** Everything else in the eval program — the SLIs, the CI thresholds, the online-eval loop, the release gates — sits on top of that number, and if the number is missing or unread, the program is running on faith.

This chapter is the pragmatic, product-side version of that calibration. It walks: how big and how representative the gold set has to be, which statistic to report and why, how to read the number honestly, and where to stop and hand off to the **`model-evaluation-engineer` peer** (the Model-Development-family role) for full methodology depth — bootstrap intervals with variance decomposition, item-response modelling, non-parametric tests, calibration-set power analysis. The peer does that work at model-development altitude. Your job at product altitude is to know when you are past what quick-calibration can defend, so you know when to escalate.

## Core concepts

### What "calibration" means here

**Calibration**, for this chapter, means: *your judge and a competent human agree on a specified gold set of your surface's inputs, to a specified statistic, above a specified threshold.* It is a scoped claim, not a global one. The claim only defends judgements on the surface, in the rubric version, and against the input distribution the gold set was drawn from. When any of those change (new surface, rubric edit, drift in the input distribution), you re-calibrate.

Calibration is not:

- The judge is "right" in some absolute sense.
- The judge tracks a benchmark's leaderboard.
- The rubric is well-designed (chapter 01's job).
- The judge is unbiased (chapter 02's job).

All four are prerequisites for calibration to be meaningful. Calibration is the last check that pins the rubric + judge + pipeline to a human-agreement number the release gate can defend.

### How big does the gold set have to be

The pragmatic product-side answer is **~50–100 items per criterion for a first-pass calibration**, stratified to cover the outcome levels of the rubric. That is small enough that a working engineer can hand-label in a day, and large enough to produce a statistic whose confidence interval is narrower than the effect sizes you care about for release gates. It is also large enough that swapping a judge model gives you a measurable delta, not noise.

Two forces set the number:

- **Below ~30 items**, the confidence interval on any of the statistics below is wide enough that the number barely constrains reality. Small gold sets are useful for spot-checking, not for gating.
- **Above ~200–300 items**, you are into the territory where the full-methodology work (bootstrap variance decomposition, per-stratum agreement, item-response modelling) starts to pay off. That is the `model-evaluation-engineer` peer's altitude — escalate.

If you are constrained to a smaller set (a new surface with no labelled data), be explicit in the eval plan: *"calibration is provisional at N=30; a full calibration is scheduled after online-eval accumulates M more labelled traces."* The provisional number is worth having; the honest caveat is worth writing.

### Stratification: what "representative" means

An unstratified sample of 100 items from your traffic will over-represent whatever is most common — greetings, follow-ups, easy retrievals. If your judge is 95% accurate on those and 40% accurate on rare hard cases, the aggregate agreement number will read ~90% and the release gate will pass while the hard cases regress.

Stratify by, at minimum:

1. **Rubric outcome level.** For an absolute 1–5 rubric, include roughly equal counts per level (or at least each level present, with enough per level to compute per-level agreement). For pairwise, roughly equal counts of A-wins / B-wins / ties.
2. **Surface class.** If your surface has meaningful sub-classes (topic categories, tool-call vs no-tool-call, first-turn vs follow-up), include each class in proportion to its production traffic — or over-sample rare classes when they are what the release gate cares about.
3. **Failure mode categories.** If you have a bug taxonomy from mod-102's bisection work or mod-109's review labels, include known failure classes explicitly.

Log the stratification scheme with the calibration result. When someone asks *"why does the judge score 85% on our production traffic but only 70% on this new arm,"* the answer is often *"the arm changed the distribution across strata."*

### Which statistic to report

There is no one-size-fits-all statistic — the shape of the rubric determines the right choice. The four you need to know at product altitude:

| Rubric shape | Preferred statistic | Reason |
|---|---|---|
| Absolute nominal (labels, no ordering: `safe` / `unsafe` / `not_applicable`) | **Cohen's κ** (or Fleiss's κ for > 2 raters) | Accounts for chance agreement; standard for categorical inter-rater agreement |
| Absolute ordinal (1–5 with anchors) | **Spearman's ρ** or **Kendall's τ** on the paired scores | Ordinal — treats 4-vs-3 as closer than 4-vs-1. Kendall is more conservative on small N |
| Pairwise (A / B / tie) | Cohen's κ over the three-class winner labels, and **agreement rate on non-tie decisions** as a secondary | The three-class κ answers "does the judge agree on the winner"; the non-tie agreement is the number the release-gate ultimately cares about |
| Absolute continuous (rare in product-side judging) | Pearson's r AND a Bland-Altman-style bias check | Correlation alone hides systematic offsets |

**Cohen's κ**: <https://en.wikipedia.org/wiki/Cohen%27s_kappa> (canonical Landis-Koch bands 0.61–0.80 "substantial", 0.81–1.0 "almost perfect" — do not treat as hard cutoffs; they are heuristics from a 1977 paper and their applicability to LLM-judge agreement is not settled). **Spearman's ρ** / **Kendall's τ**: standard non-parametric rank statistics; both live in `scipy.stats`. **Bootstrap confidence intervals**: resample the paired data with replacement, recompute the statistic per resample, report the 2.5th / 97.5th percentiles.

The 2–3 line rule for reporting: *"On N=100 stratified traces, {rubric}, judge {model+version}, statistic {name} = {value} (95% bootstrap CI [{lo}, {hi}])."* Anything less is not a defensible number.

### Reading the numbers honestly

Two failure modes to catch when interpreting the calibration:

- **Confidence intervals that overlap the null.** A κ = 0.55 with CI [0.30, 0.75] is *not* the same claim as κ = 0.55 with CI [0.52, 0.58]. The former does not defend a release gate; the latter does. Report the CI, not just the point estimate.
- **A good aggregate hiding a bad stratum.** Aggregate κ = 0.72 while κ on the `safety-critical` stratum = 0.34. Report per-stratum agreement in the calibration doc; do not average over a stratum where a bad judgement is disproportionately costly.

The correct response to a bad calibration is not "run again with more samples until it looks better." It is (in order): rewrite the rubric anchors (chapter 01), add missing bias controls (chapter 02), switch judge tier (chapter 03), and — if none of those work — escalate.

### When to escalate to the model-evaluation-engineer peer

Escalate the calibration methodology to the `model-evaluation-engineer` peer when any of:

- The rubric is high-stakes (release-gate, safety, regulated content) and you need a **defensible variance decomposition** — how much of the disagreement is judge noise, how much is inherent rater disagreement, how much is item ambiguity. This is the domain of generalisability theory and item-response models.
- You need a **statistical power analysis** — "how big does the gold set need to be to detect a 3-point regression at α=0.05, β=0.2." Quick-calibration cannot answer that; the peer role can.
- The rubric produces **ordinal scores with unequal-interval anchors** and you need to decide whether a linear model is defensible. The peer runs the analysis; you consume the recommendation.
- You are **combining multiple raters** (LLM + human panel + a second LLM) and need a defensible aggregation rule (weighted κ, Fleiss's κ, Krippendorff's α). Not a product-altitude decision.
- **Bootstrap confidence intervals disagree with analytic ones,** or the underlying distribution has structure a naïve bootstrap gets wrong (small N, clustered items, hierarchical data). The peer runs the correct resampling scheme.

Escalation is not a failure. It is scope. The product-side calibration answers *"is this judge good enough to gate this release."* The peer's work answers *"what is the model-family posture on judge reliability for our whole track of releases, and how do we plan for it." Both need to happen; running the second one under the wrong role is how you get either wrong numbers or missed deadlines.

Write the escalation as a **contract**: *"Peer role to produce {analysis} for {rubric family}; interim provisional calibration continues in production until the peer's number replaces it."* The mod-112 governance module walks the delegation-contract shape in full; treat the calibration hand-off as an early instance of the same pattern.

### The calibration document

A calibration is a written artefact. The minimum contents:

1. **Rubric and version.** Pointer to the chapter-01 rubric doc and the prompt-template hash.
2. **Judge model.** Family, version, snapshot, weights hash (for OSS), framework wrapper.
3. **Gold set.** Size, source, stratification scheme, per-stratum counts. Where the labels live (a CSV, a Braintrust dataset, a mod-110 table).
4. **Rater(s).** Who labelled (name + role), when, whether inter-rater agreement was itself measured on a sub-sample.
5. **Statistic and value.** Which statistic, the value, the 95% CI, the number of resamples, per-stratum breakdown.
6. **Decision.** Whether the judge is considered calibrated for this rubric at this time, and against what threshold.
7. **Re-calibration triggers.** What changes require re-calibration (judge model upgrade, rubric edit, distribution shift). Chapter 05's drift monitor consumes this list.
8. **Escalation status.** Whether the model-evaluation-engineer peer has produced (or is scheduled to produce) full-methodology work, and what interim number stands.

The doc is short — a page or two — and lives next to the rubric. mod-107's online-eval loop points at the calibration when a stakeholder asks why a judge score can be trusted.

### A pragmatic recipe

Given a rubric and a judge you want to calibrate:

1. Pull a stratified sample of 50–100 items from production (respecting the mod-102 sampling and PII policy).
2. Hand-label with the rubric — the same rubric prompt the judge sees, minus the "you are a model" framing.
3. Run the judge on the same items via your framework (chapter 03). Record its outputs.
4. Compute the appropriate statistic from the table above (Cohen's κ / Spearman ρ / Kendall τ / non-tie agreement) with bootstrap 95% CI. Both `scipy.stats` and `sklearn.metrics` cover the base statistics; do the bootstrap by resampling in a small loop or with `scipy.stats.bootstrap`.
5. Break down per stratum. Report the worst stratum, not just the aggregate.
6. Write the calibration document. Include the raw paired data as a CSV.
7. If the number does not defend the release gate, iterate: chapter-01 rubric anchors → chapter-02 bias controls → chapter-03 judge choice → escalate to peer.

Do all of this once per rubric per major judge version. Chapter 05's drift monitor triggers the re-calibration on judge upgrades.

## Summary

- **Calibration = statistic-of-agreement between your judge and human raters on a stratified gold set.** It is a scoped claim per (surface, rubric, judge version).
- Target gold-set size at product altitude is **~50–100 items per criterion**, stratified by rubric outcome level, surface sub-class, and known failure categories. Below 30 is spot-check territory; above ~200–300 is peer-role territory.
- Report the right statistic for the rubric shape: **Cohen's κ** for nominal, **Spearman ρ / Kendall τ** for ordinal, **three-class κ + non-tie agreement** for pairwise. Always report bootstrap 95% CIs and the per-stratum breakdown.
- Escalate to the **`model-evaluation-engineer` peer** for variance decomposition, statistical power analysis, unequal-interval ordinal analysis, multi-rater aggregation rules, and non-standard resampling schemes. Write the escalation as a delegation contract.
- Produce a **calibration document** with rubric version, judge version, gold set, rater, statistic + CI + per-stratum, decision, and re-calibration triggers. The doc lives next to the rubric and is what mod-107 points to when asked to defend a judgement.
- Do not "run again until it looks better." Fix the rubric, the pipeline, or the judge tier — or escalate.

Chapter 05 walks judge-tier routing and drift monitoring — the runtime pattern that keeps the calibration this chapter produced from silently going stale.
